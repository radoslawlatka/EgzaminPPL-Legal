---
layout: default
title: Privacy Policy - Egzamin PPL
---

# Privacy Policy - Egzamin PPL

**Effective date:** 20 June 2026
**Last updated:** 7 July 2026

This Privacy Policy explains what data the **Egzamin PPL** mobile application ("the App") collects, why, and what your rights are. The App is published for Android (Google Play) and iOS (App Store).

A Polish version of this policy is available at [/privacy-pl](./privacy-pl).

## 1. Who we are

The data controller for the App is:

**Radosław Łątka**
Email: [radoslaw.latka.dev@gmail.com](mailto:radoslaw.latka.dev@gmail.com)

The App is an independent, non-commercial project that helps users prepare for the Polish PPL (Private Pilot Licence) theoretical exam administered by the Polish Civil Aviation Authority (ULC).

## 2. What we do *not* collect

The App does **not** collect, store, or transmit:

- Your name, email address, phone number, or any account credentials
- Your location (GPS or otherwise)
- Your contacts, photos, microphone, camera, or files
- Your Advertising ID
- Your exam answers, scores, or study history
- Marketing or analytics profiles

The App has no user accounts and no login flow.

Beyond the anonymous diagnostic data described in section 3, the only information that ever leaves your device is feedback you actively choose to send: a thumbs-up or thumbs-down on an explanation, a problem report about a question, or an email you send us from Settings. Votes and problem reports are anonymous — never linked to your identity. An email, by its nature, comes from your own address, and you decide exactly what it contains. All of this is optional and is described in section 3.

## 3. What we do collect

We collect a minimal amount of **anonymous diagnostic data** to keep the App stable and — if you choose to use these features — your **explanation votes** and **problem reports**.

| Data | Purpose | How |
|---|---|---|
| Crash reports (stack trace, exception type, app screen at time of crash) | Detect and fix bugs | Firebase Crashlytics |
| Device model, OS version, app version, system language, available memory | Reproduce crashes on the correct device class | Firebase Crashlytics |
| Firebase Installation ID (a randomly generated UUID per app install) | Group crash reports from the same installation so we can tell whether a bug affects one user or many | Firebase Installations (required by Crashlytics) |
| Install ID (a separate, randomly generated UUID per app install) | Link your optional votes and problem reports to a single installation — so we can count one vote per installation and prevent manipulation of this feedback — without identifying you | Stored locally and sent with each vote or report to Cloud Firestore |
| Your explanation votes (the vote direction, which explanation it relates to — content code, language, and revision — and which screen it was cast from) | A private quality signal that tells us which explanations need improvement | Cloud Firestore (collection "contentVotes") |
| A problem report you choose to send (the reason you pick, which content it concerns — content code, category, licence, language, and explanation revision — the app version, and an optional note you type) | Find and fix content errors — a wrong answer key, a faulty explanation, or a mistake in a question | Cloud Firestore (collection "problemReports") |

This data is **not linked to your identity**. The Firebase Installation ID and the Install ID are separate per-install random identifiers; both are reset when you uninstall the App or clear its data, and neither is connected to your Google account, advertising ID, or any other identifier.

The App makes anonymous network requests to **Cloud Firestore** to download exam questions, categories, and explanations; these requests carry no user-identifying information. When you vote on an explanation or send a problem report, the App additionally writes that vote or report to Cloud Firestore as described above — what we store includes your Install ID and the vote or report data, but no information that identifies you personally. All Cloud Firestore traffic is protected by Firebase App Check (Play Integrity on Android, App Attest on iOS).

Your votes and problem reports stay private. Only we see them, and only to fix and improve the App's content; they are never shown to other users.

**The optional note in a problem report** is free text that you type yourself. Because there is no way for us to reply to a report, please describe only the content problem and do **not** include any personal data (such as your name or email address). Report notes are never shown to other users.

**Sending feedback by email.** Settings includes a "Send feedback" option for app bugs and suggestions. It opens your own email app with a message addressed to us, pre-filled with your app version, platform, selected licence, and language so we can understand the context. That email is sent through your own email provider and is not stored in our databases; unlike a problem report, it does come from your email address, because that is how we can reply to you. You see and control everything it contains before you send it.

## 4. Legal basis (GDPR)

Where the GDPR applies, the legal basis for processing each of these categories of data is our **legitimate interest** (Art. 6(1)(f) GDPR):

- **Diagnostic data** — our interest in keeping the App functional and free of bugs.
- **Explanation votes (Install ID and vote data)** — our interest in improving the quality of the App's explanations and in preventing manipulation of that feedback.
- **Problem reports (Install ID, report details, and any optional note)** — our interest in finding and fixing errors in the App's content and in preventing manipulation of this feedback.
- **Feedback you email us** — our interest in reading and responding to the feedback you choose to send.

We have weighed this interest against your privacy. The processing uses only the data it needs, relies on a random per-install identifier rather than anything that identifies you, keeps your votes and reports private, and happens only when you choose to vote, send a report, or write to us. We therefore consider that it does not override your interests, rights, or freedoms.

## 5. Who processes the data

Diagnostic data, explanation votes, and problem reports are processed on our behalf by:

- **Google LLC / Google Ireland Limited** - provider of Firebase Crashlytics, Firebase Installations, Cloud Firestore, and Firebase App Check. Google's privacy practices are described at [policies.google.com/privacy](https://policies.google.com/privacy) and [firebase.google.com/support/privacy](https://firebase.google.com/support/privacy).

Data may be transferred to and stored on Google servers outside the European Economic Area. Such transfers are governed by Google's Data Processing Addendum and Standard Contractual Clauses approved by the European Commission.

If you contact us through the "Send feedback" email option, your message is handled by your email provider and ours in the ordinary way, and we use it only to read and respond to what you sent.

## 6. Retention

- Crash reports and associated diagnostic data are retained by Firebase Crashlytics for **approximately 90 days** and then automatically deleted, in accordance with Firebase's default retention policy.
- The Firebase Installation ID is reset whenever you uninstall the App or clear its storage.
- Exam content downloaded to your device is cached locally and is removed when you uninstall the App.
- Explanation votes are stored on Google's servers (Cloud Firestore). Uninstalling the App resets your local Install ID but does **not** delete votes you have already sent. We keep a vote only for as long as it remains useful as a quality signal for the relevant explanation, and delete it when it is no longer needed. You can also remove any vote yourself at any time from within the App (see section 7).
- Problem reports are likewise stored on Google's servers (Cloud Firestore). We keep a report only for as long as it is useful for fixing the content it refers to, and delete it during periodic cleanups once it has been reviewed and resolved. Uninstalling the App resets your local Install ID but does **not** delete reports you have already sent.

## 7. Your rights

Under the GDPR you have the right to:

- Request access to the data we hold about you
- Request correction or deletion
- Object to processing based on legitimate interest
- Lodge a complaint with a supervisory authority (in Poland: Urząd Ochrony Danych Osobowych, [uodo.gov.pl](https://uodo.gov.pl/))

Because we do not collect any data that identifies you personally, in practice we cannot locate a specific person's records on request. You can still exercise the most effective form of deletion yourself: **uninstall the App** (or clear its data in system settings). This deletes the local cache and resets the Firebase Installation ID, after which any remaining diagnostic data ages out within ~90 days.

You can withdraw a vote at any time, right in the App: tap the highlighted thumb again and that vote is deleted from our servers. Problem reports cannot be withdrawn one by one — for your privacy, the App is not allowed to read the reports back — but they are deleted during our periodic cleanups (see section 6), and uninstalling the App breaks the last link between you and any remaining report, as explained next.

Uninstalling the App also resets your local Install ID. Any votes or problem reports you have already sent stay stored, but they are linked only to that random local identifier — and once it is gone, with no copy kept anywhere else, they can no longer be traced back to you or your device.

If you have questions or wish to exercise a right, email [radoslaw.latka.dev@gmail.com](mailto:radoslaw.latka.dev@gmail.com).

## 8. Children

The App is intended for users preparing for an aviation licence and is **not directed at children under 16**. We do not knowingly collect data from children.

## 9. Ads and tracking

The App contains **no advertising, no third-party trackers, and no analytics SDKs** beyond Firebase Crashlytics as described above.

## 10. Changes to this policy

If we change this policy we will update the "Last updated" date above and, where the changes are material, we will note them in the App's release notes on Google Play and the App Store.

## 11. Contact

For any privacy-related question:

**Radosław Łątka**
Email: [radoslaw.latka.dev@gmail.com](mailto:radoslaw.latka.dev@gmail.com)
