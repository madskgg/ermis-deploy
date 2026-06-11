# ERMIS — Cyber Threat Intelligence Platform

ERMIS is a self-hosted Cyber Threat Intelligence (CTI) platform that automatically collects, enriches, and surfaces threat data from open-source security research. It scrapes articles from curated security blogs and GitHub-hosted CTI reports, extracts Indicators of Compromise (IoCs) and named entities with fine-tuned NLP models, maps adversary behaviour to the MITRE ATT&CK framework, builds a knowledge graph of entity relationships, and exports the resulting intelligence to MISP, OpenCTI, and STIX 2.1 bundles. A web portal provides analysts with article browsing, full-text search, graph exploration, on-demand file/URL analysis, and user/role management.

This repository (`ermis-deploy`) is the orchestration layer: it contains the Docker Compose configuration, the shared base image, and the Nginx reverse-proxy configuration that tie the other ERMIS repositories together into a running stack.

---

## Repositories

ERMIS is split across four repositories that must be cloned **as siblings** in the same parent directory:

| Repository | Purpose |
|---|---|
| [`ermis-deploy`](https://github.com/madskgg/ermis-deploy) | Docker Compose stack, base image, Nginx config (this repo) |
| [`ermis-common`](https://github.com/madskgg/ermis-common) | Shared Python library — DB models, IOC/NER extraction, ML helpers |
| [`ermis-ingestion`](https://github.com/madskgg/ermis-ingestion) | Scraping & enrichment pipeline |
| [`ermis-portal`](https://github.com/madskgg/ermis-portal) | Django REST backend + Celery workers + React frontend |

Expected layout after cloning:

```
ERMIS/
├── ermis-deploy/      ← you are here (docker-compose.yml lives here)
├── ermis-common/
├── ermis-ingestion/
└── ermis-portal/
```

---

## Architecture

```
ERMIS/
├── ermis-deploy/        # Docker Compose, base image, Nginx config
├── ermis-common/        # Shared Python library (DB models, ML helpers, IOC/NER extraction)
├── ermis-ingestion/      # Scraping & enrichment pipeline (run on demand)
└── ermis-portal/
    ├── backend/         # Django REST API + Celery workers
    └── frontend/        # React / TypeScript SPA (Vite)
```

**Runtime services** (defined in `ermis-deploy/docker-compose.yml`)

| Service | Purpose |
|---|---|
| `backend` | Django REST API (port 8000, internal) |
| `celery_worker` | Async ML processing (NER, IOC extraction, ATT&CK mapping, relation extraction) |
| `frontend` | React SPA (port 5173, internal) |
| `nginx` | Reverse proxy (exposed on **port 8081**) |
| `redis` | Celery broker, result backend, and Django cache |
| `postgresql` | Primary relational store for articles, IoCs, entities |
| `neo4j` | Graph store for entity relationships |
| `base` | Shared base image (Python deps + ML model packages) used by `backend`, `celery_worker`, and `ingestion` |
| `ingestion` | On-demand scraping & enrichment container (`--profile tools`) |

---

## Machine Learning Models

ERMIS uses a pipeline of locally hosted, fine-tuned transformer models (loaded via HuggingFace `transformers`) plus one cloud LLM:

| Model | Task | Notes |
|---|---|---|
| [`attack-vector/SecureModernBERT-NER`](https://huggingface.co/attack-vector/SecureModernBERT-NER) | Contextual named-entity recognition (malware, threat actors, tools, campaigns, sectors, locations, etc.) | ModernBERT-large, 22 BIO labels, run with `aggregation_strategy="first"` over 120-token chunks |
| `ibm-research/CTI-BERT` (fine-tuned locally → `model_cti_bert_custom`) | Binary CTI-relevance classification gate | BERT-base, fine-tuned on a curated ERMIS corpus; 5-fold CV accuracy ≈ 97.8% |
| [`sarahwei/MITRE-v16-tactic-bert-case-based`](https://huggingface.co/sarahwei/MITRE-v16-tactic-bert-case-based) | MITRE ATT&CK tactic classification | `AutoModelForSequenceClassification` |
| [`nanda-rani/TTPXHunter`](https://huggingface.co/nanda-rani/TTPXHunter) | MITRE ATT&CK technique (TTP) extraction | RoBERTa-based sequence classifier |
| Google Gemini API (`gemini-1.5-flash`, configurable) | Entity-relation extraction | Cloud LLM call via `google-generativeai`; planned to be replaced by a fine-tuned local reasoning model |

Model weights are cached in a shared `hf_cache` Docker volume and downloaded automatically on first run.

---

## Technology Stack & Key Packages

### Shared library — `ermis-common`
- `tldextract`, `nltk` — text/domain utilities for IOC and NER preprocessing
- `iocsearcher` — IOC pattern extraction (installed from the malicialab GitHub repo)
- Optional extras:
  - `[ml]` — `torch`, `transformers`, `sentencepiece` (NER, ATT&CK, TTP models)
  - `[pdf]` — `PyMuPDF`, `pymupdf4llm`, `pymupdf-layout` (APTNotes / PDF report ingestion)
  - `[scrape]` — `selenium`, `webdriver-manager` (headless-browser scraping)

### Ingestion pipeline — `ermis-ingestion`
- `requests`, `beautifulsoup4`, `lxml`, `cloudscraper`, `ultimate-sitemap-parser` — scraping
- `selenium`, `webdriver-manager` — headless Firefox for JS-rendered pages and PDF download
- `pymupdf4llm`, `PyMuPDF` — PDF → markdown/text extraction
- `nltk`, `numpy`, `tldextract`, `iocsearcher` — IOC extraction pipeline
- `gliner` — auxiliary entity extraction
- `pymisp` — MISP event/attribute export
- `pycti` — OpenCTI client (SDOs, observables, indicators)
- `psycopg2-binary`, `SQLAlchemy`, `pandas` — PostgreSQL access and data shaping
- `neo4j` — graph database driver
- `google-generativeai` — Gemini API client (relation extraction)
- `bibtexparser`, `python-dateutil`

### Portal backend — `ermis-portal/backend`
- `django`, `djangorestframework`, `djangorestframework-simplejwt`, `django-cors-headers`
- `celery`, `redis` — async task queue
- `psycopg2-binary` — PostgreSQL ORM access (managed=False models)
- `neo4j` — graph reader for the Cytoscape view
- `stix2` — STIX 2.1 Bundle export
- `python-dotenv`, `requests`

### Portal frontend — `ermis-portal/frontend`
- `react` 19, `react-dom`, `react-router-dom` 7 — SPA framework & routing
- `typescript` 5.9, `vite` 7 — build tooling
- `axios` — API client with JWT refresh interceptors
- `@mui/material`, `@mui/icons-material`, `@emotion/*` — UI components
- `bootstrap`, `bootstrap-icons` — grid system & layout utilities only (visual design is native CSS)
- `cytoscape`, `cytoscape-fcose`, `react-cytoscapejs` — entity relationship graph
- `react-simple-maps`, `world-atlas` — geographic threat map
- `recharts` — dashboard charts

---

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/) ≥ 24 and [Docker Compose](https://docs.docker.com/compose/) ≥ 2.20
- Git

No local Python or Node.js installation is required — everything runs inside containers.

---

## Installation

### 1. Clone all four repositories as siblings

```bash
mkdir ERMIS && cd ERMIS

git clone https://github.com/madskgg/ermis-deploy.git
git clone https://github.com/madskgg/ermis-common.git
git clone https://github.com/madskgg/ermis-ingestion.git
git clone https://github.com/madskgg/ermis-portal.git
```

All subsequent commands are run from inside `ermis-deploy/`:

```bash
cd ermis-deploy
```

### 2. Create environment files

Two `.env` files are required, placed in the **sibling repos** (`ermis-portal/.env` and `ermis-ingestion/.env`, i.e. `../ermis-portal/.env` and `../ermis-ingestion/.env` relative to `ermis-deploy/`). Create them from the templates below and fill in your values.

**`ermis-portal/.env`** — portal backend & workers

```dotenv
# Django
DJANGO_SECRET_KEY=change-me-to-a-long-random-string
DEBUG=False
ALLOWED_HOSTS=localhost,127.0.0.1
CORS_ALLOWED_ORIGINS=http://localhost:8081

# PostgreSQL (must match the postgresql service in docker-compose.yml)
DB_NAME=ermis
DB_USER=ermis
DB_PASS=change-me
DB_HOST=postgresql
DB_PORT=5432

# Neo4j (must match the neo4j service in docker-compose.yml)
NEO4J_URI=bolt://neo4j:7687
NEO4J_USER=neo4j
NEO4J_PASSWORD=change-me
NEO4J_DATABASE=neo4j

# Redis (must match the redis service in docker-compose.yml)
REDIS_PASSWORD=change-me
CELERY_BROKER_URL=redis://:change-me@redis:6379/0
CELERY_RESULT_BACKEND=redis://:change-me@redis:6379/0

# Shared output volume (matches the /output volume mount)
INGESTION_OUTPUT_ROOT=/output

# MISP integration (optional)
MISP_URL=https://your-misp-instance/
MISP_KEY=your-misp-api-key
MISP_VERIFY_SSL=False

# OpenCTI integration (optional)
OPENCTI_URL=http://your-opencti-instance:8080
OPENCTI_TOKEN=your-opencti-token

# Email (for password reset)
EMAIL_HOST=smtp.example.com
EMAIL_PORT=587
EMAIL_HOST_USER=noreply@example.com
EMAIL_HOST_PASSWORD=your-smtp-password
EMAIL_USE_TLS=True
```

**`ermis-ingestion/.env`** — ingestion pipeline

```dotenv
# PostgreSQL connection string (same DB as the portal)
DATABASE_URL=postgresql://ermis:change-me@postgresql:5432/ermis

# Neo4j
NEO4J_URI=bolt://neo4j:7687
NEO4J_USER=neo4j
NEO4J_PASSWORD=change-me
NEO4J_DATABASE=neo4j

# Output directory (matches the /output volume mount)
INGESTION_OUTPUT_ROOT=/output

# MISP
MISP_URL=https://your-misp-instance/
MISP_KEY=your-misp-api-key
MISP_VERIFY_SSL=false
PUSH_TO_MISP=true

# OpenCTI
OPENCTI_URL=http://your-opencti-instance:8080
OPENCTI_TOKEN=your-opencti-token

# Gemini API (relation extraction)
GOOGLE_API_KEY=your-google-api-key
GEMINI_MODEL=gemini-1.5-flash

# Local CTI-BERT relevance gate threshold
LOCAL_MALWARE_THRESHOLD=0.80

# GitHub token (for scraping GitHub-hosted CTI reports)
GITHUB_TOKEN=your-github-token
```

> **Important:** the `POSTGRES_PASSWORD` and `NEO4J_AUTH` values hard-coded in `docker-compose.yml` (`change-me` placeholders for the `postgresql` and `neo4j` services) must match `DB_PASS` / `NEO4J_PASSWORD` in both `.env` files above.

### 3. Build the base image

The portal backend, Celery worker, and ingestion container share a common base image that must be built first:

```bash
docker compose build base
```

### 4. Start all services

```bash
docker compose up -d
```

On first startup Docker will:
- Pull the PostgreSQL 16, Neo4j 5, Redis 7, and Nginx images
- Build the backend, Celery worker, and frontend images
- Download HuggingFace model weights into the `hf_cache` volume (this takes several minutes on the first run)

### 5. Run database migrations

```bash
docker compose exec backend python manage.py migrate
```

### 6. Create the first admin user

```bash
docker compose exec backend python manage.py createsuperuser
```

### 7. Access the portal

Open **http://localhost:8081** in a browser.

---

## Running the Ingestion Pipeline

The ingestion service scrapes configured sources, extracts IoCs and entities, and pushes them to MISP / OpenCTI. It is defined under the `tools` profile and is **not** started by `docker compose up -d`.

**Run a single ingestion cycle:**

```bash
docker compose --profile tools run --rm ingestion
```

**Scrape a specific list of URLs:**

```bash
docker compose --profile tools run --rm ingestion python main.py sites.txt --ignore-robots
```

Output articles are written to the shared `../output` volume, which is also mounted by the backend so the portal can serve them.

---

## Environment Variable Reference

| Variable | Component | Description |
|---|---|---|
| `DJANGO_SECRET_KEY` | Portal | Django secret key — must be unique and kept private |
| `DEBUG` | Portal | Set `False` in production |
| `ALLOWED_HOSTS` | Portal | Comma-separated list of allowed hostnames |
| `CORS_ALLOWED_ORIGINS` | Portal | Comma-separated allowed CORS origins |
| `DB_NAME / DB_USER / DB_PASS / DB_HOST / DB_PORT` | Portal | PostgreSQL credentials |
| `DATABASE_URL` | Ingestion | Full PostgreSQL connection string |
| `NEO4J_URI / NEO4J_USER / NEO4J_PASSWORD / NEO4J_DATABASE` | Both | Neo4j connection |
| `REDIS_PASSWORD` | Portal | Redis authentication password |
| `CELERY_BROKER_URL / CELERY_RESULT_BACKEND` | Portal | Celery transport URLs (must include the Redis password) |
| `INGESTION_OUTPUT_ROOT` | Both | Absolute path to the shared output volume (`/output`) |
| `MISP_URL / MISP_KEY / MISP_VERIFY_SSL` | Both | MISP connection details |
| `OPENCTI_URL / OPENCTI_TOKEN` | Both | OpenCTI connection details |
| `GOOGLE_API_KEY / GEMINI_MODEL` | Ingestion | Gemini API credentials for relation extraction |
| `LOCAL_MALWARE_THRESHOLD` | Ingestion | Confidence threshold for the local CTI-BERT relevance gate |
| `GITHUB_TOKEN` | Ingestion | GitHub personal access token for scraping private or rate-limited repos |

---

## This Repository's Structure

```
ermis-deploy/
├── docker-compose.yml    # Orchestrates all services (paths reference sibling repos via ../)
├── Dockerfile.base        # Shared base image: Python + ML deps (torch, transformers, etc.)
└── nginx/
    └── default.conf       # Nginx reverse-proxy configuration
```
