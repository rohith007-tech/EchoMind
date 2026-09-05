<div align="center">

# 🎙️ EchoMind

**Point it at a video. Get the meeting back.**

EchoMind turns any YouTube video (or local recording) into a searchable, question-answerable knowledge base — automatically transcribed, summarized, and broken into action items, decisions, and open questions, with a RAG-powered chat interface to ask it anything the recording said.

[![Python](https://img.shields.io/badge/Python-3.10%2B-3776AB?logo=python&logoColor=white)](https://www.python.org/)
[![LangChain](https://img.shields.io/badge/LangChain-LCEL-1C3C3C?logo=langchain&logoColor=white)](https://www.langchain.com/)
[![Whisper](https://img.shields.io/badge/OpenAI-Whisper-412991?logo=openai&logoColor=white)](https://github.com/openai/whisper)
[![ChromaDB](https://img.shields.io/badge/ChromaDB-Vector%20Store-FF6F61)](https://www.trychroma.com/)
</div>

---

## What it does

Give EchoMind a YouTube link. It will:

1. **Download & chunk** the audio (`yt-dlp` + `pydub`)
2. **Transcribe** it — locally via Whisper (English) or via the Sarvam AI speech API (Hinglish)
3. **Generate a title** and a **structured summary** of the recording
4. **Extract** action items, key decisions, and open questions as clean, formatted lists
5. **Embed and index** the transcript into a Chroma vector store
6. **Let you ask it questions** — a LangChain RAG pipeline answers strictly from the transcript, with source-grounded responses

It's usable as a **\*\*CLI\*\*** (`main.py`) for processing videos and asking questions about the recording through the command line.

---

## Demo flow

```
  YouTube URL
       │
       ▼
 ┌─────────────┐    ┌──────────────┐    ┌─────────────┐
 │  Download    │ →  │ Transcribe   │ →  │ Summarize +  │
 │  & chunk     │    │ (Whisper /   │    │ extract      │
 │  audio       │    │  Sarvam AI)  │    │ (Mistral AI) │
 └─────────────┘    └──────────────┘    └─────────────┘
                                                │
                                                ▼
                                       ┌─────────────────┐
                                       │ Embed + index    │
                                       │ (ChromaDB)        │
                                       └─────────────────┘
                                                │
                                                ▼
                                       💬 Ask it anything
```

---

## Tech stack

| Layer | Tech |
|---|---|
| Orchestration | [LangChain](https://www.langchain.com/) (LCEL pipelines) |
| LLM | [Mistral AI](https://mistral.ai/) (`mistral-small-latest`) |
| Transcription | [OpenAI Whisper](https://github.com/openai/whisper) (local) · [Sarvam AI](https://www.sarvam.ai/) (Hinglish) |
| Vector store | [ChromaDB](https://www.trychroma.com/) + HuggingFace `all-MiniLM-L6-v2` embeddings |
| Audio pipeline | `yt-dlp`, `pydub`, `ffmpeg` |

---

## Project structure

```
EchoMind/
├── core/
│   ├── transcriber.py      # Whisper / Sarvam AI transcription
│   ├── summarize.py        # title + summary generation
│   ├── extractor.py        # action items / decisions / questions
│   ├── rag.py               # RAG chain: build + query
│   └── vector_store.py     # Chroma embedding + retrieval
├── utils/
│   └── audio_processor.py  # download, convert, chunk audio
├── main.py                  # CLI entry point
└── requirements.txt
```

---

## Getting started

### Prerequisites

- Python 3.10+
- `ffmpeg` installed and on your PATH
- A [Mistral AI](https://mistral.ai/) API key
- A [Sarvam AI](https://www.sarvam.ai/) API key *(optional — only needed for Hinglish transcription)*

### 1. Clone & install

```bash
git clone https://github.com/rohith007-tech/EchoMind.git
cd EchoMind
pip install -r requirements.txt
```

### 2. Configure environment variables

Create a `.env` file in the project root:

```env
MISTRAL_API_KEY=your_mistral_key_here
SARVAM_API_KEY=your_sarvam_key_here
WHISPER_MODEL=small
```

### 3. Run it

**As a CLI:**
```bash
python main.py
```

---


## Roadmap

- [ ] Multi-file / batch processing
- [ ] Export summary + action items to PDF
- [ ] Speaker diarization
- [ ] Persistent job history (currently in-memory, per session)

---

<div align="center">
Built by <a href="https://github.com/rohith007-tech">Rohith</a>
</div>
