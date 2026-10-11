# Northstar IT Service Desk

An enterprise-style local prototype for IT self-service, grounded knowledge answers, safe mock self-healing, and software provisioning. The default configuration is offline-friendly: FastAPI serves a mock ServiceNow API and in-memory store, and retrieval uses local lexical vector similarity. Optional integrations enable Hugging Face inference, sentence-transformers + FAISS, MongoDB Atlas, and the ServiceNow Table API.

## Run locally

Prerequisites: Node.js 20.19+ and Python 3.10+.

In one terminal, install and run the API:

```powershell
cd backend
python -m venv .venv
.venv\Scripts\Activate.ps1
pip install -r requirements.txt
uvicorn main:app --reload --port 8000
```

In another terminal, run the employee portal from the repository root:

```powershell
npm install
npm run dev
```

Open <http://localhost:5173>. FastAPI's interactive API reference is at <http://127.0.0.1:8000/docs>. Use the portal's sample prompts to demonstrate each workflow.

## Architecture

```text
React + Vite employee portal
       | /api (Vite dev proxy)
FastAPI intent router and workflow agents
       |                 |                 |
Approved Markdown KB   ServiceNow API   MongoDB Atlas
       |
Article retrieval -> optional sentence-transformers -> FAISS
       |
Grounded response -> optional Hugging Face Inference API
```

Classification and action selection are explicit and inspectable. Common issue phrases route to a known workflow, not an unconstrained agent. Knowledge answers use approved article content; unsupported questions receive an explicit no-guidance response and a ticket option. The default confidence threshold is `0.65`; low-confidence requests recommend human escalation. The UI displays intent, confidence, suggested resolution, and retrieved article names.

## AI and retrieval choices

- **Generation:** `Qwen/Qwen2.5-1.5B-Instruct` through the Hugging Face Inference API is a compact open-weight instruction model suitable for a prototype. Set `HF_TOKEN` and optionally `HF_MODEL` to enable it. The prompt restricts responses to retrieved approved guidance; if the service is unavailable, the backend returns a grounded article summary rather than fabricating model output.
- **Embeddings:** `sentence-transformers/all-MiniLM-L6-v2` is a compact open-source sentence embedding model suitable for short IT support articles. Optional installation: `pip install -r backend/requirements-ai.txt`.
- **Vector store:** with optional AI dependencies installed, article content is embedded and ranked by cosine similarity using FAISS `IndexFlatIP`. The default dependencies use normalized bag-of-words cosine retrieval so the project can run without downloading a model. Both return the same source-attributed result shape.
- **Chunking:** the sample corpus uses short curated articles as retrieval units. Split longer enterprise documents into heading-aware chunks with metadata and overlap before embedding.

## Data and integrations

Seven sample Markdown documents live in `backend/knowledge/`. FastAPI loads them at startup. With `MONGODB_URI`, the backend writes `tickets`, `knowledge`, `chat`, `audit`, and `software_requests` collections to the configured Atlas database, seeds article metadata, and hydrates stored records on restart. Without it, the demo uses seeded in-memory incidents and in-memory workflow records, which reset when the process stops.

Without ServiceNow credentials, incident create/get/update and software requests use the mock API. To connect a developer instance, configure `SNOW_INSTANCE_URL`, `SNOW_USERNAME`, and `SNOW_PASSWORD` in the backend environment. Incident creation maps AI summary, description, category, subcategory, priority, impact, urgency, and assignment group to `POST /api/now/table/incident`; updates use `PATCH`. Real software request creation uses `POST /api/now/table/sc_request` when `SNOW_REQUESTED_FOR_SYS_ID` is also configured; the downstream provisioning remains mocked. For production, use OAuth, resolve reference sys_ids, enforce ServiceNow ACLs, add idempotency, and connect provisioning to an approved managed deployment service.

Never commit `.env` or real credentials. `.env.example` lists optional settings; load secrets in the shell or through a local untracked `.env` loader.

## API

| Method | Path | Purpose |
| --- | --- | --- |
| `GET` | `/api/health` | Integration mode and API health |
| `POST` | `/api/assistant` | Intent, confidence, article sources, answer, and next workflow |
| `GET` | `/api/knowledge` | Approved article catalogue |
| `GET` | `/api/knowledge/search?q=...` | Ranked semantic/lexical article search |
| `POST` | `/api/tickets` | Create ServiceNow/mock incident |
| `GET` | `/api/tickets` and `/api/tickets/{number}` | List/get incidents |
| `PATCH` | `/api/tickets/{number}` | Update incident state or resolution |
| `POST` | `/api/automations/password-reset` | Audited mock password self-heal and incident resolution |
| `POST` | `/api/software-requests` | Catalogue request and mock provisioning |
| `GET` | `/api/audit` | Automation audit events |
| `GET` | `/api/dashboard` | Dashboard metrics |

## Tests

From the repository root after installing backend requirements:

```powershell
python -m pytest
```

Tests cover VPN ticket intake, sourced Outlook guidance, password self-heal and audit, software provisioning, unknown-question escalation, and dashboard metrics.

## Safety and prototype limits

- Password reset and provisioning operations are mock actions; no account, password, or device is changed.
- The default intent router uses deterministic phrase-based classification, not autonomous production-grade reasoning.
- Hugging Face generation is optional and can be unavailable or rate-limited; the fallback does not invent an answer.
- Atlas persistence is opt-in and assumes writable collections and network access.
- ServiceNow Basic Auth is only a prototype integration path. Production use should employ OAuth, TLS, least-privilege roles, reference-field lookup, idempotency, and secret management.
- Do not send real credentials or sensitive employee data to this demo.