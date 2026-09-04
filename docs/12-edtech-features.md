# 12. Educational Technology & Pedagogy Feature Catalog

PageLM is specifically architected as an **Active Learning Educational Engine**, not a passive chatbot. This guide provides a detailed tour of each educational feature, explaining the pedagogical theory, user workflow, backend service, and UI components that power it.

---

## 1. Feature Map & Learning Methodologies

| Feature | Pedagogical Foundation | Core User Action | Output Artifact |
| :--- | :--- | :--- | :--- |
| **Active Feynman Chat** | Feynman Technique, Anti-Rote learning, Progressive Disclosure | Ask conceptual questions or upload course materials | Structured markdown explanation + auto-generated flashcards |
| **SmartNotes** | Dual Coding Theory, Information chunking | Input topic, raw scribbles, or slide decks | Downloadable high-structure `.md` study summary |
| **Cognitive Flashcards** | Spaced Repetition, Elaborative Interrogation | Review questions tagged with cognitive metadata | 3D flip card practice UI |
| **Interactive Quizzes** | Retrieval Practice & Generation Effect | Request quiz on any academic topic | 4-option MCQs with hints & immediate feedback |
| **Study Podcasts** | Audio-Verbal Learning, Socratic Dialogue | Enter topic or study guide | Multi-minute MP3 dialogue between two co-hosts |
| **Audio Transcriber** | Multimodal Processing, Cognitive Scaffolding | Upload lecture recording or speak into mic | Transcript + automated study guide & vocabulary |
| **Intelligent Planner** | Executive Function Support, Energy-Aware Scheduling | Ingest syllabus text or assignments | Weekly balanced schedule, break alerts, daily digests |
| **ExamLab** | Simulation Training, Timed Stress Exposure | Select standardized exam blueprint | Full-length timed exam environment with scoring |
| **Socratic Debate** | Dialectical Reasoning & Critical Thinking | Take a stance "for" or "against" a thesis | Turn-by-turn debate against AI opponent + rubric score |
| **Study Companion** | Grounded Context Assistance | Open floating dock while reading any material | Strict hallucination-free document Q&A |

---

## 2. Feature Deep Dives

### 2.1 Active Feynman Chat (`frontend/src/pages/Chat.tsx`)
- **Philosophy**: Answering a question is not enough; the learner must develop intuitive understanding.
- **How It Works**:
  - The user enters a question or uploads a lecture PDF.
  - The backend retrieves relevant passages (`researcher` agent + `rag.search`).
  - The model generates an explanation using analogies, mental models, and real-world failure stories.
  - Flashcards are automatically generated from the conversation turn and saved to the user's collection.

### 2.2 SmartNotes Generator (`frontend/src/pages/SmartNotes.tsx`)
- **Philosophy**: Condenses high-entropy academic textbooks into digestible, structured reference notes.
- **Inputs**: Topic string, raw scribbled notes, or uploaded lecture slide decks.
- **Output**: Generates a GitHub-flavored Markdown file stored in `storage/smartnotes/uuid.md` featuring:
  - Executive overview.
  - Core mechanisms & formulas explained intuitively.
  - Common cognitive pitfalls and edge cases.
  - Self-assessment checkpoints.

### 2.3 Cognitive Flashcard Studio (`frontend/src/pages/Flashcards.tsx`)
- **Philosophy**: Active recall and spaced repetition strengthen synaptic retention far more than re-reading.
- **Enhanced Metadata Tags**:
  - `cognitive_load`: Lowers mental friction for complex abstractions.
  - `transfer`: Connects concepts across unrelated domains.
  - `metacognition`: Exercises awareness of one's own understanding.
  - `anti_rote`: Requires reasoning from first principles rather than memorizing terms.
- **UI Experience**: Flip animations, difficulty tagging, and quick deletion.

### 2.4 Diagnostic Quizzes (`frontend/src/pages/Quiz.tsx`)
- **Philosophy**: Testing as a learning event (The Generation Effect).
- **Format**:
  - Multiple-choice questions with exactly 4 options.
  - Progressive hints that guide thinking without giving away answers.
  - Detailed post-answer explanations for both correct and incorrect choices.

### 2.5 Audio Podcasts (`frontend/src/pages/Podcast.tsx`)
- **Philosophy**: Enables passive and commute learning through engaging conversational dialogue.
- **Pipeline**:
  1. Script generator writes a two-character conversation (Host 1: curious interviewer; Host 2: domain expert).
  2. TTS engine synthesizes individual voice tracks using distinct neural voices (`en-US-GuyNeural` and `en-US-JennyNeural`).
  3. FFmpeg concatenates the segments into a unified, high-bitrate MP3 (`storage/podcasts/:pid/podcast.mp3`).
  4. Audio player UI provides scrubbing, playback rate control, and instant download.

### 2.6 Audio Transcriber & Study Generator (`frontend/src/components/Transcriber.tsx`)
- **Philosophy**: Eliminates transcription fatigue so students can focus on comprehension during live lectures.
- **Features**:
  - Direct microphone recording via MediaRecorder API or audio file upload.
  - Providers: OpenAI Whisper, Google Speech, AssemblyAI.
  - Automated study kit extraction (main concepts, definitions glossary, test questions).

### 2.7 AI Study Planner (`frontend/src/pages/Planner.tsx`)
- **Philosophy**: Students frequently fail not from a lack of intellect, but from executive function overload and poor time estimation.
- **Capabilities**:
  - **Syllabus Ingestion**: Accepts raw assignment text or syllabus files and parses out course names, due dates, and estimated hours.
  - **Slot Decomposition**: Automatically breaks large assignments into 30–60 minute focused study blocks.
  - **Energy-Aware Scheduling**: Distributes heavy conceptual work to peak energy hours and review sessions to lighter slots.
  - **Proactive Wellness Alerts**: Background daemon notifies students to take eye breaks and delivers morning digests and evening reviews.

### 2.8 ExamLab Simulation Environment (`frontend/src/pages/ExamLab.tsx`)
- **Philosophy**: True mastery requires executing under realistic test constraints.
- **Features**:
  - LangGraph stateful validation pipeline ensures 100% schema integrity.
  - Timed sections with countdown clocks.
  - Flagging questions for review.
  - Detailed diagnostic scorecard at conclusion.

### 2.9 Socratic AI Debate Arena (`frontend/src/pages/Debate.tsx`)
- **Philosophy**: The highest tier of Bloom's Taxonomy is evaluation and synthesis; defending an argument against counter-evidence exposes shallow understanding.
- **Interaction**:
  - Student picks a controversial or academic topic and selects their stance (`for` or `against`).
  - Opponent AI presents rigorous counter-arguments.
  - If the student presents undeniable logical arguments, the AI concedes gracefully (`ai_concede`).
  - Final adjudication rubric scores both participants on logic, factual grounding, and clarity.

### 2.10 Floating Study Companion Dock (`frontend/src/components/CompanionDock.tsx`)
- **Philosophy**: Ambient assistance without context switching.
- **Implementation**:
  - Persistent sliding drawer available on any page.
  - Reads active document text with a 1.5 MB safety guardrail.
  - Grounded prompt forces the LLM to admit when the text does not contain the answer, guaranteeing zero hallucinations.

---

## 3. Pedagogical Summary for Open-Source Contributors

When designing or improving any feature in PageLM, evaluate it against the **Three Active Learning Questions**:
1. *Does this encourage the student to think, or is it merely giving them the answer?*
2. *Does this test conceptual understanding or rote recall?*
3. *Can the student immediately apply, test, or verify this knowledge?*
