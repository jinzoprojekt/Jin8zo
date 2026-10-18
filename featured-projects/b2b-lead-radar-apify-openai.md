# N8N – B2B Lead Radar: scraping firm, kwalifikacja przez AI i szkice maili w Gmailu

## Projekt: lista prawdziwych firm z Google Maps → ocena potencjału 1-10 przez OpenAI → zapis do arkusza → szkice spersonalizowanych maili dla najlepszych leadów

### Dlaczego to wymaga workflowa, a nie jednego promptu

Czat z AI może ocenić jedną firmę, którą mu opiszę. Nie pobierze jednak sam listy firm z wybranego miasta, nie przetworzy ich hurtowo tym samym, spójnym promptem, nie zapisze wyników w arkuszu i nie przygotuje szkiców w skrzynce. Tu cały proces biegnie bez mojego udziału:

- dane o firmach pobierane są automatycznie z zewnętrznego źródła,
- każda firma przechodzi przez tę samą ocenę (skala 1-10 z uzasadnieniem),
- wszystkie wyniki trafiają do Google Sheets,
- tylko firmy z oceną co najmniej 7 dostają szkic maila w Gmailu,
- mail nie jest wysyłany sam, czeka na moją akceptację.

To odpowiednik automatyzacji, którą wcześniej zbudowałem w Zapierze (scraping firm z okolicy, ocena AI, zapis do arkusza, mail), tym razem w n8n i z większą kontrolą nad każdym krokiem.

### Cel lekcji

- zbudowanie wieloetapowego pipeline'u: źródło danych, oczyszczenie, ocena AI, zapis, decyzja, akcja,
- pobieranie prawdziwych danych z zewnętrznego scrapera (Apify, Google Maps Scraper) przez HTTP Request,
- wymuszenie na OpenAI odpowiedzi w formacie JSON i wykorzystanie jej w kolejnych węzłach,
- dopasowanie węzła Code do rzeczywistej struktury danych ze scrapera,
- tworzenie szkiców w Gmailu zamiast automatycznej wysyłki.

---

## Część 1 – Wersja testowa na danych przykładowych (7 węzłów)

Najpierw cały przepływ budowany jest na darmowych danych testowych. Dzięki temu mogę sprawdzić logikę (Code, OpenAI, arkusz, warunek, szkic), zanim zacznę płacić za prawdziwe dane.

## Krok 1 – Schedule Trigger

Węzeł decyduje, kiedy proces ma się uruchamiać.

- Trigger Interval: `Days`
- Days Between Runs: `1`
- At Time: `09:00`

Na czas testów workflow uruchamiam ręcznie przyciskiem **Execute workflow**.

## Krok 2 – HTTP Request (źródło testowe)

Węzeł pobiera dane firm z zewnętrznego źródła.

- Method: `GET`
- URL: `https://jsonplaceholder.typicode.com/users` (darmowy generator danych testowych)
- Authentication: `None`
- Send Headers / Query Parameters: wyłączone

Po wklejeniu adresu klikam Test step, żeby upewnić się, że dane się pobierają.

## Krok 3 – Code (czyszczenie i przygotowanie danych)

Węzeł porządkuje surowe dane i zostawia tylko pola, których potrzebuję.

- Language: `JavaScript`
- Mode: `Run Once for All Items`

```javascript
return $input.all().map(item => {
  const raw = item.json;

  return {
    json: {
      company_name: raw.company?.name || raw.name || "Brak nazwy",
      website: raw.website || "Brak strony",
      industry: raw.company?.bs || "Usługi B2B",
      description: raw.company?.catchPhrase || "Brak opisu",
      contact_email: raw.email || "test@example.com"
    }
  };
});
```

Po Test step po prawej stronie widać czyste pola: `company_name`, `website`, `industry`, `description`, `contact_email`.

## Krok 4 – OpenAI (ocena AI i treść maila)

Węzeł ocenia potencjał firmy w skali 1-10 i pisze propozycję maila.

- Resource: `Text`
- Operation: `Message a Model` (w niektórych wersjach `Complete`)
- Credential: klucz OpenAI API
- Model: `gpt-4o-mini`
- Prompt: `Define Below`

**Wiadomość 1, rola System:**

```
Jesteś analitykiem B2B. Twoim zadaniem jest ocena potencjału firmy pod kątem wdrożeń automatyzacji AI w skali 1-10 oraz napisanie propozycji e-maila.
Odpowiadaj ZAWSZE w czystym formacie JSON bez formatowania Markdown.
```

**Wiadomość 2 (Add Message), rola User:**

```
Przeanalizuj poniższe dane firmy:
Nazwa: {{ $json.company_name }}
Strona: {{ $json.website }}
Branża: {{ $json.industry }}
Opis: {{ $json.description }}

Zwróć wynik w formacie JSON o takiej strukturze:
{
  "score": 8,
  "reasoning": "Tutaj krótkie uzasadnienie oceny",
  "email_subject": "Propozycja automatyzacji dla {{ $json.company_name }}",
  "email_body": "Tutaj treść spersonalizowanego e-maila"
}
```

**Format odpowiedzi:** na dole węzła Options, Add Option, **Response Format**, ustawione na `JSON Object`. To technicznie wymusza poprawny JSON, zamiast polegać wyłącznie na instrukcji w prompcie.

Po Test step wynik zawiera obiekt z polami `score`, `reasoning`, `email_subject` i `email_body`.

## Krok 5 – Google Sheets (zapis wszystkich ocenionych firm)

- Resource: `Document`, Operation: `Append Row`
- Document: wybrany plik, Sheet: zakładka (np. `Leady`)
- Data Mode: `Map Each Column Below`

W pierwszym wierszu arkusza muszą być nagłówki: `Nazwa`, `E-mail`, `Ocena`, `Uzasadnienie`.

Mapowanie (każde pole w trybie Expression):

| Kolumna | Wyrażenie |
|---|---|
| Nazwa | `{{ $('Code').item.json.company_name }}` |
| E-mail | `{{ $('Code').item.json.contact_email }}` |
| Ocena | `{{ JSON.parse($json.message.content).score }}` |
| Uzasadnienie | `{{ JSON.parse($json.message.content).reasoning }}` |

Dane firmy biorę jawnie z węzła Code (`$('Code')`), a ocenę z odpowiedzi OpenAI. To ta sama zasada, którą wypracowałem wcześniej: po drodze węzły podmieniają zawartość bieżącego itemu, więc do wcześniejszych danych odwołuję się po nazwie węzła.

Uwaga: zależnie od wersji węzła OpenAI pole `message.content` może być tekstem (wtedy `JSON.parse` jest potrzebne) albo już gotowym obiektem (wtedy `JSON.parse` trzeba pominąć). Zawsze sprawdzam w output węzła, jak to wygląda.

Po Test step w arkuszu powinien pojawić się nowy wiersz.

## Krok 6 – If (czy ocena wynosi co najmniej 7)

- Condition type: `Number`
- Value 1 (Expression): `{{ JSON.parse($('OpenAI').item.json.message.content).score }}`
- Operation: `Larger or Equal` (>=)
- Value 2: `7`

Węzeł ma dwa wyjścia: `true` i `false`. Do `true` podpinam Gmail, `false` zostawiam puste: słabsze firmy są zapisane w arkuszu, ale nie dostają szkicu.

## Krok 7 – Gmail (szkic maila)

Dla firm z oceną 7 lub wyższą tworzę wersję roboczą w skrzynce. Mail nie wysyła się sam.

- Resource: `Draft`, Operation: `Create Draft`
- To (Expression): `{{ $('Code').item.json.contact_email }}`
- Subject (Expression): `{{ JSON.parse($('OpenAI').item.json.message.content).email_subject }}`
- Email Type: `Plain Text`
- Message (Expression): `{{ JSON.parse($('OpenAI').item.json.message.content).email_body }}`

Po Test step sprawdzam folder Szkice (Drafts) w Gmailu.

---

## Część 2 – Przejście na prawdziwe dane: Apify i Google Maps

Darmowe API testowe nie pozwala przeszukiwać prawdziwych firm. Do tego służy scraper. Wybrałem Apify z gotowym narzędziem (Actor) **Google Maps Scraper**.

## Krok 8 – Konfiguracja scrapera w Apify

1. Zakładam konto na Apify.com (plan darmowy z miesięcznym kredytem na zużycie).
2. W wyszukiwarce Apify znajduję **Google Maps Scraper** i otwieram jego formularz Input.
3. **Search term(s):** branża, której szukam, np. `agencja marketingowa`, `sklep internetowy`, `software house`.
4. **Location:** miasto, tu `Katowice, Poland`.
5. **Max items:** niska wartość na start (10), żeby nie zużywać kredytu.
6. **Add-on: Company contacts enrichment (from website) ($):** włączam przełącznik (zmienia kolor na niebieski). Scraper wchodzi wtedy na strony www firm i wyciąga z nich adresy e-mail. To płatny dodatek, zużywa kredyt Apify.
7. **Add-on: Search filters & categories:** zaznaczam opcję scrapowania tylko miejsc, które mają stronę internetową, żeby nie płacić za wizytówki bez strony (bez strony nie ma skąd pobrać maila).
8. Klikam zielony przycisk **Save & start** (Start). Po kilkunastu sekundach (z dodatkiem do kontaktów dłużej) status zmienia się na `SUCCEEDED`.

## Krok 9 – Wyciągnięcie linku API do węzła HTTP Request

1. Po zakończeniu pracy scrapera klikam w prawym górnym rogu przycisk **API**.
2. Wybieram zakładkę **Get dataset items**.
3. Kopiuję cały adres URL, który wygląda tak:

```
https://api.apify.com/v2/datasets/TWOJE_ID_DATASETU/items?token=TWOJ_TOKEN_APIFY
```

4. W n8n w węźle **HTTP Request** wklejam go w pole URL (Method `GET`, Authentication `None`, bo token jest już w adresie) i klikam Execute step.

Token w adresie traktuję jak hasło i nie publikuję go w portfolio ani na zrzutach ekranu.

## Krok 10 – Węzeł Code dopasowany do prawdziwych danych

Dane ze scrapera Google Maps mają inne nazwy pól niż dane testowe (`title`, `website`, `categoryName`, `address`, `totalScore`, `reviewsCount`, a e-mail w jednym z kilku możliwych pól). Węzeł Code z Kroku 3 zamieniam na wersję finalną:

```javascript
return $input.all().map(item => {
  const raw = item.json;

  let foundEmail = raw.email || raw.contactEmail || raw.supportEmail || (raw.emails && raw.emails[0]) || null;

  return {
    json: {
      company_name: raw.title || raw.name || "Nieznana firma",
      website: raw.website || "Brak strony",
      industry: raw.categoryName || raw.industry || "B2B",
      description: `Adres: ${raw.address || 'Brak'}. Ocena Google: ${raw.totalScore || 'Brak'} (${raw.reviewsCount || 0} opinii).`,
      contact_email: foundEmail || "kontakt@praktyczna-automatyzacja.pl"
    }
  };
});
```

Kod sprawdza kolejno kilka możliwych pól z adresem e-mail (scraper z dodatkiem do kontaktów zwraca je w różnych miejscach) i bierze pierwsze, które istnieje. Opis firmy składam z adresu, oceny Google i liczby opinii, bo to dane, którymi AI może się posłużyć w personalizacji.

## Krok 11 (opcjonalnie) – Dodatkowe wzbogacanie e-maili przez Hunter.io

Jeśli pole `contact_email` wraca puste, przed węzłem OpenAI można dodać pomocniczy węzeł **HTTP Request** odpytujący Hunter.io o adresy powiązane z domeną firmy:

```
https://api.hunter.io/v2/domain-search?domain={{ $json.website }}&api_key=TWOJ_KLUCZ_HUNTER
```

To propozycja rozszerzenia, której w tej wersji workflowa nie wdrażałem. Hunter zwraca adresy znalezione publicznie wraz z oceną pewności, więc nie traktuję ich jako gwarantowanie poprawnych.

## Krok 12 – Prompt pod prawdziwy cold mailing

Żeby maile nie brzmiały jak generyczny spam, zmieniam prompt w OpenAI na bardziej naturalny i konkretny.

**Rola System:**

```
Jesteś ekspertem cold mailingu B2B. Piszesz krótkie (max 4-5 zdań), naturalne wiadomości e-mail. Nie używasz zwrotów handlowych typu "Mam nadzieję, że ten e-mail zastaje Cię w dobrym zdrowiu". Piszesz konkretnie, jak człowiek do człowieka.
```

**Rola User:**

```
Przeanalizuj firmę:
Nazwa: {{ $json.company_name }}
Strona: {{ $json.website }}
Opis z Google: {{ $json.description }}

Wygeneruj JSON w strukturze:
{
  "score": 8,
  "reasoning": "Uzasadnienie po polsku",
  "email_subject": "Szybkie pytanie ws. {{ $json.company_name }}",
  "email_body": "Treść maila"
}

Pole score to ocena potencjału firmy na usługi IT i automatyzacji w skali 1-10.

Przykład treści maila (zachowaj ten styl i długość):
"Cześć,\n\nZauważyłem Wasz profil w Google ({{ $json.company_name }}). Zajmuję się wdrażaniem automatyzacji, które pomagają firmom z branży {{ $json.industry }} oszczędzać czas na ręcznej obsłudze zgłoszeń. Czy znaleźlibyście 5 minut w tym tygodniu na krótką rozmowę, żeby sprawdzić, czy u Was też da się to wdrożyć?\n\nPozdrawiam,\nDaniel"
```

---

## Napotkane problemy i naprawy

### Problem 1: workflow nie wygenerował żadnych szkiców, a w Code zamiast danych widać kod HTML

Na panelu INPUT węzła Code zamiast tablicy z firmami pojawiał się surowy kod strony (`<!DOCTYPE html> ... <title>Apify Console</title>`).

**Przyczyna:** w węźle HTTP Request wklejony był adres strony konsoli Apify (`console.apify.com`), a nie adres API zbioru danych (Dataset). Zamiast JSON-a z firmami n8n pobrał stronę logowania, więc dalsze węzły nie miały żadnych prawdziwych danych do przetworzenia.

**Naprawa:** pobranie właściwego linku z przycisku **API, Get dataset items** w Apify (adres zaczynający się od `https://api.apify.com/v2/datasets/...`, nie od `console.apify.com`). Po wklejeniu go do HTTP Request panel INPUT w Code pokazuje czystą tablicę JSON z nazwami firm z wybranego miasta.

### Problem 2: scraper zwracał firmy bez adresów e-mail

Samo Google Maps podaje stronę www, ale zwykle nie podaje adresu e-mail. Bez maila nie ma dokąd adresować szkicu.

**Naprawa:** włączenie w Apify dodatku **Company contacts enrichment (from website)**, dzięki któremu scraper wchodzi na strony firm i wyciąga z nich adresy e-mail, oraz filtra ograniczającego wyniki do miejsc posiadających stronę internetową. Po zakończeniu pracy scrapera trzeba skopiować nowy link do datasetu, bo każde uruchomienie tworzy nowy zbiór danych.

### Problem 3: błąd `Unexpected token ')'` w węźle Code

Przyczyną był nadmiarowy nawias zamykający (`)` albo `}`) skopiowany przypadkiem na końcu kodu przy wklejaniu.

**Naprawa:** zaznaczenie całej zawartości edytora kodu, skasowanie jej i wklejenie od nowa kompletnego kodu z Kroku 10, z równymi parami nawiasów. Po ponownym uruchomieniu błąd zniknął.

---

## Test

**Test 1 – dane przykładowe:** workflow z JSONPlaceholder (Kroki 1-7). Sprawdzam: czy Code zwraca czyste pola, czy OpenAI zwraca JSON z `score`, czy w arkuszu pojawiają się wiersze, czy szkice powstają tylko dla ocen 7 i wyższych.

**Test 2 – prawdziwe dane:** po podmianie URL na dataset z Apify (10 firm z Katowic). Sprawdzam: czy w INPUT węzła Code widać JSON, a nie HTML, czy każda firma ma wypełnione pola, czy szkice w Gmailu mają właściwego adresata i spersonalizowany temat.

**Test 3 – próg oceny:** firma z niską oceną musi trafić do arkusza, ale nie może dostać szkicu.

---

## Uwagi, ograniczenia i dobre praktyki

- **Stały link do datasetu.** W tej wersji wklejam link do konkretnego, już ukończonego uruchomienia scrapera. Przy uruchamianiu workflowa codziennie (Schedule Trigger) ten sam link zwracałby te same firmy za każdym razem i duplikowałby wiersze w arkuszu. Docelowo scraper powinien być uruchamiany z poziomu n8n przez API Apify (albo zaplanowanym zadaniem w Apify), a workflow powinien pobierać świeże wyniki.
- **Adres zastępczy w Code.** W wersji finalnej, gdy scraper nie znajdzie maila, kod wpisuje adres zastępczy. Szkic trafiłby wtedy na adres, który nie należy do tej firmy. Bezpieczniejsze podejście: wpisać pustą wartość i dodać do węzła If drugi warunek (AND) `contact_email is not empty`, żeby szkice powstawały tylko dla firm z realnym adresem.
- **Sztywna personalizacja w prompcie.** Wcześniejsza wersja szablonu maila zakładała zdanie o dużej liczbie opinii. Dla firm z małą liczbą opinii byłoby to nieprawdziwe. W prompcie warto dodać polecenie, żeby model odwoływał się wyłącznie do faktów obecnych w danych.
- **Format odpowiedzi OpenAI.** Zależnie od wersji węzła `message.content` jest tekstem lub obiektem, więc przed użyciem `JSON.parse` sprawdzam output.
- **Koszty.** Zużycie kredytu Apify zależy od liczby firm i od włączonych dodatków (kontakty z witryn są płatne), więc zaczynam od małego limitu. Koszt OpenAI przy `gpt-4o-mini` przy kilkudziesięciu firmach jest pomijalny.
- **Prawo i etyka.** Workflow tworzy wyłącznie szkice, więc każdy mail przechodzi przeze mnie przed wysyłką. Zanim zacznę faktycznie wysyłać zimne maile do firm, sprawdzam zasady RODO i przepisy o wysyłaniu informacji handlowych drogą elektroniczną (zgoda na kontakt marketingowy, obowiązek informacyjny, możliwość rezygnacji). To nie jest porada prawna, w razie wątpliwości konsultuję się ze specjalistą. Dane ze scraperów pochodzą z publicznych wizytówek, ale warto też znać regulamin używanego narzędzia.

---

## Pełny schemat workflowa

```
Schedule Trigger (codziennie 09:00)
        │
        ▼
HTTP Request (GET, Apify: Get dataset items)
        │
        ▼
Code (company_name, website, industry, description, contact_email)
        │
        ▼
OpenAI (gpt-4o-mini, JSON: score, reasoning, email_subject, email_body)
        │
        ▼
Google Sheets (Append Row: Nazwa, E-mail, Ocena, Uzasadnienie)
        │
        ▼
      If (score >= 7 ?)
     /            \
  true            false
    │               │
    ▼               ▼
Gmail (Create Draft)  (koniec)
```

Źródło danych, przygotowywane osobno w Apify:

```
Apify: Google Maps Scraper
(Search term + Location: Katowice, Poland + Max items: 10
 + Add-on: Company contacts enrichment
 + filtr: tylko miejsca ze stroną www)
        │
        ▼
Dataset (SUCCEEDED) → przycisk API → Get dataset items → URL
        │
        ▼
Węzeł HTTP Request w n8n
```

---

## Czego się nauczyłem

- budowania pełnego pipeline'u B2B w n8n: źródło danych, oczyszczenie, ocena AI, zapis, warunek i akcja
- że warto zaczynać od danych testowych (JSONPlaceholder) i dopiero po sprawdzeniu logiki przejść na płatne, prawdziwe źródło
- konfiguracji scrapera Google Maps w Apify: wyszukiwane frazy, lokalizacja, limit wyników, dodatek do wyciągania kontaktów ze stron firm i filtr miejsc ze stroną www
- jak wyciągnąć adres API zbioru danych z Apify (API, Get dataset items) i że link do konsoli (`console.apify.com`) zwraca stronę HTML zamiast danych
- jak rozpoznać taki błąd: na INPUT węzła Code widać `<!DOCTYPE html>` zamiast JSON-a
- pisania węzła Code, który dopasowuje różne nazwy pól ze scrapera do jednego, spójnego schematu (kilka możliwych pól na adres e-mail, wartości domyślne)
- wymuszania na OpenAI odpowiedzi w formacie JSON (Response Format) i odczytywania jej w kolejnych węzłach, z jawnym odwoływaniem się do węzłów po nazwie
- że szkic w Gmailu z ręczną akceptacją jest bezpieczniejszy niż automatyczna wysyłka zimnych maili
- że workflow demonstracyjny trzeba świadomie rozróżniać od produkcyjnego: stały link do datasetu, adres zastępczy i sztywny szablon maila to uproszczenia, które trzeba zastąpić przed realnym użyciem
