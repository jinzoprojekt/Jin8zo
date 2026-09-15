# N8N – Dzień 24: Klasyfikacja zgłoszeń budżetowych w Airtable

## Projekt: Formularz → baza danych Airtable → logika biznesowa → aktualizacja rekordu → powiadomienie

### Dlaczego akurat Airtable

Airtable nie jest dominującym CRM-em na polskim rynku — pod tym względem znacznie ważniejsze docelowo będą HubSpot, Pipedrive, monday.com czy rodzime rozwiązania jak Livespace i Firmao. Airtable ma jednak ponad 500 tysięcy organizacji korzystających z niego na świecie i jest znakomitym środowiskiem treningowym: łączy bazę danych, elastyczne pola, automatyzacje i API w jednym miejscu, a n8n ma z nim gotową integrację.

Celem tej lekcji nie jest nauczenie się "jeszcze jednego narzędzia" na pamięć, tylko opanowanie **konkretnej klasy problemu**, którą później przenosi się na dowolny inny system: formularz → zapis do bazy → klasyfikacja według reguł biznesowych → aktualizacja rekordu → powiadomienie zespołu.

### Cel lekcji

- konfiguracja integracji n8n z Airtable,
- zapis zgłoszenia z formularza jako rekord w bazie Airtable,
- klasyfikacja rekordu według progów liczbowych za pomocą Switch,
- rozpoznanie i naprawienie problemu nakładających się warunków w Switchu,
- aktualizacja istniejącego rekordu o wynik klasyfikacji,
- powiadomienie zespołu przy zgłoszeniach najwyższej kategorii.

---

## Krok 1 – Przygotowanie bazy w Airtable

Nowa baza (Base) w Airtable, z tabelą zawierającą kolumny odpowiadające polom formularza: Imię, Email, Temat, Budżet (Number), Kategoria (Single select: Niski, Średni, Wysoki, VIP), Status.

## Krok 2 – Credential Airtable w n8n

W n8n, przy dodawaniu node'a Airtable, utworzenie nowego credentiala — autoryzacja przez token dostępu (Personal Access Token) wygenerowany w ustawieniach konta Airtable.

## Krok 3 – Formularz i zapis zgłoszenia

Trigger **On Form Submission** z polami: Imię, Email, Temat, Budżet (Number).

Node **Airtable**, operacja tworzenia rekordu (Create), wskazujący na wcześniej przygotowaną bazę i tabelę. Mapowanie pól formularza do odpowiednich kolumn Airtable.

## Krok 4 – Switch: klasyfikacja według budżetu

Node **Switch**, tryb `Rules`, cztery reguły oparte na polu `Budżet`:

- `VIP`: `>= 10000`
- `Wysoki`: `>= 5000`
- `Średni`: `>= 2000`
- `Niski`: `< 2000`

---

## Napotkany problem: dane trafiają do dwóch gałęzi jednocześnie

Przy teście zgłoszenia z budżetem `10000` dane trafiły zarówno do gałęzi `Wysoki`, jak i `VIP`, zamiast wyłącznie do `VIP`.

### Diagnoza

Reguły Switcha są sprawdzane **niezależnie od siebie**, nie w formie "albo-albo". Wartość `10000` jednocześnie spełnia warunek `>= 5000` (reguła Wysoki) **oraz** `>= 10000` (reguła VIP) — Switch, o ile nie skonfigurowany inaczej, wysyła dane do **każdej** pasującej gałęzi, a nie tylko do jednej, "najbardziej trafnej".

### Rozważona, ale odrzucona naprawa: jawne zakresy z warunkiem AND

Teoretycznym rozwiązaniem byłoby zawężenie reguły `Wysoki` do zakresu `>= 5000 AND < 10000`, tak aby wykluczała ona wartości należące już do `VIP`. W używanej wersji Switcha nie ma jednak możliwości połączenia dwóch warunków operatorem AND w pojedynczej regule — ta metoda nie była więc dostępna.

### Zastosowana naprawa: "First matching output" i kolejność reguł

Właściwym rozwiązaniem okazała się zmiana sposobu, w jaki Switch obsługuje wiele pasujących reguł, a nie modyfikacja samych warunków.

1. W ustawieniach node'a Switch odnaleziona i włączona opcja **"First matching output"** — Switch, zamiast wysyłać dane do wszystkich pasujących gałęzi, zatrzymuje się na pierwszej regule, która pasuje, i ignoruje pozostałe.

2. **Kolejność reguł ma przy tym kluczowe znaczenie** — muszą być ułożone od najbardziej szczegółowej (najwyższy próg) do najbardziej ogólnej (najniższy próg):
```
1. VIP     >= 10000
2. Wysoki  >= 5000
3. Średni  >= 2000
4. Niski   < 2000
```

Przy takim ułożeniu i włączonym "First matching output", wartość `10000` sprawdzana jest najpierw pod kątem reguły VIP — pasuje, więc Switch zatrzymuje się w tym miejscu, nie sprawdzając już reguły Wysoki, mimo że formalnie również by pasowała.

**Wynik po naprawie:**

| Budżet | Pierwsza pasująca reguła | Wynik |
|---|---|---|
| 1500 | `< 2000` | Niski |
| 3000 | `>= 2000` | Średni |
| 7000 | `>= 5000` | Wysoki |
| 10000 | `>= 10000` | VIP |
| 25000 | `>= 10000` | VIP |

### Wniosek

Samo poprawne zdefiniowanie progów liczbowych nie gwarantuje, że dany element trafi tylko do jednej gałęzi — trzeba dodatkowo świadomie ustawić sposób, w jaki router (tu: Switch) obsługuje sytuację, w której więcej niż jedna reguła jednocześnie pasuje do danych wejściowych. To ogólna zasada projektowania logiki warunkowej, nie tylko specyfika Airtable czy tego konkretnego projektu.

---

## Krok 5 – Aktualizacja rekordu o wynik klasyfikacji

Za każdą gałęzią Switcha: node **Airtable**, operacja **Update Record**, ustawiający kolumnę `Kategoria` na odpowiednią wartość (`Niski`, `Średni`, `Wysoki`, `VIP`) w rekordzie utworzonym w Kroku 3.

## Krok 6 – Powiadomienie przy zgłoszeniach VIP

Na gałęzi `VIP`, po aktualizacji rekordu: node **Slack**, wysyłający powiadomienie do zespołu o nowym zgłoszeniu najwyższej kategorii, z wykorzystaniem tego samego credentiala Slack co w Dniach 22–23.

---

## Pełny schemat workflowa

```
On Form Submission
        │
        ▼
     Airtable (Create Record)
        │
        ▼
      Switch (First matching output, reguły od VIP do Niski)
        │
   ┌────┼────┬────┐
   ▼    ▼    ▼    ▼
 Niski Średni Wysoki VIP
   │    │     │      │
   ▼    ▼     ▼      ▼
Airtable (Update Record: Kategoria)
                       │
                       ▼
                    Slack (tylko gałąź VIP)
```

---

## Czego się nauczyłem


- że Airtable, mimo że nie jest głównym CRM-em na polskim rynku, jest wartościowym środowiskiem do nauki wzorca "formularz → baza → logika → aktualizacja → powiadomienie"
- konfiguracji integracji n8n z Airtable przez Personal Access Token
- tworzenia i aktualizowania rekordów w zewnętrznej bazie danych z poziomu n8n
- że reguły Switcha są domyślnie sprawdzane niezależnie, przez co dane mogą trafić jednocześnie do kilku pasujących gałęzi
- że opcja "First matching output" w połączeniu z odpowiednią kolejnością reguł (od najbardziej do najmniej szczegółowej) rozwiązuje problem nakładających się warunków bez potrzeby operatora AND
- że sam poprawny dobór progów liczbowych nie wystarczy — trzeba też świadomie zdecydować, jak router ma się zachować przy wielu pasujących regułach naraz
- że tę samą klasę problemu (klasyfikacja + aktualizacja rekordu + powiadomienie) będzie można w przyszłości przenieść na dowolny inny CRM czy bazę danych, z którą n8n ma integrację

# N8N – Day 24: Classification of Budget Requests in Airtable

## Project: Form -> Airtable database -> business logic -> record update -> notification

### Why specifically Airtable

Airtable is not the dominant CRM on the Polish market — in this respect, HubSpot, Pipedrive, monday.com, or native solutions like Livespace and Firmao will ultimately be much more important. However, Airtable has over 500,000 organizations using it globally and serves as an excellent training environment: it combines a database, flexible fields, automations, and an API in one place, and n8n comes with an out-of-the-box integration for it.

The goal of this lesson is not to memorize "yet another tool," but to master a **specific class of problem** that can later be applied to any other system: form -> database record -> classification by business rules -> record update -> team notification.

### Lesson Goal

- configuring the n8n integration with Airtable,
- saving a form submission as a record in an Airtable database,
- classifying the record based on numerical thresholds using Switch,
- identifying and fixing the issue of overlapping conditions in Switch,
- updating an existing record with the classification result,
- notifying the team for top-category requests.

---

## Step 1 – Database setup in Airtable

A new base in Airtable with a table containing columns matching the form fields: First Name, Email, Subject, Budget (Number), Category (Single select: Low, Medium, High, VIP), Status.

## Step 2 – Airtable Credential in n8n

In n8n, when adding the Airtable node, create a new credential — authorization via a Personal Access Token generated in the Airtable account settings.

## Step 3 – Form and submission saving

Trigger On Form Submission with fields: First Name, Email, Subject, Budget (Number).

Node Airtable, Create record operation, pointing to the previously prepared base and table. Mapping form fields to the corresponding Airtable columns.

## Step 4 – Switch: budget-based classification

Node Switch, Rules mode, four rules based on the Budget field:

- VIP: >= 10000
- High: >= 5000
- Medium: >= 2000
- Low: < 2000

---

## Encountered problem: data routed to two branches simultaneously

During a test submission with a budget of 10000, data was routed to both the High and VIP branches instead of exclusively to VIP.

### Diagnosis

Switch rules are checked **independently of each other**, not in an "either-or" manner. The value 10000 simultaneously meets the condition >= 5000 (High rule) **and** >= 10000 (VIP rule) — unless configured otherwise, Switch sends data to **every** matching branch, not just the single "most relevant" one.

### Considered but rejected fix: explicit ranges with an AND condition

A theoretical solution would be to narrow down the High rule to the range >= 5000 AND < 10000, so that it excludes values already belonging to VIP. However, in the used version of Switch, there is no option to combine two conditions with an AND operator within a single rule — thus, this method was not available.

### Applied fix: "First matching output" and rule order

The proper solution turned out to be changing how Switch handles multiple matching rules rather than modifying the conditions themselves.

1. In the Switch node settings, locate and enable the **"First matching output"** option — instead of sending data to all matching branches, Switch stops at the first rule that matches and ignores the rest.

2. **Rule order is crucial here** — rules must be arranged from most specific (highest threshold) to most general (lowest threshold):
    1. VIP     >= 10000
    2. High    >= 5000
    3. Medium  >= 2000
    4. Low     < 2000

With this arrangement and "First matching output" enabled, the value 10000 is checked first against the VIP rule — it matches, so Switch stops right there without checking the High rule, even though it would formally match as well.

**Result after the fix:**

| Budget | First matching rule | Result |
|---|---|---|
| 1500 | `< 2000` | Low |
| 3000 | `>= 2000` | Medium |
| 7000 | `>= 5000` | High |
| 10000 | `>= 10000` | VIP |
| 25000 | `>= 10000` | VIP |

### Conclusion

Simply defining numerical thresholds correctly does not guarantee that a given item will land in only one branch — you must also consciously configure how the router (here: Switch) handles situations where more than one rule matches the input data simultaneously. This is a general principle of designing conditional logic, not just a quirk of Airtable or this specific project.

---

## Step 5 – Updating the record with the classification result

After each Switch branch: node Airtable, Update Record operation, setting the Category column to the corresponding value (Low, Medium, High, VIP) in the record created in Step 3.

## Step 6 – Notification for VIP requests

On the VIP branch, after updating the record: node Slack, sending a notification to the team about a new top-category request, using the same Slack credential as in Days 22–23.

---

## Full workflow diagram

On Form Submission
        │
        ▼
     Airtable (Create Record)
        │
        ▼
      Switch (First matching output, rules from VIP to Low)
        │
   ┌────┼────┬────┐
   ▼    ▼    ▼    ▼
  Low Medium High VIP
   │    │     │    │
   ▼    ▼     ▼    ▼
Airtable (Update Record: Category)
                     │
                     ▼
                  Slack (VIP branch only)

---

## What I learned

- that Airtable, despite not being the primary CRM on the Polish market, is a valuable environment for learning the "form -> database -> logic -> update -> notification" pattern
- configuring n8n integration with Airtable via a Personal Access Token
- creating and updating records in an external database directly from n8n
- that Switch rules are checked independently by default, which can cause data to route to multiple matching branches at once
- that the "First matching output" option combined with proper rule ordering (from most to least specific) solves the overlapping conditions problem without needing an AND operator
- that proper numerical thresholds alone are not enough — you must also consciously decide how the router should behave when multiple rules match at once
- that this same class of problem (classification + record update + notification) can be transferred in the future to any other CRM or database integrated with n8n
