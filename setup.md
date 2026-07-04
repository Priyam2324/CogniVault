# CogniVault - Local Setup Guide

CogniVault is a FastAPI backend (`main.py`) + a single-file HTML/JS frontend
(`index.html`) that lets you upload documents (PDF, images, Word docs),
ask questions about them, and remember context across conversations via `cognee`.

This guide covers everything needed to run it on a fresh machine.

---

## 1. Prerequisites

| Requirement | Notes |
|---|---|
| Python 3.10+ | 3.11 recommended |
| pip | comes with Python |
| Tesseract OCR (system binary) | **Not** a pip package - `pytesseract` is just a Python wrapper around it. Without this installed, image uploads will fail to extract text. |
| An LLM API key | From whichever provider you configure (Groq, OpenAI, Anthropic, etc. - anything [litellm supports](https://docs.litellm.ai/docs/providers)) |
| A modern browser | To open `index.html` |

### Installing Tesseract (system dependency, not pip-installable)

- **Windows**: Download the installer from https://github.com/UB-Mannheim/tesseract/wiki and install it. Note the install path (e.g. `C:\Program Files\Tesseract-OCR\tesseract.exe`) - you may need to add it to your system `PATH`, or point `pytesseract.pytesseract.tesseract_cmd` at it if it's not auto-detected.
- **macOS**: `brew install tesseract`
- **Linux (Debian/Ubuntu)**: `sudo apt-get install tesseract-ocr`

Verify it's on your PATH with:
```bash
tesseract --version
```

---

## 2. Get the code

`git clone` creates the project folder for you - you don't need to make one
first. Just `cd` into whichever *parent* directory you want the project to
live under (e.g. `cd ~/projects`), then clone:

```bash
cd ~/projects                              # optional: cd into wherever you want it cloned
git clone https://github.com/Priyam2324/CogniVault.git
cd CogniVault
```

---

## 3. Create a virtual environment

```bash
python -m venv venv

# Activate it:
# Windows
venv\Scripts\activate
# macOS/Linux
source venv/bin/activate
```

---

## 4. Install Python dependencies

```bash
pip install -r requirements.txt
```

---

## 5. Configure your `.env` file

Copy the template and edit it:

```bash
cp .env.example .env
```

Open `.env` and fill in every field marked `# CHANGE ME`:

1. **`LLM_MODEL` / `LLM_API_KEY`** - Pick a provider/model supported by `litellm`
   (e.g. `groq/llama-3.3-70b-versatile`, `openai/gpt-4o-mini`) and put your
   *own* API key from that provider. Get a fresh key from your provider's
   dashboard - don't reuse a key that's ever been shared, committed, or pasted
   anywhere public.
2. **`LLM_ENDPOINT`** (optional) - Only needed if you're using a self-hosted or
   custom OpenAI-compatible endpoint. Leave commented out otherwise.
3. **`DATA_ROOT_DIRECTORY` / `SYSTEM_ROOT_DIRECTORY`** - Absolute paths on
   *your* machine where `cognee` will store its data/graph/vector files.
   These folders don't need to pre-exist with content, but the parent path
   should be writable. Example:
   - Windows: `C:/Users/<you>/Documents/CogniVault/data`
   - macOS/Linux: `/home/<you>/cognivault/data`

Everything else in `.env.example` (embedding provider, DB provider, infra
flags) has sane defaults for local/dev use and typically doesn't need
changing.

**Never commit your real `.env` to git.** Add it to `.gitignore`:
```bash
echo ".env" >> .gitignore
```

---

## 6. Run the backend

From the project root (with the venv activated):

```bash
python main.py
```

This starts the FastAPI server on `http://127.0.0.1:8000` (via `uvicorn`,
with `reload=True` for auto-restart on code changes).

On first run, `main.py` will automatically create:
- `chat_history.db` (SQLite, in the project root)
- `uploads/` (folder for uploaded file storage)
- the `cognee_data` / data-root folder you configured in `.env`

---

## 7. Open the frontend

Just open `index.html` directly in your browser (double-click it, or
`open index.html` / `start index.html`).

The frontend is hardcoded to talk to the backend at:
```js
const API_URL = "http://127.0.0.1:8000/api";
```
in `index.html`. If you ever run the backend on a different host/port
(e.g. deploying it, or changing `uvicorn.run(...)` port in `main.py`), update
that constant in `index.html` to match.

---

## 8. Quick smoke test

1. Backend running (`python main.py`) with no errors in the terminal.
2. Open `index.html` - you should see the CogniVault welcome screen and an
   empty chat created automatically.
3. Type a message and send it - you should get a reply.
4. Attach a PDF/image/docx and send - you should get a summary back, and the
   file should appear as an attachment bubble you can click to open.

---

## 9. Common issues

| Symptom | Likely cause |
|---|---|
| `RuntimeError: LLM_MODEL environment variable is not set` | `.env` wasn't loaded or is missing that field - confirm `.env` exists in the same folder as `main.py` |
| OCR returns empty text on images | Tesseract binary isn't installed or isn't on PATH (see step 1) |
| `database is locked` errors | Should be handled already (WAL mode + busy_timeout), but if you still see it, make sure nothing else has `chat_history.db` open (e.g. a DB browser tool) |
| Frontend shows "Could not reach the server" | Backend isn't running, or is running on a different port than `API_URL` in `index.html` points to |
| CORS errors in browser console | Backend allows all origins by default (`allow_origins=["*"]`) - if you changed that, add your frontend's origin back in |

---

## 10. Project structure recap

```
.
├── main.py              # FastAPI backend
├── index.html           # Single-file frontend (HTML/CSS/JS)
├── requirements.txt      # Python dependencies
├── .env.example          # Environment variable template (copy to .env)
├── .env                  # Your real config (gitignored, never commit)
├── chat_history.db        # Created automatically on first run
└── uploads/               # Created automatically on first run
```
