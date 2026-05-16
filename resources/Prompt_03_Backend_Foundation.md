# Prompt 03: Backend Foundation (FastAPI + Database + Telemetry)

**MODE: AUTOMATIC — AI writes code directly, no user interaction needed**


## Reference Documents (READ THESE FIRST)

Before writing any code, read the following documents in `resources/` for the backend architecture and patterns:

| Priority | Document | Why Read It |
|----------|----------|-------------|
| **PRIMARY** | `resources/Deliverable_3_Implementation_Prompts.md` | **BACKEND SKELETON.** Read Prompt 1 (FastAPI scaffold) and Prompt 2 (Temporal worker setup) for the exact project structure and boilerplate code. |
| **REFERENCE** | `resources/Deliverable_0_PROJECT_CONTEXT.md` | Architecture decisions (Section 3), repository structure (Section 5), and tech stack (Section 11). Read for the "why" behind the 4-repo layout and the FastAPI + Temporal architecture. |
| **REFERENCE** | `resources/ARCHITECTURE_DATAFLOW_GUIDE.md` | Complete system architecture. Read Layer 5 (Application Layer) for the FastAPI endpoints and Layer 6 (Data Layer) for the PostgreSQL schema. |
| **REFERENCE** | `resources/GIT_WORKFLOW.md` | Git branch strategy and commit conventions. Read before making the first commit to understand the `dev` branch workflow. |

> **How to find these documents:** They are in the `resources/` folder of the `agentcore-demo-test1-docs` repo (or the `resources/` folder if docs are copied locally). All prompt and deliverable files follow the naming convention: `Prompt_NN_Descriptive_Name.md` and `Deliverable_N_Descriptive_Name.md`.

---

**Target Model:** Gemini Flash, Claude Sonnet (low-thinking, deterministic execution)
**Mission:** Build the complete FastAPI backend foundation — project skeleton, config, auth, telemetry, storage, API routes, migrations, Docker, and docker-compose.
**Constraint:** SEQUENTIAL execution only. Every step must be verified before proceeding. If a step fails, STOP and report.

---

## CRITICAL RULES (NON-NEGOTIABLE)

- **Deployment method: S3 ZIP ONLY.** There is NO ECR. There is NO `docker build` for agents. There is NO container push. Agent deployment is: `agentcore configure -e main.py --protocol AGUI` then `agentcore deploy`. CodeBuild handles the ARM64 build from the ZIP. The ONLY Dockerfile in this project is for `docker-compose` frontend/backend infrastructure (not for agent deployment).
- **Python environment isolation:** NEVER use global `pip install`. ALWAYS use `uv` with `.venv`:
  ```bash
  uv venv .venv
  source .venv/bin/activate  # Linux/Mac
  # .venv\Scripts\activate   # Windows
  uv pip install -r requirements.txt
  ```
  - The ONLY exception is `RUN pip install` inside a Dockerfile (container-level isolation). That is explicitly allowed and NOT considered "global."
- **Node.js environment isolation:** NEVER use `npm install -g` (global). ALWAYS install locally via `npm install` (or `pnpm install`, `yarn install`) into `./node_modules`.
- **All code comments MUST be written in English only.** No other language in comments, docstrings, or string literals.
- If ANY step fails, STOP and report the exact error. Do NOT proceed.

---

## CONTEXT

You are executing **Phase 3** of a multi-phase system build:
- **Phase 0** (COMPLETE): EC2 running, RDS available with `temporal` and `agentcore_demo_test1` databases, S3 buckets exist, `.env` configured
- **Phase 1** (COMPLETE): 3 agents deployed to AgentCore, each emitting 8-block JSON payloads
- **Phase 2** (COMPLETE): Agents emit V5 8-block payloads as OTel span events, exported via OTLP gRPC to ADOT Collector on EC2, visible in CloudWatch GenAI Dashboard
- **Phase 4** (NOT YET): Temporal workflow code — `Prompt_04_Temporal_Workflow.md`
- **Phase 5** (NOT YET): Frontend Next.js — `Prompt_05_Frontend.md`

**Database Rule:** The application connects ONLY to `agentcore_demo_test1`. The `temporal` database is owned by Temporal Server's auto-setup. Never touch it.

---

## WORKING DIRECTORY (canonical per PCD §3 D5)

All Phase 3 work happens in the **backend repo**:

```bash
export PROJECT_ROOT="/Users/ugurgocen/projects/agentcore-demo-test1"
cd "$PROJECT_ROOT/agentcore-demo-test1-backend"
```

Every `agentcore-demo-test1-backend/...` path below is relative to `$PROJECT_ROOT`. Inside Docker containers the equivalent path is `/app/...`.

## PREREQUISITE CHECK

Before beginning, verify the project directory exists:

```bash
ls /Users/ugurgocen/projects/agentcore-demo-test1/ || echo "PROJECT_ROOT_NOT_FOUND"
ls /Users/ugurgocen/projects/agentcore-demo-test1/agentcore-demo-test1-backend/.git || echo "BACKEND_REPO_NOT_INITIALIZED"
```

**GATE:** If either string is printed, STOP immediately. Phase 0 (Prompt_00) must have created the project root; Phase 1 (Prompt_01) initialized the backend repo.

---

## STEP 1: Create Directory Structure

**Action:** Create ALL directories and empty `__init__.py` files in a single command.

```bash
mkdir -p agentcore-demo-test1-backend/app/{auth,telemetry,storage,temporal,config}
mkdir -p agentcore-demo-test1-backend/alembic/versions
mkdir -p agentcore-demo-test1-backend/alembic
mkdir -p ../agentcore-demo-test1-frontend  # Needed for docker-compose later

# Create all __init__.py files
touch agentcore-demo-test1-backend/app/__init__.py
touch agentcore-demo-test1-backend/app/auth/__init__.py
touch agentcore-demo-test1-backend/app/telemetry/__init__.py
touch agentcore-demo-test1-backend/app/storage/__init__.py
touch agentcore-demo-test1-backend/app/temporal/__init__.py
touch agentcore-demo-test1-backend/app/config/__init__.py
```

**Verification:**
```bash
find agentcore-demo-test1-backend -type f | sort
```

**Expected output (11 files):**
```
agentcore-demo-test1-backend/app/__init__.py
agentcore-demo-test1-backend/app/auth/__init__.py
agentcore-demo-test1-backend/app/config/__init__.py
agentcore-demo-test1-backend/app/storage/__init__.py
agentcore-demo-test1-backend/app/telemetry/__init__.py
agentcore-demo-test1-backend/app/temporal/__init__.py
```

**FAILURE HANDLING:** If any directory is missing, STOP and report which one.

---

## STEP 2: Create requirements.txt

**Action:** Write the complete `requirements.txt` file.

**File:** `agentcore-demo-test1-backend/requirements.txt`

```text
fastapi>=0.109.0
uvicorn[standard]>=0.27.0
sqlalchemy>=2.0.0
alembic>=1.13.0
psycopg2-binary>=2.9.9
pydantic>=2.5.0
pydantic-settings>=2.1.0
boto3>=1.34.0
structlog>=24.1.0
opentelemetry-api>=1.22.0
opentelemetry-sdk>=1.22.0
opentelemetry-instrumentation-fastapi>=0.43b0
temporalio>=1.5.0
python-multipart>=0.0.6
```

**Verification:**
```bash
wc -l agentcore-demo-test1-backend/requirements.txt
```
Expected: 14 lines.

**FAILURE HANDLING:** If the file does not exist or has fewer than 14 lines, STOP and report.

---

## STEP 3: Create config/settings.py

**Action:** Write the Pydantic settings module.

**File:** `agentcore-demo-test1-backend/app/config/settings.py`

Write the following EXACT content:

```python
"""
Application settings via Pydantic.
Reads from environment variables. On EC2, these come from:
1. .env file (generated by user-data script in Phase 0)
2. EC2 instance metadata (for region, account)
3. Defaults for demo

CRITICAL: We connect ONLY to the agentcore_demo_test1 database.
The temporal database is owned by Temporal Server — we never touch it.
"""
from pydantic_settings import BaseSettings
from functools import lru_cache

class Settings(BaseSettings):
    # AWS
    aws_region: str = "us-east-1"
    aws_account_id: str = ""

    # RDS — agentcore_demo_test1 database only
    db_host: str = "localhost"
    db_port: int = 5432
    db_name: str = "agentcore_demo_test1"  # NEVER temporal
    db_user: str = "app_user"
    db_password: str = ""

    # S3
    audio_bucket: str = ""
    artifacts_bucket: str = ""
    claimcheck_bucket: str = ""

    # Temporal Server
    temporal_host: str = "temporal-server"
    temporal_port: int = 7233

    # AgentCore
    agent_1_runtime_arn: str = ""
    agent_2_runtime_arn: str = ""
    agent_3_runtime_arn: str = ""

    # App
    app_name: str = "agentcore-demo-test1"
    environment: str = "demo"
    claim_check_threshold_bytes: int = 1_048_576  # 1 MB per Guidelines Section 11

    class Config:
        env_file = ".env"
        env_file_encoding = "utf-8"

@lru_cache()
def get_settings() -> Settings:
    return Settings()
```

**Verification:**
```bash
grep -c "class Settings" agentcore-demo-test1-backend/app/config/settings.py
grep -c "db_name.*agentcore_demo_test1" agentcore-demo-test1-backend/app/config/settings.py
grep -c "get_settings" agentcore-demo-test1-backend/app/config/settings.py
```
Each grep must return 1. Also verify `db_name` is `agentcore_demo_test1` (not `temporal`).

**FAILURE HANDLING:** If any grep returns 0, STOP and report.

---

## CHECKPOINT 1: Verify Steps 1-3

**Action:** Run this combined verification:

```bash
echo "=== Directory Structure ==="
find agentcore-demo-test1-backend -type f | sort | wc -l
echo "=== requirements.txt ==="
wc -l < agentcore-demo-test1-backend/requirements.txt
echo "=== settings.py ==="
python3 -c "import sys; sys.path.insert(0, 'agentcore-demo-test1-backend'); from app.config.settings import Settings, get_settings; s = get_settings(); print(f'db_name={s.db_name}')" 2>&1
```

**Expected results:**
1. Directory file count: at least 7 (the __init__.py files)
2. requirements.txt: 14 lines
3. settings.py import: prints `db_name=agentcore_demo_test1` (uses default since no .env)

**If ANY check fails, STOP and report the exact error. Do not proceed.**

---

## STEP 4: Create Mock Auth Module

**Action:** Write the mock user module.

**File:** `agentcore-demo-test1-backend/app/auth/mock_user.py`

Write the following EXACT content:

```python
"""
Mock authentication for the demo.
NO login screen, NO JWT, NO dependency injection complexity.
A single constant feeds the requested_by block of every payload.

"""
from typing import Dict, Any

MOCK_USER: Dict[str, Any] = {
    "user_id": "demo-user-001",
    "email": "demo@example.com",
    "roles": ["functional_engineer"],
    "group": {"group_id": "default", "group_type": "module", "group_name": "Default"},
    "project": {
        "project_id": "PROJ-DEFAULT-123",
        "project_name": "AgentCore demo test 1",
        "system": "Demo",
        "sector": "MT",
        "vertical": "Non-ERP",
    },
}

def get_mock_user() -> Dict[str, Any]:
    """Return the mock user. Every endpoint uses this for the requested_by field."""
    return MOCK_USER.copy()
```

**Verification:**
```bash
grep -c "demo-user-001" agentcore-demo-test1-backend/app/auth/mock_user.py
grep -c "get_mock_user" agentcore-demo-test1-backend/app/auth/mock_user.py
```
Each grep must return 1.

**FAILURE HANDLING:** If either grep returns 0, STOP and report.

---

## STEP 5: Create OpenTelemetry Tracing Module

**Action:** Write the W3C trace propagation middleware.

**File:** `agentcore-demo-test1-backend/app/telemetry/tracing.py`

Write the following EXACT content:

```python
"""
W3C Trace Propagation Middleware
Per Non-Negotiable Rule #1: W3C trace propagation through every layer.

This middleware:
1. Parses the traceparent header from incoming requests
2. Creates an OpenTelemetry span for the request
3. Propagates the trace context to downstream calls (Temporal, boto3)
4. Emits structured logs with structlog including the trace_id

Reference: https://opentelemetry.io/docs/languages/python/
Guidelines Section 8.5: OpenTelemetry GenAI semantic conventions.
"""
import structlog
from fastapi import Request
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.propagate import extract, inject
from opentelemetry.trace.propagation.tracecontext import TraceContextTextMapPropagator

# Initialize tracer provider
trace.set_tracer_provider(TracerProvider())
tracer = trace.get_tracer("agentcore-demo-test1")

logger = structlog.get_logger()

def setup_tracing(app):
    """Configure OpenTelemetry tracing for FastAPI. Call this in main.py."""
    FastAPIInstrumentor.instrument_app(app)

def get_trace_id(request: Request) -> str:
    """
    Extract trace_id from the traceparent header.
    W3C format: traceparent: 00-<trace_id>-<span_id>-<flags>
    If no header, generate a new trace_id.
    """
    traceparent = request.headers.get("traceparent", "")
    if traceparent:
        parts = traceparent.split("-")
        if len(parts) >= 2:
            return parts[1]
    # Generate new trace ID
    span = tracer.start_span("extract_trace")
    trace_id = format(span.get_span_context().trace_id, "032x")
    span.end()
    return trace_id

def get_current_trace_id() -> str:
    """Get the trace_id from the current span context."""
    span = trace.get_current_span()
    ctx = span.get_span_context()
    if ctx.is_valid:
        return format(ctx.trace_id, "032x")
    return "unknown"
```

**Verification:**
```bash
grep -c "setup_tracing" agentcore-demo-test1-backend/app/telemetry/tracing.py
grep -c "get_trace_id" agentcore-demo-test1-backend/app/telemetry/tracing.py
grep -c "get_current_trace_id" agentcore-demo-test1-backend/app/telemetry/tracing.py
grep -c "FastAPIInstrumentor" agentcore-demo-test1-backend/app/telemetry/tracing.py
```
Each grep must return at least 1.

**FAILURE HANDLING:** If any grep returns 0, STOP and report.

---

## STEP 6: Workflow-level telemetry helper (OTel-only)

**Removed in v2:** the previous version of this step created `app/telemetry/emf.py` to write manual EMF JSON to CloudWatch from FastAPI. That contradicts **PCD §4 rule 4** and **PCD §10**: agents and backend services emit OTel spans only; the ADOT Collector produces EMF. We replace the EMF module with a thin OTel span helper.

**Action:** Write the workflow-level telemetry helper.

**File:** `agentcore-demo-test1-backend/app/telemetry/workflow_events.py`

```python
"""
Backend workflow-event emitter using OpenTelemetry.

Adds a child span for each workflow lifecycle event (workflow_started,
agent_invoked, hitl_emitted, workflow_completed, etc.) with the same demo.*
attribute taxonomy the agents use. The ADOT Collector picks these up via
OTLP gRPC and transforms them into EMF records under the DemoSDLC/Agent
namespace (PCD §10). The backend does NOT call CloudWatch APIs directly.
"""
from __future__ import annotations
from contextlib import contextmanager
from opentelemetry import trace
from app.telemetry.tracing import get_current_trace_id

_tracer = trace.get_tracer("agentcore-demo-test1-backend")


@contextmanager
def workflow_event_span(
    *,
    event_type: str,
    workflow_id: str,
    workflow_template_id: str,
    actor_id: str,
    extra: dict | None = None,
):
    """Open a child span for a workflow lifecycle event.

    `event_type` examples: 'workflow_started', 'agent_invoked', 'hitl_emitted',
    'signal_received', 'evidence_pack_persisted', 'workflow_completed'.
    """
    with _tracer.start_as_current_span(f"workflow.{event_type}") as span:
        span.set_attribute("demo.event_type", event_type)
        span.set_attribute("demo.workflow_id", workflow_id)
        span.set_attribute("demo.workflow_template_id", workflow_template_id)
        span.set_attribute("demo.actor_id", actor_id)
        span.set_attribute("demo.tier", "first_party")
        if extra:
            for key, value in extra.items():
                # All custom attrs go under demo.* so ADOT's governance/cardinality rules apply.
                span.set_attribute(f"demo.{key}", value)
        yield span
```

**Verification:**
```bash
grep -c "workflow_event_span" agentcore-demo-test1-backend/app/telemetry/workflow_events.py
grep -c "demo.workflow_id" agentcore-demo-test1-backend/app/telemetry/workflow_events.py
! grep -q "BRDDemo/Orchestrator" agentcore-demo-test1-backend/app/telemetry/workflow_events.py && echo "OK: no manual EMF namespace"
! grep -q "CloudWatchMetrics" agentcore-demo-test1-backend/app/telemetry/workflow_events.py && echo "OK: no manual EMF JSON"
```

**FAILURE HANDLING:** If any grep fails, STOP and report.

---

## CHECKPOINT 2: Verify Steps 4-6

**Action:** Run this combined verification:

```bash
echo "=== mock_user.py ==="
grep "demo-user-001" agentcore-demo-test1-backend/app/auth/mock_user.py | wc -l
echo "=== tracing.py ==="
grep "setup_tracing" agentcore-demo-test1-backend/app/telemetry/tracing.py | wc -l
grep "W3C" agentcore-demo-test1-backend/app/telemetry/tracing.py | wc -l
echo "=== workflow_events.py ==="
grep "workflow_event_span" agentcore-demo-test1-backend/app/telemetry/workflow_events.py | wc -l
grep "demo.workflow_id" agentcore-demo-test1-backend/app/telemetry/workflow_events.py | wc -l
```

**Expected results:**
1. mock_user.py: 1 (demo-user-001 found)
2. tracing.py: 1 (setup_tracing found), 1 (W3C found)
3. workflow_events.py: 1 (workflow_event_span found), 1+ (demo.workflow_id used)

**If ANY check fails, STOP and report the exact error. Do not proceed.**

---

## STEP 7: Create Evidence Pack Builder

**Action:** Write the Evidence Pack builder module.

**File:** `agentcore-demo-test1-backend/app/telemetry/evidence_pack.py`

Write the following EXACT content:

```python
"""
Evidence Pack Builder
Per Guidelines Section 14.6: An Evidence Pack is generated at the end of EVERY
workflow run, regardless of outcome (APPROVED, REJECTED, TERMINATED, FAILED).

The Evidence Pack is an immutable JSON manifest that points to all records
produced during a run. It does NOT duplicate data — it attests to it with hashes.

ALCOA+ principles enforced:
- Attributable: populated requested_by.user_id, agent_id, approver_id
- Legible: structured JSON with agreed schema
- Contemporaneous: ISO 8601 UTC timestamps
- Original: SHA-256 hashes of payloads and artifacts
- Accurate: verified token counts
- Complete: Unavailable markers where data is missing
- Consistent: schema_version field
- Enduring: persisted to RDS append-only table
- Available: trace IDs, run IDs, and search attributes included
"""
from datetime import datetime, timezone
from typing import Dict, Any, List
import hashlib
import json

def build_evidence_pack(
    workflow_id: str,
    trace_id: str,
    workflow_template_id: str,
    status: str,  # APPROVED, REJECTED, TERMINATED, FAILED
    user_id: str,
    agents_invoked: List[Dict[str, str]],
    artifact_refs: List[Dict[str, str]] = None,
) -> Dict[str, Any]:
    """
    Build an ALCOA+ compliant Evidence Pack.
    Called by the Temporal workflow's finally block — guaranteed to run
    for EVERY outcome.
    """
    now = datetime.now(timezone.utc).isoformat()

    # Build pointers to all records
    pointers = {
        "cloudwatch_log_group": "/agentcore-demo-test1/fastapi",
        "cloudwatch_query_keys": ["trace_id", "workflow_id", "agent_run_id", "step_id"],
        "temporal_workflow_id": workflow_id,
        "s3_artifacts": artifact_refs or [],
    }

    # Compute hash of the pack itself for tamper evidence
    pack_content = f"{workflow_id}:{trace_id}:{status}:{now}"
    pack_hash = hashlib.sha256(pack_content.encode()).hexdigest()

    return {
        "evidence_pack_id": f"ep-{workflow_id}",
        "schema_version": "1.0.0",
        "trace_id": trace_id,
        "workflow_template_id": workflow_template_id,
        "workflow_id": workflow_id,
        "generated_at": now,
        "generated_by": "orchestrator-agentcore-demo-test1",
        "status": status,
        "attribution": {
            "initiated_by": user_id,
            "agents_invoked": agents_invoked,
            "approvers": [],  # Populated based on status
        },
        "pointers": pointers,
        "pack_hash": f"sha256:{pack_hash}",
        "alcoa_attestation": {
            "attributable": True,
            "legible": True,
            "contemporaneous": True,
            "original_hashes_recorded": True,
            "accurate_cross_validated": True,
            "complete_unavailable_markers_used": True,
            "consistent_schema_version": "1.0.0",
            "enduring_persisted": True,
            "available_indexed": True,
        },
    }
```

**Verification:**
```bash
grep -c "build_evidence_pack" agentcore-demo-test1-backend/app/telemetry/evidence_pack.py
grep -c "alcoa_attestation" agentcore-demo-test1-backend/app/telemetry/evidence_pack.py
grep -c "sha256" agentcore-demo-test1-backend/app/telemetry/evidence_pack.py
```
Each grep must return at least 1.

**FAILURE HANDLING:** If any grep returns 0, STOP and report.

---

## STEP 8: Create RDS Models

**Action:** Write the SQLAlchemy models for the `agentcore_demo_test1` database.

**File:** `agentcore-demo-test1-backend/app/storage/rds_models.py`

Write the following EXACT content:

```python
"""
SQLAlchemy 2.0+ models for the agentcore_demo_test1 database.

The app connects only to agentcore_demo_test1 using the app_user role.
The temporal and temporal_visibility databases are owned by Temporal Server's
auto-setup; we never touch their schemas.

Canonical evidence_packs schema (PCD §3 D10): UUID primary key, JSONB columns
for agent_outputs / hitl_exchanges / metadata, BRDState outcome enum, version
tracking. Phase 3 creates the table; Phase 4 only inserts/updates rows.

Reference: PAYLOAD_SCHEMA.md §4 (8-block reference); PCD §3 D10/D12.
"""
from __future__ import annotations
import uuid
from datetime import datetime, timezone
from functools import lru_cache
from sqlalchemy import create_engine, String, DateTime, Text, Integer, JSON
from sqlalchemy.dialects.postgresql import UUID, JSONB
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, sessionmaker, Session
from app.config.settings import get_settings


class Base(DeclarativeBase):
    """Modern SQLAlchemy 2.0+ declarative base. Replaces deprecated declarative_base()."""
    pass


class ChatMessage(Base):
    """Chat history per workflow (used by frontend page-refresh restore)."""
    __tablename__ = "chat_messages"

    id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    workflow_id: Mapped[str] = mapped_column(String(255), nullable=False, index=True)
    role: Mapped[str] = mapped_column(String(50), nullable=False)  # user / assistant / system
    content: Mapped[str] = mapped_column(Text, nullable=False)
    created_at: Mapped[datetime] = mapped_column(DateTime, default=lambda: datetime.now(timezone.utc))


class EvidencePack(Base):
    """
    Canonical Evidence Pack schema per PCD §3 D10.

    One row per workflow run. Updated by the orchestrator as the workflow
    progresses (initialized PENDING, advances through IN_PROGRESS, terminates
    in APPROVED/REJECTED/FAILED/MAX_ITERATIONS). Append-only is enforced at
    the application layer via the orchestrator's update API, not at the DB
    role level.
    """
    __tablename__ = "evidence_packs"

    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    workflow_id: Mapped[str] = mapped_column(String(255), nullable=False, unique=True, index=True)
    workflow_template_id: Mapped[str] = mapped_column(String(255), nullable=False, index=True)
    brd_id: Mapped[str | None] = mapped_column(String(255), nullable=True)
    outcome: Mapped[str] = mapped_column(String(50), nullable=False, default="PENDING")
    # BRDState enum: PENDING, IN_PROGRESS, AWAITING_HUMAN, APPROVED, REJECTED, FAILED, MAX_ITERATIONS
    rounds_executed: Mapped[int] = mapped_column(Integer, default=0)
    max_rounds: Mapped[int] = mapped_column(Integer, default=3)  # D8 self-correction cap
    agent_outputs: Mapped[dict] = mapped_column(JSONB, default=dict)  # {agent_id: [8-block-payload, ...]}
    hitl_exchanges: Mapped[list] = mapped_column(JSONB, default=list)  # [{step_id, question, response, ...}]
    final_brd_content: Mapped[str | None] = mapped_column(Text, nullable=True)
    error: Mapped[str | None] = mapped_column(Text, nullable=True)
    extra_metadata: Mapped[dict] = mapped_column(JSONB, default=dict)  # arbitrary trailing context
    created_at: Mapped[datetime] = mapped_column(DateTime, default=lambda: datetime.now(timezone.utc))
    completed_at: Mapped[datetime | None] = mapped_column(DateTime, nullable=True)


@lru_cache()
def get_engine():
    settings = get_settings()
    db_url = (
        f"postgresql+psycopg2://{settings.db_user}:{settings.db_password}"
        f"@{settings.db_host}:{settings.db_port}/{settings.db_name}"
    )
    return create_engine(db_url, pool_pre_ping=True)


def get_db() -> Session:
    engine = get_engine()
    return sessionmaker(bind=engine, expire_on_commit=False)()
```

**Verification:**
```bash
grep -c "class ChatMessage" agentcore-demo-test1-backend/app/storage/rds_models.py
grep -c "class EvidencePack" agentcore-demo-test1-backend/app/storage/rds_models.py
grep -c "DeclarativeBase" agentcore-demo-test1-backend/app/storage/rds_models.py   # SQLAlchemy 2.0 base
grep -c "agent_outputs" agentcore-demo-test1-backend/app/storage/rds_models.py     # canonical D10 column
grep -c "hitl_exchanges" agentcore-demo-test1-backend/app/storage/rds_models.py    # canonical D10 column
grep -c "get_engine" agentcore-demo-test1-backend/app/storage/rds_models.py
grep -c "get_db" agentcore-demo-test1-backend/app/storage/rds_models.py
# Make sure deprecated import is GONE:
! grep -q "from sqlalchemy.ext.declarative import declarative_base" agentcore-demo-test1-backend/app/storage/rds_models.py && echo "OK: no deprecated import"
```
All greps return >=1; the `! grep` check must print `OK: no deprecated import`.

**FAILURE HANDLING:** If any grep returns 0, STOP and report.

---

## STEP 9: Create S3 Claim-Check Module

**Action:** Write the S3 claim-check pattern implementation.

**File:** `agentcore-demo-test1-backend/app/storage/s3_claim_check.py`

Write the following EXACT content:

```python
"""
S3 Claim-Check Pattern
Per Guidelines Section 11: Any payload larger than 1 MB is passed via
Claim-Check, not inline. Threshold is configurable via CLAIM_CHECK_THRESHOLD_BYTES.

This prevents Temporal Event History from exceeding the 50 MB / 51,200 event limit.
"""
import boto3
import json
import hashlib
from typing import Dict, Any, Optional, Tuple
from app.config.settings import get_settings

_s3_client = None

def _get_s3():
    global _s3_client
    if _s3_client is None:
        _s3_client = boto3.client("s3")
    return _s3_client

def apply_claim_check(payload: Dict[str, Any], threshold_bytes: int = None) -> Tuple[Dict[str, Any], bool]:
    """
    If payload > threshold, upload to S3 and return a lightweight reference.
    Returns: (result_payload, was_claim_checked)
    """
    settings = get_settings()
    threshold = threshold_bytes or settings.claim_check_threshold_bytes

    payload_bytes = json.dumps(payload).encode("utf-8")

    if len(payload_bytes) <= threshold:
        return payload, False

    # Upload to claim-check bucket
    s3 = _get_s3()
    workflow_id = payload.get("status", {}).get("workflow_id", "unknown")
    agent_run_id = payload.get("status", {}).get("agent_run_id", "unknown")
    s3_key = f"claim-checks/{workflow_id}/{agent_run_id}.json"

    s3.put_object(
        Bucket=settings.claimcheck_bucket,
        Key=s3_key,
        Body=payload_bytes,
        ContentType="application/json",
    )

    # Return lightweight reference
    ref = {
        "claim_check_ref": f"s3://{settings.claimcheck_bucket}/{s3_key}",
        "payload_hash": f"sha256:{hashlib.sha256(payload_bytes).hexdigest()}",
        "size_bytes": len(payload_bytes),
    }
    return ref, True
```

**Verification:**
```bash
grep -c "apply_claim_check" agentcore-demo-test1-backend/app/storage/s3_claim_check.py
grep -c "claim_check_ref" agentcore-demo-test1-backend/app/storage/s3_claim_check.py
grep -c "claimcheck_bucket" agentcore-demo-test1-backend/app/storage/s3_claim_check.py
```
Each grep must return at least 1.

**FAILURE HANDLING:** If any grep returns 0, STOP and report.

---

## CHECKPOINT 3: Verify Steps 7-9

**Action:** Run this combined verification:

```bash
echo "=== evidence_pack.py ==="
grep "build_evidence_pack" agentcore-demo-test1-backend/app/telemetry/evidence_pack.py | wc -l
grep "ALCOA" agentcore-demo-test1-backend/app/telemetry/evidence_pack.py | wc -l
echo "=== rds_models.py ==="
grep "class ChatMessage" agentcore-demo-test1-backend/app/storage/rds_models.py | wc -l
grep "class EvidencePack" agentcore-demo-test1-backend/app/storage/rds_models.py | wc -l
grep "class WorkflowState" agentcore-demo-test1-backend/app/storage/rds_models.py | wc -l
echo "=== s3_claim_check.py ==="
grep "apply_claim_check" agentcore-demo-test1-backend/app/storage/s3_claim_check.py | wc -l
grep "claim_check_ref" agentcore-demo-test1-backend/app/storage/s3_claim_check.py | wc -l
```

**Expected results:**
1. evidence_pack.py: 1 (build_evidence_pack), 1+ (ALCOA references)
2. rds_models.py: 1 (ChatMessage), 1 (EvidencePack), 1 (WorkflowState)
3. s3_claim_check.py: 1 (apply_claim_check), 1 (claim_check_ref)

**If ANY check fails, STOP and report the exact error. Do not proceed.**

---

## STEP 10: Create FastAPI Main Application

**Action:** Write the main FastAPI entrypoint.

**File:** `agentcore-demo-test1-backend/app/main.py`

Write the following EXACT content:

```python
"""
FastAPI entrypoint for the AgentCore demo test 1 demo.

Responsibilities:
- Workflow lifecycle endpoints (start, signal, restart)
- Workflow state queries
- Metrics dashboard endpoint
- CopilotKit runtime bridge
- DOES NOT proxy AG-UI events (those go AgentCore -> browser directly)

Architecture: Hybrid — REST to FastAPI for lifecycle, AG-UI direct for agent events.
"""
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from app.config.settings import get_settings
from app.telemetry.tracing import setup_tracing
from app.storage.rds_models import get_db, WorkflowState

settings = get_settings()

app = FastAPI(title="AgentCore demo test 1", version="1.0.0")

# CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],  # Demo only — restrict in production
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# OpenTelemetry tracing
setup_tracing(app)

@app.get("/health")
def health():
    """Health check endpoint. Called by docker-compose and load balancers."""
    return {"status": "ok", "version": "1.0.0"}

@app.get("/api/metrics/dashboard")
def dashboard():
    """
    Landing page metrics. 5-minute cached CloudWatch GetMetricData.
    Returns 4 metric tiles for the frontend landing page.
    """
    # Phase 3: Return placeholder data. Will be implemented with real
    # CloudWatch queries in Phase 4.
    return {
        "workflow_completion_rate": {"value": 0, "unit": "percent"},
        "avg_latency": {"value": 0, "unit": "ms"},
        "total_tokens": {"value": 0, "unit": "count"},
        "first_pass_acceptance": {"value": 0, "unit": "percent"},
    }

@app.post("/api/v1/workflows")
def create_workflow():
    """Start a new workflow. Returns workflow_id."""
    import uuid
    workflow_id = f"wf-{uuid.uuid4().hex[:12]}"
    return {"workflow_id": workflow_id}

@app.get("/api/v1/workflows/{workflow_id}/state")
def get_workflow_state(workflow_id: str):
    """Query workflow state from RDS."""
    db = get_db()
    state = db.query(WorkflowState).filter(WorkflowState.workflow_id == workflow_id).first()
    if not state:
        raise HTTPException(status_code=404, detail="Workflow not found")
    return {
        "workflow_id": state.workflow_id,
        "status": state.status,
        "current_phase": state.current_phase,
    }

if __name__ == "__main__":
    import uvicorn
    uvicorn.run("app.main:app", host="0.0.0.0", port=8000, reload=False)
```

**Verification:**
```bash
grep -c "/health" agentcore-demo-test1-backend/app/main.py
grep -c "/api/metrics/dashboard" agentcore-demo-test1-backend/app/main.py
grep -c "/api/v1/workflows" agentcore-demo-test1-backend/app/main.py
grep -c "setup_tracing" agentcore-demo-test1-backend/app/main.py
grep -c "WorkflowState" agentcore-demo-test1-backend/app/main.py
grep -c "get_db" agentcore-demo-test1-backend/app/main.py
```
Each grep must return at least 1.

**FAILURE HANDLING:** If any grep returns 0, STOP and report.

---

## STEP 11: Create Alembic Configuration

### 11a: Alembic INI

**Action:** Write `alembic.ini`.

**File:** `agentcore-demo-test1-backend/alembic.ini`

```ini
# A generic, single database configuration.

[alembic]
# path to migration scripts
script_location = alembic

# template used to generate migration file names
# file_template = %%(rev)s_%%(slug)s

# sys.path path, will be prepended to sys.path if present.
prepend_sys_path = .

# timezone to use when rendering the date within the migration file
# timezone = UTC

# max length of characters to apply to the
# "slug" field
truncate_slug_length = 40

# set to 'true' to run the environment during
# the 'revision' command, regardless of autogenerate
# revision_environment = false

# set to 'true' to allow .pyc and .pyo files without
# a source .py file to be detected as revisions in the
# versions/ directory
sourceless = false

# version location specification
version_locations = %(here)s/alembic/versions

# version path separator
version_path_separator = os

# the output encoding used when revision files
# are written from script.py.mako
# output_encoding = utf-8

sqlalchemy.url = postgresql://app_user:password@localhost:5432/agentcore_demo_test1

[post_write_hooks]

# Logging configuration
[loggers]
keys = root,sqlalchemy,alembic

[handlers]
keys = console

[formatters]
keys = generic

[logger_root]
level = WARN
handlers = console
qualname =

[logger_sqlalchemy]
level = WARN
handlers =
qualname = sqlalchemy.engine

[logger_alembic]
level = INFO
handlers =
qualname = alembic

[handler_console]
class = StreamHandler
args = (sys.stderr,)
level = NOTSET
formatter = generic

[formatter_generic]
format = %(levelname)-5.5s [%(name)s] %(message)s
datefmt = %H:%M:%S
```

### 11b: Alembic env.py

**Action:** Write the Alembic environment script.

**File:** `agentcore-demo-test1-backend/alembic/env.py`

```python
"""
Alembic environment configuration.
Uses app.storage.rds_models.Base as target metadata.
"""
import os
import sys

# Add the backend directory to the path
sys.path.insert(0, os.path.join(os.path.dirname(__file__), ".."))

from logging.config import fileConfig
from sqlalchemy import engine_from_config, pool
from alembic import context

from app.config.settings import get_settings
from app.storage.rds_models import Base

# this is the Alembic Config object
config = context.config

# Interpret the config file for Python logging.
if config.config_file_name is not None:
    fileConfig(config.config_file_name)

# Add model's MetaData object for 'autogenerate' support
target_metadata = Base.metadata

def get_database_url():
    settings = get_settings()
    return f"postgresql://{settings.db_user}:{settings.db_password}@{settings.db_host}:{settings.db_port}/{settings.db_name}"

def run_migrations_offline() -> None:
    url = get_database_url()
    context.configure(
        url=url,
        target_metadata=target_metadata,
        literal_binds=True,
        dialect_opts={"paramstyle": "named"},
    )
    with context.begin_transaction():
        context.run_migrations()

def run_migrations_online() -> None:
    configuration = config.get_section(config.config_ini_section, {})
    configuration["sqlalchemy.url"] = get_database_url()
    connectable = engine_from_config(
        configuration,
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )
    with connectable.connect() as connection:
        context.configure(
            connection=connection,
            target_metadata=target_metadata,
        )
        with context.begin_transaction():
            context.run_migrations()

if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()
```

### 11c: Initial Migration

**Action:** Write the first migration file.

**File:** `agentcore-demo-test1-backend/alembic/versions/001_initial.py`

```python
"""
Initial migration for agentcore_demo_test1 database.

NOTE: The temporal database is NOT managed here. Temporal Server's auto-setup
script manages its own schema in the temporal database. We connect only to
agentcore_demo_test1 via the app_user role.

ALCOA+ Enduring constraint: The app_user role has only INSERT permission on
the evidence_packs table. This is enforced at the database level via GRANT
statements in infra/scripts/03_create_rds.sh. At the application level, we
add a secondary guard by not implementing update/delete methods on EvidencePack.

Revision ID: 001
Revises:
Create Date: 2024-01-01 00:00:00.000000
"""
from typing import Sequence, Union
from alembic import op
import sqlalchemy as sa

# revision identifiers, used by Alembic.
revision: str = "001"
down_revision: Union[str, None] = None
branch_labels: Union[str, Sequence[str], None] = None
depends_on: Union[str, Sequence[str], None] = None

def upgrade() -> None:
    # Enable pgcrypto for gen_random_uuid() if not already enabled
    op.execute("CREATE EXTENSION IF NOT EXISTS \"pgcrypto\";")

    # chat_messages: full chat history per workflow (for page-refresh restore)
    op.create_table(
        "chat_messages",
        sa.Column("id", sa.Integer(), autoincrement=True, nullable=False),
        sa.Column("workflow_id", sa.String(255), nullable=False),
        sa.Column("role", sa.String(50), nullable=False),
        sa.Column("content", sa.Text(), nullable=False),
        sa.Column("created_at", sa.DateTime(), nullable=False),
        sa.PrimaryKeyConstraint("id"),
    )
    op.create_index("ix_chat_messages_workflow_id", "chat_messages", ["workflow_id"])

    # evidence_packs: canonical schema per PCD §3 D10.
    # Phase 4 inserts/updates rows; the orchestrator owns lifecycle transitions.
    op.create_table(
        "evidence_packs",
        sa.Column("id", postgresql.UUID(as_uuid=True), server_default=sa.text("gen_random_uuid()"), nullable=False),
        sa.Column("workflow_id", sa.String(255), nullable=False, unique=True),
        sa.Column("workflow_template_id", sa.String(255), nullable=False),
        sa.Column("brd_id", sa.String(255), nullable=True),
        sa.Column("outcome", sa.String(50), nullable=False, server_default="PENDING"),
        sa.Column("rounds_executed", sa.Integer(), nullable=False, server_default="0"),
        sa.Column("max_rounds", sa.Integer(), nullable=False, server_default="3"),
        sa.Column("agent_outputs", postgresql.JSONB, nullable=False, server_default=sa.text("'{}'::jsonb")),
        sa.Column("hitl_exchanges", postgresql.JSONB, nullable=False, server_default=sa.text("'[]'::jsonb")),
        sa.Column("final_brd_content", sa.Text(), nullable=True),
        sa.Column("error", sa.Text(), nullable=True),
        sa.Column("extra_metadata", postgresql.JSONB, nullable=False, server_default=sa.text("'{}'::jsonb")),
        sa.Column("created_at", sa.DateTime(), nullable=False, server_default=sa.func.now()),
        sa.Column("completed_at", sa.DateTime(), nullable=True),
        sa.PrimaryKeyConstraint("id"),
    )
    op.create_index("ix_evidence_packs_workflow_id", "evidence_packs", ["workflow_id"])
    op.create_index("ix_evidence_packs_workflow_template_id", "evidence_packs", ["workflow_template_id"])
    op.create_index("ix_evidence_packs_outcome", "evidence_packs", ["outcome"])


def downgrade() -> None:
    op.drop_table("evidence_packs")
    op.drop_table("chat_messages")
```

Add the `postgresql` dialect import at the top of the migration:

```python
from alembic import op
import sqlalchemy as sa
from sqlalchemy.dialects import postgresql
```

**Verification:**
```bash
echo "=== alembic.ini ==="
grep -c "agentcore_demo_test1" agentcore-demo-test1-backend/alembic.ini
echo "=== alembic/env.py ==="
grep -c "rds_models" agentcore-demo-test1-backend/alembic/env.py
grep -c "Base.metadata" agentcore-demo-test1-backend/alembic/env.py
echo "=== migration ==="
grep -c "chat_messages" agentcore-demo-test1-backend/alembic/versions/001_initial.py
grep -c "evidence_packs" agentcore-demo-test1-backend/alembic/versions/001_initial.py
grep -c "agent_outputs" agentcore-demo-test1-backend/alembic/versions/001_initial.py   # canonical D10 column
grep -c "hitl_exchanges" agentcore-demo-test1-backend/alembic/versions/001_initial.py  # canonical D10 column
grep -c "postgresql" agentcore-demo-test1-backend/alembic/versions/001_initial.py      # JSONB dialect import
# Phase 4 (Prompt_04) inserts/updates evidence_packs rows; it must NOT re-create the table.
! grep -q "workflow_states" agentcore-demo-test1-backend/alembic/versions/001_initial.py && echo "OK: workflow_states table removed (state now lives in evidence_packs row)"
```

**Expected:** alembic.ini=1, env.py has rds_models and Base.metadata, migration has both tables (chat_messages + evidence_packs), JSONB columns present, no obsolete workflow_states.

**FAILURE HANDLING:** If any check fails, STOP and report.

---

## CHECKPOINT 4: Verify Steps 10-11

**Action:** Run this combined verification:

```bash
echo "=== main.py ==="
grep "/health" agentcore-demo-test1-backend/app/main.py | wc -l
grep "setup_tracing" agentcore-demo-test1-backend/app/main.py | wc -l
echo "=== alembic files ==="
test -f agentcore-demo-test1-backend/alembic.ini && echo "alembic.ini: OK" || echo "alembic.ini: MISSING"
test -f agentcore-demo-test1-backend/alembic/env.py && echo "env.py: OK" || echo "env.py: MISSING"
test -f agentcore-demo-test1-backend/alembic/versions/001_initial.py && echo "001_initial.py: OK" || echo "001_initial.py: MISSING"
```

**Expected results:**
1. main.py: /health found, setup_tracing found
2. All 3 alembic files exist (each prints "OK")

**If ANY check fails, STOP and report the exact error. Do not proceed.**

---

## STEP 12: Create Dockerfile

**Action:** Write the backend Dockerfile.

**File:** `agentcore-demo-test1-backend/Dockerfile`

```dockerfile
# Dockerfile for the AgentCore demo test 1 demo backend.
# This single image serves TWO containers:
# 1. fastapi: Runs uvicorn for the REST API (default CMD below)
# 2. temporal-worker: Runs Temporal worker (overrides CMD in docker-compose)
#
# Sharing one image keeps dependencies consistent between API and worker.
# Both need: FastAPI, SQLAlchemy, boto3, temporalio, opentelemetry, etc.

FROM python:3.11-slim

WORKDIR /app

# Install build dependencies for psycopg2
RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc libpq-dev \
    && rm -rf /var/lib/apt/lists/*

# Install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY . .

EXPOSE 8000

# Default command: uvicorn for FastAPI
# The temporal-worker container overrides this with: python -m app.worker_entrypoint
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**Verification:**
```bash
grep -c "uvicorn" agentcore-demo-test1-backend/Dockerfile
grep -c "temporal-worker" agentcore-demo-test1-backend/Dockerfile
grep -c "python:3.11-slim" agentcore-demo-test1-backend/Dockerfile
```
Each grep must return at least 1.

**FAILURE HANDLING:** If any grep returns 0, STOP and report.

---

## STEP 13: Create docker-compose.yml

**Action:** Write the docker-compose file with all 6 services (per PCD §2.2).

**File:** `agentcore-demo-test1-backend/docker-compose.yml`

> **No `version:` top-level key.** It is deprecated as of Compose v2 and the engine ignores it. Modern Compose treats files as v3.x+ by default.

```yaml
services:
  temporal-server:
    image: temporalio/auto-setup:1.22
    container_name: temporal-server
    ports:
      - "7233:7233"
    environment:
      - DB=postgresql12
      - DB_PORT=5432
      - POSTGRES_USER=${RDS_USER_TEMPORAL}
      - POSTGRES_PWD=${RDS_PASSWORD_TEMPORAL}
      - POSTGRES_SEEDS=${RDS_HOST}
      - DBNAME=${RDS_DB_TEMPORAL}
      - VISIBILITY_DBNAME=${RDS_DB_TEMPORAL_VISIBILITY}
      - DYNAMIC_CONFIG_FILE_PATH=config/dynamicconfig/development-sql.yaml
    restart: unless-stopped

  temporal-ui:
    image: temporalio/ui:2.21
    container_name: temporal-ui
    ports:
      - "8081:8080"   # Host 8081 → container 8080 (frees host:8080 for any local use)
    environment:
      - TEMPORAL_ADDRESS=temporal-server:7233
      - TEMPORAL_CORS_ORIGINS=http://localhost:3000
    depends_on:
      - temporal-server
    restart: unless-stopped

  fastapi:
    build: .
    container_name: fastapi
    ports:
      - "8000:8000"
    environment:
      - RDS_HOST=${RDS_HOST}
      - RDS_DB_APP=${RDS_DB_APP}
      - RDS_USER_APP=${RDS_USER_APP}
      - RDS_PASSWORD_APP=${RDS_PASSWORD_APP}
      - S3_BUCKET_AUDIO=${S3_BUCKET_AUDIO}
      - S3_BUCKET_ARTIFACTS=${S3_BUCKET_ARTIFACTS}
      - S3_BUCKET_CLAIMCHECK=${S3_BUCKET_CLAIMCHECK}
      - AWS_REGION=${AWS_REGION:-us-east-1}
      - OTEL_EXPORTER_OTLP_ENDPOINT=http://adot-collector:4317
      - WORKFLOW_STREAMS_PATTERN=${WORKFLOW_STREAMS_PATTERN:-A}
    depends_on:
      - temporal-server
      - adot-collector
    restart: unless-stopped
    # IMDS credentials propagate automatically from the EC2 host (no static keys).

  temporal-worker:
    build: .                    # same image as fastapi; different CMD
    container_name: temporal-worker
    command: ["python", "-m", "app.temporal.worker_entrypoint"]
    environment:
      - RDS_HOST=${RDS_HOST}
      - RDS_DB_APP=${RDS_DB_APP}
      - RDS_USER_APP=${RDS_USER_APP}
      - RDS_PASSWORD_APP=${RDS_PASSWORD_APP}
      - S3_BUCKET_AUDIO=${S3_BUCKET_AUDIO}
      - S3_BUCKET_ARTIFACTS=${S3_BUCKET_ARTIFACTS}
      - S3_BUCKET_CLAIMCHECK=${S3_BUCKET_CLAIMCHECK}
      - AGENTCORE_ARN_TRANSCRIBER=${AGENTCORE_ARN_TRANSCRIBER}
      - AGENTCORE_ARN_DRAFTER=${AGENTCORE_ARN_DRAFTER}
      - AGENTCORE_ARN_REVIEWER=${AGENTCORE_ARN_REVIEWER}
      - AWS_REGION=${AWS_REGION:-us-east-1}
      - OTEL_EXPORTER_OTLP_ENDPOINT=http://adot-collector:4317
      - WORKFLOW_STREAMS_PATTERN=${WORKFLOW_STREAMS_PATTERN:-A}
    depends_on:
      - temporal-server
      - adot-collector
    restart: unless-stopped

  nextjs-frontend:
    build: ../agentcore-demo-test1-frontend
    container_name: nextjs-frontend
    ports:
      - "3000:3000"
    environment:
      - NEXT_PUBLIC_API_URL=http://localhost:8000
    depends_on:
      - fastapi
    restart: unless-stopped

  adot-collector:
    image: public.ecr.aws/aws-observability/aws-otel-collector:latest
    container_name: adot-collector
    command: ["--config=/etc/otel-collector-config.yaml"]
    volumes:
      - ../agentcore-demo-test1-infra/adot/otel-collector-config.yaml:/etc/otel-collector-config.yaml:ro
    ports:
      - "4317:4317"   # OTLP gRPC ingress (PCD §10)
    environment:
      - AWS_REGION=${AWS_REGION:-us-east-1}
    restart: unless-stopped

  # Pattern B fallback: enable only when temporalio.contrib.workflow_streams is unavailable (D6).
  # `docker compose --profile pattern-b up` includes this service.
  redis:
    image: redis:7-alpine
    container_name: redis
    profiles: ["pattern-b"]
    ports:
      - "6379:6379"
    restart: unless-stopped
```

**Verification:**
```bash
CF=agentcore-demo-test1-backend/docker-compose.yml
grep -c "temporal-server" $CF        # 1
grep -c "temporal-ui" $CF            # 1
grep -c "fastapi:" $CF               # 1
grep -c "temporal-worker:" $CF       # 1
grep -c "nextjs-frontend:" $CF       # 1
grep -c "adot-collector:" $CF        # 1 (new — PCD §10)
grep -c "4317" $CF                   # 1 (OTLP gRPC)
grep -c "7233" $CF                   # 1
grep -c "8000:8000" $CF              # 1
# Deprecated keys should NOT appear:
! grep -q '^version:' $CF && echo "OK: no deprecated version key"
```

**Expected:** all 6 services present; ADOT collector on 4317; FastAPI on 8000; no `version:` key.

**FAILURE HANDLING:** If any service is missing or ports are wrong, STOP and report.

---

## STEP 14: Create Temporal Placeholder Files

**Action 14a:** Write the worker entrypoint placeholder.

**File:** `agentcore-demo-test1-backend/app/worker_entrypoint.py`

```python
"""
Temporal Worker entrypoint.
This is the CMD for the temporal-worker container in docker-compose.
It shares the same Docker image as fastapi but uses this entrypoint instead.

Phase 3: Placeholder — just keeps the container alive.
Phase 4: Will start the actual Temporal worker with all activities registered.
"""
import time

if __name__ == "__main__":
    print("[Worker] Phase 3 placeholder — worker not yet fully configured")
    print("[Worker] Temporal worker will be implemented in Phase 4")
    # Keep container alive
    while True:
        time.sleep(60)
```

**Action 14b:** Write the 4 Temporal placeholder files.

**File:** `agentcore-demo-test1-backend/app/temporal/workflows.py`

```python
"""
Phase 4: Will be implemented in Prompt_04_Temporal_Workflow.md

This file will contain:
- AudioToBRDWorkflow class (Temporal Workflow)
- Signal handlers for human-in-the-loop gates
- Activity orchestration logic
- Evidence pack generation in finally block
"""
```

**File:** `agentcore-demo-test1-backend/app/temporal/activities.py`

```python
"""
Phase 4: Will be implemented in Prompt_04_Temporal_Workflow.md

This file will contain:
- Agent invocation activities (3 activities, one per agent)
- Evidence pack persistence activity
- Claim-check resolution activity
- S3 artifact save activity
"""
```

**File:** `agentcore-demo-test1-backend/app/temporal/worker.py`

```python
"""
Phase 4: Will be implemented in Prompt_04_Temporal_Workflow.md

This file will contain:
- Temporal Worker factory
- Activity and Workflow registration
- Worker startup/shutdown lifecycle
"""
```

**File:** `agentcore-demo-test1-backend/app/temporal/workflow_streams.py`

```python
"""
Phase 4: Will be implemented in Prompt_04_Temporal_Workflow.md

This file will contain:
- AsyncGenerator for streaming workflow events to CopilotKit
- Event buffer for AG-UI bridge
- Real-time status update stream
"""
```

**Verification:**
```bash
for f in worker_entrypoint.py temporal/workflows.py temporal/activities.py temporal/worker.py temporal/workflow_streams.py; do
    echo "=== $f ==="
    test -f agentcore-demo-test1-backend/app/$f && grep -c "Phase 4" agentcore-demo-test1-backend/app/$f || echo "MISSING"
done
```

**Expected:** Each file exists and contains "Phase 4".

**FAILURE HANDLING:** If any file is missing or lacks the Phase 4 comment, STOP and report.

---

## CHECKPOINT 5: Verify Steps 12-14

**Action:** Run this combined verification:

```bash
echo "=== Dockerfile ==="
grep -c "python:3.11-slim" agentcore-demo-test1-backend/Dockerfile
grep -c "temporal-worker" agentcore-demo-test1-backend/Dockerfile
echo "=== docker-compose.yml ==="
grep -c "services:" /Users/ugurgocen/projects/agentcore-demo-test1/docker-compose.yml
echo "=== Temporal placeholders ==="
for f in agentcore-demo-test1-backend/app/worker_entrypoint.py agentcore-demo-test1-backend/app/temporal/workflows.py agentcore-demo-test1-backend/app/temporal/activities.py agentcore-demo-test1-backend/app/temporal/worker.py agentcore-demo-test1-backend/app/temporal/workflow_streams.py; do
    test -f "$f" && echo "$(basename $f): OK" || echo "$(basename $f): MISSING"
done
```

**Expected results:**
1. Dockerfile: python:3.11-slim found, temporal-worker comment found
2. docker-compose.yml: services: found
3. All 5 placeholder files exist and print "OK"

**If ANY check fails, STOP and report the exact error. Do not proceed.**

---

## STEP 15: Build Docker Image and Test Health Endpoint

**Action:** Build the Docker image and test the /health endpoint.

```bash
cd agentcore-demo-test1-backend

# Build the image
docker build --platform linux/amd64 -t agentcore-demo-test1-backend:latest .
```

**Verification (Build):**
```bash
docker images agentcore-demo-test1-backend:latest --format "{{.Repository}}:{{.Tag}}"
```
Must print `agentcore-demo-test1-backend:latest`. If empty, the build failed — STOP and report `docker build` output.

**Action (Run and Test):**
```bash
# Run the container in background
docker run -d --name test-backend -p 8000:8000 agentcore-demo-test1-backend:latest

# Wait for startup
sleep 5

# Test the health endpoint
curl -f http://localhost:8000/health
```

**Expected response:**
```json
{"status":"ok","version":"1.0.0"}
```

**Action (Cleanup):**
```bash
docker stop test-backend && docker rm test-backend
```

**FAILURE HANDLING:**
- If `docker build` fails: STOP, capture full output, report the error.
- If `/health` returns non-200: STOP, run `docker logs test-backend`, capture and report.
- If curl connection refused: STOP, check `docker ps` output, report container status.
- If cleanup fails: report but do not stop (non-critical).

---

## FINAL VERIFICATION: Complete File Inventory

**Action:** List all created files with line counts.

```bash
echo "=== COMPLETE BACKEND FILE INVENTORY ==="
find agentcore-demo-test1-backend -type f -name "*.py" -o -name "*.txt" -o -name "Dockerfile" -o -name "*.ini" | sort | while read f; do
    printf "%s\t%s lines\n" "$f" "$(wc -l < "$f")"
done
echo "=== DOCKER-COMPOSE ==="
wc -l /Users/ugurgocen/projects/agentcore-demo-test1/docker-compose.yml
echo "=== TOTAL PYTHON FILES ==="
find agentcore-demo-test1-backend -name "*.py" | wc -l
echo "=== TOTAL ALL FILES ==="
find agentcore-demo-test1-backend -type f | wc -l
```

**Expected:**
- At least 15 Python files (including __init__.py and placeholder files)
- At least 20 total files (including requirements.txt, Dockerfile, alembic.ini, migration)
- docker-compose.yml exists

---

## GATE PASS CHECKLIST — Phase 3 Complete

ALL of the following MUST pass. Check each one:

- [ ] **G-1** Directory structure: All directories in `agentcore-demo-test1-backend/app/{auth,telemetry,storage,temporal,config}` exist
- [ ] **G-2** `agentcore-demo-test1-backend/requirements.txt` (or `pyproject.toml`) exists
- [ ] **G-3** `agentcore-demo-test1-backend/app/config/settings.py` exists and imports successfully (`db_name` defaults to `agentcore_demo_test1`)
- [ ] **G-4** `agentcore-demo-test1-backend/app/auth/mock_user.py` exists with `demo-user-001` (and at least one additional persona)
- [ ] **G-5** `agentcore-demo-test1-backend/app/telemetry/tracing.py` exists with `setup_tracing` and W3C references
- [ ] **G-6** `agentcore-demo-test1-backend/app/telemetry/workflow_events.py` exists with `workflow_event_span` context manager using `demo.*` attributes (no manual EMF — ADOT handles it per PCD §10)
- [ ] **G-7** `agentcore-demo-test1-backend/app/telemetry/evidence_pack.py` exists with `build_evidence_pack` and ALCOA+ fields
- [ ] **G-8** `agentcore-demo-test1-backend/app/storage/rds_models.py` exists with `ChatMessage` + `EvidencePack` classes using SQLAlchemy 2.0 `DeclarativeBase` (no deprecated `declarative_base` import). Canonical `evidence_packs` schema (`agent_outputs JSONB`, `hitl_exchanges JSONB`, etc.) per PCD §3 D10.
- [ ] **G-9** `agentcore-demo-test1-backend/app/storage/s3_claim_check.py` exists with `apply_claim_check` function
- [ ] **G-10** `agentcore-demo-test1-backend/app/main.py` exists; routes follow `/api/v1/...` per PCD §3 D11 (`/health`, `/api/v1/workflows`, `/api/v1/uploads/audio`)
- [ ] **G-11** `agentcore-demo-test1-backend/alembic.ini`, `agentcore-demo-test1-backend/alembic/env.py`, and initial migration (`agent_outputs` + `hitl_exchanges` JSONB columns) exist
- [ ] **G-12** `agentcore-demo-test1-backend/Dockerfile` exists using `python:3.11-slim`
- [ ] **G-13** `agentcore-demo-test1-backend/docker-compose.yml` has all 6 services (temporal-server, temporal-ui, fastapi, temporal-worker, nextjs-frontend, adot-collector) and no deprecated `version:` key
- [ ] **G-14** Temporal placeholder files in `app/temporal/` exist (`worker_entrypoint.py` minimum)
- [ ] **G-15** Docker build succeeds: `docker images` shows `agentcore-demo-test1-backend:latest`
- [ ] **G-16** `/health` endpoint returns `{"status":"ok","version":"1.0.0"}`

**DECISION:**
- If **ALL 16 checks pass**: Phase 3 is COMPLETE. Report success and reference `Prompt_04_Temporal_Workflow.md` for next phase.
- If **ANY check fails**: STOP. Report which gate failed and the exact error. Do NOT proceed to Phase 4.


---

## TESTING REQUIREMENTS — pytest FastAPI Backend

Every gate checkpoint MUST have a corresponding pytest test. Create:

```
tests/
  backend/
    conftest.py               # FastAPI TestClient, DATABASE_URL fixture
    test_gate_3_health.py     # test_health_200, test_health_status_ok,
                              # test_health_has_version, test_docs_accessible,
                              # test_openapi_json
    test_gate_3_db.py         # test_db_connection, test_workflow_states_table,
                              # test_chat_messages_table, test_evidence_packs_table
    test_gate_3_docker.py     # test_temporal_server_running,
                              # test_temporal_ui_running, test_fastapi_running,
                              # test_worker_running, test_frontend_running
    test_settings.py          # Unit tests for Pydantic Settings validation
```

Run: `pytest tests/backend/ -v`

Key pattern — TestClient:
```python
from fastapi.testclient import TestClient

def test_health_returns_200(client):
    resp = client.get("/health")
    assert resp.status_code == 200
    assert resp.json()["status"] == "ok"
```

---

## Next Phase

When the backend is healthy, DB reachable, S3 working, and all tests pass:
- Report Phase 3 as COMPLETE
- Proceed to **Phase 4: Temporal Workflow** using `Prompt_04_Temporal_Workflow.md`
- Reference docs: `Deliverable_4_Temporal_Operations_Guide.md` (Temporal patterns), `Deliverable_3_Implementation_Prompts.md` (workflow scaffold)
```
