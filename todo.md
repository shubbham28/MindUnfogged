# MindUnfogged — Engineering TODO Checklist

Use this as a living, box-ticking plan. Keep PRs small and green. Every task must land with tests.

Legend:  
- [ ] = not started [~] = in progress [x] = done

---

## M1 — Scaffold & CI
- [ ] Create Flutter app `mind_unfogged`.
- [ ] Add core packages: `flutter_riverpod`, `go_router`, `freezed_annotation`, `json_annotation`, `build_runner`, `freezed`, `json_serializable`, `fl_chart`, `logger`.
- [ ] Implement theme (Indigo #3949AB, Lavender #C5CAE9, Emerald #43A047), light/dark.
- [ ] Add 4-tab routing (/capture, /journal, /calendar, /insights) with bottom nav.
- [ ] Placeholder screens with unique Keys.
- [ ] Widget tests for tab navigation.
- [ ] Golden tests for AppBar light/dark.
- [ ] GitHub Actions workflow for `flutter test` with cache.
- [ ] Firebase emulator config placeholders committed.
- [ ] README: getting started, testing.

**DoD:** `flutter test` green locally and in CI; goldens approved.

---

## M2 — Auth + Rules
- [ ] Add `firebase_core`, `firebase_auth`, `google_sign_in`.
- [ ] AuthController (Riverpod Notifier) with signedOut/loading/signedIn.
- [ ] SignIn screen with Google button.
- [ ] Capture screen displays user displayName on sign-in.
- [ ] Route guard: unauth → SignIn, auth → tabs.
- [ ] Create `/users/{uid}` on first login (`UserProfileService`).
- [ ] Firestore rules: users & entries per-owner.
- [ ] Emulator tests for rules (allow owner, deny others).

**DoD:** Auth flow works against emulator; rules tests pass.

---

## M3 — Data Model + Repository
- [ ] Define `Entry` (Freezed) union: shared fields + task-only + goal-only fields.
- [ ] JSON (de)serialization with timestamp converters.
- [ ] Factory builders + copyWith.
- [ ] `EntriesRepository` CRUD, queries, pagination.
- [ ] `EntriesFeedController` (StreamProvider), `EntryComposerController` (Notifier).
- [ ] Journal list renders minimal tiles per type.
- [ ] Unit tests: model round-trip, tag parsing util.
- [ ] Emulator-backed CRUD tests.

**DoD:** Create/update/archive flows pass tests; journal shows entries.

---

## M4 — Capture Flow v1 (Tier 0)
- [ ] Local rules-based classifier (config patterns).
- [ ] Capture UI: textarea, "Classify & Continue", override type.
- [ ] "Skip Questions" applies defaults.
- [ ] Save entry with defaults per type.
- [ ] Unit tests: rules classifier edge cases.
- [ ] Widget test: example task detection persists correctly.

**DoD:** Create thought/task/goal without cloud/LLM dependency.

---

## M5 — Local ML (Tier 1)
- [ ] Add `tflite_flutter`; set up assets `assets/ml/`.
- [ ] `LocalClassifier` service loads MobileBERT TFLite.
- [ ] `classify(text)` returns `{type, confidence}`.
- [ ] Confidence gate: accept >= 0.75 else mark for Tier 2.
- [ ] Fallback to Tier 0 if model missing/unavailable.
- [ ] Tests: mock interpreter load, fallback path.
- [ ] Perf: warm-up on app start; log latency.

**DoD:** Tier 1 classification integrated; tests pass without real model.

---

## M6 — LLM Orchestration (Tier 2)
- [ ] Initialize Firebase Functions (TypeScript) project.
- [ ] Implement callable `classifyAndEnrich`.
- [ ] LangChain + Gemini integration with timeout/retry.
- [ ] Returns: `finalType`, `confidence`, `followUps[]`, `tags[]`, `emotions{}`.
- [ ] Consent handling: optional raw chat JSON to Storage.
- [ ] Unit tests (mocks) + emulator script.
- [ ] Frontend service to call function; error handling.

**DoD:** Mocked function call flows end-to-end; failures surface graceful banners.

---

## M7 — Chat-Style Follow-Ups + Journal Filters
- [ ] `FollowUpController` queues Q&A; typing effect.
- [ ] One-at-a-time UI; answers appended to `Entry.followUps`.
- [ ] “Skip Questions” fills defaults and continues.
- [ ] Journal filters: type chips and tag search.
- [ ] Auto-scroll to new entry.
- [ ] Archive with confirm dialog.
- [ ] Tests: follow-up flow; filtering; archive logic.

**DoD:** Capture → follow-ups → save → journal reflects immediately.

---

## M8 — Calendar + Recurrence + Insights
- [ ] Calendar week (default)/month toggle.
- [ ] Color code urgency: overdue(red + icon), soon(orange), on track(green).
- [ ] Recurrence engine: completion spawns next occurrence (immutably).
- [ ] Insights: emotion time-series + pie by type.
- [ ] Quick filters: Today, Yesterday, Last Week, Last Month, Last Year, Overall (reset to Today on reopen).
- [ ] Tests: recurrence unit tests, charts golden tests, date range filters.

**DoD:** Tasks/goals visible on calendar; insights interactive.

---

## M9 — Schedulers, Offline, Search, Notifications, E2E
- [ ] Cloud Scheduler: weekly review (Mon 09:00 Europe/Dublin) writes to `/reviews/{uid}/{weekId}`.
- [ ] Cloud Scheduler: daily digest (08:00 Europe/Dublin) writes to `/digests/{uid}/{date}`.
- [ ] Firestore offline persistence enabled.
- [ ] Offline banner; last-write-wins with “updated after sync” toast.
- [ ] Embeddings generation on create/update; server cosine search callable.
- [ ] Frontend search bar with offline keyword fallback.
- [ ] Notifications: FCM + local; deep-links route to tab and record.
- [ ] Integration test: full happy-path (Tier0/1/2) updates Journal/Calendar/Insights.
- [ ] README: final runbook and troubleshooting.

**DoD:** E2E happy path green in CI; push notifications deep-link correctly.

---

## Security & Privacy Checklist
- [ ] Firestore security rules per owner; tests for users & entries.
- [ ] Storage rules: raw chats only for owner.
- [ ] Do not store raw LLM data unless `consent=true`.
- [ ] Secrets: Functions config or Secrets Manager; no secrets in repo.
- [ ] PII minimization: only store necessary fields.

---

## Observability & Quality Gates
- [ ] Structured logs via `logger`; include latency for Tier 1 & Tier 2.
- [ ] Analytics events: capture_started, classify_tier, followup_shown, entry_saved, review_generated, notification_opened.
- [ ] Performance budget: Tier 1 < 120ms p50; Tier 2 < 6s timeout.
- [ ] CI gates: tests + lints + format check must pass before merge.
- [ ] Code owners for critical paths (auth, rules, functions).

---

## Nice-to-Haves (Defer until MVP complete)
- [ ] TinyLlama on-device fallback (feature flag).
- [ ] App settings: user-settable digest time; review cadence per goal.
- [ ] Export/import entries (JSON).
- [ ] Theming preferences persistence.
- [ ] Accessibility pass (semantic labels, large fonts).

