---
layout: default
title: Privacy Policy — Egzamin PPL
---

# Privacy Policy — Egzamin PPL

**Effective date:** 27 May 2026
**Last updated:** 27 May 2026

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
- Any answers, results, or personal study history that could identify you
- Marketing or analytics profiles

The App has no user accounts and no login flow.

## 3. What we do collect

We collect a minimal amount of **anonymous diagnostic data** to keep the App stable.

| Data | Purpose | How |
|---|---|---|
| Crash reports (stack trace, exception type, app screen at time of crash) | Detect and fix bugs | Firebase Crashlytics |
| Device model, OS version, app version, system language, available memory | Reproduce crashes on the correct device class | Firebase Crashlytics |
| Firebase Installation ID (a randomly generated UUID per app install) | Group crash reports from the same installation so we can tell whether a bug affects one user or many | Firebase Installations (required by Crashlytics) |

This data is **not linked to your identity**. The Firebase Installation ID is a per-install random identifier; it is reset when you uninstall the App or clear its data, and it is not connected to your Google account, advertising ID, or any other identifier.

The App also makes anonymous network requests to **Cloud Firestore** to download exam questions, categories, and explanations. These requests are protected by Firebase App Check (Play Integrity on Android, App Attest on iOS) and do not include any user-identifying information.

## 4. Legal basis (GDPR)

Where the GDPR applies, the legal basis for processing diagnostic data is our **legitimate interest** in keeping the App functional and free of bugs (Art. 6(1)(f) GDPR). The processing is limited to what is strictly necessary to deliver the App as a working product.

## 5. Who processes the data

Diagnostic data is processed on our behalf by:

- **Google LLC / Google Ireland Limited** — provider of Firebase Crashlytics, Firebase Installations, Cloud Firestore, and Firebase App Check. Google's privacy practices are described at [policies.google.com/privacy](https://policies.google.com/privacy) and [firebase.google.com/support/privacy](https://firebase.google.com/support/privacy).

Data may be transferred to and stored on Google servers outside the European Economic Area. Such transfers are governed by Google's Data Processing Addendum and Standard Contractual Clauses approved by the European Commission.

## 6. Retention

- Crash reports and associated diagnostic data are retained by Firebase Crashlytics for **approximately 90 days** and then automatically deleted, in accordance with Firebase's default retention policy.
- The Firebase Installation ID is reset whenever you uninstall the App or clear its storage.
- Exam content downloaded to your device is cached locally and is removed when you uninstall the App.

## 7. Your rights

Under the GDPR you have the right to:

- Request access to the data we hold about you
- Request correction or deletion
- Object to processing based on legitimate interest
- Lodge a complaint with a supervisory authority (in Poland: Urząd Ochrony Danych Osobowych, [uodo.gov.pl](https://uodo.gov.pl/))

Because we do not collect any data that identifies you personally, in practice we cannot locate a specific person's records on request. You can still exercise the most effective form of deletion yourself: **uninstall the App** (or clear its data in system settings). This deletes the local cache and resets the Firebase Installation ID, after which any remaining diagnostic data ages out within ~90 days.

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
