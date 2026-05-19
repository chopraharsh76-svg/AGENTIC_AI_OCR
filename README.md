# 🛡️ Brand Guardian AI — Video Compliance QA Pipeline

> An automated, end-to-end AI system that audits video content against regulatory and brand compliance standards, powered by **LangGraph**, **Azure AI**, and **GPT-4o**. Transforms raw video into structured JSON compliance reports with full-stack observability.

---

## 📑 Table of Contents

- [Overview](#-overview)
- [Architecture](#-architecture)
- [LangGraph Workflow](#-langgraph-workflow)
- [Tech Stack](#-tech-stack)
- [Project Structure](#-project-structure)
- [Getting Started](#-getting-started)
  - [Prerequisites](#prerequisites)
  - [Installation](#installation)
  - [Configuration](#configuration)
  - [Index the Knowledge Base](#index-the-knowledge-base)
  - [Run via CLI](#run-via-cli)
  - [Run via API](#run-via-api)
- [API Reference](#-api-reference)
- [Compliance Report Output](#-compliance-report-output)
- [Observability](#-observability)
- [Compliance Knowledge Base](#-compliance-knowledge-base)
- [Contributing](#-contributing)
- [License](#-license)

---

## 🔍 Overview

**Brand Guardian AI** is a production-grade compliance auditing pipeline that ingests a YouTube video URL and produces a structured JSON report of all regulatory violations detected within the content.

The system orchestrates four stages through a **LangGraph** stateful graph:

```
YouTube URL  →  Azure Video Indexer  →  Transcript + OCR
                                               ↓
                                  Azure AI Search (RAG)
                               + Azure OpenAI Embeddings
                                               ↓
                              GPT-4o Compliance Reasoning
                                               ↓
                          Structured JSON Compliance Report
                      (traced by LangSmith + App Insights)
```

---

## 🏗️ Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                      Brand Guardian AI                           │
│                                                                  │
│  ┌─────────────────┐     ┌──────────────────────────────────┐   │
│  │  YouTube URL    │────▶│      Azure Video Indexer         │   │
│  │  (Input)        │     │  Download · Upload · Poll        │   │
│  └─────────────────┘     │  Transcript · OCR Extraction     │   │
│                           └───────────────┬──────────────────┘   │
│                                           │                       │
│                           ┌───────────────▼──────────────────┐   │
│                           │       LangGraph Workflow          │   │
│                           │                                   │   │
│                           │   ┌───────────────────────────┐  │   │
│                           │   │  Node 1: index_video_node │  │   │
│                           │   │  yt-dlp download          │  │   │
│                           │   │  Azure VI upload & poll   │  │   │
│                           │   │  Transcript + OCR extract │  │   │
│                           │   └─────────────┬─────────────┘  │   │
│                           │                 │ VideoAuditState │   │
│                           │   ┌─────────────▼─────────────┐  │   │
│                           │   │  Node 2: audit_content    │  │   │
│                           │   │  RAG: Azure AI Search     │  │   │
│                           │   │  GPT-4o Reasoning         │  │   │
│                           │   │  Structured JSON Output   │  │   │
│                           │   └─────────────┬─────────────┘  │   │
│                           └─────────────────┼─────────────────┘   │
│                                             │                      │
│                           ┌─────────────────▼──────────────────┐  │
│                           │       JSON Compliance Report        │  │
│                           │  violations · severity · status     │  │
│                           └────────────────────────────────────┘  │
│                                                                    │
│  ┌──────────────────────┐   ┌───────────────────────────────────┐ │
│  │      LangSmith       │   │    Azure Application Insights     │ │
│  │  LLM Trace Logging   │   │  Telemetry · Logs · Perf Metrics  │ │
│  └──────────────────────┘   └───────────────────────────────────┘ │
└────────────────────────────────────────────────────────────────────┘
```

---

## 🔄 LangGraph Workflow

The pipeline is a compiled `StateGraph` with two sequential nodes sharing a typed `VideoAuditState`:

| Step | Node | Function | Responsibility |
|------|------|----------|----------------|
| 1 | `indexer` | `index_video_node` | Downloads the YouTube video via `yt-dlp`, uploads to Azure Video Indexer, polls until processing is complete, extracts transcript and OCR text |
| 2 | `auditor` | `audit_content_node` | Embeds transcript + OCR, retrieves top-K relevant compliance rules via Azure AI Search (RAG), invokes GPT-4o to detect violations, returns structured JSON |

**Graph edges:** `START → indexer → auditor → END`

**State schema (`VideoAuditState`):**

```python
class VideoAuditState(TypedDict):
    video_url: str
    video_id: str
    local_file_path: Optional[str]
    video_metadata: Dict[str, Any]
    transcript: Optional[str]
    ocr_text: List[str]
    compliance_results: Annotated[List[ComplianceIssue], operator.add]
    final_status: str          # "PASS" | "FAIL"
    final_report: str
    errors: Annotated[List[str], operator.add]
```

---

## 🛠️ Tech Stack

| Layer | Technology |
|-------|-----------|
| **Orchestration** | [LangGraph](https://github.com/langchain-ai/langgraph) `>=1.0.5` |
| **LLM Reasoning** | Azure OpenAI — GPT-4o (`temperature=0.0`) |
| **Embeddings** | Azure OpenAI — `text-embedding-3-small` |
| **Video Analysis** | Azure Video Indexer (ARM token auth) |
| **YouTube Download** | `yt-dlp` |
| **Vector Search / RAG** | Azure AI Search (`langchain-community`) |
| **API Server** | FastAPI + Uvicorn |
| **LLM Observability** | LangSmith (`LANGCHAIN_TRACING_V2=true`) |
| **App Telemetry** | Azure Application Insights (OpenTelemetry) |
| **Azure Auth** | `DefaultAzureCredential` (azure-identity) |
| **Package Manager** | [uv](https://github.com/astral-sh/uv) |
| **Python Version** | 3.12 |

---

## 📁 Project Structure

```
ComplianceQAPipeline/
│
├── main.py                          # CLI entry point — runs a simulated audit
├── pyproject.toml                   # Project metadata & dependencies (uv)
├── .env                             # Environment variables (see Configuration)
├── .python-version                  # Pins Python 3.12
│
├── backend/
│   ├── Dockerfile                   # Container definition for the backend
│   │
│   ├── data/                        # Compliance knowledge base (PDF source docs)
│   │   ├── 1001a-influencer-guide-508_1.pdf   # FTC Influencer Disclosure Guide
│   │   └── youtube-ad-specs.pdf               # YouTube Ad Policies
│   │
│   ├── scripts/
│   │   ├── index_documents.py       # One-time script: chunks PDFs → uploads to Azure AI Search
│   │   └── explanation.txt          # Notes on the indexing strategy
│   │
│   └── src/
│       ├── api/
│       │   ├── __init__.py
│       │   ├── server.py            # FastAPI app — POST /audit, GET /health
│       │   └── telemetry.py         # Azure Monitor OpenTelemetry setup
│       │
│       ├── graph/
│       │   ├── __init__.py
│       │   ├── state.py             # VideoAuditState + ComplianceIssue TypedDicts
│       │   ├── nodes.py             # index_video_node + audit_content_node
│       │   └── workflow.py          # LangGraph StateGraph definition & compilation
│       │
│       └── services/
│           ├── __init__.py
│           └── video_indexer.py     # Azure Video Indexer client (token auth, upload, poll, extract)
│
└── azure_functions/
    ├── function_app.py              # Azure Functions trigger (serverless alternative)
    ├── host.json                    # Azure Functions host config
    ├── local.settings.json          # Local dev settings for Azure Functions
    └── requirements.txt             # Azure Functions-specific dependencies
```

---

## 🚀 Getting Started

### Prerequisites

- **Python 3.12+**
- **[uv](https://github.com/astral-sh/uv)** package manager
- An Azure subscription with the following services provisioned:
  - Azure Video Indexer (ARM-managed account)
  - Azure OpenAI (GPT-4o chat deployment + `text-embedding-3-small` embedding deployment)
  - Azure AI Search (index pre-populated via `index_documents.py`)
  - Azure Application Insights workspace
- **LangSmith** account (for LLM tracing)

---

### Installation

```bash
# 1. Clone the repository
git clone https://github.com/your-org/ComplianceQAPipeline.git
cd ComplianceQAPipeline

# 2. Install dependencies with uv
uv sync
```

---

### Configuration

Populate the `.env` file at the project root:

```env
# ── Azure Storage ──────────────────────────────────────────────
AZURE_STORAGE_CONNECTION_STRING=""

# ── Azure OpenAI ───────────────────────────────────────────────
AZURE_OPENAI_API_KEY=""
AZURE_OPENAI_ENDPOINT=""                 # https://<resource>.openai.azure.com/
AZURE_OPENAI_API_VERSION=""              # e.g. 2024-02-01
AZURE_OPENAI_CHAT_DEPLOYMENT=""          # e.g. gpt-4o
AZURE_OPENAI_EMBEDDING_DEPLOYMENT=""     # e.g. text-embedding-3-small

# ── Azure AI Search ────────────────────────────────────────────
AZURE_SEARCH_ENDPOINT=""                 # https://<resource>.search.windows.net
AZURE_SEARCH_API_KEY=""
AZURE_SEARCH_INDEX_NAME=""              # e.g. compliance-rules

# ── Azure Video Indexer ────────────────────────────────────────
AZURE_VI_NAME=""                        # Resource name in Azure Portal
AZURE_VI_LOCATION=""                    # e.g. eastus
AZURE_VI_ACCOUNT_ID=""
AZURE_SUBSCRIPTION_ID=""
AZURE_RESOURCE_GROUP=""

# ── Azure Application Insights ─────────────────────────────────
APPLICATIONINSIGHTS_CONNECTION_STRING=""

# ── LangSmith ──────────────────────────────────────────────────
LANGCHAIN_TRACING_V2=true
LANGCHAIN_ENDPOINT="https://api.smith.langchain.com"
LANGCHAIN_API_KEY=""
LANGCHAIN_PROJECT=""                    # e.g. brand-guardian-qa
```

---

### Index the Knowledge Base

Run this **once** before your first audit. It chunks the PDFs in `backend/data/` and uploads vector embeddings to Azure AI Search:

```bash
uv run python backend/scripts/index_documents.py
```

Expected output:
```
✅ Indexing Complete! The Knowledge Base is ready.
Total chunks indexed: 87
```

---

### Run via CLI

```bash
uv run python main.py
```

Runs a simulated audit against the demo YouTube URL. The full compliance report is printed to the console:

```
=== COMPLIANCE AUDIT REPORT ===
Video ID:    vid_ce6c43bb
Status:      FAIL

[ VIOLATIONS DETECTED ]
- [CRITICAL] Misleading Claims: Absolute guarantee detected at 00:32
- [WARNING]  FTC Disclosure: Sponsored content indicator absent

[ FINAL SUMMARY ]
Video contains 2 violations. Immediate review required...
```

---

### Run via API

Start the FastAPI server:

```bash
uv run uvicorn backend.src.api.server:app --reload
```

The server starts at `http://localhost:8000`.

| URL | Description |
|-----|-------------|
| `http://localhost:8000/docs` | Interactive Swagger UI |
| `http://localhost:8000/health` | Health check |
| `http://localhost:8000/audit` | Main audit endpoint (POST) |

---

## 📡 API Reference

### `POST /audit`

Triggers the full compliance audit pipeline for a given video URL.

**Request body:**

```json
{
  "video_url": "https://youtu.be/dT7S75eYhcQ"
}
```

**Response:**

```json
{
  "session_id": "ce6c43bb-c71a-4f16-a377-8b493502fee2",
  "video_id": "vid_ce6c43bb",
  "status": "FAIL",
  "final_report": "The video contains 2 violations requiring immediate attention...",
  "compliance_results": [
    {
      "category": "Misleading Claims",
      "severity": "CRITICAL",
      "description": "The phrase 'clinically proven' is used without a cited study or regulatory reference."
    },
    {
      "category": "FTC Disclosure",
      "severity": "WARNING",
      "description": "No sponsored content disclosure present for a paid partnership."
    }
  ]
}
```

**Error response (500):**

```json
{
  "detail": "Workflow Execution Failed: YouTube Download Failed: ..."
}
```

---

### `GET /health`

```json
{
  "status": "healthy",
  "service": "Brand Guardian AI"
}
```

---

## 📄 Compliance Report Output

Each audit produces a final `VideoAuditState` containing:

| Field | Type | Description |
|-------|------|-------------|
| `final_status` | `"PASS"` \| `"FAIL"` | Overall compliance verdict |
| `final_report` | `str` | GPT-4o natural language summary |
| `compliance_results` | `List[ComplianceIssue]` | All detected violations |
| `transcript` | `str` | Full extracted speech-to-text |
| `ocr_text` | `List[str]` | On-screen text detected by Azure Video Indexer |
| `video_metadata` | `dict` | Duration (seconds), platform |
| `errors` | `List[str]` | Any system-level errors encountered |

**`ComplianceIssue` schema:**

```python
class ComplianceIssue(TypedDict):
    category:    str            # e.g. "FTC_DISCLOSURE", "Misleading Claims"
    description: str            # Specific detail of the violation
    severity:    str            # "CRITICAL" | "WARNING"
    timestamp:   Optional[str]  # Timestamp of occurrence (if available)
```

---

## 📊 Observability

### LangSmith — LLM Tracing

Set `LANGCHAIN_TRACING_V2=true` and provide your `LANGCHAIN_API_KEY`. Every GPT-4o call is fully traced — inputs, outputs, token counts, latency, and intermediate chain steps — enabling precise prompt debugging and cost optimization.

### Azure Application Insights — Production Telemetry

`backend/src/api/telemetry.py` initializes Azure Monitor OpenTelemetry on server startup. It automatically instruments:

- All FastAPI HTTP requests (endpoint, status code, duration)
- Azure AI Search similarity query calls
- LangGraph node execution events
- Application errors and exceptions

> If `APPLICATIONINSIGHTS_CONNECTION_STRING` is absent, telemetry is gracefully disabled — the application continues running normally.

---

## 📚 Compliance Knowledge Base

The RAG knowledge base is seeded from two authoritative PDFs in `backend/data/`:

| File | Source |
|------|--------|
| `1001a-influencer-guide-508_1.pdf` | FTC Influencer & Endorsement Disclosure Guide |
| `youtube-ad-specs.pdf` | YouTube Advertising Policies & Specifications |

Documents are chunked at `1000` characters with `200`-character overlap using `RecursiveCharacterTextSplitter`, embedded with `text-embedding-3-small`, and stored in Azure AI Search.

To extend the knowledge base, place additional PDFs in `backend/data/` and re-run:

```bash
uv run python backend/scripts/index_documents.py
```

---

## 🤝 Contributing

1. Fork the repository
2. Create a feature branch — `git checkout -b feature/my-feature`
3. Commit your changes — `git commit -m 'feat: add my feature'`
4. Push to the branch — `git push origin feature/my-feature`
5. Open a Pull Request

Please follow [Conventional Commits](https://www.conventionalcommits.org/) for commit message formatting.

---

## 📜 License

This project is licensed under the MIT License. See [LICENSE](LICENSE) for details.
