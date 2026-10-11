# N8N – AI Sales Enablement Engine

## Projekt: Mail od klienta → AI przeszukuje prywatną bazę wiedzy sprzedażowej (RAG) → gotowa odpowiedź handlowa w Gmailu

### Dlaczego to nie jest zwykły czat

Zwykły czat z AI odpowie na podstawie ogólnej wiedzy modelu. Tutaj model dostaje dostęp do **prywatnej bazy wiedzy firmy** (cennik, pakiety, karty konkurencji, argumenty na obiekcje klientów) i odpowiada wyłącznie w oparciu o te materiały. Całość uruchamia się samoczynnie po przyjściu maila od klienta, a wynik ląduje tam, gdzie handlowiec i tak pracuje: w skrzynce, jako gotowa odpowiedź.

To jest architektura RAG (Retrieval-Augmented Generation): najpierw wyszukanie trafnych fragmentów własnej bazy wiedzy, dopiero potem generowanie odpowiedzi.

### Cel lekcji

- zbudowanie pierwszego w pełni działającego układu RAG w n8n,
- praca z węzłem AI Agent i jego pod-węzłami (model, narzędzie, baza wektorowa, embeddingi),
- zasilenie bazy wektorowej własnym tekstem przez Code node i Default Data Loader,
- przekazanie wyniku agenta do Gmaila jako gotowej odpowiedzi na maila klienta,
- zrozumienie różnicy między trybem zapisu (Insert Documents) a odczytu (Retrieve Documents) bazy wektorowej.

---

## Krok 1 – Gmail Trigger (odbiór wiadomości)

Node **Gmail Trigger**.

- Credential: konto Gmail
- Event: `Message Received`
- Filters / Query (opcjonalnie): `is:unread` — workflow reaguje tylko na nieprzeczytane maile

Uwaga z Dnia 29: Gmail Trigger testowany ręcznie nie czeka na nową wiadomość, tylko pobiera istniejącą. Do testów wystarczy mail, który już leży w skrzynce.

## Krok 2 – AI Agent (mózg systemu)

Node **AI Agent**.

- Agent Type: `Tools Agent`
- **Source for Prompt (User Message): `Define below`** (domyślnie ustawione jest "Connected Chat Trigger Node", co tutaj nie ma sensu, bo wiadomość przychodzi z Gmaila, a nie z czatu)
- Po tej zmianie pojawia się pole **Prompt (User Message)**, w które wpisuję treść maila z Gmail Triggera: `{{ $json.snippet }}` (albo `{{ $json.text }}`, zależnie od tego, które pole z treścią zwraca trigger w danej wersji)
- Options → System Message:

```
Jesteś ekspertem ds. sprzedaży B2B oraz wsparcia handlowego (Sales Enablement). Twoim zadaniem jest przeanalizowanie treści przychodzącego maila od klienta, odnalezienie odpowiednich argumentów i kart walki (battlecards) w bazie wiedzy za pomocą narzędzia Vector Store, a następnie napisanie profesjonalnej, rzeczowej i perswazyjnej odpowiedzi handlowej odpierającej obiekcje klienta.
```

## Krok 3 – OpenAI Chat Model (model językowy agenta)

Do dolnego wejścia **Chat Model** w AI Agencie podpięty node **OpenAI Chat Model**.

- Credential: klucz OpenAI API
- Model: `gpt-4.1-mini`

## Krok 4 – Narzędzie RAG: Vector Store Question Answer Tool

Do dolnego wejścia **Tool** w AI Agencie podpięty node **Vector Store Question Answer Tool** (na kanwie widoczny jako "Answer questions with a vector store").

- Description of Data: `Baza wiedzy sprzedażowej B2B zawierająca cennik, pakiety oraz karty obiekcji i argumentacji (battlecards) dotyczące konkurencji.`

Opis jest ważny: na jego podstawie agent decyduje, kiedy sięgnąć po to narzędzie.

Pod tym node'em są dwa wejścia, oba wymagane:

- **Model\*** — podpięty osobny node **OpenAI Chat Model** (model narzędzia, `gpt-4o-mini`). Samo narzędzie odpowiada na pytania do bazy własnym modelem, niezależnie od modelu agenta.
- **Vector Store\*** — podpięty node **Simple Vector Store** (w starszych wersjach n8n: In-Memory Vector Store).

## Krok 5 – Baza wektorowa, embeddingi i baza wiedzy

Node **Simple Vector Store** ma własne wejścia dla pod-węzłów:

- **Embedding\*** — node **Embeddings OpenAI**, model `text-embedding-3-small`. Embeddingi zamieniają tekst na wektory liczbowe, po których baza wyszukuje fragmenty o podobnym znaczeniu do pytania.
- **Document\*** — node **Default Data Loader**, który wczytuje dane do zindeksowania.

### Skąd wziąć dane: Code node z treścią bazy wiedzy

Pierwotny plan zakładał wczytanie pliku `baza_wiedzy_sales.md` z dysku. W praktyce prostsze okazało się umieszczenie treści bazy wiedzy w node'ie **Code in JavaScript**, który zwraca ją jako jeden element:

```javascript
return [{
  json: {
    text: `TUTAJ WKLEJ CAŁĄ TREŚĆ BAZY WIEDZY: cennik, pakiety, opisy produktów, karty konkurencji, odpowiedzi na typowe obiekcje`
  }
}];
```

Przykładowy podział treści w bazie, który ułatwia wyszukiwanie: sekcja z cennikiem i pakietami, sekcja z typowymi obiekcjami klientów i gotowymi odpowiedziami, sekcja z kartami walki dla poszczególnych konkurentów.

Ważne: w node'ie Code trzeba **zastąpić przykładowy tekst prawdziwą treścią** i uruchomić go (Execute step), żeby dane przeszły dalej. Inaczej do bazy wektorowej trafi placeholder.

### Indeksowanie: Insert Documents, potem Retrieve Documents

1. Pod Simple Vector Store, przy gnieździe **Document\***, dodaję **Default Data Loader** (szary plus pod napisem Document).
2. W ustawieniach Default Data Loader, w panelu INPUT, w polu **Parent nodes** wybieram z listy node **Code in JavaScript**. To wskazuje loaderowi, skąd ma brać dane.
3. Otwieram Simple Vector Store, tryb **Insert Documents**, klikam **Execute step**. Dane są dzielone na fragmenty (u mnie 3 elementy), przeliczane na wektory i zapisywane w pamięci. Cała dolna część węzła świeci się wtedy na zielono.
4. Dopiero po tym przełączam **Operation Mode** na **Retrieve Documents (As Vector Store for Chain/Tool)**. Gniazdo Document znika, a węzeł można połączyć z portem **Vector Store\*** narzędzia z Kroku 4.

Zasada: Insert wykonuję **jeden raz**, żeby zapisać bazę wiedzy. Retrieve to tryb pracy właściwej, w którym agent tylko czyta z bazy.

## Krok 6 – Gmail: odpowiedź na maila klienta

Po prawej stronie AI Agenta node **Gmail**.

- Resource: `Draft`, Operation: `Create` (szkic do zatwierdzenia przez człowieka, ta sama zasada co w asystencie skrzynki z Dnia 35: AI przygotowuje, człowiek wysyła)
- To: `{{ $json.from }}`
- Subject: `Re: {{ $json.subject }}`
- Message: `{{ $json.output }}`

`output` to pole, w którym AI Agent zwraca wygenerowaną odpowiedź. W trakcie testów w workflowie pojawił się też wariant z node'em **Send a message** (te same pola To i Message), wygodniejszy do szybkich testów, ale do pracy docelowej bezpieczniejszy jest szkic.

Nazwy pól adresata i tematu zależą od wersji Gmail Triggera i ustawienia uproszczonego outputu. W pierwszej wersji instrukcji pojawiło się `sender`, w praktyce używane było `from`. Zawsze sprawdzam dokładne nazwy w output triggera, zamiast zgadywać (ta sama lekcja co przy błędzie z polem Email w Dniu 30).

---

## Napotkane problemy i naprawy

### Problem 1: czerwony wykrzyknik przy Simple Vector Store, brak źródła dokumentów

Gniazdo **Document\*** było puste. Naprawa: dodanie pod-węzła **Default Data Loader** (szary plus pod napisem Document).

### Problem 2: Default Data Loader nie widział żadnych danych

W polu **Parent nodes** pojawiał się komunikat "No nodes found", bo na kanwie nie było jeszcze node'a z treścią. Naprawa: dodanie node'a **Code in JavaScript** z tekstem bazy wiedzy, wykonanie go, a następnie wybranie go w polu Parent nodes.

### Problem 3: w bazie wektorowej wylądowałby tekst przykładowy

Code node zawierał domyślny placeholder. Naprawa: wklejenie właściwej treści w pole `text` i ponowne uruchomienie node'a, zanim ruszy indeksowanie.

### Problem 4: jak połączyć bazę z narzędziem

Dopóki Simple Vector Store działał w trybie Insert Documents, nie dało się go podpiąć do narzędzia jako źródła odczytu. Naprawa: po udanym indeksowaniu przełączenie trybu na **Retrieve Documents (As Vector Store for Chain/Tool)** i przeciągnięcie połączenia do portu **Vector Store\*** w narzędziu. Wykrzyknik przy narzędziu znika po podpięciu.

### Problem 5: puste gniazdo Model\* w narzędziu RAG

Narzędzie odpowiadające na pytania z bazy wymaga własnego modelu. Naprawa: plus przy **Model\*** i dodanie **OpenAI Chat Model** (`gpt-4o-mini`).

### Problem 6: czerwony wykrzyknik przy AI Agent mimo wpisanego `{{ $json.snippet }}`

Treść promptu wpisana była w pole **Source for Prompt (User Message)**, które jest rozwijaną listą wyboru źródła promptu, a nie polem na tekst. Naprawa: zmiana wartości listy na **Define below**. Dopiero wtedy pojawia się osobne pole **Prompt (User Message)**, w które wstawiam `{{ $json.snippet }}`. Pomaga też przeciągnięcie pola `snippet` z panelu INPUT po lewej, bo n8n wstawia wtedy poprawne wyrażenie.

---

## Test

Test wykonuję przyciskiem **Execute workflow**, na mailu, który leży w skrzynce.

Trzy maile testowe, które dobrze sprawdzają bazę wiedzy:

1. Obiekcja cenowa, np. klient pisze, że oferta jest za droga.
2. Porównanie z konkurentem, np. klient pisze, że konkurencja daje to samo taniej.
3. Pytanie o zawartość pakietów.

Co sprawdzam w wygenerowanej odpowiedzi:

- czy ceny, nazwy pakietów i argumenty zgadzają się z tekstem z Code node,
- czy agent nie dopisuje liczb ani funkcji, których nie ma w bazie,
- czy ton jest handlowy i rzeczowy,
- czy w node'ie Gmail pojawił się gotowy szkic (lub wiadomość) z poprawnym adresatem i tematem `Re: ...`.

---

## Pełny schemat workflowa

Główna ścieżka (co się dzieje przy każdym mailu):

```
Gmail Trigger (Message Received, is:unread)
        │
        ▼
AI Agent (Tools Agent, Source for Prompt: Define below)
   │
   ├── Chat Model ──► OpenAI Chat Model (gpt-4.1-mini)
   │
   └── Tool ──► Answer questions with a vector store
                    │
                    ├── Model ──► OpenAI Chat Model (gpt-4o-mini)
                    │
                    └── Vector Store ──► Simple Vector Store
                                         (Retrieve Documents)
                                              │
                                              └── Embedding ──► Embeddings OpenAI
                                                               (text-embedding-3-small)
        │
        ▼
Gmail (Create Draft: To = from, Subject = Re: subject, Message = output)
```

Indeksowanie bazy wiedzy (wykonywane osobno, jednorazowo, ręcznie):

```
Code in JavaScript (tekst bazy wiedzy)
        │
        ▼
Default Data Loader (Parent node: Code in JavaScript)
        │
        ▼
Simple Vector Store (Insert Documents)
        │
        └── Embedding ──► Embeddings OpenAI (text-embedding-3-small)
```

Po zakończeniu indeksowania ten sam Simple Vector Store przełączany jest w tryb Retrieve Documents i pracuje w głównej ścieżce.

---

## Ograniczenia tej wersji i co dalej

- **Pamięć nietrwała.** Simple Vector Store trzyma dane w pamięci instancji n8n. Po restarcie serwera trzeba ponownie wykonać indeksowanie (Insert Documents). Do pracy produkcyjnej zamiast niego potrzebna jest trwała baza wektorowa, np. PostgreSQL z rozszerzeniem pgvector.
- **Ochrona przed zmyślaniem.** Warto dopisać do System Message instrukcję, żeby agent odpowiadał wyłącznie na podstawie bazy wiedzy, a gdy brakuje informacji, zaznaczał, że trzeba ją potwierdzić, zamiast wymyślać cenę czy funkcję.
- **Rozszerzenie: rejestr obiekcji dla marketingu.** Pomysł z założeń lekcji, którego nie ma w powyższych krokach: po AI Agencie dodać node Google Sheets (Append Row) zapisujący datę, nadawcę, treść maila i wygenerowaną odpowiedź, a w System Message poprosić agenta o dodatkowe oznaczenie rodzaju obiekcji (cena, konkurencja, funkcje, termin). Dział marketingu widziałby wtedy, z jakimi obiekcjami klienci zgłaszają się najczęściej.

---

## Czego się nauczyłem

- czym jest RAG i jak wygląda jego odpowiednik w n8n: AI Agent, narzędzie Vector Store Question Answer Tool, baza wektorowa, embeddingi i loader dokumentów
- że AI Agent ma pod-węzły podpinane od spodu (model, narzędzie, pamięć), a narzędzie RAG ma własne, oddzielne gniazda Model i Vector Store
- że pole Source for Prompt w AI Agencie jest listą wyboru i żeby wpisać własny prompt, trzeba ustawić Define below
- jak zasilić bazę wektorową własnym tekstem przez Code node i Default Data Loader oraz że w polu Parent nodes trzeba wskazać źródło danych
- różnicy między trybem Insert Documents (zapis, jednorazowo) a Retrieve Documents (odczyt przez agenta) i że dopiero tryb odczytu pozwala podpiąć bazę do narzędzia
- że Simple Vector Store działa w pamięci i po restarcie wymaga ponownego indeksowania, więc do produkcji potrzebna jest trwała baza (np. pgvector)
- że nazwy pól z Gmail Triggera trzeba zawsze sprawdzać w outpucie, zamiast zakładać je z góry
- że szkic odpowiedzi do zatwierdzenia jest bezpieczniejszym wyjściem niż automatyczna wysyłka do klienta
