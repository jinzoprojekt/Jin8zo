# N8N – Dzień 19: Publiczne API do zgłoszeń serwisowych

## Projekt: Webhook przyjmujący zgłoszenia z zewnętrznych systemów, zbudowany tak, żeby działał identycznie lokalnie i po przeniesieniu na serwer

### Cel lekcji

- budowanie ogólnego, uniwersalnego punktu wejścia API (webhook przyjmujący POST z zewnątrz),
- stosowanie wzorca "bufora" danych (Edit Fields zaraz po triggerze) jako standardowej dobrej praktyki,
- walidację danych przychodzących z zewnętrznego systemu i zwracanie czytelnych błędów,
- zwracanie ustrukturyzowanej odpowiedzi JSON, jakiej oczekuje każdy system integrujący się przez API.

---

## Krok 1 – Nowy workflow i trigger

Nazwa: `Zgłoszenia serwisowe - API`

Trigger: **Webhook**

- Method: `POST`
- Path: `zgloszenie-serwisowe`
- Response: `Using Respond to Webhook Node`

## Krok 2 – Edit Fields: zabezpieczenie danych wejściowych

Zaraz po Webhooku node **Edit Fields**, tryb `Manual Mapping`, z polami:

- `Imię` → `{{ $json.body.imie }}`
- `Email` → `{{ $json.body.email }}`
- `Opis` → `{{ $json.body.opis }}`
- `Priorytet` → `{{ $json.body.priorytet }}`

Włączony przełącznik **"Include Other Input Fields"**.

Od tego momentu każdy kolejny node w workflowie odwołuje się wyłącznie do `$('Edit Fields').item.json...`, nigdy bezpośrednio do `$json` ani do `$('Webhook')`. To bezpośrednie zastosowanie wniosku z Dnia 17: dane zabezpieczone wcześnie w osobnym node'ie nie tracą dostępności, nawet jeśli dalsze nody silnie przetwarzają swój własny output.

## Krok 3 – IF: walidacja zgłoszenia

Node **IF**, dwa warunki połączone przez `OR`:
- `{{ $('Edit Fields').item.json.Email }}` **is empty**
- `{{ $('Edit Fields').item.json.Opis }}` **is empty**

**Gałąź TRUE (brak wymaganych danych)** → **Respond to Webhook**:
- Response Code: `400`
- Content-Type: `application/json`
- Body:
```json
{
  "status": "błąd",
  "message": "Brakuje wymaganego pola: email lub opis problemu."
}
```

**Gałąź FALSE (dane poprawne)** → Krok 4.

## Krok 4 – Google Sheets: zapis zgłoszenia

Node **Google Sheets**, `Append Row`, do arkusza `Zgłoszenia serwisowe` (kolumny: Imię, Email, Opis, Priorytet, Data).

Kolumna `Data` nie pochodzi z żadnych danych wejściowych — generuje ją sam n8n w momencie przyjęcia zgłoszenia:
```
{{ $now.format('yyyy-MM-dd HH:mm:ss') }}
```

## Krok 5 – Gmail: potwierdzenie przyjęcia zgłoszenia

Wszystkie pola treści wiadomości muszą jawnie odwoływać się do node'a **Edit Fields**, w trybie **Expression**:

```
Do: {{ $('Edit Fields').item.json["Email"] }}

Temat: Potwierdzenie przyjęcia zgłoszenia

Treść:
Cześć {{ $('Edit Fields').item.json["Imię"] }},

potwierdzamy przyjęcie zgłoszenia:
"{{ $('Edit Fields').item.json["Opis"] }}"

Priorytet: {{ $('Edit Fields').item.json["Priorytet"] }}

Odezwiemy się wkrótce.

Pozdrawiamy
```

## Krok 6 – Respond to Webhook: odpowiedź sukcesu

- Response Code: `200`
- Content-Type: `application/json`
- Body:
```json
{
  "status": "OK",
  "message": "Zgłoszenie zostało przyjęte."
}
```

## Krok 7 – Testowanie: workflow-klient

Nowy, osobny workflow: **Manual Trigger → HTTP Request**.

- Method: `POST`
- URL (tryb testowy): `http://localhost:5678/webhook-test/zgloszenie-serwisowe`
- Body Content Type: `JSON`
- Body:
```json
{
  "imie": "Daniel",
  "email": "jinzoprojekt@gmail.com",
  "opis": "Nie działa mi drukarka fiskalna",
  "priorytet": "Wysoki"
}
```

Przetestuj dwa przypadki: z kompletnymi danymi (oczekiwany `status: OK`, mail i wpis w arkuszu) oraz z pustym `email` (oczekiwany `status: błąd`, kod 400, brak wpisu w arkuszu).

## Krok 8 – Aktywacja

**Publish.** Po aktywacji adres testowy (`/webhook-test/...`) przestaje działać — workflow-klient z Kroku 7 trzeba przełączyć na adres produkcyjny (`/webhook/...`, bez „-test"), żeby dalej działał.

---

## Napotkany błąd i naprawa: treść maila nie renderowała jednego z pól

Po pierwszym pełnym teście przychodzący mail wyglądał następująco:

```
Cześć {{ $json.body.imie }},

potwierdzamy przyjęcie zgłoszenia: "Nie działa mi laptop"

Priorytet: Wysoki

Odezwiemy się wkrótce.

Pozdrawiamy
```

Pole `Imię` pozostało jako dosłowny, nieprzetworzony tekst `{{ $json.body.imie }}`, podczas gdy `Opis` i `Priorytet` wyświetliły się poprawnie jako realne wartości. Przyczyną było odwołanie bezpośrednio do `$json.body...` w treści maila Gmaila, zamiast do bufora danych z node'a Edit Fields — w tym konkretnym polu n8n nie przetworzył wyrażenia jako aktywnego expressiona.

**Naprawa:**

1. Wykonano cykl **Unpublish → Publish** workflowa (wymuszenie ponownego załadowania konfiguracji).
2. W treści maila jawnie przełączono pole na tryb **Expression**.
3. Wszystkie odwołania zmieniono z `$json.body...` na jednoznaczne odwołania do bufora danych:
```
Cześć {{ $('Edit Fields').item.json["Imię"] }},

potwierdzamy przyjęcie zgłoszenia: "{{ $('Edit Fields').item.json["Opis"] }}"

Priorytet: {{ $('Edit Fields').item.json["Priorytet"] }}

Odezwiemy się wkrótce.

Pozdrawiamy
```

Po tej zmianie wszystkie pola renderowały się poprawnie. Wniosek pokrywa się z zasadą z Dnia 17: odwoływanie się wprost do surowych danych wejściowych (`$json`) w odległych od triggera nodach jest mniej niezawodne niż jawne odwołanie do konkretnego, wcześniejszego node'a przechowującego przetworzone dane.

---

## Pełny schemat workflowa

```
Webhook (POST /zgloszenie-serwisowe)
      │
      ▼
Edit Fields (bufor danych: Imię, Email, Opis, Priorytet)
      │
      ▼
     IF (Email lub Opis puste?)
    /            \
 TRUE            FALSE
   │            Google Sheets: Append Row
Respond          │
(400, błąd)      Gmail: potwierdzenie
                  │
                  Respond to Webhook (200, OK)
```

---

## Czego się nauczyłem


- projektowania ogólnego endpointu API, niezależnego od miejsca hostowania
- stosowania wzorca "bufora danych" (Edit Fields zaraz po triggerze) jako świadomej, powtarzalnej praktyki
- walidacji danych przychodzących z zewnętrznego systemu i zwracania ustrukturyzowanych kodów/komunikatów błędów (400 vs 200)
- że odwołania do `$json` bezpośrednio w odległych od triggera nodach bywają zawodne, i że jawne, jednoznaczne odwołanie do konkretnego node'a (`$('NazwaNode').item.json[...]`) w trybie Expression jest bardziej niezawodne
- że publikacja workflowa (Unpublish/Publish) potrafi być potrzebna do wymuszenia odświeżenia konfiguracji po większych zmianach w treści node'a

# N8N – Day 19: Public API for Service Requests

## Project: A Webhook receiving service requests from external systems, built to work identically locally and after moving to a server

### Lesson Goal

- building a general, universal API entry point (webhook receiving POST from external sources),
- applying the data "buffer" pattern (Edit Fields right after the trigger) as a standard best practice,
- validating incoming data from an external system and returning readable errors,
- returning a structured JSON response expected by any system integrating via API.

---

## Step 1 – New workflow and trigger

Name: Service requests - API

Trigger: Webhook

- Method: POST
- Path: service-request
- Response: Using Respond to Webhook Node

## Step 2 – Edit Fields: securing input data

Right after the Webhook, an Edit Fields node, Manual Mapping mode, with fields:

- First Name -> {{ $json.body.first_name }}
- Email -> {{ $json.body.email }}
- Description -> {{ $json.body.description }}
- Priority -> {{ $json.body.priority }}

Toggle "Include Other Input Fields" is enabled.

From this moment, every subsequent node in the workflow refers exclusively to $('Edit Fields').item.json..., never directly to $json or to $('Webhook'). This is a direct application of the conclusion from Day 17: data secured early in a separate node does not lose availability, even if subsequent nodes heavily process their own output.

## Step 3 – IF: request validation

Node IF, two conditions combined by OR:
- {{ $('Edit Fields').item.json.Email }} is empty
- {{ $('Edit Fields').item.json.Description }} is empty

TRUE branch (missing required data) -> Respond to Webhook:
- Response Code: 400
- Content-Type: application/json
- Body:
    {
      "status": "error",
      "message": "Missing required field: email or problem description."
    }

FALSE branch (correct data) -> Step 4.

## Step 4 – Google Sheets: saving the request

Node Google Sheets, Append Row, to spreadsheet Service Requests (columns: First Name, Email, Description, Priority, Date).

The Date column does not come from any input data — it is generated by n8n itself at the moment the request is received:
    {{ $now.format('yyyy-MM-dd HH:mm:ss') }}

## Step 5 – Gmail: confirmation of request receipt

All message content fields must explicitly refer to the Edit Fields node in Expression mode:

    To: {{ $('Edit Fields').item.json["Email"] }}

    Subject: Service request receipt confirmation

    Body:
    Hi {{ $('Edit Fields').item.json["First Name"] }},

    we confirm the receipt of your request:
    "{{ $('Edit Fields').item.json["Description"] }}"

    Priority: {{ $('Edit Fields').item.json["Priority"] }}

    We will get back to you shortly.

    Best regards

## Step 6 – Respond to Webhook: success response

- Response Code: 200
- Content-Type: application/json
- Body:
    {
      "status": "OK",
      "message": "The request has been accepted."
    }

## Step 7 – Testing: client-workflow

A new, separate workflow: Manual Trigger -> HTTP Request.

- Method: POST
- URL (test mode): http://localhost:5678/webhook-test/service-request
- Body Content Type: JSON
- Body:
    {
      "first_name": "Daniel",
      "email": "jinzoprojekt@gmail.com",
      "description": "My fiscal printer is not working",
      "priority": "High"
    }

Test two cases: with complete data (expected status: OK, email, and entry in the spreadsheet) and with an empty email (expected status: error, code 400, no entry in the spreadsheet).

## Step 8 – Activation

Publish. After activation, the test address (/webhook-test/...) stops working — the client-workflow from Step 7 needs to be switched to the production address (/webhook/..., without "-test") to continue working.

---

## Encountered error and fix: email body was not rendering one of the fields

After the first full test, the incoming email looked as follows:

    Hi {{ $json.body.first_name }},

    we confirm the receipt of your request: "My laptop is not working"

    Priority: High

    We will get back to you shortly.

    Best regards

The First Name field remained as literal, unprocessed text {{ $json.body.first_name }}, while Description and Priority displayed correctly as real values. The cause was referencing $json.body... directly in the body of the Gmail email, instead of the data buffer from the Edit Fields node — in this specific field, n8n did not process the expression as an active expression.

Fix:

1. Performed an Unpublish -> Publish workflow cycle (forcing configuration reload).
2. In the email body, explicitly switched the field to Expression mode.
3. Changed all references from $json.body... to explicit references to the data buffer:
    Hi {{ $('Edit Fields').item.json["First Name"] }},

    we confirm the receipt of your request: "{{ $('Edit Fields').item.json["Description"] }}"

    Priority: {{ $('Edit Fields').item.json["Priority"] }}

    We will get back to you shortly.

    Best regards

After this change, all fields rendered correctly. The conclusion matches the rule from Day 17: referencing raw input data ($json) directly in nodes distant from the trigger is less reliable than an explicit, clear reference to a specific earlier node storing the processed data.

---

## Full workflow diagram

Webhook (POST /service-request)
      │
      ▼
Edit Fields (data buffer: First Name, Email, Description, Priority)
      │
      ▼
     IF (Email or Description empty?)
    /            \
 TRUE            FALSE
   │            Google Sheets: Append Row
Respond          │
(400, error)     Gmail: confirmation
                  │
                  Respond to Webhook (200, OK)

---

## What I learned

- designing a general API endpoint, independent of the hosting location
- using the "data buffer" pattern (Edit Fields right after the trigger) as a conscious, repeatable practice
- validating incoming data from an external system and returning structured error codes/messages (400 vs 200)
- that referencing $json directly in nodes distant from the trigger can be unreliable, and that an explicit, clear reference to a specific node ($('NodeName').item.json[...]) in Expression mode is more reliable
- that publishing a workflow (Unpublish/Publish) can be necessary to force configuration refresh after major changes in a node's content
