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
