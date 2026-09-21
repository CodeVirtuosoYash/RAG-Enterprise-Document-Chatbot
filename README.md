# RAG Enterprise Document Chatbot

A full-stack, multi-tenant Retrieval-Augmented Generation (RAG) system for enterprise document Q&A. Users log in through **AWS Cognito**, upload documents (PDF / DOCX / PPTX / TXT) that are parsed, chunked and stored in **Amazon S3**, and then ask natural-language questions answered by an LLM grounded in their own organization's documents — with links back to the exact source file.

Documents are isolated per `organization / department`, and every action is gated by a role (RAG User, Document Owner, RAG Admin).

---

## Table of Contents

- [Features](#features)
- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [How It Works](#how-it-works)
- [Roles & Permissions](#roles--permissions)
- [S3 Layout](#s3-layout)
- [Getting Started](#getting-started)
- [Environment Variables](#environment-variables)
- [API Reference](#api-reference)
- [AWS Deployment](#aws-deployment)
- [Repository Structure](#repository-structure)
- [Known Limitations / Roadmap](#known-limitations--roadmap)

---

## Features

- **Multi-format ingestion** — PDF, DOCX, PPTX and TXT parsed via `unstructured`, including table structure inference and hi-res PDF extraction.
- **Image extraction** — embedded media is pulled out of DOCX/PPTX archives and PDFs, and stored alongside the document in S3.
- **Semantic search** — BERT (`bert-base-uncased`) mean-pooled embeddings indexed in **FAISS** (`IndexFlatL2`) with an L2 distance threshold, so irrelevant matches are dropped instead of hallucinated over.
- **Grounded generation** — retrieved chunks are passed as context to **Llama 4 Scout 17B** via the Groq API.
- **Source attribution** — every answer returns the document IDs it was built from; the UI can fetch the original file back through a presigned S3 download URL (1-hour expiry).
- **Multi-tenancy** — indexes and S3 prefixes are scoped to `{organization}/{department}`, so one tenant never retrieves another's content.
- **Role-based access control** — upload, delete and user-management actions are permission-checked, and document listings are filtered by ownership.
- **Duplicate detection** — SHA-256 hashing of uploads; identical re-uploads are skipped rather than re-indexed.
- **Self-healing metadata** — metadata entries for documents no longer present in S3 are purged on startup and after deletes.
- **Profanity filtering** — queries and uploaded content are screened with `better_profanity`.

---

## Architecture

```
                    ┌────────────────────────────┐
                    │        AWS Cognito         │  Hosted UI · OAuth2 code flow
                    │  User Pool (ap-south-1)    │
                    └──────────────┬─────────────┘
                                   │ access_token / userInfo
                                   ▼
            ┌──────────────────────────────────────────┐
            │           Streamlit Frontend             │
            │  Frontend.py                             │
            │  • Login / profile (org + department)    │
            │  • Upload · Document manager · Chat      │
            │  • Role Management tab (roles.csv)       │
            └──────────────────┬───────────────────────┘
                               │ REST (JSON)
                               ▼
            ┌──────────────────────────────────────────┐
            │             FastAPI Backend              │
            │  Backend.py                              │
            │  ┌──────────────┐   ┌─────────────────┐  │
            │  │ unstructured │   │ BERT embeddings │  │
            │  │   parsing    │   │  + FAISS index  │  │
            │  └──────┬───────┘   └────────┬────────┘  │
            │         │                    │           │
            │         ▼                    ▼           │
            │  text chunks / images    retrieval       │
            └─────────┬────────────────────┬───────────┘
                      │                    │ context + query
                      ▼                    ▼
              ┌───────────────┐    ┌────────────────┐
              │   Amazon S3   │    │    Groq API    │
              │ originals ·   │    │ Llama 4 Scout  │
              │ text · images │    └────────────────┘
              └───────────────┘
```

---

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | Streamlit |
| Backend | FastAPI + Uvicorn |
| Authentication | AWS Cognito (Hosted UI, OAuth2 authorization-code flow) |
| Object storage | Amazon S3 (boto3) |
| Document parsing | `unstructured` (pdf, docx, pptx, text), PyMuPDF, Pillow |
| Embeddings | HuggingFace Transformers — `bert-base-uncased` |
| Vector search | FAISS (`IndexFlatL2`, 768-dim) |
| Generation | Groq — `meta-llama/llama-4-scout-17b-16e-instruct` |
| Access control | CSV-backed role store (`roles.csv` + `role_modfication.py`) |
| Safety | `better_profanity` |

---

## How It Works

**Ingestion**

1. A user with upload permission selects a file in the Streamlit UI.
2. The backend hashes the file (SHA-256) and skips it if an identical version already exists for that org/dept.
3. The original file is uploaded to S3 under `original_file/`.
4. `unstructured` partitions the document into elements; each non-empty element is written to S3 as a `.txt` chunk under `text/`.
5. Embedded images (DOCX/PPTX media, PDF hi-res extractions) are uploaded under `images/`.
6. An averaged embedding plus the file hash are persisted to `metadata_store.pkl`.
7. The FAISS index for that `org/dept` prefix is rebuilt.

**Query**

1. The frontend posts `{query, org, dept}` to `/Answer`.
2. If the index is cold, the backend reloads every `.txt` chunk under that prefix from S3 and rebuilds it.
3. The query is embedded with BERT and searched against FAISS (top-k = 5, L2 distance threshold = 500).
4. Matching chunks become the context in a grounded prompt sent to Groq.
5. The answer returns with the unique source document IDs, which the UI turns into presigned download links via `/get_source_document/{doc_id}`.

If nothing clears the distance threshold, the API returns `"No Relevant Document Found"` rather than inventing an answer.

---

## Roles & Permissions

| Role | Permissions |
|---|---|
| `RAG_user` | read, query |
| `doc_owner` | read, query, upload, modify, delete — sees only documents they uploaded |
| `RAG_admin` | all of the above, plus admin and manage_users |

Roles live in [roles.csv](Main/roles.csv) (`name, organization, role, email`) and are managed from the **Role Management** tab by any `RAG_admin`. Uploader identity is recorded per document (`ownership.json`), which is what makes the `doc_owner` filtering possible.

---

## S3 Layout

```
s3://<S3_BUCKET>/
└── rag-project-bucket-01/test-output/
    └── <organization>/
        └── <department>/
            └── <document-name>/
                ├── original_file/<document-name>.pdf
                ├── text/<document-name>0.txt, ...1.txt, ...
                ├── images/image_0.png, pdf_image_3.png, .keep
                └── ownership.json
```

Organization and department names are sanitized (alphanumerics, `-`, `_`, lowercased) before being used as prefixes.

---

## Getting Started

### Prerequisites

- Python 3.12
- An AWS account with an S3 bucket and a Cognito user pool
- A Groq API key
- `poppler-utils` for PDF rendering (`sudo apt install poppler-utils` on Debian/Ubuntu)

### Install

```bash
git clone <your-repo-url>
cd RAG-Enterprise-Document-Chatbot/Main
python -m venv .venv && source .venv/bin/activate   # Windows: .venv\Scripts\activate
pip install -r Requirements.txt
```

### Configure

Create a `.env` file inside `Main/`:

```env
GROQ_API_KEY=your_groq_api_key
S3_BUCKET=your-bucket-name
AWS_ACCESS_KEY=your_access_key
AWS_SECRET_KEY=your_secret_key
AWS_REGION=ap-south-1
API_URL=http://localhost:8000
```

The Cognito domain, client ID and redirect URI are set near the top of [Frontend.py](Main/Frontend.py) — point them at your own user pool.

### Run

```bash
# Terminal 1 — API
python -m uvicorn Backend:app --reload --port 8000

# Terminal 2 — UI
streamlit run Frontend.py
```

Open `http://localhost:8501`, sign in through Cognito, set your organization and department, then upload and query.

---

## Environment Variables

| Variable | Used by | Purpose |
|---|---|---|
| `GROQ_API_KEY` | Backend | Groq LLM access (required) |
| `S3_BUCKET` | Backend | Target S3 bucket (required) |
| `AWS_ACCESS_KEY` / `AWS_SECRET_KEY` | Backend | S3 credentials (required) |
| `AWS_REGION` | Backend | AWS region, e.g. `ap-south-1` |
| `API_URL` | Frontend | Backend base URL (defaults to `http://localhost:8000`) |

The backend fails fast at import time if a required variable is missing.

---

## API Reference

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/` | Build/warm the FAISS index for a user profile (`organization`, `department`) |
| `POST` | `/upload_docs` | Parse a file, push original + text chunks + images to S3, rebuild the index |
| `POST` | `/Answer` | Answer a query for an org/dept; returns `result` and `source_documents` |
| `GET` | `/index_status` | Index state, document count, known document IDs |
| `GET` | `/list_documents` | List documents for an org/dept, filtered by role and ownership |
| `GET` | `/get_source_document/{doc_id}` | Original file as base64 plus a presigned download URL |
| `GET` | `/get_source_document_url/{doc_id}` | Download URL, extracted text and images for a document |
| `DELETE` | `/delete_document` | Delete every S3 object for a document, purge metadata, rebuild the index |
| `GET` | `/metadata_status` | Metadata store size and keys |
| `POST` | `/clear_metadata` | Clear the metadata store (destructive) |

Interactive docs are served at `http://localhost:8000/docs` once the backend is running.

---

## AWS Deployment

The system runs on AWS across three services.

**Amazon Cognito** — a user pool in `ap-south-1` with the Hosted UI enabled. The app uses the OAuth2 authorization-code flow (`scope=email+openid+phone`), exchanges the code for an access token, and reads the user's email from `/oauth2/userInfo`. Add the deployed frontend URL to the pool's allowed callback and sign-out URLs, and update `REDIRECT_URI` in `Frontend.py` to match.

**Amazon S3** — one bucket holds every tenant's original files, extracted text chunks, images and ownership records under the `rag-project-bucket-01/test-output/<org>/<dept>/` prefix. Downloads are served through presigned URLs with a 1-hour expiry, so the bucket itself stays private with public access blocked.

**Compute (EC2)** — FastAPI runs under Uvicorn on port `8000` and Streamlit on `8501`, on the same instance. Because ingestion loads BERT and runs `unstructured`'s hi-res PDF pipeline locally, size the instance for CPU and RAM rather than for request volume, and install `poppler-utils` on the host.

**IAM** — the backend currently authenticates to S3 with an access key pair from environment variables. For production, replace it with an **IAM instance role** scoped to `s3:GetObject`, `s3:PutObject`, `s3:DeleteObject` and `s3:ListBucket` on the project prefix only, and drop the key variables from `.env`.

Typical launch on the instance:

```bash
python -m uvicorn Backend:app --host 0.0.0.0 --port 8000
streamlit run Frontend.py --server.port 8501 --server.address 0.0.0.0
```

Open only the ports you need in the security group, and put Streamlit behind HTTPS (an ALB or an nginx reverse proxy) before exposing it — Cognito redirect URIs should be HTTPS anywhere beyond local development.

---

## Repository Structure

```
RAG-Enterprise-Document-Chatbot/
├── Main/
│   ├── Backend.py               # FastAPI app: ingestion, FAISS, retrieval, S3, Groq
│   ├── Frontend.py              # Streamlit app: Cognito auth, upload, chat, role admin
│   ├── role_modfication.py      # CSV-backed role CRUD helpers
│   ├── roles.csv                # User → organization → role mapping
│   ├── metadata_store.pkl       # SHA-256 hashes + averaged embeddings per document
│   ├── Requirements.txt         # Backend + frontend dependencies
│   └── RAG Documentation.txt    # Notes from the earlier local-folder prototype
└── Others/                      # Earlier iterations and standalone components
    ├── Metadata Extractor/
    └── Role Modification/
```

`Others/` keeps older versions of code and components that were built separately before being folded into `Main/`.
