# Master Phase Plan: AgentCore Demo — Audio → BRD Reference Orchestration

**Version:** 2.0
**Date:** 2026-05-15
**Target Audience:** Junior developers and AI coding assistants (Claude Sonnet, Gemini Flash) following the plan step by step
**Build Philosophy:** One phase, one deliverable, verify, move on.
**Upstream Reference:** `Deliverable_0_PROJECT_CONTEXT.md` (PCD) is the single source of truth for all decisions (D1–D14), 8-block payload schema, identifier hierarchy, telemetry model, and demo simplifications.

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Master Phase / Gate Table](#2-master-phase--gate-table)
3. [Per-Phase Detail Sections](#3-per-phase-detail-sections)
4. [Gate Pass Protocol](#4-gate-pass-protocol)
5. [Document Cross-Reference Map](#5-document-cross-reference-map)
6. [Checkpoint Design Philosophy](#6-checkpoint-design-philosophy)
7. [Key Decisions (pointer to PCD §3)](#7-key-decisions)
8. [Appendix: Quick Reference Card](#8-appendix-quick-reference-card)

---

## 1. Introduction

### Purpose

This document defines an **8-phase sequential build plan** for the AgentCore demo project. Each phase produces one verifiable deliverable. The plan is designed for AI coding assistants and junior developers that require explicit, unambiguous, step-by-step instructions with no inferred context.

### Principle

Each phase performs **exactly one task**, verifies that it works, and only then advances. There are no parallel tracks, no optional branches, no decisions left to the model's discretion. Every phase has a single, verifiable deliverable.

### How to Use This Document

1. Read PCD (`Deliverable_0_PROJECT_CONTEXT.md`) first — it defines the architecture and frozen decisions.
2. Read this document second — it sequences the build.
3. For each phase, load the **Companion Prompt** file into the AI coding model along with PCD.
4. Use the **Companion Docs** column to locate reference material.
5. Do NOT begin Phase N+1 until Phase N's Exit Gate has passed.
6. If an Exit Gate fails, follow the Failure Protocol in §4 before retrying.

---

## 2. Master Phase / Gate Table

| Phase | Name | Entry Gate | Scope | Exit Gate (Pass Criteria) | Companion Prompt | Companion Docs |
|---|---|---|---|---|---|---|
| 0 | Infra Bootstrap | AWS account ready, AWS CLI configured | VPC (2 AZs), Security Groups, S3 (4 buckets), RDS (3 DBs), EC2 + Docker, IAM roles, AgentCore CLI, CloudWatch log groups, AgentCore Persistent FS setup | EC2 reachable, Docker working, AgentCore CLI installed, all AWS resources tagged and discoverable | `Prompt_00_Infra_Bootstrap.md` | `Deliverable_6_AWS_Operations_Guide.md` |
| 1 | Agent Skeleton & AgentCore Deployment | Phase 0 passed | Write 3 agent `main.py` files (5 endpoints each), `requirements.txt`, deploy via S3 ZIP, smoke test each runtime | All 3 agents return valid 8-block placeholder responses via `boto3.invoke_agent_runtime` | `Prompt_01_AgentCore_Deployment.md` | `Deliverable_2_Reference_Strands_Agent_Code.md` |
| 2 | Agent Telemetry (`demo.*` + ADOT) | Phase 1 passed | `agents/common/payload_builder.py`, `agents/common/otel_setup.py`, ADOT Collector configuration, integrate OTel boilerplate into all 3 agents | OTel spans visible in CloudWatch via ADOT, all 8 blocks reconstructable from span attributes per PCD §10 | `Prompt_02_Agent_Telemetry.md` | `Deliverable_7_CloudWatch_Telemetry_Guide.md` |
| 3 | FastAPI Backend Foundation | Phase 2 passed | FastAPI app, mock auth, RDS SQLAlchemy 2.0 models (canonical `evidence_packs` per D10), Alembic migrations, S3 claim-check, settings, health endpoints | `/health` 200, Alembic up-to-date, docker-compose up succeeds with 6 services healthy | `Prompt_03_Backend_Foundation.md` | PCD §5, `Deliverable_4_Temporal_Operations_Guide.md` (background) |
| 4 | Temporal Workflow + Workflow Streams | Phase 3 passed | `BrdFromAudioWorkflow`, activities, signal handlers, Workflow Stream (Pattern A + Pattern B fallback per D6), Evidence Pack persistence | Start workflow via API, agents invoked end-to-end, HITL signal works, all 6 workflow outcomes persist Evidence Pack | `Prompt_04_Temporal_Workflow.md` | `Deliverable_4_Temporal_Operations_Guide.md` |
| 5 | Frontend (Next.js + CopilotKit + Mic) | Phase 4 passed | Next.js 14 App Router, CopilotKit 1.50+ scoped to `/workspace/[wfId]`, AG-UI client, mic recording (D13) + file upload, dual-stream merger, HITL UI | All chat features render, audio (record or upload) starts workflow, AG-UI events flow, HITL/approval works in browser | `Prompt_05_Frontend.md` | `Deliverable_5_CopilotKit_AGUI_Guide.md`, `Deliverable_8_Frontend_Architecture_Guide.md` |
| 6 | End-to-End Integration | Phase 5 passed | 11 sequential E2E tests covering all 6 workflow outcomes, restart, telemetry assertions, Evidence Pack verification | Full demo run end-to-end via test harness, `docker compose down && up` clean restart, CloudWatch spans verified | `Prompt_06_E2E_Integration.md` | `PROMPT_TESTING_SECTIONS.md` |
| 7 | Metric Dashboard | Phase 6 passed | `/metrics` page (Recharts) + `/api/v1/metrics/*` endpoints, CloudWatch `get_metric_data` integration, Guideline §9.6 computed metrics (D14) | Dashboard renders with summary cards, time series, drill-down by template/project; data matches actual workflow runs | `Prompt_07_Metrics_Dashboard.md` | `Deliverable_7_CloudWatch_Telemetry_Guide.md` |

---

## 3. Per-Phase Detail Sections

---

### Phase 0: Infra Bootstrap

**Entry Gate:**
- AWS account exists and is accessible via CLI (`aws sts get-caller-identity` returns valid credentials).
- Default region set (`us-east-1` primary; `eu-central-1` documented as alternate but not provisioned).
- Sufficient IAM permissions to create VPC, Security Groups, S3, RDS, EC2, IAM roles, and CloudWatch resources.

**Scope:**
- Create VPC with 2 public subnets in **2 different AZs** (us-east-1a + us-east-1b — required for RDS subnet group).
- Create Security Groups: EC2 SG (ingress 22, 80, 443, 3000, 7233, 8000, 8081), RDS SG (ingress 5432 from EC2 SG).
- Create 4 S3 buckets: `${PROJECT}-audio-uploads-${ACCOUNT_ID}`, `${PROJECT}-artifacts-${ACCOUNT_ID}`, `${PROJECT}-claimcheck-${ACCOUNT_ID}`, `${PROJECT}-code-${ACCOUNT_ID}` (agent ZIPs).
- Create RDS PostgreSQL `db.t4g.micro` with 3 databases: `temporal`, `temporal_visibility`, `agentcore_demo_test1`; users `temporal_user` and `app_user` with generated passwords exported to `.vpc_env`.
- Create EC2 `t3.large` in public subnet 1, install Docker + docker-compose v2 + AWS CLI v2 + AgentCore CLI via user-data.
- Create custom least-privilege IAM role for EC2 (NO `AmazonS3FullAccess`/`AmazonBedrockFullAccess` — see Deliverable_6 for the exact policy JSON).
- Create 4 CloudWatch log groups (canonical per PCD §10): `/agentcore-demo-test1/agent-logs` (shared by all 3 agents — filter via `demo.agent_id` attribute), `/agentcore-demo-test1/adot-collector`, `/agentcore-demo-test1/fastapi`, `/agentcore-demo-test1/temporal-worker`. AWS additionally auto-creates `/aws/bedrock-agentcore/runtimes/<agent_id>-<random>-<qualifier>` vended log groups per agent on first invocation.
- Set up AgentCore Persistent Filesystem (mount path `/mnt/data/sdlc-payloads-claimcheck`) — file system + access point.
- Bootstrap AgentCore CLI on EC2 (one-time per account).
- Export all created identifiers (subnet IDs, SG IDs, bucket names, RDS endpoint, passwords) to a `.vpc_env` file consumable by later phases.

**Explicitly NOT in Scope:**
- Any application code or containers.
- Agent runtime or Bedrock configuration.
- Temporal server or worker setup (Phase 3 brings it up via docker-compose).
- Frontend code.
- SSL/TLS certificates or domain setup.
- Multi-AZ RDS or high-availability (single-AZ RDS; the 2-AZ requirement is purely for the subnet group).
- AgentCore Gateway (production-only per D3).

**Files Created/Modified:**
- `agentcore-demo-test1-infra/scripts/00_bootstrap_admin.sh`
- `agentcore-demo-test1-infra/scripts/01_vpc_subnets_sg.sh`
- `agentcore-demo-test1-infra/scripts/02_s3_buckets.sh`
- `agentcore-demo-test1-infra/scripts/03_rds_postgres.sh`
- `agentcore-demo-test1-infra/scripts/04_iam_roles.sh`
- `agentcore-demo-test1-infra/scripts/05_ec2_instance.sh`
- `agentcore-demo-test1-infra/scripts/06_cloudwatch_log_groups.sh`
- `agentcore-demo-test1-infra/scripts/07_agentcore_persistent_fs.sh`
- `agentcore-demo-test1-infra/scripts/99_teardown.sh`
- `agentcore-demo-test1-infra/iam-policies/ec2-instance-role.json` (custom least-privilege per PCD §4)
- `agentcore-demo-test1-infra/iam-policies/transcribe-data-access.json`
- `.vpc_env` (gitignored; carries all generated IDs and passwords for downstream phases)

**Dependencies:** None. First phase.

**Checkpoint Markers:**

| Checkpoint | Command to Run | Expected Output |
|---|---|---|
| CP-0.1 | `aws ec2 describe-vpcs --filters "Name=tag:Project,Values=agentcore-demo-test1" --query 'Vpcs[0].VpcId'` | VPC ID string |
| CP-0.2 | `aws ec2 describe-subnets --filters "Name=tag:Project,Values=agentcore-demo-test1" --query 'length(Subnets)'` | `2` (subnets in 2 AZs) |
| CP-0.3 | `aws s3api list-buckets --query 'length(Buckets[?starts_with(Name, \`agentcore-demo-test1-\`)])'` | `4` |
| CP-0.4 | `aws rds describe-db-instances --query 'DBInstances[?DBInstanceIdentifier==\`agentcore-demo-test1-db\`].DBInstanceStatus' --output text` | `available` |
| CP-0.5 | `psql -h $RDS_HOST -U app_user -d agentcore_demo_test1 -c "SELECT 1;"` | `1` |
| CP-0.6 | `aws ec2 describe-instances --filters "Name=tag:Project,Values=agentcore-demo-test1" --query 'Reservations[0].Instances[0].State.Name' --output text` | `running` |
| CP-0.7 | `ssh -i <key> ec2-user@<EC2_IP> "docker --version && docker compose version"` | Both version strings printed |
| CP-0.8 | `ssh -i <key> ec2-user@<EC2_IP> "docker run --rm hello-world"` | "Hello from Docker!" |
| CP-0.9 | `ssh -i <key> ec2-user@<EC2_IP> "agentcore --version"` | AgentCore CLI version string |
| CP-0.10 | `aws logs describe-log-groups --log-group-name-prefix /agentcore/ --query 'length(logGroups)'` | `4` (or more if extras created) |

**Exit Gate / Pass Criteria:**
- [ ] VPC with 2 public subnets in 2 AZs, IGW, route table attached.
- [ ] EC2 SG allows ingress on 22, 80, 443, 3000, 7233, 8000, 8081; RDS SG allows 5432 from EC2 SG only.
- [ ] All 4 S3 buckets exist with versioning enabled and public access blocked.
- [ ] RDS instance `available`; all 3 databases (`temporal`, `temporal_visibility`, `agentcore_demo_test1`) accept connections.
- [ ] EC2 instance `running`, reachable via SSH, has Docker + docker-compose + AgentCore CLI.
- [ ] AgentCore CLI bootstrap completed (account-level setup).
- [ ] AgentCore Persistent Filesystem + access point created.
- [ ] All 4 CloudWatch log groups exist.
- [ ] `.vpc_env` contains: `VPC_ID`, `SUBNET1_ID`, `SUBNET2_ID`, `EC2_SG_ID`, `RDS_SG_ID`, `EC2_IP`, `RDS_HOST`, `RDS_PASSWORD_APP`, `RDS_PASSWORD_TEMPORAL`, `FS_ARN`, `AP_ARN`, bucket names.

**Failure Protocol:**
If any checkpoint fails: STOP. Do not proceed to Phase 1. Common fixes:
- "IllegalLocationConstraintException" on S3 bucket create in us-east-1 → drop `--create-bucket-configuration` parameter entirely; us-east-1 treats it as the default.
- RDS subnet group failing → verify the two subnets are in two different AZs.
- AgentCore CLI not found → run `pip install bedrock-agentcore-starter-toolkit` in the user-data script.
- IAM permission denied → check the EC2 instance role JSON in `iam-policies/ec2-instance-role.json`; do not grant `*` blanket access.

**Companion Documents:**
- `Deliverable_6_AWS_Operations_Guide.md` (full AWS setup walkthrough, troubleshooting, teardown).

---

### Phase 1: Agent Skeleton & AgentCore Deployment

**Entry Gate:**
- Phase 0 Exit Gate has passed.
- `.vpc_env` exists with all IDs, including `FS_ARN` and `AP_ARN` for Persistent FS.

**Scope:**
- Create the three agent directories per PCD §5 (D4):
  - `agents/agent_1_transcriber/main.py` (Strands)
  - `agents/agent_2_drafter/main.py` (Strands)
  - `agents/agent_3_reviewer/main.py` (LangGraph)
- Each agent uses `BedrockAgentCoreApp()` and registers all 5 required endpoints per PCD §8:
  - `POST /invoke` (entrypoint)
  - `GET /health`
  - `GET /metadata`
  - `GET /capabilities`
  - `GET /metrics`
- Each `/invoke` returns a valid placeholder 8-block payload (D7) — Status, Resources, Timing, Financial, Artifacts, Quality, Tool Calls, Risk. Fields the placeholder genuinely cannot fill use `"Unavailable"`.
- Create minimal `requirements.txt` per agent (`bedrock-agentcore`, `boto3`, `pydantic`, `python-json-logger`). No `opentelemetry-*` (added in Phase 2), no `temporalio` (agents never import Temporal per PCD §4 rule 10).
- Deploy each agent via S3 ZIP: `agentcore configure -e main.py --protocol HTTP --alias <agent-id>-v1 && agentcore deploy`.
- Capture the resulting agent runtime ARNs and append to `.vpc_env`: `AGENTCORE_ARN_TRANSCRIBER`, `AGENTCORE_ARN_DRAFTER`, `AGENTCORE_ARN_REVIEWER`.
- Run a smoke test from EC2 using boto3 `invoke_agent_runtime`; verify all 8 blocks present and the payload validates.

**Explicitly NOT in Scope:**
- OTel/ADOT telemetry (Phase 2).
- Bedrock LLM calls (Phase 2 swaps placeholder logic for real Strands/LangGraph).
- AWS Transcribe call (Phase 2).
- AgentCore Memory integration (Phase 2 / demo simplification per PCD §12).
- Temporal integration (Phase 4).
- Frontend connection (Phase 5).
- Docker containers for agents (S3 ZIP only — there is NO Dockerfile for agents).

**Files Created/Modified:**
- `agents/common/__init__.py`
- `agents/common/payload_builder_stub.py` (returns placeholder 8 blocks; replaced with full builder in Phase 2)
- `agents/agent_1_transcriber/main.py`
- `agents/agent_1_transcriber/requirements.txt`
- `agents/agent_2_drafter/main.py`
- `agents/agent_2_drafter/requirements.txt`
- `agents/agent_3_reviewer/main.py`
- `agents/agent_3_reviewer/nodes.py` (LangGraph 4-node skeleton)
- `agents/agent_3_reviewer/requirements.txt`
- `scripts/agent_deploy.sh` (wrapper invoking `agentcore configure` + `deploy` per agent)
- `.vpc_env` (appended with 3 agent ARNs)

**Dependencies:** Phase 0.

**Checkpoint Markers:**

| Checkpoint | Command to Run | Expected Output |
|---|---|---|
| CP-1.1 | `cd agents/agent_1_transcriber && agentcore configure -e main.py --protocol HTTP --alias agent_1_transcriber-v1` | Configuration successful |
| CP-1.2 | `cd agents/agent_1_transcriber && agentcore deploy` | Runtime ARN printed, deployment active |
| CP-1.3 | repeat CP-1.1/1.2 for `agent_2_drafter` | Same |
| CP-1.4 | repeat CP-1.1/1.2 for `agent_3_reviewer` | Same |
| CP-1.5 | `python scripts/smoke_invoke.py --agent agent_1_transcriber` | JSON response containing all 8 block keys |
| CP-1.6 | Same for `agent_2_drafter` | All 8 blocks present |
| CP-1.7 | Same for `agent_3_reviewer` | All 8 blocks present |
| CP-1.8 | `jq -e '.status.workflow_id != null' < response.json` | `true` (Status block echoes identifiers) |
| CP-1.9 | `jq -e '.resources.cost_reporting_model != null' < response.json` | `true` |
| CP-1.10 | `grep -RE "temporalio|opentelemetry" agents/ \| wc -l` | `0` (no Temporal or OTel imports in Phase 1) |

**Exit Gate / Pass Criteria:**
- [ ] All 3 agent `main.py` files created with `BedrockAgentCoreApp` + `@app.entrypoint`.
- [ ] All 3 agents register `/health`, `/metadata`, `/capabilities`, `/metrics` on the underlying app (in addition to auto-created `/invocations` and `/ping`).
- [ ] All 3 agents deployed via S3 ZIP — no Dockerfile, no ECR.
- [ ] All 3 agent ARNs captured to `.vpc_env`.
- [ ] `invoke_agent_runtime` returns valid 8-block payload (all keys present) from each agent.
- [ ] No `temporalio`, no `opentelemetry-*` in agent code (deferred to Phase 2).

**Failure Protocol:**
If `agentcore deploy` fails: check `bedrock-agentcore-starter-toolkit` is installed and AWS credentials are reachable via IMDS. If smoke test returns a partial payload: re-check `agents/common/payload_builder_stub.py` for missing block keys; do not "fix" by adding OTel — that's Phase 2.

**Companion Documents:**
- `Deliverable_2_Reference_Strands_Agent_Code.md` (skeleton patterns).
- `Deliverable_6_AWS_Operations_Guide.md` §11 (AgentCore deployment specifics).

---

### Phase 2: Agent Telemetry (`demo.*` + ADOT)

**Entry Gate:**
- Phase 1 Exit Gate has passed (all 3 agents reachable, placeholder 8-block payload returns).
- ADOT Collector container ready to be added to the docker-compose stack (config files written in this phase, container started in Phase 3 alongside the rest of the stack).

**Scope:**
- Author the production `agents/common/payload_builder.py` (complete 8-block builder per PCD §6; validate function; identifier echo enforcement).
- Author `agents/common/otel_setup.py` (OTel tracer factory with OTLP gRPC exporter to ADOT on port 4317; `demo.*` attribute helpers per D1 and PCD §10).
- Author `agents/common/identifiers.py` (helpers to read identifiers from inbound request, echo into Block 1 Status).
- Update each agent's `main.py` to wrap business logic in the canonical OTel boilerplate (PCD §10), set all mandatory `demo.*` attributes on every span.
- Replace `payload_builder_stub.py` with the real builder.
- Author the ADOT Collector configuration: receivers (OTLP gRPC :4317), processors (governance filter: drop spans missing `demo.workflow_id` or `demo.workflow_template_id`), exporters (`awsemf` + `awscloudwatchlogs` per Guideline §9.5), pipelines (traces, metrics, logs).
- Author the Phase 3 docker-compose entry for the ADOT Collector container (image `public.ecr.aws/aws-observability/aws-otel-collector:latest`, mount config file, expose 4317).
- Verify spans land in CloudWatch with `demo.*` attributes via Logs Insights query.

**Explicitly NOT in Scope:**
- FastAPI backend (Phase 3).
- Temporal workflow (Phase 4).
- Frontend (Phase 5).
- AgentCore Memory integration (out of scope for demo; see PCD §12).

**Files Created/Modified:**
- `agents/common/payload_builder.py` (replaces `payload_builder_stub.py`)
- `agents/common/otel_setup.py`
- `agents/common/identifiers.py`
- `agents/agent_1_transcriber/main.py` (add OTel boilerplate, full payload)
- `agents/agent_2_drafter/main.py` (add OTel + HITL question logic)
- `agents/agent_3_reviewer/main.py` (add OTel; LangGraph nodes return real 8-block payload)
- `agents/agent_3_reviewer/nodes.py` (4 nodes: `analyze_quality`, `scan_pii`, `check_policy`, `generate_report`)
- `agentcore-demo-test1-infra/adot/otel-collector-config.yaml`
- All 3 `requirements.txt` files: add `opentelemetry-api`, `opentelemetry-sdk`, `opentelemetry-exporter-otlp-proto-grpc` >= 1.24
- Redeploy all 3 agents via S3 ZIP

**New runtime dependencies:** OpenTelemetry SDK (agent side), ADOT Collector (infra side).

**Dependencies:** Phase 1.

**Checkpoint Markers:**

| Checkpoint | Command to Run | Expected Output |
|---|---|---|
| CP-2.1 | `python -c "from agents.common.payload_builder import build_payload, validate_payload; print('ok')"` | `ok` |
| CP-2.2 | `python scripts/smoke_invoke.py --agent agent_1_transcriber --validate` | All 8 blocks present, validate_payload passes |
| CP-2.3 | Same for `agent_2_drafter` and `agent_3_reviewer` | Validation passes |
| CP-2.4 | `yq eval '.processors.filter/governance' agentcore-demo-test1-infra/adot/otel-collector-config.yaml` | Governance filter present, not null |
| CP-2.5 | `aws logs start-query --log-group-name /agentcore-demo-test1/agent-logs --query-string 'fields @message, demo.agent_id \| filter demo.agent_id = "agent_1_transcriber" \| limit 5'` then `aws logs get-query-results` | Records with `demo.agent_id` attribute for the target agent |
| CP-2.6 | `aws cloudwatch list-metrics --namespace DemoSDLC/Agent --query 'length(Metrics)'` | `>= 1` (at least one metric registered by ADOT) |
| CP-2.7 | `grep -RE "^import temporalio\|^from temporalio" agents/ \| wc -l` | `0` (still no Temporal in agent code) |

**Exit Gate / Pass Criteria:**
- [ ] `agents/common/payload_builder.py` produces all 8 blocks per PCD §6 with the canonical names.
- [ ] `validate_payload` enforces: Status block echoes `trace_id`, `workflow_template_id`, `workflow_id`, `agent_run_id`, `step_id`.
- [ ] Every agent wraps its execution in a `tracer.start_as_current_span("AgentExecution")` with all mandatory `demo.*` attributes (`workflow_template_id`, `workflow_id`, `agent_run_id`, `step_id`, `actor_id`, `data_classification`, `error_category`).
- [ ] ADOT Collector config has a governance filter that drops spans missing `demo.workflow_id` or applies a default.
- [ ] Cardinality discipline: high-cardinality fields appear at EMF root, not in `CloudWatchMetrics.Dimensions`.
- [ ] No regression: `/invocations` still returns 200; payload still has all 8 keys.
- [ ] Smoke test produces traces visible in CloudWatch Logs via the ADOT pipeline.

**Failure Protocol:**
If OTel imports fail at agent startup: verify `opentelemetry-exporter-otlp-proto-grpc` is installed (not the `-proto-http` variant). If governance filter drops every span: temporarily lower the filter to log-only and inspect attribute names — `demo.workflow_id` vs `demo.WorkflowId` is a common typo.

**Companion Documents:**
- `Deliverable_7_CloudWatch_Telemetry_Guide.md` (OTel + ADOT architecture).
- `PAYLOAD_SCHEMA.md` (P3 — canonical 8-block reference).

---

### Phase 3: FastAPI Backend Foundation

**Entry Gate:**
- Phase 2 Exit Gate has passed (all agents emit OTel spans with `demo.*` attributes).
- RDS reachable from EC2.

**Scope:**
- Create FastAPI project skeleton per PCD §5 repo layout.
- Implement mock auth (D9): persona JSON in `requested_by` block; no real OAuth/JWT.
- SQLAlchemy 2.0+ models with `DeclarativeBase`. The `evidence_packs` table uses the canonical schema (D10): UUID primary key, JSONB columns for `agent_outputs`, `hitl_exchanges`, `metadata`; columns include `workflow_id`, `workflow_template_id`, `brd_id`, `outcome` (BRDState enum), `rounds_executed`, `max_rounds`, `final_brd_content`, `error`, `created_at`, `completed_at`.
- Alembic configured; one initial migration creates the `evidence_packs` table (D10). Phase 4 only inserts/updates rows.
- Settings module (Pydantic Settings) reads from `.env` which sources `.vpc_env`. All env vars per PCD Appendix A.
- S3 claim-check service: threshold env-configurable (`CLAIM_CHECK_THRESHOLD_BYTES`); read/write with KMS encryption.
- Health endpoints: `/health`, `/health/db`, `/health/s3`.
- Upload endpoint: `POST /api/v1/uploads/audio` (multipart, max 50 MB, content-type validation, S3 putobject to audio-uploads bucket). Required for both file upload AND mic recording (D13).
- Docker compose `version` key removed (deprecated); 6-service stack defined: temporal-server, temporal-ui, temporal-worker, fastapi, nextjs-frontend, adot-collector.

**Explicitly NOT in Scope:**
- Temporal client/worker code (Phase 4).
- Workflow endpoints (Phase 4).
- Frontend connection (Phase 5).
- Metric API endpoints (Phase 7).

**Files Created/Modified:**
- `agentcore-demo-test1-backend/pyproject.toml` (uv-managed; fastapi >= 0.110, sqlalchemy >= 2.0, alembic, pydantic-settings, psycopg2-binary, boto3, structlog, opentelemetry-instrumentation-fastapi)
- `agentcore-demo-test1-backend/app/main.py`
- `agentcore-demo-test1-backend/app/otel_setup.py`
- `agentcore-demo-test1-backend/app/config/settings.py`
- `agentcore-demo-test1-backend/app/api/dependencies.py` (mock auth, DB session, tracer)
- `agentcore-demo-test1-backend/app/api/uploads_routes.py`
- `agentcore-demo-test1-backend/app/api/schemas/uploads.py`
- `agentcore-demo-test1-backend/app/models/evidence_pack.py` (canonical schema per D10)
- `agentcore-demo-test1-backend/app/models/enums.py` (BRDState: PENDING, IN_PROGRESS, AWAITING_HUMAN, APPROVED, REJECTED, FAILED, MAX_ITERATIONS)
- `agentcore-demo-test1-backend/app/services/s3_handler.py`
- `agentcore-demo-test1-backend/app/services/db.py`
- `agentcore-demo-test1-backend/alembic.ini`
- `agentcore-demo-test1-backend/alembic/env.py`
- `agentcore-demo-test1-backend/alembic/versions/0001_initial.py`
- `agentcore-demo-test1-backend/Dockerfile`
- `agentcore-demo-test1-backend/docker-compose.yml` (6-service stack; references ADOT config from Phase 2)
- `agentcore-demo-test1-backend/.env.example` (template; real `.env` is gitignored and sourced from `.vpc_env`)

**Dependencies:** Phase 2.

**Checkpoint Markers:**

| Checkpoint | Command to Run | Expected Output |
|---|---|---|
| CP-3.1 | `cd agentcore-demo-test1-backend && uv sync` | All packages installed |
| CP-3.2 | `uv run alembic upgrade head` | Migration `0001` applied |
| CP-3.3 | `docker compose up -d` | 6 containers Up (or 7 with Redis if Pattern B is active per D6) |
| CP-3.4 | `curl -s http://localhost:8000/health` | `{"status":"healthy",...}` |
| CP-3.5 | `curl -s http://localhost:8000/health/db` | `{"status":"ok","table_count":1}` (only evidence_packs initially) |
| CP-3.6 | `curl -s http://localhost:8000/health/s3` | OK; all 4 buckets reachable |
| CP-3.7 | `curl -s -X POST http://localhost:8000/api/v1/uploads/audio -F "file=@test.mp3"` | `{"s3_uri":"s3://...","content_type":"audio/mpeg"}` |
| CP-3.8 | `curl -s -X POST http://localhost:8000/api/v1/uploads/audio -F "file=@test.webm"` | `{"s3_uri":"s3://...","content_type":"audio/webm"}` (mic recording path works) |
| CP-3.9 | `docker compose logs adot-collector \| grep -i "started"` | ADOT collector running |

**Exit Gate / Pass Criteria:**
- [ ] FastAPI starts cleanly via docker-compose.
- [ ] Alembic migrations apply without error; `evidence_packs` table exists with canonical schema (D10).
- [ ] `/health`, `/health/db`, `/health/s3` all return 200.
- [ ] Audio upload endpoint accepts mp3, wav, m4a, webm; stores to S3; returns S3 URI.
- [ ] ADOT Collector container running and reachable from agents (port 4317).
- [ ] SQLAlchemy uses 2.0+ `DeclarativeBase` (no deprecated `declarative_base` import).
- [ ] docker-compose has NO `version:` key (deprecated).

**Failure Protocol:**
If `/health/db` fails: verify `RDS_HOST`, `RDS_PASSWORD_APP` are correctly sourced from `.vpc_env`. If S3 returns AccessDenied: check the EC2 IAM role policy includes the specific bucket ARNs. If Alembic complains about an existing `evidence_packs` table from a previous build: drop the table and re-run; do NOT skip the migration.

**Companion Documents:**
- PCD (full project architecture).
- `Deliverable_4_Temporal_Operations_Guide.md` (background reading for Phase 4).

---

### Phase 4: Temporal Workflow + Workflow Streams

**Entry Gate:**
- Phase 3 Exit Gate has passed.
- All 3 agent ARNs in `.vpc_env`.

**Scope:**
- Implement `BrdFromAudioWorkflow` in `app/temporal/workflows/brd_from_audio_workflow.py`.
- Implement the canonical activity `invoke_agent_runtime` in `app/temporal/activities/invoke_agent_runtime.py` (single generic wrapper per D4; agent role passed as parameter; constructs full 8-block-aware request payload per PCD §7).
- Implement `persist_evidence_input` (writes/updates `evidence_packs` row) and `publish_stream_event` (Workflow Stream emitter — Pattern A or B per D6 / PCD §11).
- Worker entrypoint at `app/temporal/worker_entrypoint.py`.
- API routes per D11: `POST /api/v1/workflows`, `GET /api/v1/workflows/{id}`, `GET /api/v1/workflows/{id}/stream` (SSE), `POST /api/v1/workflows/{id}/signal/clarification`, `POST /api/v1/workflows/{id}/signal/approval`.
- SSE bridge in `app/services/sse_bridge.py` consumes either `temporalio.contrib.workflow_streams` (Pattern A) or Redis pub/sub (Pattern B). The active pattern is set by env `WORKFLOW_STREAMS_PATTERN` (auto-detected at worker startup).
- Self-correction cap enforced inside the activity wrapper per D8: counts retries with same `(workflow_id, step_id)`, escalates to HITL after 3 total attempts.
- Idempotency: activity uses `(workflow_id, step_id, request_hash)` as the idempotency key.
- Workflow outcomes (BRDState): APPROVED, REJECTED, FAILED, MAX_ITERATIONS. The workflow updates the `evidence_packs` row to the final outcome on termination.

**Explicitly NOT in Scope:**
- Frontend integration (Phase 5).
- Metric API endpoints (Phase 7).
- AgentCore Memory (out of demo scope; conversation context for chat lives in the workflow signal payload — documented simplification).

**Files Created/Modified:**
- `app/temporal/client.py`
- `app/temporal/worker_entrypoint.py`
- `app/temporal/workflows/brd_from_audio_workflow.py`
- `app/temporal/activities/invoke_agent_runtime.py`
- `app/temporal/activities/persist_evidence_input.py`
- `app/temporal/activities/publish_stream_event.py`
- `app/api/workflows_routes.py`
- `app/api/schemas/workflows.py`
- `app/services/sse_bridge.py`
- `app/services/temporal_streams.py` (Pattern A/B feature flag and consumer)
- `agentcore-demo-test1-backend/docker-compose.yml` (add `temporal-worker` service; conditionally add `redis` for Pattern B)
- `agentcore-demo-test1-backend/alembic/versions/0002_evidence_pack_columns.py` (no-op if D10 already includes all columns; otherwise add missing)

**Dependencies:** Phase 3.

**Checkpoint Markers:**

| Checkpoint | Command to Run | Expected Output |
|---|---|---|
| CP-4.1 | `docker compose up -d` then `docker compose ps` | All services Up (6 or 7 depending on D6 pattern) |
| CP-4.2 | `curl -s http://localhost:8081` | Temporal Web UI HTML |
| CP-4.3 | `docker compose logs temporal-worker \| grep "WORKFLOW_STREAMS_PATTERN"` | Either `A` or `B` printed at startup |
| CP-4.4 | `curl -X POST http://localhost:8000/api/v1/workflows -H "Content-Type: application/json" -d '{"audio_s3_uri":"s3://...","persona_id":"ba-sap-mm"}'` | `{"workflow_id":"wf-...","status":"PENDING"}` |
| CP-4.5 | `curl -N http://localhost:8000/api/v1/workflows/<wfid>/stream` | SSE events streaming |
| CP-4.6 | Temporal Web UI: navigate to workflow; verify 3 activities (transcriber → drafter → reviewer) ran | Activity timeline visible |
| CP-4.7 | Simulate HITL clarification: `curl -X POST .../signal/clarification -d '{"answer":"..."}'` | Workflow advances |
| CP-4.8 | Simulate approval: `curl -X POST .../signal/approval -d '{"decision":"approve"}'` | Workflow terminates with outcome=APPROVED |
| CP-4.9 | `psql -c "SELECT outcome FROM evidence_packs WHERE workflow_id='<wfid>'"` | `APPROVED` |
| CP-4.10 | Run a second workflow, request 3 revisions, verify self-correction cap (D8) escalates to HITL on 4th | `hitl_review` step emitted |
| CP-4.11 | `grep -RE "^import temporalio\|^from temporalio" agents/ \| wc -l` | `0` (agents still don't import Temporal) |

**Exit Gate / Pass Criteria:**
- [ ] Workflow starts via `POST /api/v1/workflows` and runs to completion.
- [ ] All 3 agents invoked in sequence via boto3 `invoke_agent_runtime`.
- [ ] Workflow Stream events reach frontend via SSE (Pattern A or B).
- [ ] HITL signals (`clarification`, `approval`) work; workflow durably waits and resumes.
- [ ] Self-correction cap (D8) enforced; 4th attempt escalates to `hitl_review`.
- [ ] All 6 outcomes (PENDING → IN_PROGRESS → APPROVED/REJECTED/FAILED/MAX_ITERATIONS) persist to `evidence_packs`.
- [ ] Idempotency: re-invoking with same `(workflow_id, step_id, request_hash)` produces the same result.
- [ ] No regression: agents still emit OTel spans with `demo.*` attributes.

**Failure Protocol:**
If workflow starts but no activity executes: check the `temporal-worker` container logs; verify the worker is registered with the correct task queue. If signals don't resume the workflow: verify the signal name matches exactly between the API and the workflow's `@workflow.signal` handler. If Pattern A fails with `ImportError`: confirm `WORKFLOW_STREAMS_PATTERN=B` is set and the `redis` service is up.

**Companion Documents:**
- `Deliverable_4_Temporal_Operations_Guide.md` (both Pattern A and Pattern B documented).

---

### Phase 5: Frontend (Next.js + CopilotKit + Mic Recording)

**Entry Gate:**
- Phase 4 Exit Gate has passed.
- Backend API endpoints all reachable.

**Scope:**
- Initialize Next.js 14 App Router project per PCD §5.
- Package manager: pnpm (per REPO_STRATEGY).
- CopilotKit 1.50+ (verify against npm at install time; pin to latest stable 1.x or 2.x if 1.50 unavailable).
- **CopilotKit provider scoped to `/workspace/[wfId]/layout.tsx`**, NOT the root layout (per Deliverable_8 architecture).
- Landing page (`app/page.tsx`): persona selector → starts a new workflow session.
- Workspace page (`app/workspace/[wfId]/page.tsx`): 3-pane layout — chat (left), canvas/BRD preview (center), workflow status (right).
- Chat panel includes:
  - File upload (drag-drop): accepts `audio/mp3, audio/wav, audio/m4a, audio/webm`
  - **Microphone recording (D13):** `components/chat/AudioInput/useMicRecorder.ts` wraps `MediaRecorder` (no extra deps); records `audio/webm;codecs=opus`; shows a duration timer and preview before sending
  - Both paths upload via `POST /api/v1/uploads/audio` and use the returned `s3_uri` as workflow input
- AG-UI client (`@ag-ui/client` or CopilotKit's `useAgent` hook): connects to AgentCore session_id = workflow_id.
- Workflow Stream consumer: SSE EventSource from `/api/v1/workflows/{id}/stream`.
- HITL UI: clarification question card, approval card with three actions (Approve/Revise/Reject).
- All comments and identifiers in English (per PCD §4 rule 14).

**Explicitly NOT in Scope:**
- Metric dashboard (Phase 7).
- Server-side rendering of agent payloads (client-only for AG-UI events).
- Mobile responsive design (desktop-first).
- Authentication UI (mock persona selector only).

**Files Created/Modified:**
- `agentcore-demo-test1-frontend/package.json` (pnpm; Next.js 14, CopilotKit 1.50+, recharts will be added in Phase 7)
- `agentcore-demo-test1-frontend/app/layout.tsx` (root layout — NO CopilotKit here)
- `agentcore-demo-test1-frontend/app/page.tsx` (landing → persona select)
- `agentcore-demo-test1-frontend/app/workspace/[wfId]/layout.tsx` (CopilotKit provider scoped here)
- `agentcore-demo-test1-frontend/app/workspace/[wfId]/page.tsx`
- `agentcore-demo-test1-frontend/app/api/copilotkit/route.ts` (CopilotKit edge route)
- `agentcore-demo-test1-frontend/components/chat/ChatPanel.tsx`
- `agentcore-demo-test1-frontend/components/chat/AudioInput/AudioInput.tsx` (D13)
- `agentcore-demo-test1-frontend/components/chat/AudioInput/useMicRecorder.ts` (D13)
- `agentcore-demo-test1-frontend/components/canvas/BrdPreview.tsx`
- `agentcore-demo-test1-frontend/components/canvas/ReviewPanel.tsx`
- `agentcore-demo-test1-frontend/components/hitl/ClarificationCard.tsx`
- `agentcore-demo-test1-frontend/components/hitl/ApprovalCard.tsx`
- `agentcore-demo-test1-frontend/hooks/useAgent.ts`
- `agentcore-demo-test1-frontend/hooks/useWorkflowStream.ts`
- `agentcore-demo-test1-frontend/lib/api/workflows.ts`
- `agentcore-demo-test1-frontend/lib/api/uploads.ts`
- `agentcore-demo-test1-frontend/lib/auth/mockAuth.ts`
- `agentcore-demo-test1-frontend/Dockerfile`

**Dependencies:** Phase 4.

**Checkpoint Markers:**

| Checkpoint | Command to Run | Expected Output |
|---|---|---|
| CP-5.1 | `cd agentcore-demo-test1-frontend && pnpm install` | All packages installed |
| CP-5.2 | `pnpm build` | Build succeeds; `.next/` directory created |
| CP-5.3 | `curl -s http://localhost:3000` | Landing page HTML returned |
| CP-5.4 | Browser: navigate to `/`, select a persona | Redirected to `/workspace/<wfId>` |
| CP-5.5 | Browser: click the microphone button, allow permission, record 5s, stop | Preview shows duration; send button enabled |
| CP-5.6 | Browser: click send | Workflow starts; chat shows "Transcribing..." |
| CP-5.7 | Browser: upload `test.mp3` via drag-drop instead | Same: workflow starts |
| CP-5.8 | Browser: BRD preview renders Markdown draft as it streams | Canvas shows partial draft |
| CP-5.9 | Browser: HITL clarification question appears in chat | Clarification card visible |
| CP-5.10 | Browser: respond to clarification | Workflow resumes; draft regenerates |
| CP-5.11 | Browser: approval card with 3 actions appears | Card visible after Reviewer step |
| CP-5.12 | Browser: click Approve | Workflow terminates with outcome=APPROVED; Evidence Pack section visible |
| CP-5.13 | `grep -RnE "Türkçe|onayl[aı]" agentcore-demo-test1-frontend/ \| wc -l` | `0` (no Turkish text in code) |

**Exit Gate / Pass Criteria:**
- [ ] Next.js builds without errors.
- [ ] Frontend container starts on port 3000.
- [ ] Persona selector works; redirects to workspace with new `wfId`.
- [ ] Microphone recording works (`MediaRecorder` produces webm/opus, posted to `/api/v1/uploads/audio`).
- [ ] File upload works for mp3, wav, m4a, webm.
- [ ] CopilotKit provider scoped correctly (no provider at root layout).
- [ ] AG-UI events from AgentCore appear in chat in real time.
- [ ] Workflow Stream events appear in status pane in real time.
- [ ] HITL clarification card → user response → workflow resumes.
- [ ] Approval card with three actions (Approve / Revise / Reject) routes to correct signal.
- [ ] Evidence Pack viewer shows final outcome.
- [ ] All comments and identifiers in English.

**Failure Protocol:**
If `MediaRecorder` is not defined: check browser compatibility (Safari < 14.1 requires fallback). If CopilotKit features fail to render: verify provider scope is `/workspace/[wfId]/layout.tsx`, not root. If AG-UI events don't flow: check the agent ARN env var is correctly passed to the runtime URL builder.

**Companion Documents:**
- `Deliverable_5_CopilotKit_AGUI_Guide.md`
- `Deliverable_8_Frontend_Architecture_Guide.md`

---

### Phase 6: End-to-End Integration

**Entry Gate:**
- Phase 5 Exit Gate has passed.

**Scope:**
- 11 sequential E2E tests (T1–T11) per `Prompt_06_E2E_Integration.md`:
  - T1: Container health — all 6 (or 7) services Up
  - T2: Service endpoints — every health endpoint responds 200
  - T3: Full happy path (mic recording) — record → transcribe → draft → review → approve → Evidence Pack
  - T4: Full happy path (file upload) — same, via mp3 upload
  - T5: HITL clarification flow — drafter emits hitl_question, user responds, workflow resumes
  - T6: Self-correction cap — request 3 revisions, verify mandatory HITL escalation on the 4th
  - T7: Rejection — user rejects at approval; workflow terminates with REJECTED
  - T8: Revision cycle — user requests revise; workflow loops with new agent_run_id
  - T9: CloudWatch verification — query Logs Insights for `demo.workflow_id`; verify spans from all 3 agents
  - T10: Evidence Pack verification — fetch from API; verify hashes, version_sequence, all 8 blocks per agent
  - T11: Clean restart — `docker compose down && up`; verify all services healthy; run T3 again
- All tests run via `scripts/e2e_test.sh` and report pass/fail summary.

**Files Created/Modified:**
- `agentcore-demo-test1-backend/tests/e2e/test_brd_smoke.py` (pytest version)
- `agentcore-demo-test1-backend/tests/e2e/seed_test_audio.py` (creates a deterministic 3-second wav sample for upload tests)
- `agentcore-demo-test1-backend/tests/e2e/cw_logs_verify.py`
- `agentcore-demo-test1-backend/tests/e2e/evidence_pack_verify.py`
- `scripts/e2e_test.sh` (orchestrates the 11 tests; reports summary)
- `scripts/health_check_all.sh`

**Dependencies:** Phases 0–5.

**Checkpoint Markers:** (T1–T11 as above; full command list in `Prompt_06_E2E_Integration.md`)

**Exit Gate / Pass Criteria:**
- [ ] All 11 E2E tests pass.
- [ ] CloudWatch shows spans from all 3 agents with `demo.*` attributes, joined by `demo.workflow_id`.
- [ ] Evidence Pack JSON contains all 8 blocks per agent, with sha256 hashes for artifacts.
- [ ] Clean restart: `docker compose down && docker compose up -d` brings system back online; T3 still passes.
- [ ] Full demo run completes within 5 minutes for a 3-minute audio sample.

**Failure Protocol:**
If T9 (CloudWatch) fails: verify ADOT governance filter is not over-aggressive; check for typos in `demo.*` attribute names. If T11 (restart) fails: the most common cause is a docker-compose dependency cycle or a missing health check; use `depends_on` with `condition: service_healthy`.

**Companion Documents:**
- `PROMPT_TESTING_SECTIONS.md`

---

### Phase 7: Metric Dashboard

**Entry Gate:**
- Phase 6 Exit Gate has passed.
- At least 3 successful workflow runs in `evidence_packs` to populate the dashboard with non-trivial data.

**Scope:**
- Backend metric API per D14 / D11:
  - `GET /api/v1/metrics/overview` — total runs, success rate, avg cost, total time-saved hours
  - `GET /api/v1/metrics/by-template?workflow_template_id=&since=&until=` — p50/p95 latency, avg confidence, acceptance rate, total cost
  - `GET /api/v1/metrics/by-project?project_id=` — per-project aggregates
  - `GET /api/v1/metrics/cloudwatch?metric=&period=&start=&end=` — boto3 `get_metric_data` wrapper
  - `GET /api/v1/metrics/computed?since=&until=` — Guideline §9.6 formulas
- Frontend `/metrics` page (Recharts):
  - Summary cards: Total Runs, Success Rate, Total Cost (USD), Time Saved (h)
  - Line chart: cost over time
  - Line chart: p50/p95/p99 latency
  - Bar chart or table: per workflow_template breakdown
  - Project filter dropdown, date range picker (last 24h / 7d / 30d / custom)
- IAM: extend EC2 instance role policy with `cloudwatch:GetMetricData`.
- React Query default `staleTime: 60_000`; cap CloudWatch queries to last 30 days + period >= 300s for cost control.

**Files Created/Modified:**
- `agentcore-demo-test1-backend/app/api/metrics_routes.py`
- `agentcore-demo-test1-backend/app/api/schemas/metrics.py`
- `agentcore-demo-test1-backend/app/services/metrics_service.py`
- `agentcore-demo-test1-backend/app/services/cloudwatch_client.py`
- `agentcore-demo-test1-infra/iam-policies/ec2-instance-role.json` (add `cloudwatch:GetMetricData`)
- `agentcore-demo-test1-frontend/app/metrics/page.tsx`
- `agentcore-demo-test1-frontend/app/metrics/MetricsClient.tsx`
- `agentcore-demo-test1-frontend/components/metrics/SummaryCards.tsx`
- `agentcore-demo-test1-frontend/components/metrics/CostTimeSeries.tsx`
- `agentcore-demo-test1-frontend/components/metrics/LatencyChart.tsx`
- `agentcore-demo-test1-frontend/components/metrics/TemplateBreakdown.tsx`
- `agentcore-demo-test1-frontend/components/metrics/ProjectFilter.tsx`
- `agentcore-demo-test1-frontend/components/metrics/DateRangePicker.tsx`
- `agentcore-demo-test1-frontend/hooks/useMetrics.ts`
- `agentcore-demo-test1-frontend/lib/api/metrics.ts`
- `agentcore-demo-test1-frontend/package.json` (add `recharts@^2.12.0`)

**Dependencies:** Phase 6.

**Checkpoint Markers:**

| Checkpoint | Command to Run | Expected Output |
|---|---|---|
| CP-7.1 | `curl -s http://localhost:8000/api/v1/metrics/overview` | JSON with totals; status 200 |
| CP-7.2 | `curl -s "http://localhost:8000/api/v1/metrics/by-template?workflow_template_id=brd-from-audio-v1"` | Per-template aggregates |
| CP-7.3 | `curl -s "http://localhost:8000/api/v1/metrics/cloudwatch?metric=AgentLatencyMs&period=300&start=$(date -u -v-1d +%s)&end=$(date -u +%s)"` | Datapoints array |
| CP-7.4 | `curl -s http://localhost:8000/api/v1/metrics/computed` | JSON with `time_saved_hours`, `net_savings_usd`, `roi_pct`, `speed_multiplier`, `acceptance_rate`, `manual_equivalent_fte` |
| CP-7.5 | Browser: navigate to `http://localhost:3000/metrics` | Page renders; cards populated |
| CP-7.6 | Browser: switch date range to "Last 24h" | Charts re-fetch and update |
| CP-7.7 | Browser: filter by `project_id=proj-demo` | Aggregates scoped to that project |

**Exit Gate / Pass Criteria:**
- [ ] All 5 metric endpoints return 200 with the expected schema.
- [ ] CloudWatch queries succeed (IAM grants `cloudwatch:GetMetricData`).
- [ ] Frontend `/metrics` page renders cards, time series, and breakdown without errors.
- [ ] Date range and project filter both trigger data refresh.
- [ ] Computed metrics formulas match Guideline §9.6 implementations.
- [ ] No regression in `/workspace` or other routes.

**Failure Protocol:**
If CloudWatch queries fail with AccessDenied: re-deploy the IAM policy with the new `GetMetricData` action and wait for the EC2 instance role to refresh (or reboot the instance). If charts render empty: confirm at least 3 workflow runs exist in `evidence_packs`; the dashboard is data-driven.

**Companion Documents:**
- `Deliverable_7_CloudWatch_Telemetry_Guide.md` (dashboard query patterns)

---

## 4. Gate Pass Protocol

The following rules apply to **all phases** without exception:

### 4.1 Verification Checklist

1. Complete **ALL items** in the phase's Exit Gate Pass Criteria checklist.
2. Execute **ALL Checkpoint Marker commands** and verify expected output.
3. Check for **regressions**: re-run the previous phase's key checkpoints to ensure nothing broke.

### 4.2 Failure Rules

4. If **ANY** checklist item fails, **STOP** — do not proceed to the next phase.
5. Report the exact checkpoint ID, the command that failed, and the actual output received.
6. Fix the issue **within the same phase** before attempting to advance.
7. Gate passes are **binary**: ALL criteria pass, or the gate does **NOT** pass. There are no partial passes.

### 4.3 Retry Protocol

8. After fixing a failure, re-run **ALL** checkpoints for that phase (not just the failing one).
9. If the same checkpoint fails 3 times, escalate to the Failure Protocol in the phase detail section.
10. Document the fix applied in the phase notes for future reference.

### 4.4 Regression Prevention

11. Before declaring a phase complete, re-run at least 2 checkpoints from each preceding phase that intersects with the current phase's scope.
12. For Phase N > 0, always re-verify the agent invoke smoke test (from Phase 1).

---

## 5. Document Cross-Reference Map

| Deliverable | Title | Phases | Role in Build |
|---|---|---|---|
| Deliverable 0 | PCD | ALL | Single source of truth; D1–D14 decisions |
| Deliverable 1 | Infra & Cost Report | 0 | Cost estimates for us-east-1 and eu-central-1 |
| Deliverable 2 | Reference Strands/LangGraph code | 1, 2 | Skeleton agent code patterns |
| Deliverable 3 | Historical Implementation Prompts | — | Superseded by Prompts 00–07 |
| Deliverable 4 | Temporal Operations Guide | 4 | Workflow, activities, Pattern A/B Streams |
| Deliverable 5 | CopilotKit AG-UI Guide | 5 | CopilotKit hooks, AG-UI HttpAgent, generative UI |
| Deliverable 6 | AWS Operations Guide | 0 | Step-by-step AWS setup, IAM, teardown |
| Deliverable 7 | CloudWatch Telemetry Guide | 2, 7 | OTel + ADOT + Logs Insights + metric dashboard queries |
| Deliverable 8 | Frontend Architecture Guide | 5 | Dual-realm, provider scope, testing |
| Deliverable 9 | Multi-Framework AG-UI Guide | — | Future reference |
| `PAYLOAD_SCHEMA.md` | 8-block canonical reference | 2 | Validation rules and per-agent examples |
| `APPLY_PLAN.md` | Documentation cleanup plan | — | Meta: how this doc and others were aligned |

### Prompt-to-Phase Mapping

| Prompt File | Target Phase |
|---|---|
| `Prompt_00_Infra_Bootstrap.md` | Phase 0 |
| `Prompt_01_AgentCore_Deployment.md` | Phase 1 |
| `Prompt_02_Agent_Telemetry.md` | Phase 2 |
| `Prompt_03_Backend_Foundation.md` | Phase 3 |
| `Prompt_04_Temporal_Workflow.md` | Phase 4 |
| `Prompt_05_Frontend.md` | Phase 5 |
| `Prompt_06_E2E_Integration.md` | Phase 6 |
| `Prompt_07_Metrics_Dashboard.md` | Phase 7 |

---

## 6. Checkpoint Design Philosophy

This section explains the checkpoint pattern used throughout this plan. Checkpoints are designed for AI coding assistants that benefit from explicit verification steps.

### 6.1 What is a Checkpoint?

A checkpoint is a **mandatory STOP point** within a phase where the implementer runs a verification command before proceeding. Checkpoints serve three purposes:

1. **Early failure detection**: catch errors before they cascade.
2. **State validation**: confirm current state matches expectations.
3. **Progress tracking**: provide explicit evidence a sub-task is complete.

### 6.2 Checkpoint Structure

| Element | Description |
|---|---|
| **Checkpoint ID** | Unique identifier (e.g., `CP-3.5`) for reporting failures |
| **Command to Run** | Exact shell command or API call to execute |
| **Expected Output** | Precise expected result |

### 6.3 Checkpoint Rules for Implementers

1. **Run the command exactly as written** — do not substitute or simplify.
2. **Compare output to expected output literally** — partial matches are NOT passes.
3. **If output does not match: STOP** — do not proceed to the next checkpoint.
4. **Report the exact mismatch** — include both expected and actual output.
5. **Fix the issue, then re-run ALL checkpoints from the start of the phase**.

### 6.4 Checkpoint vs Exit Gate

| | Checkpoint | Exit Gate |
|---|---|---|
| **Frequency** | Multiple per phase (5–15) | One per phase |
| **Scope** | Sub-task verification | Full phase verification |
| **When run** | During phase execution | At phase end |
| **Failure action** | Stop, fix, retry checkpoint | Stop, do not advance to next phase |
| **Gate pass?** | No — intermediate | Yes — binary pass/fail |

---

## 7. Key Decisions

All architectural decisions are defined in **PCD §3 (D1–D14)** and are not renegotiable during execution. The PCD covers:

- **AWS region**: us-east-1 primary
- **Compute**: EC2 t3.large single host
- **Temporal**: docker-compose self-hosted, Server `1.22`, Web UI on port 8081
- **RDS**: db.t4g.micro, 3 databases on a 2-AZ subnet group
- **Auth**: Mock persona JSON (D9; demo only)
- **Agent runtime**: AWS Bedrock AgentCore via S3 ZIP (no Docker, no ECR for agents)
- **Agents**: transcriber (Strands) + drafter (Strands) + reviewer (LangGraph 4-node)
- **Telemetry**: OpenTelemetry + ADOT Collector; `demo.*` attribute prefix (D1)
- **Metric namespace**: `DemoSDLC/Agent`
- **8-block payload names**: Status / Resources / Timing / Financial / Artifacts / Quality / Tool Calls / Risk (D7)
- **A2A**: Temporal signals (never direct agent-to-agent HTTP)
- **HITL**: First-class step model; self-correction cap = 3 total attempts (D8)
- **Claim-Check**: > 1 MB threshold; AgentCore Persistent FS via POSIX
- **Workflow Streams**: Pattern A preferred, Pattern B fallback (D6)
- **Audio input**: file upload + browser mic recording (D13)
- **Metrics Dashboard**: `/metrics` page + 5 backend endpoints (D14)

Refer to PCD §3 for the full table.

---

## 8. Appendix: Quick Reference Card

### Phase Order (Never Skip)

```
Phase 0  →  Phase 1   →  Phase 2     →  Phase 3   →  Phase 4    →  Phase 5    →  Phase 6  →  Phase 7
(Infra)     (Agents)     (Telemetry)    (Backend)    (Temporal)    (Frontend)    (E2E)       (Metrics)
```

### Critical Ports

| Service | Port | Phase Introduced |
|---|---|---|
| Next.js Frontend | 3000 | Phase 5 |
| FastAPI Backend | 8000 | Phase 3 |
| Temporal gRPC | 7233 | Phase 3 (server starts in Phase 3 docker-compose) |
| Temporal Web UI | 8081 | Phase 3 |
| ADOT Collector OTLP gRPC | 4317 | Phase 3 (config from Phase 2) |
| AgentCore Runtime | 443 (HTTPS, AWS-managed) | Phase 1 |
| PostgreSQL (RDS) | 5432 | Phase 0 |
| Redis (Pattern B only) | 6379 | Phase 4 (conditional) |

### Verification One-Liners

```bash
# Phase 0: Infra health
aws ec2 describe-instances --filters "Name=tag:Project,Values=agentcore-demo-test1" --query 'Reservations[0].Instances[0].State.Name' --output text
aws rds describe-db-instances --query 'DBInstances[0].DBInstanceStatus' --output text

# Phase 1: Agent health (via boto3 wrapper)
python scripts/smoke_invoke.py --agent agent_1_transcriber
python scripts/smoke_invoke.py --agent agent_2_drafter
python scripts/smoke_invoke.py --agent agent_3_reviewer

# Phase 2: Telemetry
aws logs start-query --log-group-name /agentcore-demo-test1/agent-logs --query-string 'fields @message, demo.agent_id, demo.workflow_id | filter demo.agent_id = "agent_1_transcriber" | limit 5'

# Phase 3: Backend
curl -s http://localhost:8000/health && curl -s http://localhost:8000/health/db

# Phase 4: Temporal + workflow
curl -s http://localhost:8081           # Web UI
curl -X POST http://localhost:8000/api/v1/workflows -d '{"audio_s3_uri":"s3://...","persona_id":"ba-sap-mm"}'

# Phase 5: Frontend
curl -s http://localhost:3000

# Phase 6: Full stack E2E
./scripts/e2e_test.sh

# Phase 7: Metrics
curl -s http://localhost:8000/api/v1/metrics/overview
```

---

*End of Master Phase Plan v2.0. Execute phases sequentially. Verify every gate. Report failures immediately.*
