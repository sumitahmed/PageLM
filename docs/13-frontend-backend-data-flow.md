# 13. Frontend-to-Backend Data Flow & Tracing

This document provides visual traces and step-by-step lifecycles for the core data flows in PageLM. It shows how user interactions in the React UI propagate through network calls, server middleware, AI agents, file persistence, and back to the client interface.

---

## 1. Flow 1: Chat Query with Document Upload

```mermaid
sequenceDiagram
    autonumber
    actor User
    participant UI as Chat.tsx (React 19)
    participant API as api.ts (chatAskOnce)
    participant Route as chat.ts (POST /chat)
    participant Parser as upload.ts (Busboy + PDF/Word)
    participant DB as Vector Store (db.ts)
    participant WS as /ws/chat Socket
    participant Agent as Researcher Agent (execDirect)
    participant LLM as LLM Engine (ask.ts)
    participant SQLite as Keyv SQLite

    User->>UI: Enters query & attaches syllabus.pdf
    UI->>API: chatAskOnce({ q, files: [file] })
    API->>Route: POST /chat (multipart/form-data)
    
    Route->>SQLite: mkChat(title) -> create session ID
    Route-->>API: 202 Accepted { chatId, stream: "/ws/chat?chatId=..." }
    API->>WS: Connect WebSocket to /ws/chat?chatId=...
    WS-->>API: { type: "ready", chatId }

    Route->>WS: emitToAll({ type: "phase", value: "upload_start" })
    Route->>Parser: parseMultipart & stream file to storage/uploads/
    Parser->>DB: saveDocuments(namespace="chat:id", chunks)
    Route->>WS: emitToAll({ type: "phase", value: "upload_done" })

    Route->>SQLite: addMsg(role="user", content=q)
    Route->>WS: emitToAll({ type: "phase", value: "generating" })
    
    Route->>Agent: execDirect("researcher", plan: [rag.search])
    Agent->>DB: Query vector retriever
    DB-->>Agent: Top-k passages
    Agent-->>LLM: Context + Question + System Prompt
    LLM-->>Route: JSON { topic, answer, flashcards }
    
    Route->>SQLite: addMsg(role="assistant", content=answer)
    Route->>WS: emitToAll({ type: "answer", answer })
    Route->>WS: emitToAll({ type: "done" })
    
    WS-->>API: Stream events received
    API-->>UI: Update chat state, render markdown & flashcard deck
```

---

## 2. Flow 2: Audio Podcast Generation

```mermaid
sequenceDiagram
    autonumber
    actor User
    participant UI as Podcast.tsx
    participant API as api.ts
    participant Route as podcast.ts (POST /podcast)
    participant Service as services/podcast
    participant LLM as LLM Factory
    participant TTS as EdgeTTS / Speech SDK
    participant FFmpeg as FFmpeg Binary
    participant WS as /ws/podcast

    User->>UI: Submits topic: "Quantum Computing"
    UI->>API: startPodcast(topic)
    API->>Route: POST /podcast { topic }
    Route-->>API: 202 Accepted { pid, stream: "/ws/podcast?pid=..." }
    API->>WS: Connect WebSocket to /ws/podcast?pid=...

    Route->>Service: makeScript(topic)
    Service->>LLM: Generate two-host conversational script
    LLM-->>Service: Script JSON [{ speaker, text }]
    Service->>WS: emitToAll({ type: "script", data: script })

    Route->>Service: makeAudio(script, dir, base)
    loop For each dialogue turn
        Service->>TTS: Generate speech MP3 clip
        TTS-->>Service: Segment saved to disk
        Service->>WS: emitToAll({ type: "progress", segment, total })
    end

    Service->>FFmpeg: Concatenate audio clips (list.txt -> out.mp3)
    FFmpeg-->>Service: Render complete
    Service->>WS: emitToAll({ type: "audio", file: downloadUrl })
    Service->>WS: emitToAll({ type: "done" })

    WS-->>UI: Audio ready event
    UI->>UI: Load audio player, enable scrubbing & download
```

---

## 3. Flow 3: LangGraph ExamLab Generation & Chunked Delivery

```mermaid
sequenceDiagram
    autonumber
    actor User
    participant UI as ExamLab.tsx
    participant Route as examlab.ts (POST /exam)
    participant Graph as LangGraph (generate.ts)
    participant Disk as storage/cache/exam
    participant WS as /ws/exams (emitLarge)

    User->>UI: Clicks "Start GRE Quantitative Mini Test"
    UI->>Route: POST /exam { examId: "gre-quant-mini" }
    Route-->>UI: 202 Accepted { runId, stream: "/ws/exams?runId=..." }
    UI->>WS: Connect WebSocket

    Route->>Graph: handleExam(examId)
    Graph->>Graph: Node 1: load (Exam blueprint)
    Graph->>Disk: Node 2: cache (Check SHA-256)
    alt Cache Hit
        Disk-->>Graph: Return cached payload
    else Cache Miss
        Graph->>Graph: Node 3: gen (Generate multi-section questions)
        Graph->>Graph: Node 4: validate (Verify 4 options, valid hints)
        Graph->>Disk: Node 5: save (Write to disk cache)
    end
    Graph-->>Route: Validated ExamPayload

    Route->>WS: emitLarge("exam", payload, chunkBytes=128KB)
    loop Each 128KB chunk
        WS-->>UI: { type: "exam.chunk", idx, total, data }
    end
    WS-->>UI: { type: "exam.done", total }
    WS-->>UI: { type: "done" }

    UI->>UI: Reassemble chunks, mount timer, render QuestionCard
```

---

## 4. Flow 4: Syllabus Ingestion & Study Plan Generation

```mermaid
sequenceDiagram
    autonumber
    actor User
    participant UI as Planner.tsx
    participant Route as planner.ts (POST /tasks)
    participant Svc as plannerService
    participant AI as planner/ai.ts
    participant Scheduler as planner/scheduler.ts
    participant Store as planner/store.ts
    participant Room as /ws/planner (Room: default)

    User->>UI: Pastes course syllabus text
    UI->>Route: POST /tasks/ingest { text }
    Route->>Svc: createTaskFromRequest({ text })
    Svc->>AI: parseTask(text) -> extract title, course, dueAt, estMins
    Svc->>AI: generateSteps(task) -> extract learning milestones
    Svc->>Store: createTask(taskData) -> persist to SQLite
    Svc->>Scheduler: planTask(task, policy) -> calculate slots & energy
    Svc->>Room: emitToAll({ type: "task.created", task })
    Svc->>Room: emitToAll({ type: "plan.update", taskId, slots })
    Route-->>UI: 200 OK { ok: true, task }

    Room-->>UI: Live WebSocket events received
    UI->>UI: Update Kanban board, Calendar view, and upcoming deadlines
```

---

## 5. Flow 5: Speech-to-Text Transcription & Study Guide Extraction

```mermaid
sequenceDiagram
    autonumber
    actor User
    participant UI as Transcriber.tsx (MediaRecorder)
    participant Route as transcriber.ts (POST /transcriber)
    participant STT as Transcriber Service (Whisper)
    participant LLM as LLM Study Material Generator

    User->>UI: Records lecture audio via browser microphone
    UI->>Route: POST /transcriber (multipart/form-data: audio.webm)
    Route->>STT: transcribeAudio(tempFile, provider="openai")
    STT->>STT: Send audio stream to Whisper API
    STT-->>Route: Transcribed text string
    
    opt Text length > 50 characters
        Route->>LLM: generateStudyMaterials(transcribedText)
        LLM-->>Route: StudyMaterials { summary, keyPoints, vocabulary, questions }
    end

    Route->>Route: fs.unlinkSync(tempFile) -> Clean up temp disk
    Route-->>UI: 200 OK { ok: true, transcription, studyMaterials }
    UI->>UI: Display synchronized transcript, vocabulary cards, and quiz
```

---

## 6. Summary of Architectural Guarantees

1. **Non-Blocking Main Loop**: Heavy audio encoding, document embedding, and LLM completions are offloaded or executed asynchronously, keeping the HTTP event loop responsive.
2. **Deterministic Re-assembly**: Frontend streaming listeners reassemble chunked payloads safely using unique sequence indices (`idx` and `total`).
3. **Automatic Resource Cleanup**: Temporary audio files and unlinked uploads are systematically deleted after processing to avoid disk bloat.
