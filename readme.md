<div align="center">

<br/>

# Docima

**AI-powered document extraction — fully offline, zero API costs.**

<br/>

![React](https://img.shields.io/badge/React-20232A?style=for-the-badge&logo=react&logoColor=61DAFB)
![FastAPI](https://img.shields.io/badge/FastAPI-005571?style=for-the-badge&logo=fastapi)
![Python](https://img.shields.io/badge/Python_3.10-3670A0?style=for-the-badge&logo=python&logoColor=ffdd54)
![PaddleOCR](https://img.shields.io/badge/PaddleOCR-0062B0?style=for-the-badge&logo=paddlepaddle&logoColor=white)
![SQLite](https://img.shields.io/badge/SQLite-07405E?style=for-the-badge&logo=sqlite&logoColor=white)

<br/>

[What it does](#what-it-does) · [Pipeline](#pipeline) · [Getting Started](#getting-started) · [Configuration](#configuration) · [Roadmap](#roadmap)

</div>

---

## What it does

Docima ingests invoices, receipts, and forms, runs them through a local OCR + LayoutLM pipeline, and returns structured JSON. High-confidence extractions are auto-accepted. Anything below threshold goes to a human review queue.

No cloud. No API keys. Everything runs on your machine.

---

## Features

| | |
|---|---|
| **Batch ingestion** | Drag-and-drop single files or bulk upload; parallel processing via async background tasks |
| **OCR pipeline** | PaddleOCR with deskewing, CLAHE, adaptive binarization, and column-aware token sorting |
| **Layout understanding** | LayoutLMv3 classifies tokens into labels (field, value, date, amount, address, table cell) |
| **Confidence scoring** | Composite score across OCR confidence, layout label, and spatial proximity |
| **Human review queue** | Side-by-side document + extracted fields view for anything flagged below threshold |
| **Export** | CSV, Excel, or JSON with date filters and export history |
| **Analytics** | Dashboard with volume trends, auto-accept rates, and processing times |
| **Fully local** | SQLite database, no Redis, no external services required |

---

## Pipeline

Each document passes through six stages:

```
Upload
  │
  ▼
ImagePreprocessor     DPI check → grayscale → deskew → denoise → binarization → CLAHE
  │
  ▼
OCREngine             PaddleOCR → filter noise → column-aware sort → merge tokens → full_text
  │
  ▼
LLM Extractor         Ollama (llama3.2) → extract structured fields from OCR text → key-value JSON
  │
  ▼
KeyValueExtractor     Direction-aware spatial pairing → regex fallback → fuzzy schema mapping
  │
  ▼
ConfidenceScorer      40% OCR + 40% layout + 20% spatial → format/consistency bonuses
  │
  ├─ ≥ 0.90 → Auto-accepted
  ├─ ≥ 0.55 → Needs review
  └─ < 0.55 → Failed
  │
  ▼
PostProcessor         ISO 8601 dates, float amounts, SWIFT/IFSC/VAT validation, cross-validation
  │
  ▼
Export (CSV / JSON / Excel)
```

---

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | React + Vite |
| Backend | FastAPI + Uvicorn |
| Database | SQLite via SQLAlchemy ORM |
| OCR | PaddleOCR (CPU/GPU) |
| LLM | Ollama — llama3.2 (local inference) |
| Task queue | Python `asyncio` background tasks |
| Desktop shell | PyWebView (WebView2) |
| Packaging | PyInstaller + Inno Setup |

---

## Getting Started

### Prerequisites

- Python 3.10+
- Node.js 18+
- Git

> **GPU note (RTX 3050 / 4 GB VRAM):** Install `nvidia-cudnn-cu11==8.9.4.25` and `numpy<2` to avoid PaddleOCR DLL and ABI issues on Windows. The backend injects cuDNN paths at runtime automatically.

### 1. Clone

```bash
git clone https://github.com/spidysan/docima.git
cd docima
```

### 2. Backend

```bash
cd docima-backend

python -m venv venv
# Windows:
venv\Scripts\activate
# Linux/macOS:
source venv/bin/activate

pip install -r requirements.txt

cp .env.example .env
```

Edit `.env`:

```env
UPLOAD_DIR=./uploads
AUTO_ACCEPT_THRESHOLD=0.90
REVIEW_THRESHOLD=0.55
LOG_LEVEL=INFO
```

Start the server:

```bash
uvicorn main:app --reload --port 8000
```

### 3. Frontend

```bash
cd frontend
npm install
npm run dev
```

Open [http://localhost:5173](http://localhost:5173).

### 4. Desktop app (Windows only)

```bash
python desktop.py
```

This opens Docima in a native window via PyWebView. No browser needed.

---

## Configuration

All tunable settings are in `config.py` and can be overridden via `.env`:

| Variable | Default | Description |
|---|---|---|
| `AUTO_ACCEPT_THRESHOLD` | `0.90` | Confidence above which fields are auto-accepted |
| `REVIEW_THRESHOLD` | `0.55` | Confidence below which fields are marked failed |
| `MAX_UPLOAD_SIZE_MB` | `50` | Max file size per upload |
| `BATCH_SIZE_LIMIT` | `100` | Max documents per batch |
| `LOG_LEVEL` | `INFO` | Logging verbosity |

---

## Project Structure

```
docima/
├── docima-backend/
│   ├── pipeline/
│   │   ├── preprocessor.py       # Image prep
│   │   ├── ocr.py                # PaddleOCR wrapper
│   │   ├── layout_model.py       # LayoutLMv3 classification
│   │   ├── extractor.py          # Key-value pairing
│   │   ├── postprocessor.py      # Validation + normalization
│   │   └── confidence.py         # Scoring + thresholds
│   ├── tasks/
│   │   └── extraction_task.py    # Pipeline orchestration
│   ├── tests/
│   ├── main.py
│   ├── config.py
│   ├── database.py
│   ├── models/
│   ├── schemas/
│   └── utils/
├── frontend/
│   ├── src/
│   │   ├── pages/                # Dashboard, Upload, Review, Export
│   │   └── components/
│   └── package.json
├── desktop.py
├── launcher.py
├── Docima.iss
└── docker-compose.yml
```

---

## Building the Windows Installer

```bash
# 1. Build the frontend
cd frontend && npm run build

# 2. Compile the launcher (only if launcher.py changed)
.\venv\Scripts\pyinstaller.exe --name Docima --onefile --windowed `
  --add-data "frontend\public\favicon.ico;." `
  --icon frontend\public\favicon.ico launcher.py

# 3. Build the installer
& "$env:LOCALAPPDATA\Programs\Inno Setup 6\ISCC.exe" Docima.iss
```

Output: `Output\Docima_Setup.exe` + `Output\Docima_Setup-1.bin` (~430 MB compressed from ~3 GB).

---

## Roadmap

- [ ] Multi-language OCR support
- [ ] Table extraction (line-item level)
- [ ] Webhook integrations (Zapier, Make)
- [ ] Active learning from human corrections
- [ ] Docker Compose one-command setup
- [ ] Role-based access control

---

## License

[MIT](LICENSE)

---

<div align="center">
  <sub>Built by <a href="https://github.com/spidysan">Santhosh</a></sub>
</div>
