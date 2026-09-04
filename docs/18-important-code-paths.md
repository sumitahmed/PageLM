# 18. Important Code Paths & Developer Cheat Sheet

This document serves as a rapid-lookup directory for developers and contributors. If you want to customize, debug, or extend a specific feature in PageLM, use this cheat sheet to jump directly to the relevant files.

---

## 1. Quick Jump Directory: "Where do I look to change X?"

### Artificial Intelligence & Models
| Goal | Primary File to Edit | Key Function / Symbol |
| :--- | :--- | :--- |
| **Change AI teaching persona or tone** | `backend/src/lib/ai/ask.ts` | `BASE_SYSTEM_PROMPT` |
| **Change default LLM provider or fallback** | `backend/src/utils/llm/models/index.ts` | `makeModels()` & `pick()` |
| **Add a new LLM provider** | `backend/src/utils/llm/models/` | Create `<provider>.ts` implementing `LLM` |
| **Adjust OpenAI/Gemini models or keys** | `backend/src/config/env.ts` | `config.gemini_model`, `config.openai_model` |
| **Modify LangGraph ExamLab logic** | `backend/src/services/examlab/generate.ts` | `StateGraph`, `nGen`, `nValidate` |
| **Clear or inspect AI response cache** | `storage/cache/ask/` & `storage/cache/exam/`| JSON files named by SHA-256 hash |

---

### Backend API & Protocols
| Goal | Primary File to Edit | Key Function / Symbol |
| :--- | :--- | :--- |
| **Add a new HTTP or WebSocket route** | `backend/src/core/router.ts` | `registerRoutes(app)` |
| **Modify CORS or body-parsing middleware** | `backend/src/core/middleware.ts` | `setupMiddleware(app)` |
| **Inspect custom HTTP/WS server implementation** | `backend/src/utils/server/server.js` | `createServer`, `app.ws` |
| **Adjust WebSocket chunk sizes (`emitLarge`)** | `backend/src/utils/chat/ws.ts` | `emitLarge()`, `safeSend()` |
| **Change server listening port** | `backend/src/core/index.ts` | `config.port` |

---

### Document Processing & RAG
| Goal | Primary File to Edit | Key Function / Symbol |
| :--- | :--- | :--- |
| **Add support for new file formats (e.g. EPUB)**| `backend/src/lib/parser/upload.ts` | `extractText()` |
| **Adjust chunk size or overlap for RAG** | `backend/src/lib/ai/embed.ts` | `RecursiveCharacterTextSplitter` (chunkSize: 512) |
| **Change vector database (JSON vs Chroma)** | `backend/src/utils/database/db.ts` | `saveDocuments()`, `getRetriever()` |
| **Customize RAG retrieval tool** | `backend/src/agents/tools/Ragsearch.ts` | `Ragsearch.run()` |

---

### Audio, Podcasts & Speech
| Goal | Primary File to Edit | Key Function / Symbol |
| :--- | :--- | :--- |
| **Change EdgeTTS podcast voice actors** | `backend/src/services/podcast/index.ts` | `makeAudio()` voice assignments |
| **Adjust FFmpeg audio bitrate or codec** | `backend/src/utils/tts/index.ts` | `ff()` spawn arguments (`-b:a 192k`) |
| **Add a Speech SDK TTS provider** | `backend/src/utils/tts/index.ts` | `sdk()` dynamic provider factory |
| **Switch STT transcription provider** | `backend/src/services/transcriber/index.ts` | `transcribeAudio()`, `TranscriptionProvider` |
| **Modify automated study notes from audio** | `backend/src/services/transcriber/index.ts` | `generateStudyMaterials()` |

---

### Multi-Agent Framework
| Goal | Primary File to Edit | Key Function / Symbol |
| :--- | :--- | :--- |
| **Register a new autonomous agent** | `backend/src/agents/agents.ts` | `reg({ id, name, sys, tools })` |
| **Create a new agent tool** | `backend/src/agents/tools/` | Create tool adhering to `ToolIO` |
| **Adjust agent timeout or retry limits** | `backend/src/agents/runtime.ts` | `execDirect()` |
| **Inspect agent session memory** | `backend/src/agents/memory.ts` | `storage/agents/` |

---

### Database & Persistence
| Goal | Primary File to Edit | Key Function / Symbol |
| :--- | :--- | :--- |
| **Change SQLite connection or file location** | `backend/src/utils/database/keyv.ts` | `new SQLite({ uri: 'sqlite://...' })` |
| **Modify chat persistence & history length** | `backend/src/utils/chat/chat.ts` | `mkChat()`, `addMsg()`, `getMsgs()` |
| **Modify task or schedule persistence** | `backend/src/services/planner/store.ts` | `createTask()`, `updateTask()` |
| **Modify debate session persistence** | `backend/src/services/debate/index.ts` | `createDebateSession()` |

---

### Frontend UI & Client State
| Goal | Primary File to Edit | Key Function / Symbol |
| :--- | :--- | :--- |
| **Add a new page or change URL routes** | `frontend/src/App.tsx` | `<Routes>`, `<Route path="..." />` |
| **Modify global navbar or layout** | `frontend/src/components/layout/Navbar.tsx`| Nav links & branding |
| **Customize Feynman Markdown renderer** | `frontend/src/components/MarkdownView.tsx` | Syntax highlighting & KaTeX math |
| **Customize floating study companion dock** | `frontend/src/components/CompanionDock.tsx`| Sliding drawer & grounded ask handler |
| **Modify frontend API or WebSocket client** | `frontend/src/lib/api.ts` | `req()`, `wsURL()`, `connectChatStream()` |
| **Fix hardcoded debate WebSocket URL** | `frontend/src/pages/Debate.tsx` | Line 95 & 265 (`ws://localhost:5000`) |

---

### DevOps & Deployment
| Goal | Primary File to Edit | Key Function / Symbol |
| :--- | :--- | :--- |
| **Add FFmpeg to backend Docker container** | `backend/Dockerfile` | `apk add --no-cache ffmpeg` |
| **Change container ports or volume mounts** | `docker-compose.yml` | `ports`, `volumes` |
| **Add automated test step to GitHub CI** | `.github/workflows/ci.yml` | `jobs.validate.steps` |
