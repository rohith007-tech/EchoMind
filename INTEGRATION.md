# Wiring the frontend up to EchoMind

Your `core/` and `utils/` code is **untouched**. Two new folders sit next to it:

```
EchoMind/
├── core/                     (yours, unchanged)
├── utils/                    (yours, unchanged)
├── main.py                   (yours, unchanged — still works as a CLI)
├── requirements.txt          (yours, unchanged)
├── backend/                  ← NEW
│   ├── __init__.py
│   ├── app.py                 FastAPI server (HTTP + WebSocket)
│   ├── pipeline_runner.py     wraps main.py's pipeline with progress events
│   └── requirements-backend.txt
└── frontend/                 ← NEW
    ├── index.html
    └── static/
        ├── style.css
        └── app.js
```

## 1. Install the two extra packages

From your project root, with your existing virtualenv active:

```bash
pip install -r backend/requirements-backend.txt
```

That's `fastapi` and `uvicorn` — nothing else changes in your dependency tree.

## 2. Run it

Still from the project root (the folder that contains `core/`, `utils/`, `backend/`, `frontend/`):

```bash
uvicorn backend.app:app --reload --port 8000
```

Open **http://localhost:8000** — the backend serves the frontend too, so this is the only command you need. `main.py` still works exactly as before if you want the CLI.

## 3. How it fits together

**`backend/pipeline_runner.py`** replicates `main.py`'s `run_pipeline()` step by step, but instead of `print()`-ing progress, it calls an `on_update(stage, message, progress)` callback before each stage:

```
downloading → transcribing → titling → summarizing → actions → decisions → questions → indexing → ready
```

It imports your existing functions directly — `process_input`, `transcribe_all`, `summarize`, `generate_title`, the three `extract_*` functions, and `build_rag_chain` / `ask_question` from `core/rag.py`. If you rename or add stages in `core/`, you only need to edit this one file.

**`backend/app.py`** is the HTTP layer:

| Endpoint | What it does |
|---|---|
| `GET /api/stages` | Returns the pipeline stage list, so the frontend never hardcodes it |
| `POST /api/jobs` | Body `{source, language}` → kicks off the pipeline on a background thread, returns `{job_id}` immediately |
| `GET /api/jobs/{id}` | Poll a job's current status/result (fallback if WebSockets are blocked) |
| `WS /ws/jobs/{id}` | Live status stream the frontend actually uses — pushes `{status, message, progress, result, error}` roughly every 0.35s while a job runs |
| `POST /api/jobs/{id}/chat` | Body `{question}` → runs your `ask_question(rag_chain, question)` against that job's index |

Because your pipeline is blocking (Whisper, Mistral calls, embedding), each job runs via `loop.run_in_executor(None, ...)` — a background thread — so the event loop stays free to keep pushing WebSocket updates while the heavy work happens. The `rag_chain` object itself never leaves the server (it's not JSON-serializable anyway); it's kept in an in-memory dict keyed by `job_id` and only touched again when that job's chat endpoint is called.

**`frontend/app.js`** does three things: POSTs the URL to create a job, opens a WebSocket to animate the stage tracker + sonar visual live, then once `status === "ready"` it renders the results and unlocks the chat dock, which just calls the chat endpoint per message.

## 4. Things worth doing before you ship this anywhere public

- **CORS**: `backend/app.py` currently allows `allow_origins=["*"]`. Fine for local dev/demo; lock it to your actual frontend origin before deploying.
- **Job store**: jobs live in an in-memory Python dict (`JOBS`, `CHAINS`). That's perfect for a single-process demo (great for a resume project / portfolio link) but resets on server restart and won't work across multiple worker processes. If you deploy with `--workers > 1`, switch to Redis or similar.
- **Cleanup**: `utils/audio_processor.py` downloads audio into `downloades/` and chunks into `*_chunk_*.wav` files that never get deleted. Not a problem for a demo, but worth an `os.remove()` pass after transcription if you're running this for real traffic.
- **Concurrent jobs**: nothing currently limits how many pipelines can run at once — each hits Whisper (CPU/GPU) and the Mistral API. Fine for a personal demo; add a simple queue/semaphore if multiple people might use it simultaneously.
- **Auth**: there isn't any. Anyone who can reach the server can submit jobs and rack up Mistral API calls. Add an API key check on `/api/jobs` if you put this somewhere public.

## 5. If something doesn't work

- **Blank page / 404 on `/`**: you're not running `uvicorn` from the project root, so it can't find `frontend/index.html`. `cd` into the folder that contains `core/`, `backend/`, `frontend/` and re-run.
- **`ModuleNotFoundError: No module named 'core'`**: same cause — Python resolves `core`/`utils`/`backend` relative to your current working directory when you launch uvicorn.
- **WebSocket connects then immediately errors**: check the browser console — likely a CORS or proxy issue if you're running the frontend from a different origin than the backend. Simplest fix: don't — let FastAPI serve both, as set up here.
- **A job gets stuck at "downloading"**: that's `yt_dlp` actually working through the video — large videos can take a while. Watch your terminal running `uvicorn`; your existing `print()` statements in `core/`/`utils/` still fire there.
