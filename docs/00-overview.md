# 00 — PageLM Overview

Welcome to **PageLM**! This document provides a high-level, beginner-friendly introduction to the PageLM platform: what it is, why it exists, how it works, and what makes it distinct from general-purpose AI chatbots.

---

## 1. What is PageLM?

**PageLM** is an open-source, AI-powered education platform designed to transform raw study materials—such as lecture slides, PDFs, notes, audio recordings, and syllabus topics—into **structured, interactive learning resources**.

Inspired by Google's NotebookLM, PageLM is built specifically for **active learning**. Rather than merely having an open-ended conversation with an AI, PageLM actively structures information to help students and researchers retain knowledge through proven pedagogical techniques like the **Feynman Technique**, **Cornell Note-taking**, **Spaced Repetition**, and **Active Recall**.

### The Core Problem PageLM Solves
When students study today, they face a common set of hurdles:
- **Information Overload**: Hundreds of pages of dense PDFs and lecture recordings that are difficult to synthesize.
- **Passive vs. Active Learning**: Passively reading notes or re-watching lectures leads to the *illusion of competence*—learners feel like they understand the material, but struggle to recall or apply it during exams.
- **Fragmented Tooling**: Students jump between separate apps for flashcards (e.g., Anki), note-taking (e.g., Notion), quiz generation, lecture transcription, and homework scheduling.
- **Privacy & Vendor Lock-In**: Proprietary tools process private study documents on remote corporate servers and often charge recurring subscription fees.

### PageLM's Solution
PageLM provides a single, unified, self-hostable workspace where:
1. You upload documents or enter study topics.
2. The AI grounds its responses strictly in your source materials (via **RAG** — Retrieval-Augmented Generation).
3. The platform generates actionable study tools: Cornell-style PDF notes, interactive quizzes, flashcards, audio podcasts, mock exam simulations, structured homework plans, and AI debates.
4. You can run it completely locally using **Ollama** or connect to major cloud LLM providers (Gemini, OpenAI, Claude, Grok, OpenRouter, MiniMax).

---

## 2. Who is PageLM For?

- **Students**: Convert course syllabi, textbooks, and lecture recordings into quizzes, audio podcasts, and Cornell notes.
- **Educators & Tutors**: Generate question banks, exam rubrics, and structured study guides from lesson plans.
- **Self-Directed Learners & Researchers**: Break down complex technical papers into intuitive analogies, first-principles explanations, and review flashcards.
- **Developers & AI Enthusiasts**: A full-stack reference implementation demonstrating how to build production AI applications combining **React 19**, **Node.js**, **LangChain**, **LangGraph**, and **WebSockets**.

---

## 3. High-Level Architecture Diagram

Here is how a user request travels through the PageLM ecosystem:

```text
       ┌────────────────────────────────────────────────────────┐
       │                        User                            │
       └──────────────────────────┬─────────────────────────────┘
                                  │ Interacts with UI
                                  ▼
       ┌────────────────────────────────────────────────────────┐
       │                   Frontend (React 19)                  │
       │  Vite · Tailwind CSS v4 · React Router · CogniCatch UI  │
       └──────────────┬───────────────────────────▲─────────────┘
                      │                           │
          HTTP REST / │ Multipart                │ WebSocket Events
          File Upload │                           │ (Streaming & Progress)
                      ▼                           │
       ┌──────────────────────────────────────────┴─────────────┐
       │             Backend Server (Custom Fubelt HTTP)         │
       │   Node.js HTTP Server · Native WebSocket Router (ws)   │
       └──────────────┬───────────────────────────▲─────────────┘
                      │ Dispatches to             │ Results & Audio URLs
                      ▼                           │
       ┌──────────────────────────────────────────┴─────────────┐
       │                     Services Layer                     │
       │  Chat · Quiz · SmartNotes · Podcast · ExamLab · Planner │
       │  Debate · Voice Transcriber · Study Companion          │
       └──────────────┬───────────────────────────▲─────────────┘
                      │ Uses                      │ Retrieved Context
                      ▼                           │
 ┌──────────────────────────────────────────────────────────────────────────┐
 │                AI, Persistence & External Integrations                   │
 │                                                                          │
 │  ┌──────────────────────┐  ┌──────────────────┐  ┌───────────────────┐  │
 │  │      AI / LLMs       │  │   Vector / RAG   │  │    Persistence    │  │
 │  │ Gemini · OpenAI      │  │ MemoryVectorStore│  │ Keyv + SQLite     │  │
 │  │ Claude · Grok        │  │ Chroma (optional)│  │ Filesystem JSON   │  │
 │  │ Ollama · MiniMax     │  │ Recursive Split  │  │ Audio / PDF Cache │  │
 │  └──────────────────────┘  └──────────────────┘  └───────────────────┘  │
 │  ┌──────────────────────┐  ┌──────────────────┐  ┌───────────────────┐  │
 │  │     Audio & TTS      │  │  Document Parsers│  │   Agent System    │  │
 │  │ EdgeTTS · ElevenLabs │  │ pdf-parse        │  │ Researcher        │  │
 │  │ Google TTS · SpeechSDK│ │ mammoth (DOCX)   │  │ Tutor · Examiner  │  │
 │  │ Whisper STT · FFmpeg │  │ marked (Markdown)│  │ Podcaster Runtime │  │
 │  └──────────────────────┘  └──────────────────┘  └───────────────────┘  │
 └──────────────────────────────────────────────────────────────────────────┘
```

### Explaining Every Layer in Plain English:

1. **User**: The student or educator who interacts with the application through their browser (typing prompts, uploading PDFs/audio, taking quizzes, listening to podcasts).
2. **Frontend**: A single-page application built with **React 19**, **Vite**, and **Tailwind CSS v4**. It handles UI state, renders Markdown and LaTeX formulas, visualizes D3 mindmaps, and maintains real-time WebSocket connections with the backend.
3. **API / WebSocket**: The communication bridge. Standard HTTP endpoints receive file uploads and initial requests; WebSockets stream progress updates, text tokens, audio notifications, and generated data back to the browser in real time.
4. **Backend Server**: A lightweight, custom Node.js server engine called **Fubelt** (from `CaviraOSS/Fubelt`). It replaces Express to provide fast route matching, built-in JSON body parsing, static file delivery, and WebSocket connection handling with zero bloated dependencies.
5. **Services Layer**: The business logic of the application. Each feature (e.g., `services/quiz`, `services/podcast`, `services/examlab`, `services/planner`) is isolated into its own dedicated module.
6. **AI / RAG / Database / Providers**:
   - **LLM Layer**: Unified abstraction allowing plug-and-play swapping between Gemini, OpenAI, Claude, Grok, Ollama, OpenRouter, and MiniMax.
   - **RAG & Embeddings**: Splits documents into chunks, calculates semantic embeddings, and retrieves relevant passages to ground AI answers.
   - **Persistence**: Keyv backed by SQLite (`storage/database.sqlite`) stores chats, flashcards, planner tasks, and debate histories; JSON and media directories store PDFs, audio clips, and cache files.
   - **Audio & TTS**: Generates multi-speaker podcast audio using Microsoft Edge TTS, ElevenLabs, Google Cloud TTS, or Speech SDK, then stitches the dialogue using FFmpeg.
   - **Agent System**: Specialized AI agents (`tutor`, `researcher`, `examiner`, `podcaster`) that execute multi-step plans with automated retries and timeouts.

---

## 4. What Makes PageLM Different from a Simple Chatbot?

| Capability | Generic Chatbot (ChatGPT, Claude web) | PageLM |
| :--- | :--- | :--- |
| **Focus** | General-purpose conversation | **Education & active learning workflows** |
| **Output Formats** | Raw streaming text | **Interactive quizzes, flashcard decks, Cornell notes PDF, multi-speaker audio podcasts, exam simulations** |
| **Document Grounding** | Ephemeral file chat | **Persistent collections, chunked RAG indexing, source-grounded companion dock** |
| **Pedagogical Prompting** | Basic helpful assistant | **Built-in anti-rote learning mandate, Feynman technique, cognitive load optimization, socratic questioning** |
| **Audio Capabilities** | Text output or single voice readout | **Script generation for 2 alternating speakers stitched into downloadable MP3 podcasts** |
| **Offline / Self-Hosting** | Cloud-only, proprietary | **100% open-source; runs locally with Ollama and Edge-TTS without paid API keys** |
| **Task & Schedule Planning**| Simple to-do list suggestions | **Full homework planner with AI step decomposition, Pomodoro session tracking, and interactive D3 force mindmap** |

---

## 5. Major Features Summary

1. **Contextual Chat**: Ask questions grounded in uploaded documents (PDF, DOCX, MD, TXT). Answers include GitHub-flavored markdown, LaTeX math formulas, and extracted flashcards.
2. **SmartNotes**: Generates structured Cornell notes and compiles them into beautifully formatted, downloadable PDF study sheets.
3. **Interactive Quizzes**: Generates 5-question multiple-choice quizzes complete with real-time scoring, hints, explanations, and review modals.
4. **My Learning Bag (Flashcards)**: Save flashcards and notes from any chat or document for spaced-repetition study.
5. **AI Podcast**: Turns study topics or documents into realistic, two-speaker audio conversations with preview and MP3 download.
6. **Voice Transcriber**: Record audio directly in the browser with a dynamic visual orb UI or upload audio/video files to generate searchable transcripts and study guides.
7. **Homework Planner**: Ingest assignment text or files, break them into actionable sub-steps, plan weekly study slots, track deadlines, and view an interactive D3 mindmap.
8. **ExamLab**: Multi-section exam simulator powered by a LangGraph state machine, supporting standardized exam formats like GRE, GMAT, IELTS, JEE, and SAT.
9. **Debate**: Face off against AI in competitive debate rounds with formal concessions and post-debate judicial scoring.
10. **Study Companion**: A persistent, context-aware side dock that links to whatever document or chat you are currently studying.

---

## 6. How to Read This Documentation

If you are a beginner, follow this recommended path through `docs/`:
1. **[01-project-structure.md](./01-project-structure.md)** — Understand where every file lives.
2. **[02-tech-stack.md](./02-tech-stack.md)** — Learn about the libraries and tools used.
3. **[03-architecture.md](./03-architecture.md)** — See how the entire system fits together.
4. **[14-development-and-local-setup.md](./14-development-and-local-setup.md)** — Set up and run the app on your computer.
5. **[19-beginner-codebase-guide.md](./19-beginner-codebase-guide.md)** — Tips for making your first contribution safely.
