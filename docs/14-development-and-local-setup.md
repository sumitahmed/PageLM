# 14. Development & Local Setup Guide

This guide walks you through setting up, configuring, running, and troubleshooting PageLM on your local development machine (macOS, Linux, or Windows).

---

## 1. Prerequisites

Before installing PageLM, ensure you have the following installed:

1. **Node.js**: Version `20.x`, `22.x` (LTS), or `24.x`.
   ```bash
   node --version  # Must be >= 20.0.0
   npm --version   # Must be >= 9.0.0
   ```
2. **FFmpeg** *(Required for audio podcast generation)*:
   - **macOS**: `brew install ffmpeg`
   - **Ubuntu/Debian**: `sudo apt-get install ffmpeg`
   - **Windows**: `winget install Gyan.FFmpeg` or `choco install ffmpeg`
   - Verify: `ffmpeg -version`
3. **AI Provider API Key** *(At least one)*:
   - **Google Gemini** *(Recommended, free tier available)*: [Google AI Studio](https://aistudio.google.com/)
   - **OpenAI**: [OpenAI Platform](https://platform.openai.com/)
   - **Ollama** *(Optional, for 100% offline local inference)*: [ollama.com](https://ollama.com/)

---

## 2. Crucial Repository Architecture Note

> [!WARNING]
> **Repository Package Structure Clarification**:
> - The repository root (`./`) contains the **backend dependencies and scripts**.
> - There is **NO `backend/package.json`**. The `backend/` folder contains only TypeScript source code and a `tsconfig.json`.
> - Some older setup scripts (`setup.sh`, `setup.ps1`) instruct users to run `cd backend && npm install`. **Do not do this**—run `npm install` directly in the repository root for the backend!
> - The frontend is located in `frontend/` and has its own isolated `package.json`.

---

## 3. Step-by-Step Installation

### Step 1: Clone the Repository
```bash
git clone https://github.com/sumitahmed/PageLM.git
cd PageLM
```

### Step 2: Install Backend Dependencies (Repository Root)
```bash
npm install
```

### Step 3: Install Frontend Dependencies
```bash
cd frontend
npm install
cd ..
```

---

## 4. Environment Configuration

### Backend Environment (`.env`)
Create a `.env` file in the root directory (or copy `.env.example` if available):

```ini
# Server Configuration
PORT=5000
BASE_URL=http://localhost:5000
FRONTEND_URL=http://localhost:5173

# Database & Vector Storage (json or chroma)
DB_MODE=json

# AI Provider Selection (gemini, openai, claude, grok, ollama, openrouter, minimax)
PROVIDER=gemini
GEMINI_API_KEY=your_gemini_api_key_here
GEMINI_MODEL=gemini-1.5-flash

# Embeddings Provider (defaults to openai; can use openrouter)
EMB_PROVIDER=openai
OPENAI_API_KEY=your_openai_api_key_here
OPENAI_EMBED_MODEL=text-embedding-3-small

# Audio & Speech Configuration
SPEECH_SDK_MODEL=openai/gpt-4o-mini-tts
TRANSCRIPTION_PROVIDER=openai

# Path to FFmpeg executable (if not in system PATH)
# FFMPEG_PATH=/usr/bin/ffmpeg
```

### Frontend Environment (`frontend/.env`)
Create a `.env` file in the `frontend/` directory:

```ini
VITE_BACKEND_URL=http://localhost:5000
VITE_WS_BACKEND_URL=ws://localhost:5000
```

---

## 5. Running the Application Locally

You will run the backend and frontend in separate terminal windows:

### Terminal 1: Start Backend Server
From the repository root:
```bash
npm run dev
# Or to watch files:
npm run dev:backend
```
*The backend server will start on `http://localhost:5000`.*

### Terminal 2: Start Frontend Development Server
From the `frontend/` directory:
```bash
cd frontend
npm run dev
```
*The Vite development server will start on `http://localhost:5173`.*

Open your browser and navigate to **`http://localhost:5173`**.

---

## 6. Available NPM Scripts

### Root Scripts (`package.json` - Backend)
| Script | Command | Purpose |
| :--- | :--- | :--- |
| `npm run dev` | `node -r ts-node/register backend/src/core/index.ts` | Run backend directly via ts-node |
| `npm run build` | `tsc -p backend/tsconfig.json` | Compile TypeScript into `backend/dist/` |
| `npm start` | `node backend/dist/src/core/index.js` | Run compiled production backend |
| `npm test` | `vitest` | Run unit and integration tests |

### Frontend Scripts (`frontend/package.json`)
| Script | Command | Purpose |
| :--- | :--- | :--- |
| `npm run dev` | `vite` | Start Vite dev server with Hot Module Replacement (HMR) |
| `npm run build` | `tsc -b && vite build` | Typecheck and build production bundle into `dist/` |
| `npm run lint` | `eslint .` | Run ESLint checks across TypeScript files |
| `npm run preview` | `vite preview` | Preview production build locally |

---

## 7. Common Setup Pitfalls & Troubleshooting

### 1. `npm ERR! enoent ENOENT: no such file or directory, open '.../backend/package.json'`
- **Cause**: Running `npm install` inside the `backend/` folder.
- **Fix**: Run `npm install` in the **repository root**.

### 2. `ffmpeg_failed` during Podcast Generation
- **Cause**: FFmpeg is either not installed or not discoverable in the system PATH.
- **Fix**: Install FFmpeg (`brew install ffmpeg` or `winget install Gyan.FFmpeg`) and ensure running `ffmpeg -version` in a fresh terminal outputs version information. Alternatively, specify `FFMPEG_PATH=/full/path/to/ffmpeg` in `.env`.

### 3. `WebSocket connection to 'ws://localhost:5000/...' failed`
- **Cause**: Backend server is not running on port 5000 or firewall is blocking WebSocket handshakes.
- **Fix**: Verify Terminal 1 shows `Server running on port 5000`.

### 4. `No valid content extracted from file`
- **Cause**: Uploading a scanned PDF containing only images without embedded text.
- **Fix**: Upload digital PDFs with selectable text, or use Markdown (`.md`) or Word (`.docx`) files.
