# n8n – AI Product Content Generator

## Projekt: Zdjęcie produktu → analiza AI → post do social mediów → wygenerowana grafika → e-mail z gotową paczką marketingową

### Cel lekcji

Zbudowanie workflowa, który przyjmuje nazwę i zdjęcie produktu przez formularz, analizuje materiał wejściowy za pomocą modelu OpenAI, przygotowuje tekst posta do social mediów oraz prompt do grafiki, generuje obraz i wysyła wynik w wiadomości e-mail.

Workflow łączy kilka elementów w jeden proces: formularz, analizę obrazu, generowanie tekstu, generowanie grafiki oraz dostarczenie gotowego rezultatu przez Gmail.

## Pełny schemat workflowa

Główna ścieżka:

```text
[n8n Form Trigger]
       │
       ▼
[OpenAI: analiza zdjęcia i przygotowanie treści]
       │
       ├── post do social mediów
       │
       └── prompt graficzny po angielsku
       │
       ▼
[OpenAI: Generate an image – GPT-Image-1-Mini]
       │
       ▼
[Gmail: wysłanie wiadomości HTML + grafika jako załącznik]
```

W trakcie konfiguracji zmieniłem początkowy pomysł z generowaniem grafiki przez Pollinations.ai/Flux na gotowy węzeł OpenAI `Generate an image`. Dzięki temu grafika powstaje bezpośrednio przez OpenAI API, a osobny węzeł HTTP Request nie jest potrzebny.

---

## Krok 1 – n8n Form Trigger (formularz wejściowy)

Na początku dodałem węzeł **n8n Form Trigger**, który przyjmuje dane potrzebne do wygenerowania paczki marketingowej.

Formularz zawiera dwa pola:

- **Nazwa Produktu** – typ `Text`
- **Zdjecie** – typ `File`

Formularz jest punktem startowym workflowa. Po przesłaniu formularza n8n dostaje nazwę produktu oraz plik zdjęcia, który będzie analizowany przez model AI.

### Problem 1: Wskazanie pliku binarnego

W ustawieniach kolejnego węzła pole **Input Data Field Name** miało domyślną wartość `data`, ale plik z formularza mógł być zapisany pod inną nazwą.

**Rozwiązanie:**

1. W panelu **INPUT** po lewej stronie otworzyłem widok `Binary`.
2. Sprawdziłem dokładną nazwę pola zawierającego przesłane zdjęcie.
3. W polu **Input Data Field Name** wpisałem nazwę zgodną z danymi binarnymi widocznymi w n8n.

Wniosek: przy pracy z plikami nie należy zgadywać nazwy pola. Trzeba sprawdzić ją w zakładce `Binary` w danych wejściowych danego węzła.

---

## Krok 2 – OpenAI: analiza zdjęcia i przygotowanie treści

Węzeł OpenAI analizuje przesłane zdjęcie produktu i na tej podstawie przygotowuje dwa wyniki:

1. treść posta do social mediów,
2. krótki, opisowy prompt po angielsku do wygenerowania grafiki promocyjnej.

W pierwszej wersji plan zakładał zwrócenie odpowiedzi jako JSON z polami:

```json
{
  "post_social_media": "Treść posta do social mediów",
  "prompt_graficzny": "Prompt po angielsku opisujący tło reklamowe produktu"
}
```

W trakcie konfiguracji pojawił się też węzeł **Message a model** oraz model `GPT-4.1-MINI`. Konkretna struktura odpowiedzi zależy od ustawień i wersji węzła OpenAI, dlatego trzeba sprawdzić rzeczywisty output po wykonaniu testu.

### Problem 2: wyrażenie `{{ $json.message.content }}` było czerwone

n8n nie znajdował pola `message.content` w danych wejściowych. W zależności od ustawień output tekst mógł znajdować się w innym polu, np. `text`, `output` albo w zagnieżdżonej strukturze odpowiedzi.

**Rozwiązanie:**

1. Uruchomiłem poprzedni węzeł przez **Execute step**, żeby zobaczyć jego rzeczywiste dane wyjściowe.
2. W panelu **INPUT** sprawdziłem strukturę odpowiedzi w widoku `Schema` lub `JSON`.
3. Zamiast zakładać z góry nazwę pola, wskazałem faktyczne pole z wygenerowanym tekstem. Można je przeciągnąć z panelu INPUT do pola wyrażenia, aby n8n wstawił poprawną ścieżkę.

Wniosek: czerwone wyrażenie często oznacza, że wskazana ścieżka nie istnieje w aktualnych danych. Najpierw trzeba sprawdzić output poprzedniego węzła.

### Co oznacza `\n\nAnaliza:`

`\n` oznacza znak nowej linii. Zapis `\n\nAnaliza:` w tekście promptu tworzy odstęp przed sekcją „Analiza:”. To sposób na czytelne oddzielenie instrukcji od treści, którą przekazujemy modelowi.

---

## Krok 3 – przygotowanie danych i przekazywanie wyniku dalej

W początkowym projekcie planowałem dodać węzeł **Code** lub **Edit Fields**, który sparsuje JSON zwrócony przez model i wyciągnie osobno post oraz prompt graficzny.

Przykładowa logika polegała na odczytaniu tekstu odpowiedzi, usunięciu ewentualnych znaczników Markdown i wykonaniu `JSON.parse()`.

W praktyce struktura outputu OpenAI może się różnić. Przed użyciem kodu należy więc sprawdzić, gdzie dokładnie znajduje się tekst odpowiedzi. Nie należy kopiować ścieżki `message.content` bez potwierdzenia, że taka struktura istnieje w aktualnym workflowie.

---

## Krok 4 – wybór działania OpenAI

Podczas konfiguracji listy działań OpenAI wybrałem **Message a model** z sekcji `TEXT ACTIONS`, aby przygotować tekst posta oraz opis grafiki.

Model otrzymuje informacje o produkcie i instrukcję, by przygotować treść promocyjną oraz prompt graficzny. W tym kroku ważne było również prawidłowe wskazanie danych wejściowych z poprzednich węzłów.

### Problem 3: nie wiedziałem, co przeciągnąć z panelu INPUT

Nie należy przeciągać całego węzła ani przypadkowego fragmentu danych. Trzeba znaleźć konkretną właściwość zawierającą potrzebny tekst, np. treść odpowiedzi modelu albo nazwę produktu.

**Sposób postępowania:**

1. Rozwinąć właściwy węzeł w panelu INPUT.
2. Odszukać pole z potrzebną wartością.
3. Przeciągnąć samo pole do odpowiedniego miejsca w edytorze wyrażenia.
4. Sprawdzić podgląd wyniku i upewnić się, że pokazuje oczekiwany tekst.

### Problem 4: błędna nazwa węzła w wyrażeniu

Pojawił się komunikat `Referenced node doesn't exist`. Wyrażenie odwoływało się do węzła o nazwie `n8n Form Trigger`, podczas gdy rzeczywista nazwa w workflowie była inna.

**Rozwiązanie:** użyć rzeczywistej nazwy węzła z kanwy albo przeciągnąć pole z panelu INPUT, aby n8n wstawił prawidłowe odwołanie.

Wniosek: nazwa w wyrażeniu musi zgadzać się z nazwą węzła w workflowie.

---

## Krok 5 – generowanie grafiki przez OpenAI

Do wygenerowania obrazu użyłem węzła OpenAI **Generate an image** z modelem `GPT-Image-1-Mini`.

Prompt do generowania grafiki powinien pochodzić z poprzedniego etapu, w którym model przygotowuje opis wizualny po angielsku.

Po przejściu na generowanie obrazu przez OpenAI nie potrzebuję osobnego węzła HTTP Request do Pollinations.ai/Flux. Wygenerowany obraz jest zwracany jako dane binarne, a niekoniecznie jako publiczny URL.

### Problem 5: wątpliwości dotyczące kosztów

W workflowie występują osobne operacje AI: analiza zdjęcia i generowanie obrazu. Ich koszty nie są tym samym co koszt pojedynczego krótkiego zapytania tekstowego. Warto sprawdzić aktualny cennik OpenAI dla używanego modelu oraz monitorować zużycie w panelu API.

Wniosek: model „mini” nie oznacza, że cała automatyzacja jest darmowa. Koszt zależy od użytych modeli, liczby wywołań, ilości przetwarzanych danych i generowania obrazów.

---

## Krok 6 – Gmail: wysłanie gotowej paczki marketingowej

Na końcu workflowa znajduje się węzeł **Gmail**, który wysyła wiadomość HTML z nazwą produktu, treścią posta oraz informacją o wygenerowanej grafice.

Pole **Message** ustawiam w trybie **Expression**, ponieważ treść wiadomości zawiera dynamiczne dane z poprzednich węzłów.

### Problem 6: `Fixed` czy `Expression`?

Wybrałem **Expression** dla treści HTML, ponieważ wiadomość zawiera wartości pobierane z poprzednich węzłów, np. nazwę produktu i treść posta.

- `Fixed` – zwykły, stały tekst.
- `Expression` – tekst, w którym n8n oblicza wyrażenia i wstawia aktualne dane.

### Problem 7: grafika nie wyświetlała się w środku wiadomości

Próba użycia `json.url` nie działała, ponieważ węzeł `Generate an image` zwracał plik binarny, a nie pole z adresem URL. Próba osadzenia danych Base64 bezpośrednio w HTML również nie była dobrym rozwiązaniem dla tej wiadomości.

**Rozwiązanie, które zadziałało:** wysłanie grafiki jako załącznika do e-maila.

1. Usunąłem nieprawidłowy znacznik `<img ... />` z treści HTML.
2. W ustawieniach Gmail, w sekcji **Options**, dodałem opcję **Attachments**.
3. W polu **Input Data Field Name** wpisałem `data`, zgodnie z nazwą pola binarnego widoczną w panelu INPUT.
4. Uruchomiłem węzeł przez **Execute step**.

Wynik: e-mail został wysłany, a grafika dotarła jako załącznik.

### Finalna wersja treści wiadomości HTML

Wiadomość pokazuje nazwę produktu i treść posta, a przy grafice informuje, że plik znajduje się w załączniku. Nie zawiera błędnego adresu obrazka ani osadzonych danych Base64.

```html
<div style="font-family: Arial, sans-serif; max-width: 600px; margin: 0 auto; border: 1px solid #e0e0e0; border-radius: 8px; padding: 20px; background-color: #ffffff;">
  <h2 style="color: #1a73e8; margin-top: 0;">Wygenerowano nową paczkę marketingową</h2>

  <p><strong>Produkt:</strong> {{ $('On form submission').item.json['Nazwa Produktu'] }}</p>

  <hr style="border: none; border-top: 1px solid #eee; margin: 15px 0;" />

  <h3 style="color: #333;">Wygenerowana treść posta (Social Media)</h3>
  <div style="background-color: #f9f9f9; padding: 15px; border-left: 4px solid #1a73e8; border-radius: 4px; white-space: pre-wrap; font-size: 14px; line-height: 1.6; color: #2c3e50;">
{{ $('Message a model').item.json.output[0].content[0].text }}
  </div>

  <hr style="border: none; border-top: 1px solid #eee; margin: 20px 0;" />

  <h3 style="color: #333;">Wygenerowana grafika promocyjna</h3>
  <div style="background-color: #e8f0fe; padding: 12px 15px; border-radius: 6px; color: #174ea6; font-size: 13px;">
    Wygenerowana grafika promocyjna została dołączona do tej wiadomości jako załącznik PNG.
  </div>

  <p style="font-size: 12px; color: #888; text-align: center; margin-top: 25px;">
    Automatyzacja wygenerowana przez n8n + OpenAI API
  </p>
</div>
```

Uwaga z praktyki: nazwy `On form submission`, `Message a model` i ścieżka `output[0].content[0].text` muszą odpowiadać faktycznym nazwom i strukturze outputu w danej instancji. Jeśli węzły zostały nazwane inaczej albo zmienił się format odpowiedzi, trzeba sprawdzić dane wejściowe i poprawić wyrażenia.

---

## Po co workflow przyjmuje zdjęcie?

Zdjęcie jest wejściem do analizy multimodalnej. Model może na jego podstawie rozpoznać widoczne cechy produktu i wykorzystać je do przygotowania trafniejszego posta oraz promptu graficznego.

Do formularza powinno trafiać zdjęcie produktu, który ma być promowany, np. zdjęcie wykonane telefonem albo fotografia produktowa. Nazwa produktu pomaga dodatkowo określić, czego dotyczy materiał.

To podejście ma sens, jeśli celem jest przygotowanie materiałów na podstawie konkretnego produktu ze zdjęcia. Jeśli post ma powstawać wyłącznie z nazwy, opisu tekstowego lub briefu, zdjęcie nie jest niezbędne i formularz można później uprościć.

---

## Test końcowy

Po skonfigurowaniu workflowa wysłałem test z formularza. Gmail poprawnie wysłał wiadomość, a grafika dotarła jako załącznik.

Przy sprawdzaniu wyniku warto zweryfikować:

- czy nazwa produktu w mailu jest poprawna,
- czy post odpowiada rzeczywistemu produktowi i nie zawiera wymyślonych szczegółów,
- czy wygenerowana grafika znajduje się w załączniku,
- czy wiadomość nie zawiera nieaktualnej nazwy modelu,
- czy wszystkie wyrażenia odwołują się do istniejących węzłów i właściwych pól outputu.

Link do wideoprezentacji: [Watch on YouTube](https://youtu.be/I50jF8DGjIk?si=eJLon3Y5ykqFp-9C)

---

## Czego się nauczyłem

- jak sprawdzać pliki w zakładce `Binary` i znaleźć prawidłowe pole dla danych binarnych,
- że ścieżki takie jak `message.content` nie są uniwersalne i trzeba sprawdzać faktyczny output węzła,
- jak używać `Message a model` do przygotowania treści tekstowej i promptu graficznego,
- czym różnią się tryby `Fixed` i `Expression` w polach n8n,
- że odwołanie do węzła w wyrażeniu musi używać jego rzeczywistej nazwy,
- że węzeł generowania obrazu może zwracać dane binarne zamiast publicznego URL,
- jak dołączyć plik binarny do Gmaila przez **Options → Attachments** i pole `Input Data Field Name`,
- dlaczego nie warto osadzać dużego ciągu Base64 bezpośrednio w HTML wiadomości,
- że zdjęcie jest potrzebne tylko wtedy, gdy automatyzacja ma analizować wizualne cechy produktu,
- że przed publikacją workflowa trzeba przetestować całą ścieżkę od formularza do wiadomości e-mail.

# n8n – AI Product Content Generator

## Project: Product photo → AI analysis → social media post → generated graphic → email with a ready-to-use marketing package

### Lesson objective

Build a workflow that accepts a product name and photo through a form, analyzes the input using an OpenAI model, prepares social media post copy and an image-generation prompt, generates the image, and sends the result by email.

The workflow combines several elements into one process: a form, image analysis, text generation, image generation, and delivery of the final result through Gmail.

## Full workflow diagram

Main path:

```text
[n8n Form Trigger]
       │
       ▼
[OpenAI: image analysis and content preparation]
       │
       ├── social media post
       │
       └── image prompt in English
       │
       ▼
[OpenAI: Generate an image – GPT-Image-1-Mini]
       │
       ▼
[Gmail: send HTML email + graphic as an attachment]
```

During configuration, I changed the original plan to generate the image through Pollinations.ai/Flux to use the built-in OpenAI **Generate an image** node instead. This means the image is generated directly through the OpenAI API, and a separate HTTP Request node is not needed.

---

## Step 1 – n8n Form Trigger (input form)

I started by adding the **n8n Form Trigger** node, which collects the information needed to generate the marketing package.

The form contains two fields:

- **Nazwa Produktu** — type `Text`
- **Zdjecie** — type `File`

The form is the starting point of the workflow. After submission, n8n receives the product name and the image file, which will be analyzed by the AI model.

### Problem 1: binary file

In the next node's settings, **Input Data Field Name** had the default value `data`, but the form might have stored the uploaded file under a different name.

**Fix:**

1. In the left-side **INPUT** panel, I opened the `Binary` view.
2. I checked the exact name of the field containing the uploaded image.
3. In **Input Data Field Name**, I entered the name matching the binary data shown in n8n.

Lesson learned: when working with files, do not guess the field name. Check it in the `Binary` tab of the node's input data.

---

## Step 2 – OpenAI: image analysis and content preparation

The OpenAI node analyzes the submitted product photo and prepares two outputs:

1. the social media post copy,
2. a short descriptive image-generation prompt in English.

The initial plan was to return the response as JSON with these fields:

```json
{
  "post_social_media": "Social media post copy",
  "prompt_graficzny": "An English prompt describing a promotional background for the product"
}
```

During configuration, I also used the **Message a model** node with the `GPT-4.1-MINI` model. The exact output structure depends on the node's settings and version, so I need to check the actual output after running a test.

### Problem 2: the expression `{{ $json.message.content }}` was red

n8n could not find the `message.content` field in the input data. Depending on the output settings, the text might be stored in another field, such as `text`, `output`, or a nested response structure.

**Fix:**

1. I ran the previous node using **Execute step** to inspect its actual output.
2. In the **INPUT** panel, I checked the response structure in `Schema` or `JSON` view.
3. Instead of assuming the field name, I selected the actual field containing the generated text. I can drag it from the INPUT panel into the expression field so n8n inserts the correct path.

Lesson learned: a red expression often means the specified path does not exist in the current data. Check the previous node's output first.

### What does `\n\nAnaliza:` mean?

`\n` means a newline character. The sequence `\n\nAnaliza:` creates a blank line before the “Analiza:” section. It is a way to clearly separate the instructions from the content passed to the model.

---

## Step 3 – preparing data and passing the result to the next node

In the initial design, I planned to add a **Code** or **Edit Fields** node to parse the JSON returned by the model and extract the post and image prompt separately.

The general logic was to read the response text, remove any Markdown code fences, and call `JSON.parse()`.

In practice, the OpenAI output structure can vary. Before using the code, I need to check exactly where the response text is located. I should not copy the `message.content` path without confirming that this structure exists in the current workflow.

---

## Step 4 – choosing the OpenAI action

When configuring the OpenAI action list, I selected **Message a model** from the `TEXT ACTIONS` section to prepare the post copy and image description.

The model receives information about the product and instructions to create promotional copy and an image-generation prompt. Correctly selecting the input data from previous nodes was also important in this step.

### Problem 3: I did not know what to drag from the INPUT panel

I should not drag the entire node or a random part of the data. I need to find the specific property containing the required text, such as the model response or the product name.

**How to do it:**

1. Expand the relevant node in the INPUT panel.
2. Find the field containing the required value.
3. Drag that field into the appropriate place in the expression editor.
4. Check the result preview and make sure it displays the expected text.

### Problem 4: an incorrect node name in an expression

The error `Referenced node doesn't exist` appeared. The expression referred to a node named `n8n Form Trigger`, but the actual node had a different name in the workflow.

**Fix:** use the node's actual name from the canvas, or drag the field from the INPUT panel so n8n inserts the correct reference.

Lesson learned: the node name used in an expression must match the actual node name in the workflow.

---

## Step 5 – generating the image with OpenAI

I used the OpenAI **Generate an image** node with the `GPT-Image-1-Mini` model to generate the image.

The image-generation prompt should come from the previous step, where the model prepares an English visual description.

After switching to OpenAI image generation, I no longer need a separate HTTP Request node for Pollinations.ai/Flux. The generated image is returned as binary data, not necessarily as a public URL.

### Problem 5: questions about cost

The workflow contains separate AI operations: image analysis and image generation. Their costs are not the same as the cost of a single short text request. I should check OpenAI's current pricing for the model being used and monitor usage in the API dashboard.

Lesson learned: “mini” does not mean the entire automation is free. Cost depends on the models used, number of calls, amount of data processed, and image generation.

---

## Step 6 – Gmail: sending the finished marketing package

At the end of the workflow, the **Gmail** node sends an HTML email containing the product name, post copy, and information about the generated graphic.

I set the **Message** field to **Expression** mode because the email contains dynamic data from previous nodes.

### Problem 6: `Fixed` or `Expression`?

I chose **Expression** for the HTML content because the message contains values pulled from previous nodes, such as the product name and post copy.

- `Fixed` — ordinary static text.
- `Expression` — text where n8n evaluates expressions and inserts the current data.

### Problem 7: the image did not appear inside the email

Using `json.url` did not work because the **Generate an image** node returned a binary file, not a field containing a URL. Embedding Base64 data directly in the HTML was not a good solution for this email either.

**The solution that worked:** send the image as an email attachment.

1. I removed the incorrect `<img ... />` tag from the HTML.
2. In the Gmail settings, under **Options**, I added **Attachments**.
3. In **Input Data Field Name**, I entered `data`, matching the binary field name shown in the INPUT panel.
4. I ran the node using **Execute step**.

Result: the email was sent, and the graphic arrived as an attachment.

### Final version of the HTML email

The email displays the product name and post copy, and tells the recipient that the graphic is attached. It does not contain an incorrect image URL or embedded Base64 data.

```html
<div style="font-family: Arial, sans-serif; max-width: 600px; margin: 0 auto; border: 1px solid #e0e0e0; border-radius: 8px; padding: 20px; background-color: #ffffff;">
  <h2 style="color: #1a73e8; margin-top: 0;">A new marketing package has been generated</h2>

  <p><strong>Product:</strong> {{ $('On form submission').item.json['Nazwa Produktu'] }}</p>

  <hr style="border: none; border-top: 1px solid #eee; margin: 15px 0;" />

  <h3 style="color: #333;">Generated social media post</h3>
  <div style="background-color: #f9f9f9; padding: 15px; border-left: 4px solid #1a73e8; border-radius: 4px; white-space: pre-wrap; font-size: 14px; line-height: 1.6; color: #2c3e50;">
{{ $('Message a model').item.json.output[0].content[0].text }}
  </div>

  <hr style="border: none; border-top: 1px solid #eee; margin: 20px 0;" />

  <h3 style="color: #333;">Generated promotional graphic</h3>
  <div style="background-color: #e8f0fe; padding: 12px 15px; border-radius: 6px; color: #174ea6; font-size: 13px;">
    The generated promotional graphic is attached to this email as a PNG file.
  </div>

  <p style="font-size: 12px; color: #888; text-align: center; margin-top: 25px;">
    Automation built with n8n + OpenAI API
  </p>
</div>
```

Practical note: the names `On form submission` and `Message a model`, as well as the path `output[0].content[0].text`, must match the actual node names and output structure in this instance. If the nodes have different names or the response format changes, inspect the input data and update the expressions.

---

## Why does the workflow take a photo as input?

The photo is the input for multimodal analysis. The model can identify visible product characteristics and use them to create more relevant post copy and an image prompt.

The form should receive a photo of the product being promoted, for example a photo taken with a phone or a product photograph. The product name also helps clarify what the material is about.

This approach makes sense when the goal is to create marketing material based on a specific product image. If the post should be generated only from a name, text description, or brief, a photo is not essential and the form can later be simplified.

---

## Final test

After configuring the workflow, I submitted a test through the form. Gmail sent the message successfully, and the graphic arrived as an attachment.

When checking the result, I should verify:

- whether the product name in the email is correct,
- whether the post matches the actual product and does not invent details,
- whether the generated graphic is attached,
- whether the email contains no outdated model name,
- whether all expressions refer to existing nodes and the correct output fields.

Video Presentation: [Watch on YouTube](https://youtu.be/I50jF8DGjIk?si=eJLon3Y5ykqFp-9C)

---

## What I learned

- how to inspect files in the `Binary` tab and find the correct binary-data field,
- that paths such as `message.content` are not universal and the actual node output must be checked,
- how to use `Message a model` to prepare text and an image prompt,
- the difference between `Fixed` and `Expression` modes in n8n fields,
- that an expression's node reference must use the actual node name,
- that an image-generation node may return binary data instead of a public URL,
- how to attach a binary file to Gmail using **Options → Attachments** and the **Input Data Field Name** field,
- why it is better not to embed a large Base64 string directly into email HTML,
- that a photo is only needed when the automation should analyze the product's visual characteristics,
- that the entire path from the form to the email should be tested before the workflow is published.
