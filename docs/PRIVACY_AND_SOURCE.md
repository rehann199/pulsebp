# Privacy and Source

## What this repository is

A portfolio showcase for a production application. It describes the engineering of PulseBP; it is not a source distribution, and it is not health guidance.

## What PulseBP is and is not

PulseBP is a logging and tracking tool. Users type in values from their own devices, and the app stores and summarises them. It does not measure blood pressure or any other metric, has no device, camera, sensor or health-platform integration, and is not a medical device. It does not diagnose or treat any condition and does not replace professional medical advice. The app's own privacy text says this in a callout at the top, and its summaries, its Heart Score explanation and every exported PDF page repeat a not-medical-advice message. Anyone with a health concern should speak to a qualified professional.

## What is published

- Written descriptions of architecture, design decisions and trade-offs
- Original diagrams created for this showcase
- Real screenshots from the Android build, cropped to the app content (system status bar and ad-placeholder strips removed), captured with sample/placeholder data, metadata removed
- The app icon, resized
- The public store link

## What is intentionally not published

- Production source code, in whole or in part
- Scoring weights, thresholds, quota values and other tuning values
- Build and signing configuration, certificates and keystores
- Firebase and Google service configuration, API keys and tokens
- Ad network identifiers, placement IDs and account details
- Billing product identifiers
- Backend addresses, endpoint names, database paths and security-rule contents
- The identity of the third-party language-model provider and model
- Version control history of the production repository
- User data of any kind

## Counts and test claims

Line and file counts are approximate and exclude generated localisation output. Test counts are counts of test files and declarations; no pass state is claimed. Where behaviour could not be confirmed it is worded conservatively or left out.

## How the app handles data (conceptual)

This section describes design, not compliance or certification, and it does not replace the app's own privacy policy.

- **On the device.** Readings and logs live in a local SQLite database inside the app's sandbox. This repository makes no claim of additional app-level encryption.
- **Optional cloud copy.** If the user signs in, a mirror of their data is written to a per-user area of a managed cloud database, protected by owner-only security rules. Never signing in keeps data on the device only. Signing out stops syncing; it does not by itself remove a cloud copy that already exists.
- **Account deletion.** In-app deletion removes the sign-in account and the user's cloud profile record and clears local storage. This repository makes no further claim about cloud-side cleanup.
- **AI summaries.** When a signed-in Pro user presses the generate button, an aggregate snapshot (averages, extremes, latest values, counts, adherence) plus profile context (age, sex, height, body weight, declared conditions, a free-text goal, and medication names and doses) is sent through a serverless proxy to a third-party language-model API. Raw reading rows, names and email addresses are not part of the snapshot. Because profile context is included, the feature is not anonymous. The proxy stores no reading rows; it keeps per-user allowance counters, an entitlement record, and a time-limited cache of generated summary text. The output is labelled as not medical advice.
- **Analytics and crash reporting.** Usage events and crash data are collected with Firebase products under a convention that forbids free text in event parameters.
- **Ads.** The free tier shows ads through a mediation SDK, gated by a consent flow and remote switches. Pro removes them.
- **Permissions.** The declared permissions are network access, network-state access, vibration, boot-completed (used by the scheduling libraries to restore reminders), notifications, foreground service and billing. Ad SDKs can add further permissions at build time, so the merged result may include further permissions.

## Images

- **Screenshots** are real captures from the Android build, not mock-ups. The set covers the entry, reminders and paywall flows and the splash screen. It deliberately does not include any populated reading, trend or history screen, and none is described as if it were shown. The paywall capture is cropped to the title and benefits list and shows no prices. Images were re-encoded to WebP with metadata removed.
- **Diagrams** are original vector illustrations with conceptual labels.

## Access

Source access for hiring or due-diligence purposes can be discussed at the owner's discretion.
