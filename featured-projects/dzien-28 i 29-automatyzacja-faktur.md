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

 # N8N – Day 28: Invoice Processing Automation

## Project: Automatic register and invoice verification

### Why specifically this

In a company, invoices appear all the time, and handling them usually looks similar: someone receives a document, checks the data, transcribes information into a spreadsheet, passes the invoice along, informs the relevant person, and ensures nothing is missed. The goal is a workflow that takes over part of this repetitive work.

This is not yet a full accounting system — the goal is a **prototype of invoice workflow automation**, demonstrating the ability to connect multiple systems and design a business process, rather than just showcasing knowledge of individual nodes.

To start, the workflow does not yet analyze real PDF files — a form simulates the invoice data to focus on designing the process itself rather than fighting with OCR or external APIs.

### Lesson Goal

- designing a workflow for a specific business process,
- validating input data,
- handling erroneous data instead of ignoring it,
- working with numbers and disjoint ranges,
- designing conditions in Switch so they are mutually exclusive,
- working with dates,
- automatically notifying a human only when it is actually necessary.

---

## Step 1 – New workflow

Name: Invoice Automation — DEMO

## Step 2 – Form

Trigger: On Form Submission

Title: Add invoice

Fields:

| Field | Type |
|---|---|
| Invoice number | Text |
| Contractor | Text |
| TAX ID / NIP | Text |
| Email | Email |
| Issue date | Date |
| Due date | Date |
| Net amount | Number |
| VAT | Number |
| Gross amount | Number |
| Description | Text |

## Step 3 – Edit Fields: data buffer

Node Edit Fields, Include Other Input Fields enabled, with additional helper fields:

- Status -> NEW (static value)
- Date added -> {{ $now }}
- Gross amount — entered manually in the form, does not require additional calculation here

Edit Fields primarily serves as a data buffer here — subsequent nodes are meant to work on an organized set of fields, following the pattern developed in previous days.

## Step 4 – Invoice validation

Node IF, three conditions combined with AND:

- {{ $json["Invoice number"] }} is not empty
- {{ $json["Contractor"] }} is not empty
- {{ $json["Gross amount"] }} is not empty

An invoice is considered valid only when all three required pieces of information are present simultaneously. This is a more critical part of the workflow than merely connecting applications — here, the workflow begins making decisions based on data quality, not just passing data forward.

## Step 5 – Handling invalid invoices

From the FALSE branch: node Google Sheets, spreadsheet Invoices — errors.

Columns: Date, Invoice number, Contractor, TAX ID, Gross amount, Description, Reason.

Initial reason: Missing required data (static value).

Mapping from Edit Fields, e.g., Date -> {{ $json["Date added"] }}.

**Important design pattern:** a data error should not terminate the process without a trace — an invalid invoice lands in a separate register instead of simply disappearing.

## Step 6 – Register of valid invoices

From the TRUE branch: node Google Sheets, Append Row operation, spreadsheet Invoices — register.

Columns: Date added, Invoice number, Contractor, TAX ID, Issue date, Due date, Net amount, VAT, Gross amount, Status (NEW).

## Step 7 – Invoice classification by amount

After the Google Sheets node from Step 6: Switch, classifying by Gross amount:

- Low: < 1000
- Medium: >= 1000 and < 5000
- High: >= 5000 and < 10000
- Very high: >= 10000

**Note resulting from Day 24 experience (VIP/High):** ranges must be disjoint. The order of rules and the handling of multiple matching conditions (e.g., First matching output, arranging rules from most to least specific) must be configured consciously to avoid situations where an amount like 10000 falls into both High and Very high simultaneously.

## Step 8 – High-value invoice notification

High branch -> node Gmail:

```
Subject: ⚠️ High-value invoice — {{ $json["Invoice number"] }}

Body:
Hi,

an invoice requiring extra attention has appeared in the register.

Invoice number: {{ $json["Invoice number"] }}
Contractor: {{ $json["Contractor"] }}
Gross amount: {{ $json["Gross amount"] }} USD
Due date: {{ $json["Due date"] }}

The invoice has been automatically added to the register.

Best regards,
n8n Automation
```

This is the first moment in this workflow where the process actively informs a human about a situation requiring attention.

## Step 9 – Very high-value invoice notification

Very high branch (>= 10,000 USD) -> separate, more distinct node Gmail:

```
Subject: 🚨 URGENT — invoice over 10,000 USD

Body:
A high-value invoice has appeared in the system.

Number: {{ $json["Invoice number"] }}
Contractor: {{ $json["Contractor"] }}
Gross amount: {{ $json["Gross amount"] }} USD
Due date: {{ $json["Due date"] }}

The document has been saved to the register and requires manual verification.
```

It's not about email aesthetics — it's about solidifying the pattern: data -> decision -> appropriate reaction.

## Step 10 – Open task: due date

Adding an IF checking if the due date falls within the next 7 days — intentionally left without a ready expression as an exercise in working independently with DateTime.

---

## Tests

**Test 1 — valid invoice:**
```text
Invoice number: INV/2026/08/001
Contractor: ABC Corp
TAX ID: 1234567890
Email: test@example.com
Issue date: 2026-08-20
Due date: 2026-09-03
Net amount: 4000
VAT: 920
Gross amount: 4920
Description: IT Services
```
Expected: form works, validation passes TRUE, invoice lands in the register, Switch selects Medium range, no unnecessary branches execute.

**Test 2 — high-value invoice:** Gross amount: 7500. Expected: TRUE -> Register -> Switch -> High -> email.

**Test 3 — very high-value invoice:** Gross amount: 15000. Expected: lands exclusively in Very high, not simultaneously in High and Very high — verifying range disjointness from Step 7.

**Test 4 — invalid invoice:** removing the Invoice number field before submitting. Expected: Form -> Edit Fields -> IF -> FALSE -> Invoices — errors, no entry in the main register.

## Additional task

Extra control: due date cannot be earlier than issue date (e.g., issue on Aug 20 with due date on Aug 15 should be treated as an error, while issue on Aug 20 with due date on Sep 03 as valid). The first place in this project requiring real work with date comparison, not just text and numbers.

---

## Target workflow diagram

```
                  ┌──► INVALID → Errors register
                  │
Form → Edit Fields → Validation
                        │
                        ▼
                      VALID
                        │
              ┌─────────┴─────────┐
              ▼                   ▼
       Invoices register       Switch
                                │
                   ┌────────────┼────────────┐
                   ▼            ▼            ▼
                  Low          High      Very high
                                │            │
                                ▼            ▼
                              Gmail        Gmail
```

---

## Why this project fits a portfolio

Upon completion, the result is not reduced to saying "I know n8n and connected a few apps," but rather: *"I built a prototype for invoice processing automation. The system ingests document data, validates it, logs valid invoices into a register, captures invalid data, and automatically escalates high-value invoices."* This solves a concrete business problem rather than demonstrating knowledge of individual nodes.

---

## What I learned

- designing a workflow around a specific business process rather than a single integration
- validating input data using multiple combined IF conditions
- handling invalid data through a dedicated register instead of ignoring or dropping it
- working with numbers and disjoint ranges in classification (referencing earlier experience from Day 24)
- designing conditions in Switch so they are mutually exclusive
- first exposure to comparing dates within business logic
- that automatically notifying a human only makes sense when the situation actually requires it — not every process result deserves an alert

# N8N – Day 29: Automatic Invoice Workflow from Company Email

## Project: Invoice arrives in company email -> n8n detects it -> reads data -> validates -> saves to register -> classifies -> checks due date -> sends appropriate notification

### Why refactoring yesterday's workflow

On Day 28, invoice data was entered manually via a form. The workflow could then validate data, save the invoice to Google Sheets, classify it by amount, detect approaching due dates, and send the corresponding email — but an employee still had to open the invoice and transcribe over a dozen details. This is not yet good business automation if part of the problem remains on the human side.

Instead of discarding yesterday's workflow, it was expanded with an automated data source:

```
Day 28: Form -> invoice data
Day 29: Email -> attachment -> invoice data
```

From the previous workflow, the following were kept and reused: IF for validation, Google Sheets as register, Switch for amount classification, IF for due date checking, Gmail for notifications. What primarily changed was how data is delivered to the workflow.

### Lesson Goal

- reacting to incoming emails as a data source (Gmail Trigger)
- filtering the inbox so the workflow reacts only to relevant messages
- working with binary data (attachments), a new data type alongside JSON
- reading text from a PDF file using n8n's built-in node (Extract From File)
- extracting specific values from raw text using regular expressions in a Code node
- securing the workflow against invalid or non-matching attachments

---

## Step 1 – Gmail Trigger as a new entry point

The existing form trigger (On form submission) was left untouched for now — the new start of the workflow was built alongside it, and connection to the rest occurred only after testing it.

Added a Gmail Trigger node configured to react to messages arriving at an inbox dedicated to invoices (eventually a separate address, for learning purposes — a regular test inbox).

## Step 2 – Inbox filtering

The Gmail Trigger panel provides several filtering options at once: Include Spam and Trash, Include Drafts, Label Names or IDs, Search, Read Status, Sender.

Applied configuration:

| Option | Setting |
|---|---|
| Include Spam and Trash | OFF |
| Include Drafts | OFF |
| Label Names or IDs | empty |
| Search | subject:invoice |
| Read Status | broad during testing, not restricted to Unread permanently |
| Sender | empty |

The key field is Search — the workflow reacts exclusively to messages whose subject contains the word "invoice". This was intentionally kept as a simple starting point — eventually, more precise filtering can be added (e.g., specific sender + subject + presence of attachment).

Practical tip: avoid setting Read Status to Unread permanently during testing — accidentally opening a test email marks it as read, which later confusingly looks like "the trigger didn't fire."

## Step 3 – Debugging: why the trigger test "ends immediately"

Attempting to test Gmail Trigger via the standard test button resulted in an immediate "Successfully executed" status, without real data to inspect — unlike Webhook, which actually waits for an event during test mode.

Explanation: Gmail Trigger in this version of n8n does not act like a webhook waiting for a single test event — it ends the test immediately, regardless of whether a new message actually arrived.

Applied workaround:
1. Send a real test email with an invoice attachment.
2. Toggle the workflow to Active.
3. Send another message with an invoice.
4. Check the workflow's Executions history — complete input data from the real trigger was visible there.

## Step 4 – Downloading attachments

In the n8n version used, Gmail Trigger does not have a separate "Download Attachment" node — downloading attachments is a built-in part of the Gmail Trigger itself, with no additional options to configure separately.

## Step 5 – Binary data as a new data type

So far, work revolved mainly around JSON structures, e.g., {"Contractor": "ABC Corp", "Gross amount": 1230}. A file attachment appears in n8n as a separate data type — Binary — visible in a separate output section of the node alongside regular JSON. The binary field name used in the next step is attachment_0.

---

## Step 6 – Extract From File: reading text from PDF

### Identifying the correct node

Instead of guessing the availability of a specific solution beforehand, the actual features offered by the n8n installation were checked. The correct built-in node turned out to be Extract From File (functioning as "Read PDF" in older n8n versions).

### Exact configuration

Node Extract From File, added directly after Gmail Trigger:

- Operation: Extract Text from PDF (PDF selected from available types)
- Input Binary Field: attachment_0

Upon execution, the node returns the entire text of the invoice as a single long field $json.text — a raw, unprocessed text string, exactly what you would get by copying all text from an open PDF. This field serves as the starting point for the next step.

---

## Step 7 – Code node: extracting specific data with regular expressions

Extract From File returns the entire invoice text as a single string — an additional step was needed to split it into specific, named fields (Invoice number, Contractor, TAX ID, etc.) before data could move to the rest of the workflow (Edit Fields, IF, Google Sheets).

This is the first major use of the Code node in this project, developed iteratively across three versions.

### Version 1 — initial simplified test

Checking if anything meaningful can be extracted from raw text using .match():

```javascript
const text = $json.text || "";

// Extracting data using regular expressions (Regex)
const nrFaktury = text.match(/nr:\s*([^\n]+)/i)?.[1]?.trim();
const nip = text.match(/NIP\s*([\d-]+)/i)?.[1]?.trim();
const kwotaBrutto = text.match(/Razem do zapłaty:\s*([\d\s,]+)\s*zł/i)?.[1]?.replace(/\s/g, '').replace(',', '.');
const kontrahent = text.match(/Nabywca\s*\n\s*([^\n]+)/i)?.[1]?.trim();

return {
  json: {
    "Invoice number": nrFaktury || "",
    "Contractor": kontrahent || "",
    "TAX ID": nip || "",
    "Gross amount": kwotaBrutto ? parseFloat(kwotaBrutto) : 0
  }
};
```

### Version 2 — matching actual test invoice structure

After inspecting how text extracted from a specific test document looked, expressions were refined to better capture the "Invoice no:" field and buyer data:

```javascript
const text = $json.text || "";

// Extracting data using regular expressions
const nrFaktury = text.match(/Invoice\s+no:\s*([^\n\r]+)/i)?.[1]?.trim();
const nip = text.match(/Buyer[\s\S]*?TAX ID\s*([\d-]+)/i)?.[1]?.trim();
const kontrahent = text.match(/Buyer\s*\n\s*([^\n\r]+)/i)?.[1]?.trim();

// Extracting amount from "Total:" row
const kwotaMatch = text.match(/Total:\s*\n?\s*([\d\s,]+)\s*USD/i);
const kwotaBrutto = kwotaMatch ? kwotaMatch[1].replace(/\s/g, '').replace(',', '.') : null;

return {
  json: {
    "Invoice number": nrFaktury || "",
    "Contractor": kontrahent || "",
    "TAX ID": nip || "",
    "Gross amount": kwotaBrutto ? parseFloat(kwotaBrutto) : ""
  }
};
```

### Version 3 (final) — all 9 fields needed for invoice register

Final version, supplemented with dates and splitting amount into net/VAT/gross separately, plus description field:

```javascript
const text = $json.text || "";

// Regular expressions matched to invoice layouts
const nrFaktury = text.match(/Invoice\s+(?:no|number)?[:\s]*([^\n\r]+)/i)?.[1]?.trim();
const kontrahent = text.match(/Buyer\s*\n\s*([^\n\r]+)/i)?.[1]?.trim();
const nip = text.match(/Buyer[\s\S]*?TAX ID[:\s]*([\d-]+)/i)?.[1]?.trim();

// Dates
const dataWystawienia = text.match(/Issued\s+on[:\s]*([\d.-]+)/i)?.[1]?.trim() ||
                        text.match(/Issue\s+date[:\s]*([\d.-]+)/i)?.[1]?.trim();

const terminPlatnosci = text.match(/Due\s+date[:\s]*([\d.-]+)/i)?.[1]?.trim() ||
                        text.match(/Payment\s+due[:\s]*([\d.-]+)/i)?.[1]?.trim();

// Amounts
const kwotaNettoMatch = text.match(/(?:Net|Subtotal)[:\s]*\n?\s*([\d\s,]+)\s*USD/i);
const kwotaNetto = kwotaNettoMatch ? parseFloat(kwotaNettoMatch[1].replace(/\s/g, '').replace(',', '.')) : "";

const vatMatch = text.match(/(?:VAT|Tax)[:\s]*\n?\s*([\d\s,]+)\s*USD/i);
const vat = vatMatch ? parseFloat(vatMatch[1].replace(/\s/g, '').replace(',', '.')) : "";

const kwotaBruttoMatch = text.match(/(?:Total|Total\s+due)[:\s]*\n?\s*([\d\s,]+)\s*USD/i);
const kwotaBrutto = kwotaBruttoMatch ? parseFloat(kwotaBruttoMatch[1].replace(/\s/g, '').replace(',', '.')) : "";

// Description / Service name
const opis = text.match(/(?:Description|Service|Notes)[:\s]*\n?\s*([^\n\r]+)/i)?.[1]?.trim();

return {
  json: {
    "Invoice number": nrFaktury || "",
    "Contractor": kontrahent || "",
    "TAX ID": nip || "",
    "Issue date": dataWystawienia || "",
    "Due date": terminPlatnosci || "",
    "Net amount": kwotaNetto,
    "VAT": vat,
    "Gross amount": kwotaBrutto,
    "Description": opis || "Purchase of goods/services"
  }
};
```

How these expressions work — key elements:

- text.match(/pattern/i) — searches the entire text of the invoice for the given pattern, flag i ignores case sensitivity.
- ?.[1] — safe retrieval of the first capturing group (the fragment inside pattern parentheses ()) — if match() finds nothing and returns null, operator ?. prevents an error instead of crashing the Code node.
- ?.trim() — removes leading/trailing whitespaces from the matched fragment.
- .replace(/\s/g, '') — removes spaces from numbers (invoices often format amounts like 1 230.00).
- .replace(',', '.') — replaces comma decimal separators with dots, required by parseFloat().
- operator || between two .match() calls — tries two alternative patterns sequentially in case the invoice uses different phrasing for the same field.

Upon executing this node, the result is a complete JSON object with distinct, ready-to-use fields — when connected to the next node, they appear in the Input panel as single values ready to drag into Edit Fields or IF, exactly like form data on Day 28.

### Reading JPG turned out too difficult without AI

Initially, handling images (JPG) was planned as well, but extracting data from an invoice image without AI model support (e.g., OCR requiring an additional API) proved too difficult at this stage — Extract From File handles PDF text, but cannot "read" text from raster images. In practice, an intermediate solution was adopted: invoices are converted to PDF before sending via email, allowing the use of Extract From File exclusively without extra OCR tools.

---

## Step 8 – Rest of the workflow: unchanged relative to Day 28

After the Code node, data flows in the exact same manner as form data on Day 28: Edit Fields (adding Status = NEW, Date added = {{ $now }}, with Include Other Input Fields enabled) -> IF (validating presence of Invoice number, Contractor, Gross amount) -> error branch to Invoices — errors spreadsheet / valid branch to Invoices — register spreadsheet -> Switch classifying by Gross amount -> IF checking approaching due date -> Gmail with notification.

Amount classification (Switch) — range disjointness rule from Day 24/28 maintained: Very high >= 10000, High >= 5000 and < 10000, Medium >= 1000 and < 5000, Low < 1000, with rules ordered from most to least specific and "First matching output" enabled — in this version of Switch, there is still no AND operator within a single rule.

---

## Step 9 – Live test after publishing: "Invalid PDF structure" error

After publishing the workflow and testing it on live email traffic, an error occurred on the Extract From File node:

Invalid PDF structure.
Node type: n8n-nodes-base.extractFromFile

Diagnosis: an error of this type usually occurs for two reasons: the message contained no PDF attachment at all (e.g., an image in an email signature, logo, different file type), or there were multiple attachments and the node received a file that is not a PDF document.

Planned fix:

1. Filtering by file type before Extract From File. Adding an IF/Filter node between Gmail Trigger and Extract From File checking if the attachment name ends with .pdf:
   - Left value: {{ $json.fileName }} (or {{ $binary.attachment_0.fileName }})
   - Operator: Ends With
   - Right value: .pdf
   - Only the branch meeting the condition proceeds to Extract From File.

2. Securing settings of Extract From File itself. In the node's Settings tab, changing the On Error option from default Stop Workflow to Continue (using error output) or Continue (regular output) — thanks to this, a corrupted or unusual attachment does not halt the entire process.

---

## Full workflow diagram

```
Gmail Trigger (Search: subject:invoice)
        │
        ▼
Extract From File
(Operation: Extract Text from PDF,
 Input Binary Field: attachment_0)
        │
        ▼
Code — regex (Invoice number, Contractor,
TAX ID, dates, Net/VAT/Gross amount, Description)
        │
        ▼
Edit Fields (Status=NEW, Date added=$now)
        │
        ▼
      IF — validation
     /               \
 INVALID           VALID
    │                │
    ▼                ▼
Invoices - errors  Invoices - register
                       │
                       ▼
                    Switch
        (Gross amount, First matching output)
   ┌────────┬─────────┬──────────────┐
   ▼        ▼         ▼              ▼
  Low    Medium     High        Very high
                       │              │
                       └──────┬───────┘
                              ▼
                      IF — due date
                   (next 7 days?)
                              │
                              ▼
                      Gmail — notification
```

---

## Tests

Test 1 — regular invoice: Gross amount 750 USD. Checked: detection from email, proper data reading, saving to register, landing in Low category.

Test 2 — high-value invoice: Gross amount 7500 USD. Checked: register, Switch -> High, appropriate notification.

Test 3 — very high-value invoice: Gross amount 15000 USD. Checked: register, Switch -> Very high (exclusively, not simultaneously with High), appropriate notification.

Test 4 — invalid document: document that cannot be read properly or lacks required data. Checked: invoice does not land in main register, lands in Invoices - errors, error reason displayed.

---

## What I learned

- that Gmail Trigger does not behave like a Webhook during testing — it doesn't wait for a single test event, but ends testing immediately; real data is checked by activating the workflow and inspecting Executions
- filtering inbox messages (Search, Read Status, and other Gmail Trigger options) so the trigger only reacts to relevant emails
- that file data (Binary) is a distinct data type in n8n alongside JSON, and that Gmail's binary field is named attachment_0
- reading text from PDF using the built-in Extract From File node (formerly Read PDF), including precise configuration of Operation and Input Binary Field
- writing and iteratively refining regular expressions in a Code node to extract specific named fields from raw text — including practical use of optional chaining (?.), .trim(), space removal from numbers, and replacing commas with decimal dots
- that developing such a Code node iteratively (simple test on a few fields first, then matching actual document structure, finally full set of required fields) is far better than writing a final complex version outright
- that extracting data from images (JPG) without AI support is significantly harder than from PDF, and that choosing the right input format (PDF conversion) is often a more practical solution than building OCR from scratch
- that expanding existing automation with an automated data source (email instead of a form) is a qualitatively different shift than adding another integration — it shifts the boundary where human work ends and system work begins
- securing a production workflow against unusual/corrupted attachments by filtering file types and configuring error handling to Continue instead of Stop Workflow
