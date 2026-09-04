# 06. API & Routing Guide

This document provides a comprehensive, exhaustive reference for all HTTP endpoints and WebSocket channels exposed by the PageLM backend server. It details request/response contracts, headers, status codes, payload structures, error formats, and the corresponding frontend callers.

---

## 1. Overview of Server & Protocol Architecture

PageLM runs a unified HTTP/1.1 and WebSocket server built directly on Node.js core modules (`http` and `ws`) encapsulated in the custom **Fubelt** server framework (`backend/src/utils/server/server.js`).

### Port and Base URL
- **Default Port**: `5000` (Configurable via `PORT` env var).
- **Backend Base URL**: `http://localhost:5000`
- **WebSocket Base URL**: `ws://localhost:5000`
- **Frontend Proxy**: Vite dev server proxies `/api` and `/storage` requests to `http://localhost:5000`.

### Common Patterns
1. **Asynchronous Generation Pattern (HTTP 202 Accepted)**:
   Long-running AI jobs (Chat, Quiz, Notes, Podcasts, ExamLab, Debate analysis) do not block HTTP connections. Instead:
   - Client sends `POST /<feature>`.
   - Server validates parameters, generates a unique job UUID (`chatId`, `quizId`, `runId`, `pid`, `noteId`, `debateId`), and returns `202 Accepted` immediately with `{ ok: true, id, stream: "/ws/<feature>?id=..." }`.
   - Client connects to the corresponding WebSocket URL to receive real-time phases, streaming chunks, progress events, and the final payload.
2. **Synchronous REST Pattern (HTTP 200 OK)**:
   CRUD operations (e.g., Flashcards, Tasks, Sessions, Stats, Chat history) respond synchronously with standard JSON envelopes `{ ok: true, ... }`.
3. **Multipart File Ingestion**:
   Endpoints supporting document or audio uploads (`/chat`, `/transcriber`, `/tasks`, `/tasks/:id/files`) accept `multipart/form-data` parsed via streaming Busboy implementations.

---

## 2. Complete Route Catalog

| Category | Method | Path | Protocol | Purpose |
| :--- | :--- | :--- | :--- | :--- |
| **Chat** | `WS` | `/ws/chat?chatId=:id` | WebSocket | Real-time chat streaming & upload progress |
| | `POST` | `/chat` | HTTP (JSON / Multipart) | Start or continue chat turn (returns 202) |
| | `GET` | `/chats` | HTTP | List all chat sessions |
| | `GET` | `/chats/:id` | HTTP | Fetch single chat session & message history |
| **Quiz** | `WS` | `/ws/quiz?quizId=:id` | WebSocket | Real-time quiz generation stream |
| | `POST` | `/quiz` | HTTP | Request multi-question quiz generation (returns 202) |
| **Flashcards** | `POST` | `/flashcards` | HTTP | Create a flashcard |
| | `GET` | `/flashcards` | HTTP | List all flashcards |
| | `DELETE` | `/flashcards/:id` | HTTP | Delete flashcard by ID |
| **SmartNotes** | `WS` | `/ws/smartnotes?noteId=:id` | WebSocket | Note generation progress stream |
| | `POST` | `/smartnotes` | HTTP | Generate markdown study notes & synthesis (returns 202) |
| **Podcast** | `WS` | `/ws/podcast?pid=:id` | WebSocket | Script generation & audio synthesis stream |
| | `POST` | `/podcast` | HTTP | Trigger two-host dialogue & TTS synthesis (returns 202) |
| | `GET` | `/podcast/download/:pid/:filename` | HTTP | Download rendered MP3 audio file |
| **ExamLab** | `WS` | `/ws/exams?runId=:id` | WebSocket | Exam generation chunked stream |
| | `GET` | `/exams` | HTTP | List all available exam specifications |
| | `POST` | `/exam` | HTTP | Generate single exam by specification ID (returns 202) |
| | `POST` | `/exams` | HTTP | Batch generate all configured exams (returns 202) |
| **Transcriber** | `POST` | `/transcriber` | HTTP (Multipart) | Upload audio/video file for STT transcription |
| **Planner & Tasks** | `WS` | `/ws/planner?sid=:sid` | WebSocket | Planner room notifications, reminders & live updates |
| | `POST` | `/tasks` | HTTP (JSON / Multipart) | Ingest syllabus/assignment or create structured task |
| | `POST` | `/tasks/ingest` | HTTP | Ingest raw unstructured text into task plan |
| | `GET` | `/tasks` | HTTP | Filter and list tasks |
| | `GET` | `/tasks/:id` | HTTP | Get specific task details |
| | `PATCH` | `/tasks/:id` | HTTP | Update task fields (status, priority, deadline) |
| | `DELETE` | `/tasks/:id` | HTTP | Remove task |
| | `POST` | `/tasks/:id/plan` | HTTP | Trigger AI decomposition of task into study slots |
| | `POST` | `/tasks/:id/replan` | HTTP | Re-run scheduler for missed or modified tasks |
| | `POST` | `/tasks/:id/materials` | HTTP | AI-generate study guides, checklists, or summaries |
| | `POST` | `/tasks/:id/files` | HTTP (Multipart) | Attach files/readings to a task |
| | `DELETE` | `/tasks/:id/files/:fileId` | HTTP | Detach file from task |
| | `PATCH` | `/slots/:taskId/:slotId` | HTTP | Mark a specific study slot as completed or skipped |
| | `POST` | `/sessions/start` | HTTP | Start timed study session |
| | `POST` | `/sessions/:id/stop` | HTTP | Stop study session and record minutes worked |
| | `POST` | `/planner/weekly` | HTTP | Generate full weekly balanced schedule |
| | `GET` | `/planner/today` | HTTP | Get today's scheduled study sessions |
| | `GET` | `/planner/deadlines` | HTTP | Fetch upcoming deadlines grouped chronologically |
| | `GET` | `/planner/stats` | HTTP | Retrieve user productivity stats |
| | `POST` | `/reminders/schedule` | HTTP | Schedule targeted notification |
| | `POST` | `/reminders/test` | HTTP | Trigger instant test notification |
| **Debate** | `WS` | `/ws/debate?debateId=:id` | WebSocket | Debate exchange streaming (arguments, rebuttal, concession) |
| | `WS` | `/ws/debate/analyze?debateId=:id`| WebSocket | Stream debate evaluation & judge scoring |
| | `POST` | `/debate/start` | HTTP | Initialize debate session on a topic |
| | `POST` | `/debate/:debateId/argue` | HTTP | Submit user argument (returns 202) |
| | `GET` | `/debate/:debateId` | HTTP | Fetch debate history and status |
| | `GET` | `/debates` | HTTP | List all past debate matches |
| | `DELETE` | `/debate/:debateId` | HTTP | Delete debate session |
| | `POST` | `/debate/:debateId/surrender` | HTTP | Forfeit current debate |
| | `POST` | `/debate/:debateId/analyze` | HTTP | Request rubric evaluation (returns 202) |
| **Study Companion** | `POST` | `/api/companion/ask` | HTTP | Grounded document Q&A with strict context boundaries |

---

## 3. Deep-Dive Specification by Route Group

### 3.1 Chat System (`backend/src/core/routes/chat.ts`)

#### WebSocket: `/ws/chat?chatId=:chatId`
- **Query Params**: `chatId` (string, required).
- **Client Connect**: Returns `{ type: "ready", chatId }`.
- **Outgoing Events from Server**:
  - `{ type: "phase", value: "upload_start" | "upload_done" | "generating" }`
  - `{ type: "file", filename: string, mime: string }`
  - `{ type: "answer", answer: string }`
  - `{ type: "done" }`
  - `{ type: "error", error: string }`

#### `POST /chat`
- **Headers**:
  - `Content-Type: application/json` OR `multipart/form-data`
- **Body (JSON)**:
  ```json
  {
    "q": "Explain quantum tunneling in simple terms",
    "chatId": "optional-uuid-to-continue-session"
  }
  ```
- **Body (Multipart)**:
  - Form field `q`: Query string.
  - Form field `chatId`: Optional session ID.
  - File fields: One or more uploaded PDFs/DOCX/TXT files.
- **Success Response (`202 Accepted`)**:
  ```json
  {
    "ok": true,
    "chatId": "7f8b9c0d-1234-5678-9abc-def012345678",
    "stream": "/ws/chat?chatId=7f8b9c0d-1234-5678-9abc-def012345678"
  }
  ```
- **Error Responses**:
  - `400 Bad Request`: `{ "error": "q required" }`
  - `500 Internal Server Error`: Handled by global error middleware.

#### `GET /chats`
- **Response (`200 OK`)**:
  ```json
  {
    "ok": true,
    "chats": [
      { "id": "uuid-1", "title": "Quantum tunneling query", "createdAt": 1741000000000 }
    ]
  }
  ```

#### `GET /chats/:id`
- **Response (`200 OK`)**:
  ```json
  {
    "ok": true,
    "chat": { "id": "uuid-1", "title": "Quantum tunneling query" },
    "messages": [
      { "role": "user", "content": "Explain quantum tunneling...", "at": 1741000000000 },
      { "role": "assistant", "content": "Quantum tunneling occurs when...", "at": 1741000002000 }
    ]
  }
  ```

---

### 3.2 Quiz Generator (`backend/src/core/routes/quiz.ts`)

#### WebSocket: `/ws/quiz?quizId=:quizId`
- **Heartbeat**: Sends `{ type: "ping", t: 1741000000000 }` every 15s.
- **Events**:
  - `{ type: "ready", quizId }`
  - `{ type: "phase", value: "generating" }`
  - `{ type: "quiz", quiz: [ { question, options, answer, explanation } ] }`
  - `{ type: "done" }`
  - `{ type: "error", error: string }`

#### `POST /quiz`
- **Body**:
  ```json
  { "topic": "Photosynthesis Light Reactions" }
  ```
- **Response (`202 Accepted`)**:
  ```json
  {
    "ok": true,
    "quizId": "4c9a8f21-...",
    "stream": "/ws/quiz?quizId=4c9a8f21-..."
  }
  ```

---

### 3.3 Flashcard Management (`backend/src/core/routes/flashcards.ts`)

#### `POST /flashcards`
- **Body**:
  ```json
  {
    "question": "What is the Krebs Cycle?",
    "answer": "A series of chemical reactions used by aerobic organisms...",
    "tag": "Biology"
  }
  ```
- **Response (`200 OK`)**:
  ```json
  {
    "ok": true,
    "flashcard": {
      "id": "uuid",
      "question": "What is the Krebs Cycle?",
      "answer": "...",
      "tag": "Biology",
      "created": 1741000000000
    }
  }
  ```

#### `GET /flashcards`
- **Response (`200 OK`)**:
  ```json
  {
    "ok": true,
    "flashcards": [ ... ]
  }
  ```

#### `DELETE /flashcards/:id`
- **Response (`200 OK`)**:
  ```json
  { "ok": true }
  ```

---

### 3.4 SmartNotes (`backend/src/core/routes/notes.ts`)

#### WebSocket: `/ws/smartnotes?noteId=:noteId`
- **Heartbeat**: Sends `{ type: "ping" }` every 15s.
- **Events**:
  - `{ type: "ready", noteId }`
  - `{ type: "phase", value: "generating" }`
  - `{ type: "file", file: "http://localhost:5000/storage/smartnotes/uuid.md" }`
  - `{ type: "done" }`

#### `POST /smartnotes`
- **Body**:
  ```json
  {
    "topic": "Thermodynamics",
    "notes": "Optional raw user notes or lecture scribbles",
    "filePath": "optional/path/to/uploaded/doc.pdf"
  }
  ```
- **Response (`202 Accepted`)**:
  ```json
  {
    "ok": true,
    "noteId": "uuid",
    "stream": "/ws/smartnotes?noteId=uuid"
  }
  ```

---

### 3.5 Podcast Generation (`backend/src/core/routes/podcast.ts`)

#### WebSocket: `/ws/podcast?pid=:pid`
- **Events**:
  - `{ type: "ready", pid }`
  - `{ type: "script", data: [ { speaker: "Host 1", text: "..." }, { speaker: "Host 2", text: "..." } ] }`
  - `{ type: "progress", segment: 1, total: 10 }`
  - `{ type: "audio", file: "/podcast/download/:pid/:filename", staticUrl: "...", filename: "..." }`
  - `{ type: "done" }`

#### `POST /podcast`
- **Body**:
  ```json
  { "topic": "History of the Roman Republic" }
  ```
- **Response (`202 Accepted`)**:
  ```json
  {
    "ok": true,
    "pid": "uuid",
    "stream": "/ws/podcast?pid=uuid"
  }
  ```

#### `GET /podcast/download/:pid/:filename`
- **Streaming Response**: Returns binary `audio/mpeg` with `Content-Disposition: attachment; filename="..."` via Node.js `fs.createReadStream`.

---

### 3.6 ExamLab (`backend/src/core/routes/examlab.ts`)

#### `GET /exams`
- **Response (`200 OK`)**:
  Lists all pre-configured exam blueprints found in `services/examlab/loader.ts`.
  ```json
  {
    "ok": true,
    "exams": [
      {
        "id": "gre-quant-mini",
        "name": "GRE Quantitative Mini Test",
        "sections": [
          { "id": "sec1", "title": "Arithmetic & Algebra", "durationSec": 1200, "gen": { "type": "mcq", "count": 10 } }
        ]
      }
    ]
  }
  ```

#### WebSocket: `/ws/exams?runId=:runId`
- Uses `emitLarge` chunking (128 KB chunks) for payloads exceeding standard WebSocket frame limits:
  - `{ type: "phase", value: "generating", examId }`
  - `{ type: "exam:start", id, totalChunks }`
  - `{ type: "exam:chunk", id, chunkIndex, data }`
  - `{ type: "exam:end", id }`
  - `{ type: "done" }`

#### `POST /exam`
- **Body**: `{ "examId": "gre-quant-mini" }`
- **Response (`202 Accepted`)**: `{ "ok": true, "runId": "uuid", "stream": "/ws/exams?runId=uuid" }`

#### `POST /exams` (Batch generation)
- Generates all configured exams consecutively through the WebSocket run stream.

---

### 3.7 Audio Transcriber (`backend/src/core/routes/transcriber.ts`)

#### `POST /transcriber`
- **Content-Type**: `multipart/form-data`
- **Form Fields**:
  - `provider`: `"groq" | "openai" | "gemini"` (defaults to `config.transcription_provider`).
  - `file`: Binary audio or video file (`.webm`, `.mp3`, `.wav`, `.m4a`, `.mp4`).
- **Response (`200 OK`)**:
  ```json
  {
    "ok": true,
    "transcription": "Here is the transcribed lecture text...",
    "provider": "groq",
    "duration": 42.5,
    "confidence": 0.98
  }
  ```

---

### 3.8 Planner & Task Management (`backend/src/core/routes/planner.ts`)

#### WebSocket: `/ws/planner?sid=:sid`
- Room-based event broadcaster (default room `"default"`).
- Emits task lifecycle events:
  - `task.created`, `task.updated`, `task.deleted`
  - `plan.update`, `slot.update`
  - `session.started`, `session.ended`
  - `reminder`, `daily.digest`, `break.reminder`, `evening.review`

#### Key HTTP Endpoints:
- `POST /tasks`: Accepts task description or uploaded syllabus document.
- `POST /tasks/:id/plan`: Prompts LLM to break task into granular study slots with milestones.
- `POST /planner/weekly`: Balances study tasks across the student's week with energy levels.
- `GET /planner/today`: Retrieves active sessions for today.
- `GET /planner/deadlines`: Categorizes tasks into `overdue`, `today`, `thisWeek`, `later`.
- `GET /planner/stats`: Calculates total hours studied, completion rates, and streak data.
- `POST /sessions/start` & `POST /sessions/:id/stop`: Manages Pomodoro/study intervals.

---

### 3.9 Debate Platform (`backend/src/core/routes/debate.ts`)

#### WebSocket Channels:
1. `/ws/debate?debateId=:id`: Emits argument tokens, opponent rebuttals, concessions (`ai_concede`).
2. `/ws/debate/analyze?debateId=:id`: Emits phase transitions (`analyzing`, `scoring`) and final rubric scorecard.

#### `POST /debate/start`
- **Body**: `{ "topic": "Should AI replace standardized testing?", "position": "for" }`
- **Response**: `{ "ok": true, "debateId": "uuid", "stream": "/ws/debate?debateId=uuid" }`

#### `POST /debate/:debateId/argue`
- **Body**: `{ "argument": "Standardized testing fails to capture individual creative problem-solving..." }`
- **Response (`202 Accepted`)**: AI argument generation commences over WebSocket.

#### `POST /debate/:debateId/analyze`
- **Response (`202 Accepted`)**: Evaluator LLM assesses logic, factual grounding, rhetorical strength, and names a winner.

---

### 3.10 Study Companion (`backend/src/core/routes/companion.ts`)

#### `POST /api/companion/ask`
- **Body**:
  ```json
  {
    "question": "What does section 3 mean by entropy?",
    "documentText": "Full text of document...",
    "filePath": "storage/uploads/lecture1.pdf",
    "topic": "Physics",
    "history": [ { "role": "user", "content": "..." } ]
  }
  ```
- **Guardrails**:
  - `MAX_BYTES`: 1.5 MB limit for document reads.
  - Path traversal checks against `storage/` and `assets/` directories.
- **Response (`200 OK`)**:
  ```json
  {
    "ok": true,
    "companion": "Based strictly on Section 3, entropy represents..."
  }
  ```

---

## 4. Frontend Caller Mapping Table

To easily trace which frontend page or component calls which endpoint:

| Backend Route | Frontend File | Invoking Hook / Function |
| :--- | :--- | :--- |
| `POST /chat` & `/ws/chat` | `frontend/src/pages/Chat.tsx` | `sendQuestion` / WebSocket hook |
| `POST /quiz` & `/ws/quiz` | `frontend/src/pages/Quiz.tsx` | Form submit handler |
| `GET / POST / DELETE /flashcards` | `frontend/src/pages/Flashcards.tsx` | `fetchCards`, `addCard`, `deleteCard` |
| `POST /smartnotes` & `/ws/smartnotes`| `frontend/src/pages/SmartNotes.tsx` | Note generation trigger |
| `POST /podcast` & `/ws/podcast` | `frontend/src/pages/Podcast.tsx` | `startPodcast` handler |
| `GET /exams`, `POST /exam` | `frontend/src/pages/ExamLab.tsx` | `loadExams`, `runExam` |
| `POST /transcriber` | `frontend/src/components/Transcriber.tsx` | File upload / recording submit |
| `/tasks`, `/planner/*` | `frontend/src/pages/Planner.tsx` | `usePlanner` / React Query queries |
| `/debate/*` | `frontend/src/pages/Debate.tsx` | `handleSendArgument`, `handleAnalyze` |
| `POST /api/companion/ask` | `frontend/src/components/CompanionDock.tsx`| Floating drawer submit handler |

---

## 5. Summary & Key Takeaways for Beginners

1. **Dual Comm Channel**: Any feature producing creative AI output connects an HTTP trigger with a WebSocket stream. When developing a new feature, mirror this two-step architecture.
2. **Error Resiliency**: Backend HTTP handlers wrap generation in `setImmediate` or detached async runners so that timeouts or LLM rate limits do not crash the main HTTP server process.
3. **Inspect WebSocket Frames**: When debugging API issues in the browser, open **DevTools > Network > WS** to inspect the sequence of JSON packets sent across `/ws/*`.
