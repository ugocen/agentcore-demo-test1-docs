# Deliverable 4: Temporal Operations & Workflow Design Guide

> **Purpose:** Implementation aid for the AgentCore demo test 1 demo and production blueprint for future Temporal projects.
>
> **Stack:** Self-hosted Temporal (docker-compose, EC2 `t3.large`), PostgreSQL on RDS, Python SDK, FastAPI SSE bridge, Next.js frontend.
>
> **V5 (2026-05-12):** A2A communication via Temporal signals. HITL = normal operating mode (every turn = step). Max 2 self-corrections before HITL escalation. ContinueAsNew for >24h or >50% event history. Agent 3 uses LangGraph StateGraph (not Strands).

---

## Table of Contents

1. [Self-Hosted Temporal Architecture](#1-self-hosted-temporal-architecture)
2. [Worker Configuration](#2-worker-configuration)
3. [Workflow Design Patterns](#3-workflow-design-patterns)
4. [Signals, Updates, and Queries](#4-signals-updates-and-queries)
5. [Signal Handling](#5-signal-handling)
6. [Workflow Streams](#6-workflow-streams)
7. [Determinism and Replay](#7-determinism-and-replay)
8. [Retry Policies and Timeouts](#8-retry-policies-and-timeouts)
9. [Persistence and RDS](#9-persistence-and-rds)
10. [Observability and Web UI](#10-observability-and-web-ui)
11. [Production Hardening](#11-production-hardening)
12. [Troubleshooting Runbook](#12-troubleshooting-runbook)
13. [Tech Reference](#13-tech-reference)

---

## 1. Self-Hosted Temporal Architecture

### 1.1 Auto-Setup Internals

The `temporalio/auto-setup` image bundles four services + schema bootstrap:

| Service | Purpose |
|---------|---------|
| **Frontend** | gRPC (port 7233) — entrypoint for clients/workers |
| **History** | Persists workflow event history to PostgreSQL |
| **Matching** | Assigns tasks to workers by task queue |
| **Internal Worker** | System workflows (namespace mgmt, archival) |
| **Schema Setup** | Creates tables on first boot; skips if exists |

Two logical databases on RDS: `temporal` (history + state) and `temporal_visibility` (indexed query attributes).

### 1.2 Docker-Compose Topology

```yaml
services:
  temporal-server:
    image: temporalio/auto-setup:1.24.2
    environment:
      - DB=postgresql
      - POSTGRES_USER=${DB_USER}
      - POSTGRES_PWD=${DB_PASSWORD}
      - POSTGRES_SEEDS=${DB_HOST}
      - DBNAME=temporal
      - VISIBILITY_DBNAME=temporal_visibility
    ports: ["7233:7233"]

  temporal-ui:
    image: temporalio/ui:2.28.0
    environment:
      - TEMPORAL_ADDRESS=temporal-server:7233
    ports: ["8081:8080"]   # NOTE: 8081 host (AgentCore uses 8080)

  temporal-worker:
    build: .
    command: ["python", "-m", "worker"]
    environment:
      - TEMPORAL_HOST=temporal-server:7233

  fastapi:
    build: .
    command: ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
    ports: ["8000:8000"]
```

**Gotchas:**
- `temporal-ui` uses host port `8081` (AgentCore collision on `8080`)
- Worker connects to `temporal-server:7233` (internal docker DNS, not localhost)
- Worker and FastAPI share the same image; only `CMD` differs
- `auto-setup` creates schema on first boot, skips on restart

### 1.3 Demo Tuning

45 runs over 3 days (~0.35/hour peak). Defaults are sufficient. Key tuning: **15-minute signal timeout** per HITL round.

```bash
# Verify stack
docker compose exec temporal-server temporal operator cluster health
temporal task-queue describe --task-queue brd-task-queue --namespace default
temporal workflow list --namespace default --limit 20
```

---

## 2. Worker Configuration

### 2.1 Worker Code

```python
# worker.py
import asyncio
from temporalio.worker import Worker
from temporalio.client import Client
from workflows.brd_workflow import BRDWorkflow
from activities.brd_activities import (
    analyze_document_activity, generate_draft_activity,
    assemble_evidence_activity, notify_ui_activity, collect_requirements_activity,
)

async def main():
    client = await Client.connect("temporal-server:7233", namespace="default")
    worker = Worker(
        client,
        task_queue="brd-task-queue",
        workflows=[BRDWorkflow],
        activities=[
            analyze_document_activity, generate_draft_activity,
            assemble_evidence_activity, notify_ui_activity,
            collect_requirements_activity,
        ],
        max_concurrent_activities=10,
        max_concurrent_workflow_tasks=5,
        max_cached_workflows=10,
    )
    await worker.run()

if __name__ == "__main__":
    asyncio.run(main())
```

| Parameter | Demo | Rationale |
|-----------|------|-----------|
| `max_concurrent_activities` | 10 | Low concurrency; reduces memory |
| `max_concurrent_workflow_tasks` | 5 | Minimal workflow churn |
| `max_cached_workflows` | 10 | Hot workflows for signal targets |

### 2.2 Registration Mismatch = #1 Failure

Worker must register **every** workflow type and activity. Mismatch → `WorkflowNotFoundError`.

```python
# Client (FastAPI) — must match worker exactly
handle = await client.start_workflow(
    BRDWorkflow.run,                    # Must be in worker's workflows=[...]
    args=(uploaded_file_url,),
    id=f"brd-{uuid4()}",
    task_queue="brd-task-queue",        # Must match worker's task_queue
)
```

### 2.3 Health Verification

```bash
docker compose logs temporal-worker | grep -i "polling\|registered"
temporal task-queue describe --task-queue brd-task-queue --namespace default
temporal workflow start --type BRDWorkflow --task-queue brd-task-queue \
    --input '{"file_url": "s3://test/test.txt"}' --workflow-id test-health
temporal workflow describe --workflow-id test-health
```

---

## 3. Workflow Design Patterns

### 3.1 BRD Pipeline Flow

```
[Start] -> [Analyze Audio] -> [HITL Loop (max 5 rounds, 15min/round)]
  -> [Generate Draft] -> [Approval Gate] -> [Evidence Pack] -> [End]
```

### 3.2 Complete Annotated Workflow

```python
# workflows/brd_workflow.py
from dataclasses import dataclass, field
from datetime import timedelta
from typing import List, Dict, Optional
from temporalio import workflow
from temporalio.common import RetryPolicy

with workflow.unsafe.imports_passed_through():
    from activities.brd_activities import (
        analyze_document_activity, generate_draft_activity,
        assemble_evidence_activity, notify_ui_activity,
        collect_requirements_activity,
    )

MAX_CLARIFICATION_ROUNDS = 5
SIGNAL_TIMEOUT = timedelta(minutes=15)

@dataclass
class BRDState:
    """Mutable state — Temporal checkpoints after each completion."""
    file_url: str
    extracted_requirements: List[str] = field(default_factory=list)
    clarifications: List[Dict] = field(default_factory=list)
    clarification_round: int = 0
    draft_content: Optional[str] = None
    final_decision: Optional[str] = None   # "approve"|"reject"|"revise"
    evidence_pack: Optional[str] = None
    status: str = "RUNNING"

@workflow.defn
class BRDWorkflow:
    def __init__(self) -> None:
        self._state: Optional[BRDState] = None
        self._clarification_response: Optional[Dict] = None
        self._final_decision_signal: Optional[str] = None

    # -- QUERY: Read-only state for UI polling --
    @workflow.query
    def get_state(self) -> dict:
        return {
            "status": self._state.status if self._state else "INITIALIZING",
            "round": self._state.clarification_round if self._state else 0,
            "draft_preview": (self._state.draft_content[:500] + "...")
            if self._state and self._state.draft_content else None,
        }

    # -- SIGNAL: Human clarification answer --
    @workflow.signal
    def clarification_response(self, response: Dict) -> None:
        if self._clarification_response is not None:
            workflow.logger.warning("Duplicate clarification signal ignored")
            return
        self._clarification_response = response

    # -- SIGNAL: Stakeholder approve/reject/revise --
    @workflow.signal
    def final_decision(self, decision: str) -> None:
        self._final_decision_signal = decision

    # ================================================================
    @workflow.run
    async def run(self, file_url: str) -> Dict:
        self._state = BRDState(file_url=file_url)

        # Phase 1: Extract requirements
        self._state.status = "ANALYZING"
        requirements = await workflow.execute_activity(
            analyze_document_activity, args=(file_url,),
            start_to_close_timeout=timedelta(minutes=5),
            retry_policy=RetryPolicy(
                maximum_attempts=3,
                non_retryable_error_types=["ValueError", "HTTPError"],
            ),
        )
        self._state.extracted_requirements = requirements

        # Phase 2: HITL Clarification Loop (max 5 rounds)
        self._state.status = "CLARIFYING"
        while self._state.clarification_round < MAX_CLARIFICATION_ROUNDS:
            question = await workflow.execute_activity(
                collect_requirements_activity,
                args=(self._state.clarifications, requirements),
                start_to_close_timeout=timedelta(minutes=3),
            )
            await workflow.execute_activity(
                notify_ui_activity,
                args=({"type": "clarification_needed", "question": question,
                       "round": self._state.clarification_round + 1},),
                start_to_close_timeout=timedelta(seconds=30),
            )
            # Wait for human signal (15 min timeout)
            self._clarification_response = None
            await workflow.wait_condition(
                lambda: self._clarification_response is not None,
                timeout=SIGNAL_TIMEOUT,
            )
            if self._clarification_response is None:
                self._state.status = "TIMEOUT"
                return {"status": "TIMEOUT"}
            self._state.clarifications.append({
                "round": self._state.clarification_round,
                "question": question,
                "answer": self._clarification_response["answer"],
            })
            self._state.clarification_round += 1
            if self._clarification_response.get("done", False):
                break

        # Phase 3: Generate BRD Draft
        self._state.status = "DRAFTING"
        self._state.draft_content = await workflow.execute_activity(
            generate_draft_activity,
            args=(self._state.clarifications, requirements),
            start_to_close_timeout=timedelta(minutes=5),
        )

        # Phase 4: Review / Approval Gate
        self._state.status = "REVIEWING"
        self._final_decision_signal = None
        await workflow.execute_activity(
            notify_ui_activity,
            args=({"type": "review_ready", "draft": self._state.draft_content},),
            start_to_close_timeout=timedelta(seconds=30),
        )
        await workflow.wait_condition(
            lambda: self._final_decision_signal is not None,
            timeout=timedelta(minutes=30),
        )
        if self._final_decision_signal == "reject":
            self._state.status = "REJECTED"
            return {"status": "REJECTED"}
        if self._final_decision_signal == "revise":
            # Per PCD App D the canonical revise→REJECTED simplification applies; the
            # user resubmits a new workflow with the feedback in the next iteration.
            self._state.status = "REJECTED"
            return {"status": "REJECTED", "revise_rationale": self._revise_rationale}

        # Phase 5: Assemble Evidence Pack
        self._state.status = "ASSEMBLING"
        self._state.evidence_pack = await workflow.execute_activity(
            assemble_evidence_activity,
            args=(self._state.draft_content, self._state.clarifications),
            start_to_close_timeout=timedelta(minutes=3),
        )
        self._state.status = "COMPLETE"
        return {"status": "COMPLETE", "evidence_pack_url": self._state.evidence_pack}
```

### 3.3 Child Workflows — When to Use

**Not used here** (linear pipeline). Use when a sub-process has independent lifecycle, separate retry/timeout needs, or is reusable across parents.

```python
child = await workflow.execute_child_workflow(
    SubWorkflow.run, args=(...),
    id=f"sub-{workflow.info().run_id}",
    parent_close_policy=ParentClosePolicy.ABANDON,
)
```

### 3.4 Continue-As-New — When to Use

**Not used here** (~75 events/run). Use before hitting the **51,200 event / 50 MB limit**:

```python
if workflow.info().get_current_history_length() > 40000:
    raise workflow.ContinueAsNewError(*workflow.info().raw_args)
```

---

## 4. Signals, Updates, and Queries

Ref: https://docs.temporal.io/encyclopedia/workflow-message-passing

| Use Case | Primitive | Why |
|----------|-----------|-----|
| Human submits clarification | **Signal** | Fire-and-forget; async; `wait_condition()` |
| Stakeholder approves/rejects | **Signal** | One-shot command; workflow awaits |
| UI polls current status | **Query** | Read-only; immediate; no side effects |
| Force-stop stuck workflow | **Terminate API** | External imperative action |
| Restart from scratch | **Start new + terminate old** | Explicit lifecycle mgmt |

**Why not Update?** Updates execute inline (blocking). HITL is naturally async — humans take minutes. Signal + `wait_condition()` fits better.

**Key rule:** `@workflow.run` methods are invoked **only** by Temporal Server on start. Signals are the correct mid-flight mechanism.

---

## 5. Signal Handling

### 5.1 Pattern: Signal + Wait

```python
@workflow.signal
def clarification_response(self, response: Dict) -> None:
    """Store payload; main loop picks it up."""
    if self._clarification_response is not None:
        workflow.logger.warning("Duplicate signal ignored")
        return
    if not isinstance(response, dict) or "answer" not in response:
        workflow.logger.error(f"Invalid payload: {response}")
        return
    self._clarification_response = response

# Main loop:
self._clarification_response = None
await workflow.wait_condition(
    lambda: self._clarification_response is not None,
    timeout=timedelta(minutes=15),
)
if self._clarification_response is None:
    return {"status": "TIMEOUT"}   # Handle gracefully
```

### 5.2 Deduplication

```python
self._processed_ids: set = set()

@workflow.signal
def clarification_response(self, response: Dict) -> None:
    sid = response.get("signal_id", str(workflow.uuid4()))
    if sid in self._processed_ids:
        workflow.logger.info(f"Signal {sid} deduplicated")
        return
    self._processed_ids.add(sid)
    self._clarification_response = response
```

The latch + `wait_condition()` pattern survives replay because the signal handler sets state that the condition checks.

---

## 6. Workflow Streams

Ref: https://docs.temporal.io/develop/python/workflows/workflow-streams

### 6.1 Setup and Publishing

```bash
uv venv .venv && source .venv/bin/activate
uv pip install "temporalio[workflow-streams]>=1.6.0"
```

```python
from temporalio.contrib.workflow_streams import WorkflowStreamPublisher

publisher = WorkflowStreamPublisher("brd-stream", publisher_ttl=timedelta(hours=1))

# From workflow:
await publisher.publish({
    "event_type": "phase_change",
    "phase": "CLARIFYING",
    "workflow_id": workflow.info().workflow_id,
    "timestamp": workflow.now().isoformat(),
})
```

### 6.2 Publishing from Activities (Dedup on Retries)

```python
from temporalio import activity
from temporalio.contrib.workflow_streams import WorkflowStreamPublisher

publisher = WorkflowStreamPublisher("brd-stream")

@activity.defn
async def notify_ui_activity(payload: dict) -> None:
    if activity.info().attempt > 1:
        return    # Skip duplicate on retry
    await publisher.publish({
        **payload,
        "workflow_id": activity.info().workflow_id,
    })
```

### 6.3 FastAPI SSE Bridge

```python
# api/sse.py
from fastapi import APIRouter
from fastapi.responses import StreamingResponse
from temporalio.contrib.workflow_streams import WorkflowStreamSubscriber
import json

router = APIRouter()

@router.get("/api/workflows/{workflow_id}/events")
async def workflow_events(workflow_id: str):
    subscriber = WorkflowStreamSubscriber("brd-stream")
    async def gen():
        async for event in subscriber.subscribe(workflow_id=workflow_id):
            yield f"data: {json.dumps(event)}\n\n"
        yield f"data: {json.dumps({'type': 'stream_end'})}\n\n"
    return StreamingResponse(gen(), media_type="text/event-stream",
        headers={"Cache-Control": "no-cache", "X-Accel-Buffering": "no"})
```

---

## 7. Determinism and Replay

A workflow is deterministic if **replaying event history produces identical commands**. Temporal uses this to recover after worker crashes.

| Forbidden | Replacement | Why |
|-----------|-------------|-----|
| `datetime.now()` | `workflow.now()` | Fixed to history timestamp |
| `random.random()` | `workflow.random()` | Seeded from history |
| `uuid4()` | `workflow.uuid4()` | Deterministic generation |
| I/O (HTTP, DB, files) | Activities | Side effects captured in history |
| `time.sleep()` | `workflow.sleep()` | Temporal-managed timer |
| `threading.Thread` | Async + activities | Non-deterministic scheduling |

```python
# WRONG
from datetime import datetime
import random
now = datetime.now()                # BAD: changes on replay
delay = random.randint(1, 10)       # BAD
await asyncio.sleep(delay)          # BAD: blocks event loop

# RIGHT
from temporalio import workflow
now = workflow.now()                # GOOD
delay = workflow.random().randint(1, 10)  # GOOD
await workflow.sleep(delay)         # GOOD
```

### Versioning for Code Changes

```python
if workflow.patched("new-clarification-logic"):
    result = await workflow.execute_activity(new_activity, args=(...))
else:
    result = await workflow.execute_activity(old_activity, args=(...))
```

Deploy with patch → let old workflows complete → remove patch next deploy.

---

## 8. Retry Policies and Timeouts

**Default policy retries forever.** Always set `maximum_attempts` and `non_retryable_error_types`.

```python
from temporalio.common import RetryPolicy

# Document analysis — bad input shouldn't retry forever
retry_analysis = RetryPolicy(
    maximum_attempts=3,
    non_retryable_error_types=["ValueError", "HTTPError", "FileNotFoundError"],
)

# UI notification — non-critical, brief retry
retry_notify = RetryPolicy(
    maximum_attempts=5,
    initial_interval=timedelta(seconds=2),
    maximum_interval=timedelta(seconds=10),
)
```

### Timeout Selection

| Type | Use For | Demo |
|------|---------|------|
| `start_to_close_timeout` | Max time for one activity attempt | 3-5 min |
| `schedule_to_close_timeout` | Max total including retries | 10 min |
| `schedule_to_start_timeout` | Max wait for worker capacity | 30 sec |

```python
await workflow.execute_activity(
    analyze_document_activity, args=(file_url,),
    start_to_close_timeout=timedelta(minutes=5),
    retry_policy=RetryPolicy(maximum_attempts=3),
)
```

### Non-Retryable: AgentCore 4xx

```python
@activity.defn
async def call_agent_core(payload: dict) -> dict:
    async with httpx.AsyncClient() as client:
        resp = await client.post("http://agentcore:8080/invocations", json=payload)
        if 400 <= resp.status_code < 500:
            raise ValueError(f"Non-retryable HTTP {resp.status_code}")
        resp.raise_for_status()
        return resp.json()
```

`ValueError` in `non_retryable_error_types` → Temporal fails immediately.

---

## 9. Persistence and RDS

### 9.1 Key Tables

**`temporal` database:** `executions`, `history_node` (append-only event log), `history_tree`, `transfer_tasks`, `timer_tasks`.

**`temporal_visibility` database:** `executions_visibility` (indexed query attributes).

### 9.2 Event Growth

~75 events per BRD run (1 start + 5 rounds x ~8 events + 5 activities x 3 + final signal). At 45 runs: **~3,375 events** — well under limits.

### 9.3 The 50 MB / 51,200 Events Limit

When hit, the workflow **cannot progress**. Use **Claim-Check** for large payloads:

```python
# GOOD: pass reference, load in activity
@activity.defn
async def process_document(ref: str) -> str:
    doc = await httpx.AsyncClient().get(f"http://storage:9000/{ref}")
    return doc.text

await workflow.execute_activity(
    process_document, args=("s3://bucket/brd-123.txt",),
)
```

### 9.4 Schema Migrations on Upgrades

```bash
# BEFORE upgrading server image:
docker compose exec temporal-server temporal-sql-tool \
    --plugin postgres --ep ${DB_HOST} -p 5432 -u ${DB_USER} \
    --pw ${DB_PASSWORD} --db temporal \
    update-schema -d /etc/temporal/schema/postgresql/v12/temporal/versioned
docker compose up -d temporal-server
```

---

## 10. Observability and Web UI

### 10.1 Web UI

```
URL: http://<ec2-public-ip>:8081
```

| Tab | Use |
|-----|-----|
| Workflows | Filter by status, type, time range |
| Summary | Status, input/output, timing |
| History | Full event log (activities, signals, timers) |
| Stack Trace | Current await point |
| Query | Execute custom queries (`get_state`) |

### 10.2 Essential CLI

```bash
temporal workflow start  --type BRDWorkflow --task-queue brd-task-queue \
    --input '{"file_url": "s3://bucket/audio.mp3"}' --workflow-id brd-001
temporal workflow list   --namespace default --limit 20
temporal workflow describe --workflow-id brd-001
temporal workflow query  --workflow-id brd-001 --type get_state
temporal workflow signal --workflow-id brd-001 --name clarification_response \
    --input '{"answer": "Yes, multi-language"}'
temporal workflow terminate --workflow-id brd-001 --reason "Cancelled"
temporal task-queue describe --task-queue brd-task-queue --namespace default
temporal operator cluster health
```

---

## 11. Production Hardening

| Item | Demo | Production |
|------|------|------------|
| Compute | Single t3.large | Multi-AZ ASG |
| Database | Single RDS | Multi-AZ + replicas |
| Temporal Server | Single container | 2+ behind NLB |
| Workers | Shared with FastAPI | Dedicated ASG |
| Schema | auto-setup bundled | Separate migration job |
| Transport | Plain gRPC | mTLS |
| Auth | None | OIDC/API-key |
| Monitoring | Basic | Prometheus + alerts |

```yaml
# Separate migration in production
services:
  temporal-schema-setup:
    image: temporalio/admin-tools:1.24.2
    command: ["temporal-sql-tool", "setup-schema", "-y"]
    restart: "no"
  temporal-server:
    image: temporalio/server:1.24.2
    depends_on:
      temporal-schema-setup: {condition: service_completed_successfully}
```

---

## 12. Troubleshooting Runbook

| Issue | Symptoms | Diagnosis | Fix |
|-------|----------|-----------|-----|
| **Registration mismatch** | "No worker available" | `task-queue describe` shows no pollers | Match `task_queue` name; include workflow in `workflows=[...]` |
| **Signals lost** | Workflow running, never processes signal | Signal sent before `wait_condition()` entered | Add delay before sending; ensure handler in `__init__` |
| **Workflow stuck** | "Running" >15 min | `describe` shows pending signal | Re-send signal or terminate |
| **History bloat** | Slow queries; `HistoryLength` >10K | `workflow describe` | Claim-Check for large payloads; Continue-As-New before 40K |
| **Deserialization fail** | `TypeError` after code change | History references old schema | Use `workflow.patched()`; add optional fields with defaults |
| **Activity timeout but running** | "Attempt 2" but first still going | No heartbeat | Add `activity.heartbeat()` to long activities |

```bash
# General diagnostics
temporal workflow describe --workflow-id <id>
temporal workflow query --workflow-id <id> --type get_state
temporal workflow history --workflow-id <id> | head -50
docker compose logs -f temporal-worker
```

---

## 13. Tech Reference

### 13.1 Resources

| Topic | URL |
|-------|-----|
| Temporal Docs | https://docs.temporal.io |
| Python SDK | https://docs.temporal.io/develop/python |
| API Reference | https://python.temporal.io |
| Self-Hosted Guide | https://docs.temporal.io/self-hosted-guide |
| docker-compose | https://github.com/temporalio/docker-compose |
| Workflow Streams | https://docs.temporal.io/develop/python/workflows/workflow-streams |
| HITL Tutorial | https://learn.temporal.io/tutorials/ai/building-durable-ai-applications/human-in-the-loop/ |
| Message Passing | https://docs.temporal.io/encyclopedia/workflow-message-passing |

### 13.2 Version Pins

| Component | Version |
|-----------|---------|
| Temporal Server | `temporalio/auto-setup:1.24.2` |
| Temporal UI | `temporalio/ui:2.28.0` |
| Python SDK | `>=1.6.0` |
| PostgreSQL | `15.x` (RDS) |

### 13.3 Project Gotchas

- **Port 8081**: UI on `8081` (AgentCore uses `8080`)
- **Event limits**: 50 MB / 51,200 events per workflow → drives Claim-Check
- **Worker hostname**: `temporal-server:7233` (internal docker DNS)
- **Shared image**: Worker + FastAPI same image, different `CMD`
- **Auto-setup**: Creates schema on first boot, skips on restart
- **Determinism**: `workflow.now()` not `datetime.now()`
- **Two databases**: RDS hosts `temporal` + `temporal_visibility`

---

> **End of Deliverable 4: Temporal Operations & Workflow Design Guide**
>
> References: https://docs.temporal.io | https://python.temporal.io
