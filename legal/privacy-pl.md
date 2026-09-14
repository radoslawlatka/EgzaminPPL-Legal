---
layout: default
title: Polityka prywatności - Egzamin PPL
---

# Polityka prywatności - Egzamin PPL

**Data wejścia w życie:** 20 czerwca 2026
**Ostatnia aktualizacja:** 9 września 2026

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
- Twoich odpowiedzi egzaminacyjnych, wyników ani historii nauki
- Profili marketingowych ani analitycznych

Aplikacja nie posiada kont użytkowników ani procesu logowania — nie zakładasz konta, nie logujesz się i nigdy nie podajesz adresu e-mail ani hasła. Zamiast tego Aplikacja tworzy w tle niewidoczny, anonimowy identyfikator techniczny, opisany w sekcji 3.

Poza anonimowymi danymi technicznymi i diagnostycznymi opisanymi w sekcji 3, jedyne informacje, które kiedykolwiek opuszczają Twoje urządzenie, to opinie, które samodzielnie postanowisz wysłać: kciuk w górę lub w dół przy wyjaśnieniu, zgłoszenie problemu z pytaniem albo wiadomość e-mail wysłana do nas z Ustawień. Oceny i zgłoszenia problemów są anonimowe — nigdy nie są powiązane z Twoją tożsamością. Wiadomość e-mail siłą rzeczy pochodzi z Twojego własnego adresu i to Ty decydujesz, co dokładnie zawiera. Wszystko to jest dobrowolne i zostało opisane w sekcji 3.

## 3. Co zbieramy

Zbieramy minimalną ilość **anonimowych danych technicznych i diagnostycznych**, aby Aplikacja działała stabilnie, oraz — jeśli zdecydujesz się skorzystać z tych funkcji — Twoje **oceny wyjaśnień** i **zgłoszenia problemów**.

| Dane | Cel | Sposób |
|---|---|---|
| Raporty awarii (stack trace, typ wyjątku, ekran w momencie awarii) | Wykrywanie i naprawa błędów | Firebase Crashlytics |
| Model urządzenia, wersja systemu, wersja aplikacji, język systemu, dostępna pamięć | Odtworzenie awarii na właściwym typie urządzenia | Firebase Crashlytics |
| Firebase Installation ID (losowo generowany UUID na instalację aplikacji) | Grupowanie raportów awarii z tej samej instalacji, aby ocenić skalę problemu | Firebase Installations (wymagane przez Crashlytics) |
| Identyfikator instalacji (odrębny, losowo generowany UUID na instalację aplikacji) | Przypisanie Twoich dobrowolnych ocen i zgłoszeń problemów do jednej instalacji — aby zliczyć jeden głos na instalację i zapobiegać manipulacji tymi danymi — bez identyfikowania Ciebie | Przechowywany lokalnie i wysyłany z każdą oceną lub zgłoszeniem do Cloud Firestore |
| Anonimowy identyfikator uwierzytelniający (losowo generowany identyfikator na instalację aplikacji, tworzony automatycznie w tle i niewidoczny dla Ciebie) | Stała anonimowa tożsamość, na której mogą opierać się przyszłe funkcje Aplikacji — na przykład powiązanie opcjonalnego zakupu subskrypcji — bez identyfikowania Ciebie | Firebase Authentication |
| Twoje oceny wyjaśnień (kierunek oceny, którego wyjaśnienia dotyczy — kod treści, język i wersja — oraz z którego ekranu została wystawiona) | Prywatny sygnał jakości, który wskazuje nam, które wyjaśnienia wymagają poprawy | Cloud Firestore (kolekcja "contentVotes") |
| Zgłoszenie problemu, które postanowisz wysłać (wybrany powód, której treści dotyczy — kod treści, kategoria, licencja, język i wersja wyjaśnienia — wersja aplikacji oraz opcjonalna notatka, którą wpiszesz) | Wykrywanie i naprawa błędów w treści — błędnego klucza odpowiedzi, wadliwego wyjaśnienia lub błędu w pytaniu | Cloud Firestore (kolekcja "problemReports") |

Te dane **nie są powiązane z Twoją tożsamością**. Firebase Installation ID, identyfikator instalacji oraz anonimowy identyfikator uwierzytelniający to odrębne, losowe identyfikatory przypisane do konkretnej instalacji i żaden z nich nie jest połączony z Twoim kontem Google, identyfikatorem reklamowym ani żadnym innym identyfikatorem. Firebase Installation ID i identyfikator instalacji są resetowane przy odinstalowaniu Aplikacji lub wyczyszczeniu jej danych. Anonimowy identyfikator uwierzytelniający jest na Androidzie resetowany w ten sam sposób; na iOS przechowuje go systemowy pęk kluczy (Keychain), więc może przetrwać ponowną instalację — do czasu usunięcia danych pęku kluczy z urządzenia.

Aplikacja wykonuje anonimowe zapytania sieciowe do **Cloud Firestore**, aby pobrać pytania egzaminacyjne, kategorie i wyjaśnienia; zapytania te mogą zawierać anonimowy token uwierzytelniający — wskazuje on anonimową tożsamość danej instalacji, a nie Ciebie — i nie zawierają żadnych informacji, które identyfikują Cię osobiście. Gdy oceniasz wyjaśnienie lub wysyłasz zgłoszenie problemu, Aplikacja dodatkowo zapisuje tę ocenę lub zgłoszenie w Cloud Firestore w sposób opisany powyżej — zapis ten zawiera Twój identyfikator instalacji oraz dane oceny lub zgłoszenia, ale nie zawiera informacji, które identyfikują Cię osobiście. Cały ruch do Cloud Firestore jest chroniony przez Firebase App Check (Play Integrity na Androidzie, App Attest na iOS).

Twoje oceny i zgłoszenia problemów pozostają prywatne. Widzimy je tylko my i wyłącznie po to, by poprawiać i ulepszać treści w Aplikacji; nigdy nie są pokazywane innym użytkownikom.

**Opcjonalna notatka w zgłoszeniu problemu** to dowolny tekst, który wpisujesz samodzielnie. Ponieważ nie mamy możliwości odpowiedzi na zgłoszenie, opisz w niej wyłącznie problem z treścią i **nie** podawaj żadnych danych osobowych (takich jak imię, nazwisko czy adres e-mail). Notatki ze zgłoszeń nigdy nie są pokazywane innym użytkownikom.

**Wysyłanie opinii e-mailem.** Ustawienia zawierają opcję "Wyślij opinię" do zgłaszania błędów aplikacji i przesyłania sugestii. Otwiera ona Twoją własną aplikację pocztową z wiadomością zaadresowaną do nas, wstępnie wypełnioną wersją aplikacji, platformą, wybraną licencją i językiem, abyśmy rozumieli kontekst. Wiadomość ta jest wysyłana przez Twojego własnego dostawcę poczty i nie jest przechowywana w naszych bazach danych; w odróżnieniu od zgłoszenia problemu pochodzi ona z Twojego adresu e-mail, ponieważ właśnie dzięki temu możemy Ci odpowiedzieć. Przed wysłaniem widzisz i kontrolujesz całą jej treść.

## 4. Podstawa prawna (RODO)

Tam gdzie ma zastosowanie RODO, podstawą prawną przetwarzania każdej z tych kategorii danych jest nasz **prawnie uzasadniony interes** (art. 6 ust. 1 lit. f RODO):

- **Dane diagnostyczne** — nasz interes polegający na utrzymaniu sprawnej i wolnej od błędów Aplikacji.
- **Anonimowy identyfikator uwierzytelniający** — nasz interes polegający na prowadzeniu i rozwijaniu Aplikacji, w tym na przygotowaniu podstawy dla przyszłych funkcji, takich jak powiązanie opcjonalnego zakupu subskrypcji z daną instalacją.
- **Oceny wyjaśnień (identyfikator instalacji i dane oceny)** — nasz interes polegający na poprawie jakości wyjaśnień w Aplikacji oraz na zapobieganiu manipulacji tymi ocenami.
- **Zgłoszenia problemów (identyfikator instalacji, dane zgłoszenia i opcjonalna notatka)** — nasz interes polegający na wykrywaniu i naprawianiu błędów w treściach Aplikacji oraz na zapobieganiu manipulacji tymi zgłoszeniami.
- **Opinie przesłane do nas e-mailem** — nasz interes polegający na zapoznaniu się z opinią, którą postanowisz wysłać, i udzieleniu na nią odpowiedzi.

Wyważyliśmy ten interes wobec Twojej prywatności. Przetwarzanie wykorzystuje tylko te dane, które są potrzebne, opiera się na losowym identyfikatorze przypisanym do instalacji, a nie na danych, które Cię identyfikują, chroni prywatność Twoich ocen i zgłoszeń, a treści, które tworzysz samodzielnie, opuszczają urządzenie wyłącznie wtedy, gdy zdecydujesz się ocenić wyjaśnienie, wysłać zgłoszenie lub napisać do nas. Na tej podstawie uznajemy, że nie narusza ono Twoich interesów, praw ani wolności.

## 5. Kto przetwarza dane

Dane techniczne i diagnostyczne, oceny wyjaśnień oraz zgłoszenia problemów są przetwarzane w naszym imieniu przez:

- **Google LLC / Google Ireland Limited** - dostawca usług Firebase Crashlytics, Firebase Installations, Firebase Authentication, Cloud Firestore oraz Firebase App Check. Praktyki prywatności Google są opisane na stronach [policies.google.com/privacy](https://policies.google.com/privacy) oraz [firebase.google.com/support/privacy](https://firebase.google.com/support/privacy).

Dane mogą być przesyłane i przechowywane na serwerach Google poza Europejskim Obszarem Gospodarczym. Transfery te są objęte umową powierzenia przetwarzania danych Google oraz standardowymi klauzulami umownymi zatwierdzonymi przez Komisję Europejską.

Jeśli skontaktujesz się z nami za pomocą opcji "Wyślij opinię", Twoja wiadomość jest obsługiwana w zwykły sposób przez Twojego dostawcę poczty oraz naszego, a my wykorzystujemy ją wyłącznie do zapoznania się z jej treścią i udzielenia odpowiedzi.

## 6. Okres przechowywania

- Raporty awarii i powiązane dane diagnostyczne są przechowywane przez Firebase Crashlytics przez **około 90 dni**, a następnie automatycznie usuwane zgodnie z domyślną polityką Firebase.
- Firebase Installation ID jest resetowany przy każdym odinstalowaniu Aplikacji lub wyczyszczeniu jej danych.
- Anonimowy identyfikator uwierzytelniający jest na Androidzie resetowany przy odinstalowaniu Aplikacji lub wyczyszczeniu jej danych. Na iOS przechowuje go systemowy pęk kluczy (Keychain), więc może przetrwać ponowną instalację — do czasu usunięcia danych pęku kluczy z urządzenia.
- Treści egzaminacyjne pobrane do Aplikacji są przechowywane lokalnie w pamięci podręcznej i są usuwane przy odinstalowaniu.
- Oceny wyjaśnień są przechowywane na serwerach Google (Cloud Firestore). Odinstalowanie Aplikacji resetuje Twój lokalny identyfikator instalacji, ale **nie** usuwa głosów, które zostały już wysłane. Każdą ocenę przechowujemy tylko tak długo, jak długo pozostaje ona użytecznym sygnałem jakości dla danego wyjaśnienia, i usuwamy ją, gdy nie jest już potrzebna. Możesz również w każdej chwili samodzielnie usunąć każdą swoją ocenę z poziomu Aplikacji (zob. sekcja 7).
- Zgłoszenia problemów są przechowywane na serwerach Google (Cloud Firestore) w ten sam sposób. Każde zgłoszenie przechowujemy tylko tak długo, jak długo jest ono użyteczne do naprawy treści, której dotyczy, i usuwamy je podczas okresowych czyszczeń, gdy zostanie już rozpatrzone. Odinstalowanie Aplikacji resetuje Twój lokalny identyfikator instalacji, ale **nie** usuwa zgłoszeń, które zostały już wysłane.

## 7. Twoje prawa

Na podstawie RODO masz prawo do:

- Dostępu do danych, które przechowujemy na Twój temat
- Sprostowania lub usunięcia danych
- Sprzeciwu wobec przetwarzania opartego na prawnie uzasadnionym interesie
- Złożenia skargi do organu nadzorczego (w Polsce: Urząd Ochrony Danych Osobowych, [uodo.gov.pl](https://uodo.gov.pl/))

Ponieważ nie zbieramy danych, które pozwalają Cię osobiście zidentyfikować, w praktyce nie jesteśmy w stanie odnaleźć rekordów konkretnej osoby na żądanie. Najskuteczniejszą formę usunięcia danych możesz wykonać samodzielnie: **odinstaluj Aplikację** (lub wyczyść jej dane w ustawieniach systemu). Spowoduje to usunięcie lokalnej pamięci podręcznej i zresetowanie Firebase Installation ID, a wszelkie pozostałe dane diagnostyczne wygasną w ciągu około 90 dni. Anonimowy identyfikator uwierzytelniający jest w ten sposób resetowany na Androidzie; na iOS pozostaje w systemowym pęku kluczy do czasu usunięcia danych pęku kluczy z urządzenia.

Ocenę możesz wycofać w każdej chwili, bezpośrednio w Aplikacji: ponownie dotknij podświetlonego kciuka, a ten głos zostanie usunięty z naszych serwerów. Zgłoszeń problemów nie można wycofywać pojedynczo — ze względu na Twoją prywatność Aplikacja nie ma prawa odczytywać wysłanych zgłoszeń — ale są one usuwane podczas naszych okresowych czyszczeń (zob. sekcja 6), a odinstalowanie Aplikacji odcina każde pozostałe zgłoszenie od Ciebie, co wyjaśniamy poniżej.

Odinstalowanie Aplikacji resetuje również Twój lokalny identyfikator instalacji. Wysłane wcześniej oceny i zgłoszenia problemów pozostają zapisane, ale są powiązane wyłącznie z tym losowym, lokalnym identyfikatorem, który nigdzie indziej nie jest przechowywany — gdy go już nie ma, tych ocen i zgłoszeń nie da się już powiązać z Tobą ani Twoim urządzeniem za jego pośrednictwem. Nie dotyczy to anonimowego identyfikatora uwierzytelniającego, opisanego w sekcjach 3 i 6, który jest odrębny od identyfikatora instalacji: na Androidzie resetuje się w ten sam sposób, ale na iOS przechowuje go systemowy pęk kluczy (Keychain), więc może przetrwać ponowną instalację — do czasu usunięcia danych pęku kluczy z urządzenia.

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
