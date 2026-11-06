# N8N - Automatyczne uzupełnianie zapasów magazynowych

## Projekt: automatyczne zamówienia uzupełniające u dostawców

Firma, która pracuje na magazynie, musi regularnie kontrolować stany
produktów. Kiedy któregoś towaru zaczyna brakować, ktoś musi sprawdzić
poziom zapasu, ustalić ile należy domówić, znaleźć właściwego dostawcę i
wysłać zamówienie. Przy większej liczbie produktów to powtarzalna praca,
którą łatwo przeoczyć albo wykonać podwójnie.

Ten workflow przejmuje ten proces:

``` text
HARMONOGRAM — DNI ROBOCZE, 7:00
              ↓
Google Sheets — odczyt stanów magazynowych
              ↓
Code — sprawdzenie braków i blokady ponownych zamówień
              ↓
Code — grupowanie pozycji według dostawcy
              ↓
Gmail — jeden mail HTML do każdego dostawcy
              ↓
Code — rozbicie zamówień na pojedyncze pozycje
              ↓
       ┌──────┴──────┐
       ↓             ↓
Google Sheets    Google Sheets
aktualizacja     dziennik
magazynu         zamówień
```

Najważniejsze założenie: workflow nie wysyła osobnego maila dla każdego
produktu. Najpierw grupuje pozycje według dostawcy, a następnie wysyła
jeden zbiorczy mail z tabelą zamówienia. Dodatkowo zapisuje datę
ostatniego zamówienia, żeby przy kolejnym uruchomieniu nie wysłać
ponownie tego samego zamówienia.

Na etapie budowy wszystkie wiadomości trafiają na mój adres testowy.
Dopiero po sprawdzeniu działania można wyłączyć tryb testowy.

## Cel lekcji

-   praca z harmonogramem i wyrażeniem cron,
-   ustawienie strefy czasowej workflowa,
-   odczyt wielu wierszy z Google Sheets,
-   filtrowanie danych i grupowanie ich w node'zie Code,
-   budowanie wiadomości HTML z tabelą,
-   wysyłanie jednego maila dla wielu pozycji tego samego dostawcy,
-   blokada ponownych zamówień na podstawie daty,
-   rozbijanie zgrupowanych danych na pojedyncze pozycje,
-   rozgałęzienie jednego strumienia danych do dwóch niezależnych akcji,
-   aktualizacja arkusza i prowadzenie dziennika zamówień,
-   testowanie workflowa bez kontaktowania prawdziwych dostawców.

------------------------------------------------------------------------

## Krok 0 -- Przygotowanie Google Sheets

Tworzę nowy plik `Magazyn - zapasy` z dwiema zakładkami.

### Zakładka `Magazyn`

W pierwszym wierszu dodaję nagłówki dokładnie w takiej pisowni:

`SKU` \| `Produkt` \| `Stan` \| `Min` \| `Cel` \| `Dostawca` \| `Email`
\| `OstatnieZamowienie`

Znaczenie najważniejszych kolumn:

-   `Stan` -- aktualna liczba sztuk na magazynie,
-   `Min` -- poziom minimalny; jeżeli stan jest mniejszy lub równy tej
    wartości, trzeba zamówić towar,
-   `Cel` -- stan, do którego chcemy uzupełnić zapas,
-   `OstatnieZamowienie` -- data ostatniego zamówienia danej pozycji.

Ilość do zamówienia jest obliczana jako `Cel - Stan`. Jeżeli wynik byłby
mniejszy niż 1, kod i tak ustawia minimalną ilość zamówienia na 1.

Kolumnę `OstatnieZamowienie` ustawiam jako zwykły tekst:

1.  Zaznaczam całą kolumnę.
2.  Wybieram **Format → Liczba → Zwykły tekst**.

Dzięki temu Google Sheets nie powinien samodzielnie zmieniać zapisu
daty.

### Dane testowe

Wszystkie dane są fikcyjne. W kolumnie `Email` wpisuję we wszystkich
wierszach własny adres testowy.

  ------------------------------------------------------------------------------------------
  SKU       Produkt         Stan     Min     Cel Dostawca     Email     OstatnieZamowienie
  --------- ------------ ------- ------- ------- ------------ --------- --------------------
  KAR-001   Karton            40     100     400 Alfa         mój adres 
            30x20x10                             Opakowania   testowy   

  KAR-002   Karton           250     100     400 Alfa         mój adres 
            40x30x20                             Opakowania   testowy   

  TAS-010   Taśma pakowa      12      30     120 Alfa         mój adres 
            48 mm                                Opakowania   testowy   

  FOL-005   Folia              8      10      40 Beta Pack    mój adres 
            stretch 500                                       testowy   
            mm                                                          

  WYP-020   Wypełniacz        90      50     200 Beta Pack    mój adres 
            papierowy                                         testowy   

  KOP-030   Koperty           15      20     100 Beta Pack    mój adres wczorajsza data
            bąbelkowe                                         testowy   

  ETY-100   Etykiety           3       5      20 Gamma Druk   mój adres 
            100x150                                           testowy   
            (rolka)                                                     
  ------------------------------------------------------------------------------------------

### Zakładka `Zamowienia`

To dziennik zamówień. Dodaję nagłówki:

`Data` \| `SKU` \| `Produkt` \| `Ilosc` \| `Dostawca` \| `Status`

Oczekiwany wynik pierwszego uruchomienia:

-   3 maile: Alfa Opakowania, Beta Pack i Gamma Druk,
-   4 pozycje do zamówienia:
    -   KAR-001 -- 360 szt.,
    -   TAS-010 -- 108 szt.,
    -   FOL-005 -- 32 szt.,
    -   ETY-100 -- 17 szt.

Pozostałe pozycje nie powinny przejść dalej:

-   KAR-002 i WYP-020 mają stan powyżej minimum,
-   KOP-030 był zamawiany niedawno i blokada ma go zatrzymać.

------------------------------------------------------------------------

## Krok 1 -- Schedule Trigger (harmonogram)

Tworzę nowy workflow o nazwie:

`Uzupelnianie zapasow magazynowych`

Dodaję node **Schedule Trigger**.

Ustawienia:

-   Trigger Rules → Add Rule,
-   Trigger Interval: `Custom (Cron)`,
-   Expression: `0 7 * * 1-5`.

Wyrażenie oznacza:

-   minuta: `0`,
-   godzina: `7`,
-   dzień miesiąca: każdy,
-   miesiąc: każdy,
-   dzień tygodnia: od poniedziałku do piątku.

Czyli workflow ma uruchamiać się w dni robocze o 7:00.

### Strefa czasowa

W ustawieniach workflowa klikam trzy kropki w prawym górnym rogu,
przechodzę do **Settings** i w polu **Timezone** ustawiam:

`Europe/Warsaw`

To ważne, ponieważ bez właściwej strefy czasowej harmonogram może
uruchamiać się o innej godzinie, niż zakładam.

Podczas budowy uruchamiam workflow ręcznie przyciskiem **Execute
workflow**. Harmonogram będzie potrzebny po zakończeniu testów i
aktywowaniu workflowa.

## Krok 2 -- Google Sheets: odczyt stanów magazynowych

Dodaję node **Google Sheets** i nazywam go:

`Odczyt magazynu`

Ustawienia:

-   Resource: `Sheet Within Document`,
-   Operation: `Get Row(s)`,
-   Document: `Magazyn - zapasy`,
-   Sheet: `Magazyn`,
-   Filters: bez filtrów --- pobieram wszystkie wiersze.

Połączenie: `Schedule Trigger → Odczyt magazynu`.

**Test step:** powinno wrócić 7 wierszy z polami `SKU`, `Produkt`,
`Stan`, `Min`, `Cel`, `Dostawca`, `Email` i `OstatnieZamowienie`.

## Krok 3 -- Code: wykrywanie braków i grupowanie dostawców

To najważniejsza część workflowa. Dodaję node **Code** o nazwie:

`Znajdz braki`

Ustawienia:

-   Language: `JavaScript`,
-   Mode: `Run Once for All Items`.

Wklejam kod:

``` javascript
// ===== USTAWIENIA =====
const TRYB_TESTOWY = true;                  // true = wszystkie maile idą na TEST_EMAIL
const TEST_EMAIL = 'twoj.adres@gmail.com';  // wpisz swój adres
const DNI_BLOKADY = 3;                      // ile dni nie ponawiamy zamówienia tej samej pozycji

const dzis = new Date();
dzis.setHours(0, 0, 0, 0);
const dataTekst = `${String(dzis.getDate()).padStart(2, '0')}.${String(dzis.getMonth() + 1).padStart(2, '0')}.${dzis.getFullYear()}`;

const esc = s => String(s ?? '').replace(/&/g, '&amp;').replace(/</g, '&lt;').replace(/>/g, '&gt;');

function liczba(v) {
  const n = Number(String(v ?? '').replace(/\s/g, '').replace(',', '.'));
  return Number.isNaN(n) ? 0 : n;
}

function parseData(s) {
  if (!s) return null;
  s = String(s).trim();
  let m = s.match(/^(\d{4})-(\d{2})-(\d{2})/);
  if (m) return new Date(+m[1], +m[2] - 1, +m[3]);
  m = s.match(/^(\d{1,2})\.(\d{1,2})\.(\d{4})/);
  if (m) return new Date(+m[3], +m[2] - 1, +m[1]);
  return null;
}

// ===== 1. Szukamy pozycji do zamówienia i grupujemy po dostawcy =====
const dostawcy = {};

for (const item of $input.all()) {
  const r = item.json;
  const sku = String(r['SKU'] || '').trim();
  if (!sku) continue;

  const stan = liczba(r['Stan']);
  const min = liczba(r['Min']);
  const cel = liczba(r['Cel']);

  if (stan > min) continue;                    // stan w normie

  const ostatnie = parseData(r['OstatnieZamowienie']);
  if (ostatnie) {
    const dni = Math.floor((dzis - ostatnie) / 86400000);
    if (dni < DNI_BLOKADY) continue;           // niedawno zamawiane
  }

  const email = String(r['Email'] || '').trim();
  if (!email) continue;                        // bez adresu nie ma gdzie wysłać

  const nazwa = String(r['Dostawca'] || 'Brak dostawcy').trim();
  const klucz = nazwa + '|' + email;
  if (!dostawcy[klucz]) dostawcy[klucz] = { dostawca: nazwa, email, pozycje: [] };

  dostawcy[klucz].pozycje.push({
    sku,
    produkt: r['Produkt'],
    stan,
    min,
    ilosc: Math.max(cel - stan, 1)
  });
}

// ===== 2. Dla każdego dostawcy budujemy jeden mail =====
const komorka = 'padding:6px 10px;border:1px solid #ddd';
const wynik = [];

for (const d of Object.values(dostawcy)) {
  const wiersze = d.pozycje.map(p =>
    `<tr>
      <td style="${komorka}">${esc(p.sku)}</td>
      <td style="${komorka}">${esc(p.produkt)}</td>
      <td style="${komorka};text-align:right">${p.stan}</td>
      <td style="${komorka};text-align:right"><b>${p.ilosc}</b></td>
    </tr>`
  ).join('');

  const html = `
<p>Dzień dobry,</p>
<p>prosimy o realizację zamówienia uzupełniającego z dnia ${dataTekst}:</p>
<table style="border-collapse:collapse;font-family:Arial,sans-serif;font-size:14px">
  <thead>
    <tr style="background:#f2f2f2">
      <th style="${komorka};text-align:left">SKU</th>
      <th style="${komorka};text-align:left">Produkt</th>
      <th style="${komorka}">Stan obecny</th>
      <th style="${komorka}">Zamawiana ilość</th>
    </tr>
  </thead>
  <tbody>${wiersze}</tbody>
</table>
<p>Prosimy o potwierdzenie przyjęcia zamówienia i podanie przewidywanego terminu dostawy.</p>
<p>Pozdrawiam<br>Dział zakupów</p>`;

  wynik.push({
    json: {
      dostawca: d.dostawca,
      emailDostawcy: d.email,
      emailDo: TRYB_TESTOWY ? TEST_EMAIL : d.email,
      temat: `${TRYB_TESTOWY ? '[TEST] ' : ''}Zamówienie uzupełniające ${dataTekst} - ${d.dostawca}`,
      liczbaPozycji: d.pozycje.length,
      html,
      pozycje: d.pozycje
    }
  });
}

return wynik;
```

W kodzie zmieniam `twoj.adres@gmail.com` na własny adres testowy.

### Jak działa ten kod

1.  Pobiera wszystkie wiersze z poprzedniego node'a.
2.  Pomija wiersze bez SKU.
3.  Odczytuje stan, minimum i docelowy poziom zapasu.
4.  Pomija produkt, jeżeli `Stan` jest większy niż `Min`.
5.  Sprawdza datę ostatniego zamówienia. Jeśli minęło mniej niż 3 dni,
    pozycja nie przechodzi dalej.
6.  Pomija pozycje bez adresu e-mail.
7.  Grupuje pozostałe pozycje według nazwy dostawcy i adresu e-mail.
8.  Buduje jeden mail HTML dla każdego dostawcy.
9.  W trybie testowym podmienia adres odbiorcy na mój adres testowy.

Dzięki grupowaniu cztery pozycje od jednego dostawcy nie generują
czterech osobnych wiadomości.

**Test step:** dla danych testowych powinny wrócić 3 elementy: Alfa
Opakowania, Beta Pack i Gamma Druk. Każdy element zawiera m.in.
`emailDo`, `temat`, `html` i tablicę `pozycje`.

Jeśli nic nie wymaga zamówienia, Code zwraca pustą listę i dalsza część
workflowa nie otrzymuje żadnych elementów.

## Krok 4 -- Gmail: wysyłka maila do dostawcy

Dodaję node **Gmail** i nazywam go:

`Mail do dostawcy`

Ustawienia:

-   Resource: `Message`,
-   Operation: `Send`,
-   To (Expression): `{{ $json.emailDo }}`,
-   Subject (Expression): `{{ $json.temat }}`,
-   Email Type: `HTML`,
-   Message (Expression): `{{ $json.html }}`,
-   Options → Add Option: wyłączam `Append n8n Attribution`.

Opcjonalnie w polu CC można dodać własny adres lub adres działu zakupów.

Połączenie: `Znajdz braki → Mail do dostawcy`.

Node wykona się osobno dla każdego elementu, czyli raz dla każdego
dostawcy.

**Test step:** w skrzynce testowej powinny pojawić się 3 maile z
tabelami i tematem zaczynającym się od `[TEST]`.

## Krok 5 -- Code: rozbicie zamówień na pojedyncze pozycje

Po wysłaniu maili mam dane pogrupowane według dostawców. Do aktualizacji
magazynu i dziennika potrzebuję jednak osobnego elementu dla każdego
SKU.

Dodaję node **Code** o nazwie:

`Rozbij na pozycje`

Ustawienie:

-   Mode: `Run Once for All Items`.

Kod:

``` javascript
const wynik = [];

for (const dostawca of $('Znajdz braki').all()) {
  const d = dostawca.json;

  for (const p of d.pozycje) {
    wynik.push({
      json: {
        data: $now.toFormat('yyyy-MM-dd'),
        sku: p.sku,
        produkt: p.produkt,
        ilosc: p.ilosc,
        dostawca: d.dostawca,
        email: d.emailDostawcy,
        status: 'Wysłano'
      }
    });
  }
}

return wynik;
```

Dane pobieram bezpośrednio z node'a `Znajdz braki`, używając jego nazwy.
Po Gmailu bieżący element zawiera już dane wysłanej wiadomości, a nie
pełną listę pozycji zamówienia.

Ten node wykonuje się za node'em Gmail. Dzięki temu oznaczenie pozycji i
zapis do dziennika następują dopiero po przejściu przez etap wysyłki.

**Test step:** powinny wrócić 4 elementy:

-   KAR-001,
-   TAS-010,
-   FOL-005,
-   ETY-100.

Każdy element ma osobne pola: `data`, `sku`, `produkt`, `ilosc`,
`dostawca`, `email` i `status`.

## Krok 6 -- Google Sheets: oznaczenie zamówionych pozycji w magazynie

Z node'a `Rozbij na pozycje` prowadzę dwa równoległe połączenia.
Pierwsze aktualizuje arkusz magazynowy.

Dodaję node **Google Sheets** o nazwie:

`Oznacz w magazynie`

Ustawienia:

-   Operation: `Update Row`,
-   Document: `Magazyn - zapasy`,
-   Sheet: `Magazyn`,
-   Mapping Column Mode: `Map Each Column Manually`,
-   Column to match on: `SKU`.

W **Values to Update** ustawiam:

-   `SKU` (using to match): `{{ $json.sku }}`,
-   `OstatnieZamowienie`: `{{ $json.data }}`.

Pole `SKU` z dopiskiem „using to match" trzeba wypełnić wyrażeniem. Samo
wybranie kolumny do dopasowania nie oznacza jeszcze, że n8n
automatycznie pobierze właściwą wartość z poprzedniego node'a.

Połączenie:

`Rozbij na pozycje → Oznacz w magazynie`

## Krok 7 -- Google Sheets: dziennik zamówień

Drugie połączenie z tego samego wyjścia node'a `Rozbij na pozycje`
kieruję do kolejnego node'a **Google Sheets**.

Nazwa:

`Dziennik zamowien`

Ustawienia:

-   Operation: `Append Row`,
-   Document: `Magazyn - zapasy`,
-   Sheet: `Zamowienia`.

Mapowanie kolumn:

  Kolumna w arkuszu   Wartość (Expression)
  ------------------- ------------------------
  `Data`              `{{ $json.data }}`
  `SKU`               `{{ $json.sku }}`
  `Produkt`           `{{ $json.produkt }}`
  `Ilosc`             `{{ $json.ilosc }}`
  `Dostawca`          `{{ $json.dostawca }}`
  `Status`            `{{ $json.status }}`

Połączenie:

`Rozbij na pozycje → Dziennik zamowien`

To jest rozgałęzienie równoległe: te same cztery pozycje trafiają
jednocześnie do aktualizacji magazynu i do dziennika zamówień.

------------------------------------------------------------------------

## Krok 8 -- Testy workflowa

### Test 1 -- pierwsze uruchomienie

Pozostawiam `TRYB_TESTOWY = true` i uruchamiam workflow przyciskiem
**Execute workflow**.

Sprawdzam:

-   czy otrzymałem 3 maile z tabelami,
-   czy w zakładce `Zamowienia` pojawiły się 4 wiersze,
-   czy w zakładce `Magazyn` zapisano daty w kolumnie
    `OstatnieZamowienie` przy pozycjach KAR-001, TAS-010, FOL-005 i
    ETY-100.

### Test 2 -- blokada duplikatów

Uruchamiam workflow ponownie od razu po pierwszym teście.

Oczekiwany wynik:

-   żaden mail nie wychodzi,
-   `Znajdz braki` zwraca pustą listę,
-   workflow kończy się bez błędu.

Po pierwszym uruchomieniu daty zamówienia są już zapisane w arkuszu,
więc pozycje nie powinny ponownie przejść przez blokadę.

### Test 3 -- ponowienie po czasie

W arkuszu cofam datę przy jednej pozycji o 5 dni, np. ustawiam
`2026-10-15`.

Uruchamiam workflow ponownie.

Oczekiwany wynik: tylko ta pozycja wraca do zamówienia, ponieważ minęły
3 dni blokady.

### Test 4 -- brak adresu e-mail

Usuwam adres e-mail przy jednym dostawcy i uruchamiam workflow.

Oczekiwany wynik: pozycje tego dostawcy są pomijane, ponieważ kod nie ma
adresu, na który mógłby wysłać wiadomość.

### Sprzątanie po testach

Po zakończeniu testów:

1.  Czyszczę testowe daty z kolumny `OstatnieZamowienie`.
2.  Czyszczę testowe wiersze z zakładki `Zamowienia`.
3.  Sprawdzam, czy `TEST_EMAIL` zawiera mój adres.
4.  Dopiero po upewnieniu się, że konfiguracja jest poprawna, rozważam
    zmianę `TRYB_TESTOWY` na `false`.

Nie przełączam trybu testowego na `false`, dopóki nie mam pewności, że
arkusz zawiera właściwe adresy dostawców i że workflow został
przetestowany na danych fikcyjnych.

------------------------------------------------------------------------

## Pełny schemat workflowa

``` text
Schedule Trigger
(Cron: 0 7 * * 1-5, Europe/Warsaw)
        │
        ▼
Google Sheets: Odczyt magazynu
(Get Row(s), wszystkie wiersze)
        │
        ▼
Code: Znajdz braki
(filtrowanie + blokada 3 dni
+ grupowanie po dostawcy
+ budowanie maila HTML)
        │
        ▼
Gmail: Mail do dostawcy
(Send, HTML, adres testowy)
        │
        ▼
Code: Rozbij na pozycje
(jeden element na każde SKU)
        │
        ├──────────────► Google Sheets: Oznacz w magazynie
        │                (Update Row, dopasowanie po SKU)
        │
        └──────────────► Google Sheets: Dziennik zamowien
                         (Append Row)
```

------------------------------------------------------------------------

## Napotkane problemy, ograniczenia i możliwe ulepszenia

### 1. Źródło danych

W tej wersji Google Sheets jest ręcznie uzupełnianym źródłem stanów
magazynowych. W realnym zastosowaniu dane mogłyby pochodzić z systemu
magazynowego albo z cyklicznego eksportu CSV. Dalsza część workflowa
mogłaby pozostać taka sama, o ile dane wejściowe zachowają odpowiednią
strukturę.

### 2. Brak adresu dostawcy

Pozycje bez adresu e-mail są pomijane przez kod. W wersji produkcyjnej
warto dodać osobny alert informujący, że produkt wymaga zamówienia, ale
nie ma przypisanego kontaktu do dostawcy.

### 3. Częściowa awaria podczas wysyłki

Jeżeli Gmail wyśle mail do pierwszego dostawcy, a przy drugim wystąpi
błąd, workflow może zatrzymać się przed aktualizacją arkusza. Wtedy
zamówienie pierwszego dostawcy mogłoby zostać wysłane ponownie przy
kolejnym uruchomieniu.

To świadome uproszczenie tej wersji. W bardziej odpornej wersji
należałoby rejestrować status wysyłki osobno dla każdego dostawcy i
aktualizować pozycje po potwierdzeniu wysłania konkretnego maila.

### 4. Automatyczne zamówienie a zatwierdzenie przez człowieka

Obecna wersja wysyła wiadomości automatycznie. Przy prawdziwych
dostawcach bezpieczniejszym rozwiązaniem może być tworzenie szkiców
wiadomości (Draft), które pracownik sprawdza i zatwierdza przed
wysłaniem.

### 5. Dane firmowe

Do testów i materiałów publikowanych w portfolio używam wyłącznie
fikcyjnych produktów, dostawców, stanów magazynowych i adresów e-mail.
Prawdziwe dane pracodawcy nie powinny trafiać do publicznych zrzutów
ekranu ani dokumentacji.

------------------------------------------------------------------------

## Przez co przeszliśmy w tym workflow

-   konfigurowania Schedule Trigger z własnym wyrażeniem cron i
    ustawiania strefy czasowej workflowa,
-   pobierania wielu rekordów z Google Sheets i przetwarzania ich w
    jednym node'zie Code,
-   używania `Run Once for All Items` do pracy na całej liście danych,
-   filtrowania pozycji na podstawie kilku warunków biznesowych,
-   parsowania dat w różnych formatach i wykorzystywania ich do
    blokowania ponownych zamówień,
-   grupowania wielu produktów według dostawcy, aby zamiast wielu
    pojedynczych maili wysyłać jedną zbiorczą wiadomość,
-   generowania wiadomości HTML z tabelą na podstawie danych z arkusza,
-   stosowania trybu testowego, który kieruje wszystkie wiadomości na
    mój adres zamiast do prawdziwych odbiorców,
-   odwoływania się do danych z wcześniejszego node'a po jego nazwie,
    kiedy bieżący node nie zawiera już potrzebnych informacji,
-   rozbijania zgrupowanych danych na pojedyncze elementy, aby móc
    aktualizować każdy produkt osobno,
-   tworzenia rozgałęzienia równoległego, w którym jeden strumień danych
    zasila dwie niezależne akcje,
-   aktualizowania istniejącego wiersza w Google Sheets przez
    dopasowanie po SKU i jednoczesnego dopisywania wpisów do dziennika,
-   projektowania testów obejmujących normalne działanie, blokadę
    duplikatów, ponowienie po czasie i brak adresu dostawcy,
-   rozpoznawania ograniczeń prototypu, zwłaszcza ryzyka częściowej
    awarii podczas wysyłania maili do kilku dostawców.

------------------------------------------------------------------------

# English version

# Automatic Warehouse Stock Replenishment

## Project: automatic replenishment orders to suppliers

A company that operates a warehouse must regularly monitor product stock
levels. When a product starts running low, someone has to check the
stock level, determine how much needs to be reordered, identify the
correct supplier and send an order. With a larger number of products,
this is repetitive work that can easily be overlooked or duplicated.

This workflow takes over that process:

``` text
SCHEDULE — WEEKDAYS AT 7:00
              ↓
Google Sheets — read warehouse stock levels
              ↓
Code — identify shortages and apply reorder restrictions
              ↓
Code — group items by supplier
              ↓
Gmail — send one HTML email to each supplier
              ↓
Code — split orders into individual items
              ↓
       ┌──────┴──────┐
       ↓             ↓
Google Sheets    Google Sheets
update stock     order log
```

The key assumption is that the workflow does not send a separate email
for every product. It first groups items by supplier and then sends one
consolidated email containing an order table. It also records the date
of the last order so that the same order is not sent again during the
next run.

During development, all messages are sent to my test email address. Only
after verifying that everything works should test mode be disabled.

## What this workflow covers

-   configuring Schedule Trigger with a cron expression,
-   setting the workflow time zone,
-   reading multiple rows from Google Sheets,
-   filtering and grouping data in a Code node,
-   building HTML messages with tables,
-   sending one email for multiple items from the same supplier,
-   preventing repeat orders based on dates,
-   splitting grouped data into individual items,
-   branching one data stream into two independent actions,
-   updating the spreadsheet and maintaining an order log,
-   testing the workflow without contacting real suppliers.

------------------------------------------------------------------------

## Step 0 -- Prepare Google Sheets

I create a new file called `Warehouse - stock` with two tabs.

### `Warehouse` tab

In the first row, I add these column headers exactly as written:

`SKU` \| `Product` \| `Stock` \| `Min` \| `Target` \| `Supplier` \|
`Email` \| `LastOrderDate`

Meaning of the most important columns:

-   `Stock` -- the current number of units in the warehouse,
-   `Min` -- the minimum stock level; if stock is less than or equal to
    this value, the product must be reordered,
-   `Target` -- the stock level we want to replenish to,
-   `LastOrderDate` -- the date when the given item was last ordered.

I set the `LastOrderDate` column to plain text:

1.  Select the entire column.
2.  Choose **Format → Number → Plain text**.

This prevents Google Sheets from automatically changing the date format.

### Test data

All data is fictional. In the `Email` column, I enter my own test email
address in every row.

  --------------------------------------------------------------------------------------------
  SKU       Product          Stock        Min     Target Supplier    Email     LastOrderDate
  --------- ----------- ---------- ---------- ---------- ----------- --------- ---------------
  KAR-001   Box                 40        100        400 Alfa        my test   
            30x20x10                                     Packaging   address   

  KAR-002   Box                250        100        400 Alfa        my test   
            40x30x20                                     Packaging   address   

  TAS-010   Packing             12         30        120 Alfa        my test   
            tape 48 mm                                   Packaging   address   

  FOL-005   Stretch              8         10         40 Beta Pack   my test   
            film 500 mm                                              address   

  WYP-020   Paper               90         50        200 Beta Pack   my test   
            filler                                                   address   

  KOP-030   Bubble              15         20        100 Beta Pack   my test   yesterday's
            envelopes                                                address   date

  ETY-100   Labels               3          5         20 Gamma Print my test   
            100x150                                                  address   
            (roll)                                                             
  --------------------------------------------------------------------------------------------

### `Orders` tab

This is the order log. I add the following headers:

`Date` \| `SKU` \| `Product` \| `Quantity` \| `Supplier` \| `Status`

Expected result of the first run:

-   3 emails: Alfa Packaging, Beta Pack and Gamma Print,
-   4 items to reorder:
    -   KAR-001 -- 360 units,
    -   TAS-010 -- 108 units,
    -   FOL-005 -- 32 units,
    -   ETY-100 -- 17 units.

The remaining items should not continue:

-   KAR-002 and WYP-020 have stock levels above the minimum,
-   KOP-030 was ordered recently and should be stopped by the
    restriction.

------------------------------------------------------------------------

## Step 1 -- Schedule Trigger

I create a new workflow called:

`Warehouse stock replenishment`

I add a **Schedule Trigger** node.

Settings:

-   Trigger Rules → Add Rule,
-   Trigger Interval: `Custom (Cron)`,
-   Expression: `0 7 * * 1-5`.

The expression means:

-   minute: `0`,
-   hour: `7`,
-   day of the month: every day,
-   month: every month,
-   day of the week: Monday through Friday.

The workflow should therefore run at 7:00 on weekdays.

### Time zone

In the workflow settings, I click the three dots in the upper-right
corner, open **Settings**, and set the **Timezone** field to:

`Europe/Warsaw`

This is important because without the correct time zone, the schedule
may run at a different time than intended.

During development, I run the workflow manually using **Execute
workflow**. The schedule will be used after testing is complete and the
workflow is activated.

## Step 2 -- Google Sheets: read warehouse stock

I add a **Google Sheets** node and name it:

`Read warehouse`

Settings:

-   Resource: `Sheet Within Document`,
-   Operation: `Get Row(s)`,
-   Document: `Warehouse - stock`,
-   Sheet: `Warehouse`,
-   Filters: none --- I read all rows.

Connection: `Schedule Trigger → Read warehouse`.

**Test step:** 7 rows should be returned with fields such as `SKU`,
`Product`, `Stock`, `Min`, `Target`, `Supplier`, `Email` and
`LastOrderDate`.

## Step 3 -- Code: identify shortages and group suppliers

This is the most important part of the workflow. I add a **Code** node
named:

`Find shortages`

Settings:

-   Language: `JavaScript`,
-   Mode: `Run Once for All Items`.

I paste the code:

``` javascript
// ===== SETTINGS =====
const TEST_MODE = true;                   // true = send all emails to TEST_EMAIL
const TEST_EMAIL = 'your.address@gmail.com'; // enter your address
const BLOCK_DAYS = 3;                     // number of days before the same item can be reordered

const today = new Date();
today.setHours(0, 0, 0, 0);
const dateText = `${String(today.getDate()).padStart(2, '0')}.${String(today.getMonth() + 1).padStart(2, '0')}.${today.getFullYear()}`;

const esc = s => String(s ?? '').replace(/&/g, '&amp;').replace(/</g, '&lt;').replace(/>/g, '&gt;');

function numberValue(v) {
  const n = Number(String(v ?? '').replace(/\s/g, '').replace(',', '.'));
  return Number.isNaN(n) ? 0 : n;
}

function parseDate(s) {
  if (!s) return null;
  s = String(s).trim();
  let m = s.match(/^(\d{4})-(\d{2})-(\d{2})/);
  if (m) return new Date(+m[1], +m[2] - 1, +m[3]);
  m = s.match(/^(\d{1,2})\.(\d{1,2})\.(\d{4})/);
  if (m) return new Date(+m[3], +m[2] - 1, +m[1]);
  return null;
}

// ===== 1. Find items to reorder and group them by supplier =====
const suppliers = {};

for (const item of $input.all()) {
  const r = item.json;
  const sku = String(r['SKU'] || '').trim();
  if (!sku) continue;

  const stock = numberValue(r['Stock']);
  const min = numberValue(r['Min']);
  const target = numberValue(r['Target']);

  if (stock > min) continue;                // stock is sufficient

  const lastOrder = parseDate(r['LastOrderDate']);
  if (lastOrder) {
    const days = Math.floor((today - lastOrder) / 86400000);
    if (days < BLOCK_DAYS) continue;        // ordered recently
  }

  const email = String(r['Email'] || '').trim();
  if (!email) continue;                     // no address to send to

  const name = String(r['Supplier'] || 'No supplier').trim();
  const key = name + '|' + email;
  if (!suppliers[key]) suppliers[key] = { supplier: name, email, items: [] };

  suppliers[key].items.push({
    sku,
    product: r['Product'],
    stock,
    min,
    quantity: Math.max(target - stock, 1)
  });
}

// ===== 2. Build one email for each supplier =====
const cell = 'padding:6px 10px;border:1px solid #ddd';
const result = [];

for (const s of Object.values(suppliers)) {
  const rows = s.items.map(p =>
    `<tr>
      <td style="${cell}">${esc(p.sku)}</td>
      <td style="${cell}">${esc(p.product)}</td>
      <td style="${cell};text-align:right">${p.stock}</td>
      <td style="${cell};text-align:right"><b>${p.quantity}</b></td>
    </tr>`
  ).join('');

  const html = `
<p>Hello,</p>
<p>Please process the following replenishment order dated ${dateText}:</p>
<table style="border-collapse:collapse;font-family:Arial,sans-serif;font-size:14px">
  <thead>
    <tr style="background:#f2f2f2">
      <th style="${cell};text-align:left">SKU</th>
      <th style="${cell};text-align:left">Product</th>
      <th style="${cell}">Current stock</th>
      <th style="${cell}">Order quantity</th>
    </tr>
  </thead>
  <tbody>${rows}</tbody>
</table>
<p>Please confirm receipt of the order and provide the expected delivery date.</p>
<p>Kind regards,<br>Purchasing Department</p>`;

  result.push({
    json: {
      supplier: s.supplier,
      supplierEmail: s.email,
      recipientEmail: TEST_MODE ? TEST_EMAIL : s.email,
      subject: `${TEST_MODE ? '[TEST] ' : ''}Replenishment order ${dateText} - ${s.supplier}`,
      itemCount: s.items.length,
      html,
      items: s.items
    }
  });
}

return result;
```

I replace `your.address@gmail.com` with my own test email address.

### How the code works

1.  It retrieves all rows from the previous node.
2.  It skips rows without an SKU.
3.  It reads the stock, minimum and target levels.
4.  It skips a product if `Stock` is greater than `Min`.
5.  It checks the last order date. If fewer than 3 days have passed, the
    item is blocked.
6.  It skips items without an email address.
7.  It groups the remaining items by supplier name and email address.
8.  It builds one HTML email for each supplier.
9.  In test mode, it replaces the recipient address with my test email
    address.

Thanks to grouping, four items from one supplier do not generate four
separate messages.

**Test step:** 3 items should be returned: Alfa Packaging, Beta Pack and
Gamma Print. Each item contains, among other fields, `recipientEmail`,
`subject`, `html` and an `items` array.

If nothing needs to be reordered, Code returns an empty list and the
remaining workflow receives no items.

## Step 4 -- Gmail: send an email to the supplier

I add a **Gmail** node and name it:

`Send supplier email`

Settings:

-   Resource: `Message`,
-   Operation: `Send`,
-   To (Expression): `{{ $json.recipientEmail }}`,
-   Subject (Expression): `{{ $json.subject }}`,
-   Email Type: `HTML`,
-   Message (Expression): `{{ $json.html }}`,
-   Options → Add Option: I disable `Append n8n Attribution`.

Optionally, I can add my own address or the purchasing department's
address in CC.

Connection: `Find shortages → Send supplier email`.

The node runs separately for each item, which means once for each
supplier.

**Test step:** 3 emails with tables and subjects starting with `[TEST]`
should appear in the test inbox.

## Step 5 -- Code: split orders into individual items

After sending the emails, I have data grouped by supplier. To update the
warehouse sheet and the order log, I need a separate item for each SKU.

I add a **Code** node named:

`Split into items`

Setting:

-   Mode: `Run Once for All Items`.

Code:

``` javascript
const result = [];

for (const supplier of $('Find shortages').all()) {
  const s = supplier.json;

  for (const p of s.items) {
    result.push({
      json: {
        date: $now.toFormat('yyyy-MM-dd'),
        sku: p.sku,
        product: p.product,
        quantity: p.quantity,
        supplier: s.supplier,
        email: s.supplierEmail,
        status: 'Sent'
      }
    });
  }
}

return result;
```

I retrieve the data directly from the `Find shortages` node by its name.
After Gmail, the current item contains the sent message data rather than
the full list of order items.

This node runs after Gmail. As a result, the spreadsheet update and
order log steps happen only after the email sending step has completed.

**Test step:** 4 items should be returned:

-   KAR-001,
-   TAS-010,
-   FOL-005,
-   ETY-100.

Each item has separate fields: `date`, `sku`, `product`, `quantity`,
`supplier`, `email` and `status`.

## Step 6 -- Google Sheets: mark ordered items in the warehouse sheet

I create two parallel connections from the `Split into items` node. The
first one updates the warehouse sheet.

I add a **Google Sheets** node named:

`Mark in warehouse`

Settings:

-   Operation: `Update Row`,
-   Document: `Warehouse - stock`,
-   Sheet: `Warehouse`,
-   Mapping Column Mode: `Map Each Column Manually`,
-   Column to match on: `SKU`.

In **Values to Update**, I set:

-   `SKU` (using to match): `{{ $json.sku }}`,
-   `LastOrderDate`: `{{ $json.date }}`.

The `SKU` field marked "using to match" must be populated with an
expression. Selecting the matching column does not automatically mean
that n8n will retrieve the correct value from the previous node.

Connection:

`Split into items → Mark in warehouse`

## Step 7 -- Google Sheets: order log

The second connection from the same output of `Split into items` goes to
another **Google Sheets** node.

Name:

`Order log`

Settings:

-   Operation: `Append Row`,
-   Document: `Warehouse - stock`,
-   Sheet: `Orders`.

Column mapping:

  Sheet column   Value (Expression)
  -------------- ------------------------
  `Date`         `{{ $json.date }}`
  `SKU`          `{{ $json.sku }}`
  `Product`      `{{ $json.product }}`
  `Quantity`     `{{ $json.quantity }}`
  `Supplier`     `{{ $json.supplier }}`
  `Status`       `{{ $json.status }}`

Connection:

`Split into items → Order log`

This is a parallel branch: the same four items go both to the warehouse
update and to the order log.

------------------------------------------------------------------------

## Step 8 -- Workflow tests

### Test 1 -- first run

I leave `TEST_MODE = true` and run the workflow using **Execute
workflow**.

I check:

-   whether I received 3 emails with tables,
-   whether 4 rows appeared in the `Orders` tab,
-   whether dates were saved in the `LastOrderDate` column for KAR-001,
    TAS-010, FOL-005 and ETY-100.

### Test 2 -- duplicate prevention

I run the workflow again immediately after the first test.

Expected result:

-   no email is sent,
-   `Find shortages` returns an empty list,
-   the workflow finishes without an error.

After the first run, the order dates are already saved in the sheet, so
the items should not pass through the restriction again.

### Test 3 -- retry after the blocking period

In the sheet, I move the date for one item back by 5 days, for example
to `2026-10-15`.

I run the workflow again.

Expected result: only that item is included in the order because the
3-day blocking period has passed.

### Test 4 -- missing email address

I remove the email address for one supplier and run the workflow.

Expected result: that supplier's items are skipped because the workflow
has no address to send the message to.

### Cleanup after testing

After completing the tests:

1.  I clear the test dates from the `LastOrderDate` column.
2.  I clear the test rows from the `Orders` tab.
3.  I check that `TEST_EMAIL` contains my own address.
4.  Only after confirming that the configuration is correct do I
    consider changing `TEST_MODE` to `false`.

I do not disable test mode until I am certain that the sheet contains
the correct supplier addresses and the workflow has been tested with
fictional data.

------------------------------------------------------------------------

## Full workflow diagram

``` text
Schedule Trigger
(Cron: 0 7 * * 1-5, Europe/Warsaw)
        │
        ▼
Google Sheets: Read warehouse
(Get Row(s), all rows)
        │
        ▼
Code: Find shortages
(filtering + 3-day restriction
+ grouping by supplier
+ building HTML email)
        │
        ▼
Gmail: Send supplier email
(Send, HTML, test recipient)
        │
        ▼
Code: Split into items
(one item per SKU)
        │
        ├──────────────► Google Sheets: Mark in warehouse
        │                (Update Row, match by SKU)
        │
        └──────────────► Google Sheets: Order log
                         (Append Row)
```

------------------------------------------------------------------------

## Pitfalls, limitations and possible improvements

### 1. Data source

In this version, Google Sheets is a manually maintained source of
warehouse stock levels. In a real implementation, the data could come
from a warehouse management system or a recurring CSV export. The rest
of the workflow could remain unchanged as long as the input data keeps
the same structure.

### 2. Missing supplier email address

Items without an email address are skipped by the code. In a production
version, it would be useful to add a separate alert indicating that a
product needs to be reordered but has no supplier contact assigned.

### 3. Partial failure during sending

If Gmail sends an email to the first supplier but an error occurs while
sending to the second, the workflow may stop before the spreadsheet is
updated. In that case, the first supplier's order could be sent again
during the next run.

This is a deliberate simplification in this version. A more resilient
version should record the sending status separately for each supplier
and update items after confirming that the specific email was sent.

### 4. Automatic order versus human approval

The current version sends messages automatically. When working with real
suppliers, a safer solution may be to create email drafts (Draft) that
an employee reviews and approves before sending.

### 5. Company data

For tests and portfolio materials, I use only fictional products,
suppliers, stock levels and email addresses. Real employer data should
not appear in public screenshots or documentation.

------------------------------------------------------------------------

## What we covered in this workflow

-   configuring Schedule Trigger with a custom cron expression and
    setting the workflow time zone,
-   retrieving multiple records from Google Sheets and processing them
    in a single Code node,
-   using `Run Once for All Items` to work with the entire data set,
-   filtering items based on several business conditions,
-   parsing dates in different formats and using them to prevent repeat
    orders,
-   grouping multiple products by supplier so that one consolidated
    message is sent instead of many individual emails,
-   generating HTML email messages with tables based on spreadsheet
    data,
-   using test mode to route all messages to my own address instead of
    real recipients,
-   referencing data from an earlier node by its name when the current
    node no longer contains the required information,
-   splitting grouped data into individual items so that each product
    can be updated separately,
-   creating a parallel branch in which one data stream feeds two
    independent actions,
-   updating an existing Google Sheets row by matching the SKU and
    appending entries to an order log at the same time,
-   designing tests for normal operation, duplicate prevention, retrying
    after the blocking period and missing supplier email addresses,
-   identifying prototype limitations, especially the risk of partial
    failure when sending emails to multiple suppliers.
