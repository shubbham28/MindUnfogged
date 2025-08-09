
# MindUnfogged – MVP Developer Specification

## 1. Overview
**App Name:** MindUnfogged  
**Purpose:** Journaling + To-Do mobile and web app to capture thoughts, tasks, and goals, with LLM-powered classification, follow-ups, analytics, and offline-first functionality.  
**Platforms:**  
- Android (primary) – Flutter  
- Web (responsive, same layout) – Flutter Web  

## 2. Core Functional Requirements

### 2.1 Entry Capture
- User taps **“New”**, types freely.
- On submission:
  - **Tier 0:** Quick keyword/rule check (offline)
  - **Tier 1:** MobileBERT (TFLite) classification (thought/task/goal) + confidence
  - **Tier 2 (if <0.75 confidence OR text >250 chars):** Gemini via LangChain for classification
- Confirmed type triggers **dynamic LLM follow-up flow**:
  - Thoughts → empathetic tone, emotion inference (0–5 scale per emotion) + user mood rating (0–10)
  - Tasks → neutral tone, urgency/due date questions
  - Goals → neutral tone, ask number of steps, generate breakdown with deadlines by difficulty
- Follow-ups appear **one at a time** in chat-style interface.
- “Skip Questions” option auto-fills defaults.

**Defaults on skip:**
- Thoughts: all emotions = 0
- Tasks: urgency = later, due date = none
- Goals: status = active, steps = empty

### 2.2 Data Schema (Firebase Firestore)
**Collection:** `entries` (one collection for all types)  
**Fields:**
```json
{
  "id": "string",
  "userId": "string",
  "type": "thought" | "task" | "goal",
  "text": "string",
  "tags": ["string"],
  "createdAt": "timestamp",
  "updatedAt": "timestamp",
  "isArchived": "bool",

  "emotions": { "joy": 0, "fear": 0, "sadness": 0, "anger": 0, "disgust": 0 },
  "moodScore": "int",
  "followUps": [{ "question": "string", "answer": "string", "timestamp": "timestamp" }],

  "urgency": "now" | "soon" | "later",
  "dueDate": "timestamp|null",
  "status": "todo" | "doing" | "done",
  "checklist": [{ "title": "string", "done": "bool" }],
  "recurrence": { "type": "string", "interval": "int", "pattern": "string" },

  "targetDate": "timestamp",
  "steps": [{ "title": "string", "estDate": "timestamp", "done": "bool", "notes": "string", "subTasks": [{ "title": "string", "done": "bool" }] }],
  "reviewCadence": "weekly" | "monthly"
}
```

### 2.3 Offline Support
- Offline-first with Firestore caching.
- **TinyLlama (Q4_K_M)** fallback for LLM follow-ups when Gemini not available.
- Show small top banner when in offline mode.
- On-demand TinyLlama download on first use (background).

### 2.4 Journal View
- Hybrid: Chronological feed with type filter.
- Auto-scroll to new entry.
- Delete requires confirmation dialog.
- Semantic search via Firebase-stored embeddings + on-device FAISS.

### 2.5 Calendar View
- Toggle: Week (default) / Month.
- Shows only tasks/goals with color-coded urgency:
  - Red + icon = overdue
  - Orange = due soon
  - Green = on track
- Detailed popup on tap.
- Recurring items:
  - Progress tracked separately per occurrence.
  - Complete → next occurrence appears immediately with original schedule intact.
  - Reminder only for first occurrence unless incomplete.

### 2.6 Insights Tab
- Dynamic, filterable charts:
  - Emotion time-series (smoothed)
  - Pie chart with quick filters: Today, Yesterday, Last Week, Last Month, Last Year, Overall (reset to Today on reopen)
  - Streak tracking
- Weekly review (Monday 9 AM):
  - LLM-generated narrative (analytical + emphatic ending + contextual motivational quote)
  - Top 3 most common tags for week
- Daily digest (default 8 AM, user-settable):
  - All tasks sorted by urgency with overdue tasks grouped at top
  - Yesterday’s overall mood highlight

### 2.7 Notifications
- Daily digest
- Weekly review
- Goal reminders X days before deadline (configurable)
- All push notifications deep-link to relevant tab

## 3. UI/UX

### 3.1 Theme
- **Primary:** Indigo 600 `#3949AB`
- **Secondary:** Soft Lavender `#C5CAE9`
- **Accent:** Emerald Green `#43A047`
- Light background with strong contrast.
- Auto-switch light/dark mode (light default).

### 3.2 Navigation
- Bottom tabs:
  1. Capture
  2. Journal
  3. Calendar
  4. Insights

### 3.3 Capture Flow
- Chat-style follow-up interface with typing effect for LLM responses.
- Auto-tagging: top 3 tags (multi-word up to 3 words), shown above input bar for user edit before save.
- Entities stored in plain text.
- Option to store raw conversation in JSON if per-entry consent given.

### 3.4 Icons & Branding
- App icon: “MU” + reserved space for later symbolic element.
- Fixed Material color set.

## 4. Architecture

### 4.1 Frontend
- Flutter for Android + Web (single codebase).
- State management: Riverpod or Bloc.
- Charts: `fl_chart`.
- Local ML: TFLite (MobileBERT), llama.cpp bindings (TinyLlama).

### 4.2 Backend
- Firebase Auth (Google Sign-In).
- Firebase Firestore (offline sync).
- Firebase Storage (raw LLM chat JSON/XML if saved).
- Firebase Functions (scheduled tasks: weekly review, daily digest).

### 4.3 LLM Orchestration
- LangChain central chain with type-specific tuned nodes.
- Node 1: Classification
- Node 2: Follow-up Q generation
- Node 3: Tagging & emotion scoring
- Primary model: Gemini
- Offline model: TinyLlama Q4_K_M

## 5. Data Handling & Security
- Firebase server-side encryption at rest.
- Firestore Security Rules:
  - Auth required for all reads/writes.
  - `request.auth.uid == resource.data.userId`.
  - Raw LLM chat saved only if `consent = true`.
- Indexed fields: `type`, `createdAt`, `dueDate`, `tags`.

## 6. Error Handling
- LLM fail → fallback to defaults.
- Model download fail → retry with exponential backoff.
- Offline edits: last edit wins.
- UI error banners for:
  - Network lost
  - LLM unavailable
  - Sync failed

## 7. Testing Plan

### 7.1 Unit Tests
- Entry classification accuracy
- Emotion inference correctness
- Tag extraction length & format

### 7.2 Integration Tests
- End-to-end capture flow (online/offline)
- Notifications scheduling
- Calendar view toggle

### 7.3 UI Tests
- Tab navigation
- Chart filtering
- Tag editing before save

### 7.4 Performance Tests
- Model inference latency (Gemini vs TinyLlama)
- Offline-to-online sync speed

## 8. Deployment Plan
- Firebase project setup
- Flutter build for Android & Web
- Staged rollout via Play Store Internal Testing
- Collect feedback → fix → open beta
