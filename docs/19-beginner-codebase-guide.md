# 19. Beginner's Onboarding & Codebase Navigation Guide

Welcome! If you are a beginner full-stack developer who is new to open-source, or if this is your first time working on a production AI application, this guide was written specifically for you.

---

## 1. Why PageLM is Built Differently

Most AI tutorials show you a simple Python script calling `openai.chat.completions.create(...)` and printing the output. While that works for prototypes, real-world full-stack AI applications face different challenges:
1. **Long Latencies**: Generating multi-step study guides or audio takes 10 to 60 seconds. You cannot block HTTP requests.
2. **Pedagogical Integrity**: Standard chatbots give answers immediately, which encourages passive memorization. PageLM enforces active learning through structured prompts, Feynman analogies, and auto-generated flashcards.
3. **Resilience**: API keys can run out of quota, networks can stutter, and users can upload corrupt PDFs. PageLM uses fallback providers, timeouts, retries, and disk caches.

---

## 2. Recommended Reading Order (How to Explore Without Overwhelm)

Do not try to read all 150+ files at once! Follow this step-by-step reading roadmap:

```mermaid
flowchart TD
    Day1[Phase 1: The Core Loop] --> Read1[1. backend/src/core/index.ts & router.ts]
    Read1 --> Read2[2. backend/src/core/routes/flashcards.ts (Simplest CRUD)]
    Read2 --> Read3[3. backend/src/lib/ai/ask.ts (The Heart of AI Prompts)]
    
    Day1 --> Day2[Phase 2: Real-time Communication]
    Day2 --> Read4[4. backend/src/utils/chat/ws.ts (emitToAll & emitLarge)]
    Read4 --> Read5[5. backend/src/core/routes/quiz.ts (HTTP 202 + WebSocket)]
    Read5 --> Read6[6. frontend/src/lib/api.ts (Client API Wrappers)]
    
    Day2 --> Day3[Phase 3: Deep AI Architecture]
    Day3 --> Read7[7. backend/src/utils/llm/models/index.ts (Model Factory)]
    Read7 --> Read8[8. backend/src/lib/parser/upload.ts & embed.ts (RAG Pipeline)]
    Read8 --> Read9[9. backend/src/services/examlab/generate.ts (LangGraph StateGraph)]
```

### Phase 1: Understanding the Basics (1–2 Hours)
1. **[backend/src/core/index.ts](file:///c:/Users/sksum/OneDrive/Documents/Projects/PageLM/backend/src/core/index.ts)**: See how the server starts and loads configuration.
2. **[backend/src/core/routes/flashcards.ts](file:///c:/Users/sksum/OneDrive/Documents/Projects/PageLM/backend/src/core/routes/flashcards.ts)**: The simplest REST endpoint in the project. Shows how `db.set` and `db.get` work with Keyv and SQLite.
3. **[backend/src/lib/ai/ask.ts](file:///c:/Users/sksum/OneDrive/Documents/Projects/PageLM/backend/src/lib/ai/ask.ts)**: Read `BASE_SYSTEM_PROMPT`. Notice how it forbids rote learning and forces structured JSON output.

### Phase 2: Understanding Real-Time Communication (2 Hours)
4. **[backend/src/core/routes/quiz.ts](file:///c:/Users/sksum/OneDrive/Documents/Projects/PageLM/backend/src/core/routes/quiz.ts)**: Understand why the route returns `202 Accepted` immediately and sends the quiz questions across `/ws/quiz`.
5. **[frontend/src/lib/api.ts](file:///c:/Users/sksum/OneDrive/Documents/Projects/PageLM/frontend/src/lib/api.ts)**: See how the frontend initiates requests and hooks into WebSocket streams.

### Phase 3: Advanced AI & RAG (2–3 Hours)
6. **[backend/src/utils/llm/models/index.ts](file:///c:/Users/sksum/OneDrive/Documents/Projects/PageLM/backend/src/utils/llm/models/index.ts)**: Study the model factory pattern. Notice how `makeModels()` allows switching from Gemini to OpenAI to local Ollama with one environment variable.
7. **[backend/src/lib/parser/upload.ts](file:///c:/Users/sksum/OneDrive/Documents/Projects/PageLM/backend/src/lib/parser/upload.ts)**: See how PDF and Word files are extracted into text and passed to `@langchain/textsplitters`.
8. **[backend/src/services/examlab/generate.ts](file:///c:/Users/sksum/OneDrive/Documents/Projects/PageLM/backend/src/services/examlab/generate.ts)**: Inspect the LangGraph `StateGraph`. Notice how each node (`load` -> `cache` -> `gen` -> `validate` -> `save`) handles one dedicated concern.

---

## 3. Five Safe First Tasks to Build Your Confidence

If you are eager to write code and open your first pull request, choose one of these starter tasks:

### Task 1: Fix Hardcoded WebSocket URLs in Debate (Estimated time: 15 mins)
- **File**: `frontend/src/pages/Debate.tsx`
- **Issue**: Lines 95 and 265 use `new WebSocket("ws://localhost:5000/...")` instead of `new WebSocket(wsURL("..."))`.
- **What to do**: Import `wsURL` from `../lib/api` and replace the hardcoded string.

### Task 2: Add FFmpeg to Backend Dockerfile (Estimated time: 20 mins)
- **File**: `backend/Dockerfile`
- **Issue**: Alpine image does not have `ffmpeg`, which breaks podcasts in Docker.
- **What to do**: Add `RUN apk add --no-cache ffmpeg` to the `runtime` stage.

### Task 3: Fix `Array.isArray` Bug in `ask.ts` (Estimated time: 30 mins)
- **File**: `backend/src/lib/ai/ask.ts`
- **Issue**: Line 338 does `Array.isArray(rag)` on the output of `execDirect`, but `execDirect` returns `{ trace, result, threadId }`.
- **What to do**: Change to `Array.isArray(rag?.result) ? rag.result : []`.

### Task 4: Add `npm test` to GitHub CI (Estimated time: 20 mins)
- **File**: `.github/workflows/ci.yml`
- **Issue**: CI validates builds and audits, but doesn't run `vitest`.
- **What to do**: Add a step `run: npm test` under `validate` job.

### Task 5: Improve Flashcard UI Animation (Estimated time: 45 mins)
- **File**: `frontend/src/pages/Flashcards.tsx`
- **What to do**: Add a smooth CSS 3D perspective flip transition when revealing the answer on the card.

---

## 4. Common Rookie Mistakes to Avoid

1. **Running `npm install` in `backend/`**:
   Remember: root `package.json` manages backend dependencies. There is no `backend/package.json`.
2. **Assuming LLMs Always Return Valid JSON**:
   LLMs occasionally hallucinate trailing commas or markdown code blocks around JSON. Notice how `extractFirstJsonObject()` and `tryParse()` in `backend/src/lib/ai/ask.ts` protect against JSON parsing crashes.
3. **Hardcoding URLs**:
   Never hardcode `http://localhost:5000` in frontend components. Always use `env.backend` or the `wsURL()` utility function from `frontend/src/lib/api.ts`.
4. **Committing `.env` Secrets**:
   Never commit API keys. Your `.env` file should always remain in `.gitignore`.

---

## 5. Congratulations!

You now possess a comprehensive, end-to-end understanding of the PageLM codebase. Refer back to these documentation guides whenever you need to trace an API, inspect a service, or design a new feature.

Happy coding, and welcome to the open-source community! 🚀
