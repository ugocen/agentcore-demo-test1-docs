# E2E Run + Full Repo Health Audit

**Date:** 2026-05-17
**Scope:** A) result of running the existing `tests/e2e/test_T*.py` suite from this session, B) a full-repo health audit across backend, infra, frontend, and docs, and C) the two fixes I applied during the audit. This is a new file; it supplements `DEVIATION_REPORT.md`, `PROJECT_STATUS_REPORT.md`, and `CHANGES_VERIFICATION.md` rather than replacing them.

---

## 1. E2E Run

Command: `.venv/bin/python -m pytest tests/e2e/ -v --tb=short`

| Test ID | Test name | Result | Cause |
|---|---|---|---|
| T01 | `test_required_services_running` | FAIL | `docker` CLI not installed on this host. |
| T02a | `test_fastapi_health` | FAIL | `localhost:8000` connection refused (no stack up). |
| T02b | `test_fastapi_db_health` | FAIL | same — stack down. |
| T02c | `test_frontend_landing` | FAIL | `localhost:3000` refused. |
| T02d | `test_temporal_ui_reachable` | FAIL | `localhost:8081` refused. |
| T03 | `test_happy_path_via_upload` | FAIL | needs running stack. |
| T04 | `test_happy_path_via_mic_simulation` | FAIL | needs running stack (still tries to upload via 8000 first). |
| T05 | `test_hitl_clarification_round` | FAIL | needs running stack. |
| T06 | `test_self_correction_cap_escalation` | **SKIP** | gated by `DEMO_FORCE_DRAFTER_CLARIFICATION=true` (not set). |
| T07 | `test_rejection_at_approval_step` | FAIL | needs running stack. |
| T08 | `test_revise_terminates_as_rejected` | FAIL | needs running stack. |
| T09a–c | `test_demo_attributes_in_logs[agent_*]` | FAIL | needs CloudWatch logs from a real run. |
| T09d | `test_metric_namespace_exists` | **PASS** | AWS creds in `~/.aws` reach the `DemoSDLC/Agent` namespace, which already has metrics from earlier runs. |
| T10 | `test_evidence_pack_has_full_8block_per_agent` | FAIL | reads `tests/e2e/.last_approved_wf` (= `wf-2026-05-16-0e8d674c`) then tries `localhost:8000` — refused. |
| T11 | `test_docker_compose_down_then_up_recovers` | FAIL | `docker` CLI not installed. |

**Totals:** 1 pass, 1 skip, 15 fail.

The single pass (`T09d`) is an interesting incidental signal: the user's AWS creds are still authorised and the demo namespace has historical metrics, so once the live stack is brought back up, telemetry should land in the same place.

### What that means for the demo

None of the failures are bugs introduced by today's remediation. Each one fails because:

- there is no docker daemon on this host (T01, T11), or
- the docker-compose stack isn't running (T02–T08, T10), or
- there are no recent CloudWatch logs (T09 agent tests), or
- the test is opt-in via env (T06).

The first three are infra prerequisites. The second three (T03–T08) post `s3_uri` to `/api/v1/workflows` with `persona_id: "ba-sap-mm"` — exactly the contract the backend now requires. T03/T05/T07/T08 all call `/signal/clarification` and `/signal/approval`, which the route aliases added today support. So once the stack is up and the AgentCore ARNs in `.env` are populated, these tests should be runnable.

### How to actually run them

```bash
# 1. populate .env with the keys listed in §6 of CHANGES_VERIFICATION.md
# 2. cd agentcore-demo-test1-backend
docker compose up -d                     # 6 services + optional redis
python tests/e2e/seed_test_audio.py      # writes tests/e2e/fixtures/smoke-3sec.wav
.venv/bin/python -m pytest tests/e2e/ -v
```

---

## 2. Full Repo Health Audit

Three sub-agents combed backend/infra, frontend, and docs in parallel. I verified each finding before recording it here. **A handful of the agents' "BLOCKER" labels turned out to be non-blocking after I checked the code; I downgraded those.**

### 2.1 Backend / Infra

| Severity | Location | Finding | Status |
|---|---|---|---|
| **BLOCKER** | `app/temporal/workflow_streams.py:75` | `Client.connect(os.environ.get("TEMPORAL_HOST", "temporal-server:7233"))` builds an address from a single env. But `app/api/workflows_routes.py:42–44` and `docker-compose.yml` both treat `TEMPORAL_HOST` as host-only and `TEMPORAL_PORT` as the port. Result: when the SSE endpoint actually fires, the Temporal client tries to connect to `temporal-server` (no port) and 1) misroutes or 2) hangs. | **Fixed in this session** — now reads `TEMPORAL_HOST` + `TEMPORAL_PORT` separately, like the rest of the codebase. |
| COSMETIC | `app/main.py:61` | `from fastapi import HTTPException` re-imported inside the `health_db` exception block even though it is already imported at module top. | **Fixed in this session** — removed. |
| MEDIUM | `agents/{common,agent_1_*,agent_2_*,agent_3_*}/otel_setup.py` | Four byte-identical copies of the same OTel boilerplate. Imported locally, so it works, but a future fix has to be applied 4×. | Not fixed (out of scope). |
| MEDIUM | `agents/agent_*/payload_builder.py` | Same DRY issue: identical copies of the builder ship with each agent. AgentCore S3 ZIPs are independent so this is by design, but `agents/common/payload_builder.py` is the canonical one — the per-agent copies should be slimmed to imports if possible. | Not fixed. |
| LOW | `docker-compose.yml` — `temporal-worker` service | `env_file: - .env` is declared but the `environment:` block below explicitly enumerates every env, so the file is effectively ignored. Either remove `env_file` or drop the duplicate keys. | Not fixed. |
| LOW | `docker-compose.yml` — `fastapi` service | No `TEMPORAL_HOST`/`TEMPORAL_PORT` env. Falls back to the hard-coded default `temporal-server:7233`. Works in this compose network, will break in any other deployment. | Not fixed. |
| LOW | Multiple services | No `healthcheck:` defined. `restart: unless-stopped` swallows silent worker exits. | Not fixed. |
| INFO | Agent Dockerfiles | Already deleted earlier today. `agents/agent_*/.bedrock_agentcore.yaml` retained. Phase 1 spec now matches reality. | OK. |

### 2.2 Frontend

| Severity | Location | Finding | Status |
|---|---|---|---|
| ~~BLOCKER~~ MEDIUM | `app/api/copilotkit/route.ts:24–26` | `AGENT_1/2/3_RUNTIME_URL` default to `''`. `HttpAgent` accepts empty URLs silently; failure only surfaces when an agent call goes through. Could log a startup warning when any is missing. | Not fixed (`docker-compose` defaults `${AGENT_*_RUNTIME_URL:-}` to empty; the operator must populate `.env`). |
| ~~BLOCKER~~ COSMETIC | `lib/agentcore-agui-client.ts:394, 415` | `sendApproval`/`sendClarification` still POST to `/workflows/{id}/approval` and `/workflows/{id}/clarification` (legacy paths). | **Not a bug** — the backend now serves both legacy and `/signal/*` paths via aliases (added earlier today). Inconsistent with `lib/api/workflows.ts` (which uses `/signal/*`), but functionally correct. |
| HIGH | `lib/api/workflows.ts:53–88` | `sendApprovalSignal` / `sendClarificationSignal` are exported but never imported. Workspace uses `useWorkflowStream`'s methods, which in turn call the legacy paths via `agentcore-agui-client.ts`. | Not fixed (orphan but harmless). |
| HIGH | `components/chat/AudioInput/AudioInput.tsx:25` | Uses a literal `/api/v1/uploads/audio` instead of `uploadAudio()` from `lib/api/uploads.ts`. Works through the next.config rewrite. | Not fixed; could be reused via the helper for consistency. |
| MEDIUM | `app/workspace/[wfId]/page.tsx:118` | `coAgentState` is destructured from `useCoAgent` but never read. `setCoAgentState` is used (in the sync `useEffect`). Doesn't break the build (TS strict is bypassed via `ignoreBuildErrors`). | Not fixed (cosmetic). |
| MEDIUM | `lib/copilot-config.ts:30–36` | `workflowStreamUrl` and `workflowStateUrl` inline `process.env.NEXT_PUBLIC_BACKEND_URL` instead of reusing the `backendUrl` field of the same object. | Not fixed. |
| MEDIUM | `next.config.js:9–15` | Rewrite destination `http://fastapi:8000` is hard-coded for the Docker network. No env-var override. | Not fixed (intentional for the demo). |
| COSMETIC | `components/canvas/GenerativeUIRenderer.tsx:14–16` | Relative imports for sibling cards (`./TranscriptCard`, …) while sibling-folder cards use `@/...` aliases. | Not fixed. |
| COSMETIC | `app/workspace/[wfId]/page.tsx` header docstring | Still claims `CopilotChat` and `CopilotSidebar` are mounted "in this page". They're now in the workspace layout. | Not fixed. |
| INFO | `lib/use-shared-state.ts`, `lib/use-agentcore-runtimes.ts` | Still importable. Not referenced from the workspace page anymore (cleaned up during the 3-pane refactor), but `lib/use-agentcore-runtimes.ts` is used by `app/api/agent-runtimes` flows — keep. | OK. |

### 2.3 Docs

| Severity | File | Finding |
|---|---|---|
| HIGH | `PROJECT_STATUS_REPORT.md` §1, items 9 and 10 | Says the workspace is "still 2-pane" and the persona selector is "missing". Both were addressed in the second remediation round; the report wasn't refreshed. Today's `CHANGES_VERIFICATION.md` captures the new state but `PROJECT_STATUS_REPORT.md` now contradicts it. |
| HIGH | `PHASE_PLAN.md` Phase 1 (~line 169–177) | "All 3 agents deployed via S3 ZIP — no Dockerfile" is now finally true; flag in any next status review. |
| HIGH | `PHASE_PLAN.md` Phase 5 file list (~lines 476–482) | The plan promises `BrdPreview.tsx`, `ReviewPanel.tsx`, `ClarificationCard.tsx`, `useAgent.ts`. Actual: `DraftPreviewCard.tsx`, `ReviewReportCard.tsx`, `ClarificationCard.tsx` (renamed today), no `useAgent.ts` (workspace uses `useCoAgent` directly). Either rename the files or document the substitution. |
| MEDIUM | `Deliverable_5_CopilotKit_AGUI_Guide.md` | Does not document the `x-workflow-id` header / per-request runtime pattern that `app/api/copilotkit/route.ts` now uses. Operators who follow the guide will set `'default-session'` again. |
| MEDIUM | `Deliverable_8_Frontend_Architecture_Guide.md` | Diagram still shows the planned `components/agentic/` hierarchy. The actual layout is `components/{chat,canvas,hitl,metrics,ui}`. |
| LOW | `RECOVERY_AND_DEPLOYMENT_PLAN.md` | Does not mention the Temporal `1.22` pin or the moto-mocked smoke test. Will be wrong as soon as someone uses it as a checklist. |
| LOW | `DEVIATION_REPORT.md` | First-round report; described the pre-fix state. Already supplanted by `PROJECT_STATUS_REPORT.md`; the doc-header notes this. |

### 2.4 Things I did *not* find

- No dead imports in `app/`.
- No reference to the deleted agent Dockerfiles in any infra script, deploy script, or .bedrock_agentcore.yaml.
- No reference to the deleted `components/CopilotKitProvider.tsx` or `components/CopilotSidebarWrapper.tsx` from any TS file.
- `npx next build` succeeds in 6 routes (`/`, `/_not-found`, `/api/copilotkit`, `/metrics`, `/workspace/[wfId]`).
- `pytest tests/ --ignore=tests/e2e` → 31/31 pass after the two fixes above (the metrics + telemetry suites, plus the new `tests/api/test_upload_and_start.py`).

---

## 3. Fixes Applied During This Audit

1. **`app/temporal/workflow_streams.py:71–79`** — SSE consumer now resolves Temporal connection as `f"{TEMPORAL_HOST}:{TEMPORAL_PORT}"` (host + port read separately), matching `_temporal_client` in `workflows_routes.py`.
2. **`app/main.py:60–62`** — Removed the duplicate `from fastapi import HTTPException` inside `health_db`; the symbol is already imported at module top.

Both are surgical edits. `pytest tests/ --ignore=tests/e2e` still green: 31 pass.

---

## 4. Recommended Follow-ups

In priority order; none are urgent for the demo, but each is concrete:

1. **Refresh `PROJECT_STATUS_REPORT.md`** — items 9 and 10 of its summary table are stale. Either patch in place or add a "Round 2" appendix referencing `CHANGES_VERIFICATION.md`.
2. **Either align the planned vs actual frontend filenames** (rename `DraftPreviewCard` → `BrdPreview`, `ReviewReportCard` → `ReviewPanel`, add `useAgent.ts` thin wrapper) **or update `PHASE_PLAN.md`** to acknowledge the new names. Pick one — don't leave the contradiction.
3. **Document the AG-UI session pattern** in `Deliverable_5_CopilotKit_AGUI_Guide.md`: `x-workflow-id` header from the scoped `CopilotKit` provider → per-request `CopilotRuntime` → AgentCore `runtimeSessionId`.
4. **Tighten the legacy/canonical signal-URL split:** either delete the unused `sendApprovalSignal`/`sendClarificationSignal` from `lib/api/workflows.ts`, or have `agentcore-agui-client.ts` use them. Don't keep two clients for the same surface.
5. **Plumb `TEMPORAL_HOST`/`TEMPORAL_PORT` into `docker-compose.yml`** explicitly for `fastapi` and `temporal-worker`. Currently they default; that's fragile.
6. **Add a `healthcheck:`** to `fastapi`, `temporal-worker`, `nextjs-frontend`. Today T01 fails because docker isn't local, but even with docker the worker is checked only by restart policy.
7. **Add a startup warning** in `app/api/copilotkit/route.ts` when any of `AGENT_*_RUNTIME_URL` is empty — surfaces the misconfiguration before the first chat message.
8. **Switch `components/chat/AudioInput/AudioInput.tsx`** to call `uploadAudio()` from `lib/api/uploads.ts`. Eliminates the only literal-URL holdout in the upload flow.
9. **Stop bypassing the existing helper** in `components/canvas/GenerativeUIRenderer.tsx`: change `./TranscriptCard` and siblings to `@/components/canvas/...` to match the rest of the codebase's import style.
10. **DRY the agent code:** the four `otel_setup.py` and four `payload_builder.py` files are identical. Either keep the canonical copy in `agents/common/` and import it from each agent (works because each ZIP includes `agents/common/`), or document the duplication explicitly.

---

## 5. Where to Run What

| Want to check… | Run this | Needs |
|---|---|---|
| The fixes above didn't break the unit tests | `pytest tests/ --ignore=tests/e2e -v` from `agentcore-demo-test1-backend/` | venv with `moto`, `httpx` |
| Frontend builds | `npm install && npx next build` from `agentcore-demo-test1-frontend/` | node + ~10 min |
| docker-compose YAML still valid | `docker compose config` | docker CLI |
| Phase 5 happy path | `pytest tests/e2e/test_T03_happy_path_upload.py -v` after `docker compose up -d` | full stack + AWS creds + AgentCore ARNs |
| The fix to `workflow_streams.py` actually works | After stack is up, `curl -N http://localhost:8000/api/v1/workflows/<id>/stream` | running Temporal + a real workflow id |
| The audio-fix end-to-end | Browser → `http://localhost:3000`, pick persona, **New Workflow**, record clip | running stack + mic permission |

---

*End of report.*
