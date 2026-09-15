# Dzień 35 – AI Asystent Skrzynki Firmowej

## Pierwszy projekt z wykorzystaniem AI

## Projekt do portfolio: automatyczna obsługa przychodzących wiadomości

Firma, do której codziennie przychodzi kilkadziesiąt maili: klient pyta o ofertę, ktoś chce umówić spotkanie, klient zgłasza problem, ktoś wysyła fakturę, ktoś pyta o status zamówienia, wiadomość jest pilna, wiadomość jest kompletnie nieistotna. Człowiek musi to wszystko przeczytać, zrozumieć, zdecydować co z tym zrobić, oznaczyć wiadomość i często jeszcze przygotować odpowiedź.

Ten Zap wykonuje pierwszą część tej pracy automatycznie:

```
NOWY EMAIL
    ↓
  Gmail
    ↓
AI analizuje mail
    ↓
┌────────────┼────────────┐
↓            ↓            ↓
PILNE      KLIENT     SPOTKANIE
↓            ↓            ↓
oznacz    przygotuj   przygotuj
wiadomość  odpowiedź   odpowiedź
↓            ↓            ↓
└────────────┼────────────┘
    ↓
Gmail — DRAFT
    ↓
człowiek zatwierdza
```

Kluczowe założenie projektu: żadna odpowiedź nie jest wysyłana automatycznie do klientów. AI przygotowuje wyłącznie szkic (draft) w Gmailu, a człowiek go zatwierdza i wysyła ręcznie. To dużo bardziej realistyczne biznesowo niż pełna automatyzacja wysyłki.

Cel końcowy do pokazania: "Zbudowałem w Zapierze automatycznego asystenta skrzynki firmowej. Analizuje przychodzące wiadomości, klasyfikuje je według typu i pilności, a następnie przygotowuje odpowiedź w Gmailu jako draft do zatwierdzenia przez pracownika."

---

## Krok 1 – Przygotowanie Gmaila

Utworzone etykiety: AUTO/Pilne, AUTO/Klient, AUTO/Spotkanie, AUTO/Faktura, AUTO/Inne.

Etykiety dają namacalny, wizualny dowód działania automatyzacji — po wykonaniu Zapa można wejść do Gmaila i pokazać konkretną wiadomość oznaczoną jako pilna, co jest dużo bardziej przekonujące niż dane siedzące wyłącznie w arkuszu.

## Krok 2 – Testowe maile

Pięć wiadomości testowych wysłanych do siebie, pokrywających różne kategorie:

1. Klient — "Prośba o ofertę" (zapytanie o cennik dla ~50 pracowników)
2. Pilny — "PILNE - problem z zamówieniem" (niedostarczone zamówienie nr 48392)
3. Spotkanie — "Spotkanie w przyszłym tygodniu" (propozycja terminu)
4. Faktura — "Faktura za sierpień" (mail z załącznikiem)
5. Nieistotne — "Newsletter" (informacja niewymagająca reakcji)

## Krok 3 – Nowy Zap

Nazwa: AI Email Assistant - Portfolio

## Krok 4 – Trigger

Gmail, Event: New Email, Label/Mailbox: Inbox (bez dodatkowych filtrów na start).

## Krok 5 – AI by Zapier: Analyze and Return Data

To moment, w którym workflow przestaje być prostym "A → B" — AI dostaje treść maila i podejmuje decyzję na jego podstawie.

## Krok 6 – Input dla AI

- Email subject → pole Subject z Gmaila
- Email body → pole Body Plain / treść wiadomości z Gmaila

## Krok 7 – Prompt

```
Jesteś asystentem obsługującym firmową skrzynkę email.

Przeanalizuj otrzymaną wiadomość i określ:

1. Typ wiadomości:
- klient
- spotkanie
- faktura
- problem
- inne

2. Priorytet:
- niski
- normalny
- wysoki
- pilny

3. Czy wiadomość wymaga odpowiedzi:
- tak
- nie

4. Napisz krótkie podsumowanie wiadomości w 1-2 zdaniach.

5. Jeżeli wiadomość wymaga odpowiedzi, przygotuj profesjonalny draft odpowiedzi po polsku.

Nie wymyślaj informacji, których nie ma w wiadomości.
Nie wysyłaj odpowiedzi — przygotuj jedynie tekst draftu.

Email subject:
{{Email subject}}

Email body:
{{Email body}}
```

Pola {{Email subject}} i {{Email body}} wstawiane bezpośrednio z danych poprzedniego kroku przez interfejs Zapiera, nie wpisywane ręcznie jako tekst.

## Krok 8 – Output fields

Message Type — Category / Single select, Required — klasyfikacja jako jedna z: klient, spotkanie, faktura, problem, inne.

Priority — Category / Single select, Required — klasyfikacja pilności jako: niski, normalny, wysoki, pilny.

Requires Response — Boolean, Required — czy wiadomość wymaga ludzkiej odpowiedzi.

Message Summary — Text, Required — podsumowanie wiadomości w 1-2 zdaniach po polsku.

Response Draft — Text — proponowana treść odpowiedzi (puste, jeśli odpowiedź niepotrzebna).

Ważne wyjaśnienie napotkane w trakcie budowy: typ pola "Category / Single select" nie wymaga ręcznego zdefiniowania listy wartości w interfejsie Output fields — to wyłącznie informacja o kształcie danych, jakich AI ma zwrócić jedną wartość tekstową. Faktyczne, konkretne wartości (np. "klient", "faktura", "pilny") są zdefiniowane w treści promptu z Kroku 7, nie w konfiguracji Output fields.

## Krok 9 – Test AI

Test na przykładzie "PILNE - problem z zamówieniem" — oczekiwany wynik zbliżony do:

category: problem, priority: pilny, requires_reply: true, summary: klient zgłasza problem z niedostarczonym zamówieniem nr 48392 i oczekuje szybkiego wyjaśnienia sytuacji, draft_reply: gotowa, uprzejma odpowiedź przepraszająca za sytuację i informująca o sprawdzeniu statusu zamówienia.

Zasada przyjęta na tym etapie: jeśli AI zwraca coś nietrafnego, poprawia się prompt zamiast przechodzić dalej z niedopracowaną klasyfikacją.

## Krok 10 – Paths (routing)

Paths by Zapier, cztery ścieżki: Path A — Pilne, Path B — Klient, Path C — Spotkanie, Path D — Faktura. Ścieżka "Inne" pominięta jako niepotrzebna.

Warunki:

Path A: PILNE — pole priority (z kroku AI), warunek Text → Exactly matches, wartość pilny.
Path B: KLIENT — pole category (z kroku AI), warunek Text → Exactly matches, wartość klient.
Path C: SPOTKANIE — pole category (z kroku AI), warunek Text → Exactly matches, wartość spotkanie.
Path D: FAKTURA — pole category (z kroku AI), warunek Text → Exactly matches, wartość faktura.

## Krok 11 – Akcje wewnątrz każdej ścieżki

W każdej z czterech ścieżek te same dwie akcje, różniące się wyłącznie etykietą w pierwszej z nich:

Akcja 1 — Gmail → Add Label to Email
- Label: właściwa dla danej ścieżki (AUTO/Pilne, AUTO/Klient, AUTO/Spotkanie, AUTO/Faktura)
- Message: ID (nie Thread ID!) z Kroku 1 (New Email in Gmail)

Akcja 2 — Gmail → Create Draft Reply
- Thread: Thread ID z Kroku 1
- To: From Address z Kroku 1 — pole konieczne, żeby Gmail poprawnie "zaczepił" i podpiął odpowiedź pod istniejący wątek
- Body: draft_reply (Response Draft) z kroku AI

Nigdy nie wybiera się Send Email — wyłącznie tworzenie szkicu. Cel: AI → draft → człowiek → SEND, nie AI → klient bezpośrednio.

---

## Napotkane problemy i naprawy

### Problem 1: draft tworzył się jako całkiem nowa wiadomość, niepowiązana z oryginalnym mailem

Przyczyna: wybrana została zwykła akcja Gmaila Create Draft (tworząca nową, samodzielną wiadomość, wymagającą pól To/Subject od zera), zamiast akcji tworzącej odpowiedź w istniejącym wątku.

Naprawa: zmiana akcji na Create Draft Reply (w niektórych wersjach Zapiera: Reply to Email z opcją Draft zamiast natychmiastowej wysyłki).

### Problem 2: nie wiadomo skąd wziąć właściwe Thread ID / Message ID

Pole wyboru wartości w Zapierze domyślnie otwierało zakładkę Static — listę sztywno zapisanych, przykładowych/starych wiadomości. Wybranie ID z tej listy sprawiałoby, że Zapier zawsze odnosiłby się do tej samej, jednej starej wiadomości, niezależnie od tego, jaki mail faktycznie uruchomił workflow.

Naprawa: przełączenie zakładki z Static na Dynamic i wybranie danych z Kroku 1 (trigger Gmail) — konkretnie pola ID (dla Message) lub Thread Id (dla Thread). Dzięki temu Zapier za każdym razem dynamicznie pobiera identyfikator tej konkretnej wiadomości, która akurat uruchomiła Zapa.

### Problem 3: draft trafiał do złej kategorii (np. Faktura zamiast Pilne)

Przyczyna: w Kroku 1 (trigger) wciąż załadowany był przykładowy mail z innego testu (np. "Faktura za sierpień"), mimo że intencją było przetestowanie ścieżki Pilne. AI analizowało dokładnie ten mail, który akurat siedział w danych triggera, niezależnie od tego, którą ścieżkę próbowano testować.

Naprawa — poprawna procedura testowania konkretnej ścieżki:
1. Wrócić do Kroku 1 (Trigger), kliknąć Find new records, żeby pobrać najnowsze maile ze skrzynki (w tym właściwy mail testowy).
2. Wybrać z listy dokładnie ten mail, który ma być przetestowany w danym momencie.
3. Przejść do kroku AI i kliknąć Re-test step, żeby upewnić się, że AI zwróciło oczekiwaną klasyfikację (np. priority: pilny).
4. Dopiero wtedy przejść do testowania konkretnej ścieżki (warunku Path) i jej akcji.

### Duplikowanie akcji między ścieżkami

Skoro każda ścieżka odwołuje się do tych samych pól dynamicznych (Thread Id, From Address z Kroku 1, oraz draft_reply z kroku AI), całą akcję Create Draft Reply można bezpiecznie zduplikować i wkleić do kolejnych ścieżek bez zmian. Jedyne, co wymaga ręcznej zmiany po zduplikowaniu, to wartość pola Label w akcji Add Label to Email — musi odpowiadać danej ścieżce (np. AUTO/Klient zamiast AUTO/Pilne).

### Problem 4: etykieta nie pojawiała się w Gmailu mimo testu w Zapierze

Lista kontrolna zastosowana do zdiagnozowania:
1. Sprawdzenie pola Message w Add Label to Email — musi zawierać ID konkretnej wiadomości, nie Thread ID ani treść maila.
2. Ponowny test i sprawdzenie komunikatu — czy Zapier pokazał zielone potwierdzenie sukcesu.
3. Odświeżenie Gmaila (F5) — czasami interfejs Gmaila nie aktualizuje się natychmiastowo.
4. Zgodność nazwy etykiety — wybieranie etykiety z rozwijanej listy w Zapierze, zamiast wpisywania nazwy ręcznie jako tekst, co eliminuje ryzyko literówki lub niezgodności wielkości liter.

### Ten sam problem Static/Dynamic powrócił przy konfigurowaniu Add Label to Email

Dokładnie ta sama pułapka co w Problemie 2 pojawiła się ponownie przy wybieraniu pola Message w kolejnej akcji — domyślnie otwierała się zakładka Static z listą starych, przykładowych wiadomości. Rozwiązanie identyczne: przełączenie na Dynamic i wybór pola ID z Kroku 1. Dodatkowo, żeby zobaczyć na tej liście najnowsze, dopiero co wysłane maile testowe, konieczne było ponowne kliknięcie Find new records w Kroku 1.

---

## Pełny schemat workflowa

```
Gmail Trigger (New Email, Inbox)
        │
        ▼
AI by Zapier — Analyze and Return Data
(Input: Email subject, Email body
 Output: Message Type, Priority,
 Requires Response, Message Summary,
 Response Draft)
        │
        ▼
    Paths by Zapier
   ┌────┼────┬────────┐
   ▼    ▼    ▼        ▼
 Pilne Klient Spotkanie Faktura
 (priority (category  (category  (category
  = pilny)  = klient)  = spotkanie) = faktura)
   │    │    │        │
   ▼    ▼    ▼        ▼
 Add Label to Email (etykieta właściwa dla ścieżki, Message ID z Kroku 1)
   │    │    │        │
   ▼    ▼    ▼        ▼
 Create Draft Reply (Thread ID + To=From Address z Kroku 1, Body=draft_reply z AI)
```

---

## Czego się nauczyłeś

- budowania pierwszego workflowa wykorzystującego AI do klasyfikacji i generowania treści (AI by Zapier — Analyze and Return Data)
- projektowania promptu, który jednocześnie definiuje kategorie klasyfikacji i instruuje model, czego nie robić (np. "nie wymyślaj informacji", "nie wysyłaj odpowiedzi")
- że typ pola Output (np. Category/Single select) określa wyłącznie kształt oczekiwanych danych, a nie samą listę dopuszczalnych wartości — te definiuje się w treści promptu
- kluczowej różnicy między zakładką Static (sztywne, przykładowe dane) a Dynamic (rzeczywiste dane z konkretnego kroku) przy wybieraniu wartości pól w Zapierze — i że pomylenie ich prowadzi do trwałego, błędnego przypięcia do jednej, starej wiadomości
- że tworzenie draftu odpowiedzi w istniejącym wątku wymaga innej akcji niż tworzenie zwykłego, nowego szkicu, oraz że wymaga zarówno Thread ID, jak i adresu w polu To
- poprawnej procedury testowania pojedynczej ścieżki w rozgałęzionym Zapie: odświeżenie danych triggera (Find new records), ponowny test kroku AI, dopiero potem test konkretnej ścieżki i jej akcji
- że akcje odwołujące się wyłącznie do danych dynamicznych z wcześniejszych, wspólnych kroków można bezpiecznie duplikować między ścieżkami, zmieniając tylko to, co faktycznie ma się różnić (np. etykietę)
- świadomego zaprojektowania granicy między automatyzacją a decyzją człowieka: AI przygotowuje, człowiek zatwierdza i wysyła — bez automatycznej wysyłki odpowiedzi bezpośrednio do klientów
