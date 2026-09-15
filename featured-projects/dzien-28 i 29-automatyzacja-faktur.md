# N8N – Dzień 28: Automatyzacja obsługi faktur

## Projekt: Automatyczny rejestr i weryfikacja faktur

### Dlaczego akurat to

W firmie faktury pojawiają się cały czas, a ich obsługa zwykle wygląda podobnie: ktoś dostaje dokument, sprawdza dane, przepisuje informacje do arkusza, przekazuje fakturę dalej, informuje odpowiednią osobę, pilnuje, żeby niczego nie pominąć. Celem jest workflow przejmujący część tej powtarzalnej pracy.

Nie chodzi jeszcze o pełny system księgowy — celem jest **prototyp automatyzacji obiegu faktur**, pokazujący umiejętność łączenia kilku systemów i projektowania procesu biznesowego, a nie tylko demonstrację znajomości pojedynczych nodów.

Na początek workflow nie analizuje jeszcze prawdziwych plików PDF — formularz symuluje dane faktury, żeby skupić się na projektowaniu samego procesu, a nie na walce z OCR czy zewnętrznym API.

### Cel lekcji

- projektowanie workflowa pod konkretny proces biznesowy,
- walidacja danych wejściowych,
- obsługa błędnych danych zamiast ich ignorowania,
- praca z liczbami i rozłącznymi zakresami,
- projektowanie warunków w Switch tak, by się wzajemnie wykluczały,
- praca z datami,
- automatyczne powiadamianie człowieka tylko wtedy, gdy jest to rzeczywiście potrzebne.

---

## Krok 1 – Nowy workflow

Nazwa: `Automatyzacja faktur — DEMO`

## Krok 2 – Formularz

Trigger: **On Form Submission**

Tytuł: `Dodaj fakturę`

Pola:

| Pole | Typ |
|---|---|
| Numer faktury | Text |
| Kontrahent | Text |
| NIP | Text |
| Email | Email |
| Data wystawienia | Date |
| Termin płatności | Date |
| Kwota netto | Number |
| VAT | Number |
| Kwota brutto | Number |
| Opis | Text |

## Krok 3 – Edit Fields: bufor danych

Node **Edit Fields**, `Include Other Input Fields` włączone, z dodatkowymi polami pomocniczymi:

- `Status` → `NOWA` (wartość stała)
- `Data dodania` → `{{ $now }}`
- `Kwota brutto` — wpisywana ręcznie w formularzu, nie wymaga tu dodatkowego liczenia

Edit Fields pełni tu przede wszystkim rolę bufora danych — dalsze nody mają pracować na uporządkowanym zestawie pól, zgodnie z wzorcem wypracowanym w poprzednich dniach.

## Krok 4 – Walidacja faktury

Node **IF**, trzy warunki połączone przez `AND`:

- `{{ $json["Numer faktury"] }}` `is not empty`
- `{{ $json["Kontrahent"] }}` `is not empty`
- `{{ $json["Kwota brutto"] }}` `is not empty`

Faktura uznawana jest za poprawną tylko wtedy, gdy wszystkie trzy wymagane informacje są obecne jednocześnie. To ważniejsza część workflowa niż samo podłączanie aplikacji — tu workflow zaczyna podejmować decyzję na podstawie jakości danych, nie tylko przekazywać je dalej.

## Krok 5 – Obsługa błędnej faktury

Z gałęzi **FALSE**: node **Google Sheets**, arkusz `Faktury — błędy`.

Kolumny: Data, Numer faktury, Kontrahent, NIP, Kwota brutto, Opis, Powód.

Powód na start: `Brak wymaganych danych` (wartość stała).

Mapowanie z Edit Fields, np. `Data` → `{{ $json["Data dodania"] }}`.

**Ważny wzorzec projektowy:** błąd danych nie powinien kończyć procesu bez śladu — błędna faktura trafia do osobnego rejestru zamiast po prostu znikać.

## Krok 6 – Rejestr poprawnych faktur

Z gałęzi **TRUE**: node **Google Sheets**, operacja `Append Row`, arkusz `Faktury — rejestr`.

Kolumny: Data dodania, Numer faktury, Kontrahent, NIP, Data wystawienia, Termin płatności, Kwota netto, VAT, Kwota brutto, Status (`NOWA`).

## Krok 7 – Klasyfikacja faktury według kwoty

Za node'em Google Sheets z Kroku 6: **Switch**, klasyfikujący według `Kwota brutto`:

- `Niska`: `< 1000`
- `Średnia`: `>= 1000` i `< 5000`
- `Wysoka`: `>= 5000` i `< 10000`
- `Bardzo wysoka`: `>= 10000`

**Uwaga wynikająca z doświadczenia z Dnia 24 (VIP/Wysoki):** zakresy muszą być rozłączne. Kolejność reguł i sposób obsługi wielu pasujących warunków (np. First matching output, ułożenie reguł od najbardziej do najmniej szczegółowej) trzeba ustawić świadomie, żeby uniknąć sytuacji, w której np. kwota `10000` trafia jednocześnie do `Wysoka` i `Bardzo wysoka`.

## Krok 8 – Powiadomienie o wysokiej fakturze

Gałąź `Wysoka` → node **Gmail**:

```
Temat: ⚠️ Faktura wysokiej wartości — {{ $json["Numer faktury"] }}

Treść:
Cześć,

w rejestrze pojawiła się faktura wymagająca dodatkowej uwagi.

Numer faktury: {{ $json["Numer faktury"] }}
Kontrahent: {{ $json["Kontrahent"] }}
Kwota brutto: {{ $json["Kwota brutto"] }} zł
Termin płatności: {{ $json["Termin płatności"] }}

Faktura została automatycznie dodana do rejestru.

Pozdrawiam,
Automatyzacja n8n
```

To pierwszy moment w tym workflowie, w którym proces zaczyna aktywnie informować człowieka o sytuacji wymagającej uwagi.

## Krok 9 – Powiadomienie o fakturze bardzo wysokiej

Gałąź `Bardzo wysoka` (`>= 10 000 zł`) → osobny, wyraźniejszy node **Gmail**:

```
Temat: 🚨 PILNE — faktura powyżej 10 000 zł

Treść:
W systemie pojawiła się faktura o wysokiej wartości.

Numer: {{ $json["Numer faktury"] }}
Kontrahent: {{ $json["Kontrahent"] }}
Kwota brutto: {{ $json["Kwota brutto"] }} zł
Termin płatności: {{ $json["Termin płatności"] }}

Dokument został zapisany w rejestrze i wymaga ręcznej weryfikacji.
```

Nie chodzi o estetykę maila — chodzi o utrwalenie wzorca: dane → decyzja → odpowiednia reakcja.

## Krok 10 – Zadanie otwarte: termin płatności

Dodanie IF sprawdzającego, czy termin płatności przypada w ciągu najbliższych 7 dni — celowo bez gotowego expression, jako ćwiczenie samodzielnej pracy z `DateTime`.

---

## Testy

**Test 1 — poprawna faktura:**
```
Numer faktury: FV/2026/08/001
Kontrahent: ABC Sp. z o.o.
NIP: 1234567890
Email: test@example.com
Data wystawienia: 2026-08-20
Termin płatności: 2026-09-03
Kwota netto: 4000
VAT: 920
Kwota brutto: 4920
Opis: Usługi IT
```
Oczekiwane: formularz działa, walidacja przechodzi TRUE, faktura trafia do rejestru, Switch wybiera zakres `Średnia`, żadna niepotrzebna gałąź się nie uruchamia.

**Test 2 — faktura wysokiej wartości:** `Kwota brutto: 7500`. Oczekiwane: TRUE → Rejestr → Switch → `Wysoka` → mail.

**Test 3 — faktura bardzo wysoka:** `Kwota brutto: 15000`. Oczekiwane: trafia wyłącznie do `Bardzo wysoka`, nie jednocześnie do `Wysoka` i `Bardzo wysoka` — sprawdzenie poprawności rozłączności zakresów z Kroku 7.

**Test 4 — błędna faktura:** usunięcie pola `Numer faktury` przed wysłaniem. Oczekiwane: `Formularz → Edit Fields → IF → FALSE → Faktury — błędy`, brak trafienia do głównego rejestru.

## Zadanie dodatkowe

Dodatkowa kontrola: termin płatności nie może być wcześniejszy niż data wystawienia (np. wystawienie 20.08 z terminem płatności 15.08 powinno być traktowane jako błąd, a wystawienie 20.08 z terminem 03.09 jako poprawne). Pierwsze miejsce w tym projekcie wymagające realnej pracy z porównywaniem dat, nie tylko tekstem i liczbami.

---

## Docelowy schemat workflowa

```
                  ┌──► BŁĘDNA → Rejestr błędów
                  │
Formularz → Edit Fields → Walidacja
                              │
                              ▼
                         POPRAWNA
                              │
                    ┌─────────┴─────────┐
                    ▼                   ▼
             Rejestr faktur          Switch
                                      │
                         ┌────────────┼────────────┐
                         ▼            ▼            ▼
                       Niska       Wysoka     Bardzo wysoka
                                      │            │
                                      ▼            ▼
                                    Gmail        Gmail
```

---

## Dlaczego ten projekt nadaje się do portfolio

Po ukończeniu efekt nie sprowadza się do stwierdzenia "umiem n8n i połączyłem kilka aplikacji", tylko do: *"Zbudowałem prototyp automatyzacji obiegu faktur. System przyjmuje dane dokumentu, waliduje je, zapisuje poprawne faktury w rejestrze, przechwytuje błędne dane i automatycznie eskaluje faktury o wysokiej wartości."* To rozwiązanie konkretnego problemu biznesowego, a nie demonstracja znajomości pojedynczych nodów.

---

## Czego się nauczyłeś

- projektowania workflowa pod konkretny proces biznesowy, a nie pod pojedynczą integrację
- walidacji danych wejściowych za pomocą wielu połączonych warunków IF
- obsługi błędnych danych przez osobny rejestr zamiast ich ignorowania lub porzucania
- pracy z liczbami i rozłącznymi zakresami w klasyfikacji (nawiązanie do wcześniejszego doświadczenia z Dnia 24)
- projektowania warunków w Switch tak, by realnie się wykluczały
- pierwszego kontaktu z koniecznością porównywania dat w logice biznesowej
- że automatyczne powiadomienie człowieka ma sens tylko wtedy, gdy sytuacja rzeczywiście tego wymaga — nie każdy wynik procesu zasługuje na alert

# N8N – Dzień 29: Automatyczny obieg faktur z firmowego maila

## Projekt: faktura przychodzi na firmowy mail → n8n ją wykrywa → odczytuje dane → sprawdza poprawność → zapisuje do rejestru → klasyfikuje → sprawdza termin płatności → wysyła odpowiednie powiadomienie

### Dlaczego przerabiam wczorajszy workflow

W Dniu 28 dane faktury wpisywane były ręcznie przez formularz. Workflow potrafił potem walidować dane, zapisać fakturę do Google Sheets, klasyfikować ją według kwoty, wykrywać zbliżający się termin płatności i wysłać odpowiedniego maila — ale pracownik nadal musiał otworzyć fakturę i przepisać z niej kilkanaście informacji. To nie jest jeszcze dobra automatyzacja biznesowa, skoro część problemu zostaje po stronie człowieka.

Zamiast wyrzucać wczorajszy workflow, rozbudowano go o automatyczne źródło danych:

```
Dzień 28:  Formularz → dane faktury
Dzień 29:  Email → załącznik → dane faktury
```

Z poprzedniego workflowa zachowano i wykorzystano: IF do walidacji, Google Sheets jako rejestr, Switch do klasyfikacji kwoty, IF do sprawdzania terminu, Gmail do powiadomień. Zmienił się przede wszystkim sposób dostarczania danych do workflowa.

### Cel lekcji

- reagowanie na przychodzące wiadomości e-mail jako źródło danych (Gmail Trigger)
- filtrowanie skrzynki tak, żeby workflow reagował tylko na właściwe wiadomości
- praca z danymi binarnymi (załącznikami), nowym rodzajem danych obok dotychczasowego JSON-a
- odczyt tekstu z pliku PDF wbudowanym node'em n8n (Extract From File)
- wyciąganie konkretnych wartości z surowego tekstu za pomocą wyrażeń regularnych w node'ie Code
- zabezpieczenie workflowa przed nieprawidłowymi lub niepasującymi załącznikami

---

## Krok 1 – Gmail Trigger jako nowy punkt wejścia

Dotychczasowy trigger formularza (On form submission) pozostawiono na razie nietknięty — nowy początek workflowa budowany był obok, a podpięcie do reszty nastąpiło dopiero po jego przetestowaniu.

Dodano node **Gmail Trigger**, mający reagować na wiadomości przychodzące na skrzynkę przeznaczoną do faktur (docelowo osobny adres, na czas nauki — zwykła skrzynka testowa).

## Krok 2 – Filtrowanie skrzynki

Panel Gmail Triggera udostępnia kilka opcji filtrowania naraz: Include Spam and Trash, Include Drafts, Label Names or IDs, Search, Read Status, Sender.

Zastosowana konfiguracja:

| Opcja | Ustawienie |
|---|---|
| Include Spam and Trash | OFF |
| Include Drafts | OFF |
| Label Names or IDs | puste |
| Search | subject:faktura |
| Read Status | szeroko na czas testów, nie zawężane do Unread na stałe |
| Sender | puste |

Kluczowe pole to Search — workflow reaguje wyłącznie na wiadomości, których temat zawiera słowo "faktura". Świadomie pozostawiono to jako prosty wariant startowy — docelowo można dodać bardziej precyzyjne filtrowanie (np. konkretny nadawca + temat + obecność załącznika).

Uwaga praktyczna: nie warto ustawiać Read Status na Unread na stałe podczas testów — przypadkowe otwarcie testowego maila oznacza go jako przeczytany, co potem myląco wygląda jak "trigger nie zadziałał".

## Krok 3 – Debugowanie: dlaczego test triggera "od razu się kończy"

Próba przetestowania Gmail Triggera przez standardowy przycisk testowy kończyła się natychmiastowym statusem "Successfully executed", bez realnych danych do podejrzenia — inaczej niż w przypadku Webhooka, który w trybie testowym rzeczywiście czeka na zdarzenie.

Wyjaśnienie: Gmail Trigger w tej wersji n8n nie działa jak webhook oczekujący na jedno zdarzenie testowe — kończy test natychmiast, niezależnie od tego, czy nowa wiadomość faktycznie nadeszła.

Zastosowane obejście:
1. Wysłanie prawdziwego testowego maila z fakturą w załączniku.
2. Włączenie workflowa jako Active.
3. Wysłanie kolejnej wiadomości z fakturą.
4. Sprawdzenie historii Executions workflowa — tam widoczne były pełne dane wejściowe z realnego triggera.

## Krok 4 – Pobranie załącznika

W używanej wersji n8n Gmail Trigger nie ma osobnego node'a "Download Attachment" — pobieranie załącznika jest wbudowaną częścią samego Gmail Triggera, bez dodatkowych opcji do skonfigurowania osobno.

## Krok 5 – Dane binarne jako nowy rodzaj danych

Do tej pory praca odbywała się głównie na strukturach JSON, np. {"Kontrahent": "ABC Sp. z o.o.", "Kwota brutto": 1230}. Załącznik pliku pojawia się w n8n jako osobny rodzaj danych — Binary — widoczny w osobnej sekcji output node'a, obok zwykłego JSON-a. Wykorzystywana w kolejnym kroku nazwa pola binarnego to attachment_0.

---

## Krok 6 – Extract From File: odczyt tekstu z PDF

### Ustalenie właściwego node'a

Zamiast zgadywać dostępność konkretnego rozwiązania z góry, sprawdzono, co faktycznie oferuje posiadana instalacja n8n. Właściwym, wbudowanym node'em okazał się Extract From File (w starszych wersjach n8n funkcjonujący pod nazwą "Read PDF").

### Dokładna konfiguracja

Node Extract From File, dodany bezpośrednio po Gmail Trigger:

- Operation: Extract Text from PDF (z listy dostępnych typów wybrany PDF)
- Input Binary Field: attachment_0

Po uruchomieniu node zwraca cały tekst faktury jako jedno, długie pole $json.text — surowy, nieprzetworzony ciąg znaków, dokładnie taki, jaki dałoby się otrzymać kopiując cały tekst z otwartego PDF-a. To pole jest punktem wyjścia dla kolejnego kroku.

---

## Krok 7 – Code node: wyciąganie konkretnych danych wyrażeniami regularnymi

Extract From File zwraca cały tekst faktury jako jeden ciąg znaków — potrzebny był dodatkowy krok rozbijający go na konkretne, nazwane pola (Numer faktury, Kontrahent, NIP itd.), zanim dane mogły trafić do reszty workflowa (Edit Fields, IF, Google Sheets).

To pierwsze poważniejsze użycie node'a Code w tym projekcie, rozwijane iteracyjnie w trzech kolejnych wersjach.

### Wersja 1 — pierwszy, uproszczony test

Sprawdzenie, czy w ogóle da się coś sensownego wyciągnąć z surowego tekstu za pomocą .match():

```javascript
const text = $json.text || "";

// Wyciąganie danych przy użyciu wyrażeń regularnych (Regex)
const nrFaktury = text.match(/nr:\s*([^\n]+)/i)?.[1]?.trim();
const nip = text.match(/NIP\s*([\d-]+)/i)?.[1]?.trim();
const kwotaBrutto = text.match(/Razem do zapłaty:\s*([\d\s,]+)\s*zł/i)?.[1]?.replace(/\s/g, '').replace(',', '.');
const kontrahent = text.match(/Nabywca\s*\n\s*([^\n]+)/i)?.[1]?.trim();

return {
  json: {
    "Numer faktury": nrFaktury || "",
    "Kontrahent": kontrahent || "",
    "NIP": nip || "",
    "Kwota brutto": kwotaBrutto ? parseFloat(kwotaBrutto) : 0
  }
};
```

### Wersja 2 — dopasowanie do rzeczywistej struktury testowej faktury

Po zobaczeniu, jak dokładnie wygląda tekst wyekstrahowany z konkretnego testowego dokumentu, wyrażenia poprawiono tak, by trafniej łapały pole "Faktura nr:" oraz dane nabywcy:

```javascript
const text = $json.text || "";

// Wyciąganie danych przy użyciu wyrażeń regularnych
const nrFaktury = text.match(/Faktura\s+nr:\s*([^\n\r]+)/i)?.[1]?.trim();
const nip = text.match(/Nabywca[\s\S]*?NIP\s*([\d-]+)/i)?.[1]?.trim();
const kontrahent = text.match(/Nabywca\s*\n\s*([^\n\r]+)/i)?.[1]?.trim();

// Wyciąganie kwoty z wiersza "Razem:"
const kwotaMatch = text.match(/Razem:\s*\n?\s*([\d\s,]+)\s*zł/i);
const kwotaBrutto = kwotaMatch ? kwotaMatch[1].replace(/\s/g, '').replace(',', '.') : null;

return {
  json: {
    "Numer faktury": nrFaktury || "",
    "Kontrahent": kontrahent || "",
    "NIP": nip || "",
    "Kwota brutto": kwotaBrutto ? parseFloat(kwotaBrutto) : ""
  }
};
```

### Wersja 3 (finalna) — wszystkie 9 pól potrzebnych do rejestru faktur

Ostateczna wersja, uzupełniona o daty oraz rozbicie kwoty na netto/VAT/brutto osobno, plus pole opisu:

```javascript
const text = $json.text || "";

// Wyrażenia regularne dopasowane do polskiego układu faktur
const nrFaktury = text.match(/Faktura\s+(?:nr|numer)?[:\s]*([^\n\r]+)/i)?.[1]?.trim();
const kontrahent = text.match(/Nabywca\s*\n\s*([^\n\r]+)/i)?.[1]?.trim();
const nip = text.match(/Nabywca[\s\S]*?NIP[:\s]*([\d-]+)/i)?.[1]?.trim();

// Daty
const dataWystawienia = text.match(/Wystawiona\s+w\s+dnius[:\s]*([\d.-]+)/i)?.[1]?.trim() ||
                        text.match(/Data\s+wystawienia[:\s]*([\d.-]+)/i)?.[1]?.trim();

const terminPlatnosci = text.match(/Termin\s+płatności[:\s]*([\d.-]+)/i)?.[1]?.trim() ||
                        text.match(/Sprzedaż\s+i\s+usługi[:\s]*([\d.-]+)/i)?.[1]?.trim();

// Kwoty
const kwotaNettoMatch = text.match(/(?:W\s+tym|Netto)[:\s]*\n?\s*([\d\s,]+)\s*zł/i);
const kwotaNetto = kwotaNettoMatch ? parseFloat(kwotaNettoMatch[1].replace(/\s/g, '').replace(',', '.')) : "";

const vatMatch = text.match(/(?:23%|VAT)[:\s]*\n?\s*([\d\s,]+)\s*zł/i);
const vat = vatMatch ? parseFloat(vatMatch[1].replace(/\s/g, '').replace(',', '.')) : "";

const kwotaBruttoMatch = text.match(/(?:Razem|Razem\s+do\s+zapłaty)[:\s]*\n?\s*([\d\s,]+)\s*zł/i);
const kwotaBrutto = kwotaBruttoMatch ? parseFloat(kwotaBruttoMatch[1].replace(/\s/g, '').replace(',', '.')) : "";

// Opis / Nazwa usługi
const opis = text.match(/(?:Nazwa\s+towaru\s+lub\s+usługi|Uwagi)[:\s]*\n?\s*([^\n\r]+)/i)?.[1]?.trim();

return {
  json: {
    "Numer faktury": nrFaktury || "",
    "Kontrahent": kontrahent || "",
    "NIP": nip || "",
    "Data wystawienia": dataWystawienia || "",
    "Termin płatności": terminPlatnosci || "",
    "Kwota netto": kwotaNetto,
    "VAT": vat,
    "Kwota brutto": kwotaBrutto,
    "Opis": opis || "Zakup usług/towarów"
  }
};
```

Jak dokładnie działają te wyrażenia — kluczowe elementy:

- text.match(/wzorzec/i) — przeszukuje cały tekst faktury pod kątem podanego wzorca, flaga i oznacza ignorowanie wielkości liter.
- ?.[1] — bezpieczne pobranie pierwszej grupy przechwytującej (fragmentu w nawiasach () wzorca) — jeśli match() nic nie znajdzie i zwróci null, operator ?. zapobiega błędowi zamiast wywalić cały Code node.
- ?.trim() — usunięcie białych znaków z początku/końca dopasowanego fragmentu.
- .replace(/\s/g, '') — usunięcie spacji z liczb (faktury często zapisują kwoty jako 1 230,00).
- .replace(',', '.') — zamiana polskiego separatora dziesiętnego (przecinek) na kropkę, wymaganą przez parseFloat().
- operator || między dwoma .match() — próba dwóch alternatywnych wzorców pod rząd, na wypadek gdyby faktura używała innego sformułowania tego samego pola.

Po wykonaniu tego node'a wynikiem jest kompletny obiekt JSON z osobnymi, gotowymi polami — po podłączeniu do kolejnego node'a widać je w panelu Input jako pojedyncze wartości, gotowe do przeciągnięcia w Edit Fields czy IF, dokładnie tak jak wcześniej dane z formularza w Dniu 28.

### Odczyt JPG okazał się zbyt trudny bez AI

Pierwotnie planowano obsłużyć również obrazy (JPG), ale odczyt danych ze zdjęcia faktury bez wsparcia modelu AI (np. OCR wymagającego dodatkowego API) okazał się zbyt trudny na tym etapie — Extract From File obsługuje tekst PDF, nie potrafi jednak "przeczytać" tekstu z obrazu rastrowego. W praktyce przyjęto rozwiązanie pośrednie: faktury konwertowane są do PDF przed wysłaniem mailem, co pozwala korzystać wyłącznie z Extract From File bez dodatkowych narzędzi OCR.

---

## Krok 8 – Reszta workflowa: bez zmian względem Dnia 28

Po Code node dane trafiają w dokładnie ten sam sposób, co dane z formularza w Dniu 28: Edit Fields (dodanie Status = NOWA, Data dodania = {{ $now }}, z Include Other Input Fields włączonym) → IF (walidacja obecności Numeru faktury, Kontrahenta, Kwoty brutto) → gałąź błędna do arkusza Faktury — błędy / gałąź poprawna do arkusza Faktury — rejestr → Switch klasyfikujący według Kwota brutto → IF sprawdzający zbliżający się termin płatności → Gmail z powiadomieniem.

Klasyfikacja kwoty (Switch) — utrzymana zasada rozłączności zakresów z Dnia 24/28: Bardzo wysoka >= 10000, Wysoka >= 5000 i < 10000, Średnia >= 1000 i < 5000, Niska < 1000, z regułami ułożonymi od najbardziej do najmniej szczegółowej i włączonym "First matching output" — w tej wersji Switcha wciąż nie ma operatora AND w pojedynczej regule.

---

## Krok 9 – Test na żywo po publikacji: błąd "Invalid PDF structure"

Po opublikowaniu workflowa i przetestowaniu go na realnym ruchu pocztowym pojawił się błąd na node'ie Extract From File:

Invalid PDF structure.
Node type: n8n-nodes-base.extractFromFile

Diagnoza: błąd tego typu pojawia się zwykle z dwóch powodów: wiadomość nie zawierała w ogóle załącznika PDF (np. obrazek w podpisie maila, logo, plik innego typu), albo załączników było kilka, a node dostał do przetworzenia plik, który nie jest dokumentem PDF.

Planowana naprawa:

1. Filtrowanie po typie pliku przed Extract From File. Dodanie node'a IF/Filter między Gmail Trigger a Extract From File, sprawdzającego, czy nazwa załącznika kończy się na .pdf:
   - Left value: {{ $json.fileName }} (lub {{ $binary.attachment_0.fileName }})
   - Operator: Ends With
   - Right value: .pdf
   - Tylko gałąź spełniająca warunek prowadzi dalej do Extract From File.

2. Zabezpieczenie ustawień samego Extract From File. W zakładce Settings node'a zmiana opcji On Error z domyślnego Stop Workflow na Continue (using error output) lub Continue (regular output) — dzięki temu uszkodzony lub nietypowy załącznik nie zatrzymuje całego procesu.

---

## Pełny schemat workflowa

```
Gmail Trigger (Search: subject:faktura)
        │
        ▼
Extract From File
(Operation: Extract Text from PDF,
 Input Binary Field: attachment_0)
        │
        ▼
Code — regex (Numer faktury, Kontrahent,
NIP, daty, Kwota netto/VAT/brutto, Opis)
        │
        ▼
Edit Fields (Status=NOWA, Data dodania=$now)
        │
        ▼
       IF — walidacja
      /            \
  BŁĘDNA         POPRAWNA
     │               │
     ▼               ▼
Faktury - błędy   Faktury - rejestr
                       │
                       ▼
                    Switch
        (Kwota brutto, First matching output)
   ┌────────┬─────────┬──────────────┐
   ▼        ▼         ▼              ▼
 Niska   Średnia    Wysoka     Bardzo wysoka
                       │              │
                       └──────┬───────┘
                              ▼
                       IF — termin płatności
                       (najbliższe 7 dni?)
                              │
                              ▼
                        Gmail — powiadomienie
```

---

## Testy

Test 1 — zwykła faktura: Kwota brutto 750 zł. Sprawdzane: wykrycie z maila, poprawny odczyt danych, zapis do rejestru, trafienie do kategorii Niska.

Test 2 — wysoka faktura: Kwota brutto 7500 zł. Sprawdzane: rejestr, Switch → Wysoka, odpowiednie powiadomienie.

Test 3 — bardzo wysoka faktura: Kwota brutto 15000 zł. Sprawdzane: rejestr, Switch → Bardzo wysoka (wyłącznie, nie jednocześnie z Wysoka), odpowiednie powiadomienie.

Test 4 — błędny dokument: dokument, którego nie da się poprawnie odczytać, albo brakuje w nim wymaganych danych. Sprawdzane: faktura nie trafia do głównego rejestru, trafia do Faktury - błędy, pojawia się informacja o powodzie błędu.

---

## Czego się nauczyłeś

- że Gmail Trigger nie zachowuje się jak Webhook przy testowaniu — nie czeka na pojedyncze zdarzenie testowe, tylko kończy test natychmiast; realne dane sprawdza się przez aktywację workflowa i podgląd Executions
- filtrowania skrzynki pocztowej (Search, Read Status i pozostałe opcje Gmail Triggera) tak, by trigger reagował tylko na właściwe wiadomości
- że dane plikowe (Binary) to osobny rodzaj danych w n8n, obok dotychczas używanego JSON-a, i że pole binarne z Gmaila nazywa się attachment_0
- odczytu tekstu z PDF wbudowanym node'em Extract From File (dawniej Read PDF), łącznie z dokładną konfiguracją Operation i Input Binary Field
- pisania i iteracyjnego dopracowywania wyrażeń regularnych w node'ie Code do wyciągania konkretnych, nazwanych pól z surowego tekstu — w tym praktycznego użycia optional chaining (?.), .trim(), usuwania spacji z liczb i zamiany przecinka na kropkę dziesiętną
- że warto rozwijać taki Code node iteracyjnie (najpierw prosty test na kilku polach, potem dopasowanie do rzeczywistej struktury dokumentu, na końcu komplet potrzebnych pól), zamiast pisać od razu finalną, złożoną wersję
- że odczyt danych z obrazu (JPG) bez wsparcia AI jest znacząco trudniejszy niż z PDF, i że dobór formatu wejściowego (konwersja do PDF) bywa praktyczniejszym rozwiązaniem niż budowanie OCR od zera
- że rozbudowa istniejącej automatyzacji o automatyczne źródło danych (mail zamiast formularza) to jakościowo inna zmiana niż dodanie kolejnej integracji — to przesunięcie punktu, w którym kończy się praca człowieka, a zaczyna praca systemu
- zabezpieczania workflowa produkcyjnego przed nietypowymi/uszkodzonymi załącznikami przez filtrowanie typu pliku i ustawienie obsługi błędów na Continue zamiast Stop Workflow
