# Store data-safety deltas — Problem Reporting (Phase 6)

Console-only deliverables for the `problem_reporting` rollout. These are **not code**; they are changes
to the Google Play Data safety form and the Apple App Store privacy labels. They must be published
**before** the `problem_reporting` Remote Config flag is turned on — the same hard gate as the privacy
policy (see [implementation-plan.md](implementation-plan.md) §6 and [concept.md](concept.md) §6).

The `contact_us` (Settings "Send feedback" email) lane is **not** gated by this: it stores nothing on our
servers — the email leaves through the user's own mail client — so it changes no store data-safety
declaration. Only the content-report write path (`problemReports` in Cloud Firestore) does.

## What actually changes vs. what already exists

The voting feature already declared, on both stores, a device-scoped random identifier and app-diagnostic
data, **not linked to identity**, **not shared**, **not used for tracking**. A problem report reuses that
same identifier (the Install ID) and adds **one genuinely new data type**: the optional free-text **note**
the user can type — i.e. *user-generated content*. Everything else in a report (content code, category,
licence, language, revision, reason, app version, timestamp) is non-personal content metadata already
covered by the "App activity / App info and performance" style declarations, or is derivable content
context rather than a new personal-data type.

So the only substantive addition on each store is: **one optional "other user-generated content" item, not
linked to the user, collected for app functionality, not shared, not used for tracking.**

## Google Play — Data safety form

Path: Play Console → app → *Policy and programs → App content → Data safety*.

Add / confirm under **Data collected** (not "Data shared" — we share nothing):

| Field | Value |
|---|---|
| Data type | **App activity → Other user-generated content** (the optional free-text note) |
| Collected? | Yes |
| Shared? | No |
| Processing | Not processed ephemerally — it is stored (Cloud Firestore) |
| Required or optional? | **Optional** (user may leave the note blank) |
| Purposes | **App functionality** (content-quality triage). Not "Analytics", not "Personalisation", not "Advertising" |
| Linked to the user's identity? | **No** — tied only to a random per-install ID, no account |
| Used to track users? | **No** |

Confirm the pre-existing declarations still hold (unchanged by this feature):

- **Device or other IDs** — the random per-install identifier — Collected: Yes, Shared: No, purpose *App
  functionality / Fraud prevention*, **not** linked to identity, **not** for tracking.
- **App info and performance → Crash logs / Diagnostics** — via Firebase Crashlytics, as already declared.

Do **not** add: Personal info (name, email, address), Location, Contacts, Photos/Media, Financial info.
The note is the only free-text field and the schema (Firestore rules key-whitelist) makes attachments and
extra fields impossible, so nothing else can ride along.

## Apple App Store — App privacy labels

Path: App Store Connect → app → *App Privacy → Edit*.

Add under **Data Not Linked to You**:

| Field | Value |
|---|---|
| Category → Type | **User Content → Other User Content** (the optional free-text note) |
| Purpose | **App Functionality** |
| Used for tracking? | **No** |
| Linked to identity? | **No** (Data *Not* Linked to You) |

Confirm the pre-existing entries still hold (unchanged):

- **Identifiers → Device ID** (the random per-install identifier) — *Data Not Linked to You*, App
  Functionality, not used for tracking.
- **Diagnostics → Crash Data / Performance Data** — via Firebase Crashlytics, as already declared.

## Verification / exit criteria (hard gate for `problem_reporting`)

- [ ] `docs/legal/privacy-en.md` + `docs/legal/privacy-pl.md` problem-report clause merged **and published**
      to the live policy URLs (publish runs on push to `release`; see
      `.github/workflows/publish-privacy-policy.yml`) — confirm the EN and PL pages render the new clause.
- [ ] Google Play Data safety form updated and **published** with the "Other user-generated content" item.
- [ ] Apple App Store privacy labels updated and **published** with "Other User Content".
- [ ] Only then flip the `problem_reporting` Remote Config flag on.

`contact_us` may be enabled independently at any time — it is out of scope for every checkbox above.
