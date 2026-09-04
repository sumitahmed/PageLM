# 04 — Frontend Architecture & Guide

This document is a beginner-friendly, comprehensive guide to the **PageLM frontend**. It explains how the React application is initialized, how routing and layouts work, how state is managed, how components communicate with the backend, and how key user interface components are built.

---

## 1. Application Entry & Bootstrap

When a user visits `http://localhost:5173`, the browser follows this bootstrap sequence:

```text
index.html
   ↓
frontend/src/main.tsx  (Initializes React DOM & React Router)
   ↓
frontend/src/App.tsx   (App Shell: Providers, Sidebar, Outlet, CompanionDock)
   ↓
frontend/src/pages/*   (Rendered inside <Outlet /> based on current URL)
```

### 1.1 `index.html`
The HTML entry point. It loads the browser viewport settings, links Google Fonts (`Schibsted Grotesk`), and includes `<script type="module" src="/src/main.tsx"></script>`.

### 1.2 `main.tsx`
Creates the root DOM node with `ReactDOM.createRoot(document.getElementById("root")!)` and declares the route hierarchy:

```tsx
<BrowserRouter>
  <Routes>
    <Route path="/" element={<App />}>
      <Route index element={<Landing />} />
      <Route path="chat" element={<Chat />} />
      <Route path="quiz" element={<Quiz />} />
      <Route path="tools" element={<Tools />} />
      <Route path="planner" element={<PlannerPage />} />
      <Route path="debate" element={<Debate />} />
      <Route path="cards" element={<FlashCards />} />
      <Route path="exam" element={<ExamLabs />} />
      <Route path="*" element={<NotFound />} />
    </Route>
  </Routes>
</BrowserRouter>
```

### 1.3 `App.tsx`
The top-level application wrapper:
- **`<CompanionProvider>`**: Provides a global React Context holding the currently active study document and controlling the Companion Dock.
- **`<AdaptiveToastProvider theme="dark" />`**: Renders toast alerts across all views.
- **`<Sidebar />`**: Fixed navigation on the left (collapsible on mobile, displays history drawer).
- **`<Outlet />`**: React Router placeholder where the current page component is injected.
- **`<CompanionDock />`**: Persistent floating study companion docked at the bottom right.

---

## 2. Component Directory Structure

The components in `frontend/src/components/` are organized by feature:

```text
frontend/src/components/
├── Chat/                      # Active chat components
│   ├── ActionRow.tsx          # Post-response action buttons (Summarize, Quiz, Podcast)
│   ├── BagDrawer.tsx          # Slide-out drawer showing saved items
│   ├── BagFab.tsx             # Floating Action Button displaying bag count
│   ├── Composer.tsx           # Follow-up question input textarea
│   ├── FlashCards.tsx         # Side panel displaying flashcards from current answer
│   ├── LoadingIndicator.tsx   # Animated loading/thinking indicator
│   ├── MarkdownView.tsx       # Markdown + LaTeX + syntax-highlighted code renderer
│   └── SelectionPopup.tsx     # Text-selection popup ("Add Note", "Ask Doubt")
├── Companion/                 # Global study companion
│   ├── CompanionDock.tsx      # Floating slide-up companion window
│   └── CompanionProvider.tsx  # Context provider for active document state
├── Landing/                   # Home screen components
│   ├── ExploreTopics.tsx      # Pre-built popular study topic pills
│   ├── PromptBox.tsx          # Main textarea with drag-and-drop file upload zone
│   └── PromptRail.tsx         # Decorative gradient rail
├── Quiz/                      # Interactive quiz components
│   ├── QuestionCard.tsx       # Question text, options buttons, hints, and explanations
│   ├── QuizHeader.tsx         # Progress bar, current score, and question index
│   ├── ResultsPanel.tsx       # Final score summary, score emoji, and action buttons
│   ├── ReviewModal.tsx        # Modal reviewing all questions with correct answers
│   └── TopicBar.tsx           # Topic input bar for starting a new quiz
├── Tools/                     # Tools screen sub-components
│   ├── ComingSoon.tsx         # Placeholder for roadmap features
│   ├── PodcastGenerator.tsx   # Topic input, audio preview player, and download button
│   ├── SmartNotes.tsx         # Cornell notes generator and PDF download link
│   └── Transcriber.tsx        # Voice recording orb UI and study guide generator
├── planner/                   # Homework planner components
│   ├── Planner.tsx            # View switcher (Today, List, Mindmap) and task list
│   ├── PlannerMindmap.tsx     # D3 force-directed interactive node graph
│   ├── QuickAdd.tsx           # Text and file input for creating new tasks
│   ├── TodayFocus.tsx         # Today's focus sessions and Pomodoro tracker
│   └── mindmap/               # Physics engine and line anchoring geometry
├── Sidebar.tsx                # Left-side navigation bar with chat history drawer
└── Footer.tsx                 # Application footer
```

---

## 3. Deep Dive into Important Components

### 3.1 `PromptBox` (`components/Landing/PromptBox.tsx`)
- **Purpose**: Main entry prompt input on the landing page, handling text entry, drag-and-drop file staging, and submit triggers.
- **Inputs / Props**:
  - `value: string`, `onChange: (val: string) => void`
  - `onSend: () => void`, `busy: boolean`
  - `onPickFile: () => void`, `onRemoveFile: () => void`
  - `stagedFileName: string | null`
  - `onDragOver`, `onDrop`
- **What renders it**: `pages/Landing.tsx`.

### 3.2 `MarkdownView` (`components/Chat/MarkdownView.tsx`)
- **Purpose**: Renders rich educational output generated by the AI, including tables, blockquotes, LaTeX mathematical equations, and syntax-highlighted code.
- **Inputs / Props**: `md: string` (GitHub-Flavored Markdown string).
- **Underlying Plugins**: `react-markdown`, `remark-gfm`, `remark-breaks`, `remark-math`, `rehype-katex`, `rehype-highlight`.
- **What renders it**: `pages/Chat.tsx`, `components/Companion/CompanionDock.tsx`.

### 3.3 `FlashCards` (`components/Chat/FlashCards.tsx`)
- **Purpose**: Renders the generated flashcards returned alongside an AI answer in a dedicated side-column. Each card can be flipped or added directly to "My Learning Bag".
- **Inputs / Props**:
  - `items: FlashCard[]` (array of `{ q, a, tags }`).
  - `onAdd: (item: { kind: 'flashcard' | 'note'; title: string; content: string }) => void`.
- **State**: Tracks which cards have been added to prevent duplicate additions.
- **What renders it**: `pages/Chat.tsx`.

### 3.4 `CompanionDock` (`components/Companion/CompanionDock.tsx`)
- **Purpose**: A floating study assistant that stays accessible across the application. When a user opens a chat or document, the companion automatically links its context to that material, answering questions grounded exclusively in that document.
- **State**:
  - `input`: Current question typed by the user.
  - `messages`: Conversation history with the companion.
  - `busy`: Thinking state indicator.
  - `error`: Error messages if API fails.
- **API Calls**: Invokes `companionAsk()` in `lib/api.ts` which calls `POST /api/companion/ask`.
- **What renders it**: Rendered globally in `App.tsx`.

### 3.5 `QuestionCard` (`components/Quiz/QuestionCard.tsx`)
- **Purpose**: Displays a single multiple-choice question, its 4 selectable options, a collapsible hint toggle, and the answer explanation once answered.
- **Inputs / Props**:
  - `q: Question` (`{ id, question, options, correct, hint, explanation }`).
  - `selected: number | null`: The option index clicked by the user.
  - `showExp: boolean`: Whether the answer explanation is visible.
  - `showHint: boolean`: Whether the hint is visible.
  - `onSelect: (index: number) => void`.
  - `onHint: () => void`.
  - `onNext: () => void`.
  - `isLast: boolean`.
- **What renders it**: `pages/Quiz.tsx`, `pages/examlab.tsx`.

### 3.6 `PlannerMindmap` (`components/planner/PlannerMindmap.tsx`)
- **Purpose**: Renders an interactive D3 force-directed physics graph of homework tasks and their sub-steps. Users can drag nodes, zoom/pan, click nodes to view study steps, and generate AI materials directly from nodes.
- **Inputs / Props**:
  - `tasks: PlannerTask[]`, `plan: WeeklyPlan | null`
  - `onPlan`, `onAssist`, `onUpdateStatus`, `onUpload`, `onDelete`
- **State**: Stores 2D positions (`positions: Record<string, {x, y}>`), zoom level (`zoom`), pan offset (`pan`), active dragging state (`drag`), and contextual AI steps.
- **What renders it**: `components/planner/Planner.tsx`.

### 3.7 `Transcriber` (`components/Tools/Transcriber.tsx`)
- **Purpose**: Converts voice notes and lecture recordings into text and structured study materials.
- **Key Features**:
  - **Siri-Style Orb Overlay**: When recording live, renders a fullscreen visual orb video (`/voice/orb.mp4`) that pulses dynamically based on real-time microphone volume measured via the Web Audio API (`AudioContext` and `AnalyserNode`).
  - **Study Guide Presentation**: Renders key points, topics, categories, and study questions.
- **API Calls**: Calls `transcribeAudio(file)` in `lib/api.ts` which calls `POST /transcriber`.
- **What renders it**: `pages/Tools.tsx`.

---

## 4. State Management Architecture

PageLM relies on a clean, decoupled state strategy without heavy boilerplate libraries like Redux:

1. **Global Context (`CompanionProvider`)**:
   - Stores `document: CompanionDocument | null` and `open: boolean`.
   - Any page component (such as `Chat.tsx`) can call `setDocument({ id, title, text, filePath })` to sync the current topic to the floating Study Companion.
2. **Local Component State (`useState`, `useRef`)**:
   - Individual views (Quiz, Chat, Debate) manage their own ephemeral state (current question index, input text, streaming tokens, active audio playback).
3. **Persistent Server State (Keyv + SQLite via REST/WS)**:
   - All critical long-term entities (Chats, Messages, Learning Bag Flashcards, Tasks) are persisted on the backend and fetched on demand.

---

## 5. End-to-End User Flow Traces

### Flow 1: Landing Page Prompt to Active Chat
1. User enters *"Explain Black Holes"* on `Landing.tsx` and clicks Send.
2. `Landing.tsx` checks `mode === "Chat"` and calls `chatJSON({ q: "Explain Black Holes" })`.
3. Backend responds with `{ ok: true, chatId: "xyz", stream: "/ws/chat?chatId=xyz" }`.
4. `Landing.tsx` triggers `navigate("/chat?chatId=xyz&q=Explain%20Black%20Holes")`.
5. `Chat.tsx` mounts, connects to WebSocket `/ws/chat?chatId=xyz`.
6. WebSocket receives events:
   - `{ type: "phase", value: "generating" }` → Shows loading indicator.
   - `{ type: "answer", answer: { topic, answer, flashcards } }` → Renders Markdown answer and populates the flashcards side panel.
   - `{ type: "done" }` → Closes loading state.
7. `Chat.tsx` updates `CompanionContext` with the generated answer, arming the Study Companion.

### Flow 2: Taking a Quiz
1. User enters `/quiz?topic=Quantum%20Computing`.
2. `Quiz.tsx` calls `quizStart("Quantum Computing")`.
3. Backend responds with `{ ok: true, quizId: "abc", stream: "/ws/quiz?quizId=abc" }`.
4. `Quiz.tsx` connects to `/ws/quiz?quizId=abc`.
5. WebSocket receives `{ type: "quiz", quiz: [5 Question objects] }`.
6. User clicks an option on `QuestionCard.tsx`.
7. `Quiz.tsx` evaluates answer, increments score, shows explanation, and moves to question index 2 after 350ms.
8. On question 5 completion, `ResultsPanel.tsx` is displayed with final score percentage and review modal options.
