# Apply Plan: Documentation Alignment with AI SDLC Agent Development Guidelines v5

**Author:** Claude (Opus 4.7) — under user direction
**Date:** 2026-05-15
**Status:** Awaiting user approval before execution

This document is the single source of truth for the documentation cleanup work. It lists 12 sequential change packages (P1–P12). Each package is independent enough to land on its own commit, but the order respects dependencies.

---

## 0. Project-Wide Decisions (FROZEN — do not relitigate inside packages)

| # | Decision | Source |
|---|---|---|
| D1 | **Span attribute prefix = `demo.*`** (not `jnj.*`). ADOT collector configuration uses `demo.*` everywhere. Production guideline notes that `jnj.*` is the JnJ enterprise namespace; the demo intentionally diverges with a clear comment. | User decision 2026-05-15 |
| D2 | **Orchestration = `audio → BRD`**. Reference orchestration template name: `brd-from-audio-v1`. Agents: `agent-1-transcriber`, `agent-2-drafter`, `agent-3-reviewer`. | User decision 2026-05-15 |
| D3 | **Cedar policies = doc-only**. PCD explains the GitOps pattern and includes an illustrative `.cedar` file; no runtime enforcement in the demo (Gateway is not deployed). Marked "Production-only" with rationale. | User decision 2026-05-15 |
| D4 | **Agent naming = transcriber / drafter / reviewer**, directories `agents/agent_1_transcriber/`, `agents/agent_2_drafter/`, `agents/agent_3_reviewer/`, entry file `main.py`. Hyphen-case for `agent_id` strings, snake_case for filesystem paths. | User decision 2026-05-15 |
| D5 | **Local dev path = `/Users/ugurgocen/projects/agentcore-demo-test1/`** for every prompt and guide. Prompt_06 (E2E) calls out the EC2 path separately when SSH'd in (`/home/ec2-user/agentcore-demo-test1/`). | User decision 2026-05-15 |
| D6 | **Temporal Workflow Streams kept**. Demo will exercise streams to push step-level updates to the SSE bridge. Since the Python SDK's `temporalio.contrib.workflow_streams` is not yet a stable public API, the implementation falls back to: **Temporal Update API + side-channel pub/sub (Redis or in-process broker) → FastAPI SSE → AG-UI** if the streams import is unavailable. The PCD documents both shapes; Prompt_04 picks the available one at runtime with a feature flag. | User decision 2026-05-15 + technical fallback |
| D7 | **8-block payload names = Status / Resources / Timing / Financial / Artifacts / Quality / Tool Calls / Risk**. The alternate set in PHASE_PLAN.md (`context_block`, `input_block`, ...) is wrong and gets removed everywhere. | Guideline §8 |
| D8 | **Self-correction cap = 2 attempts after initial generation (3 total)** before mandatory HITL escalation. Tracked in Block 6 `total_attempts`. Each attempt is a new `step_id` with `step_type=revise`; `agent_run_id` stays the same when the agent invocation is logically continuous, otherwise the orchestrator allocates a fresh one. | Guideline §16.5 |
| D9 | **Mock auth = demo-only simplification**. Every doc that mentions auth explicitly marks "DEMO ONLY; production uses JnJ Entra ID + PingID per Guideline §15.1.1". No `useAuth` hooks pretending to be real OAuth. | Guideline §15.1 |
| D10 | **`evidence_packs` canonical schema** = the UUID/JSONB shape from Prompt_04 (rounds_executed, agent_outputs, hitl_exchanges, etc.). Phase 3's alembic migration is updated to create this schema directly; Phase 4 no longer re-creates the table. | Conflict reconciliation |
| D11 | **API path canonical = `/api/v1/workflows/...`** with subroutes: `POST /api/v1/workflows`, `GET /api/v1/workflows/{id}`, `GET /api/v1/workflows/{id}/stream` (SSE), `POST /api/v1/workflows/{id}/signal/clarification`, `POST /api/v1/workflows/{id}/signal/approval`. Plus metrics namespace: `GET /api/v1/metrics/overview`, `/api/v1/metrics/by-template`, `/api/v1/metrics/by-project`, `/api/v1/metrics/cloudwatch`, `/api/v1/metrics/computed`. Every prompt aligns to these. | Conflict reconciliation |
| D12 | **DB credentials canonical**: database `agentcore_demo_test1`, app user `app_user`, temporal user `temporal_user`. Passwords stored in `.env` (gitignored). `temporal_user` password is generated in Prompt_00 and exported to `.vpc_env` for Prompt_03 to consume. | Conflict reconciliation |
| D13 | **Audio input = MediaRecorder (webm/opus) + file upload, user picks**. Browser-native `MediaRecorder` API (no extra deps). Both webm/opus and mp3/wav/m4a accepted. AWS Transcribe supports webm/opus directly. Chat input shows two buttons: 🎤 Record and 📎 Upload, plus a unified preview area before send. | User decision 2026-05-15 |
| D14 | **Metric Dashboard page exists at `/metrics`** — Next.js page rendered with `recharts`. Pulls data from FastAPI `/api/v1/metrics/*` endpoints, which aggregate from RDS (per-run records) and pull live CloudWatch metrics via boto3 `get_metric_data`. Computed metrics follow Guideline §9.6 formulas (time_saved, net_savings, roi_pct, speed_multiplier, acceptance_rate, manual_equivalent_fte). Drill-down: workflow_template → individual workflow run → step chain. | User decision 2026-05-15 |

---

## 1. Package Map (read-only — execution starts at P1)

| Package | Title | Files Touched | Depends On |
|---|---|---|---|
| **P1** | Rewrite Deliverable_0 (PCD) as the single source of truth | Deliverable_0_PROJECT_CONTEXT.md | — |
| **P2** | Strip stale content from PHASE_PLAN | PHASE_PLAN.md | P1 |
| **P3** | Create canonical PAYLOAD_SCHEMA + naming conventions | PAYLOAD_SCHEMA.md (new), DELIVERABLE_MAPPING.md | P1 |
| **P4** | Fix Prompt_00 (infrastructure) | Prompt_00_Infra_Bootstrap.md, Deliverable_6_AWS_Operations_Guide.md | P1, P3 |
| **P5** | Fix Prompt_01 (agent skeletons) | Prompt_01_AgentCore_Deployment.md, Deliverable_2_Reference_Strands_Agent_Code.md | P1, P3, P4 |
| **P6** | Complete Prompt_02 (telemetry, no stubs) | Prompt_02_Agent_Telemetry.md, Deliverable_7_CloudWatch_Telemetry_Guide.md | P1, P3, P5 |
| **P7** | Fix Prompt_03 (backend foundation) | Prompt_03_Backend_Foundation.md | P1, P3, P4 |
| **P8** | Fix Prompt_04 (Temporal workflow + Streams fallback) | Prompt_04_Temporal_Workflow.md, Deliverable_4_Temporal_Operations_Guide.md | P1, P3, P7 |
| **P9** | Fix Prompt_05 (frontend, all-English) | Prompt_05_Frontend.md, Deliverable_5_CopilotKit_AGUI_Guide.md, Deliverable_8_Frontend_Architecture_Guide.md | P1, P3, P7 |
| **P10** | Rewrite Prompt_06 (E2E integration) | Prompt_06_E2E_Integration.md, PROMPT_TESTING_SECTIONS.md | P1, P3, P4–P9 |
| **P11** | Reconcile side guides | DEVELOPER_ONBOARDING.md, REPO_STRATEGY.md, GIT_WORKFLOW.md, DEMO_GELISTIRME_KILAVUZU.md, V5_IMPLEMENTATION_PLAN.md, V5_TELEMETRY_INTEGRATION_ANALYSIS.md | P1–P10 |
| **P12** | Metric Dashboard (new page + backend metrics API) | Prompt_07_Metrics_Dashboard.md (new), Prompt_03_Backend_Foundation.md (additions), Prompt_05_Frontend.md (additions), Deliverable_7_CloudWatch_Telemetry_Guide.md (additions) | P1–P9 |
| **P13** | Final audit pass — update FILE_STATUS_REPORT, CONSISTENCY_AUDIT_REPORT, write a fresh "Round 3" report | FILE_STATUS_REPORT.md, CONSISTENCY_AUDIT_REPORT_ROUND3.md (new), DOCUMENT_AUDIT_REPORT.md | All |

---

## 2. Per-Package Plans

### P1 — Rewrite Deliverable_0 (Project Context Document)

**Goal:** Make PCD the single authoritative reference. Every other prompt/guide references PCD; PCD references the Guideline.

**Acceptance criteria (post-package):**
- PCD documents the 8-block payload with kanonik names from Guideline §8
- Identifier hierarchy §3.4 reproduced verbatim including `workflow_template_id`, `requested_by`, `execution_context`, `task`, `inputs`, `execution_options`
- Five required endpoints listed (/invoke, /health, /metadata, /capabilities, /metrics)
- Status / Error code dictionary from Guideline Appendix B included as PCD appendix
- D1–D12 frozen decisions enumerated in a "Decisions" section
- A "Demo simplifications vs production" section explicitly lists: mock auth, no Gateway, no Token Vault, local docker-compose vs platform EKS/AgentCore, `demo.*` vs `jnj.*` prefix
- Companion documents list trimmed: only Prompts 00-06 + Deliverables 0,2,4,5,6,7,8 + side guides
- Repo structure section uses the canonical names from D4
- Container topology corrected to 6 services (text and list agree)
- Typos: "Strains Agents", "prespective-guidance" fixed
- Temporal import correction: `from temporalio import activity` (not `from temporalio.activity import activity`)
- Section 11.6 contradiction fixed (S3 ZIP, not container-on-EC2)

**Execution steps:**
1. Read current Deliverable_0 in full
2. Build a section-by-section diff plan as inline comments
3. Apply edits via Edit tool (not full rewrite — preserve sections that are correct)
4. Add new appendices: Error Codes, Status Codes, Demo vs Production matrix
5. Verify cross-references (every file mentioned exists in the resources dir)

---

### P2 — Strip stale content from PHASE_PLAN.md

**Goal:** PHASE_PLAN becomes a clean phase/gate roadmap with no contradictions.

**Acceptance criteria:**
- Date "2026-03-XX" replaced with "2026-05-15"
- All typos fixed: "moand on" → "move on"; "serand" → "serve"; "Approand" → "Approve"; "responsiand" → "response"; "natiand" → "native"
- Phase 1 Files Created/Modified: remove Dockerfile per agent, remove `docker-compose.agents.yml`, remove `scripts/build-push-agents.sh`; replace with S3 ZIP packaging notes
- Phase 1 Failure Protocol: remove "docker logs <container>"; replace with AgentCore Runtime CloudWatch log group queries
- Phase 2 Scope: all `cloudwatch_emf` references → `otel_setup` (agent-side) + ADOT config (infra-side)
- Phase 2 checkpoints: EMF query patterns → OTel/CloudWatch Insights patterns (use Guideline §9.5 example)
- Phase 2 Exit Gate: "EMF logs visible" → "OTel spans landed in CloudWatch via ADOT"
- Phase 5: "Next.js 15" → "Next.js 14+" (align with ARCHITECTURE_DATAFLOW_GUIDE)
- Section 7 Key Decisions: drop "ag_ui_strands" (rename to `ag_ui` or just AG-UI protocol)
- Section 7 Key Decisions: CopilotKit version pinned to `^1.50.0` consistent with PCD
- Section 7 AgentCore Deployment: English-only, complete sentence on line 786
- Appendix Section 8: translate Turkish fragments to English
- Phase 0 Companion Documents: complete the truncated sentence
- 8-block payload names anywhere it appears → kanonik names from Guideline §8
- Reference to PCD's "Decisions" section instead of restating decisions

**Execution steps:**
1. Read PHASE_PLAN.md fully
2. Spawn a single Edit pass per section
3. After all edits, do a grep for "context_block", "ag_ui_strands", "EMF", "Approand" to confirm no residue
4. Cross-link to PCD's Decisions and PAYLOAD_SCHEMA.md (P3)

---

### P3 — Create canonical PAYLOAD_SCHEMA.md

**Goal:** Single reference for the 8-block payload that every prompt, every test, and every code file points to.

**Acceptance criteria:**
- New file `agentcore-demo-test1-docs/resources/PAYLOAD_SCHEMA.md` exists
- Contains: request schema (Guideline §7.3), response schema (Guideline §8), identifier hierarchy, error codes, status codes
- Worked example for each agent (transcriber, drafter, reviewer) with cost_reporting_model = `token_based` (drafter, reviewer) or `vendor_cost` (transcriber for AWS Transcribe)
- Validation pseudo-code (`validate_payload(payload, tier)`) included
- Naming convention table: directories, files, activity names, DB names, API paths, env vars — all from D11/D12
- Cross-reference table: which prompt produces which payload field
- DELIVERABLE_MAPPING.md updated to point to PAYLOAD_SCHEMA instead of restating block names

**Execution steps:**
1. Author PAYLOAD_SCHEMA.md from scratch based on Guideline §7.3 + §8 + Appendix B
2. Add demo-specific examples (audio→BRD orchestration)
3. Update DELIVERABLE_MAPPING.md's "Quick Reference" section to link

---

### P4 — Fix Prompt_00 (Infrastructure Bootstrap)

**Goal:** Prompt_00 executes end-to-end on a fresh AWS account without manual fixes.

**Acceptance criteria:**
- S3 bucket creation for us-east-1: drop `LocationConstraint` when region is us-east-1 (AWS quirk)
- IAM ARN format: `arn:aws:iam::ACCOUNT_ID:role/NAME` (single colon after `iam:`)
- Bucket naming canonicalized: `${PROJECT}-audio-uploads-${ACCOUNT_ID}`, `${PROJECT}-artifacts-${ACCOUNT_ID}`, `${PROJECT}-claimcheck-${ACCOUNT_ID}`, `${PROJECT}-code-${ACCOUNT_ID}`
- Transcribe IAM policy ARN pattern matches bucket names
- `SUBNET1` and `SUBNET2` (2 AZs) exported to `.vpc_env`; EC2 launch sources it
- RDS subnet group uses both AZs (us-east-1a + us-east-1b)
- Step numbering monotonic (no skipping)
- CHECKPOINT 7B (ECR check) removed; replaced with S3 ZIP staging check (`aws s3 ls s3://${CODE_BUCKET}/agents/`)
- Gate count says "10 GATES" not "9 GATES"
- Cleanup script renumbered to consecutive steps
- `s3files` client usage removed — replace with `aws s3api` calls; persistent filesystem creation moved to Prompt_01 (where the runtime exists)
- `temporal_user` password generated, captured into `.vpc_env`
- Deliverable_6 updated in parallel: same S3 bucket creation, same subnet AZ fix, Turkish IMDS checklist translated to English, teardown policies fixed (custom policy detached, not `AmazonS3FullAccess`)
- AMI ID looked up dynamically (`aws ssm get-parameters --names /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-6.1-x86_64`)
- macOS date workaround documented (`date -u -v-1d` on macOS, `date -u -d '1 day ago'` on Linux)

**Execution steps:**
1. Read Prompt_00 + Deliverable_6 in full
2. Build a patch list per script
3. Apply edits
4. Add a "Verification" sub-step at the end: `bash` snippet that verifies every Gate's outputs
5. Verify variable names match between Prompt_00 and what Prompt_01 expects

---

### P5 — Fix Prompt_01 (Agent Skeleton + Deployment)

**Goal:** Three agents deploy to AgentCore from S3 ZIP, return placeholder 8-block payloads, pass smoke test.

**Acceptance criteria:**
- Three agent directories created: `agents/agent_1_transcriber/`, `agents/agent_2_drafter/`, `agents/agent_3_reviewer/`
- Each agent entry file: `main.py` (uniform across all prompts)
- Each agent implements ALL FIVE endpoints: `/invoke`, `/health`, `/metadata`, `/capabilities`, `/metrics`
- Initial `/invoke` returns a valid 8-block payload (with `Unavailable` for fields the placeholder can't fill)
- Persistent Filesystem setup deferred from Prompt_00, now in Step 0 of Prompt_01 (`FS_ARN` and `AP_ARN` created here, exported to `.vpc_env`)
- Git remote uses `ugocen` (resolved placeholder)
- Phase 1 `requirements.txt` minimal: `bedrock-agentcore`, `boto3`, `pydantic`, `python-json-logger` only (no `temporalio`, no `opentelemetry-*` — those land in Phase 2/3)
- Agent code imports validated: `from bedrock_agentcore.runtime import BedrockAgentCoreApp`
- Deploy commands use uniform agent ARN pattern: `arn:aws:bedrock-agentcore:us-east-1:${ACCOUNT_ID}:runtime/agent-{1,2,3}-{transcriber,drafter,reviewer}`
- Smoke test: boto3 client invocation script returns 8-block payload from each agent
- Deliverable_2 updated: directory and file names canonicalized; Transcribe `DataAccessRoleArn` moved to top-level (not inside `Settings`); `LanguageOptions` added; format detection from URI; bucket names from env not hardcoded; HITL `request_clarification` actually wired into Agent 2; LangGraph nodes use idiomatic partial state returns
- Token estimation function uses `tiktoken` or equivalent, not `len(draft.split())`

**Execution steps:**
1. Read Prompt_01, Deliverable_2 in full
2. Write the three canonical `main.py` files (placeholder logic, full endpoint surface, valid 8-block return)
3. Update Prompt_01 to instruct creation of these files
4. Add Persistent FS setup step at the beginning
5. Patch Deliverable_2 to match the canonical agent contracts

---

### P6 — Complete Prompt_02 (Telemetry)

**Goal:** Steps 5-7 contain real code (not stubs). ADOT YAML parses. `demo.*` attribute boilerplate applied to every agent.

**Acceptance criteria:**
- Steps 5, 6, 7 each contain the full agent file rewrite with `demo.*` OTel boilerplate from Guideline §9.5 (adapted from `jnj.*` to `demo.*`)
- Shared `agents/common/__init__.py` created
- `agents/common/payload_builder.py` complete: all 8 blocks, validate function, identifier echo enforcement
- `agents/common/otel_setup.py` complete: tracer factory, `demo.*` attribute helpers, idempotent setup (call multiple times OK)
- ADOT YAML correct (real newlines, no `\n` literals), `governance` filter processor implemented (drops spans missing `demo.workflow_id` or applies default), `governance` processor wired into the traces pipeline
- OTel import path validated: `from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter` (gRPC, port 4317)
- `DEPLOYMENT_ENVIRONMENT_NAME` constant used (post 1.20 API)
- Turkish "Faz 1" → "Phase 1"
- Agent file names = `main.py` (not `agent.py`)
- Each agent's `main.py` modification preserves Phase 1 endpoint surface; only adds telemetry wrappers
- Verification: a Python script that runs each agent's `/invoke` and checks that a span with the required `demo.*` attributes was emitted (via a stub OTLP receiver)
- Deliverable_7 updated to match: `demo.*` everywhere, log group names canonicalized to match Deliverable_6, IAM namespace matches ADOT namespace (`DemoSDLC/Agent`), governance filter wired into pipeline, alarm descriptions match thresholds, macOS date workaround

**Execution steps:**
1. Read Prompt_02 + Deliverable_7
2. Author the complete payload_builder, otel_setup
3. Author the three agent rewrites (each ~100 lines: imports + tracer + endpoints with span wrappers + 8-block builder)
4. Validate ADOT YAML with `yq` (or equivalent)
5. Reconcile log group + metric namespace between D6 and D7

---

### P7 — Fix Prompt_03 (Backend Foundation)

**Goal:** FastAPI backend starts, Alembic migrations run, docker-compose up succeeds.

**Acceptance criteria:**
- Working directory uniform: all paths begin with `/Users/ugurgocen/projects/agentcore-demo-test1/agentcore-demo-test1-backend/`
- Path inside container: `/app/...`
- `evidence_packs` table created with the canonical schema (D10) — single Alembic migration
- DB credentials come from `.env` which sources `.vpc_env` (canonical D12)
- `temporal-server` docker-compose service uses `POSTGRES_USER=temporal_user`, `POSTGRES_PWD=${TEMPORAL_PASSWORD}` (separately tracked)
- SQLAlchemy 2.0+ imports: `from sqlalchemy.orm import DeclarativeBase` (not deprecated `declarative_base`)
- docker-compose `version: "3.8"` removed
- API paths canonical (D11)
- Settings module includes all variables needed downstream (Workflow Streams broker URL, S3 bucket env vars, AgentCore ARNs from Phase 1)

**Execution steps:**
1. Read Prompt_03
2. Patch working directories
3. Patch DB schema migration to canonical
4. Patch SQLAlchemy + docker-compose
5. Patch API path constants
6. Add `.env` template that mirrors `.vpc_env`

---

### P8 — Fix Prompt_04 (Temporal Workflow)

**Goal:** Workflow runs end-to-end, calls all 3 agents via boto3, streams updates back to the SSE bridge, persists evidence pack.

**Acceptance criteria:**
- Corrupted code (lines ~1139-1154 in current file) replaced with the canonical activity implementation
- `evidence_packs` table NOT re-created; Phase 4 just inserts/updates rows
- DB credentials use canonical `.env` (D12)
- Agent role naming uses transcriber/drafter/reviewer (not analyst/architect/reviewer)
- Workflow Streams implementation: try `temporalio.contrib.workflow_streams`; on ImportError, fall back to **Workflow Update API + Redis pub/sub side channel** documented as Pattern B. The PCD links to both patterns. Demo defaults to Pattern A if SDK supports it, Pattern B otherwise.
- API path canonical (D11)
- Agents are invoked via boto3 with full 8-block request payload (D11 + D12 + Guideline §7.3)
- Each agent invocation is wrapped in a retry policy; idempotency key = `f"{workflow_id}:{step_id}:{request_hash}"`
- HITL pattern: workflow waits on signal `clarification` and `approval`; on signal, re-invokes the same agent with `human_input` populated
- Self-correction cap (D8) enforced inside `invoke_agent_activity`: counts retries with same `step_id`, escalates to HITL after 3 total
- Workflow outcomes canonical: `APPROVED`, `REJECTED`, `TERMINATED`, `FAILED`, `MAX_ITERATIONS` (covers both old enums)
- Deliverable_4 updated: Temporal Server image pinned to canonical version (use `1.22` since Phase 0 created the DB schema; or bump both to a current stable version like `1.24`; need to decide → recommendation: stay on `1.22` for predictable schema)
- `temporal_visibility` database addition documented (it's required by `temporalio/auto-setup` image; add it to Phase 0 too — feeds back into P4 as a follow-up)
- ContinueAsNew pattern: check both runtime duration AND event count
- Workflow status enum aligned to `BRDState` (5 values: PENDING, IN_PROGRESS, AWAITING_HUMAN, APPROVED, REJECTED, FAILED)

**Execution steps:**
1. Read Prompt_04 + Deliverable_4
2. Author the canonical `activities.py`, `workflows.py`, `worker.py`
3. Document the Streams Pattern A (preferred) vs Pattern B (fallback) decision tree in PCD
4. Patch Prompt_04 to instruct creation of these files
5. Add the `temporal_visibility` DB to Prompt_00 if not present (small backport)
6. Verify env var names match Prompt_03's settings.py

---

### P9 — Fix Prompt_05 (Frontend)

**Goal:** Next.js frontend builds, connects to FastAPI, renders chat + canvas, all-English.

**Acceptance criteria:**
- All Turkish text translated to English (lines 167, 169, 1339, 1349, 1439, etc.)
- File creation order respects dependencies (use-shared-state.ts created BEFORE the checkpoint that verifies it)
- File count claim ("There should be 5 files") matches actual count
- Typo `resoland` → `resolve`
- Package name `agentcore-demo-test1-frontend` (not `legal-agent-frontend`)
- CopilotKit agent name `audio-to-brd-agent` (not `legal-research-agent`)
- `GenerativeUIRenderer.tsx` either created or removed from the directory listing (it's referenced but never written)
- CopilotKit `ExperimentalEmptyAdapter` replaced with the real `OpenAIAdapter` or `BedrockAdapter` (whichever CopilotKit 1.50 supports); if no real adapter is available, use the `CopilotRuntime`'s direct AG-UI bridge
- CopilotKit version verified — if `^1.50.0` doesn't exist, pin to the latest stable as of 2026-05 (likely `^1.5.x` or `^2.0.x`)
- API path canonical (D11)
- CopilotKit provider scoped to `/workspace/[wfId]/layout.tsx` (Deliverable_8's correct pattern), NOT root
- Deliverable_5 updated: HttpAgent code fragments restored (currently truncated lines 542, 558, 571, 636-641, 727-733, 749, 803, 815); auth references replaced with mock-auth pattern; `renderAndWaitForResponse` usage standardized
- Deliverable_8 updated: ESLint config fixed (use `import/no-restricted-paths` for `zones`, separate rule from `no-restricted-imports`); MSW v2 API (`http`, `HttpResponse`); MockEventSource test fixed (instances array maintained); CanvasProvider test has real assertions; remove the contradiction about `useQuery` in workspace pages or revise the rule
- **Mic recording (D13):** New `components/AudioInput/`:
  - `useMicRecorder.ts` hook: wraps `MediaRecorder`, returns `{isRecording, start, stop, blob, duration, error}`; requests `getUserMedia({audio: true})`; records `audio/webm;codecs=opus`
  - `AudioInput.tsx` component: two buttons (🎤 Record / 📎 Upload), waveform preview, duration timer, send/cancel actions; accepts file types `audio/mp3,audio/mpeg,audio/wav,audio/webm,audio/m4a`
  - Integration into chat input bar (existing `ChatPanel.tsx`)
  - Upload path: browser → `POST /api/v1/uploads/audio` (FastAPI multipart endpoint) → S3 `${PROJECT}-audio-uploads-${ACCOUNT_ID}` → returns `s3://...` URI used as workflow input
  - Backend endpoint: streamed multipart upload, max 50 MB, content-type validation, S3 putobject with `Metadata={user_id, recorded_at, content_type}`
  - Permissions: explicit "Microphone permission required" toast if `getUserMedia` rejects
  - Unit test: `MockMediaRecorder` exercises start→stop→blob path
  - Browser support note: Chrome/Edge/Firefox; Safari 14.1+

**Execution steps:**
1. Read Prompt_05, Deliverable_5, Deliverable_8
2. Translate Turkish → English in Prompt_05
3. Restore HttpAgent code in Deliverable_5
4. Fix lint/test patterns in Deliverable_8
5. Validate CopilotKit version exists (web fetch if needed)
6. Update CopilotKit provider scope decisions consistently
7. Author the mic recording subsystem (hook + component + backend upload endpoint)

---

### P10 — Rewrite Prompt_06 (E2E Integration)

**Goal:** E2E tests pass against the system that Prompts 00-05 actually build.

**Acceptance criteria:**
- Working directory: `/Users/ugurgocen/projects/agentcore-demo-test1/agentcore-demo-test1-system/` REMOVED — use `/Users/ugurgocen/projects/agentcore-demo-test1/` (or EC2 path when SSH'd in)
- Phase descriptions match actual phases (no ECS/ALB references)
- API paths canonical (D11)
- Evidence Pack statuses match BRDState enum (D8)
- 8-block payload assertions check canonical block names (D7)
- Service list matches docker-compose: fastapi, temporal-server, temporal-ui, temporal-worker, nextjs-frontend, adot-collector (6 services)
- `seed_test_audio.py` created BEFORE its existence is checked
- CloudWatch verification queries use `demo.*` attribute filters
- Test orchestrator script runs all 11 tests (T1–T11) and produces a pass/fail summary
- PROMPT_TESTING_SECTIONS.md updated: include actual test files, not just structure

**Execution steps:**
1. Read Prompt_06 + PROMPT_TESTING_SECTIONS
2. Major rewrite — most of Prompt_06 is built against the wrong architecture
3. Re-author against the actual system from Prompts 00-05
4. Add the missing test fixture creation steps

---

### P11 — Reconcile side guides

**Goal:** Side guides (onboarding, repo strategy, git workflow, demo guide) align with the canonical decisions.

**Acceptance criteria:**
- DEVELOPER_ONBOARDING.md Step 5: replace monorepo layout with 4-repo polyrepo (matches REPO_STRATEGY)
- DEVELOPER_ONBOARDING.md Step 2.5: Node.js 20+ (not 18+)
- DEVELOPER_ONBOARDING.md daily workflow: `dev` branch usage documented
- REPO_STRATEGY.md Section 10.1: "Push to ECR" comment removed (no ECR in this project)
- REPO_STRATEGY.md Section 3.1: frontend deployment correctly described (docker-compose on EC2 — no CloudFront unless explicitly demoed)
- REPO_STRATEGY.md Section 5.3: clarify pnpm-only OR also-accepting-npm (decide: pnpm only; npm is for global tools only)
- GIT_WORKFLOW.md `.gitignore`: clarify uv.lock decision (commit it; remove from gitignore) OR document why (pnpm equivalent vs uv equivalent)
- GIT_WORKFLOW.md Section 8.1: `git cherry-pick <commit-hash>` (not branch name)
- DEMO_GELISTIRME_KILAVUZU.md: npm references → pnpm
- DEMO_GELISTIRME_KILAVUZU.md: 8-block names → canonical (D7)
- DEMO_GELISTIRME_KILAVUZU.md: section 0 path placeholder; remove user-specific paths
- V5_IMPLEMENTATION_PLAN.md: "files to delete" section marked as historical (work done)
- V5_TELEMETRY_INTEGRATION_ANALYSIS.md: port 4317 (not 4318) for gRPC; agent role names = transcriber/drafter/reviewer (not analyst/architect)

**Execution steps:**
1. Read all six side guides
2. Patch them to align with the now-canonical Prompts and PCD
3. Reconcile npm/pnpm story
4. Reconcile uv.lock story

---

### P12 — Metric Dashboard (new Prompt_07 + page + backend API)

**Goal:** A dedicated `/metrics` page in the frontend that shows project-level and workflow-level metrics, pulling from RDS (run records) and live CloudWatch metrics via FastAPI.

**Acceptance criteria:**
- New file: `Prompt_07_Metrics_Dashboard.md` — self-contained prompt following the same format as Prompt_00–06 (entry gate, scope, files created, exit gate, troubleshooting)
- **Backend additions (Prompt_03 update + Prompt_07 detail):**
  - `backend/app/api/metrics_routes.py` — five endpoints:
    - `GET /api/v1/metrics/overview` → total runs (period), success rate, avg cost USD, total agent-hours, total estimated time-saved hours (from agent payload's `manual_baseline_hours - agent_active_hours - human_review_hours`)
    - `GET /api/v1/metrics/by-template?workflow_template_id=...&since=...&until=...` → per-template aggregates (count, p50/p95 latency, avg confidence, acceptance rate, total cost)
    - `GET /api/v1/metrics/by-project?project_id=...` → per-project aggregates with same shape, plus `sector` and `vertical` filters
    - `GET /api/v1/metrics/cloudwatch?metric=Latency|TokensInput|TokensOutput|AgentCostUsd&period=300&start=...&end=...&template=...` → boto3 `cloudwatch.get_metric_data()` query returning datapoints array
    - `GET /api/v1/metrics/computed?since=...&until=...` → Guideline §9.6 formulas applied: `time_saved_hours`, `net_savings_usd`, `roi_pct`, `manual_equivalent_fte`, `speed_multiplier`, `acceptance_rate`, `traceability_completeness`
  - `backend/app/services/metrics_service.py` — aggregation logic reading from `evidence_packs` and `agent_runs` tables; SQLAlchemy 2.0 queries with `func.percentile_cont` for p50/p95
  - `backend/app/services/cloudwatch_client.py` — boto3 wrapper, paginated `get_metric_data`, transforms response to `[{timestamp, value}]`
  - Pydantic response models for each endpoint (in `backend/app/api/schemas/metrics.py`)
  - IAM: EC2 role needs `cloudwatch:GetMetricData` (add to Prompt_00 IAM policy if missing)
- **Frontend additions (Prompt_05 update + Prompt_07 detail):**
  - `frontend/app/metrics/page.tsx` — server component renders the dashboard shell
  - `frontend/app/metrics/MetricsClient.tsx` — client component with React Query hooks
  - `frontend/components/metrics/SummaryCards.tsx` — 4 cards: Total Runs / Success Rate / Total Cost / Time Saved
  - `frontend/components/metrics/CostTimeSeries.tsx` — Recharts `LineChart` of cost over time
  - `frontend/components/metrics/LatencyChart.tsx` — Recharts `LineChart` with p50/p95/p99 lines
  - `frontend/components/metrics/TemplateBreakdown.tsx` — Recharts `BarChart` or table per workflow_template
  - `frontend/components/metrics/ProjectFilter.tsx` — dropdown filter; respects sector/vertical
  - `frontend/components/metrics/DateRangePicker.tsx` — `last 24h / last 7 days / last 30 days / custom`
  - `frontend/hooks/useMetrics.ts` — React Query hooks: `useOverview`, `useByTemplate`, `useByProject`, `useCloudWatchSeries`, `useComputedMetrics`
  - `frontend/lib/api/metrics.ts` — fetch wrappers, TypeScript types matching backend Pydantic models
  - Page navigation: top-level link from header to `/metrics` (separate from `/workspace`)
  - Recharts dependency added to `package.json`: `recharts@^2.12.0`
- **Deliverable_7 updated:** new "Reading from the dashboard" section explaining the query patterns; CloudWatch IAM addition; metric namespace consistent with what ADOT writes (`DemoSDLC/Agent`)
- **Permission scope:** Mock auth persona must include role `metrics_viewer` (auto-granted to all demo personas)
- **Caching:** React Query default `staleTime: 60_000` (1 min) so the dashboard does not hammer CloudWatch
- **Smoke test:** Prompt_07 includes a `curl` for each endpoint + a Playwright/manual check that the page renders with mock data
- **Cost note:** CloudWatch `get_metric_data` is billed per 1000 metric values; the dashboard caps queries to last 30 days max + period >= 300s to keep cost predictable

**Execution steps:**
1. Author Prompt_07_Metrics_Dashboard.md from scratch
2. Patch Prompt_03 with the metrics_routes registration, IAM addition, schema models
3. Patch Prompt_05 with the recharts dependency and metrics page scaffolding (so when P9 lands, the page already exists)
4. Patch Deliverable_7 with the dashboard query patterns
5. Verify Guideline §9.6 formulas are encoded correctly in metrics_service.py
6. Update PHASE_PLAN.md to add Phase 7 (Metrics Dashboard) — entry gate after Phase 6, exit gate checks dashboard renders

---

### P13 — Final audit pass

**Goal:** Produce a fresh audit report ("Round 3") that lists what was changed, what remains as known issue, and a clean FILE_STATUS_REPORT.

**Acceptance criteria:**
- New file: CONSISTENCY_AUDIT_REPORT_ROUND3.md, lists every issue found in this exercise, the fix applied, and the package number that landed the fix
- FILE_STATUS_REPORT.md: internal contradiction on Deliverable_7 resolved; status column matches detailed notes; PROJECT_STARTER, APPLY_PLAN, PAYLOAD_SCHEMA, Prompt_07, CONSISTENCY_AUDIT_REPORT_ROUND3 added to the index
- DOCUMENT_AUDIT_REPORT.md: marked as "Historical (Round 1)"; pointer to Round 3 added
- Cross-reference check: for every file mentioned in PCD, confirm it exists in the resources dir
- Naming check: grep for "ag_ui_strands", "context_block", "input_block", "analyst", "architect", "EMF formatting", "jnj." (outside historical reports) — none should appear in current Prompts/Deliverables
- Date check: every doc with a date field has 2026-05-15 or a more recent date
- Final pass: spawn an Explore agent to verify all decision items D1-D14 are honored across the file set

**Execution steps:**
1. Run grep over all files for the forbidden strings
2. Write Round 3 report
3. Update FILE_STATUS_REPORT
4. Update DOCUMENT_AUDIT_REPORT header
5. Run final cross-reference pass

---

## 3. Approval Checkpoint

This plan is awaiting user approval. After approval:
- Execution proceeds **package by package**
- At the end of each package, a status report is produced
- The user can pause, redirect, or approve continuation
- If a package reveals new contradictions, those are surfaced before continuing

End of plan.
