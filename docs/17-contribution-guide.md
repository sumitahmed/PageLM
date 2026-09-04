# 17. Contributor & Open-Source Guidelines

Welcome to the PageLM community! We welcome contributions from developers of all skill levels. Whether you are fixing a typo, adding unit tests, fixing an architectural bug, or creating a new educational feature, this guide will help you contribute smoothly.

---

## 1. Development Principles & Code of Conduct

1. **Active Learning First**: Any new feature or prompt change must serve deep comprehension, active recall, and curiosity over passive answers.
2. **Strict Non-Destructive Changes**: Do not refactor core APIs or break existing WebSocket protocols without consensus from maintainers.
3. **Type Safety**: Write clean TypeScript with strict types. Avoid using `any` whenever explicit interfaces can be defined.
4. **Be Kind & Constructive**: We are a welcoming, learning-focused community. Respectful collaboration is mandatory.

---

## 2. Git Branching Strategy

- **`main`**: Production-ready branch. Only tagged releases and hotfixes merge here.
- **`develop`**: Active integration branch. All feature branches and PRs target `develop`.
- **Branch Naming Conventions**:
  - `feature/short-description` (e.g., `feature/add-dark-mode-toggle`)
  - `fix/issue-description` (e.g., `fix/debate-websocket-urls`)
  - `docs/doc-description` (e.g., `docs/add-api-endpoints`)
  - `test/test-scope` (e.g., `test/add-planner-unit-tests`)

---

## 3. Contribution Workflow

### Step 1: Fork & Clone
Fork the repository on GitHub and clone your fork locally:
```bash
git clone https://github.com/YOUR_USERNAME/PageLM.git
cd PageLM
git checkout develop
git checkout -b feature/my-new-feature
```

### Step 2: Install & Verify Clean Baseline
Install dependencies and make sure baseline builds pass:
```bash
# Backend (in root)
npm install
npm run build

# Frontend
cd frontend
npm install
npm run build
cd ..
```

### Step 3: Implement Your Changes
Make focused, modular changes. Ensure you write or update unit tests in `vitest`.

### Step 4: Run Tests & Typechecks
```bash
# Run backend typecheck and tests
npm run build
npm test

# Run frontend typecheck and linter
cd frontend
npm run lint
npm run build
cd ..
```

### Step 5: Commit Your Changes
Use conventional commit prefixes:
- `feat:` (New feature)
- `fix:` (Bug fix)
- `docs:` (Documentation)
- `test:` (Adding or improving tests)
- `refactor:` (Code change that neither fixes a bug nor adds a feature)
- `ci:` (Changes to CI configuration)

Example:
```bash
git commit -m "fix(debate): use wsURL helper instead of hardcoded localhost"
```

### Step 6: Submit a Pull Request
Push your branch to GitHub and open a PR targeting the **`develop`** branch. Fill out all sections of the [Pull Request Template](file:///c:/Users/sksum/OneDrive/Documents/Projects/PageLM/.github/pull_request_template.md).

---

## 4. Top 10 Beginner-Friendly Contribution Opportunities

If you are looking for a high-impact first contribution, here are 10 concrete issues discovered during our architectural audit:

| # | Task | Area | Difficulty | Where to Look |
| :--- | :--- | :--- | :--- | :--- |
| **1** | **Fix Hardcoded WebSocket URLs in Debate** | Frontend | 🟢 Beginner | `frontend/src/pages/Debate.tsx` (Replace `ws://localhost:5000` with `wsURL(...)`) |
| **2** | **Add FFmpeg to Backend Dockerfile** | DevOps | 🟢 Beginner | `backend/Dockerfile` (Add `apk add --no-cache ffmpeg` in runtime stage) |
| **3** | **Fix `Array.isArray(rag)` bug in `ask.ts`** | Backend / AI | 🟡 Intermediate | `backend/src/lib/ai/ask.ts` (Check `rag?.result` instead of `rag`) |
| **4** | **Add `npm test` Step to CI Pipeline** | DevOps / CI | 🟢 Beginner | `.github/workflows/ci.yml` (Add test step before summary) |
| **5** | **Add Unit Tests for Chat & Flashcard APIs** | Testing | 🟡 Intermediate | Create `backend/src/core/routes/__tests__/flashcards.test.ts` |
| **6** | **Remove or Replace Unused `sqlite.ts`** | Backend | 🟢 Beginner | `backend/src/utils/database/sqlite.ts` |
| **7** | **Fix Setup Scripts (`setup.sh` / `setup.ps1`)** | Tooling | 🟢 Beginner | Remove `cd backend && npm install` step; instruct running in root |
| **8** | **Document or Create `docker-compose.prod.yml`** | DevOps | 🟢 Beginner | Add production compose file or update `README.md` reference |
| **9** | **Add Optical Character Recognition (OCR)** | AI / Ingestion | 🔴 Advanced | Integrate Tesseract or PDF OCR in `backend/src/lib/parser/upload.ts` |
| **10**| **Add Dark Mode / Light Mode Theme Toggle** | Frontend UI | 🟡 Intermediate | Add Tailwind color-scheme toggle in `frontend/src/App.tsx` |

---

## 5. Review Checklist for PR Authors

Before requesting a review, verify:
- [ ] Code builds without TypeScript errors (`npm run build` in root and `frontend/`).
- [ ] ESLint passes without warnings (`npm run lint` in `frontend/`).
- [ ] Any new backend endpoint is accompanied by documentation in `docs/06-api-and-routing.md`.
- [ ] No secrets or personal API keys are committed in `.env` files or git history.
