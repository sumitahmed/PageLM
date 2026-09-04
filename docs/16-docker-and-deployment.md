# 16. Containerization & Deployment Guide

This guide covers building, configuring, and deploying PageLM using Docker and Docker Compose. It documents the multi-stage image builds, volume mounts, port bindings, production considerations, and known container configuration fixes.

---

## 1. Container Architecture Overview

PageLM provides container configurations for orchestrating both the frontend and backend as isolated services:

```mermaid
flowchart TD
    Client[Browser / User] -->|Port 5173| FrontCont[Container: pagelm-frontend (serve)]
    Client -->|Port 5000| BackCont[Container: pagelm-backend (Node 22)]
    
    FrontCont -->|API Requests| BackCont
    
    subgraph Host Volumes
        HostStorage[./storage/]
        HostEnv[./.env]
    end

    BackCont -->|Mounts rw| HostStorage
    BackCont -->|Mounts ro| HostEnv
```

---

## 2. Docker Compose Deployment

The fastest way to deploy PageLM in a containerized environment is via `docker-compose.yml`:

### Prerequisites:
Ensure your root `.env` file exists and has your LLM API keys set (e.g., `GEMINI_API_KEY`, `OPENAI_API_KEY`).

### Launch Services:
```bash
# Build and run containers in detached mode
docker-compose up -d --build

# View real-time logs
docker-compose logs -f

# Stop containers
docker-compose down
```

### Services & Port Mappings:
| Container Name | Service | Internal Port | Host Port | Purpose |
| :--- | :--- | :--- | :--- | :--- |
| `pagelm-backend` | `backend` | 5000 | `5000` | Fubelt HTTP & WebSocket server |
| `pagelm-frontend`| `frontend`| 5173 | `5173` | React 19 SPA served via `serve -s` |

---

## 3. Dockerfile Deep Dive

### 3.1 Backend Multi-Stage Build (`backend/Dockerfile`)
Built on `node:22.16.0-alpine`:
1. **`builder` stage**:
   - Copies `package.json` and `package-lock.json`.
   - Runs `npm ci --legacy-peer-deps`.
   - Copies `backend/` source and compiles TypeScript via `npm run build`.
2. **`runtime` stage**:
   - Copies compiled output (`backend/dist/`) and production `node_modules`.
   - Exposes port `5000`.
   - Executes `npm start` (`node backend/dist/src/core/index.js`).

### 3.2 Frontend Multi-Stage Build (`frontend/Dockerfile`)
Built on `node:22.16.0-alpine`:
1. **`deps` stage**:
   - Installs `pnpm@10.13.1`.
   - Runs `pnpm i --frozen-lockfile`.
2. **`build` stage**:
   - Injects build arguments `VITE_BACKEND_URL` and `VITE_TIMEOUT`.
   - Compiles production bundle with `pnpm build` (`vite build`).
3. **`runtime` stage**:
   - Copies compiled `dist/` directory.
   - Installs `serve` globally.
   - Serves static assets on port `5173` (`serve -s dist -l 5173`).

---

## 4. Persistent Volume Management

The backend container mounts the local host directory `./storage` to `/app/storage`:
```yaml
volumes:
  - ./storage:/app/storage
  - ./.env:/app/.env:ro
```
This ensures:
1. SQLite database (`storage/database.sqlite`), user flashcards, and tasks persist across container rebuilds.
2. Cached answers (`storage/cache/`) survive container restarts.
3. Uploaded syllabus files and generated podcast audio files remain accessible on the host machine.

---

## 5. Critical Docker Findings & Fixes for DevOps

> [!CAUTION]
> **Issue 1: Missing FFmpeg in Backend Dockerfile**:
> `backend/Dockerfile` builds on `node:22.16.0-alpine` but **does not install `ffmpeg`**.
> If a user triggers podcast generation inside the Docker container, the process will fail with `Error: ffmpeg_failed`.
> 
> **The Fix**:
> Add `apk add --no-cache ffmpeg` to the `runtime` stage of `backend/Dockerfile`:
> ```dockerfile
> FROM base AS runtime
> RUN apk add --no-cache ffmpeg
> ...
> ```

> [!NOTE]
> **Issue 2: README Discrepancy on `docker-compose.prod.yml`**:
> The root `README.md` references running `docker-compose -f docker-compose.prod.yml up -d`.
> However, only `docker-compose.yml` exists in the repository. Use `docker-compose -f docker-compose.yml up -d` or simply `docker-compose up -d`.

---

## 6. Production Hardening Recommendations

When deploying PageLM to production (e.g., AWS ECS, GCP Cloud Run, or DigitalOcean):
1. **Reverse Proxy & SSL**:
   Place an Nginx or Traefik reverse proxy in front of the application to handle TLS/SSL termination and proxy WebSocket connections (`Upgrade: websocket` headers).
2. **Read-Only Container Filesystems**:
   Mount `/app/storage` as the only writable volume.
3. **ChromaDB Production Cluster**:
   For deployments handling thousands of documents, switch from `DB_MODE=json` to `DB_MODE=chroma` and spin up a dedicated ChromaDB container alongside the backend.
