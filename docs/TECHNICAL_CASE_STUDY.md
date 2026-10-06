# Technical Case Study

PulseBP is a Flutter application for Android that lets a person log blood pressure and related wellness metrics by hand, keeps the data on the device first, and derives history, trends, a summary and reports from it. This document walks through the engineering end to end. It describes software behaviour only; the app is a logging tool and makes no health claims (see [What PulseBP is and is not](PRODUCT_OVERVIEW.md#what-pulsebp-is-and-is-not)).

Approximate sizes (line counts): about 56k lines of Dart across roughly 245 files in the application source (excluding generated localisation output), about 19k lines across roughly 90 test files, five Kotlin files for the native Android layer, and nine TypeScript files (about 1.5k lines) for the serverless proxy.

## 1. Design goals

1. Logging never waits on the network: a value typed with no signal is saved immediately.
2. Domain math is pure and testable: averages, bucketing, scores and adherence are functions of their inputs.
3. Cloud features are optional layers: sync, AI summaries and subscriptions can fail without breaking the core loop.
4. Paid server-side features get server-side checks, because a flag in the app cannot protect something that costs money per call.
5. Say only what is true: readings are typed by the user, and the app does not measure.

## 2. Application structure

The code is a layered, feature-organised Flutter app.

- **Presentation:** screens, modal sheets and widgets read state through Riverpod providers. Navigation is a small phase machine at the top (loading, language, onboarding, setup, app) that cross-fades between stages, then a six-tab shell with most detail and entry UIs opened as bottom sheets.
- **State:** notifiers own mutable state and call a thin repository; derived providers (trends, score, reports, AI input) recompute from a single recompute signal so a change from the user and a change applied from the cloud invalidate the same things.
- **Repository and database:** one repository fronts a SQLite helper. The database is schema-versioned and migrated in place with add-column-if-missing and create-if-not-exists steps, and opening it is guarded so concurrent first access cannot race.
- **Pure services:** trends, adherence, the composite score, rule-based insights, report building and the AI payload builder have no database or Riverpod access and take an explicit current time.
- **Cross-cutting modules:** sync, billing, ads, consent, analytics, messaging and auth are separate modules.

## 3. Entry, validation and categorisation

Blood pressure entry uses two steppers and tap-to-type dialogs. Typed values are checked against the same bounds as the steppers and rejected in place with a message that names the allowed range; nothing is silently changed while the user is typing. At save time the lower value is adjusted if it would reach the upper, and a timestamp later than now is clamped to now. A reading can carry optional context tags (preset or free text) and stores decimals.

Other metrics have their own bounds.

Each blood pressure reading is assigned one of five **app-defined bands** by fixed threshold checks on the two values. These are display categories, evaluated in a fixed order, with no personalisation by age, sex or declared conditions. This repository does not attribute them to any guideline, and the thresholds are not published here.

## 4. Local-first storage

SQLite is the source of truth. Separate tables hold each metric, the profile, reminders, medication definitions and schedules, and generated dose events. Dose events carry a uniqueness constraint over medication and scheduled time, so regenerating a window cannot create duplicates. Small settings live in shared preferences.

Because the UI reads the database, offline use needs no special mode. A free-tier history window is enforced at the query layer rather than by deleting rows.

## 5. History, averages, trends and the Heart Score

A pure trends service buckets samples by the chosen period: one point per reading for a day, one per calendar day for week and month, one per calendar month for year, and daily or monthly buckets for custom ranges depending on span. Blood pressure charts plot upper and lower series aligned by bucket. Overall average, minimum and maximum are computed from raw samples, not from bucket means.

The **Heart Score** is an app-defined composite: each metric with data in a recent window is mapped to a sub-score and the available sub-scores are combined, with a confidence indicator reflecting how many metrics contributed. It is documented in the app as not a diagnosis and not medical advice, and it is not a risk model. The exact mapping is not published here.

The **rule-based insights engine** runs locally and deterministically, in all supported languages. It flags simple threshold co-occurrences (for example a logged short-sleep pattern alongside out-of-band readings) and requires a minimum number of readings before it comments on a trend. These are checks, not statistical analysis.

## 6. Reminders and local notifications

Reminders are title, category, time and enabled state. On Android, scheduling uses native background-work APIs through a platform channel: periodic work aligned to the next occurrence for recurring reminders, one-shot work for medication doses generated from the rolling schedule window. The build strips exact-alarm permissions on purpose. The platform's background-work library persists scheduled work across reboot and update, and the app resyncs all schedules on every launch. A notification-permission pre-prompt explains the request, and a blocked state deep-links to system settings.

A medication-schedule rule suppresses generic medication reminders once at least one medication has been added, so the dose events drive notifications. Push messaging is separate: message types and deep links are checked against allow-lists, and the custom link scheme accepts only a short list of known destinations.

## 7. Reports and export

A pure report service assembles per-metric statistics for a period; the PDF builder lays them out on A4 with a header and a not-medical-advice footer on every page, and a CSV lists every entry in the selected period. Both are written to a temporary file and shared through the OS share sheet. PDF compression is deliberately disabled because compressed streams produced corrupt files in the compiled Android build; this is a deliberate decision.

## 8. Accounts and sync

The user is a guest by default. Signing in with Google or Apple links the guest identity, so existing local data stays. Cloud data lives in owner-scoped Firestore subcollections protected by owner-only security rules.

The sync design is the most substantial subsystem. Every local mutation enqueues a durable operation. A cycle pushes the queue, pulls remote changes, scans for local rows missing from the cloud, then pushes again, with exponential backoff and classification of retryable errors. Deletes are soft markers so they propagate. Conflicts resolve by last-updated-wins. A guard flag marks remote-applied writes so they are not re-enqueued. Triggers include app resume, sign-in, connectivity returning and a periodic timer, and realtime listeners attach only after the first full reconciliation. Details are in [Engineering challenges](ENGINEERING_CHALLENGES.md).

In-app account deletion removes the sign-in account and the user's cloud profile record and clears local storage. This repository makes no claim beyond that about cloud-side data cleanup.

## 9. Monetisation

The free tier shows ads through a mediation SDK (AdMob among the networks), with a consent flow, a shared cooldown between interstitials, request throttles and remote switches that default to off until a real read says otherwise. The Pro subscription uses native billing with monthly and yearly plans and a trial offer, reads prices from the store, and supports restore. Pro lifts the history window and reminder cap, unlocks the medications sheet and trend extras, removes ads and enables the AI summary. Feature gating goes through one central gate.

## 10. The AI summary path

The optional AI summary is the one place where logged data leaves the device for processing by a third party, so it is built as a bounded relay.

- The app computes aggregates locally (period averages, extremes, latest values, counts, adherence) and builds a compact snapshot. Raw reading rows are not sent, and no name or email is included. Profile context is part of the snapshot: age, sex, height, body weight, declared conditions, a free-text goal, and medication names and doses. It is not anonymous.
- A serverless proxy built on Cloudflare Workers receives the request. It verifies the caller's sign-in token and a stored purchase entitlement (set by a separate receipt-verification call) before doing anything, applies per-user weekly allowances and a short-window rate limit, caches repeat requests keyed by a fingerprint of the snapshot, and calls a third-party language-model API with a versioned prompt. The key for that API exists only on the proxy.
- The prompt instructs the model to rely on supplied data only, to avoid diagnosing, not to recommend starting, stopping or changing prescription medication without clinician guidance, and to answer in strict JSON. The requested output is a score, a status label, a short summary, lists of positives, concerns, trends and recommendations, and severity-tagged alerts; the prompt also asks the model to flag values that may need professional attention. The proxy checks and clamps the result against a schema before returning it. Wording quality is not guaranteed.
- The client caches results and only calls on an explicit button press; reopening a cached result costs nothing. Failures do not consume allowance.

Local Pro features are gated in the app; server-side entitlement is the planned direction. Today the server checks protect the one feature that has a per-call cost.

## 11. Start-up

A native splash is followed by an immediately painted matching Flutter frame; notification channels, reminder resync, messaging setup, consent and ad initialisation run after the first frame, with per-step timings sent to analytics. No timing figures are published here.

## 12. Localisation, analytics and crash reporting

Fifteen locales are generated from message templates with ICU placeholders, including right-to-left layouts, and layout-overflow tests run against locales. Translation is tooled by scripts; no claim is made about translation review. Firebase Analytics records screen views, feature use, purchase and ad funnels, sync telemetry and start-up timing under a convention that forbids free text, and Crashlytics and Performance traces are enabled.

## 13. Tests

The suite has about 90 test files and roughly 700 test declarations, run with Flutter's test runner and an FFI-based SQLite implementation on the host. They cover blood pressure and other metric logging, database CRUD and migrations, provider behaviour, sync phases and providers, auth, subscription and Pro-gate logic, purchase flows, reminders and notification services, the medication schedule, scheduler and adherence engines (including an end-to-end flow), the Heart Score, trends, the local insights engine, the AI client and input builder, report export, ad and consent phases, analytics, tours, onboarding widgets and localisation overflow. The proxy has a type-check script but no test files. No test results are claimed.

## 14. Trade-offs and future work

- Reminder timing relies on system background scheduling, so delivery can shift under battery optimisation.
- Conflict resolution is simple (last-updated-wins); a per-edit version clock would handle multi-device edits better.
- Local storage relies on platform sandboxing; no additional app-level encryption layer is claimed.
- Local features are gated in the app; server-side entitlement is the planned direction.
