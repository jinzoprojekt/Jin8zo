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
