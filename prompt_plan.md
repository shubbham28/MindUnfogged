# MindUnfogged — MVP Build Blueprint & Prompt Plan

This document contains:
- A crisp blueprint for the MVP (decisions and scope).
- A three‑pass breakdown (milestones → epics → right‑sized tasks).
- A **sequenced set of prompts** for a code‑generation LLM to implement the MVP **incrementally** and **test‑first**.
- Each prompt is separated, tagged as `text`, and builds directly on the previous one. No orphaned code.

---

## Architecture Snapshot (locked for MVP)
- **Frontend:** Flutter (Android + Web), Riverpod for state, `go_router` for routing, `freezed` + `json_serializable` for models, `fl_chart` for charts, `logger` for observability.
- **Backend:** Firebase Auth (Google), Firestore (with offline persistence), Firebase Functions (TypeScript) for Gemini+LangChain orchestration, Cloud Scheduler for weekly/daily narratives.
- **LLM Flow:**
  - Tier 0: local keyword/rule filter.
  - Tier 1: **MobileBERT (TFLite)** classifier on-device.
  - Tier 2: **Gemini via Functions** (LangChain chain) when confidence < 0.75 or text length > 250 chars.
  - Offline fallback: defaults + simple rules; TinyLlama interface stubbed behind a feature flag for later.
- **Search:** Server-generated embeddings (Functions) → Firestore; frontend search calls server function. Offline: keyword fallback.
- **Notifications:** Firebase Messaging + local notifications; deep-links into screens.
- **Testing & CI:** Flutter unit/widget/golden tests, integration tests via emulator; Firestore rules tests; Functions unit tests; GitHub Actions.

---

## Three-Pass Breakdown

### Pass 1 — Milestones (coarse)
1. **Scaffold & CI**
2. **Auth + Rules**
3. **Data Model + Repository Layer**
4. **Capture Flow v1 (Tier 0)**
5. **Local ML (Tier 1)**
6. **LLM Orchestration (Tier 2)**
7. **Journal + Calendar**
8. **Insights + Schedulers**
9. **Offline, Search, Notifications + E2E Polish**

### Pass 2 — Epics per Milestone (medium)
- See `todo.md` for acceptance criteria and checklists.

### Pass 3 — Right-Sized Tasks (granular, testable)
- Each prompt below corresponds to a compact, test-driven increment.

---

# Prompts for a Code-Generation LLM
> Use these **in order**. Each returns a unified diff patch and tests that pass locally with emulators. Keep commits small and green.

## Prompt 01 — Repo Scaffold, Themes, Routing
```text
You are a senior Flutter engineer. Output must be correct, minimal, production-grade, readable, and tested. Follow SOLID, idiomatic Dart, and Riverpod patterns.

TASK:
1) Create a new Flutter app named "mind_unfogged".
2) Add packages: flutter_riverpod, go_router, freezed_annotation, json_annotation, build_runner, freezed, json_serializable, fl_chart, logger.
3) Implement:
   - App theme (Indigo 600 #3949AB primary, Lavender #C5CAE9 secondary, Emerald #43A047 accent), light/dark.
   - go_router with 4 tabs: /capture, /journal, /calendar, /insights, bottom navigation.
   - Placeholder screens with unique Keys for tests.

TESTS:
- Widget test: tapping bottom nav items navigates to the correct screen (assert unique Keys).
- Golden test: AppBar matches golden for light and dark.

OUTPUT FORMAT:
- Return a unified diff patch against a clean `flutter create mind_unfogged` scaffold.
- Include all new files (paths), pubspec updates, and tests.
- Ensure `flutter test` passes after your patch.
```

## Prompt 02 — Riverpod Bootstrap + Logger
```text
ROLE: Senior Flutter engineer.

CONTEXT: Codebase from Prompt 01.

TASK:
1) Add a simple Logger wrapper service and Riverpod Provider for it.
2) Add an AppBootstrap Provider that sets up Logger and exposes app start timing metrics.
3) Wire the bootstrap into main.dart so the app won’t render routes until bootstrap completes (fast future).

TESTS:
- Unit test for AppBootstrap provider initializes within 200ms (fake clock).
- Widget test ensures a minimal splash (CircularProgressIndicator) appears during bootstrap, then routes render.

OUTPUT: unified diff patch.
```

## Prompt 03 — Firebase Init + Emulators + CI
```text
ROLE: Senior Full-Stack engineer (Flutter + Firebase).

TASK:
1) Add Firebase to Android and Web targets (manual config placeholders ok).
2) Add firebase_core to pubspec; create FirebaseInit service and Provider.
3) Add `firebase.json`, `firebaserc`, emulator config (auth, firestore, functions), and a `scripts/dev.sh` to run `firebase emulators:start` and `flutter test`.
4) Create `.github/workflows/flutter.yml`:
   - Setup Flutter stable, cache, `flutter test`.
   - (For now) skip real Firebase; just ensure tests pass.
5) Update README with setup steps.

TESTS:
- Basic test that FirebaseInit provider exposes initialized flag (mocked).

OUTPUT: unified diff patch with all files.
```

## Prompt 04 — Google Auth + Auth State
```text
ROLE: Senior Flutter engineer.

CONTEXT: Project from Prompts 01–03. We’re using Firebase Auth (Google).

TASK:
1) Add `firebase_auth` and `google_sign_in`.
2) Create `AuthController` (Riverpod Notifier) with states: signedOut, loading, signedIn(user).
3) SignInScreen: Google button; CaptureScreen shows signed-in user displayName.
4) Routing: If signed out → SignInScreen; else → tabs.

TESTS:
- Widget test: when AuthController is mocked to signedOut, app shows SignInScreen.
- Widget test: when signedIn, app shows the tabbed layout and displayName is visible.

OUTPUT: unified diff patch.
```

## Prompt 05 — Firestore Rules + Users Profile (Emulator Tests)
```text
ROLE: Senior Firebase engineer.

TASK:
1) Add Firestore rules:
   - `/users/{uid}`: read/write only if `request.auth.uid == uid`.
   - `/entries/{id}`: read/write only if `request.auth.uid == resource.data.userId` on update; on create, `request.resource.data.userId == request.auth.uid`.
2) Add `users` profile creation on first login (`UserProfileService`).
3) Add Firestore emulator unit tests (Node or Dart) to verify:
   - Owner can read/write their user doc.
   - Others cannot read/write.
   - Owner can create/read/update their entries; others denied.

OUTPUT:
- unified diff including rules, minimal service, and emulator tests with npm scripts to run them (documented in README).
```

## Prompt 06 — Entry Model (Freezed) + Serialization
```text
ROLE: Senior Flutter engineer.

TASK:
1) Define `Entry` union with Freezed:
   - shared: id, userId, type("thought"|"task"|"goal"), text, tags[], createdAt, updatedAt, isArchived
   - optional: emotions{joy,fear,sadness,anger,disgust}, moodScore, followUps[{question,answer,timestamp}]
   - task-only: urgency("now"|"soon"|"later"), dueDate, status("todo"|"doing"|"done"), checklist[{title,done}], recurrence{type,interval,pattern}
   - goal-only: targetDate, steps[{title,estDate,done,notes,subTasks[{title,done}]}], reviewCadence
2) Add JSON (de)serialization, converters for timestamps.
3) Provide factory builders and copyWith helpers.

TESTS:
- Round-trip JSON tests for at least 1 thought, 1 task, 1 goal sample.
- Ensure default values (e.g., emotions default zeros) are applied.

OUTPUT: unified diff with models, tests, and codegen updates.
```

## Prompt 07 — EntriesRepository + Riverpod Controllers
```text
ROLE: Senior Flutter engineer.

TASK:
1) Implement `EntriesRepository` with methods:
   - create(Entry), update(Entry), archive(id), watchFeed({typeFilter?, tag?, limit, startAfter?}), getById(id)
2) Implement `EntriesFeedController` (StreamProvider) and `EntryComposerController` (Notifier) to manage composing/saving.
3) JournalScreen shows a list of entries from `EntriesFeedController` with minimal tiles per type.

TESTS:
- Emulator-backed tests: CRUD success and security failures for other users.
- Widget test: feed renders with thought/task/goal indicators.
- Golden tests: empty state, non-empty list.

OUTPUT: unified diff.
```

## Prompt 08 — Tier 0 Rules Classifier + Capture Flow v1
```text
ROLE: Senior Flutter engineer.

TASK:
1) Implement a local rules-based classifier:
   - "goal" if text includes patterns like "goal:", "I want to", "by <date>" (config driven).
   - "task" if leading verb + time/urgency tokens ("today", "tomorrow", "now", "ASAP").
   - else "thought".
2) Capture screen: textarea + "Classify & Continue" → shows detected type and allows override; "Skip Questions" applies defaults (per spec).
3) On save, create Entry with defaults per type.

TESTS:
- Unit tests for classifier patterns.
- Widget test: typing "Run 5km tomorrow" → detected as task; save persists task entry.

OUTPUT: unified diff.
```

## Prompt 09 — MobileBERT TFLite (Tier 1) Integration
```text
ROLE: Senior Flutter engineer with on-device ML.

TASK:
1) Add `tflite_flutter` and wire a `LocalClassifier` service:
   - Loads MobileBERT TFLite model from assets.
   - Exposes `classify(text) -> {type, confidence}`.
   - Fallback to Tier 0 rules if model unavailable.
2) Update Capture flow: run Tier 1; if confidence >= 0.75, accept; else mark "needs LLM".
3) Add basic model loader tests (mock interpreter) and unit tests that verify fallback path.

NOTE: Placeholders for the actual model file are fine; code should be ready to drop a real model into `assets/ml/`.

OUTPUT: unified diff with service, providers, config, tests, and asset registration.
```

## Prompt 10 — Functions: Gemini + LangChain Classification & Follow-Ups (Tier 2)
```text
ROLE: Senior Firebase Functions engineer (TypeScript).

TASK:
1) Create Functions project (TypeScript): `functions/src/index.ts`.
2) Implement callable: `classifyAndEnrich({ text, priorType, uid, consent }) -> { finalType, confidence, followUps[], tags[], emotions{} }`
   - Use Gemini via LangChain run-time.
   - Follow spec logic: if long text or confidence low, refine classification.
   - Generate 1-at-a-time follow-up questions per type rules.
   - Return top-3 tags (max 3 words each) and emotions on 0–5 scale.
3) If `consent===true`, store raw LLM chat to Storage under `raw_chats/{uid}/{entryId}.json`.
4) Add Functions unit tests with mocks; run under emulator.
5) Add retry policy and timeouts (e.g., 6s budget).

OUTPUT:
- unified diff with Functions code, package.json, tests, and emulator config updates.
- README: how to run emulators + tests.
```

## Prompt 11 — Frontend Tier 2 Wire-Up: Chat-Style Follow-Ups
```text
ROLE: Senior Flutter engineer.

TASK:
1) Add `FollowUpController` managing a queue of follow-ups from the Functions call.
2) Update Capture flow: if Tier 1 low-confidence or text > 250 chars, call Functions; display one follow-up at a time with typing effect; record answers into Entry.followUps.
3) “Skip Questions” fills defaults and proceeds to save.

TESTS:
- Widget test: simulate a Functions mock returning two follow-ups; verify the conversation flow and saved answers.
- Golden test: follow-up bubble rendering.

OUTPUT: unified diff, tests.
```

## Prompt 12 — Journal Feed Polishing + Filters
```text
ROLE: Senior Flutter engineer.

TASK:
1) Journal filters: type chips (All/Thought/Task/Goal), tag search box.
2) Auto-scroll to newly saved entry (animated).
3) Archive with confirm dialog (soft delete toggle).

TESTS:
- Widget test: filtering by type reduces list accordingly.
- Unit test: archive flips isArchived and removes from default feed.

OUTPUT: unified diff.
```

## Prompt 13 — Calendar Week/Month + Urgency Colors + Recurrence
```text
ROLE: Senior Flutter engineer.

TASK:
1) Implement Calendar view with Week (default) and Month toggle.
2) Color code: overdue(red + icon), due soon(orange), on track(green).
3) Recurrence: data model to generate next occurrence on completion; do NOT mutate original schedule (clone next occurrence).

TESTS:
- Unit tests for recurrence generation.
- Widget tests: tapping a day shows detail popup; completing one creates the next.

OUTPUT: unified diff.
```

## Prompt 14 — Insights: Charts + Filters
```text
ROLE: Senior Flutter engineer.

TASK:
1) Add Insights tab with:
   - Emotion time-series (smoothed) using fl_chart.
   - Pie chart by type with quick filters: Today, Yesterday, Last Week, Last Month, Last Year, Overall (default Today on reopen).
2) Add selectors and persist last selection per session only.

TESTS:
- Golden tests for charts with sample data.
- Unit tests for date range filters.

OUTPUT: unified diff.
```

## Prompt 15 — Cloud Scheduler: Weekly Review + Daily Digest Writers
```text
ROLE: Senior Firebase Functions engineer.

TASK:
1) Add scheduled Functions:
   - Weekly (Mon 09:00 Europe/Dublin): generate LLM narrative, top 3 tags for the past week; write to `/reviews/{uid}/{weekId}`.
   - Daily (08:00 Europe/Dublin, user-settable later): create digest doc `/digests/{uid}/{date}` with tasks ordered by urgency; include yesterday’s mood highlight.
2) Ensure idempotency + re-runs safe.

TESTS:
- Unit tests with fake timers, asserting written docs structure for mocked data.

OUTPUT: unified diff.
```

## Prompt 16 — Notifications + Deep Links
```text
ROLE: Senior Flutter engineer.

TASK:
1) Add Firebase Messaging + local notifications.
2) On digest/review doc creation, backend sends push with deep-link params (tab + record id).
3) App handles deep links to open the correct tab and, if present, a detail card.

TESTS:
- Unit test for deep-link parser.
- Widget test: simulate deep-link navigation to Insights or Calendar detail.

OUTPUT: unified diff (frontend + minimal Functions sender updates).
```

## Prompt 17 — Offline Indicators + Conflict Strategy
```text
ROLE: Senior Flutter engineer.

TASK:
1) Enable Firestore offline persistence.
2) Show a small top banner when offline.
3) If conflict on update, adopt last-write-wins but surface a toast “updated after sync”.

TESTS:
- Integration test (emulator): simulate offline save then online sync; verify UI banner and resolution.

OUTPUT: unified diff.
```

## Prompt 18 — Embeddings Pipeline (Server) + Search Frontend
```text
ROLE: Senior Firebase engineer.

TASK:
1) Functions: on entry create/update, generate text embedding (Gemini or Vertex) and store `{embedding: number[], norm}` on a side collection.
2) Add callable function `searchEntries({query, topK}) -> {entryIds[]}` computing cosine similarity server-side.
3) Frontend: search bar that calls server search; offline keyword fallback.

TESTS:
- Functions unit tests: cosine similarity ranking with fixtures.
- Frontend unit tests: fallback when offline.

OUTPUT: unified diff.
```

## Prompt 19 — Consent Gate for Raw LLM Storage + Rules Update
```text
ROLE: Senior engineer.

TASK:
1) Add a per-entry toggle “Save raw chat JSON” before classification; default false.
2) If true, include `consent=true` in Functions request; store into Storage bucket path `raw_chats/{uid}/{entryId}.json`.
3) Update rules to restrict access to owner only.

TESTS:
- Rules test: only owner can list/get their raw chats.
- Widget test: toggling consent flips the flag passed to backend.

OUTPUT: unified diff.
```

## Prompt 20 — E2E Polish, Error Surfaces, App Wiring Check
```text
ROLE: Senior engineer.

TASK:
1) Add global error banners for:
   - Network lost
   - LLM unavailable
   - Sync failed
2) Verify all flows are wired:
   - Capture → Classify (Tier0/1/2) → Follow-ups → Save → Journal + Calendar reflect changes → Insights includes data.
3) Add an `integration_test` that:
   - Signs in (mock)
   - Creates a task via Tier 0
   - Creates a thought via Tier 1 fallback
   - Triggers Tier 2 mock and handles 1 follow-up
   - Verifies Journal, Calendar, Insights updates
4) README final runbook (dev & test).

OUTPUT: unified diff with tests and docs.
```

---

## Sanity Check on Step Size
- Each prompt is small, testable, and provides immediate value.
- Higher-risk items (on-device ML, notifications, embeddings) are isolated behind providers and feature flags.
- CI + emulators are introduced early to keep the main branch always green.

