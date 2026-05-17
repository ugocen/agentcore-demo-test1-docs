# Project Status & Deviation Report

**Date:** 2026-05-17 (Round 1 snapshot — see appendix for Round 2 + 3 changes)
**Scope:** Snapshot of the AgentCore demo after the first round of remediation. Supersedes `DEVIATION_REPORT.md` (which captured the pre-fix state). Subsequent changes are recorded in `CHANGES_VERIFICATION.md` (Round 2) and `E2E_RUN_REPORT.md` (Round 3 + ongoing follow-ups).
**Baseline:** `PHASE_PLAN.md` v2.0, `Deliverable_0_PROJECT_CONTEXT.md`, `Deliverable_5_CopilotKit_AGUI_Guide.md`, `Deliverable_8_Frontend_Architecture_Guide.md`.

> **Note (2026-05-17, later in the day):** Items 9, 10, 11, and 12 in §1 were closed in Round 2 and Round 3. See the **Appendix: Status as of latest update** at the bottom of this file for the current state of each row.

---

## 1. Executive Summary

| # | Item | Plan | Status Now | Severity (Before → After) |
|---|---|---|---|---|
| 1 | Audio capture UI in workspace (mic + file upload) | Required (D13) | **Fixed** — `<AudioInput>` mounted in `app/workspace/[wfId]/page.tsx` when `wfId === 'new'`; `useMicRecorder` and file picker wired through | CRITICAL → OK |
| 2 | `POST /api/v1/uploads/audio` actually stores to S3 | Required (Phase 3) | **Fixed** — `app/main.py` validates content-type + 50 MB cap and calls `boto3 s3.put_object` | CRITICAL → OK |
| 3 | `POST /api/v1/workflows` requires `audio_s3_uri` | Required (Phase 4 / D11) | **Fixed** — placeholder fallback removed; missing/invalid URI returns 400 | CRITICAL → OK |
| 4 | CopilotKit provider scoped to `/workspace/[wfId]` | Required (Deliverable 8) | **Fixed** — `app/workspace/[wfId]/layout.tsx` added, root layout cleared | High → OK |
| 5 | AG-UI session id = active `workflow_id` | Required (Phase 5) | **Fixed** — `/api/copilotkit/route.ts` rebuilds runtime per request and uses `x-workflow-id` header (padded to AgentCore's 33-char minimum) | High → OK |
| 6 | AG-UI runtime URLs reachable in container | Required | **Fixed** — `AGENT_1/2/3_RUNTIME_URL` + `NEXT_PUBLIC_*` env block added to `nextjs-frontend` service in `docker-compose.yml` | High → OK |
| 7 | Workflow signal endpoint paths | `/signal/clarification`, `/signal/approval` | **Fixed** — backend now serves both the planned `/signal/*` and the current `/clarification`/`/approval` paths via route aliases | Low → OK |
| 8 | Temporal server image | `temporalio/auto-setup:1.22` | **Fixed** — pinned to `1.22` in `docker-compose.yml` | Low → OK |
| 9 | Workspace layout = 3-pane (chat / canvas / status) | Required (Phase 5) | **Still 2-pane** (Generative UI / Event Stream) + audio panel + floating CopilotSidebar | Medium |
| 10 | Frontend component layout matches Deliverable 8 | `components/{chat,canvas,hitl}/...`, `lib/{api,auth}/...` | **Partial** — required logic exists, but several files live at flat paths instead | Medium |
| 11 | Agents shipped as S3 ZIP only | Required (Phase 1) | **Still has Dockerfiles** in each `agents/*` directory | Low |
| 12 | Persona selector on landing | Required (Phase 5) | **Missing** — `persona_id='ba-sap-mm'` hard-coded in workspace upload handler | Low–Medium |

The four CRITICAL/High items called out in the previous report are all now closed in code; the remaining items are scoped polish or unrelated to the user's reported blocker.

---

## 2. What Was Changed Today

### 2.1 Backend (`agentcore-demo-test1-backend/`)

| File | Change |
|---|---|
| `app/main.py` | Real S3 upload in `POST /api/v1/uploads/audio`: validates `Content-Type` against the D13 allowlist (`mp3`, `mpeg`, `wav`, `x-wav`, `m4a`, `x-m4a`, `webm`, `ogg`), enforces 50 MB cap, refuses empty bodies, `put_object` to `AUDIO_BUCKET`, returns `s3://…` URI. Adds `boto3`/`botocore` imports. |
| `app/api/workflows_routes.py` | `StartWorkflowRequest.audio_s3_uri` is now required (was `str | None`). `start_workflow` validates the `s3://` prefix and raises 400 instead of silently substituting `placeholder/empty.wav`. Removed the `AUDIO_BUCKET` placeholder construction. |
| `app/api/workflows_routes.py` | Added `/{workflow_id}/signal/clarification` and `/{workflow_id}/signal/approval` route aliases alongside the existing short paths so the docs-canonical URLs work. |
| `docker-compose.yml` | `temporal-server` image pinned to `1.22`. Added runtime `environment` block on `nextjs-frontend` for `NEXT_PUBLIC_BACKEND_URL`, `NEXT_PUBLIC_COPILOT_RUNTIME_URL`, `AGENT_1/2/3_RUNTIME_URL`, `AWS_REGION`. |

### 2.2 Frontend (`agentcore-demo-test1-frontend/`)

| File | Change |
|---|---|
| `app/layout.tsx` | Removed `CopilotKitProvider` from the root layout. Now intentionally bare. |
| `app/workspace/[wfId]/layout.tsx` | **New file.** Mounts `<CopilotKit>` and `<CopilotSidebar>` scoped to the workspace. Reads `wfId` from the route and forwards it to the CopilotKit runtime as `x-workflow-id` header. |
| `app/api/copilotkit/route.ts` | Rewritten. Builds `CopilotRuntime` per request and feeds `X-Amzn-Bedrock-AgentCore-Runtime-Session-Id` from the incoming `x-workflow-id`. Pads to 33 chars when the header is shorter than the AgentCore minimum. Three `HttpAgent` entries (`strands_transcriber`/`strands_drafter`/`strands_reviewer`) read `AGENT_1/2/3_RUNTIME_URL`. |
| `app/workspace/[wfId]/page.tsx` | Imports `AudioInput`. Treats `wfId === 'new'` as a synthetic id: SSE (`useWorkflowStream`) and state polling are gated to live ids; the audio panel renders with mic + upload; `onUploaded` posts `audio_s3_uri` to `POST /api/v1/workflows` and `router.replace`s to the real workspace url. Mini-metrics, steps, and the 2-column live grid are hidden while `isNew`. |
| `app/page.tsx` | Landing "New Workflow" button now just `router.push('/workspace/new')`. The workflow is created lazily once audio is uploaded. |

### 2.3 Files unchanged but worth noting

- `components/chat/AudioInput/AudioInput.tsx` and `components/chat/AudioInput/useMicRecorder.ts` already had the right contract — they only needed to be mounted.
- `components/CopilotKitProvider.tsx` is now orphan code (no imports). Safe to delete later.

---

## 3. Verification Performed

| Check | Result |
|---|---|
| `python -m yaml.safe_load` of `docker-compose.yml` | Parses; services: `temporal-server`, `temporal-ui`, `fastapi`, `temporal-worker`, `nextjs-frontend`, `adot-collector`, `redis` |
| `ast.parse` of `app/main.py`, `app/api/workflows_routes.py` | Clean parse, no syntax errors |
| `grep -RnE "AudioInput\|useMicRecorder"` from frontend root | Now referenced from `app/workspace/[wfId]/page.tsx`, not just its own folder |
| JSX bracket balance for the new `{!isNew && (…)}` guards in workspace page | Verified by re-reading the modified block (lines 444–571) |

What was **not** run because the local toolchain isn't available in this session:

- `pnpm install && pnpm build`
- `docker compose up -d` with a real AWS environment
- End-to-end recording → workflow start → SSE
- Pytest suite under `agentcore-demo-test1-backend/tests/`

Treat the report as "code-level remediation verified; runtime verification still pending."

---

## 4. Remaining Deviations from the Plan

### 4.1 Workspace layout (Medium)

`PHASE_PLAN.md` Phase 5 calls for a 3-pane workspace: chat (left), canvas/BRD preview (centre), workflow status (right). The current page is 2-column (Generative UI cards left, AG-UI event stream right) with the CopilotKit sidebar floating in. The audio input now renders above this grid when the workspace is new. Functional, but not structurally aligned with Deliverable 8 mockups.

### 4.2 Component file layout (Medium)

The plan lists these paths; current locations differ:

| Planned | Actual |
|---|---|
| `components/chat/ChatPanel.tsx` | Missing — workspace inlines the audio panel |
| `components/canvas/BrdPreview.tsx` | Replaced by `components/DraftPreviewCard.tsx` |
| `components/canvas/ReviewPanel.tsx` | Replaced by `components/ReviewReportCard.tsx` |
| `components/hitl/ClarificationCard.tsx` | Lives at `components/ClarificationQuestionCard.tsx` |
| `components/hitl/ApprovalCard.tsx` | Lives at `components/ApprovalCard.tsx` |
| `hooks/useAgent.ts` | Missing — `useCoAgent` used directly |
| `hooks/useWorkflowStream.ts` | At `lib/use-workflow-stream.ts` |
| `lib/api/workflows.ts`, `lib/api/uploads.ts`, `lib/auth/mockAuth.ts` | Inlined in `app/workspace/[wfId]/page.tsx`, `app/page.tsx`, `AudioInput.tsx` |

### 4.3 Persona selector (Low–Medium)

`PHASE_PLAN` Phase 5 / Scope: *"Landing page (`app/page.tsx`): persona selector → starts a new workflow session."* The current landing page lists feature checklist + metrics + workflow table, no persona selector. The workspace audio-upload handler hard-codes `persona_id: 'ba-sap-mm'`. Acceptable for a single-persona demo but diverges from D9.

### 4.4 Agent Dockerfiles (Low)

`Dockerfile` is present in each `agents/agent_*` directory. PHASE_PLAN Phase 1 explicitly: *"All 3 agents deployed via S3 ZIP — no Dockerfile, no ECR."* The Dockerfiles are unused by the deployment path but represent doc drift.

### 4.5 Orphan provider component (Cosmetic)

`components/CopilotKitProvider.tsx` is no longer imported anywhere. Either delete it or repurpose if a future surface needs CopilotKit outside `/workspace/[wfId]`.

### 4.6 AudioInput uses a literal URL (Cosmetic)

`components/chat/AudioInput/AudioInput.tsx:25` calls `fetch("/api/v1/uploads/audio", …)`. This works because of the Next.js rewrite in `next.config.js`, but it bypasses `COPILOT_CONFIG.backendUrl` and will fail if the frontend is ever pointed at a non-default backend without the rewrite in place.

---

## 5. Pre-Existing Items Confirmed Still Working

These were already correct before the remediation and remain so:

- All 3 agent runtimes (`agents/agent_1_transcriber`, `agent_2_drafter`, `agent_3_reviewer`) with `payload_builder`, `otel_setup`, Strands/LangGraph code.
- Temporal `BrdFromAudioWorkflow` (`app/temporal/workflows.py`) with `transcriber → drafter (HITL with self-correction cap) → reviewer → approval-gate`, evidence-pack init/append/finalize.
- Single canonical `invoke_agent_runtime_activity` (per D4) with 8-block validation, request hashing, and `AGENTCORE_DRY_RUN` short-circuit.
- SSE workflow stream endpoint (`GET /api/v1/workflows/{id}/stream`) and REST snapshot endpoint (`/state`).
- ADOT collector container + CloudWatch emit path (`_emit_cloudwatch_telemetry`).
- AG-UI client packages installed (`@ag-ui/client` 0.0.53, `@ag-ui/core` 0.0.53) and imported by `lib/agentcore-agui-client.ts` and `app/api/copilotkit/route.ts`.
- Mock-auth persona table (`get_mock_user`) and `evidence_packs` SQLAlchemy model.

---

## 6. Operator Action Items Before Running

These are environmental, not code:

1. **`.env` values.** Populate at least:
   - `RDS_HOST`, `RDS_USER_TEMPORAL`, `RDS_PASSWORD_TEMPORAL`, `RDS_DB_TEMPORAL`, `RDS_DB_TEMPORAL_VISIBILITY` (for `temporal-server`)
   - `RDS_USER_APP`, `RDS_PASSWORD_APP`, `RDS_DB_APP` (for `fastapi`/`temporal-worker`)
   - `S3_BUCKET_AUDIO`, `S3_BUCKET_ARTIFACTS`, `S3_BUCKET_CLAIMCHECK`
   - `AGENTCORE_ARN_TRANSCRIBER`, `AGENTCORE_ARN_DRAFTER`, `AGENTCORE_ARN_REVIEWER`
   - `AGENT_1_RUNTIME_URL`, `AGENT_2_RUNTIME_URL`, `AGENT_3_RUNTIME_URL` (now consumed by the Next.js runtime route)
   - `AWS_REGION` (defaults to `us-east-1`)
2. **AWS credentials reachable inside the `fastapi` and `temporal-worker` containers.** The new S3 put call relies on the default boto3 credential chain. On EC2 the instance role works; locally you need `~/.aws/credentials` mounted or `AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY` env vars.
3. **Temporal volume reset.** If you previously ran the stack with `auto-setup:1.29`, the Postgres schema may be ahead of `1.22`. If `temporal-server` fails to start after the version pin change, you need to drop and recreate the `temporal` and `temporal_visibility` databases, or revert to a newer image if downgrade is impractical (in which case update `PHASE_PLAN` to match).
4. **Browser permission for the microphone.** Modern browsers require https or `http://localhost` for `getUserMedia`. The default `http://localhost:3000` is fine; a remote IP needs TLS.

---

## 7. End-to-End Flow After Remediation

1. User visits `/`.
2. Clicks **New Workflow** → `router.push('/workspace/new')`.
3. Workspace layout mounts CopilotKit scoped to `wfId='new'` (session id padded to 33 chars).
4. Workspace page sees `isNew === true`, renders `<AudioInput>` only (no live metrics/grid).
5. User clicks 🎤 Record or 📎 Upload → `useMicRecorder` produces a webm/opus blob OR the file picker hands over a file.
6. Send → `POST /api/v1/uploads/audio` (Next.js rewrite → FastAPI) → S3 `put_object` → returns `{s3_uri, content_type}`.
7. Workspace `handleAudioUploaded` posts `{audio_s3_uri, persona_id: 'ba-sap-mm'}` to `POST /api/v1/workflows`.
8. Backend validates `audio_s3_uri`, creates the `EvidencePack` row, starts `BrdFromAudioWorkflow` on Temporal, returns `workflow_id`.
9. Frontend `router.replace('/workspace/<wfId>')`. Layout remounts with the real id; runtime header switches to the real workflow id; SSE and `/state` polling begin.
10. Transcriber → Drafter (HITL loop) → Reviewer → Approval gate, rendered live via Generative UI cards and AG-UI event stream.

---

## 8. Recommended Next Steps (Not Yet Done)

Roughly in order of value:

1. **Restore a persona selector on the landing page.** Replace the hard-coded `'ba-sap-mm'` with a chooser and pass the user's pick into the workspace upload handler.
2. **Restructure the workspace into the 3-pane layout** from Deliverable 8 — promotes the audio panel into a persistent left chat column, and adds the canvas pane explicitly.
3. **Move/rename components** to match the planned paths (`components/{chat,canvas,hitl}/...`, `hooks/useWorkflowStream.ts`, `lib/api/{workflows,uploads}.ts`). Mostly mechanical, but keeps the codebase aligned with the architecture docs.
4. **Delete `components/CopilotKitProvider.tsx`** (orphan).
5. **Decide on the agent Dockerfiles** — either delete (to match Phase 1 spec) or update the docs to acknowledge the alternative path.
6. **Switch the AudioInput fetch URL to `COPILOT_CONFIG.backendUrl`** for environments without the Next.js rewrite.
7. **Add a smoke pytest** that posts a small wav to `/api/v1/uploads/audio` against `moto`-mocked S3, then posts the returned URI to `/api/v1/workflows`, asserting 202 + the workflow id format.
8. **Run the full Phase 5 checkpoint list** (CP-5.1 through CP-5.13) once the demo environment is up; treat any failure as a STOP per the Gate Pass Protocol in `PHASE_PLAN` §4.

---

## Appendix: Status as of latest update (Round 3 + follow-ups)

The original §1 table above captures the state after Round 1 only. Subsequent
sessions on the same day made the following changes; the row numbers track
the §1 numbering. See `CHANGES_VERIFICATION.md` for Round 2 evidence and
`E2E_RUN_REPORT.md` for Round 3 + follow-up evidence.

| # | Item | Status now | Pointer |
|---|---|---|---|
| 1 | Audio capture UI | OK (unchanged) | `components/chat/AudioInput/AudioInput.tsx`; now goes through `uploadAudio()` helper (FU#8) |
| 2 | Real S3 upload endpoint | OK (unchanged) | `app/main.py` |
| 3 | `audio_s3_uri` required | OK (unchanged) | `app/api/workflows_routes.py` |
| 4 | CopilotKit provider scope | OK (unchanged) | `app/workspace/[wfId]/layout.tsx` |
| 5 | AG-UI session id = workflow id | OK (unchanged) | `app/api/copilotkit/route.ts`. **FU#7 added:** module-load `console.warn` when any `AGENT_*_RUNTIME_URL` is empty. |
| 6 | AG-UI runtime URLs in container env | OK; **FU#5 added** `TEMPORAL_HOST`/`TEMPORAL_PORT`/`TEMPORAL_TASK_QUEUE` and dropped redundant `env_file` on `temporal-worker`. **FU#6:** healthchecks for `fastapi`/`temporal-worker`/`nextjs-frontend`. | `docker-compose.yml` |
| 7 | Signal endpoint paths | OK (unchanged) — both `/clarification` & `/signal/clarification`, both `/approval` & `/signal/approval`. | `app/api/workflows_routes.py` |
| 8 | Temporal image | OK (unchanged) — `temporalio/auto-setup:1.22`. | `docker-compose.yml` |
| 9 | **Workspace 3-pane layout** | **CLOSED in Round 2.** Workspace is now chat / canvas / status. | `app/workspace/[wfId]/page.tsx` + `components/chat/ChatPanel.tsx` |
| 10 | **Component layout matches plan** | **CLOSED in Round 2** (with one substitution table now in `PHASE_PLAN`). Files live under `components/{chat,canvas,hitl}/...`, `hooks/useWorkflowStream.ts`, `lib/api/{workflows,uploads}.ts`, `lib/auth/personas.ts`. **FU#4 cleanup:** removed the orphan `sendApprovalSignal`/`sendClarificationSignal` from `lib/api/workflows.ts`. **FU#9:** card imports in `GenerativeUIRenderer` switched from sibling-relative to `@/components/canvas/*`. | various |
| 11 | Agent Dockerfiles deleted | **CLOSED in Round 2.** Each `agents/agent_*/.bedrock_agentcore.yaml` retained; AgentCore deploys S3 ZIPs only. **FU#10 doc:** added `agents/common/README.md` + `scripts/sync_agent_common.sh` to keep the per-agent copies in lockstep with the canonical helpers. | `agents/` |
| 12 | **Persona selector** | **CLOSED in Round 2.** Landing page now has a `<select>` populated from `lib/auth/personas.ts`; the choice rides through to `/workspace/new?persona=…` and into `POST /workflows`. | `app/page.tsx`, `app/workspace/[wfId]/page.tsx` |

### Round 3 fixes (from `E2E_RUN_REPORT.md`)

- **`app/temporal/workflow_streams.py`** — SSE consumer now reads `TEMPORAL_HOST` + `TEMPORAL_PORT` separately (was previously expecting `host:port` in one var, contradicting the rest of the codebase).
- **`app/main.py`** — Removed a duplicate `from fastapi import HTTPException` inside the `health_db` exception block.

### Verification snapshot

- `pytest tests/ --ignore=tests/e2e` → **31/31 pass** (5 new `tests/api/test_upload_and_start.py` + 26 pre-existing).
- `npx next build` → **6 routes built** (`/`, `/_not-found`, `/api/copilotkit`, `/metrics`, `/workspace/[wfId]`).
- `python -m yaml.safe_load docker-compose.yml` → 7 services parse; `fastapi`, `temporal-worker`, `nextjs-frontend` carry healthchecks; `fastapi` + `temporal-worker` carry `TEMPORAL_HOST`/`TEMPORAL_PORT`.
- `bash scripts/sync_agent_common.sh` → idempotent (exit 0 when in sync).
- `tests/e2e/` → 1 pass (`T09::test_metric_namespace_exists` against the live AWS account), 1 skip (T06 opt-in env gate), 15 fail (all due to missing local docker / no running stack — none caused by remediation).

*End of report.*
