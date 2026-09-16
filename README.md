# Invoice Review

Human-in-the-loop review for supplier invoices and expense receipts. Upload a PDF, JPEG, or PNG; Azure extracts the fields; local finance rules check VAT and totals; you confirm a GL account and approve or reject.

Built for a fictional facilities company, **Northstar Facilities B.V.** Documents can be English, Dutch, German, or French. The app copy is English.

## What it does

1. Classifies the file as an invoice or a receipt (Azure OpenAI).
2. Extracts fields with Azure AI Document Intelligence (`prebuilt-invoice` or `prebuilt-receipt`).
3. Runs an independent Azure OpenAI pass on the same file. Document Intelligence stays primary; the model only fills gaps and those fallbacks are visible.
4. Applies deterministic Northstar policy (EU VAT format/checksum via `python-stdnum`, totals, duplicates). No live VIES lookup.
5. Suggests one account from a fixed GL catalog (`6100`–`6190`). You can override it.
6. Lets you correct fields, approve, reject, or draft a supplier correction email (copy/close only — nothing is sent).

Reviews are stored in local SQLite. History can be deleted so the same sample can be demoed again.

## Stack

| Area | Choice |
| --- | --- |
| API | Python 3.12, FastAPI, uvicorn, Pydantic v2 |
| Data | SQLAlchemy 2, SQLite, local file uploads |
| Extraction | Azure AI Document Intelligence |
| Classification, review, GL, email draft | Azure OpenAI Responses API |
| UI | Vite, React 19, TypeScript, Tailwind CSS, pnpm |
| Packages | `uv` (backend), `pnpm` (frontend) |

## Prerequisites

- Python 3.12 or newer and [uv](https://docs.astral.sh/uv/)
- Node.js 22 or newer and pnpm 11
- An Azure Document Intelligence resource
- An Azure OpenAI / Foundry chat deployment that supports the Responses API with PDF/image input and structured output

## Setup

```bash
cd backend
uv sync --locked
copy .env.example .env   # Windows: Copy-Item .env.example .env

cd ../frontend
pnpm install --frozen-lockfile
copy .env.example .env
```

Fill `backend/.env` (never commit this file):

```env
AZURE_DOCUMENT_INTELLIGENCE_ENDPOINT=https://your-resource.cognitiveservices.azure.com/
AZURE_DOCUMENT_INTELLIGENCE_KEY=your-key

AZURE_OPENAI_ENDPOINT=https://your-resource.openai.azure.com/openai/v1/
AZURE_OPENAI_DEPLOYMENT=your-deployment-name
AZURE_OPENAI_API_KEY=your-key
```

`frontend/.env` only needs the API origin (default `http://localhost:8000`). Optional backend password gate: `APP_ACCESS_PASSWORD` and `APP_SESSION_SECRET`. Leave them unset for local use.

## Run

Two terminals from the repo:

```bash
# Terminal A — API  http://localhost:8000
cd backend
uv run --locked --no-sync uvicorn app.main:create_app --factory --reload

# Terminal B — UI   http://localhost:5173
cd frontend
pnpm dev
```

On Git Bash you can also run `./scripts/dev.sh` from the repo root.

Swagger: [http://localhost:8000/docs](http://localhost:8000/docs)

## Sample documents

Fictional corpus in `samples/generated/` (12 invoices + 1 Dutch fuel receipt). Expected fields and policy codes are in `samples/manifest.json`.

Good first pass: `02-nl-happy-compact.pdf` (approve), `08-en-total-mismatch.pdf` (totals error), `13-nl-fuel-receipt.png` (receipt).

## API

| Method | Path | Purpose |
| --- | --- | --- |
| `GET` | `/health` | Liveness |
| `GET` | `/api/accounting/gl-accounts` | Fixed GL catalog |
| `POST` | `/api/documents` | Upload and run the full pipeline |
| `GET` | `/api/documents` | List reviews |
| `GET` | `/api/documents/{id}` | One review |
| `GET` | `/api/documents/{id}/file` | Stored original |
| `PUT` | `/api/documents/{id}` | Field corrections + revalidate |
| `PUT` | `/api/documents/{id}/accounting` | Confirm / override GL |
| `POST` | `/api/documents/{id}/decision` | Approve or reject |
| `POST` | `/api/documents/{id}/correction-email` | Draft email (not sent) |
| `DELETE` | `/api/documents/{id}` | Delete review and file |

More detail: [docs/api-and-pipeline.md](docs/api-and-pipeline.md), [docs/architecture.md](docs/architecture.md), [docs/client-brief.md](docs/client-brief.md). Optional Azure Container Apps deploy: [docs/azure-deploy.md](docs/azure-deploy.md).

## Limits

- One PDF, JPEG, or PNG per upload, max 4 MB
- Azure usage is billed per Document Intelligence page and OpenAI tokens; see [docs/pricing.md](docs/pricing.md)

---

Inspired by Dave Ebbelaar's invoice review project.
