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

# Day 35 – AI Company Inbox Assistant

## First project utilizing AI

## Portfolio project: automated incoming message handling

In a company receiving dozens of emails daily, tasks vary wildly: a customer asks for a quote, someone wants to schedule a meeting, a client reports an issue, a vendor sends an invoice, a buyer asks about order status, a message is urgent, or an email is completely irrelevant. A human has to read all of these, understand them, decide what to do, label the message, and often draft a response.

This Zap performs the first part of this work automatically:

```
NEW EMAIL
    ↓
  Gmail
    ↓
AI analyzes email
    ↓
┌────────────┼────────────┐
↓            ↓            ↓
URGENT     CLIENT      MEETING
↓            ↓            ↓
label     prepare      prepare
message   response     response
↓            ↓            ↓
└────────────┼────────────┘
    ↓
Gmail — DRAFT
    ↓
Human approves
```

Key project assumption: no response is sent automatically to clients. The AI exclusively prepares a draft in Gmail, which a human then reviews, approves, and sends manually. This is significantly more realistic from a business perspective than full auto-sending.

Final portfolio pitch: "I built an automated inbox assistant in Zapier. It analyzes incoming messages, classifies them by type and urgency, and prepares a response in Gmail as a draft ready for employee approval."

---

## Step 1 – Gmail preparation

Created labels: AUTO/Urgent, AUTO/Client, AUTO/Meeting, AUTO/Invoice, AUTO/Other.

Labels provide tangible, visual proof of the automation at work — after running the Zap, you can open Gmail and display a specific email tagged as urgent, which is far more convincing than data sitting silently in a spreadsheet.

## Step 2 – Test emails

Five test messages sent to oneself, covering different categories:

1. Client — "Quote Request" (pricing inquiry for ~50 employees)
2. Urgent — "URGENT - Order Problem" (undelivered order #48392)
3. Meeting — "Meeting Next Week" (time proposal)
4. Invoice — "August Invoice" (email with an attachment)
5. Irrelevant — "Newsletter" (information requiring no action)

## Step 3 – New Zap

Name: AI Email Assistant - Portfolio

## Step 4 – Trigger

Gmail, Event: New Email, Label/Mailbox: Inbox (no additional filters at the start).

## Step 5 – AI by Zapier: Analyze and Return Data

This is the moment the workflow stops being a simple "A → B" sequence — the AI receives the email content and makes a decision based on it.

## Step 6 – AI Input

- Email subject → Subject field from Gmail
- Email body → Body Plain field / message body from Gmail

## Step 7 – Prompt

```
You are an assistant handling a company email inbox.

Analyze the received message and determine:

1. Message type:
- client
- meeting
- invoice
- problem
- other

2. Priority:
- low
- normal
- high
- urgent

3. Whether the message requires a response:
- yes
- no

4. Write a short summary of the message in 1-2 sentences.

5. If the message requires a response, prepare a professional response draft in Polish.

Do not fabricate information not present in the message.
Do not send the response — prepare only the draft text.

Email subject:
{{Email subject}}

Email body:
{{Email body}}
```

The {{Email subject}} and {{Email body}} fields are inserted dynamically from the previous step's data via the Zapier interface, not typed manually as plain text.

## Step 8 – Output fields

Message Type — Category / Single select, Required — classification as one of: client, meeting, invoice, problem, other.

Priority — Category / Single select, Required — urgency classification as: low, normal, high, urgent.

Requires Response — Boolean, Required — whether the message requires a human response.

Message Summary — Text, Required — 1-2 sentence message summary in Polish.

Response Draft — Text — proposed response content (empty if no response is needed).

Important clarification encountered during setup: the "Category / Single select" field type does not require manually defining a value list in the Output fields interface — it merely describes the shape of the data, indicating that the AI should return a single text value. The actual, concrete values (e.g., "client", "invoice", "urgent") are defined inside the prompt text in Step 7, not in the Output fields configuration.

## Step 9 – AI Testing

Tested using the "URGENT - Order Problem" email — expected output similar to:

category: problem, priority: urgent, requires_reply: true, summary: client reports an issue with undelivered order #48392 and expects a prompt explanation, draft_reply: ready, polite response apologizing for the situation and stating that the order status is being checked.

Rule adopted at this stage: if the AI returns inaccurate data, refine the prompt instead of moving forward with flawed classification.

## Step 10 – Paths (routing)

Paths by Zapier, four paths: Path A — Urgent, Path B — Client, Path C — Meeting, Path D — Invoice. The "Other" path was omitted as unnecessary.

Conditions:

Path A: URGENT — priority field (from AI step), condition Text → Exactly matches, value urgent.
Path B: CLIENT — category field (from AI step), condition Text → Exactly matches, value client.
Path C: MEETING — category field (from AI step), condition Text → Exactly matches, value meeting.
Path D: INVOICE — category field (from AI step), condition Text → Exactly matches, value invoice.

## Step 11 – Actions inside each path

Inside each of the four paths, the same two actions are configured, differing only by the label selected in the first action:

Action 1 — Gmail → Add Label to Email
- Label: appropriate for the given path (AUTO/Urgent, AUTO/Client, AUTO/Meeting, AUTO/Invoice)
- Message: ID (not Thread ID!) from Step 1 (New Email in Gmail)

Action 2 — Gmail → Create Draft Reply
- Thread: Thread ID from Step 1
- To: From Address from Step 1 — required field so Gmail correctly attaches and pairs the reply with the existing thread
- Body: draft_reply (Response Draft) from the AI step

Never select Send Email — only create a draft. Goal: AI → draft → human → SEND, not AI → client directly.

---

## Encountered issues and fixes

### Problem 1: Draft was created as a brand-new email, unlinked from the original message

Cause: The standard Gmail action Create Draft was selected (which creates a new standalone message requiring To/Subject fields from scratch), instead of the action that drafts a reply within an existing thread.

Fix: Changed the action to Create Draft Reply (in some Zapier versions: Reply to Email with the Draft option enabled instead of immediate sending).

### Problem 2: Unsure where to get the correct Thread ID / Message ID

The field value picker in Zapier defaulted to the Static tab — a hardcoded list of sample/old messages. Selecting an ID from this list would cause Zapier to always reference the exact same old email, regardless of which email actually triggered the workflow.

Fix: Switched the tab from Static to Dynamic and selected data from Step 1 (Gmail trigger) — specifically the ID field (for Message) or Thread Id (for Thread). This ensures Zapier dynamically fetches the identifier of the exact message that triggered the Zap every time.

### Problem 3: Draft landed in the wrong category (e.g., Invoice instead of Urgent)

Cause: Step 1 (trigger) still had a sample email from a different test loaded (e.g., "August Invoice"), even though the intention was to test the Urgent path. The AI analyzed whichever email happened to be in the trigger sample data, regardless of which path was being tested.

Fix — correct procedure for testing a specific path:
1. Return to Step 1 (Trigger), click Find new records to pull the latest emails from the inbox (including the correct test email).
2. Select the exact email intended for testing from the list.
3. Move to the AI step and click Re-test step to ensure the AI returns the expected classification (e.g., priority: urgent).
4. Only then proceed to test the specific path condition (Path) and its actions.

### Duplicating actions between paths

Since every path references the exact same dynamic fields (Thread Id, From Address from Step 1, and draft_reply from the AI step), the entire Create Draft Reply action can be safely duplicated and pasted into subsequent paths without modifications. The only item requiring manual adjustment after duplicating is the Label field in the Add Label to Email action — it must match the path (e.g., AUTO/Client instead of AUTO/Urgent).

### Problem 4: Label did not appear in Gmail despite a successful test in Zapier

Diagnostic checklist applied:
1. Check the Message field in Add Label to Email — it must contain the specific message's ID, not the Thread ID or body content.
2. Re-test and verify the message — check if Zapier returned a green success confirmation.
3. Refresh Gmail (F5) — sometimes the Gmail UI does not update instantaneously.
4. Label name match — select the label from Zapier's dropdown list instead of typing the name manually as text to eliminate the risk of typos or casing mismatch.

### The same Static/Dynamic issue returned when configuring Add Label to Email

The exact same trap as in Problem 2 reoccurred when selecting the Message field in the next action — it opened by default to the Static tab displaying old sample emails. Resolution was identical: switch to Dynamic and select the ID field from Step 1. Additionally, to see the newest, just-sent test emails in that list, it was necessary to click Find new records again in Step 1.

---

## Full workflow diagram

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
 Urgent Client Meeting Invoice
 (priority (category  (category  (category
  = urgent) = client)  = meeting) = invoice)
   │    │    │        │
   ▼    ▼    ▼        ▼
 Add Label to Email (path-appropriate label, Message ID from Step 1)
   │    │    │        │
   ▼    ▼    ▼        ▼
 Create Draft Reply (Thread ID + To=From Address from Step 1, Body=draft_reply from AI)
```

---

## What you learned

- building your first workflow utilizing AI for classification and content generation (AI by Zapier — Analyze and Return Data)
- designing a prompt that simultaneously defines classification categories and instructs the model on what not to do (e.g., "do not fabricate information", "do not send responses")
- that the Output field type (e.g., Category/Single select) defines only the expected data structure, not the list of allowed values — those are specified inside the prompt text
- the critical distinction between the Static tab (hardcoded sample data) and Dynamic tab (real data from a specific step) when assigning field values in Zapier — and that confusing them permanently binds actions to a single old message
- that drafting a reply within an existing thread requires a different action than creating a regular new draft, and that it requires both a Thread ID and a recipient in the To field
- the correct testing procedure for an individual path in a branching Zap: refresh trigger sample data (Find new records), re-test the AI step, and only then test the specific path and its actions
- that actions referencing only dynamic data from earlier shared steps can be safely duplicated across paths, modifying only what actually differs (e.g., the label)
- intentionally establishing the boundary between automation and human oversight: AI prepares, human approves and sends — without automated direct sending to clients
