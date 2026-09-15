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
