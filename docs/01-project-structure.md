# 01 — PageLM Project Structure

This document details the folder structure of the **PageLM** repository. It explains what each directory and file does, why it exists, how the directories depend on one another, and where you should look when working on a particular layer.

---

## 1. High-Level Repository Tree

Here is an architectural map of the PageLM repository:

```text
PageLM/
├── .github/                      # GitHub workflows (CI/CD, security audit) and issue templates
│   ├── ISSUE_TEMPLATE/           # Bug report and feature request templates
│   ├── workflows/                # GitHub Actions: ci.yml, security.yml
│   └── pull_request_template.md  # PR checklist and guidelines
├── assets/                       # Static media and fonts used by backend services
│   ├── fonts/                    # TTF fonts (Lexend.ttf) for PDF document generation
│   └── smartnotes/               # Fillable PDF templates for Cornell note generation
├── backend/                      # Backend TypeScript code and container configuration
│   ├── Dockerfile                # Multi-stage production container for the backend
│   ├── tsconfig.json             # Backend TypeScript compiler configuration
│   └── src/                      # Backend application source code
│       ├── agents/               # Multi-agent system (registry, runtime, tools, memory)
│       ├── config/               # Environment variable loading and configuration
│       ├── core/                 # Server entry point, custom middleware, route registry
│       ├── lib/                  # AI reasoning, document parsing, and RAG embeddings
│       ├── services/             # Feature-specific business logic (quiz, podcast, etc.)
│       └── utils/                # Utilities: custom webserver, LLM models, TTS, database
├── frontend/                     # React 19 single-page frontend application
│   ├── public/                   # Static browser assets (logo, audio orb video)
│   ├── src/                      # Frontend application source code
│   │   ├── components/           # Reusable UI components organized by feature
│   │   ├── config/               # Frontend environment settings (backend URL, timeout)
│   │   ├── lib/                  # API client, WebSocket clients, and TypeScript contracts
│   │   ├── pages/                # Route page views (Landing, Chat, Quiz, Debate, etc.)
│   │   ├── types/                # Ambient TypeScript type declarations (e.g., d3-force)
│   │   ├── App.tsx               # Root application layout (providers, sidebar, companion dock)
│   │   ├── main.tsx              # React DOM entry point and React Router route table
│   │   └── index.css             # Global styles and Tailwind CSS v4 directives
│   ├── Dockerfile                # Multi-stage production container for the frontend
│   ├── package.json              # Frontend dependencies and scripts
│   ├── pnpm-lock.yaml            # Lockfile for frontend pnpm packages
│   ├── tsconfig.json             # TypeScript project references for frontend
│   └── vite.config.ts            # Vite bundler configuration
├── modules/                      # Declarative YAML exam definitions for ExamLab
│   ├── gmat.yml                  # GMAT exam sections, question prompts, and rubrics
│   ├── gre.yml                   # GRE General Simulation definition
│   ├── ielts.yml                 # IELTS exam structure
│   ├── jee.yml                   # JEE exam specification
│   └── sat.yml                   # SAT test blueprint
├── storage/                      # Runtime data directory (created on first run, persistent)
│   ├── agents/                   # Per-session agent memory JSON files
│   ├── cache/                    # SHA256 disk cache for AI ask queries and exams
│   ├── json/                     # RAG document collections in JSON format
│   ├── podcasts/                 # Generated podcast audio files and segments
│   ├── smartnotes/               # Generated Cornell notes PDF files
│   ├── uploads/                  # Temporary uploaded files for parsing and transcription
│   └── database.sqlite           # SQLite database for Keyv key-value storage
├── .dockerignore                 # Docker build exclusions
├── .env.example                  # Template of all supported environment variables
├── .gitignore                    # Git exclusions
├── .npmrc                        # NPM configuration
├── .nvmrc                        # Preferred Node.js version (Node 21+)
├── docker-compose.yml            # Docker Compose specification for backend + frontend
├── nodemon.json                  # Hot-reload configuration for backend development
├── package.json                  # Root package file (hosts backend dependencies and scripts)
├── package-lock.json             # Root NPM lockfile
├── setup.ps1                     # Windows PowerShell quick setup script
├── setup.sh                      # Linux/macOS bash quick setup script
├── Why.md                        # Comparison between PageLM and general LLMs (ChatGPT)
└── CONTRIBUTING.md               # Contribution workflow and community rules
```

---

## 2. Directory-by-Directory Breakdown

### 2.1 Root Directory (`/`)
* **Classification**: Infrastructure / Project Configuration / Backend Root.
* **Why it exists**: The root directory anchors the repository and contains project-wide configurations, setup scripts, documentation, and the **backend's `package.json`**.
* **Key Files**:
  - `package.json`: Contains scripts (`npm run dev`, `npm run build`, `npm start`, `npm test`) and backend dependencies (`@langchain/*`, `@keyv/sqlite`, `ws`, `pdf-lib`, `node-edge-tts`, etc.).
  - `nodemon.json`: Configures `nodemon` to watch `backend/src` and transpile TypeScript on the fly using `tsx`.
  - `.env.example`: Complete inventory of all available configuration flags and API keys.
  - `docker-compose.yml`: Spins up both backend (`:5000`) and frontend (`:5173`) with volume mounting for persistent storage.

> [!NOTE]
> In this repository structure, the root `package.json` manages the **backend** dependencies. The **frontend** has its own isolated `frontend/package.json`.

---

### 2.2 `backend/`
* **Classification**: Backend server & execution engine.
* **What it contains**:
  - `Dockerfile`: Multi-stage Alpine container build running Node 22.
  - `tsconfig.json`: Compiles TypeScript files from `backend/src/` into CommonJS (`dist/`).
  - `src/`: The TypeScript backend implementation.
* **Subdirectories in `backend/src/`**:

#### `backend/src/core/`
* **Purpose**: Core server setup, routing, and HTTP/WebSocket endpoints.
* **Important Files**:
  - [`index.ts`](file:///c:/Users/sksum/OneDrive/Documents/Projects/PageLM/backend/src/core/index.ts): Initializes the server, loads `.env`, applies CORS and static file serving (`/storage`), registers all routes, and listens on port 5000.
  - [`router.ts`](file:///c:/Users/sksum/OneDrive/Documents/Projects/PageLM/backend/src/core/router.ts): Aggregates all 10 route modules into a single `registerRoutes` function.
  - [`middleware.ts`](file:///c:/Users/sksum/OneDrive/Documents/Projects/PageLM/backend/src/core/middleware.ts): Request logging middleware.
  - `routes/`: Houses the 10 domain route handlers:
    - `chat.ts`: `/chat`, `/chats`, `/chats/:id`, `/ws/chat`.
    - `quiz.ts`: `/quiz`, `/ws/quiz`.
    - `flashcards.ts`: `/flashcards`, `/flashcards/:id`.
    - `notes.ts`: `/smartnotes`, `/ws/smartnotes`.
    - `podcast.ts`: `/podcast`, `/podcast/download/...`, `/ws/podcast`.
    - `examlab.ts`: `/exam`, `/exams`, `/ws/exams`.
    - `transcriber.ts`: `/transcriber`.
    - `planner.ts`: `/tasks`, `/planner/*`, `/ws/planner`.
    - `debate.ts`: `/debate/*`, `/ws/debate`, `/ws/debate/analyze`.
    - `companion.ts`: `/api/companion/ask`.

#### `backend/src/services/`
* **Purpose**: Houses the business logic and feature implementations separate from HTTP routing.
* **Submodules**:
  - `debate/`: Logic for multi-turn debating, concession checking, and AI judicial post-debate analysis.
  - `examlab/`: Exam loader (`loader.ts`), LangGraph state graph workflow (`generate.ts`), and section item generator (`generator.ts`).
  - `planner/`: Task ingestion (`ingest.ts`), heuristic and AI step scheduling (`scheduler.ts`, `ai.ts`), and database persistence (`store.ts`, `service.ts`).
  - `podcast/`: Two-speaker script generation and audio creation via TTS and FFmpeg.
  - `quiz/`: 5-item MCQ prompt generation, strict validation, and coercion.
  - `smartnotes/`: Cornell note generation and PDF rendering (template form-filling or raw Helvetica drawing).
  - `transcriber/`: Multi-provider speech-to-text (Whisper, Google, AssemblyAI, ElevenLabs) and automatic study guide synthesis.

#### `backend/src/agents/`
* **Purpose**: An extensible AI Agent execution engine.
* **Important Files**:
  - [`registry.ts`](file:///c:/Users/sksum/OneDrive/Documents/Projects/PageLM/backend/src/agents/registry.ts): In-memory agent and tool registry.
  - [`runtime.ts`](file:///c:/Users/sksum/OneDrive/Documents/Projects/PageLM/backend/src/agents/runtime.ts): Deterministic execution loop (`execDirect`) with step timeouts, retries, and trace logs.
  - [`agents.ts`](file:///c:/Users/sksum/OneDrive/Documents/Projects/PageLM/backend/src/agents/agents.ts): Defines predefined agents (`tutor`, `researcher`, `examiner`, `podcaster`).
  - [`memory.ts`](file:///c:/Users/sksum/OneDrive/Documents/Projects/PageLM/backend/src/agents/memory.ts): Session memory serialization to `storage/agents/${sid}.json`.
  - `tools/`: Individual tool definitions (`Ragsearch.ts`, `ask.ts`, `notes.ts`, `quiz.ts`, `examlab.ts`, `podcast.ts`, `nop.ts`).

#### `backend/src/lib/`
* **Purpose**: Lower-level AI and document utilities that bridge routes, services, and raw input files.
* **Important Files**:
  - `ai/ask.ts`: Implements `handleAsk` and `askWithContext`, houses the pedagogical `BASE_SYSTEM_PROMPT`, JSON output extractor, and query cache.
  - `ai/embed.ts`: Splits text using `RecursiveCharacterTextSplitter` and invokes `saveDocuments`.
  - `parser/upload.ts`: Multipart form-data parsing (Busboy) and text extraction from PDF, DOCX, Markdown, and TXT.

#### `backend/src/utils/`
* **Purpose**: Shared infrastructure adapters.
* **Important Files**:
  - `server/server.js`: **Fubelt** custom HTTP + WebSocket server engine.
  - `llm/llm.ts` & `llm/models/`: Multi-provider model factory supporting Gemini, OpenAI, Claude, Grok, Ollama, OpenRouter, and MiniMax.
  - `database/db.ts`: Dual-mode vector retrieval (in-memory JSON vector store or Chroma).
  - `database/keyv.ts`: Persistent key-value storage backed by SQLite (`database.sqlite`).
  - `tts/index.ts`: Speech synthesis engine supporting EdgeTTS, ElevenLabs, Google TTS, and `@speech-sdk/core`.
  - `chat/ws.ts`: Safe WebSocket broadcast and large payload chunking (`emitLarge`).
  - `chat/chat.ts`: Chat metadata and message history persistence helpers.

---

### 2.3 `frontend/`
* **Classification**: Client-side React application.
* **What it contains**:
  - `src/main.tsx`: Client entry point that mounts React 19 into `#root` and configures `BrowserRouter`.
  - `src/App.tsx`: Base layout providing the `CompanionProvider`, toast notifications, collapsible navigation `Sidebar`, and global `CompanionDock`.
  - `src/index.css`: Global styling and `@import "tailwindcss";` for Tailwind CSS v4.
  - `src/config/env.ts`: Reads `import.meta.env.VITE_BACKEND_URL` and `VITE_TIMEOUT`.
  - `src/lib/api.ts`: Comprehensive API client library wrapping HTTP requests and WebSocket streams.
* **Pages (`frontend/src/pages/`)**:
  - `Landing.tsx`: Main entrance where users can type prompts or drop study files in either "Chat" or "Quiz" mode.
  - `Chat.tsx`: Active learning chat view with Markdown responses, flashcards side rail, and Study Bag drawer.
  - `Quiz.tsx`: Interactive multiple-choice quiz runner with real-time scoring and answer review.
  - `Tools.tsx`: Tabbed tools hub housing SmartNotes, Podcast Generator, and Audio Transcriber.
  - `FlashCards.tsx`: "My Learning Bag" page displaying saved flashcards and study notes.
  - `examlab.tsx`: Exam simulation interface displaying available standardized exam modules.
  - `Planner.tsx`: Homework planner hosting Today's Focus, Weekly Schedule, and Mindmap views.
  - `Debate.tsx`: Live debate interface with real-time argument streaming and post-debate judge verdict.
  - `404.tsx`: Fallback page for non-existent routes.
* **Components (`frontend/src/components/`)**:
  - `Chat/`: `Composer.tsx`, `MarkdownView.tsx`, `FlashCards.tsx`, `ActionRow.tsx`, `SelectionPopup.tsx`, `BagDrawer.tsx`, `BagFab.tsx`, `LoadingIndicator.tsx`.
  - `Companion/`: `CompanionProvider.tsx`, `CompanionDock.tsx`.
  - `Landing/`: `PromptBox.tsx`, `PromptRail.tsx`, `ExploreTopics.tsx`.
  - `Quiz/`: `QuestionCard.tsx`, `QuizHeader.tsx`, `ResultsPanel.tsx`, `ReviewModal.tsx`, `TopicBar.tsx`.
  - `Tools/`: `SmartNotes.tsx`, `PodcastGenerator.tsx`, `Transcriber.tsx`, `ComingSoon.tsx`.
  - `planner/`: `Planner.tsx`, `PlannerMindmap.tsx`, `TodayFocus.tsx`, `QuickAdd.tsx`, `mindmap/geometry.ts`, `mindmap/engine.ts`.
  - `Sidebar.tsx`: Collapsible vertical navigation bar.

---

### 2.4 `modules/`
* **Classification**: Configuration / Educational Content Specifications.
* **What it contains**: YAML files defining multi-section standardized exams for the **ExamLab** feature:
  - `gre.yml`: Quantitative, Verbal, and Analytical Writing sections.
  - `gmat.yml`: Quantitative and Verbal reasoning sections.
  - `ielts.yml`: Academic reading and writing sections.
  - `jee.yml`: Mathematics, Physics, and Chemistry problem sections.
  - `sat.yml`: Math and Reading/Writing test sections.
* **How it is used**: Read by `backend/src/services/examlab/loader.ts` to instruct LangGraph how to generate tailored examination questions and rubrics.

---

### 2.5 `storage/` (Runtime Directory)
* **Classification**: Persistent Runtime Storage.
* **What it contains**: Created dynamically when PageLM starts:
  - `database.sqlite`: SQLite database storing chats, messages, flashcards, planner tasks, and debate sessions.
  - `json/`: Extracted document chunk collections used by the default `json` vector database mode.
  - `cache/`: SHA256 disk cache for AI query responses and generated exam items to prevent redundant LLM invocations.
  - `uploads/`: Temporary files uploaded via multipart requests before extraction.
  - `podcasts/`: Audio files and segment MP3s created by TTS pipelines.
  - `smartnotes/`: Completed Cornell notes PDF files.
  - `agents/`: JSON files containing session memory for multi-turn agents.

---

## 3. Relationships & Data Dependencies

The diagram below shows how control and data flow from the user interface down to the backend storage and AI models:

```text
[Frontend Pages]
   e.g. Chat.tsx, Quiz.tsx, Planner.tsx, Debate.tsx
         │
         ▼
[Frontend Components]
   e.g. Composer, PromptBox, PlannerMindmap, CompanionDock
         │
         ▼
[Frontend API Layer]
   frontend/src/lib/api.ts
         │ (HTTP REST / File Uploads & WebSocket Streams)
         ▼
[Backend Core Router]
   backend/src/core/router.ts -> backend/src/core/routes/*
         │
         ├────────────────────────────────────────┐
         ▼                                        ▼
[Backend Services Layer]                 [Backend Lib / AI / Parser]
   services/quiz, services/podcast,         lib/ai/ask.ts, lib/ai/embed.ts,
   services/planner, services/examlab       lib/parser/upload.ts
         │                                        │
         ├───────────────────┬────────────────────┘
         ▼                   ▼
[Agent System]       [AI Model Factory]         [Persistence Layer]
   agents/runtime       utils/llm/llm.ts           utils/database/db.ts
   agents/tools         utils/llm/models/*         utils/database/keyv.ts
         │                   │                          │
         ▼                   ▼                          ▼
 External Providers  LLM APIs (Gemini/GPT/Ollama)  storage/database.sqlite
 & TTS Engines       TTS APIs (Edge/ElevenLabs)    storage/json/*.json
```
