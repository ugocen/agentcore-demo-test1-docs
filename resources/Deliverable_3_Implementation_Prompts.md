# Implementation Prompts (V5 Compliant)

Four self-contained prompts for low-thinking AI coding models. Each prompt builds one layer of the AgentCore demo test 1 demo. No SDK — V5 Section 9.5: standard opentelemetry-sdk only.

**Architecture:** 2 Strands agents + 1 LangGraph agent. Temporal self-hosted. CopilotKit + AG-UI frontend. Mock auth. S3 zip deployment via `agentcore configure/deploy`.

**V5 Key Changes from Previous Versions:**
- No SDK — raw dict payload builder, OTel spans, ADOT Collector
- `BedrockAgentCoreApp` with `@app.entrypoint` (not custom FastAPI)
- S3 zip deployment (not ECR/container)
- V5 payload schema (Section 7) — `tier: "native"`, `step_type` enum, `custom_metadata`
- Endpoints: `/invoke` (V5), `/metadata`, `/capabilities`, `/metrics`, `/health`
- AgentCore maps `/invocations` → `/invoke`, `/ping` → auto
- A2A for agent↔agent (Temporal signals), AG-UI for frontend↔agent (CopilotKit)

---

## PROMPT 1: Backend Foundation (FastAPI + Temporal + OTel)

**Header:** This is Prompt 1 of 4 implementing an AgentCore demo test 1 demo per AI SDLC Guidelines V5. Upload `PROJECT_CONTEXT.md` as companion document.

**Tech References:** FastAPI, opentelemetry-api/sdk/distro, OTel FastAPI instrumentation, Temporal Python SDK, SQLAlchemy, Alembic, Pydantic Settings, structlog.

**Files to Create:**

1. `backend/app/main.py` — FastAPI entrypoint with OTel auto-instrumentation
2. `backend/app/auth/mock_user.py` — `MOCK_USER` constant (no Cognito)
3. `backend/app/otel/setup.py` — Tracer setup (standard opentelemetry-sdk)
4. `backend/app/storage/rds_models.py` — SQLAlchemy models for `agentcore_demo_test1`
5. `backend/app/storage/s3_claim_check.py` — Claim-Check handler (V5 Section 12)
6. `backend/app/config/settings.py` — Pydantic Settings
7. `backend/alembic/versions/001_initial.py` — Migration (3 tables)
8. `backend/Dockerfile` — Shared by fastapi and temporal-worker
9. `docker-compose.yml` — 5 services (temporal-server, temporal-ui, fastapi, temporal-worker, nextjs)
10. `backend/app/temporal/workflows.py` — Placeholder (Prompt 2 fills)
11. `backend/app/temporal/activities.py` — Placeholder
12. `backend/app/temporal/worker.py` — Placeholder
13. `backend/app/worker_entrypoint.py` — Placeholder

**OTel Setup:**
```python
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
FastAPIInstrumentor.instrument_app(app)
```

**Acceptance Criteria:**
- [ ] `docker compose up -d` starts 5 services
- [ ] `/health` returns 200
- [ ] OTel traces visible in CloudWatch/X-Ray
- [ ] Alembic migrations applied to `agentcore_demo_test1`
- [ ] No SDK references anywhere

---

## PROMPT 2: Temporal Workflow + Activities

**Header:** This is Prompt 2 of 4. Upload `PROJECT_CONTEXT.md`, `ARCHITECTURE_DATAFLOW_GUIDE.md`, and `Prompt_02_Agent_Telemetry.md`.

**Tech References:** Temporal Python SDK, Bedrock AgentCore Runtime (`invoke_agent_runtime`), OTel span events, V5 payload validation.

**Files to Create:**

1. `backend/app/temporal/workflows.py` — BRDWorkflow with HITL signal loop (max 5 rounds, 15 min timeout)
2. `backend/app/temporal/activities.py` — Activities that invoke AgentCore via boto3
3. `backend/app/temporal/worker.py` — Worker registration
4. `backend/app/temporal/workflow_streams.py` — Workflow Streams publisher
5. `backend/app/worker_entrypoint.py` — `if __name__ == "__main__"`
6. `backend/app/telemetry/evidence_pack.py` — Evidence Pack builder (ALCOA+, V5 Section 14.6)

**Activity Pattern:**
```python
@activity.defn
def transcribe_activity(audio_uri: str, workflow_id: str):
    client = boto3.client("bedrock-agentcore-runtime")
    result = client.invoke_agent_runtime(
        agentRuntimeArn=AGENT_1_ARN,
        qualifier="DEFAULT",
        payload={"messages": [{"role": "user", "content": audio_uri}]},
        sessionId=workflow_id,  # Session isolation
    )
    payload = result["payload"]
    # Validate 8 blocks
    assert "status" in payload and "risk" in payload
    return payload
```

**A2A Pattern:** Agent 1 completes → Temporal signal → Agent 2 starts (not direct agent↔agent HTTP).

**Acceptance Criteria:**
- [ ] Workflow starts, pauses on signal, resumes, completes
- [ ] Evidence Pack in RDS for ALL 4 outcomes (APPROVED, REJECTED, TERMINATED, FAILED)
- [ ] OTel spans for each workflow step visible in CloudWatch
- [ ] 8-block payload validated in every activity return

---

## PROMPT 3: Frontend (Next.js + CopilotKit + AG-UI)

**Header:** This is Prompt 3 of 4. Upload `PROJECT_CONTEXT.md` and `ARCHITECTURE_DATAFLOW_GUIDE.md`.

**Tech References:** Next.js 14 App Router, CopilotKit React SDK (>=1.50), AG-UI protocol, react-markdown.

**AG-UI vs A2A Clarification:**
- **AG-UI** = Frontend ↔ Agent (CopilotKit, SSE, chat streaming, HITL) — THIS PROMPT
- **A2A** = Agent ↔ Agent (Temporal signals) — NOT this prompt

**Files to Create:**

1. `frontend/app/page.tsx` — Landing page (4 metric tiles)
2. `frontend/app/workspace/[wfId]/page.tsx` — Workspace with CopilotKit
3. `frontend/lib/agentcore-agui-client.ts` — HttpAgent subclass for AgentCore
4. `frontend/lib/use-workflow-stream.ts` — Workflow Stream SSE hook (Temporal → FastAPI → Browser)
5. `frontend/components/TranscriptCard.tsx` — useFrontendTool
6. `frontend/components/DraftPreviewCard.tsx` — STATE_DELTA handling
7. `frontend/components/ClarificationQuestionCard.tsx` — renderAndWaitForResponse + dual-response
8. `frontend/components/ApprovalCard.tsx` — Final approval (AG-UI + Temporal Signal)
9. `frontend/Dockerfile`

**Acceptance Criteria:**
- [ ] All 8 CopilotKit features render
- [ ] AG-UI events flow AgentCore → Browser (Network tab SSE)
- [ ] Workflow Stream separate SSE from FastAPI
- [ ] HITL dual-response (AG-UI respond + Temporal Signal)
- [ ] Page refresh restores chat + canvas from RDS

---

## PROMPT 4: Agent Development + Deployment (2 Strands + 1 LangGraph)

**Header:** This is Prompt 4 of 4. Upload `PROJECT_CONTEXT.md` and `Deliverable_2_Reference_Strands_Agent_Code.md`.

**Tech References:** Strands SDK, LangGraph, BedrockAgentCoreApp (`bedrock-agentcore`), `agentcore configure/deploy`, V5 payload builder, OTel.

**Deployment Method (S3 Zip — NOT ECR/Container):**
```bash
# For each agent:
cd agents/agent_N_xxx
agentcore configure -e agent.py --protocol AGUI -r us-east-1
agentcore deploy  # Packages ZIP → S3 → CodeBuild → Runtime
```

**No Dockerfile. No ECR. No `docker build`. AgentCore handles everything.**

**3 Agents:**

**Agent 1 (Transcriber) — Strands:**
- Uses `BedrockAgentCoreApp` with `@app.entrypoint`
- Tool: `transcribe_audio_tool` (Amazon Transcribe, `IdentifyLanguage=True`)
- V5 payload with `custom_metadata: {audio_language_detected, audio_duration_seconds}`
- OTel spans via `setup_tracer("transcriber")`

**Agent 2 (Drafter) — Strands:**
- Uses `BedrockAgentCoreApp` with `@app.entrypoint`
- Tools: `request_clarification` (HITL), `generate_markdown_draft`
- V5 payload with `waiting_for_human_ms`, `step_type: "human_question"`
- `template_adherence_pct: 85.0` in Quality block

**Agent 3 (Reviewer) — LANGGRAPH:**
- Uses `StateGraph` (NOT Strands) with 4 nodes
- Uses `BedrockAgentCoreApp` with `@app.entrypoint`
- Nodes: `analyze_quality` → `scan_pii` → `check_policy` → `generate_report`
- V5 payload with `PolicyViolation` list, `defects_found_later`
- OTel span per LangGraph node

**Shared Files:**
- `agents/common/payload_builder.py` — V5 8-block raw dict builder
- `agents/common/otel_setup.py` — OTel tracer setup
- `agents/common/a2a_envelope.py` — Stub (A2A reserved for future)

**Acceptance Criteria:**
- [ ] All 3 agents deploy via `agentcore configure/deploy`
- [ ] All 3 respond to `invoke_agent_runtime`
- [ ] Agent 1 payload has `custom_metadata` with language + duration
- [ ] Agent 3 uses LangGraph (not Strands)
- [ ] All payloads have ALL 8 blocks (validated)
- [ ] OTel traces visible in CloudWatch for all 3 agents
- [ ] No SDK references anywhere
- [ ] `tier: "native"` in all payloads
