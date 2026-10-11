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

- konfiguracji RAG i jak wygląda jego odpowiednik w n8n: AI Agent, narzędzie Vector Store Question Answer Tool, baza wektorowa, embeddingi i loader dokumentów
- że AI Agent ma pod-węzły podpinane od spodu (model, narzędzie, pamięć), a narzędzie RAG ma własne, oddzielne gniazda Model i Vector Store
- że pole Source for Prompt w AI Agencie jest listą wyboru i żeby wpisać własny prompt, trzeba ustawić Define below
- jak zasilić bazę wektorową własnym tekstem przez Code node i Default Data Loader oraz że w polu Parent nodes trzeba wskazać źródło danych
- różnicy między trybem Insert Documents (zapis, jednorazowo) a Retrieve Documents (odczyt przez agenta) i że dopiero tryb odczytu pozwala podpiąć bazę do narzędzia
- że Simple Vector Store działa w pamięci i po restarcie wymaga ponownego indeksowania, więc do produkcji potrzebna jest trwała baza (np. pgvector)
- że nazwy pól z Gmail Triggera trzeba zawsze sprawdzać w outpucie, zamiast zakładać je z góry
- że szkic odpowiedzi do zatwierdzenia jest bezpieczniejszym wyjściem niż automatyczna wysyłka do klienta

- # n8n – AI Sales Enablement Engine

## Project: Customer email → AI searches the private sales knowledge base (RAG) → ready-to-use sales reply in Gmail

### Why this is not just an ordinary chat

A regular AI chat answers using the model's general knowledge. Here, the model gets access to the **company's private knowledge base** (pricing, packages, competitor battlecards, and objection-handling arguments) and responds based only on those materials. The process starts automatically when a customer email arrives, and the result lands where the sales representative already works: in the inbox as a ready-to-use reply.

This is a RAG (Retrieval-Augmented Generation) architecture: first, relevant passages from the company's own knowledge base are retrieved; only then is the answer generated.

### Lesson objectives

- build a fully working RAG setup in n8n,
- work with the AI Agent node and its sub-nodes (model, tool, vector store, embeddings),
- populate a vector database with custom text using a Code node and Default Data Loader,
- pass the agent's output to Gmail as a ready-to-use reply to a customer's email,
- understand the difference between vector-store write mode (`Insert Documents`) and read mode (`Retrieve Documents`).

---

## Step 1 – Gmail Trigger (receiving messages)

Node: **Gmail Trigger**.

- Credential: Gmail account
- Event: `Message Received`
- Filters / Query (optional): `is:unread` — the workflow reacts only to unread emails

**Note from Day 29:** When tested manually, Gmail Trigger does not wait for a new message; it fetches an existing one. For testing, an email that is already in the inbox is enough.

## Step 2 – AI Agent (the system's brain)

Node: **AI Agent**.

- Agent Type: `Tools Agent`
- **Source for Prompt (User Message): `Define below`** (the default is `Connected Chat Trigger Node`, which does not make sense here because the message comes from Gmail, not a chat)
- After changing this setting, the **Prompt (User Message)** field appears. I put the email text from Gmail Trigger there: `{{ $json.snippet }}` (or `{{ $json.text }}`, depending on which field the trigger returns in this version)
- Options → System Message:

```
You are an expert in B2B sales and sales support (Sales Enablement). Your task is to analyze the incoming customer email, find relevant arguments and competitor battlecards in the knowledge base using the Vector Store tool, and then write a professional, clear, persuasive sales reply that addresses the customer's objections.
```

## Step 3 – OpenAI Chat Model (the agent's language model)

The **OpenAI Chat Model** node is connected to the AI Agent's lower **Chat Model** input.

- Credential: OpenAI API key
- Model: `gpt-4.1-mini`

## Step 4 – RAG tool: Vector Store Question Answer Tool

The **Vector Store Question Answer Tool** node is connected to the AI Agent's lower **Tool** input (displayed on the canvas as “Answer questions with a vector store”).

- Description of Data: `B2B sales knowledge base containing pricing, packages, and competitor objection-handling and argumentation battlecards.`

The description matters: the agent uses it to decide when to call the tool.

The tool has two required inputs:

- **Model\*** — a separate **OpenAI Chat Model** node is connected here (the tool's model, `gpt-4o-mini`). The tool uses its own model to answer questions about the knowledge base, independently of the agent's model.
- **Vector Store\*** — a **Simple Vector Store** node is connected here (in older n8n versions: In-Memory Vector Store).

## Step 5 – Vector database, embeddings, and the knowledge base

The **Simple Vector Store** node has its own sub-node inputs:

- **Embedding\*** — **Embeddings OpenAI**, model `text-embedding-3-small`. Embeddings convert text into numerical vectors that the database can use to find passages with similar meaning to a question.
- **Document\*** — **Default Data Loader**, which loads the data to be indexed.

### Where the data comes from: a Code node containing the knowledge base text

The original plan was to load a `sales_knowledge_base.md` file from disk. In practice, it was easier to put the knowledge-base text in a **Code in JavaScript** node, which returns it as one item:

```javascript
return [{
  json: {
    text: `PASTE THE FULL KNOWLEDGE BASE HERE: pricing, packages, product descriptions, competitor battlecards, answers to common objections`
  }
}];
```

Example sections that make searching easier: pricing and packages, common customer objections with suggested answers, and battlecards for individual competitors.

**Important:** replace the example text in the Code node with the actual knowledge-base content and run it (`Execute step`) before indexing. Otherwise, the placeholder will be added to the vector database.

### Indexing: Insert Documents, then Retrieve Documents

1. Under Simple Vector Store, next to the **Document\*** input, add **Default Data Loader** (the grey plus beneath the word Document).
2. In Default Data Loader settings, open the INPUT panel and select **Code in JavaScript** in the **Parent nodes** field. This tells the loader where to get its data.
3. Open Simple Vector Store, select **Insert Documents**, and click **Execute step**. The data is split into chunks (three items in my case), converted into vectors, and stored in memory. The lower part of the node turns green when this succeeds.
4. Only after that, switch **Operation Mode** to **Retrieve Documents (As Vector Store for Chain/Tool)**. The Document input disappears, and the node can be connected to the **Vector Store\*** input of the tool from Step 4.

Rule: run Insert once to store the knowledge base. Retrieve is the mode used during normal operation, when the agent reads from the database.

## Step 6 – Gmail: reply to the customer's email

A **Gmail** node is placed to the right of the AI Agent.

- Resource: `Draft`, Operation: `Create` (a draft for human approval; the same principle as the inbox assistant from Day 35: AI prepares the reply, a person sends it)
- To: `{{ $json.from }}`
- Subject: `Re: {{ $json.subject }}`
- Message: `{{ $json.output }}`

`output` is the field where AI Agent returns the generated reply. During testing, I also used the **Send a message** node operation (the same To and Message fields), which is convenient for quick tests, but creating a draft is safer for the intended workflow.

The recipient and subject field names depend on the Gmail Trigger version and simplified-output settings. The first version of the instructions used `sender`; in practice, I used `from`. Always check the trigger's actual output instead of guessing (the same lesson as the Email-field error on Day 30).

---

## Problems encountered and fixes

### Problem 1: red exclamation mark by Simple Vector Store; no document source

The **Document\*** input was empty. Fix: add a **Default Data Loader** sub-node (the grey plus beneath Document).

### Problem 2: Default Data Loader could not see any data

The **Parent nodes** field showed “No nodes found” because the canvas did not yet contain a node with the knowledge-base text. Fix: add a **Code in JavaScript** node with the knowledge-base text, execute it, then select it in Parent nodes.

### Problem 3: the vector database would have received placeholder text

The Code node contained the default placeholder. Fix: paste the actual knowledge-base content into the `text` field and run the node again before indexing.

### Problem 4: connecting the database to the tool

While Simple Vector Store was in Insert Documents mode, it could not be connected to the tool as a read source. Fix: after indexing successfully, switch to **Retrieve Documents (As Vector Store for Chain/Tool)** and connect it to the tool's **Vector Store\*** input. The tool's exclamation mark disappears once the connection is made.

### Problem 5: empty Model\* input on the RAG tool

The knowledge-base question-answering tool requires its own model. Fix: click the plus next to **Model\*** and add **OpenAI Chat Model** (`gpt-4o-mini`).

### Problem 6: red exclamation mark by AI Agent even though `{{ $json.snippet }}` was entered

The prompt text had been entered into **Source for Prompt (User Message)**, which is a dropdown for choosing the prompt source, not a text field. Fix: change the dropdown value to **Define below**. The separate **Prompt (User Message)** field then appears, and I put `{{ $json.snippet }}` there. Dragging the `snippet` field from the INPUT panel also helps because n8n inserts the correct expression.

---

## Test

I test the workflow using **Execute workflow** with an email that is already in the inbox.

Three test emails that are useful for checking the knowledge base:

1. A pricing objection, e.g. the customer says the offer is too expensive.
2. A competitor comparison, e.g. the customer says a competitor offers the same thing for less.
3. A question about what is included in the packages.

When reviewing the generated reply, I check:

- whether prices, package names, and arguments match the Code node's knowledge-base text,
- whether the agent invents numbers or features that are not in the knowledge base,
- whether the tone is professional and sales-oriented,
- whether the Gmail node creates a draft (or sends a message) with the correct recipient and subject `Re: ...`.

---

## Full workflow diagram

Main path (what happens for every email):

```text
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

Knowledge-base indexing (performed separately, manually, once):

```text
Code in JavaScript (knowledge-base text)
        │
        ▼
Default Data Loader (Parent node: Code in JavaScript)
        │
        ▼
Simple Vector Store (Insert Documents)
        │
        └── Embedding ──► Embeddings OpenAI (text-embedding-3-small)
```

After indexing is complete, the same Simple Vector Store is switched to Retrieve Documents mode and used by the main workflow.

---

## Limitations of this version and next steps

- **Non-persistent memory.** Simple Vector Store keeps data in the n8n instance's memory. After a server restart, the knowledge base must be indexed again using Insert Documents. For production, use a persistent vector database, e.g. PostgreSQL with the pgvector extension.
- **Reducing hallucinations.** Add an instruction to the System Message telling the agent to answer only from the knowledge base and, when information is missing, say it needs to be confirmed rather than inventing a price or feature.
- **Extension: objection tracking for marketing.** An idea from the lesson plan that is not included in the steps above: add a Google Sheets node after AI Agent (Append Row) to log the date, sender, email text, and generated reply. Ask the agent in the System Message to label the objection type as well (price, competitor, features, timing). Marketing could then see which objections customers raise most often.

---

## What I learned

- RAG configuration is and how it maps to n8n: AI Agent, Vector Store Question Answer Tool, vector database, embeddings, and document loader,
- that AI Agent has sub-nodes connected underneath it (model, tool, memory), and the RAG tool has its own separate Model and Vector Store inputs,
- that Source for Prompt in AI Agent is a dropdown, and `Define below` must be selected to enter a custom prompt,
- how to populate a vector database with custom text using a Code node and Default Data Loader, and that Parent nodes must identify the source node,
- the difference between Insert Documents (one-time write) and Retrieve Documents (read by the agent), and that the database can only be connected to the tool in read mode,
- that Simple Vector Store is in-memory and requires re-indexing after a restart, so production needs persistent storage such as pgvector,
- that Gmail Trigger field names should always be checked in the actual output instead of assumed,
- that creating a reply draft for approval is safer than automatically sending a response to a customer.
