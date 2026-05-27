---
layout: default
title: Polityka prywatności — Egzamin PPL
---

# Polityka prywatności — Egzamin PPL

**Data wejścia w życie:** 27 maja 2026
**Ostatnia aktualizacja:** 27 maja 2026

Niniejsza Polityka prywatności wyjaśnia, jakie dane zbiera aplikacja mobilna **Egzamin PPL** ("Aplikacja"), w jakim celu oraz jakie prawa Ci przysługują. Aplikacja jest publikowana na Android (Google Play) oraz iOS (App Store).

Wersja angielska tej polityki jest dostępna pod adresem [/privacy-en](./privacy-en).

## 1. Kto jest administratorem

Administratorem danych dla Aplikacji jest:

**Radosław Łątka**
E-mail: [radoslaw.latka.dev@gmail.com](mailto:radoslaw.latka.dev@gmail.com)

Aplikacja jest niezależnym, niekomercyjnym projektem wspomagającym przygotowanie do egzaminu teoretycznego PPL (Private Pilot Licence) prowadzonego przez Urząd Lotnictwa Cywilnego (ULC).

## 2. Czego **nie** zbieramy

Aplikacja **nie** zbiera, nie przechowuje ani nie przesyła:

- Twojego imienia, nazwiska, adresu e-mail, numeru telefonu ani żadnych danych logowania
- Twojej lokalizacji (GPS ani innej)
- Twoich kontaktów, zdjęć, mikrofonu, kamery ani plików
- Twojego identyfikatora reklamowego (Advertising ID)
- Żadnych odpowiedzi, wyników ani historii nauki, które mogłyby Cię zidentyfikować
- Profili marketingowych ani analitycznych

Aplikacja nie posiada kont użytkowników ani procesu logowania.

## 3. Co zbieramy

Zbieramy minimalną ilość **anonimowych danych diagnostycznych**, aby Aplikacja działała stabilnie.

| Dane | Cel | Sposób |
|---|---|---|
| Raporty awarii (stack trace, typ wyjątku, ekran w momencie awarii) | Wykrywanie i naprawa błędów | Firebase Crashlytics |
| Model urządzenia, wersja systemu, wersja aplikacji, język systemu, dostępna pamięć | Odtworzenie awarii na właściwym typie urządzenia | Firebase Crashlytics |
| Firebase Installation ID (losowo generowany UUID na instalację aplikacji) | Grupowanie raportów awarii z tej samej instalacji, aby ocenić skalę problemu | Firebase Installations (wymagane przez Crashlytics) |

Te dane **nie są powiązane z Twoją tożsamością**. Firebase Installation ID to losowy identyfikator przypisany do konkretnej instalacji; jest resetowany przy odinstalowaniu Aplikacji lub wyczyszczeniu jej danych i nie jest połączony z Twoim kontem Google, identyfikatorem reklamowym ani żadnym innym identyfikatorem.

Aplikacja wykonuje też anonimowe zapytania sieciowe do **Cloud Firestore**, aby pobrać pytania egzaminacyjne, kategorie i wyjaśnienia. Zapytania te są chronione przez Firebase App Check (Play Integrity na Androidzie, App Attest na iOS) i nie zawierają żadnych informacji identyfikujących użytkownika.

## 4. Podstawa prawna (RODO)

Tam gdzie ma zastosowanie RODO, podstawą prawną przetwarzania danych diagnostycznych jest nasz **prawnie uzasadniony interes** polegający na utrzymaniu sprawnej i wolnej od błędów Aplikacji (art. 6 ust. 1 lit. f RODO). Przetwarzanie ograniczone jest do zakresu niezbędnego do dostarczenia działającego produktu.

## 5. Kto przetwarza dane

Dane diagnostyczne są przetwarzane w naszym imieniu przez:

- **Google LLC / Google Ireland Limited** — dostawca usług Firebase Crashlytics, Firebase Installations, Cloud Firestore oraz Firebase App Check. Praktyki prywatności Google są opisane na stronach [policies.google.com/privacy](https://policies.google.com/privacy) oraz [firebase.google.com/support/privacy](https://firebase.google.com/support/privacy).

Dane mogą być przesyłane i przechowywane na serwerach Google poza Europejskim Obszarem Gospodarczym. Transfery te są objęte umową powierzenia przetwarzania danych Google oraz standardowymi klauzulami umownymi zatwierdzonymi przez Komisję Europejską.

## 6. Okres przechowywania

- Raporty awarii i powiązane dane diagnostyczne są przechowywane przez Firebase Crashlytics przez **około 90 dni**, a następnie automatycznie usuwane zgodnie z domyślną polityką Firebase.
- Firebase Installation ID jest resetowany przy każdym odinstalowaniu Aplikacji lub wyczyszczeniu jej danych.
- Treści egzaminacyjne pobrane do Aplikacji są przechowywane lokalnie w pamięci podręcznej i są usuwane przy odinstalowaniu.

## 7. Twoje prawa

Na podstawie RODO masz prawo do:

- Dostępu do danych, które przechowujemy na Twój temat
- Sprostowania lub usunięcia danych
- Sprzeciwu wobec przetwarzania opartego na prawnie uzasadnionym interesie
- Złożenia skargi do organu nadzorczego (w Polsce: Urząd Ochrony Danych Osobowych, [uodo.gov.pl](https://uodo.gov.pl/))

Ponieważ nie zbieramy danych, które pozwalają Cię osobiście zidentyfikować, w praktyce nie jesteśmy w stanie odnaleźć rekordów konkretnej osoby na żądanie. Najskuteczniejszą formą usunięcia danych możesz wykonać samodzielnie: **odinstaluj Aplikację** (lub wyczyść jej dane w ustawieniach systemu). Spowoduje to usunięcie lokalnej pamięci podręcznej i zresetowanie Firebase Installation ID, a wszelkie pozostałe dane diagnostyczne wygasną w ciągu około 90 dni.

W razie pytań lub chęci skorzystania z któregoś z praw napisz na [radoslaw.latka.dev@gmail.com](mailto:radoslaw.latka.dev@gmail.com).

## 8. Dzieci

Aplikacja jest przeznaczona dla osób przygotowujących się do uzyskania licencji lotniczej i **nie jest skierowana do dzieci poniżej 16. roku życia**. Świadomie nie zbieramy danych od dzieci.

## 9. Reklamy i śledzenie

Aplikacja **nie zawiera reklam, zewnętrznych narzędzi śledzących ani pakietów SDK do analityki** poza Firebase Crashlytics, opisanym powyżej.

## 10. Zmiany w polityce

W przypadku zmiany niniejszej polityki zaktualizujemy datę "Ostatnia aktualizacja" powyżej. Jeśli zmiany będą istotne, odnotujemy je w opisie aktualizacji Aplikacji na Google Play i App Store.

## 11. Kontakt

W sprawach związanych z prywatnością:

**Radosław Łątka**
E-mail: [radoslaw.latka.dev@gmail.com](mailto:radoslaw.latka.dev@gmail.com)
