# PROJECT CONTEXT DOCUMENT (PCD)

## AgentCore Demo — Audio → BRD Automation (Reference Orchestration)

**Document ID:** DELIV-000-PCD
**Version:** 2.0
**Date:** 2026-05-15
**Status:** BASELINE — Single Source of Truth
**Classification:** Internal — Shared Baseline for All Coding Sessions
**Upstream Guideline:** `AI_SDLC_Agent_Development_Guidelines_v5.docx` (Sections 3.4 identifier hierarchy, §7 standard agent interface, §8 8-block payload, §9 telemetry, §12 Temporal, §16 HITL)

> **About this version.** v2.0 supersedes v1.0 (2025-01-20). The previous version pre-dated full alignment with the Guideline; v2.0 reconciles every contract surface (payload block names, identifier hierarchy, required endpoints, telemetry attribute prefix, claim-check threshold, self-correction cap, HITL step types, agent boundary vs Temporal) to the Guideline. Every other document in this repo (Prompts 00–07, Deliverables 1–9, side guides) is being rewritten to point at this PCD.

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Architecture Overview](#2-architecture-overview)
3. [Frozen Decisions (D1–D14)](#3-frozen-decisions-d1d14)
4. [Non-Negotiable Rules](#4-non-negotiable-rules)
5. [Repository Structure](#5-repository-structure)
6. [8-Block Response Payload](#6-8-block-response-payload)
7. [Identifier Hierarchy](#7-identifier-hierarchy)
8. [Required Agent Endpoints](#8-required-agent-endpoints)
9. [HITL Iteration Model](#9-hitl-iteration-model)
10. [Telemetry Model (OTel + ADOT + CloudWatch)](#10-telemetry-model)
11. [Workflow Streams Strategy (Pattern A + Fallback)](#11-workflow-streams-strategy)
12. [Demo Simplifications vs Production](#12-demo-simplifications-vs-production)
13. [Comment Density Policy](#13-comment-density-policy)
14. [Companion Documents](#14-companion-documents)
15. [Consolidated Tech Reference](#15-consolidated-tech-reference)
16. [Appendix A — Environment Variables](#appendix-a--environment-variables)
17. [Appendix B — Mock User Auth (Demo Only)](#appendix-b--mock-user-auth-demo-only)
18. [Appendix C — Error Code Dictionary](#appendix-c--error-code-dictionary)
19. [Appendix D — Status Code Dictionary](#appendix-d--status-code-dictionary)
20. [Appendix E — Quick Start](#appendix-e--quick-start)

---

## 1. Project Overview

This project delivers an **enterprise-grade reference demo** of the AI SDLC Orchestration Platform pattern. The reference orchestration is **`brd-from-audio-v1`** — a Business Requirements Document (BRD) generation workflow that takes an audio recording (microphone capture or file upload) and produces a versioned, approved BRD via three AI agents under Temporal orchestration.

**Why this demo exists.** The Guideline defines the full platform contract (8-block payload, identifier hierarchy, AgentCore deployment, ADOT telemetry, HITL step model, A2A via Temporal, Claim-Check pattern, Cedar policies, Evidence Pack). This demo proves every contract surface end-to-end in a single, runnable system at one EC2 instance + AgentCore runtime cost. Junior developers, Gemini-Flash-class LLMs, or any AI coding assistant can reproduce the system by reading this PCD and executing Prompts 00–07 in order.

**User-facing surface.** A single chat-driven web app at `http://<ec2-host>:3000`. The user:
1. Selects a mock persona (Business Analyst — SAP MM module by default).
2. Provides audio: either records via the in-app microphone button (browser `MediaRecorder` → webm/opus) or uploads a file (mp3/wav/m4a/webm, max 50 MB).
3. Watches a workflow stream as Agent 1 (Transcriber, Strands) → Agent 2 (Drafter, Strands) → Agent 3 (Reviewer, LangGraph 4-node) execute.
4. Answers any clarification questions emitted by the Drafter (HITL).
5. Reviews the BRD draft and the Reviewer's findings; approves, rejects, or requests revisions (HITL).
6. Downloads the final BRD (Markdown + PDF) and inspects the Evidence Pack at workflow end.
7. Visits `/metrics` for a live dashboard of project- and workflow-level metrics (cost, latency, savings, acceptance rate).

**Demo volume.** 15 runs/day × 3 days = 45 total runs. Audio length 2–3 minutes per file (~50 MB max).

---

## 2. Architecture Overview

### 2.1 Topology Diagram

```
+--------------------------------------------------------------------+
| AWS CLOUD (MANAGED SERVICES)                                       |
|                                                                    |
|  +-------------------------+  +--------------------------------+   |
|  | AgentCore Runtime       |  | Amazon Transcribe              |   |
|  | (AWS managed)           |  | (auto language detection)      |   |
|  | Firecracker MicroVMs    |  +--------------------------------+   |
|  | Per-session isolation   |  +--------------------------------+   |
|  | Persistent FS mount     |  | Amazon Bedrock                 |   |
|  | S3 ZIP deployment       |  | Claude 3.5 Sonnet v2           |   |
|  +-------------------------+  +--------------------------------+   |
|                                                                    |
|  +-------------------------+  +--------------------------------+   |
|  | S3 buckets (4)          |  | CloudWatch Logs + Metrics      |   |
|  | audio-uploads           |  | DemoSDLC/Agent namespace       |   |
|  | artifacts               |  | + X-Ray traces                 |   |
|  | claimcheck              |  +--------------------------------+   |
|  | code (agent ZIPs)       |                                       |
|  +-------------------------+                                       |
+--------------------------------------------------------------------+
                              ^
                              | IMDSv2 + boto3 (no static keys)
                              | OTLP gRPC :4317
                              v
+--------------------------------------------------------------------+
| EC2 t3.large — Docker Compose host (developer-managed)             |
| 6 containers:                                                      |
|   1. temporal-server   :7233 (gRPC)                                |
|   2. temporal-ui       :8081                                       |
|   3. temporal-worker   (no public port)                            |
|   4. fastapi           :8000 (REST + SSE)                          |
|   5. nextjs-frontend   :3000                                       |
|   6. adot-collector    :4317 (OTLP gRPC ingress)                   |
+--------------------------------------------------------------------+
                              |
                +-------------+-------------+
                v                           v
        +------------------+    +---------------------------+
        | RDS PostgreSQL   |    | S3 (artifacts/claimcheck) |
        | db.t4g.micro     |    | KMS-encrypted             |
        | 2 AZs (subnet    |    +---------------------------+
        | group requires)  |
        | DBs:             |
        |  - temporal      |
        |  - temporal_     |
        |    visibility    |
        |  - agentcore_    |
        |    demo_test1    |
        +------------------+
```

**Critical Clarification (do not confuse).** AgentCore Runtime is an **AWS-managed service**, NOT a container running on EC2. The developer experience is "submit Python source, get a running agent": you run `agentcore configure -e main.py` + `agentcore deploy` from the starter toolkit and the toolkit handles everything between source code and a running Firecracker MicroVM. When a Temporal activity calls `boto3.invoke_agent_runtime()`, AWS provisions a MicroVM for that session, runs the agent, and terminates the MicroVM.

> **Deployment mode reality.** Under the hood, the toolkit's default `--deployment-type direct_code_deploy` mode (a) zips the directory, (b) uploads the ZIP to S3 (`bedrock-agentcore-codebuild-sources-...`), (c) runs **AWS CodeBuild** to build a container image, and (d) registers the runtime — so the AgentCore Runtime IS container-backed, even though you never write a Dockerfile or push to ECR yourself. The alternative `--deployment-type container` mode is an explicit container path: it auto-creates an ECR repository and pushes the image there (visible via `aws ecr describe-repositories`). The demo uses `direct_code_deploy` so developers don't touch Docker, but the toolkit's `destroy --delete-ecr-repo` flag exists because ECR resources DO get created in container mode. Earlier drafts of this doc claimed "no container, no ECR" — that is the developer's surface, not the implementation.

### 2.2 Component List (6 containers + AWS managed)

| Component | Technology | Port | Role |
|---|---|---|---|
| temporal-server | `temporalio/auto-setup:1.22` + PostgreSQL persistence | 7233 (gRPC) | Workflow orchestration engine |
| temporal-ui | Temporal Web UI | 8081 | Workflow inspection and debugging |
| temporal-worker | Temporal Python SDK 1.6+ worker | none | Executes workflow and activity code |
| fastapi | FastAPI 0.110+ (Python 3.11+) | 8000 | REST API + SSE bridge + workflow control |
| nextjs-frontend | Next.js 14+ + CopilotKit 1.50+ | 3000 | React frontend; chat + canvas + metrics dashboard |
| adot-collector | AWS Distro for OpenTelemetry | 4317 (OTLP gRPC) | OTel span ingress, EMF generation, governance gate, push to CloudWatch |
| AgentCore Runtime | AWS Bedrock AgentCore (managed) | 443 (HTTPS) | Hosts 3 agents; Firecracker MicroVM per session; Persistent FS mount |
| RDS PostgreSQL | db.t4g.micro, **2-AZ subnet group** (required by AWS) | 5432 | 3 databases: `temporal`, `temporal_visibility`, `agentcore_demo_test1` |
| S3 | 4 Standard buckets | 443 | audio-uploads, artifacts, claimcheck, code |
| Amazon Transcribe | Batch async | n/a | Audio → transcript JSON with auto language detection |
| Amazon Bedrock | Claude 3.5 Sonnet v2 | n/a | LLM for translation, BRD drafting, review |
| CloudWatch | Logs + Metrics (EMF via ADOT) + X-Ray | n/a | Long-term telemetry; `DemoSDLC/Agent` metric namespace |

### 2.3 Dual-Stream Hybrid Model

Two independent streams reach the browser:

- **Stream A — AG-UI (frontend → AgentCore SSE).** Real-time agent thinking, tool visualization, partial output. Carries ephemeral events: `agent_start`, `step_start`, `tool_call`, `step_end`, `completion`. AgentCore microVM session_id = Temporal workflow_id.
- **Stream B — Workflow Stream (FastAPI SSE bridge → browser).** Orchestration state: HITL questions, status transitions, evidence pack completion. Two-pattern strategy (see §11):
  - **Pattern A (preferred):** `temporalio.contrib.workflow_streams` if the installed Python SDK exposes it.
  - **Pattern B (fallback):** Temporal Workflow Update API + side-channel pub/sub (Redis or in-process broker) → FastAPI SSE.

These streams are **independent**. AG-UI gives "what is the agent doing right now"; Workflow Stream gives "where is the orchestration overall". Mixing them causes coupling and head-of-line blocking.

**HITL coordination.** When the Drafter needs a clarification, it returns Status code `REQUIRE_HUMAN_REVIEW` in its 8-block payload (Guideline §12.9.5). The Temporal workflow:
1. Catches `REQUIRE_HUMAN_REVIEW` from the activity result.
2. Opens a `hitl_question` step (own `step_id`) and emits a Workflow Stream event (Stream B).
3. Durably waits on a Signal handler with timeout (15 minutes by default).
4. When the user replies via `POST /api/v1/workflows/{id}/signal/clarification`, the workflow re-invokes the same agent with `human_input` populated. Conversation state for chat lives in AgentCore Memory (actor_id + session_id scoped), NOT in Temporal payloads.

### 2.4 End-to-End Data Flow

```
1. User loads http://<ec2-host>:3000 → frontend (Next.js)
2. User selects mock persona, records or uploads audio
3. Frontend POST /api/v1/uploads/audio → FastAPI streams to S3 (audio-uploads bucket)
   Returns: s3://...mp3 (or .webm)
4. Frontend POST /api/v1/workflows {audio_s3_uri, persona_id}
   FastAPI: generates workflow_template_id="brd-from-audio-v1",
   workflow_id="wf-<uuid>", trace_id="<W3C>", starts Temporal workflow
5. Temporal worker runs BrdFromAudioWorkflow:
   - Activity 1: invoke_agent_runtime(agent_1_transcriber, payload)
     → AWS provisions MicroVM → Strands agent calls Transcribe →
     returns 8-block payload (Status, Resources, Timing, Financial,
     Artifacts[transcript.json], Quality, Tool Calls[Transcribe], Risk)
   - Activity 2: persist_evidence_input (DB row for step)
   - Activity 3: invoke_agent_runtime(agent_2_drafter, payload + transcript ref)
     → MicroVM → Strands agent reads transcript via POSIX mount if >1 MB →
     calls Bedrock Claude → returns 8-block payload with BRD draft artifact
     OR Status=REQUIRE_HUMAN_REVIEW + hitl_question
   - If REQUIRE_HUMAN_REVIEW:
       Workflow opens hitl_question step, waits for signal
       User responds → workflow re-invokes Drafter with human_input
       Self-correction cap: 2 attempts after initial (3 total) → mandatory escalation
   - Activity 4: invoke_agent_runtime(agent_3_reviewer, payload + draft ref)
     → MicroVM → LangGraph 4-node graph (compliance, quality, PII, compile) →
     returns 8-block payload with review report artifact
   - hitl_approval step: user approves/rejects/revises
     - Approved: orchestrator assembles Evidence Pack, publishes final artifact
     - Revise: workflow loops to Drafter with feedback (new agent_run_id, step_type=revise)
     - Rejected: workflow terminates with status=REJECTED
6. Every agent invocation emits OTel spans with demo.* attributes →
   ADOT Collector (in EC2 docker-compose) → CloudWatch Logs + Metrics
7. Frontend receives Workflow Stream events throughout (HITL prompts, status transitions)
   and AG-UI events from each agent invocation (live agent thinking)
8. /metrics page reads from FastAPI:
   - GET /api/v1/metrics/overview (aggregated from RDS evidence_packs)
   - GET /api/v1/metrics/cloudwatch (live CloudWatch get_metric_data)
   - GET /api/v1/metrics/computed (Guideline §9.6 formulas)
```

---

## 3. Frozen Decisions (D1–D14)

These decisions are locked. Changing one requires re-validating every downstream document. They were finalized 2026-05-15 with the user.

| # | Decision | Source |
|---|---|---|
| **D1** | **Span attribute prefix = `demo.*`** (not `jnj.*`). Demo intentionally uses a generic, non-enterprise prefix. ADOT Collector configuration filters and promotes `demo.*` attributes. In a production JnJ deployment the prefix would be `jnj.*` per Guideline §9.5. | User 2026-05-15 |
| **D2** | **Reference orchestration = `audio → BRD`**, template ID `brd-from-audio-v1`. | User 2026-05-15 |
| **D3** | **Cedar policies = documentation-only**. PCD and the AWS guide describe the GitOps pattern and include one illustrative `.cedar` file; runtime enforcement (AgentCore Gateway) is not deployed in the demo. Marked "Production-only". | User 2026-05-15 |
| **D4** | **Agent naming.** Three agents: transcriber, drafter, reviewer. Directories `agents/agent_1_transcriber/`, `agents/agent_2_drafter/`, `agents/agent_3_reviewer/`. Entry file `main.py`. **`agent_id` strings (in payloads, runtimes, ARNs) use UNDERSCORES** matching the directory name verbatim: `agent_1_transcriber`, `agent_2_drafter`, `agent_3_reviewer`. The starter toolkit derives `agent_id` from the directory name without transformation. The **runtime ARN appends a toolkit-generated random suffix**: `arn:aws:bedrock-agentcore:us-east-1:<acct>:runtime/agent_1_transcriber-<10char>` (e.g. `agent_1_transcriber-ZgDOxI4k9f`). Capture each runtime's actual ARN at deploy time and store in `.vpc_env`. | User 2026-05-15 + verified against live deployment |
| **D5** | **Local dev path = `/Users/ugurgocen/projects/agentcore-demo-test1/`** (Mac, primary developer environment). Prompt_06 (E2E) additionally calls out the EC2 path (`/home/ec2-user/agentcore-demo-test1/`) when running inside the instance. | User 2026-05-15 |
| **D6** | **Temporal Workflow Streams retained, with fallback.** Pattern A: `temporalio.contrib.workflow_streams` (if SDK supports). Pattern B: Workflow Update API + side-channel pub/sub. Feature flag chooses at runtime. See §11. | User 2026-05-15 + technical risk mitigation |
| **D7** | **8-block payload names = Status / Resources / Timing / Financial / Artifacts / Quality / Tool Calls / Risk** (Guideline §8 verbatim). Any alternate set (e.g. `context_block`, `input_block`, `signature_block`) is wrong and removed everywhere. | Guideline §8 |
| **D8** | **Self-correction cap.** 2 attempts after the initial generation (3 total) before mandatory HITL escalation. Tracked in Block 6 (`total_attempts`). Each attempt is a new `step_id` with `step_type=revise`. `agent_run_id` stays the same only when the agent invocation is logically continuous. | Guideline §16.5 |
| **D9** | **Mock auth = demo-only**. Every doc that touches auth explicitly marks "DEMO ONLY; production uses JnJ Entra ID + PingID per Guideline §15.1.1". The mock persona is a JSON object passed in the inbound request `requested_by` block; there is no real OAuth, no Cognito, no JWT validation. | Guideline §15.1 + simplification |
| **D10** | **`evidence_packs` canonical schema** (UUID + JSONB shape; rounds_executed, agent_outputs, hitl_exchanges, final_brd_content, error, metadata). Single Alembic migration owns table creation (Phase 3); Phase 4 only inserts/updates rows. | Conflict reconciliation |
| **D11** | **API path canonical = `/api/v1/...`** with subroutes: `POST /api/v1/uploads/audio`, `POST /api/v1/workflows`, `GET /api/v1/workflows/{id}`, `GET /api/v1/workflows/{id}/stream` (SSE), `POST /api/v1/workflows/{id}/signal/clarification`, `POST /api/v1/workflows/{id}/signal/approval`, plus metrics: `GET /api/v1/metrics/overview`, `/api/v1/metrics/by-template`, `/api/v1/metrics/by-project`, `/api/v1/metrics/cloudwatch`, `/api/v1/metrics/computed`. | Conflict reconciliation |
| **D12** | **DB credentials canonical.** Database `agentcore_demo_test1`, app user `app_user`. Also: `temporal` and `temporal_visibility` databases, owned by `temporal_user`. Passwords generated in Prompt_00, exported to `.vpc_env`, consumed by `.env` in Prompt_03. | Conflict reconciliation |
| **D13** | **Audio input = browser MediaRecorder + file upload.** Both paths supported in the chat input. Recording produces `audio/webm;codecs=opus` (Transcribe-compatible). Upload accepts mp3/wav/m4a/webm. No extra libraries (no lamejs). Max 50 MB. | User 2026-05-15 |
| **D14** | **Metric Dashboard page at `/metrics`**, separate route from `/workspace`. React + `recharts`. Backed by 5 FastAPI endpoints (D11). Computed metrics per Guideline §9.6 formulas: time_saved_hours, net_savings_usd, roi_pct, manual_equivalent_fte, speed_multiplier, acceptance_rate. | User 2026-05-15 |

---

## 4. Non-Negotiable Rules

1. **8-block payload, every invocation, every agent.** Status, Resources, Timing, Financial, Artifacts, Quality, Tool Calls, Risk (D7). No partial payloads. `"Unavailable"` is the marker for fields the agent genuinely cannot produce. No alternate block names.

2. **Identifier hierarchy, no inversions.** Platform creates `trace_id`, `workflow_template_id`, `workflow_id`. Orchestrator creates `agent_run_id`, `step_id`, `parent_run_id`, `task_id`, `step_sequence`. Agents echo all identifiers in Block 1 (Status) and propagate them on every outbound call. Agents NEVER invent identifiers a higher layer should have provided.

3. **W3C `traceparent` propagation, every layer.** Every HTTP request, Temporal activity, boto3 call carries `traceparent`. OpenTelemetry spans wrap every layer boundary.

4. **OpenTelemetry + ADOT, no manual EMF.** Agents emit OTel spans with `demo.*` attributes (D1). The platform-managed ADOT Collector transforms spans to EMF and pushes to CloudWatch. Agents do NOT construct EMF JSON. Agents do NOT call CloudWatch APIs directly.

5. **A2A via Temporal signals, never direct agent-to-agent HTTP.** Agent-to-agent messaging is backend-only and durable. Agents are deliberately unaware of each other; the workflow chains them. AG-UI handles user-facing streaming, A2A handles agent coordination.

6. **Claim-Check pattern for payloads >1 MB.** Trigger threshold configurable via `CLAIM_CHECK_THRESHOLD_BYTES` (default 1,048,576). First-party (AgentCore-hosted) agents read/write claim-check payloads via POSIX mount `/mnt/data/sdlc-payloads-claimcheck/`. Outside AgentCore, boto3 fallback is acceptable.

7. **Conversational state in AgentCore Memory, not Temporal payloads.** Chat history and semantic context belong to AgentCore Memory keyed by `actor_id` + `session_id`. Temporal payloads carry business inputs and identifiers only.

8. **Idempotency.** Same `(workflow_id, step_id, request_hash)` produces the same observable outcome. Activity retry is at-least-once by default; agents must tolerate re-invocation.

9. **Evidence Pack at every workflow termination** (APPROVED, REJECTED, TERMINATED, FAILED, MAX_ITERATIONS) per Guideline §15.6. Orchestrator assembles it; agents contribute hashes and references in Block 5.

10. **AgentCore boundary.** Agents do NOT import `temporalio`. Agents do NOT speak Temporal. Activities wrap agents; activities live in the platform's workflow repository. In this demo, "platform" = the backend repo.

11. **Self-correction cap = 3 total attempts** (D8). Beyond that, mandatory `hitl_review` escalation.

12. **No secrets in code/env/payload.** EC2 → AWS via IMDSv2 instance profile only. Tokens for external systems (not in scope for this demo) would use AgentCore Token Vault in production.

13. **Comments explain WHY, not WHAT.** Code comments are sparse but precise. Documentation lives in Markdown.

14. **All comments and identifiers in English.** No Turkish, no mixed languages inside `.py`, `.ts`, `.yaml` files. (User-facing UI may be translated separately if requested.)

---

## 5. Repository Structure

Polyrepo strategy: 4 independent Git repos under `/Users/ugurgocen/projects/agentcore-demo-test1/`.

### Repo 1: `agentcore-demo-test1-backend`

```
agentcore-demo-test1-backend/
├── .gitignore                                # excludes .venv, __pycache__, .env
├── .python-version                            # "3.11"
├── pyproject.toml                             # uv-managed; fastapi, temporalio, boto3, opentelemetry-*
├── uv.lock                                    # committed
├── Dockerfile                                 # FastAPI + Worker only (NOT agents)
├── docker-compose.yml                         # 6-service local stack
├── alembic.ini
├── alembic/                                   # DB migrations
│   └── versions/
│       └── 0001_initial.py                    # evidence_packs (D10)
├── app/
│   ├── __init__.py
│   ├── main.py                                # FastAPI app, lifespan, routers
│   ├── otel_setup.py                          # OTLP exporter → ADOT :4317
│   ├── config/
│   │   └── settings.py                        # Pydantic Settings (env vars)
│   ├── api/
│   │   ├── __init__.py
│   │   ├── workflows_routes.py                # /api/v1/workflows/...
│   │   ├── uploads_routes.py                  # /api/v1/uploads/audio
│   │   ├── evidence_routes.py                 # /api/v1/evidence/{id}
│   │   ├── metrics_routes.py                  # /api/v1/metrics/... (P12)
│   │   ├── dependencies.py                    # Mock auth, DB session, OTel tracer
│   │   └── schemas/
│   │       ├── workflows.py
│   │       ├── payload.py                     # 8-block Pydantic mirrors (D7)
│   │       └── metrics.py
│   ├── temporal/
│   │   ├── __init__.py
│   │   ├── client.py                          # Temporal client singleton
│   │   ├── worker_entrypoint.py
│   │   ├── workflows/
│   │   │   └── brd_from_audio_workflow.py     # BrdFromAudioWorkflow
│   │   └── activities/
│   │       ├── invoke_agent_runtime.py        # Single generic wrapper (D4)
│   │       ├── persist_evidence_input.py
│   │       └── publish_stream_event.py        # Pattern A or B (D6)
│   ├── services/
│   │   ├── s3_handler.py                      # Claim-check (>1 MB)
│   │   ├── evidence_pack.py                   # Assembly + PDF render
│   │   ├── sse_bridge.py                      # Workflow Stream → SSE
│   │   ├── metrics_service.py                 # Aggregation logic (P12)
│   │   ├── cloudwatch_client.py               # get_metric_data wrapper (P12)
│   │   └── db.py
│   └── models/
│       ├── evidence_pack.py                   # SQLAlchemy 2.0+ (D10)
│       └── enums.py                           # BRDState, StepType
├── agents/                                    # `direct_code_deploy`: dev submits source, toolkit handles container build via CodeBuild
│   ├── common/
│   │   ├── __init__.py
│   │   ├── payload_builder.py                 # 8-block builder (D7)
│   │   ├── otel_setup.py                      # demo.* attribute helpers (D1)
│   │   └── identifiers.py                     # Identifier echo helpers
│   ├── agent_1_transcriber/
│   │   ├── main.py                            # 5 endpoints; Strands
│   │   └── requirements.txt
│   ├── agent_2_drafter/
│   │   ├── main.py                            # 5 endpoints; Strands; HITL
│   │   └── requirements.txt
│   └── agent_3_reviewer/
│       ├── main.py                            # 5 endpoints; LangGraph
│       ├── nodes.py                           # 4 nodes (D4)
│       └── requirements.txt
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/
└── scripts/
    ├── dev_server.sh
    ├── worker.sh
    ├── agent_deploy.sh
    └── db_migrate.sh
```

### Repo 2: `agentcore-demo-test1-frontend`

```
agentcore-demo-test1-frontend/
├── .gitignore
├── package.json                                # pnpm; next 14+, @copilotkit/* 1.50+, recharts
├── pnpm-lock.yaml
├── Dockerfile
├── docker-compose.yml
├── next.config.js
├── tailwind.config.ts
├── tsconfig.json
├── app/                                        # Next.js App Router
│   ├── layout.tsx                              # Root layout (no CopilotKit here)
│   ├── page.tsx                                # Landing → persona select
│   ├── workspace/
│   │   └── [wfId]/
│   │       ├── layout.tsx                      # CopilotKit provider scoped here (D11)
│   │       └── page.tsx                        # Chat + canvas
│   ├── metrics/                                # P12 dashboard
│   │   ├── page.tsx
│   │   └── MetricsClient.tsx
│   └── api/
│       └── copilotkit/route.ts                 # CopilotKit edge route
├── components/
│   ├── ui/                                     # shadcn/ui
│   ├── canvas/
│   │   ├── BrdPreview.tsx
│   │   ├── ReviewPanel.tsx
│   │   └── EvidencePack.tsx
│   ├── chat/
│   │   ├── ChatPanel.tsx
│   │   ├── MessageBubble.tsx
│   │   ├── ToolCallCard.tsx
│   │   └── AudioInput/                         # D13
│   │       ├── AudioInput.tsx                  # Record + upload UI
│   │       └── useMicRecorder.ts               # MediaRecorder hook
│   ├── hitl/
│   │   └── ClarificationCard.tsx
│   └── metrics/                                # P12
│       ├── SummaryCards.tsx
│       ├── CostTimeSeries.tsx
│       ├── LatencyChart.tsx
│       └── TemplateBreakdown.tsx
├── hooks/
│   ├── useAgent.ts                             # CopilotKit useAgent
│   ├── useWorkflowStream.ts                    # SSE → Workflow Stream
│   └── useMetrics.ts                           # React Query for /metrics (P12)
├── lib/
│   ├── api/
│   │   ├── workflows.ts
│   │   ├── uploads.ts
│   │   └── metrics.ts
│   └── auth/mockAuth.ts                        # D9
└── tests/
    ├── unit/
    └── e2e/
```

### Repo 3: `agentcore-demo-test1-infra`

```
agentcore-demo-test1-infra/
├── README.md
├── scripts/                                   # Per Prompt_00, shell-script based IaC
│   ├── 00_bootstrap_admin.sh
│   ├── 01_vpc_subnets_sg.sh
│   ├── 02_s3_buckets.sh
│   ├── 03_rds_postgres.sh
│   ├── 04_iam_roles.sh
│   ├── 05_ec2_instance.sh
│   ├── 06_cloudwatch_log_groups.sh
│   ├── 07_agentcore_persistent_fs.sh
│   └── 99_teardown.sh
├── iam-policies/
│   ├── ec2-instance-role.json                 # Custom least-privilege (D11)
│   ├── transcribe-data-access.json
│   └── agentcore-runtime.json
└── adot/
    └── otel-collector-config.yaml             # demo.* governance gate (D1)
```

### Repo 4: `agentcore-demo-test1-docs`

```
agentcore-demo-test1-docs/
└── resources/
    ├── Deliverable_0_PROJECT_CONTEXT.md       # THIS FILE
    ├── Deliverable_1_..._9_*.md
    ├── Prompt_00_..._07_*.md                  # 8 prompts (Prompt_07 = metrics, P12)
    ├── PAYLOAD_SCHEMA.md                      # P3 canonical reference
    ├── APPLY_PLAN.md                          # P1–P13 execution plan
    ├── ARCHITECTURE_DATAFLOW_GUIDE.md
    ├── REPO_STRATEGY.md
    ├── GIT_WORKFLOW.md
    ├── DEMO_GELISTIRME_KILAVUZU.md            # Turkish dev guide
    ├── DEVELOPER_ONBOARDING.md
    ├── PERSISTENT_FILESYSTEM_GUIDE.md
    ├── PHASE_PLAN.md
    ├── V5_IMPLEMENTATION_PLAN.md
    ├── V5_TELEMETRY_INTEGRATION_ANALYSIS.md
    ├── DELIVERABLE_MAPPING.md
    ├── DOCUMENT_AUDIT_REPORT.md
    ├── CONSISTENCY_AUDIT_REPORT_ROUND2.md
    ├── CONSISTENCY_AUDIT_REPORT_ROUND3.md     # P13
    ├── FILE_STATUS_REPORT.md
    ├── PROJECT_STARTER.md
    ├── PROMPT_TESTING_SECTIONS.md
    └── WORKSPACE_SETUP_GUIDE.md
```

---

## 6. 8-Block Response Payload

Every agent invocation returns ALL 8 blocks. Names per Guideline §8 (D7). For full schema with field types and worked examples per agent, see `PAYLOAD_SCHEMA.md`.

| # | Block | Purpose | Required (Native) | Critical Fields |
|---|---|---|---|---|
| 1 | **Status** | Outcome + identifier echo | MUST | `code` (success/partial_success/partial_failure/failed/requires_human_review/blocked/cancelled/timeout), `http_status`, `error_code`, `error_category`, `retryable`, `trace_id`, `workflow_template_id`, `workflow_id`, `agent_run_id`, `step_id`, optional `custom_metadata` |
| 2 | **Resources** | Cost reporting model | MUST | `cost_reporting_model` (token_based/vendor_cost/fixed_subscription/both); sub-objects per model. For LLM-using agents: `model_used`, `tokens.input/output/total_outer`, `internal_llm_calls`, `total_tokens_all_calls` |
| 3 | **Timing** | Wall-clock breakdown | MUST | `total_elapsed_ms`, `queue_duration_ms`, `agent_active_ms`, `waiting_for_tools_ms`, `waiting_for_human_ms`, `timing_breakdown` (per-phase), `critical_path` |
| 4 | **Financial** | Value-realization inputs | MUST | `task_type`, `complexity_tier`, `manual_baseline_hours`, `agent_active_hours`, `human_review_hours_estimated` (null until review closes), `agent_cost_usd`, `retry_cost_usd`, `currency: "usd"` |
| 5 | **Artifacts** | Output metadata + versioning | MUST if produced | `artifact_id`, `artifact_type` (controlled enum), `artifact_format`, `content_ref` (s3://), `content_hash` (sha256), `versioning` (version, version_sequence, is_new_artifact, previous_version_ref, change_summary), `timestamps`, `template_used`, `template_adherence_pct`, `sections_completed/total`, `information_classification`, `retention_policy_ref` |
| 6 | **Quality** | Self-reported quality | MUST | `confidence` (0.0–1.0), `initial_attempt_success`, `total_attempts` (D8), `self_corrections`, `quality_concerns[]`, `unsupported_claims[]`, `completeness_assessment` |
| 7 | **Tool Calls** | Every MCP/API call | MUST | array of `{sequence, mcp_server, tool, duration_ms, downstream_system_time_ms, mcp_overhead_ms, status, data_classification, result_summary}` + `tool_summary` aggregates |
| 8 | **Risk** | PII / policy / compliance | MUST | `pii_detected`, `pii_filtered_count`, `secrets_detected`, `policy_violations`, `sensitivity_compliant`, `missing_inputs[]`, `unsupported_claims[]`, `compliance_checks[]` |

**Block 5 `artifact_type` controlled enum:** `user_story`, `design_document`, `code`, `migration_code`, `test_script`, `quality_document`, `break_fix`, `other`. Demo's BRD draft uses `design_document` with `artifact_subtype="functional_spec"`.

**Block 6 self-correction tracking (D8):** Each `revise` step has its own `step_id`. `total_attempts` counts initial + revisions, capped at 3 before mandatory HITL escalation.

---

## 7. Identifier Hierarchy

Per Guideline §3.4. The inbound `/invoke` request schema is:

```json
{
  "trace_id": "TRC-<W3C-trace-hex>",
  "workflow_template_id": "brd-from-audio-v1",
  "workflow_id": "wf-<uuid>",
  "agent_run_id": "run-<uuid>",
  "step_id": "step-<seq>",
  "step_sequence": 1,
  "parent_run_id": "wf-<uuid>",
  "task_id": "TASK-BRD-<id>",
  "agent_id": "agent_1_transcriber",
  "agent_version": "1.0.0",
  "requested_by": {
    "user_id": "demo-user-001",
    "email": "demo@local",
    "roles": ["business_analyst"],
    "group": {"group_id": "sap-mm", "group_type": "module", "group_name": "SAP MM"},
    "project": {"project_id": "proj-demo", "project_name": "Demo BRD", "system": "DEMO", "sector": "MT", "vertical": "ERP"}
  },
  "execution_context": {"environment": "dev", "data_classification": "INTERNAL"},
  "task": {
    "task_type": "audio_transcription | brd_generation | brd_review",
    "complexity_tier": "simple | medium | complex | very_complex",
    "instruction": "<free text>",
    "output_format": "json | markdown | docx",
    "template_id": "brd-template-v1"
  },
  "inputs": {"text": null, "artifact_refs": ["s3://..."], "metadata": {}},
  "execution_options": {"dry_run": false, "require_human_approval": true, "max_attempts": 3, "timeout_seconds": 900, "quality_threshold": 0.85},
  "actor_id": "demo-user-001",
  "session_id": "sess-<uuid>",
  "human_input": null,
  "human_approved": null
}
```

**Propagation rules:**
- `trace_id` is set at FastAPI workflow-start. W3C `traceparent` carries it through every HTTP, gRPC, and boto3 call.
- `workflow_id` is the Temporal WorkflowId (1:1 mapping). Used as AgentCore microVM `session_id`.
- `agent_run_id` allocated per agent invocation (per activity).
- `step_id` allocated per logical step inside the workflow. HITL turns are their own steps.
- Agents NEVER invent any of these. They echo all of them in Block 1 (Status) and propagate them.

**Step type enum (Guideline §16):** `agent_action`, `hitl_question`, `hitl_review`, `hitl_approval`, `hitl_branch_decision`, `tool_call`, `claim_check_io`, `revise`.

---

## 8. Required Agent Endpoints

Per Guideline §7.1, every agent exposes:

| Endpoint | Method | Purpose |
|---|---|---|
| `/invoke` | POST | Execute the task; takes the request schema above; returns 8-block payload |
| `/health` | GET | Liveness + dependency status + version |
| `/metadata` | GET | Agent identity (id, owner, framework, version, tier) and supported task types |
| `/capabilities` | GET | Supported inputs, outputs, tools, execution modes |
| `/metrics` | GET | Runtime metrics (request count, error count, p50/p95 latency) — light, in-process |

Optional (not required for demo): `/validate`, `/dry-run`, `/cancel`, `/resume`, `/feedback`.

The `BedrockAgentCoreApp` class auto-creates `POST /invocations` (which maps to `/invoke`) and `GET /ping` (health). The other four endpoints (`/health`, `/metadata`, `/capabilities`, `/metrics`) are registered manually on the same FastAPI/Starlette app instance inside the agent's `main.py`.

---

## 9. HITL Iteration Model

Per Guideline §16. HITL is a normal operating mode, not an exception.

**Each HITL turn is its own step.** When the Drafter needs clarification:
1. Agent returns Status `code=requires_human_review` with `step_type=hitl_question` + a `review_summary` (reason, agent_confidence, draft_decision if any, supporting_artifacts).
2. Temporal workflow opens a new step with a fresh `step_id`, emits a Workflow Stream event, durably waits on Signal `clarification` with a 15-minute timeout.
3. Frontend surfaces the question in chat (CopilotKit), captures the user's response, POSTs to `/api/v1/workflows/{id}/signal/clarification`.
4. Signal handler resumes the workflow, re-invokes the Drafter with `human_input` populated (carries reviewer rationale and decision).
5. Agent reads `human_input`, continues; or, if self-correction cap reached (D8), escalates further.

**Self-correction cap (D8):** Initial + 2 revisions = 3 total attempts. Block 6 `total_attempts` reflects the count. Beyond cap → escalate to `hitl_review` step regardless of confidence.

**Review approval pattern:** After Reviewer produces a report, workflow emits `hitl_approval` step. Frontend shows the review with three actions: **Approve** / **Revise** / **Reject**. Approve → publish artifact + assemble Evidence Pack. Revise → loop back to Drafter with feedback (new agent_run_id, step_type=revise). Reject → workflow terminates with status=REJECTED, Evidence Pack still assembled.

**Memory:** Conversational state across HITL turns lives in **AgentCore Memory** (`actor_id` + `session_id` scoped), NOT in Temporal payloads. Agents read prior turn notes from Memory; the platform passes `actor_id` and `session_id` on every invocation. (In the demo, AgentCore Memory may be replaced with an in-process dictionary or a Redis key for simplicity — documented as a demo simplification in §12.)

---

## 10. Telemetry Model

Per Guideline §9.5 — three-tier observability:

| Tier | What it carries | Who owns it |
|---|---|---|
| **Infrastructure telemetry** | Foundational events emitted natively by AgentCore: `InvokeAgentRuntime`, `ExecuteTool`, `CreateEvent` (Memory); vended logs + X-Ray traces | AgentCore (managed). No agent code. |
| **Business metadata** | `demo.*` attributes on OTel spans (D1): `demo.workflow_template_id`, `demo.workflow_id`, `demo.agent_run_id`, `demo.step_id`, `demo.actor_id`, `demo.cost_center`, `demo.data_classification`, `demo.error_category` | Agent developer, using standard `opentelemetry-sdk` |
| **Enforcement + formatting** | Span → EMF transformation, dimension assembly, cardinality enforcement, governance gate (drop spans missing mandatory attributes) | ADOT Collector (in EC2 docker-compose). No agent code. |

**Cardinality discipline (Guideline §9.4):**
- HIGH-cardinality (`demo.trace_id`, `demo.workflow_id`, `demo.agent_run_id`, `demo.step_id`, `demo.user_id`) → EMF JSON root only, NEVER in `CloudWatchMetrics.Dimensions`.
- LOW-cardinality (`demo.environment`, `demo.agent_id`, `demo.agent_version`, `demo.workflow_template_id`, `demo.task_type`, `demo.artifact_type`, `demo.tier`, `demo.error_category`) → OK in Dimensions.

**Metric namespace:** `DemoSDLC/Agent`. IAM policy attached to EC2 instance role restricts `cloudwatch:PutMetricData` to this namespace.

**Mandatory boilerplate (every agent's `main.py`):**
```python
from opentelemetry import trace
tracer = trace.get_tracer(__name__)

with tracer.start_as_current_span("AgentExecution") as span:
    span.set_attribute("demo.workflow_template_id", workflow_template_id)
    span.set_attribute("demo.workflow_id", workflow_id)
    span.set_attribute("demo.agent_run_id", agent_run_id)
    span.set_attribute("demo.step_id", step_id)
    span.set_attribute("demo.actor_id", actor_id)
    span.set_attribute("demo.data_classification", "INTERNAL")
    span.set_attribute("demo.error_category", "none")
    try:
        # business logic
        ...
    except Exception as e:
        span.set_attribute("demo.error_category", "system_failure")
        span.record_exception(e)
        raise
```

**What agents MUST NOT log to CloudWatch (Guideline §9.3):**
- Raw prompt text
- Raw model output text
- Unmasked PII or secrets
- Full document/code payload contents

Content lives in S3 (artifacts) or in agent-to-agent payload storage (claim-check). CloudWatch carries metadata only.

---

## 11. Workflow Streams Strategy

Demo retains Temporal Workflow Streams for backend → frontend push (D6). The Python SDK's `temporalio.contrib.workflow_streams` is preferred but not yet stable in all SDK versions. The demo uses a feature-flagged strategy:

**Pattern A — `temporalio.contrib.workflow_streams` (preferred):**
```python
# In workflow:
self.status_stream = workflow.workflow_stream("status")
await self.status_stream.write({"event": "step_started", "step_id": step_id})

# In FastAPI SSE bridge:
async with client.get_workflow_handle(workflow_id) as handle:
    async for event in handle.read_stream("status"):
        yield f"data: {json.dumps(event)}\n\n"
```

**Pattern B — Update API + side-channel (fallback):**
```python
# In workflow: use workflow.update for ad-hoc state queries; emit events
# to a Redis pub/sub channel from the activity layer.
# Activity:
await redis.publish(f"workflow:{workflow_id}", json.dumps(event))

# FastAPI SSE bridge:
pubsub = redis.pubsub()
await pubsub.subscribe(f"workflow:{workflow_id}")
async for message in pubsub.listen():
    yield f"data: {message['data']}\n\n"
```

**Runtime selection:** At worker startup, try `from temporalio.contrib.workflow_streams import WorkflowStream`. On `ImportError`, set env `WORKFLOW_STREAMS_PATTERN=B` and use Redis. The FastAPI bridge reads the same env var to pick the consumer side.

**Redis service:** When Pattern B is active, docker-compose includes a 7th container `redis:7-alpine` on port 6379. When Pattern A is active, this container is omitted.

This decision is documented in Prompt_04 with both code paths fully written.

---

## 12. Demo Simplifications vs Production

| Concern | Production (Guideline) | Demo |
|---|---|---|
| Inbound auth | JnJ Entra ID + PingID JWT validated by AgentCore Identity | Mock persona JSON in `requested_by` block (D9) |
| Outbound credentials | AgentCore Token Vault | EC2 IMDSv2 instance profile (boto3 picks up automatically) |
| Network posture | Intranet-only; AgentCore Gateway is the only egress | Public EC2 with security group rules; boto3 to AWS managed services directly |
| Span attribute prefix | `jnj.*` | `demo.*` (D1) |
| Metric namespace | `JnJSDLC/Agent` or similar | `DemoSDLC/Agent` |
| AgentCore Gateway (MCP & external APIs) | Required; Cedar policies enforce | Not deployed; Cedar policies are documentation-only (D3) |
| AgentCore Memory | Managed service, isolated by actor_id+session_id | **Auto-created by the toolkit** (`STM_ONLY` mode, 30-day event expiry) unless you pass `agentcore configure --disable-memory`. Per-agent memory resource: `<agent_id>_mem-<random>`. Visible via `aws bedrock-agentcore-control list-memories`. |
| ECR repository | Pre-created in CI/CD | **Auto-created on first `agentcore deploy`** when `--deployment-type container` is used (the live deployment uses this mode). Single repo: `agentcore-demo-test1`. `direct_code_deploy` mode skips ECR. |
| CodeBuild project | Pre-created | **Auto-created per agent**: `bedrock-agentcore-<agent_id>-builder`. |
| IAM execution role | Pre-created with least privilege | **Auto-created if `--execution-role` not passed**: `AmazonBedrockAgentCoreSDKRuntime-<region>-<random>` (runtime) + `AmazonBedrockAgentCoreSDKCodeBuild-<region>-<random>` (build). |
| Temporal cluster | Platform-managed | Self-hosted single-node `temporalio/auto-setup:1.22` in docker-compose |
| Multiple orchestrations | Many, all calling shared agents | One: `brd-from-audio-v1` |
| Evidence Vault | GxP-compliant Object Lock S3 + RIM retention | S3 bucket with default versioning; retention via lifecycle rules (configurable) |
| Frontend hosting | Internal portal behind SSO | Local dev (`localhost:3000`) or EC2 public IP |

Every prompt and deliverable that interacts with one of these concerns explicitly calls out the production equivalent. The demo's purpose is to demonstrate the contract; the simplifications above never weaken contract enforcement at the agent/workflow layer.

---

## 13. Comment Density Policy

Code comments are sparse. Documentation lives in Markdown.

**Rule:** A comment exists only when the WHY is non-obvious. Examples of legitimate comments:
- A hidden invariant (`# step_id is monotonically increasing; do not reuse on retry`)
- A workaround for a known bug (`# AWS Transcribe rejects MediaFormat="webm"; we pass "ogg" which works for opus`)
- A subtle constraint (`# This must run before any S3 client init; otherwise IMDS lookup fails inside docker-compose`)

**Rule:** Never write a comment that paraphrases the code. `# Send to S3` above `s3.put_object(...)` is noise.

**Rule:** Every payload-builder call site has a one-line comment naming the block: `# 8-block: Block 4 (Financial) — manual_baseline_hours sourced from task.complexity_tier mapping`.

**Rule:** Every HITL transition has a one-line comment naming the step_type and timeout.

Detailed explanations belong in Markdown next to the code (e.g., `PAYLOAD_SCHEMA.md`, prompt files).

---

## 14. Companion Documents

### Reference Deliverables (numbered)

| # | File | Purpose |
|---|---|---|
| 1 | `Deliverable_1_Infrastructure_Cost_Report.md` | Cost breakdown for both us-east-1 and eu-central-1 |
| 2 | `Deliverable_2_Reference_Strands_Agent_Code.md` | Reference Python for the 3 agents + shared modules |
| 3 | `Deliverable_3_Implementation_Prompts.md` | Historical (superseded by Prompts 00–07) |
| 4 | `Deliverable_4_Temporal_Operations_Guide.md` | Temporal patterns, both Workflow Stream patterns, troubleshooting |
| 5 | `Deliverable_5_CopilotKit_AGUI_Guide.md` | CopilotKit integration, AG-UI HttpAgent, generative UI cards |
| 6 | `Deliverable_6_AWS_Operations_Guide.md` | AWS bootstrap from scratch, S3 buckets, RDS, EC2, IAM, AgentCore CLI |
| 7 | `Deliverable_7_CloudWatch_Telemetry_Guide.md` | OTel + ADOT architecture, dashboard, queries, alarms |
| 8 | `Deliverable_8_Frontend_Architecture_Guide.md` | Dual-realm React, provider layout, testing |
| 9 | `Deliverable_9_Multi_Framework_AGUI_Guide.md` | LangGraph/CrewAI AG-UI adapters, multi-framework strategy |

### Prompts (sequential, atomic)

| # | File | What it builds |
|---|---|---|
| 00 | `Prompt_00_Infra_Bootstrap.md` | VPC, S3, RDS, IAM, EC2, AgentCore CLI bootstrap |
| 01 | `Prompt_01_AgentCore_Deployment.md` | 3 agents skeleton + 5 endpoints + S3 ZIP deploy |
| 02 | `Prompt_02_Agent_Telemetry.md` | 8-block payload builder + OTel + ADOT |
| 03 | `Prompt_03_Backend_Foundation.md` | FastAPI app, DB migrations, settings |
| 04 | `Prompt_04_Temporal_Workflow.md` | Workflow + activities + SSE bridge (Pattern A/B) |
| 05 | `Prompt_05_Frontend.md` | Next.js + CopilotKit + mic recording (D13) |
| 06 | `Prompt_06_E2E_Integration.md` | 11 end-to-end tests |
| 07 | `Prompt_07_Metrics_Dashboard.md` | Metrics page + backend metrics API (D14) |

### Side Guides

| File | Purpose |
|---|---|
| `ARCHITECTURE_DATAFLOW_GUIDE.md` | Detailed architecture and data flow (deep dive) |
| `REPO_STRATEGY.md` | 4-repo polyrepo strategy, CI/CD posture |
| `GIT_WORKFLOW.md` | Branch strategy, commit rules, PR process |
| `DEVELOPER_ONBOARDING.md` | New developer environment setup |
| `DEMO_GELISTIRME_KILAVUZU.md` | Turkish: step-by-step development guide |
| `PERSISTENT_FILESYSTEM_GUIDE.md` | AgentCore Persistent FS (POSIX) details |
| `PHASE_PLAN.md` | Phase-by-phase roadmap with gates |
| `V5_IMPLEMENTATION_PLAN.md` | Historical migration plan to v5 |
| `V5_TELEMETRY_INTEGRATION_ANALYSIS.md` | Telemetry deep dive (Turkish) |
| `WORKSPACE_SETUP_GUIDE.md` | VS Code workspace setup |
| `PAYLOAD_SCHEMA.md` | (P3) Canonical 8-block schema with examples per agent |
| `APPLY_PLAN.md` | (P1–P13) Documentation cleanup execution plan |

### Audit / Status Reports

| File | Purpose |
|---|---|
| `DOCUMENT_AUDIT_REPORT.md` | Round 1 audit (historical) |
| `CONSISTENCY_AUDIT_REPORT_ROUND2.md` | Round 2 audit (historical) |
| `CONSISTENCY_AUDIT_REPORT_ROUND3.md` | (P13) Round 3 audit — current state |
| `FILE_STATUS_REPORT.md` | Current freshness status per file |
| `DELIVERABLE_MAPPING.md` | Prompt-to-Deliverable mapping |
| `PROJECT_STARTER.md` | One-shot workspace creation script |
| `PROMPT_TESTING_SECTIONS.md` | Test directory layout per phase |

---

## 15. Consolidated Tech Reference

### 15.1 `bedrock-agentcore` Runtime SDK (Python)

**Package:** `bedrock-agentcore`. Requires Python 3.11+.

**Pattern:** Use `BedrockAgentCoreApp` with `@app.entrypoint` for the `/invoke` handler. Register the other four endpoints (`/health`, `/metadata`, `/capabilities`, `/metrics`) on the same underlying app.

**Quick pattern:**
```python
from bedrock_agentcore.runtime import BedrockAgentCoreApp

app = BedrockAgentCoreApp()

@app.entrypoint
def handler(request: dict) -> dict:
    # Validate input, run agent logic, build 8-block payload, return.
    return {"status": {...}, "resources": {...}, ...}  # all 8 blocks

# /health, /metadata, /capabilities, /metrics added below using app.fastapi.add_api_route(...)
```

Deployment (default `direct_code_deploy` mode): `agentcore configure -e main.py --protocol HTTP --non-interactive` + `agentcore deploy --env KEY=VALUE --auto-update-on-conflict`. The toolkit zips the directory, uploads to S3, runs CodeBuild to produce a container image, and registers the runtime — all without you writing a Dockerfile. To verify state: `agentcore status`. To tear down: `agentcore destroy [--delete-ecr-repo]`.

### 15.2 Strands SDK (Agents 1 & 2)

**Package:** `strands-agents`. Used for agent definition (tools, system prompt, reasoning) only. Runtime is always `bedrock-agentcore`.

**Docs:**
- https://strandsagents.com — Strands Agents docs (fixed typo from earlier version)
- https://docs.aws.amazon.com/prescriptive-guidance/latest/agentic-ai-frameworks/strands-agents.html — AWS Prescriptive Guidance (fixed typo)

### 15.3 LangGraph (Agent 3)

**Package:** `langgraph`. Used to define the 4-node review graph: `analyze_quality`, `scan_pii`, `check_policy`, `generate_report`.

Quick pattern: build a `StateGraph(ReviewerState)`, add nodes with `add_node`, wire edges with `add_edge`, `set_entry_point`, then `compile()`. Each node returns a partial state dict (idiomatic LangGraph; merger handled by the reducer).

### 15.4 AG-UI Protocol

**Docs:**
- https://docs.ag-ui.com/
- https://github.com/ag-ui-protocol/ag-ui
- https://docs.copilotkit.ai/backend/ag-ui

**Note:** Events are ephemeral (not persisted). Reconnection uses `Last-Event-ID`. CopilotKit `useAgent` handles SSE management.

### 15.5 CopilotKit React SDK

**Pinned:** `^1.50.0` (verify against npm registry at install time; if not available, pin to the latest stable 1.x).

**Provider scope:** `<CopilotKit>` lives at `app/workspace/[wfId]/layout.tsx`, NOT at root. The `/metrics` page does not use CopilotKit.

### 15.6 AWS Bedrock — Claude Sonnet 4.6

**Model ID (Mayıs 2026 stable):** `anthropic.claude-sonnet-4-6`.

Other current options in us-east-1 (verify with `aws bedrock list-foundation-models`):
- `anthropic.claude-sonnet-4-6` — recommended default (May 2026 stable)
- `anthropic.claude-sonnet-4-5-20250929-v1:0` — date-pinned stable
- `anthropic.claude-opus-4-7` — highest capability (cost trade-off)
- `anthropic.claude-haiku-4-5-20251001-v1:0` — fastest, cheapest

The older `anthropic.claude-sonnet-4-6` (Oct 2024) is still available but is the previous generation; do not pin new code to it.

Use `boto3.client("bedrock-runtime").invoke_model(...)`. Implement exponential backoff with jitter for `ThrottlingException`.

### 15.7 Amazon Transcribe

Async only (`StartTranscriptionJob`). Set `IdentifyLanguage=True` + `LanguageOptions=["en-US","es-US","fr-FR","de-DE","zh-CN"]`. `DataAccessRoleArn` is a top-level parameter of `start_transcription_job`, NOT nested inside `Settings`. Supported formats include MP3, MP4, WAV, FLAC, OGG, AMR, WEBM (opus inside webm is treated as ogg-compatible).

### 15.8 Temporal Python SDK

**Pinned:** `temporalio >= 1.6.0`. **Server pinned:** `temporalio/auto-setup:1.22` (Server compatible with SDK 1.6+).

**Correct imports:**
```python
from temporalio import workflow, activity   # NOT: from temporalio.activity import activity
from temporalio.client import Client
from temporalio.worker import Worker
from datetime import timedelta

@workflow.defn
class BrdFromAudioWorkflow:
    @workflow.run
    async def run(self, audio_uri: str) -> dict:
        ...

@activity.defn
async def invoke_agent_runtime(...):
    ...
```

**Event History limits:** 50 MB total, 51,200 events. Use Claim-Check for any payload >1 MB. Use ContinueAsNew when wall-clock duration > 24h OR event count > 50% of cluster limit.

### 15.9 Workflow Streams (Pattern A or B)

See §11. Two-pattern strategy with feature flag. Both patterns documented in Prompt_04 with complete code.

### 15.10 FastAPI + structlog + OpenTelemetry

**Pinned:** FastAPI ≥ 0.110, structlog ≥ 24.1, opentelemetry ≥ 1.24.

**OTel imports (correct paths):**
```python
from opentelemetry import trace
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.sdk.resources import Resource, DEPLOYMENT_ENVIRONMENT_NAME  # post-1.20 name
```

Export to ADOT at `http://adot-collector:4317` (OTLP gRPC).

### 15.11 Recharts (Metrics Dashboard, P12)

**Pinned:** `recharts@^2.12.0`. Used for `LineChart`, `BarChart`, `AreaChart` in the `/metrics` page.

### 15.12 pdfkit / wkhtmltopdf

For Markdown → PDF rendering of the final BRD. Install `wkhtmltopdf 0.12.6` system binary in the FastAPI container.

### 15.13 Next.js App Router

**Pinned:** Next.js 14+ (latest stable 14.x). CopilotKit route uses edge runtime.

---

## Appendix A — Environment Variables

| Variable | Default | Description |
|---|---|---|
| `AWS_REGION` | `us-east-1` | Primary AWS region |
| `TEMPORAL_HOST` | `temporal-server` | Temporal Server gRPC host |
| `TEMPORAL_PORT` | `7233` | Temporal gRPC port |
| `RDS_HOST` | (from `.vpc_env`) | PostgreSQL endpoint |
| `RDS_DB_TEMPORAL` | `temporal` | Temporal persistence DB |
| `RDS_DB_TEMPORAL_VISIBILITY` | `temporal_visibility` | Temporal visibility DB |
| `RDS_DB_APP` | `agentcore_demo_test1` | Application DB |
| `RDS_USER_APP` | `app_user` | Application DB user |
| `RDS_USER_TEMPORAL` | `temporal_user` | Temporal DB user |
| `RDS_PASSWORD_APP` | (from `.vpc_env`) | `app_user` password (generated in Prompt_00) |
| `RDS_PASSWORD_TEMPORAL` | (from `.vpc_env`) | `temporal_user` password (generated in Prompt_00) |
| `S3_BUCKET_AUDIO` | (from `.vpc_env`) | `${PROJECT}-audio-uploads-${ACCOUNT_ID}` |
| `S3_BUCKET_ARTIFACTS` | (from `.vpc_env`) | `${PROJECT}-artifacts-${ACCOUNT_ID}` |
| `S3_BUCKET_CLAIMCHECK` | (from `.vpc_env`) | `${PROJECT}-claimcheck-${ACCOUNT_ID}` |
| `S3_BUCKET_CODE` | (from `.vpc_env`) | `${PROJECT}-code-${ACCOUNT_ID}` (agent ZIPs) |
| `CLAIM_CHECK_THRESHOLD_BYTES` | `1048576` | 1 MB |
| `BEDROCK_MODEL_ID` | `anthropic.claude-sonnet-4-6` | LLM model |
| `HITL_MAX_ROUNDS` | `5` | Max clarification rounds |
| `HITL_TIMEOUT_SECONDS` | `900` | 15 min per question |
| `WORKFLOW_TIMEOUT_SECONDS` | `3600` | 1 hour total |
| `SELF_CORRECTION_CAP` | `3` | D8: 2 revisions after initial |
| `AGENTCORE_ARN_TRANSCRIBER` | (from Prompt_01) | `arn:aws:bedrock-agentcore:us-east-1:<acct>:runtime/agent_1_transcriber` |
| `AGENTCORE_ARN_DRAFTER` | (from Prompt_01) | `arn:aws:bedrock-agentcore:us-east-1:<acct>:runtime/agent_2_drafter` |
| `AGENTCORE_ARN_REVIEWER` | (from Prompt_01) | `arn:aws:bedrock-agentcore:us-east-1:<acct>:runtime/agent_3_reviewer` |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | `http://adot-collector:4317` | OTLP gRPC |
| `OTEL_RESOURCE_ATTRIBUTES` | (composed) | `service.name=<svc>,deployment.environment=dev` |
| `WORKFLOW_STREAMS_PATTERN` | `A` or `B` | D6 feature flag (set by worker at startup) |
| `REDIS_URL` | `redis://redis:6379` | Only if Pattern B active |
| `LOG_LEVEL` | `INFO` | structlog level |
| `METRICS_NAMESPACE` | `DemoSDLC/Agent` | CloudWatch namespace |

---

## Appendix B — Mock User Auth (Demo Only)

```python
# DEMO ONLY. Production uses JnJ Entra ID + PingID per Guideline §15.1.1.
MOCK_PERSONAS = {
    "ba-sap-mm": {
        "user_id": "demo-user-001",
        "email": "ba.sap.mm@demo.local",
        "roles": ["business_analyst"],
        "group": {"group_id": "sap-mm", "group_type": "module", "group_name": "SAP MM"},
        "project": {"project_id": "proj-demo", "project_name": "Demo BRD",
                    "system": "DEMO", "sector": "MT", "vertical": "ERP"},
    },
    "ba-sap-fico": {
        "user_id": "demo-user-002",
        "email": "ba.sap.fico@demo.local",
        "roles": ["business_analyst", "metrics_viewer"],
        "group": {"group_id": "sap-fico", "group_type": "module", "group_name": "SAP FICO"},
        "project": {"project_id": "proj-demo", "project_name": "Demo BRD",
                    "system": "DEMO", "sector": "MT", "vertical": "ERP"},
    },
}
```

Frontend renders a persona selector at landing. Selected persona is held in a cookie/localStorage and injected into every API call as the `requested_by` block. This is the **only** auth mechanism for this demo — no Cognito, no OAuth, no JWT validation.

---

## Appendix C — Error Code Dictionary

Per Guideline Appendix B. Block 1 (Status) `error_category` enum:

| error_category | Example error_code | Retryable? |
|---|---|---|
| `input_validation` | `MISSING_REQUIRED_FIELD`, `INVALID_ARTIFACT_REF` | No |
| `authorization` | `USER_NOT_AUTHORIZED`, `TOOL_SCOPE_DENIED` | No |
| `tool_failure` | `MCP_SERVER_TIMEOUT`, `TRANSCRIBE_JOB_FAILED` | Yes (transient) |
| `llm_failure` | `MODEL_RATE_LIMIT`, `MODEL_TIMEOUT` | Yes |
| `quality_gate` | `CONFIDENCE_BELOW_THRESHOLD`, `UNSUPPORTED_CLAIMS_FOUND` | No, after self-correction cap (D8); then HITL |
| `policy_violation` | `PII_POLICY_BLOCK`, `SECRET_DETECTED` | No |
| `dependency_unavailable` | `BEDROCK_UNAVAILABLE`, `S3_UNAVAILABLE` | Yes |
| `timeout` | `AGENT_TIMEOUT`, `WORKFLOW_TIMEOUT` | Workflow-dependent |
| `claim_check_failure` | `CLAIM_CHECK_UPLOAD_FAILED`, `CLAIM_CHECK_HASH_MISMATCH` | Yes (transient) |
| `evidence_pack_input` | `EVIDENCE_PACK_INPUT_INCOMPLETE` | No; escalate to orchestrator |
| `unknown` | `UNCLASSIFIED_ERROR` | Manual triage |

---

## Appendix D — Status Code Dictionary

Per Guideline Appendix B. Block 1 (Status) `code` enum:

| code | Meaning |
|---|---|
| `success` | Task completed successfully |
| `partial_success` | Output produced with limitations or warnings |
| `partial_failure` | Some steps failed; limited output or diagnostic result exists |
| `failed` | Task failed completely |
| `requires_human_review` | HITL required; workflow opens `hitl_review` or `hitl_approval` step |
| `blocked` | Missing permission, dependency, input, or policy approval |
| `cancelled` | Task cancelled (user signal or orchestrator decision) |
| `timeout` | Task exceeded allowed execution time |

Workflow-level `BRDState` enum (in app DB): `PENDING`, `IN_PROGRESS`, `AWAITING_HUMAN`, `APPROVED`, `REJECTED`, `FAILED`, `MAX_ITERATIONS`.

---

## Appendix E — Quick Start

Once Prompts 00–07 have all executed successfully on the developer machine:

```bash
# On the Mac dev environment
cd /Users/ugurgocen/projects/agentcore-demo-test1/agentcore-demo-test1-backend
docker-compose up -d
docker-compose ps   # expect 6 (or 7 if Pattern B) services Up

# Verify the stack
curl http://localhost:8000/health                 # FastAPI healthy
open http://localhost:8081                        # Temporal UI
open http://localhost:3000                        # Frontend
open http://localhost:3000/metrics                # Metrics dashboard

# Run an end-to-end smoke test
cd ../agentcore-demo-test1-backend
uv run pytest tests/e2e/test_brd_smoke.py -v
```

If running on the EC2 instance (after Prompt_00 stands it up):

```bash
ssh ec2-user@<ec2-public-ip>
cd /home/ec2-user/agentcore-demo-test1/agentcore-demo-test1-backend
docker-compose up -d
# Then visit http://<ec2-public-ip>:3000 from your browser
```

---

**END OF DOCUMENT**

*This document is the shared baseline for all coding sessions. Read in full before writing code. All D1–D14 decisions here are final unless revised by the project owner.*
