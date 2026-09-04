# 05 — Backend Architecture & Layering

This document explains the architecture of the **PageLM backend**. It covers the server lifecycle, routing, services, utilities, database integration, AI abstractions, and agent pipelines, explaining the architectural purpose of each layer and tracing real examples through the code.

---

## 1. Server Entry Point & Lifecycle

The backend starts in [`backend/src/core/index.ts`](file:///c:/Users/sksum/OneDrive/Documents/Projects/PageLM/backend/src/core/index.ts). The server boot process unfolds in six steps:

```text
1. process.loadEnvFile('.env')
   ↓
2. app = server()                      (Initializes Fubelt custom HTTP server)
   ↓
3. app.use(loggerMiddleware)           (Request method & path logging)
   ↓
4. app.use(cors(...))                  (CORS headers configuration)
   ↓
5. app.use(app.serverStatic("/storage", "./storage")) (Static file streaming)
   ↓
6. registerRoutes(app)                 (Registers all 10 domain route groups)
   ↓
7. app.listen(5000)                    (Binds HTTP & WebSocket upgrade listeners)
```

### 1.1 Environment Loading
PageLM uses Node.js's built-in `process.loadEnvFile(path.resolve(process.cwd(), '.env'))`. This eliminates external dependencies like `dotenv` and requires Node `>= 21.18.0`.

### 1.2 Custom Server Engine: Fubelt
The server does **not** import Express. Instead, it imports `server` from `backend/src/utils/server/server.js`:
- Built on `http.createServer()` and `WebSocket.Server({ noServer: true })`.
- Provides Express-like methods: `app.get()`, `app.post()`, `app.patch()`, `app.delete()`, `app.use()`, and `app.ws()`.
- Express-like response decorations: `res.status(code)`, `res.json(obj)`, `res.send(data)`.
- Automatic JSON parsing: parses incoming `application/json` bodies up to a 1MB limit (`1e6`).

---

## 2. Architectural Layers: Understanding the Separation

A beginner might wonder: *Why not put all the code directly in the route handler?*
PageLM strictly separates responsibilities into distinct architectural layers:

```text
┌─────────────────────────────────────────────────────────────────────────┐
│ 1. ROUTE LAYER (core/routes/*)                                          │
│    - Validates incoming HTTP requests and WebSocket connections         │
│    - Returns immediate HTTP responses (e.g. 202 Accepted)               │
│    - Manages WebSocket client Sets for connection lifecycle             │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │ Calls
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 2. SERVICE LAYER (services/*)                                           │
│    - Implements domain-specific business logic                          │
│    - Builds domain data models (quizzes, podcasts, exam state machines) │
│    - Orchestrates multi-step tasks                                      │
└──────────────────┬─────────────────┴──────────────────┬─────────────────┘
                   │ Calls                              │ Calls
                   ▼                                    ▼
┌──────────────────────────────────────┐  ┌───────────────────────────────┐
│ 3. AGENT LAYER (agents/*)            │  │ 4. AI ABSTRACTION (utils/llm) │
│    - Executes multi-step tool plans  │  │    - Standardizes LLM APIs    │
│    - Handles step retries & timeouts │  │    - Abstracts model providers│
│    - Records execution traces        │  │    - Formats system prompts   │
└──────────────────┬───────────────────┘  └─────────────┬─────────────────┘
                   │                                    │
                   ▼                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 5. DATABASE & PERSISTENCE LAYER (utils/database/*)                      │
│    - Manages SQLite key-value data (Keyv)                               │
│    - Manages document vector storage (MemoryVectorStore / Chroma)       │
│    - Manages disk cache (storage/cache/*)                               │
└─────────────────────────────────────────────────────────────────────────┘
```

### Why This Separation Matters:
1. **Separation of Concerns**: If you want to change how questions are scored, you modify `services/quiz/index.ts`, without touching HTTP status codes in `core/routes/quiz.ts`.
2. **Provider Swapping**: If you switch from Google Gemini to OpenAI or Ollama, zero changes are required in the services or routes—only `utils/llm/llm.ts` adapts.
3. **Testability**: Services can be unit tested in isolation without spinning up HTTP servers or mocking network sockets.

---

## 3. Core Route Groups (`core/router.ts`)

All routes are registered via `registerRoutes(app)` in `backend/src/core/router.ts`:

1. **`chatRoutes`** (`routes/chat.ts`):
   - `POST /chat`: Receives questions and multipart study files; returns `202 Accepted` with chat ID and WebSocket stream URL.
   - `GET /chats`: Lists the 50 most recent chat sessions.
   - `GET /chats/:id`: Returns detailed message history for a chat.
   - `WS /ws/chat`: Streams real-time upload progress, generating phases, and final answers.
2. **`quizRoutes`** (`routes/quiz.ts`):
   - `POST /quiz`: Starts a quiz generation job for a topic.
   - `WS /ws/quiz`: Streams generation status, heartbeats, and quiz payload.
3. **`flashcardRoutes`** (`routes/flashcards.ts`):
   - `POST /flashcards`: Saves a flashcard or note to SQLite.
   - `GET /flashcards`: Retrieves all saved learning bag items.
   - `DELETE /flashcards/:id`: Deletes a card from the learning bag.
4. **`smartnotesRoutes`** (`routes/notes.ts`):
   - `POST /smartnotes`: Initiates Cornell note generation.
   - `WS /ws/smartnotes`: Emits progress and returns the generated PDF storage URL.
5. **`podcastRoutes`** (`routes/podcast.ts`):
   - `POST /podcast`: Initiates two-speaker podcast creation.
   - `GET /podcast/download/:pid/:filename`: Binary audio download endpoint.
   - `WS /ws/podcast`: Streams script creation, audio synthesis progress, and download URLs.
6. **`examRoutes`** (`routes/examlab.ts`):
   - `GET /exams`: Returns available standardized exam blueprints (`modules/*.yml`).
   - `POST /exam`: Runs an exam simulation via LangGraph.
   - `WS /ws/exams`: Streams generated exam sections using `emitLarge`.
7. **`transcriberRoutes`** (`routes/transcriber.ts`):
   - `POST /transcriber`: Accepts multipart audio, calls STT providers, and returns transcripts with AI study guides.
8. **`plannerRoutes`** (`routes/planner.ts`):
   - `POST /tasks`: Ingests and schedules homework tasks.
   - `GET /tasks`: Lists filtered tasks.
   - `POST /planner/weekly`: Generates Pomodoro slots across the week.
   - `WS /ws/planner`: Broadcasts task updates, daily 8 AM digests, and break reminders.
9. **`debateRoutes`** (`routes/debate.ts`):
   - `POST /debate/start`: Starts a structured debate session.
   - `POST /debate/:debateId/argue`: Submits user argument and triggers streaming rebuttal.
   - `POST /debate/:debateId/analyze`: Triggers AI judicial evaluation.
   - `WS /ws/debate` & `WS /ws/debate/analyze`: Streams live counterarguments and analysis.
10. **`companionRoutes`** (`routes/companion.ts`):
    - `POST /api/companion/ask`: Answers questions grounded strictly in the provided document text or file path.

---

## 4. Tracing a Real Execution Path: From Route to Agent to Storage

To see how these pieces connect, let's trace what happens when a user asks a question in the chat:

```text
1. Client POST /chat with { q: "How does backpropagation work?" }
   ↓
2. backend/src/core/routes/chat.ts
   - Creates or loads chat session via mkChat()
   - Responds with HTTP 202 Accepted { ok: true, chatId, stream: "/ws/chat?chatId=..." }
   - Launches async background task
   ↓
3. Background task calls handleAsk() in backend/src/lib/ai/ask.ts
   - Invokes the AI Agent system:
     execDirect({
       agent: "researcher",
       plan: { steps: [{ tool: "rag.search", input: { q, ns: "chat:id", k: 6 } }] }
     })
   ↓
4. backend/src/agents/runtime.ts executes the step:
   - Finds "rag.search" tool in researcher agent tools
   - Runs tool in backend/src/agents/tools/Ragsearch.ts
   ↓
5. Ragsearch calls getRetriever() in backend/src/utils/database/db.ts
   - Loads document chunk collection from storage/json/chat:id.json
   - Performs vector similarity search and returns top-6 passages
   ↓
6. handleAsk() receives context, formats BASE_SYSTEM_PROMPT, and calls:
   llm.call(messages)
   ↓
7. backend/src/utils/llm/llm.ts invokes the active provider (e.g. Gemini or GPT-4o)
   ↓
8. handleAsk() receives response, extracts JSON, caches to storage/cache/ask/,
   and returns { topic, answer, flashcards }
   ↓
9. chat.ts saves assistant answer to SQLite via addMsg()
   and broadcasts to WebSocket:
   emitToAll(chatSockets.get(id), { type: "answer", answer })
   emitToAll(chatSockets.get(id), { type: "done" })
```

---

## 5. Error Handling & Guardrails

1. **Request Timeouts**: Long-running asynchronous service jobs are wrapped in `withTimeout(promise, ms, label)` to prevent hung processes from consuming memory indefinitely.
2. **Body Size Protections**:
   - `server.js` enforces a 1,000,000-byte (1MB) limit on incoming JSON payloads, returning HTTP `413 Payload Too Large` if exceeded.
   - Document companion reads in `routes/companion.ts` enforce a 1.5MB file read guardrail (`MAX_BYTES`).
3. **Directory Traversal Protection**:
   - The static file server (`serverStatic`) checks `path.relative(root, target)` to ensure clients cannot access files outside `./storage`.
   - The companion file loader checks candidate paths against `allowedRoots` (`./storage` and `./assets`).
4. **WebSocket Safety**:
   - Socket closures clean up the active `Set` in each route's socket map; if no clients remain, the map key is deleted to prevent memory leaks.
