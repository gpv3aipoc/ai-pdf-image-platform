# AI PDF & Image Processing Platform

A runnable local-first full-stack starter application for PDF parsing, image extraction/mapping, OCR, image detection, segmentation, processing, annotation, and human review.

## Architecture

- **Frontend:** React + Vite
- **Backend:** Python + FastAPI
- **Workflow:** LangGraph state graph
- **Database:** PostgreSQL (Docker Compose); SQLAlchemy models and Alembic-ready structure
- **Vector support:** PostgreSQL `pgvector` extension setup; embeddings are an optional integration point
- **Observability:** OpenTelemetry instrumentation (opt-in)
- **Processing:** PyMuPDF, Pillow, optional OpenCV and Tesseract OCR

The default pipeline is deliberately usable without paid model credentials. SQLAlchemy document/job/text models and a pgvector column are included; initialize them with `from app.database import init_db; init_db()` after PostgreSQL is available. It extracts PDF text and embedded images, creates image metadata and page mapping, runs optional OCR, uses a conservative OpenCV contour-based region proposal when available, creates basic segmentation masks, and generates annotated previews. For production-grade semantic detection/segmentation, configure a model adapter in the relevant agent module.

## Quick start

### 1. Requirements
- Python 3.11+
- Node.js 20+
- Docker Desktop (recommended for PostgreSQL)

### 2. Configure
```bash
cp .env.example .env
```
Edit `.env` if needed. Never commit real credentials.

### 3. Start PostgreSQL
```bash
docker compose -f infrastructure/docker-compose.yml up -d
```
The database enables `vector` extension on startup.

### 4. Start backend
```bash
cd backend
python -m venv .venv
# Windows:
.venv\\Scripts\\activate
# macOS/Linux:
source .venv/bin/activate
pip install -r requirements.txt
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```
API docs: http://localhost:8000/docs

### 5. Start frontend (new terminal)
```bash
cd frontend
npm install
npm run dev
```
Open http://localhost:5173

## Workflow

1. Upload a PDF and provide plain-language processing instructions.
2. The backend creates a job and executes the LangGraph pipeline.
3. Agents parse pages, extract images, map images to source pages, perform OCR (if enabled), propose visual regions, create masks, process images, and annotate previews.
4. With HITL enabled, the job pauses in `pending_review`.
5. Inspect results in the UI and approve or request changes.
6. Download the JSON report and generated images.

## API

- `POST /api/v1/documents/upload` — upload PDF (`multipart/form-data`, field `file`)
- `POST /api/v1/jobs` — start job (`document_id`, `instructions`)
- `GET /api/v1/jobs/{job_id}` — job status
- `GET /api/v1/jobs/{job_id}/agents` — agent statuses
- `GET /api/v1/jobs/{job_id}/results` — results
- `POST /api/v1/jobs/{job_id}/review` — HITL decision (`decision`: `approve` or `request_changes`, optional `feedback`)
- `GET /api/v1/jobs/{job_id}/download` — JSON report
- `GET /api/v1/images/{image_id}` — image file
- `GET /health` — health check

## Configuration

All backend runtime settings are loaded centrally through `backend/app/core/config.py` from the root `.env`. Frontend public settings use `VITE_API_BASE_URL` in `frontend/.env` (copy `frontend/.env.example`). Never put secrets in Vite variables because they are exposed to browsers.

## Production notes

This repository is a development baseline, not a security-reviewed production deployment. Before public DNS exposure:
- Add authentication/authorization, rate limiting, CSRF protections where applicable, and malware scanning.
- Use private object storage (S3-compatible) instead of local disk.
- Use a durable PostgreSQL-backed LangGraph checkpointer and persist job state in PostgreSQL. The included in-process job registry and LangGraph `MemorySaver` are suitable for local single-process development only. The included SQLAlchemy models are a persistence foundation; the demo API currently keeps active job state in memory and does not yet write each workflow transition to PostgreSQL.
- Replace contour-based region proposals with a validated object detection and segmentation model; validate model licenses and hardware requirements.
- Configure HTTPS at a reverse proxy, restrict CORS, rotate secrets, and avoid exposing storage paths.
- Configure a production OTLP collector and exporter.
- Add retention policies, backups, and audit logging.

## Troubleshooting

- If OCR is enabled, install the Tesseract binary separately and ensure it is on `PATH`.
- If OpenCV is unavailable, detection/mask generation returns a graceful "not configured" result.
- PostgreSQL is optional for the local demo pipeline but is included for deployment and schema evolution.


## Source archive

The complete generated project bundle is stored in `PROJECT_SOURCE.zip.base64`. To reconstruct the ZIP on Windows PowerShell:

```powershell
[IO.File]::WriteAllBytes('project-source.zip', [Convert]::FromBase64String((Get-Content .\PROJECT_SOURCE.zip.base64 -Raw)))
Expand-Archive .\project-source.zip -DestinationPath .\source
```
