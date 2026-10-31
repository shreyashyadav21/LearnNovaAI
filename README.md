# LearnNova AI

LearnNova AI is a Streamlit study assistant for PDF notes. It uses Groq for
chat generation, deterministic local feature vectors for document search, and
ChromaDB to store and retrieve document passages. It does not require a
locally running Ollama server or a separate embedding API key.

## Features

- Upload multiple PDFs and choose an active document.
- Extract selectable PDF text with `pypdf`; optionally OCR scanned pages.
- Ask source-grounded questions, including follow-ups, with streamed answers.
- Search the document through local feature hashing and ChromaDB vector
  retrieval.
- Generate and download a fast extractive PDF summary.
- Create MCQs, short- and long-answer questions, passage-based and mixed
  questions, revision notes, and flashcards from source statements.
- Attempt interactive MCQ quizzes and view the score and explanations.

## Requirements

- Python 3.10 or later
- A [Groq API key](https://console.groq.com/keys)
- The Python packages in [`requirements.txt`](requirements.txt)

Groq generates chat answers. PDF and query vectors are created locally with a
deterministic feature-hashing vectorizer. Only chat prompts and retrieved
passages are sent to Groq.

## Run locally on Windows

Open PowerShell in the project folder:

```powershell
cd "C:\path\to\StudyMateAI"
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
```

Set the API keys in the active terminal. Replace the placeholders with your
own keys; never put real keys in source code or commit them:

```powershell
$env:GROQ_API_KEY = "your-groq-api-key"
```

Run the app:

```powershell
python -m streamlit run app.py
```

Open the local URL printed by Streamlit, usually
`http://localhost:8501`. Upload a PDF and wait for its first-time indexing to
complete.

### Using Streamlit secrets instead

Create `.streamlit/secrets.toml` locally:

```toml
GROQ_API_KEY = "your-groq-api-key"
```

Keep this file private and out of version control. On Render or another host,
add both values through the service's **Environment** settings instead of
committing a secrets file.

## Deploy on Render

1. Push the project to a Git repository and create a Render **Web Service**
   from it.
2. Select the Python runtime and set the build command:

   ```text
   pip install -r requirements.txt
   ```

3. Set the start command:

   ```text
   streamlit run app.py --server.address 0.0.0.0 --server.port $PORT
   ```

4. Add these environment variables in the Render dashboard:

   ```text
   GROQ_API_KEY     = your Groq API key
   ```

5. Deploy and open the Render URL. No Ollama service or `OLLAMA_HOST`
   configuration is needed.

ChromaDB's `studymate_db` directory is local to the service. Render's
filesystem is ephemeral by default, so document indexes may need to be rebuilt
after a restart or redeployment. If persistent local storage is available for
your service plan, mount a disk and configure the database path to live under
that mount before relying on indexes across deployments.

## Workflow: what happens to a PDF

1. **Upload:** The Streamlit UI reads the uploaded PDF bytes and identifies
   the document.
2. **Extract:** `pypdf` extracts selectable text page by page and the app
   normalizes it. Optional OCR can process pages without selectable text.
3. **Chunk:** Text is split into overlapping word chunks so adjacent chunks
   retain context across boundaries.
4. **Vectorize:** The app hashes normalized words and word pairs into
   deterministic, normalized 768-dimensional vectors locally. No embedding
   API key or network request is needed.
5. **Store:** ChromaDB persists the vectors in a collection for the selected
   document. The collection name includes the PDF hash and embedding-version
   suffix.
6. **Ask:** The question is vectorized with the same local feature hasher.
   ChromaDB retrieves similar chunks, which are lightly re-ranked by matching
   topic terms. Feature hashing is fast and private, but less semantically
   capable than a trained embedding model.
7. **Answer:** The question, recent conversation history, and retrieved notes
   are sent to Groq's chat API. Groq streams its response into the UI.

The active app lives in [`app.py`](app.py). The summary and exam-material
generators use extracted text directly; they do not call Groq. The summary is
extractive, not a model-written summary. Exam generators report when the
source has too few statements for the requested topic or count rather than
filling gaps with unsupported claims.

## Configuration and index compatibility

The current model settings in `app.py` are:

```python
CHAT_MODEL = "openai/gpt-oss-20b"
EMBEDDING_DIMENSIONS = 768
```

The `GROQ_API_KEY` is read from the environment, with Streamlit secrets as a
fallback. It is needed for chat answers and AI-generated flashcards. PDF
indexing, summaries, revision notes, MCQs, and other locally generated exam
formats do not require a provider key.

The app uses a new ChromaDB collection suffix, `local_hash_v1`, because local
feature vectors cannot be mixed with the older Ollama or OpenAI embeddings.
Existing collections are left in place but are not used by the current app.
Upload the PDF again if its new collection has not been indexed yet.

## Optional OCR for scanned PDFs

Text-based PDFs do not need OCR. The OCR Python packages are included in
`requirements.txt`. Install them in the same environment used to run Streamlit:

```powershell
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
```

Install Tesseract OCR separately. The app currently expects
`C:\Program Files\Tesseract-OCR\tesseract.exe`. If it is installed elsewhere,
update `pytesseract.pytesseract.tesseract_cmd` near the top of
[`app.py`](app.py). OCR may be slower and may miss text in poor-quality scans,
tables, or complex page layouts.

## Project files

| File or folder | Purpose |
| --- | --- |
| `app.py` | Main Streamlit application and active entrypoint. |
| `requirements.txt` | Python dependencies for the website. |
| `studymate_db/` | Local persistent ChromaDB indexes. |
| `LearnNova_AI_app.py` | Earlier alternate Streamlit app; not the current entrypoint. |
| `read_pdf.py`, `chunk_pdf.py` | Command-line PDF reading and chunking experiments. |
| `embedding_test.py`, `embed_pdf.py` | Earlier embedding experiments. |
| `search_pdf.py`, `search_chroma.py`, `store_pdf.py` | Retrieval and vector-store experiments. |
| `rag_chat.py`, `rag_chroma.py` | Earlier command-line chat prototypes. |
| `app_backup.py`, `app.zip` | Backup/archive files; not required to run the current app. |

Some experiments are older and may still import PyMuPDF or Ollama directly;
they are separate from the website and are not needed to run `app.py`.

## Troubleshooting

### Missing `GROQ_API_KEY`

Set the key in the environment where Streamlit runs. For Render, add it in
the web service's **Environment** settings and redeploy. Never paste API keys
into chat, source code, or a public repository.

### Invalid key, quota, or network error

Check that each key belongs to the correct provider, is active, and has API
access/billing available as required by that provider. Check the Render logs
for the provider's error response.

### PDF has no readable text

Image-only PDFs need the optional OCR components described above. Otherwise,
use a PDF with selectable text. The app also shows extraction status and a
preview when text is available.

### PowerShell will not activate `.venv`

Use the virtual environment's Python directly:

```powershell
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
.\.venv\Scripts\python.exe -m streamlit run app.py
```

## Privacy and limitations

- Chat prompts and selected PDF passages are sent to Groq to generate
  answers. Document embeddings are created locally and are not sent to an
  embedding provider.
- ChromaDB stores the indexed chunks locally in `studymate_db`; protect this
  directory if it contains sensitive material.
- The quality of answers depends on the PDF extraction and retrieved
  passages. Check important information against the original document.
- OCR may miss content in poor scans, images, tables, and complex layouts.
- This project is a study aid, not a substitute for course materials or
  instructor guidance.
