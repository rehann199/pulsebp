# Engineering Challenges

Seven problems that stand out in the production code. Outcomes describe observable design behaviour, not measured metrics; none are claimed. The app is a logging tool, so these are software problems, not health ones.

---

## 1. Offline-first sync without a custom API server

**Problem**
People log on a phone that is frequently offline and may use several devices. The product has no bespoke server for user data, so sync has to be built from a managed document database and careful client logic.

**Context**
SQLite is the source of truth for the UI. Cloud storage is a per-user mirror that only exists once the user signs in.

**Engineering approach**
- Write locally first. Every create, update and delete also appends an operation to a persistent queue.
- Run a cycle of push, pull, scan for local rows missing remotely, then push again.
- Retry with exponential backoff and a cap, and distinguish retryable from permanent errors.
- Represent deletes as soft markers so they propagate to other devices.
- Resolve conflicts by last-updated-wins.
- Mark remote-applied writes with a guard flag so they do not re-enter the queue and echo back.
- Attach realtime listeners only after the first full reconciliation, pause them during a cycle, ignore local pending-write echoes and debounce changes.
- Trigger cycles on app resume, sign-in, connectivity returning and a foreground timer.
- For signed-in users, require a successful backup before the "reset app" action wipes local data.

**Trade-offs**
Last-updated-wins is simple but coarse. For some metrics the version clock is derived from the reading's own timestamp rather than an edit time, so two edits that keep the same timestamp compare as equal. An explicit per-edit version would be a natural improvement for multi-device editing.

**Outcome**
Logging has no dependency on connectivity, and sync is a background concern. Phases of the sync design have dedicated unit tests.

---

## 2. Server-side checks in front of a paid feature

**Problem**
A flag stored in the app cannot enforce a paid, per-call feature on its own.

**Context**
The optional AI summary calls a third-party language-model API. The key must never ship in the app, and cost must be bounded per user.

**Engineering approach**
- Put a thin serverless proxy between app and model API.
- Verify the caller's sign-in token at the proxy against the identity provider's public keys, without a server SDK.
- Forward the store-signed purchase receipt from the app to a separate verification call at the proxy, which checks it using the stores' signature schemes and records an entitlement for that user. The summary call checks the stored entitlement; if anything is missing or invalid, deny.
- Apply a per-user weekly allowance and a short-window rate limit; do not consume allowance on failures.
- Cache repeat requests by a fingerprint of the snapshot, and treat language translation as a separate, cheaper mode so changing the app language does not spend an analysis.

**Trade-offs**
Local Pro features (history window, reminder cap, export) are gated in the app; server-side entitlement is the planned direction. The server checks protect the one feature with a marginal cost.

**Outcome**
No model key in the app, bounded and predictable usage per user, and repeated views of the same result are free.

---

## 3. Keeping the AI path bounded and honest

**Problem**
Language-model output about health data can drift into diagnosis-like statements, break its schema, or leak more data than the feature needs.

**Context**
The feature is optional, behind sign-in and Pro, and explicitly informational.

**Engineering approach**
- Compute all aggregates locally; the model is given numbers and does no arithmetic on raw readings.
- Send a compact snapshot rather than reading rows, with no name or email. Profile context is included: age, sex, height, body weight, declared conditions, a free-text goal, and medication names and doses. The feature is therefore not anonymous.
- Use a versioned prompt that tells the model to rely only on supplied data, to avoid diagnosing, and not to recommend starting, stopping or changing prescription medication without clinician guidance. It also asks for a score, a status label, a summary, recommendations and severity-tagged alerts, and to flag values that may need professional attention.
- Request strict JSON and check it at the proxy: clamp the numeric score, restrict enumerations and cap list lengths.
- Call the proxy only on an explicit button press, cache results on the device, and keep a deterministic, free, rule-based insights engine as the default experience.
- Label the output as not medical advice in the UI.

**Trade-offs**
Prompt instructions reduce but do not eliminate the chance of unhelpful wording, which is why the output is labelled informational and is optional. The quality of generated text in each language is not claimed.

**Outcome**
A small, bounded surface: a single request type, schema-checked output, and a clear fallback.

---

## 4. Reminders without exact-alarm permission

**Problem**
Exact alarms add permissions and store-review friction, yet reminders must survive reboots and app updates and must not duplicate each other.

**Context**
Reminders cover several categories, and medication reminders can come from real dose schedules.

**Engineering approach**
- Schedule through native background-work APIs reached via a platform channel: periodic work aligned to the next occurrence for recurring reminders, one-shot work for medication doses.
- Strip exact-alarm permissions from the merged build.
- Rely on the background-work library to persist scheduled work across reboots and updates, and resynchronise every schedule on each launch.
- Suppress generic medication reminders once at least one medication has been added, so dose events drive notifications.
- Use a permission pre-prompt that explains the request, and a blocked state that deep-links to system settings.

**Trade-offs**
Background work is not guaranteed to fire at the exact minute under battery optimisation, so the app does not promise exact-time delivery. Time zone or daylight-saving changes can shift alignment until the next resync.

**Outcome**
Fewer permissions and a simpler review path, with the timing trade-off stated openly.

---

## 5. A deterministic medication schedule engine

**Problem**
Schedules can be daily, weekday-based or a course with an end, with several times per day. They must produce consistent dose records for reminders and adherence, and regenerating them must be safe to repeat.

**Engineering approach**
- A pure generator expands a schedule into dose events for a rolling window.
- A uniqueness constraint over medication and scheduled time makes insertion idempotent.
- Regeneration replaces events inside a range within one transaction and runs at start-up.
- Adherence counts a dose as met only when marked taken; skipped and still-pending doses count against the total.
- Weekly figures use a defined week start and feed the composite score and insights.

**Outcome**
A unit-tested pure core, an end-to-end flow test, and one source of truth for both reminders and adherence.

---

## 6. Start-up with heavy SDKs

**Problem**
Firebase, consent, notifications and ad SDKs risk a blank frame and slow cold start.

**Engineering approach**
- Show a native splash immediately, then paint a matching first frame from Flutter before any asynchronous initialisation.
- Perform only essential work before the main app (locale, configuration, orientation, core Firebase).
- Defer notification channels, reminder resync, messaging setup, consent and ad initialisation until after the first frame.
- Keep a single persistent banner alive across tab switches by holding tabs in a stack.
- Record per-step timings and send them to analytics, with Performance traces.

**Outcome**
Instrumented start-up. No measured numbers are published here.

---

## 7. A free tier with fail-closed ad controls and graceful gating

**Problem**
Monetising a free app needs consent handling and abuse limits that can change without a new release, while free users keep core logging and Pro features stay discoverable.

**Engineering approach**
- Request ads only with consent, only for free users, and only when a remote per-placement switch is true; the switches default to off, with a global kill switch.
- Share a cooldown across interstitial placements, enforce a load-time budget, and throttle requests per zone.
- Route feature gating through one central gate that records which trigger opened the paywall.
- Keep logging free and enforce the history window at the query layer rather than deleting data.
- Allow temporary rewarded unlocks for export and summaries, persisted across restarts.

**Trade-offs**
Ads inside a health-logging app are a product decision with a cost to the experience; the controls exist to make that behaviour tunable and conservative.

**Outcome**
Ad behaviour is remotely controllable, and gating is consistent and covered by tests.

---

## What the test suite covers

The suite has about 90 test files and roughly 700 test declarations: logging and validation for each metric, database CRUD and migrations, provider behaviour, sync phases, auth, subscription and gate logic, reminders and notifications, the medication engines, the composite score, trends, rule-based insights, the AI client and input builder, report export, ads and consent, analytics, onboarding widgets and localisation overflow. No test results are claimed.
