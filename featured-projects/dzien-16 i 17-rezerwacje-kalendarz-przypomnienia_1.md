# N8N – Dzień 16/17: Rezerwacje z Google Calendar i automatycznym przypomnieniem

## Projekt: System rezerwacji z weryfikacją dostępności terminu, rozdzieleniem zapisu według usługi i automatycznym przypomnieniem dzień przed wizytą

### Cel lekcji

- sprawdzanie dostępności terminu w Google Calendar przed przyjęciem rezerwacji,
- automatyczne tworzenie wydarzenia w kalendarzu na podstawie zgłoszenia z formularza,
- rozdzielanie zapisu do różnych arkuszy w zależności od wybranej usługi,
- opóźnianie wykonania części workflowa do konkretnego momentu w przyszłości,
- wysyłanie automatycznego przypomnienia klientowi na dzień przed wizytą.

---

## Krok 1 – Nowy workflow i trigger

Nazwa workflowa: `Rezerwacje z kalendarzem`

Trigger: **On Form Submission**

## Krok 2 – Konfiguracja formularza

Tytuł: `Rezerwacja wizyty`

Pola formularza:
- **Imię** – Text Input
- **Email** – Email
- **Telefon** – Text Input
- **Usługa** – Dropdown, opcje: `Broda`, `Włosy`, `Włosy + Broda`
- **Data wizyty** – Date
- **Godzina wizyty** – Dropdown, opcje: pełne godziny pracy, np. `09:00`, `10:00`, `11:00`, `12:00`, `13:00`, `14:00`, `15:00`, `16:00`, `17:00`

Data i godzina są celowo dwoma osobnymi polami — formularz w n8n nie udostępnia typu łączącego oba naraz, więc pełny znacznik czasu budujemy w kolejnym kroku.

## Krok 3 – Edit Fields: budowa pełnego znacznika czasu

Node **Edit Fields**, tryb `Manual Mapping`.

Nowe pole `DataCzas` (String):
```
{{ $json["Data wizyty"] }}T{{ $json["Godzina wizyty"] }}:00
```

Wynikiem jest pełny znacznik czasu w formacie ISO, np. `2026-08-20T10:00:00`, gotowy do użycia w Google Calendar.

Włącz przełącznik **"Include Other Input Fields"** — dzięki temu na wyjściu tego node'a, oprócz nowego pola `DataCzas`, pozostają też wszystkie pozostałe dane z formularza (Imię, Email, Telefon, Usługa). Od tego momentu wszystkie kolejne nody w workflowie odwołują się do danych z Edit Fields — to jedno, spójne źródło prawdy dla całej dalszej części procesu.

## Krok 4 – Google Calendar: sprawdzenie dostępności terminu

Node **Google Calendar**, operacja **Get many events**.

- Calendar: Twój kalendarz (From list)
- Return All: wyłączone
- Limit: `50`
- **After:** `{{ $('Edit Fields').item.json.DataCzas }}`
- **Before:** `{{ DateTime.fromISO($('Edit Fields').item.json.DataCzas).plus({ hours: 1 }).toISO() }}`

W zakładce **Settings** włącz **"Always Output Data"**. Bez tego, jeśli termin jest wolny (zero zwróconych wydarzeń), node nie przekaże niczego dalej i kolejne kroki workflowa się nie wykonają.

## Krok 5 – IF: weryfikacja dostępności terminu

Node **IF**, warunek: `{{ $json.id }}` **is empty**

Każde realne wydarzenie zwrócone przez Google Calendar posiada pole `id`. Pusty placeholder wygenerowany dzięki "Always Output Data" (gdy nic nie znaleziono) go nie posiada — to niezawodny sposób odróżnienia terminu wolnego od zajętego.

**Gałąź TRUE (termin wolny)** → Krok 6.

**Gałąź FALSE (termin zajęty)** → node Gmail z informacją dla klienta:

```
Temat: Wybrany termin jest niestety zajęty

Treść:
Cześć {{ $('Edit Fields').item.json.Imię }},

niestety wybrany termin ({{ $('Edit Fields').item.json.DataCzas }}) jest już
zarezerwowany. Skontaktujemy się wkrótce, aby zaproponować inny dostępny termin.

Pozdrawiamy,
Salon Jinzo
```

Workflow kończy się na tej gałęzi.

## Krok 6 – Google Calendar: utworzenie wydarzenia

Node **Create an event**, gałąź TRUE.

- Start: `{{ $('Edit Fields').item.json.DataCzas }}`
- End: `{{ DateTime.fromISO($('Edit Fields').item.json.DataCzas).plus({ hours: 1 }).toISO() }}`
- Attendees: `{{ $('Edit Fields').item.json.Email }}`
- Summary: `{{ $('Edit Fields').item.json.Usługa }} - {{ $('Edit Fields').item.json.Imię }}`

## Krok 7 – Switch: rozdzielenie zapisu według usługi

Trzy usługi oznaczają trzy osobne arkusze Google Sheets: `Broda`, `Włosy`, `Włosy + Broda`. Żeby dane trafiały zawsze do właściwego arkusza, dodajesz rozgałęzienie.

Node **Switch**, tryb `Rules`, podłączony za **Create an event**:

- `{{ $('Edit Fields').item.json.Usługa }}` `is equal to` `Broda` → output: `Broda`
- `{{ $('Edit Fields').item.json.Usługa }}` `is equal to` `Włosy` → output: `Włosy`
- `{{ $('Edit Fields').item.json.Usługa }}` `is equal to` `Włosy + Broda` → output: `Włosy + Broda`

## Krok 8 – Append Row do trzech arkuszy

Pod każdą gałęzią Switcha — node **Append row**, wskazujący na odpowiedni arkusz (`Broda`, `Włosy`, `Włosy + Broda`). Mapowanie identyczne w każdej kopii, wszystkie pola z Edit Fields: Imię, Email, Telefon, DataCzas.

## Krok 9 – Merge: powrót do jednej ścieżki

Dalsze kroki (potwierdzenie mailowe, przypomnienie) są identyczne niezależnie od wybranej usługi. Po trzech Append Row dodajesz node **Merge** (tryb `Append`), łączący wszystkie trzy gałęzie z powrotem w jedną, żeby nie duplikować Gmaila i Waita trzykrotnie.

## Krok 10 – Gmail: potwierdzenie rezerwacji

Node **Gmail**, operacja: `Send a message`.

- Do: `{{ $('Edit Fields').item.json.Email }}`

```
Temat: Potwierdzenie rezerwacji

Treść:
Cześć {{ $('Edit Fields').item.json.Imię }},

potwierdzamy rezerwację na usługę: {{ $('Edit Fields').item.json.Usługa }}
w dniu: {{ $('Edit Fields').item.json.DataCzas }}.

Do zobaczenia!

Pozdrawiamy,
Salon Jinzo
```

## Krok 11 – Node Wait: opóźnienie do dnia przed wizytą

Node **Wait**, Resume: `At Specified Time`.

Docelowa wartość, licząca 24 godziny przed terminem wizyty:
```
{{ DateTime.fromISO($('Edit Fields').item.json.DataCzas).minus({ hours: 24 }).toISO() }}
```

**Testowanie tego kroku:** czekanie realnych 24 godzin, żeby zweryfikować działanie przypomnienia, jest niepraktyczne podczas budowy workflowa. Rozwiązaniem jest tymczasowy przełącznik trybu testowego — dodatkowe pole boolowskie `TrybTestowy` (np. w node'ie Edit Fields), ustawiane ręcznie na `true` podczas testów. W polu Date & Time node'a Wait:
```
{{ $json.TrybTestowy ? DateTime.now().plus({ minutes: 1 }).toISO() : DateTime.fromISO($('Edit Fields').item.json.DataCzas).minus({ hours: 24 }).toISO() }}
```

Przy `TrybTestowy = true` Wait czeka tylko minutę, co pozwala zweryfikować całą dalszą ścieżkę (Wait → Gmail przypomnienie) w czasie rzeczywistym. Po zakończeniu testów przełącznik wraca na `false`, przywracając docelowe liczenie 24 godzin przed wizytą.

Na self-hosted n8n instancja musi działać nieprzerwanie przez cały czas oczekiwania — jeśli serwer zostanie zatrzymany, wykonanie wznowi się dopiero po jego ponownym uruchomieniu.

## Krok 12 – Gmail: przypomnienie o wizycie

Node **Gmail**, po node'ie Wait.

```
Temat: Przypomnienie o jutrzejszej wizycie

Treść:
Cześć {{ $('Edit Fields').item.json.Imię }},

przypominamy o wizycie już jutro ({{ $('Edit Fields').item.json.DataCzas }})
na usługę: {{ $('Edit Fields').item.json.Usługa }}.

Do zobaczenia!

Pozdrawiamy,
Salon Jinzo
```

## Krok 13 – Aktywacja workflowa

Kliknij **Publish**, aby workflow zaczął działać samodzielnie przy realnych zgłoszeniach z formularza.

---

## Pełny schemat workflowa

```
Form Trigger
      │
      ▼
Edit Fields (DataCzas, Include Other Input Fields: ON)
      │
      ▼
Google Calendar: Get many events (Always Output Data: ON)
      │
      ▼
     IF (id is empty?)
    /            \
 TRUE            FALSE
   │            Gmail: "termin zajęty"
   ▼            (koniec)
Create an event
   │
   ▼
Switch (po polu Usługa)
   ├── Broda ──────────► Append row (Broda)
   ├── Włosy ─────────► Append row (Włosy)
   └── Włosy + Broda ─► Append row (Włosy + Broda)
                              │
                              ▼
                           Merge
                              │
                              ▼
                    Gmail: potwierdzenie rezerwacji
                              │
                              ▼
                Wait (24h przed wizytą / tryb testowy: 1 min)
                              │
                              ▼
                    Gmail: przypomnienie o wizycie
```

---

## Czego się nauczyłem


- łączenia osobnych pól formularza (daty i godziny) w jeden pełny znacznik czasu przez Edit Fields
- znaczenia opcji "Include Other Input Fields" przy pracy z Manual Mapping, gdy dalsze nody potrzebują zarówno nowych, jak i oryginalnych danych
- roli "Always Output Data" przy sprawdzaniu warunków opartych na pustym wyniku zapytania
- budowania niezawodnego warunku sprawdzającego istnienie danych na podstawie charakterystycznego pola rekordu, zamiast liczenia elementów
- automatycznego tworzenia wydarzeń w kalendarzu z zaproszeniem uczestnika
- rozdzielania zapisu danych do wielu arkuszy za pomocą Switch, oraz scalania wielu gałęzi z powrotem przez Merge, aby nie duplikować wspólnej dalszej logiki
- działania node'a Wait w trybie "At Specified Time" oraz praktycznej metody jego szybkiego testowania przez warunkowy przełącznik trybu testowego

# N8N – Day 16/17: Reservations with Google Calendar and Automatic Reminder

## Project: Reservation system with slot availability verification, separation of records by service, and automatic reminder the day before the visit

### Lesson Goal

- checking slot availability in Google Calendar before accepting a reservation,
- automatic creation of an event in the calendar based on a form submission,
- separating records into different spreadsheets depending on the selected service,
- delaying the execution of part of the workflow to a specific point in the future,
- sending an automatic reminder to the client the day before the visit.

---

## Step 1 – New workflow and trigger

Workflow name: Reservations with calendar

Trigger: On Form Submission

## Step 2 – Form configuration

Title: Booking a visit

Form fields:
- First Name – Text Input
- Email – Email
- Phone – Text Input
- Service – Dropdown, options: Beard, Hair, Hair + Beard
- Visit Date – Date
- Visit Time – Dropdown, options: full working hours, e.g., 09:00, 10:00, 11:00, 12:00, 13:00, 14:00, 15:00, 16:00, 17:00

Date and time are intentionally two separate fields — the n8n form does not provide a type connecting both at once, so we build the full timestamp in the next step.

## Step 3 – Edit Fields: building the full timestamp

Node Edit Fields, Manual Mapping mode.

New field DateTime (String):
    {{ $json["Visit Date"] }}T{{ $json["Visit Time"] }}:00

The result is a full timestamp in ISO format, e.g., 2026-08-20T10:00:00, ready for use in Google Calendar.

Enable the "Include Other Input Fields" toggle — thanks to this, on the output of this node, in addition to the new DateTime field, all other form data (First Name, Email, Phone, Service) remains. From this point, all subsequent nodes in the workflow refer to the data from Edit Fields — this is a single, consistent source of truth for the entire rest of the process.

## Step 4 – Google Calendar: checking slot availability

Node Google Calendar, operation Get many events.

- Calendar: Your calendar (From list)
- Return All: disabled
- Limit: 50
- After: {{ $('Edit Fields').item.json.DateTime }}
- Before: {{ DateTime.fromISO($('Edit Fields').item.json.DateTime).plus({ hours: 1 }).toISO() }}

In the Settings tab, enable "Always Output Data". Without this, if the slot is free (zero returned events), the node will not pass anything forward and the subsequent steps of the workflow will not execute.

## Step 5 – IF: slot availability verification

Node IF, condition: {{ $json.id }} is empty

Every real event returned by Google Calendar has an id field. An empty placeholder generated thanks to "Always Output Data" (when nothing was found) does not have it — this is a reliable way to distinguish a free slot from an occupied one.

TRUE branch (free slot) -> Step 6.

FALSE branch (occupied slot) -> Gmail node with information for the client:

    Subject: Unfortunately, the selected time slot is occupied

    Body:
    Hi {{ $('Edit Fields').item.json.FirstName }},

    unfortunately, the selected time slot ({{ $('Edit Fields').item.json.DateTime }}) is already
    booked. We will contact you shortly to suggest another available time slot.

    Best regards,
    Jinzo Salon

The workflow ends on this branch.

## Step 6 – Google Calendar: event creation

Node Create an event, TRUE branch.

- Start: {{ $('Edit Fields').item.json.DateTime }}
- End: {{ DateTime.fromISO($('Edit Fields').item.json.DateTime).plus({ hours: 1 }).toISO() }}
- Attendees: {{ $('Edit Fields').item.json.Email }}
- Summary: {{ $('Edit Fields').item.json.Service }} - {{ $('Edit Fields').item.json.FirstName }}

## Step 7 – Switch: separating records by service

Three services mean three separate Google Sheets spreadsheets: Beard, Hair, Hair + Beard. To ensure data always ends up in the correct spreadsheet, you add a branch.

Node Switch, Rules mode, connected after Create an event:

- {{ $('Edit Fields').item.json.Service }} is equal to Beard -> output: Beard
- {{ $('Edit Fields').item.json.Service }} is equal to Hair -> output: Hair
- {{ $('Edit Fields').item.json.Service }} is equal to Hair + Beard -> output: Hair + Beard

## Step 8 – Append Row to three spreadsheets

Under each Switch branch — an Append row node, pointing to the appropriate spreadsheet (Beard, Hair, Hair + Beard). Mapping is identical in each copy, all fields from Edit Fields: FirstName, Email, Phone, DateTime.

## Step 9 – Merge: return to a single path

Subsequent steps (email confirmation, reminder) are identical regardless of the selected service. After three Append Row nodes, you add a Merge node (Append mode), connecting all three branches back into one, so as not to duplicate Gmail and Wait three times.

## Step 10 – Gmail: reservation confirmation

Node Gmail, operation: Send a message.

- To: {{ $('Edit Fields').item.json.Email }}

    Subject: Reservation confirmation

    Body:
    Hi {{ $('Edit Fields').item.json.FirstName }},

    we confirm your reservation for the service: {{ $('Edit Fields').item.json.Service }}
    on: {{ $('Edit Fields').item.json.DateTime }}.

    See you soon!

    Best regards,
    Jinzo Salon

## Step 11 – Node Wait: delay until the day before the visit

Node Wait, Resume: At Specified Time.

Target value, calculating 24 hours before the visit slot:
    {{ DateTime.fromISO($('Edit Fields').item.json.DateTime).minus({ hours: 24 }).toISO() }}

Testing this step: waiting a real 24 hours to verify the functioning of the reminder is impractical during workflow construction. The solution is a temporary test mode switch — an additional boolean field TestMode (e.g., in the Edit Fields node), manually set to true during tests. In the Date & Time field of the Wait node:
    {{ $json.TestMode ? DateTime.now().plus({ minutes: 1 }).toISO() : DateTime.fromISO($('Edit Fields').item.json.DateTime).minus({ hours: 24 }).toISO() }}

With TestMode = true, Wait only waits a minute, allowing the entire further path (Wait -> Gmail reminder) to be verified in real-time. After testing, the switch returns to false, restoring the target calculation of 24 hours before the visit.

On self-hosted n8n, the instance must run continuously throughout the waiting period — if the server is stopped, execution will resume only after it is restarted.

## Step 12 – Gmail: visit reminder

Node Gmail, after the Wait node.

    Subject: Reminder about tomorrow's visit

    Body:
    Hi {{ $('Edit Fields').item.json.FirstName }},

    we remind you about your visit tomorrow ({{ $('Edit Fields').item.json.DateTime }})
    for the service: {{ $('Edit Fields').item.json.Service }}.

    See you soon!

    Best regards,
    Jinzo Salon

## Step 13 – Workflow activation

Click Publish so the workflow starts running independently with real form submissions.

---

## Full workflow diagram

Form Trigger
      │
      ▼
Edit Fields (DateTime, Include Other Input Fields: ON)
      │
      ▼
Google Calendar: Get many events (Always Output Data: ON)
      │
      ▼
     IF (id is empty?)
    /            \
 TRUE            FALSE
   │            Gmail: "slot occupied"
   ▼            (end)
Create an event
   │
   ▼
Switch (by Service field)
   ├── Beard ──────────► Append row (Beard)
   ├── Hair ───────────► Append row (Hair)
   └── Hair + Beard ───► Append row (Hair + Beard)
                              │
                              ▼
                           Merge
                              │
                              ▼
                    Gmail: reservation confirmation
                              │
                              ▼
                Wait (24h before visit / test mode: 1 min)
                              │
                              ▼
                    Gmail: visit reminder

---

## What I learned

- combining separate form fields (date and time) into a single full timestamp via Edit Fields
- the importance of the "Include Other Input Fields" option when working with Manual Mapping when subsequent nodes need both new and original data
- the role of "Always Output Data" when checking conditions based on an empty query result
- building a reliable condition checking data existence based on a characteristic record field, instead of counting elements
- automatic creation of calendar events with participant invitation
- separating data saving into multiple spreadsheets using Switch, and merging multiple branches back together via Merge, to avoid duplicating common subsequent logic
- the operation of the Wait node in "At Specified Time" mode and a practical method of testing it quickly via a conditional test mode switch
