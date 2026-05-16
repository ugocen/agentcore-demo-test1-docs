# Prompt 04: Temporal Workflow + Workflow Streams (Phase 4)

**MODE: AUTOMATIC — AI writes code directly, sequential execution, every step verified.**

## Reference Documents (READ FIRST)

| Priority | Document | Why |
|----------|----------|-----|
| **PRIMARY** | `resources/Deliverable_0_PROJECT_CONTEXT.md` (PCD) | Frozen decisions D1–D14, identifier hierarchy §7, Workflow Streams strategy §11 |
| **PRIMARY** | `resources/PAYLOAD_SCHEMA.md` | 8-block request/response contract; HITL re-invocation §8 |
| **PRIMARY** | `resources/Deliverable_4_Temporal_Operations_Guide.md` | Temporal patterns, signals, retry policies, ContinueAsNew |
| **REFERENCE** | `resources/Prompt_03_Backend_Foundation.md` | The canonical `evidence_packs` schema (D10) was created in Phase 3 — Phase 4 only inserts/updates rows |

---

## CONTEXT

You are executing **Phase 4** of a multi-phase build:
- **Phases 0–3 COMPLETE**: AWS infra, 3 agents deployed to AgentCore (each emitting valid 8-block payloads with `demo.*` OTel attributes), FastAPI backend running, `evidence_packs` table created via Alembic.
- **THIS PHASE**: Add Temporal workflow + activities that call the agents end-to-end. Add Workflow Stream publisher (Pattern A or B per PCD D6). Wire FastAPI routes that start workflows and forward signals.
- **Phases 5–7 NEXT**: Frontend, E2E tests, Metrics dashboard.

**Reference orchestration:** `brd-from-audio-v1`. Agents (canonical naming per PCD D4): `agent_1_transcriber`, `agent_2_drafter`, `agent_3_reviewer`.

## CRITICAL RULES (NON-NEGOTIABLE)

- **No schema creation here.** The `evidence_packs` table was created in Phase 3 (Prompt_03) with the canonical D10 schema. Phase 4 only inserts/updates rows. Do NOT add an Alembic migration for this table.
- **Agents NEVER import `temporalio`** (PCD §4 rule 10). The activity wrapper lives in the backend; agents are pure HTTP services that take an 8-block request and return an 8-block response.
- **Workflow Streams** uses the two-pattern strategy per PCD §11 / D6:
  - Pattern A (preferred): `temporalio.contrib.workflow_streams` if importable.
  - Pattern B (fallback): Workflow Update API + Redis pub/sub.
  Selection is determined at worker startup by env `WORKFLOW_STREAMS_PATTERN` (auto-detected).
- **Identifiers**: every activity invocation passes a complete inbound request per PAYLOAD_SCHEMA §2 (trace_id, workflow_template_id, workflow_id, agent_run_id, step_id, step_sequence, parent_run_id, task_id, agent_id, agent_version, requested_by, execution_context, task, inputs, execution_options, actor_id, session_id).
- **Idempotency**: same `(workflow_id, step_id, request_hash)` → same outcome. Activity wrapper enforces this.
- **Self-correction cap**: 3 total attempts (D8). After cap, mandatory `hitl_review` step regardless of agent's Status.
- **All comments in English.**

## WORKING DIRECTORY

```bash
export PROJECT_ROOT="/Users/ugurgocen/projects/agentcore-demo-test1"
cd "$PROJECT_ROOT/agentcore-demo-test1-backend"
```

---

## STEP 1: Create `app/temporal/__init__.py` and shared enums

**File:** `agentcore-demo-test1-backend/app/temporal/__init__.py`

```python
"""Temporal workflow package."""
```

**File:** `agentcore-demo-test1-backend/app/temporal/types.py`

```python
"""Shared enums and dataclasses for the Temporal layer."""
from __future__ import annotations
from dataclasses import dataclass
from enum import Enum
from typing import Any


class BRDState(str, Enum):
    """Workflow outcome enum. Matches evidence_packs.outcome column values."""
    PENDING = "PENDING"
    IN_PROGRESS = "IN_PROGRESS"
    AWAITING_HUMAN = "AWAITING_HUMAN"
    APPROVED = "APPROVED"
    REJECTED = "REJECTED"
    FAILED = "FAILED"
    MAX_ITERATIONS = "MAX_ITERATIONS"


class AgentRole(str, Enum):
    """Canonical agent roles per PCD D4."""
    TRANSCRIBER = "transcriber"
    DRAFTER = "drafter"
    REVIEWER = "reviewer"


class StepType(str, Enum):
    """Step type enum per Guideline §16 / PAYLOAD_SCHEMA §7."""
    AGENT_ACTION = "agent_action"
    HITL_QUESTION = "hitl_question"
    HITL_REVIEW = "hitl_review"
    HITL_APPROVAL = "hitl_approval"
    HITL_BRANCH_DECISION = "hitl_branch_decision"
    TOOL_CALL = "tool_call"
    CLAIM_CHECK_IO = "claim_check_io"
    REVISE = "revise"


@dataclass
class WorkflowInput:
    """Inbound payload from FastAPI to start a workflow."""
    workflow_template_id: str
    workflow_id: str
    trace_id: str
    audio_s3_uri: str
    persona: dict[str, Any]


@dataclass
class AgentInvocation:
    """Per-step activity input. Constructed by the workflow."""
    agent_role: AgentRole
    agent_runtime_arn: str
    workflow_template_id: str
    workflow_id: str
    trace_id: str
    agent_run_id: str
    step_id: str
    step_sequence: int
    parent_run_id: str
    task_id: str
    requested_by: dict[str, Any]
    execution_context: dict[str, Any]
    task: dict[str, Any]
    inputs: dict[str, Any]
    actor_id: str
    session_id: str
    human_input: dict[str, Any] | None = None
    attempt_number: int = 1
```

CHECKPOINT 1: `python -c "from app.temporal.types import BRDState, AgentRole, StepType; print('OK')"` succeeds.

---

## STEP 2: Workflow Streams (Pattern A + Pattern B fallback)

**File:** `agentcore-demo-test1-backend/app/temporal/workflow_streams.py`

```python
"""
Workflow Stream publisher with Pattern A / Pattern B fallback (PCD §11 / D6).

Pattern A (preferred): temporalio.contrib.workflow_streams.WorkflowStream
Pattern B (fallback): Redis pub/sub side channel + Temporal Update API

Active pattern is determined at module import time by attempting to import
the contrib module. Env var WORKFLOW_STREAMS_PATTERN overrides detection.
"""
from __future__ import annotations
import os
import json
import logging
from typing import Any, AsyncIterator

logger = logging.getLogger(__name__)

try:
    from temporalio.contrib.workflow_streams import WorkflowStream  # type: ignore
    _PATTERN_A_AVAILABLE = True
except ImportError:
    _PATTERN_A_AVAILABLE = False

WORKFLOW_STREAMS_PATTERN = os.environ.get(
    "WORKFLOW_STREAMS_PATTERN", "A" if _PATTERN_A_AVAILABLE else "B"
)

if WORKFLOW_STREAMS_PATTERN == "A" and not _PATTERN_A_AVAILABLE:
    logger.warning("Pattern A requested but unavailable; falling back to Pattern B.")
    WORKFLOW_STREAMS_PATTERN = "B"


# --- Pattern A ----------------------------------------------------------
async def publish_event_pattern_a(stream, event: dict[str, Any]) -> None:
    await stream.write(event)


# --- Pattern B ----------------------------------------------------------
def _redis_client():
    import redis.asyncio as redis_asyncio  # lazy import
    return redis_asyncio.from_url(
        os.environ.get("REDIS_URL", "redis://redis:6379"),
        decode_responses=True,
    )


async def publish_event_pattern_b(workflow_id: str, event: dict[str, Any]) -> None:
    client = _redis_client()
    try:
        await client.publish(f"workflow:{workflow_id}", json.dumps(event))
    finally:
        await client.aclose()


# --- Public surface -----------------------------------------------------
async def publish_workflow_event(workflow_id: str, event: dict[str, Any], stream=None) -> None:
    if WORKFLOW_STREAMS_PATTERN == "A" and stream is not None:
        await publish_event_pattern_a(stream, event)
    else:
        await publish_event_pattern_b(workflow_id, event)


async def consume_workflow_events(workflow_id: str) -> AsyncIterator[str]:
    """Yield SSE-formatted strings as workflow events arrive."""
    if WORKFLOW_STREAMS_PATTERN == "A":
        from temporalio.client import Client
        client = await Client.connect(os.environ.get("TEMPORAL_HOST", "temporal-server:7233"))
        handle = client.get_workflow_handle(workflow_id)
        async for event in handle.read_stream("status"):  # type: ignore[attr-defined]
            yield f"data: {json.dumps(event)}\n\n"
    else:
        client = _redis_client()
        pubsub = client.pubsub()
        try:
            await pubsub.subscribe(f"workflow:{workflow_id}")
            async for message in pubsub.listen():
                if message["type"] == "message":
                    yield f"data: {message['data']}\n\n"
        finally:
            await pubsub.unsubscribe()
            await pubsub.close()
            await client.aclose()
```

CHECKPOINT 2: `python -c "from app.temporal.workflow_streams import WORKFLOW_STREAMS_PATTERN; print('Pattern:', WORKFLOW_STREAMS_PATTERN)"` prints `A` or `B`.

---

## STEP 3: Activities (`invoke_agent_runtime_activity` + persistence)

**File:** `agentcore-demo-test1-backend/app/temporal/activities.py`

```python
"""
Temporal activities that wrap agent invocations and persist evidence.

Single generic activity invoke_agent_runtime_activity is used for ALL 3 agents
(per PCD D4); the role is passed as part of the AgentInvocation dataclass.
"""
from __future__ import annotations
import os
import json
import hashlib
from datetime import datetime, timezone
import boto3
from temporalio import activity
from app.temporal.types import AgentInvocation, AgentRole
from app.temporal.workflow_streams import publish_workflow_event
from app.storage.rds_models import EvidencePack, get_db


def _agentcore_client():
    """Lazy boto3 client; created inside the activity, not in workflow code."""
    return boto3.client("bedrock-agentcore-runtime",
                        region_name=os.environ.get("AWS_REGION", "us-east-1"))


REQUIRED_BLOCKS = {"status", "resources", "timing", "financial",
                   "artifacts", "quality", "tool_calls", "risk"}


def _validate_8block(payload: dict) -> None:
    missing = REQUIRED_BLOCKS - set(payload.keys())
    if missing:
        raise ValueError(f"Response missing blocks: {sorted(missing)}")
    status = payload.get("status", {})
    for key in ("trace_id", "workflow_template_id", "workflow_id",
                "agent_run_id", "step_id"):
        if not status.get(key):
            raise ValueError(f"Status.{key} not echoed by agent")


def _request_hash(req: dict) -> str:
    return hashlib.sha256(json.dumps(req, sort_keys=True).encode()).hexdigest()


@activity.defn
async def invoke_agent_runtime_activity(inv: AgentInvocation) -> dict:
    """Invoke an AgentCore agent and return its full 8-block response."""
    agent_id_map = {AgentRole.TRANSCRIBER: "agent_1_transcriber",
                    AgentRole.DRAFTER: "agent_2_drafter",
                    AgentRole.REVIEWER: "agent_3_reviewer"}
    request = {
        "trace_id": inv.trace_id,
        "workflow_template_id": inv.workflow_template_id,
        "workflow_id": inv.workflow_id,
        "agent_run_id": inv.agent_run_id,
        "step_id": inv.step_id,
        "step_sequence": inv.step_sequence,
        "parent_run_id": inv.parent_run_id,
        "task_id": inv.task_id,
        "agent_id": agent_id_map[inv.agent_role],
        "agent_version": "1.0.0",
        "requested_by": inv.requested_by,
        "execution_context": inv.execution_context,
        "task": {**inv.task, "attempt_number": inv.attempt_number},
        "inputs": inv.inputs,
        "execution_options": {
            "dry_run": False, "require_human_approval": True,
            "max_attempts": int(os.environ.get("SELF_CORRECTION_CAP", "3")),
            "timeout_seconds": 900, "quality_threshold": 0.85,
        },
        "actor_id": inv.actor_id,
        "session_id": inv.session_id,
        "human_input": inv.human_input,
        "human_approved": (inv.human_input.get("decision") == "approve") if inv.human_input else None,
    }
    activity.logger.info(
        "Invoking agent",
        extra={"agent_role": inv.agent_role.value, "workflow_id": inv.workflow_id,
               "step_id": inv.step_id, "attempt": inv.attempt_number,
               "request_hash": _request_hash(request)[:16]},
    )

    client = _agentcore_client()
    resp = client.invoke_agent_runtime(
        agentRuntimeArn=inv.agent_runtime_arn,
        runtimeSessionId=inv.session_id,
        traceParent=inv.trace_id,
        qualifier="DEFAULT",
        payload=json.dumps(request).encode(),
    )
    body = b"".join(chunk for chunk in resp.get("response", []))
    payload = json.loads(body.decode())
    _validate_8block(payload)

    await publish_workflow_event(inv.workflow_id, {
        "event_type": "agent_completed",
        "agent_role": inv.agent_role.value,
        "step_id": inv.step_id,
        "agent_run_id": inv.agent_run_id,
        "status_code": payload["status"]["code"],
        "ts": datetime.now(timezone.utc).isoformat(),
    })
    return payload


@activity.defn
async def persist_agent_output_activity(workflow_id: str, agent_role: str,
                                        payload: dict) -> None:
    """Append the agent's 8-block output to evidence_packs.agent_outputs."""
    with get_db() as session:
        pack = session.query(EvidencePack).filter_by(workflow_id=workflow_id).one_or_none()
        if pack is None:
            return
        outputs = dict(pack.agent_outputs or {})
        outputs.setdefault(agent_role, []).append(payload)
        pack.agent_outputs = outputs
        pack.rounds_executed = (pack.rounds_executed or 0) + 1
        session.commit()


@activity.defn
async def append_hitl_exchange_activity(workflow_id: str, exchange: dict) -> None:
    """Append a HITL exchange to evidence_packs.hitl_exchanges."""
    with get_db() as session:
        pack = session.query(EvidencePack).filter_by(workflow_id=workflow_id).one_or_none()
        if pack is None:
            return
        exchanges = list(pack.hitl_exchanges or [])
        exchanges.append({**exchange, "ts": datetime.now(timezone.utc).isoformat()})
        pack.hitl_exchanges = exchanges
        session.commit()


@activity.defn
async def initialize_evidence_pack_activity(workflow_id: str,
                                            workflow_template_id: str) -> None:
    """Create the evidence_packs row at workflow start. Idempotent."""
    with get_db() as session:
        existing = session.query(EvidencePack).filter_by(workflow_id=workflow_id).one_or_none()
        if existing:
            return
        pack = EvidencePack(
            workflow_id=workflow_id,
            workflow_template_id=workflow_template_id,
            outcome="PENDING",
            rounds_executed=0,
            max_rounds=int(os.environ.get("SELF_CORRECTION_CAP", "3")),
        )
        session.add(pack)
        session.commit()


@activity.defn
async def finalize_evidence_pack_activity(workflow_id: str, outcome: str,
                                          final_brd_content: str | None = None,
                                          error: str | None = None) -> None:
    """Set the terminal outcome on the evidence_packs row."""
    with get_db() as session:
        pack = session.query(EvidencePack).filter_by(workflow_id=workflow_id).one_or_none()
        if pack is None:
            return
        pack.outcome = outcome
        pack.final_brd_content = final_brd_content
        pack.error = error
        pack.completed_at = datetime.now(timezone.utc)
        session.commit()
```

CHECKPOINT 3: All 5 activity functions import without errors.

---

## STEP 4: The workflow (`BrdFromAudioWorkflow`)

**File:** `agentcore-demo-test1-backend/app/temporal/workflows.py`

```python
"""BRD-from-audio reference workflow per PCD §2.4 and §9."""
from __future__ import annotations
import os
import uuid
from datetime import timedelta
from temporalio import workflow
from temporalio.common import RetryPolicy

with workflow.unsafe.imports_passed_through():
    from app.temporal.types import AgentInvocation, AgentRole, BRDState, WorkflowInput
    from app.temporal.activities import (
        invoke_agent_runtime_activity,
        persist_agent_output_activity,
        append_hitl_exchange_activity,
        initialize_evidence_pack_activity,
        finalize_evidence_pack_activity,
    )


HITL_TIMEOUT = timedelta(seconds=int(os.environ.get("HITL_TIMEOUT_SECONDS", "900")))
WORKFLOW_TIMEOUT = timedelta(seconds=int(os.environ.get("WORKFLOW_TIMEOUT_SECONDS", "3600")))
SELF_CORRECTION_CAP = int(os.environ.get("SELF_CORRECTION_CAP", "3"))

DEFAULT_RETRY = RetryPolicy(
    initial_interval=timedelta(seconds=2),
    maximum_interval=timedelta(seconds=30),
    maximum_attempts=3,
    non_retryable_error_types=["ValueError", "PermissionError"],
)


@workflow.defn
class BrdFromAudioWorkflow:
    def __init__(self) -> None:
        self._pending_clarification: dict | None = None
        self._pending_approval: dict | None = None
        self._cancelled: bool = False

    # --- Signal handlers --------------------------------------------------
    @workflow.signal
    def clarification(self, payload: dict) -> None:
        self._pending_clarification = payload

    @workflow.signal
    def approval(self, payload: dict) -> None:
        self._pending_approval = payload

    @workflow.signal
    def cancel(self) -> None:
        self._cancelled = True

    # --- Main entry -------------------------------------------------------
    @workflow.run
    async def run(self, wi: WorkflowInput) -> dict:
        await workflow.execute_activity(
            initialize_evidence_pack_activity,
            args=(wi.workflow_id, wi.workflow_template_id),
            start_to_close_timeout=timedelta(seconds=30),
            retry_policy=DEFAULT_RETRY,
        )
        try:
            # Step 1: Transcriber
            transcript_result = await self._call_agent_step(
                wi=wi, role=AgentRole.TRANSCRIBER, step_sequence=1,
                inputs={"artifact_refs": [wi.audio_s3_uri]},
                task={"task_type": "audio_transcription", "complexity_tier": "simple",
                      "instruction": "Transcribe the audio.", "output_format": "json",
                      "template_id": None},
            )

            # Step 2: Drafter (HITL loop with self-correction cap per D8)
            draft_result = await self._drafter_with_hitl(wi, transcript_result)
            if draft_result is None:
                return await self._finish(wi.workflow_id, BRDState.MAX_ITERATIONS,
                                          error="Drafter exhausted self-correction cap")

            # Step 3: Reviewer
            review_result, brd_for_approval = await self._reviewer(wi, draft_result)

            # Step 4: Final approval
            outcome = await self._await_approval(wi.workflow_id)
            return await self._finish(wi.workflow_id, outcome,
                                      final_brd_content=brd_for_approval)
        except Exception as exc:  # noqa: BLE001
            return await self._finish(wi.workflow_id, BRDState.FAILED, error=str(exc))

    # --- Helpers ---------------------------------------------------------
    async def _call_agent_step(self, *, wi: WorkflowInput, role: AgentRole,
                                step_sequence: int, inputs: dict, task: dict,
                                human_input: dict | None = None,
                                attempt_number: int = 1) -> dict:
        arn = {
            AgentRole.TRANSCRIBER: os.environ["AGENTCORE_ARN_TRANSCRIBER"],
            AgentRole.DRAFTER:     os.environ["AGENTCORE_ARN_DRAFTER"],
            AgentRole.REVIEWER:    os.environ["AGENTCORE_ARN_REVIEWER"],
        }[role]
        step_id = f"step-{step_sequence:03d}-{role.value}-a{attempt_number}"
        inv = AgentInvocation(
            agent_role=role, agent_runtime_arn=arn,
            workflow_template_id=wi.workflow_template_id,
            workflow_id=wi.workflow_id, trace_id=wi.trace_id,
            agent_run_id=f"run-{uuid.uuid4().hex[:8]}",
            step_id=step_id, step_sequence=step_sequence,
            parent_run_id=wi.workflow_id,
            task_id=f"TASK-{wi.workflow_id}-{step_sequence}",
            requested_by=wi.persona,
            execution_context={"environment": "dev", "data_classification": "INTERNAL"},
            task=task, inputs=inputs,
            actor_id=wi.persona.get("user_id", "Unavailable"),
            session_id=wi.workflow_id,
            human_input=human_input, attempt_number=attempt_number,
        )
        result = await workflow.execute_activity(
            invoke_agent_runtime_activity, inv,
            start_to_close_timeout=timedelta(minutes=15),
            retry_policy=DEFAULT_RETRY,
        )
        await workflow.execute_activity(
            persist_agent_output_activity,
            args=(wi.workflow_id, role.value, result),
            start_to_close_timeout=timedelta(seconds=30),
            retry_policy=DEFAULT_RETRY,
        )
        return result

    async def _drafter_with_hitl(self, wi: WorkflowInput,
                                  transcript_result: dict) -> dict | None:
        transcript_uri = transcript_result["artifacts"][0]["content_ref"]
        human_input: dict | None = None
        for attempt in range(1, SELF_CORRECTION_CAP + 1):
            result = await self._call_agent_step(
                wi=wi, role=AgentRole.DRAFTER, step_sequence=2,
                inputs={"artifact_refs": [transcript_uri]},
                task={"task_type": "brd_generation", "complexity_tier": "medium",
                      "instruction": "Draft the BRD.", "output_format": "md",
                      "template_id": "brd-template-v1"},
                human_input=human_input, attempt_number=attempt,
            )
            status = result["status"]["code"]
            if status == "success":
                return result
            if status == "requires_human_review":
                human_input = await self._await_clarification(wi.workflow_id, result)
                continue
            raise RuntimeError(f"Drafter returned status={status!r}")
        return None  # cap exhausted

    async def _reviewer(self, wi: WorkflowInput,
                         draft_result: dict) -> tuple[dict, str]:
        draft_uri = draft_result["artifacts"][0]["content_ref"]
        review = await self._call_agent_step(
            wi=wi, role=AgentRole.REVIEWER, step_sequence=3,
            inputs={"artifact_refs": [draft_uri]},
            task={"task_type": "brd_review", "complexity_tier": "medium",
                  "instruction": "Review the BRD.", "output_format": "md",
                  "template_id": "brd-review-template-v1"},
        )
        return review, draft_uri

    async def _await_clarification(self, workflow_id: str,
                                    agent_result: dict) -> dict:
        question = (agent_result["status"]
                    .get("custom_metadata", {})
                    .get("review_summary", {})
                    .get("reason", "Clarification required"))
        await workflow.execute_activity(
            append_hitl_exchange_activity,
            args=(workflow_id, {"step_type": "hitl_question",
                                "question": question, "response": None}),
            start_to_close_timeout=timedelta(seconds=30),
            retry_policy=DEFAULT_RETRY,
        )
        self._pending_clarification = None
        await workflow.wait_condition(
            lambda: self._pending_clarification is not None,
            timeout=HITL_TIMEOUT,
        )
        response = self._pending_clarification or {}
        await workflow.execute_activity(
            append_hitl_exchange_activity,
            args=(workflow_id, {"step_type": "hitl_question", "question": question,
                                "response": response.get("rationale", ""),
                                "responder_user_id": response.get("responder_user_id")}),
            start_to_close_timeout=timedelta(seconds=30),
            retry_policy=DEFAULT_RETRY,
        )
        return {"decision": "answer", "rationale": response.get("rationale", ""),
                "responder_user_id": response.get("responder_user_id")}

    async def _await_approval(self, workflow_id: str) -> BRDState:
        await workflow.execute_activity(
            append_hitl_exchange_activity,
            args=(workflow_id, {"step_type": "hitl_approval",
                                "question": "Approve BRD?", "response": None}),
            start_to_close_timeout=timedelta(seconds=30),
            retry_policy=DEFAULT_RETRY,
        )
        self._pending_approval = None
        await workflow.wait_condition(
            lambda: self._pending_approval is not None or self._cancelled,
            timeout=HITL_TIMEOUT,
        )
        if self._cancelled:
            return BRDState.REJECTED
        decision = (self._pending_approval or {}).get("decision", "reject")
        if decision == "approve":
            return BRDState.APPROVED
        return BRDState.REJECTED

    async def _finish(self, workflow_id: str, outcome: BRDState,
                       final_brd_content: str | None = None,
                       error: str | None = None) -> dict:
        await workflow.execute_activity(
            finalize_evidence_pack_activity,
            args=(workflow_id, outcome.value, final_brd_content, error),
            start_to_close_timeout=timedelta(seconds=30),
            retry_policy=DEFAULT_RETRY,
        )
        return {"workflow_id": workflow_id, "outcome": outcome.value, "error": error}
```

CHECKPOINT 4: `python -c "from app.temporal.workflows import BrdFromAudioWorkflow; print('OK')"` succeeds.

---

## STEP 5: Worker entrypoint

**File:** `agentcore-demo-test1-backend/app/temporal/worker_entrypoint.py`

```python
"""Temporal worker: registers the workflow and all activities."""
import asyncio
import os
from temporalio.client import Client
from temporalio.worker import Worker

from app.temporal.workflows import BrdFromAudioWorkflow
from app.temporal.activities import (
    invoke_agent_runtime_activity,
    persist_agent_output_activity,
    append_hitl_exchange_activity,
    initialize_evidence_pack_activity,
    finalize_evidence_pack_activity,
)
from app.telemetry.tracing import setup_tracing


async def main() -> None:
    setup_tracing("temporal-worker")
    host = os.environ.get("TEMPORAL_HOST", "temporal-server")
    port = int(os.environ.get("TEMPORAL_PORT", "7233"))
    task_queue = os.environ.get("TEMPORAL_TASK_QUEUE", "brd-from-audio-tq")

    client = await Client.connect(f"{host}:{port}")
    worker = Worker(
        client, task_queue=task_queue,
        workflows=[BrdFromAudioWorkflow],
        activities=[
            invoke_agent_runtime_activity,
            persist_agent_output_activity,
            append_hitl_exchange_activity,
            initialize_evidence_pack_activity,
            finalize_evidence_pack_activity,
        ],
    )
    print(f"[worker] connected to {host}:{port}; task_queue={task_queue}", flush=True)
    await worker.run()


if __name__ == "__main__":
    asyncio.run(main())
```

CHECKPOINT 5: `python -c "from app.temporal.worker_entrypoint import main; print('OK')"` succeeds.

---

## STEP 6: FastAPI routes (`/api/v1/workflows`)

**File:** `agentcore-demo-test1-backend/app/api/workflows_routes.py`

```python
"""FastAPI routes for the BRD-from-audio workflow (PCD §3 D11)."""
from __future__ import annotations
import os
import uuid
from datetime import datetime, timezone
from fastapi import APIRouter, HTTPException, status
from fastapi.responses import StreamingResponse
from pydantic import BaseModel, Field
from temporalio.client import Client

from app.auth.mock_user import get_mock_user
from app.temporal.types import WorkflowInput, BRDState
from app.temporal.workflow_streams import consume_workflow_events
from app.storage.rds_models import EvidencePack, get_db

router = APIRouter(prefix="/api/v1/workflows")


class StartWorkflowRequest(BaseModel):
    audio_s3_uri: str = Field(..., description="s3:// URI from /api/v1/uploads/audio")
    persona_id: str = Field("ba-sap-mm", description="Mock persona key (D9)")


class StartWorkflowResponse(BaseModel):
    workflow_id: str
    workflow_template_id: str
    trace_id: str
    outcome: str


async def _temporal_client() -> Client:
    host = os.environ.get("TEMPORAL_HOST", "temporal-server")
    port = int(os.environ.get("TEMPORAL_PORT", "7233"))
    return await Client.connect(f"{host}:{port}")


@router.post("", response_model=StartWorkflowResponse, status_code=status.HTTP_202_ACCEPTED)
async def start_workflow(req: StartWorkflowRequest) -> StartWorkflowResponse:
    persona = get_mock_user(req.persona_id)
    now = datetime.now(timezone.utc)
    workflow_id = f"wf-{now.strftime('%Y-%m-%d')}-{uuid.uuid4().hex[:8]}"
    trace_id = f"TRC-{uuid.uuid4().hex}"
    wi = WorkflowInput(
        workflow_template_id="brd-from-audio-v1",
        workflow_id=workflow_id, trace_id=trace_id,
        audio_s3_uri=req.audio_s3_uri, persona=persona,
    )
    client = await _temporal_client()
    await client.start_workflow(
        "BrdFromAudioWorkflow", wi, id=workflow_id,
        task_queue=os.environ.get("TEMPORAL_TASK_QUEUE", "brd-from-audio-tq"),
    )
    return StartWorkflowResponse(
        workflow_id=workflow_id, workflow_template_id=wi.workflow_template_id,
        trace_id=trace_id, outcome=BRDState.PENDING.value,
    )


@router.get("/{workflow_id}")
async def get_workflow_status(workflow_id: str) -> dict:
    with get_db() as session:
        pack = session.query(EvidencePack).filter_by(workflow_id=workflow_id).one_or_none()
        if pack is None:
            raise HTTPException(status_code=404, detail="workflow not found")
        return {
            "workflow_id": pack.workflow_id,
            "workflow_template_id": pack.workflow_template_id,
            "outcome": pack.outcome,
            "rounds_executed": pack.rounds_executed,
            "max_rounds": pack.max_rounds,
            "created_at": pack.created_at.isoformat() if pack.created_at else None,
            "completed_at": pack.completed_at.isoformat() if pack.completed_at else None,
            "error": pack.error,
            "agent_outputs_summary": {role: len(payloads)
                                      for role, payloads in (pack.agent_outputs or {}).items()},
            "hitl_exchange_count": len(pack.hitl_exchanges or []),
        }


@router.get("/{workflow_id}/stream")
async def stream_workflow(workflow_id: str) -> StreamingResponse:
    return StreamingResponse(consume_workflow_events(workflow_id),
                              media_type="text/event-stream")


class ClarificationSignal(BaseModel):
    rationale: str
    responder_user_id: str | None = None


@router.post("/{workflow_id}/signal/clarification", status_code=status.HTTP_204_NO_CONTENT)
async def signal_clarification(workflow_id: str, payload: ClarificationSignal) -> None:
    client = await _temporal_client()
    handle = client.get_workflow_handle(workflow_id)
    await handle.signal("clarification",
                        {"decision": "answer", "rationale": payload.rationale,
                         "responder_user_id": payload.responder_user_id})


class ApprovalSignal(BaseModel):
    decision: str
    rationale: str | None = None


@router.post("/{workflow_id}/signal/approval", status_code=status.HTTP_204_NO_CONTENT)
async def signal_approval(workflow_id: str, payload: ApprovalSignal) -> None:
    if payload.decision not in ("approve", "revise", "reject"):
        raise HTTPException(400, "decision must be approve/revise/reject")
    client = await _temporal_client()
    handle = client.get_workflow_handle(workflow_id)
    await handle.signal("approval", {"decision": payload.decision,
                                      "rationale": payload.rationale or ""})
```

Register the router in `app/main.py`:

```python
from app.api.workflows_routes import router as workflows_router
app.include_router(workflows_router)
```

CHECKPOINT 6: `curl -s http://localhost:8000/openapi.json | jq '.paths | keys[] | select(startswith("/api/v1/workflows"))' | wc -l` returns `5`.

---

## STEP 7: End-to-end smoke run

```bash
cd "$PROJECT_ROOT/agentcore-demo-test1-backend"
docker compose up -d

# Verify workflow can start
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
RESPONSE=$(curl -sX POST http://localhost:8000/api/v1/workflows \
    -H "Content-Type: application/json" \
    -d "{\"audio_s3_uri\":\"s3://agentcore-demo-test1-audio-uploads-${ACCOUNT_ID}/seed/3sec.wav\",\"persona_id\":\"ba-sap-mm\"}")
WF=$(echo "$RESPONSE" | jq -r .workflow_id)
echo "$RESPONSE" | jq .

# Poll
for _ in {1..10}; do
    curl -s "http://localhost:8000/api/v1/workflows/$WF" | jq '.outcome,.rounds_executed'
    sleep 3
done

# Approve
curl -X POST "http://localhost:8000/api/v1/workflows/$WF/signal/approval" \
    -H "Content-Type: application/json" -d '{"decision":"approve"}'

curl -s "http://localhost:8000/api/v1/workflows/$WF" | jq '.outcome'
# Expect: "APPROVED"
```

CHECKPOINT 7: Final outcome is `APPROVED`; `agent_outputs` populated for all 3 agents; at least one `hitl_exchanges` entry.

---

## GATE PASS CHECKLIST — Phase 4 Complete

- [ ] **G4-1** `app/temporal/types.py` defines BRDState, AgentRole, StepType, WorkflowInput, AgentInvocation (canonical names per PCD D4).
- [ ] **G4-2** `app/temporal/workflow_streams.py` reports Pattern A or Pattern B at import.
- [ ] **G4-3** `app/temporal/activities.py` exports all 5 activity functions; `_validate_8block()` enforces identifier echo.
- [ ] **G4-4** `app/temporal/workflows.py` defines `BrdFromAudioWorkflow` using `AgentRole.TRANSCRIBER/DRAFTER/REVIEWER`.
- [ ] **G4-5** `app/temporal/worker_entrypoint.py` registers the workflow and all 5 activities on `brd-from-audio-tq`.
- [ ] **G4-6** `app/api/workflows_routes.py` mounts all 5 routes under `/api/v1/workflows` (D11).
- [ ] **G4-7** No new Alembic migration for `evidence_packs` (Phase 3 owns the table per D10).
- [ ] **G4-8** Smoke run reaches APPROVED outcome and persists agent outputs + HITL exchanges to RDS.
- [ ] **G4-9** No `temporalio` import in `agents/`: `! grep -rE "^(import|from) temporalio" agents/`.
- [ ] **G4-10** No stale strings: `! grep -rE "AgentRole.ANALYST|AgentRole.ARCHITECT|brd_app|brd_platform|REVISION_REQUESTED|MAX_ROUNDS\"" agentcore-demo-test1-backend/app/temporal/`.

---

## TESTING REQUIREMENTS — pytest

```
tests/
  temporal/
    conftest.py
    test_workflow_basic.py
    test_self_correction_cap.py
    test_hitl_clarification.py
    test_hitl_approval.py
    test_streams_pattern_a.py
    test_streams_pattern_b.py
    test_activities_validate.py
    test_activities_idempotency.py
```

Run: `uv run pytest tests/temporal/ -v`

---

*End of Prompt 04.*
