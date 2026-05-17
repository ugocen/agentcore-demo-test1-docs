# Architecture Deviation Report

**Date:** 2026-05-17
**Scope:** Comparison of the current repository state against the original architecture defined in `PHASE_PLAN.md` (v2.0) and `Deliverable_0_PROJECT_CONTEXT.md`.
**Primary finding:** Multiple Phase 5 (Frontend) requirements are missing or incorrectly scoped. **The user-reported blocker — no way to upload or record audio when starting a new workflow — is confirmed and is a direct violation of decision D13.**

---

## 1. Executive Summary

| Area | Plan | Actual | Severity |
|---|---|---|---|
| Audio input UI in workspace (record + upload) | Required (D13, Phase 5) | **Not wired to any page** — `AudioInput` component exists but is never imported | **CRITICAL** |
| Workflow start contract | `POST /api/v1/workflows {audio_s3_uri, persona_id}` after audio upload | Landing page posts `{agent_name, input:{query}}` *before* any audio exists; backend silently substitutes a placeholder S3 URI | **CRITICAL** |
| Backend audio upload endpoint | Multipart, content-type validation, S3 put with KMS | Stub: returns a fake `s3_uri` without performing any `PutObject` | **CRITICAL** |
| CopilotKit provider scope | `app/workspace/[wfId]/layout.tsx` (per Deliverable 8) | Mounted in root `app/layout.tsx`; no `workspace/[wfId]/layout.tsx` exists | **High** |
| AG-UI (`@ag-ui/client`, `@ag-ui/core`) installation | Required | **Installed** (package.json:20-21) and imported in `lib/agentcore-agui-client.ts` and `app/api/copilotkit/route.ts` | OK |
| AG-UI runtime wiring | `HttpAgent` per agent, URL built from runtime env vars | Code present in `app/api/copilotkit/route.ts` but `AGENT_*_RUNTIME_URL` env vars are **not injected** into the `nextjs-frontend` service in `docker-compose.yml` | **High** |
| Workspace layout | 3-pane (chat / canvas / status) | 2-pane (Generative UI cards / AG-UI event stream) + optional sidebar; **no chat pane** | Medium |
| Required UI components | `ChatPanel`, `BrdPreview`, `ReviewPanel`, `hitl/ClarificationCard`, `hitl/ApprovalCard`, `useAgent.ts`, `mockAuth.ts` | Not present (functionality partially replaced by `GenerativeUIRenderer`, `ApprovalCard`, `ClarificationQuestionCard` at different paths) | Medium |
| Agent packaging | "S3 ZIP only — NO Dockerfile" (Phase 1) | A `Dockerfile` exists in each agent directory | Low–Medium |
| Workflow signal API paths | `/api/v1/workflows/{id}/signal/clarification` and `signal/approval` (Phase 4) | `/api/v1/workflows/{id}/clarification` and `/approval` (no `signal/` segment) | Low (FE matches) |
| Temporal server image | `temporalio/auto-setup:1.22` (PCD §3) | `1.29` | Low |

---

## 2. The Audio-Input Blocker (User's Reported Issue)

### 2.1 What the plan says

`PHASE_PLAN.md` §Phase 5 / Scope (lines 449–460):

> Chat panel includes:
> - File upload (drag-drop): accepts `audio/mp3, audio/wav, audio/m4a, audio/webm`
> - **Microphone recording (D13):** `components/chat/AudioInput/useMicRecorder.ts` wraps `MediaRecorder` (no extra deps); records `audio/webm;codecs=opus`; shows a duration timer and preview before sending
> - Both paths upload via `POST /api/v1/uploads/audio` and use the returned `s3_uri` as workflow input

Phase 5 Exit Gate explicitly requires:
- *"Microphone recording works (`MediaRecorder` produces webm/opus, posted to `/api/v1/uploads/audio`)"*
- *"File upload works for mp3, wav, m4a, webm"*

### 2.2 What is in the code today

| Asset | Status | Path |
|---|---|---|
| `useMicRecorder` hook | Implemented | `components/chat/AudioInput/useMicRecorder.ts` |
| `AudioInput` UI (record + file picker + upload to `/api/v1/uploads/audio`) | Implemented | `components/chat/AudioInput/AudioInput.tsx` |
| Any import of `AudioInput` or `useMicRecorder` outside the folder | **None** (`grep` returns 0 hits) | — |
| Workspace page (`app/workspace/[wfId]/page.tsx`) | Renders Generative UI cards and AG-UI event stream only — no chat pane, no input field, no record/upload button | `app/workspace/[wfId]/page.tsx` |
| Landing page "New Workflow" handler | Posts a JSON body shaped as `{agent_name, input:{query:'Start new BRD-from-audio workflow'}}` directly (`app/page.tsx:126-145`) before any audio is captured | `app/page.tsx` |
| Backend `POST /api/v1/workflows` | Pydantic model is `{audio_s3_uri?: str, persona_id: str}` with `extra: "ignore"`. When `audio_s3_uri` is missing it substitutes `s3://<AUDIO_BUCKET>/placeholder/empty.wav` (`app/api/workflows_routes.py:106-132`) | `workflows_routes.py` |
| Backend `POST /api/v1/uploads/audio` | Returns a synthesised `s3_uri` string but performs **no S3 upload** (`app/main.py:64-73`). The file bytes are never read or persisted. | `main.py` |

### 2.3 Effect on the running demo

1. User clicks **New Workflow** on the landing page.
2. Frontend immediately starts a Temporal workflow with no audio.
3. Backend silently uses a placeholder S3 URI pointing to a non-existent object.
4. Transcriber activity either dry-runs a canned response or fails to read the object.
5. There is no UI surface on `/workspace/[wfId]` to upload audio after the fact, and even if there were, the workflow has already started without the URI.

This is exactly the user's symptom: "no area where I can upload audio… neither the microphone button nor manual upload is available."

### 2.4 Minimum fix to restore D13 behaviour

1. Add `app/workspace/[wfId]/layout.tsx` that mounts `CopilotKitProvider` (move it off the root layout per Deliverable 8).
2. Add a `ChatPanel` (or extend the workspace page) that renders `<AudioInput onUploaded={...} />`.
3. Change the "New Workflow" flow so the workflow is started **after** a successful upload, with the returned `s3_uri` sent as `audio_s3_uri` (matching the backend Pydantic model). Two viable shapes:
   - Landing page collects audio first, then redirects.
   - Workspace page accepts audio for `wfId === 'new'` and creates the workflow lazily.
4. Replace the stub `POST /api/v1/uploads/audio` in `app/main.py` with a real S3 putobject (the original Phase 3 endpoint contract — `agentcore-demo-test1-backend/app/api/uploads_routes.py` referenced in the plan does not exist today).
5. (Optional, also required by plan) Remove the silent placeholder fallback in `start_workflow` so a missing audio URI fails closed with HTTP 400.

---

## 3. AGUI Status

**Installed and partially wired.** Specifically:

- `@ag-ui/client` 0.0.53 and `@ag-ui/core` 0.0.53 are pinned in `package.json` (lines 20-21).
- `HttpAgent` from `@ag-ui/client` is imported and instantiated for three named agents (`strands_transcriber`, `strands_drafter`, `strands_reviewer`) in `app/api/copilotkit/route.ts:42-60`.
- `lib/agentcore-agui-client.ts` defines AG-UI event types and an `AgentCoreHttpAgent` wrapper consumed by `lib/use-workflow-stream.ts`.
- The workspace page uses `useCoAgent`, `useCopilotReadable`, `useCopilotAction`, and `useCopilotChat` from `@copilotkit/react-core` (correct CopilotKit 1.50+ surface).

**Gaps:**

- `AGENT_1_RUNTIME_URL`, `AGENT_2_RUNTIME_URL`, `AGENT_3_RUNTIME_URL` are referenced in `app/api/copilotkit/route.ts:43-58` but are **not** declared in the `nextjs-frontend` service of `agentcore-demo-test1-backend/docker-compose.yml`. The Next.js container therefore boots with `process.env.AGENT_*_RUNTIME_URL = undefined`, the `HttpAgent` constructor receives `url: undefined`, and any AG-UI call from the runtime route will fail at request time.
- The session header is hard-coded to `'default-session'` rather than per-workflow `workflow_id` (plan: *"Session ID = workflow_id (AgentCore microVM session isolation)"*). This breaks per-workflow agent state isolation.

So AGUI is installed, but the demo currently cannot actually reach an AgentCore agent through the CopilotKit runtime route without further env wiring, and is using a single shared session.

---

## 4. Frontend Architecture Deviations

### 4.1 CopilotKit provider scope

- **Plan (PHASE_PLAN §Phase 5 / Files):** `app/workspace/[wfId]/layout.tsx` — CopilotKit provider scoped here. Root layout has NO CopilotKit.
- **Actual:** `app/layout.tsx` wraps the entire app in `<CopilotKitProvider>` which itself renders `<CopilotSidebar>` (`components/CopilotKitProvider.tsx:20-37`). There is no per-workflow layout file.

Consequence: every page (landing, metrics) runs the chat sidebar and the runtime route, and the chat has no per-workflow `wfId` context.

### 4.2 Workspace layout

- **Plan:** 3-pane — chat (left), canvas/BRD preview (centre), workflow status (right).
- **Actual:** Header + mini metrics + steps progress + a **2-pane** grid (Generative UI cards on the left, AG-UI event stream on the right) + optional right sidebar that just shows the `useCopilotChat` message log. No persistent chat input is visible in the workspace.

### 4.3 Missing or relocated components

| Planned (Phase 5 Files) | Status |
|---|---|
| `components/chat/ChatPanel.tsx` | Missing |
| `components/canvas/BrdPreview.tsx` | Replaced by `components/DraftPreviewCard.tsx` (different path/contract) |
| `components/canvas/ReviewPanel.tsx` | Replaced by `components/ReviewReportCard.tsx` |
| `components/hitl/ClarificationCard.tsx` | Lives at `components/ClarificationQuestionCard.tsx` |
| `components/hitl/ApprovalCard.tsx` | Lives at `components/ApprovalCard.tsx` |
| `hooks/useAgent.ts` | Missing (uses `useCoAgent` directly in page) |
| `hooks/useWorkflowStream.ts` | Lives at `lib/use-workflow-stream.ts` |
| `lib/api/workflows.ts`, `lib/api/uploads.ts`, `lib/auth/mockAuth.ts` | Missing — calls are inlined in `app/page.tsx` and `components/chat/AudioInput/AudioInput.tsx` |

This is mostly cosmetic, but it makes the codebase deviate noticeably from the cross-referenced Deliverable 8 layout.

---

## 5. Backend Deviations

### 5.1 Stub upload endpoint

`app/main.py:64-73` returns:
```python
return {"s3_uri": f"s3://{settings.audio_bucket}/{key}", "content_type": content_type or "audio/webm"}
```
without reading `file` or calling boto3. Phase 3 spec required content-type validation, 50 MB cap, and an actual `putobject` with KMS encryption. The richer endpoint planned at `app/api/uploads_routes.py` does not exist.

### 5.2 Workflow start lenient on missing audio

`StartWorkflowRequest` allows `audio_s3_uri: str | None = None` and substitutes a placeholder URI. The plan's contract (Phase 4 / D11) requires `audio_s3_uri` to be set so the transcriber can actually fetch a recording. This is the backend half of the audio-blocker bug.

### 5.3 Signal endpoint paths

- Plan: `POST /api/v1/workflows/{id}/signal/clarification`, `…/signal/approval`.
- Actual: `POST /api/v1/workflows/{id}/clarification`, `…/approval`.

The frontend SSE client calls the actual paths, so the demo is internally consistent, but it diverges from `PHASE_PLAN` and `Deliverable_4_Temporal_Operations_Guide.md`.

### 5.4 Agents have Dockerfiles

Plan (Phase 1 / Exit Gate): *"All 3 agents deployed via S3 ZIP — no Dockerfile, no ECR."* Each of `agents/agent_1_transcriber`, `agents/agent_2_drafter`, `agents/agent_3_reviewer` ships a `Dockerfile`. The `.bedrock_agentcore.yaml` files exist alongside, so the ZIP path may still be operational, but the Dockerfiles are a documentation/process drift.

### 5.5 Temporal server version

`docker-compose.yml:3` uses `temporalio/auto-setup:1.29`; PCD/PHASE_PLAN specify `1.22`. Likely harmless but worth aligning.

### 5.6 Docker-compose missing planned environment

The `nextjs-frontend` service does **not** pass `AGENT_1_RUNTIME_URL`, `AGENT_2_RUNTIME_URL`, `AGENT_3_RUNTIME_URL`, `NEXT_PUBLIC_BACKEND_URL`, or `NEXT_PUBLIC_COPILOT_RUNTIME_URL` from `.env`. They are only baked in as build-args for two of them (`docker-compose.yml:86-88`), which means runtime resolution of agent URLs fails.

---

## 6. What is Actually Working

To be balanced, the following plan items are present and correct:

- All 3 agent skeletons exist with OTel setup, payload builder, and Strands/LangGraph nodes (`agents/agent_1_transcriber`, `agent_2_drafter`, `agent_3_reviewer`).
- Temporal `BrdFromAudioWorkflow` is implemented with the canonical activity wrapper, HITL signals (clarification + approval), self-correction cap, and evidence-pack persistence (`app/temporal/workflows.py`, `activities.py`).
- Evidence pack persistence (init / append-output / append-HITL / finalize) is wired to RDS.
- SSE workflow stream endpoint exists (`GET /api/v1/workflows/{id}/stream`).
- AG-UI / CopilotKit packages are installed and the runtime route compiles.
- ADOT collector and CloudWatch emission paths are in place (with a dry-run telemetry emitter).

---

## 7. Recommended Remediation Order

1. **Unblock audio (highest priority)**
   1. Replace the stub `POST /api/v1/uploads/audio` with the real Phase 3 implementation (read upload, validate content-type and size, `boto3 s3.put_object` to `AUDIO_BUCKET`).
   2. Move CopilotKit provider out of root layout into `app/workspace/[wfId]/layout.tsx`.
   3. Add a chat / control pane to the workspace page that renders `<AudioInput />`. Or: render `<AudioInput />` on the landing page and only redirect to `/workspace/[wfId]` after `POST /api/v1/uploads/audio` → `POST /api/v1/workflows` returns the new `workflow_id`.
   4. Remove the placeholder-URI fallback in `start_workflow` so a missing `audio_s3_uri` returns 400.
2. **Restore AG-UI runtime wiring**
   1. Add `AGENT_1_RUNTIME_URL`, `AGENT_2_RUNTIME_URL`, `AGENT_3_RUNTIME_URL` (and the `NEXT_PUBLIC_*` vars) to the `nextjs-frontend` service in `docker-compose.yml`, sourced from `.env`.
   2. Replace the hard-coded `'default-session'` header with the active `workflow_id` so AgentCore session isolation works (plan: session_id = workflow_id).
3. **Align signal endpoints and Temporal image** with the published plan (`signal/clarification`, `signal/approval`; `temporalio/auto-setup:1.22`) or update the docs to match the new paths and image versions if those are intentional.
4. **Optional cleanup** of agent Dockerfiles (delete or document why they stay) and reorganise frontend components to match the Deliverable 8 paths.

---

## 8. Files of Interest (for the fix)

- Frontend
  - `app/layout.tsx` (move CopilotKit out)
  - `app/workspace/[wfId]/layout.tsx` (create — mount CopilotKit here)
  - `app/workspace/[wfId]/page.tsx` (host the chat / `AudioInput`)
  - `app/page.tsx:126-145` (start-workflow handler — change contract to send `audio_s3_uri`)
  - `components/chat/AudioInput/AudioInput.tsx` (already correct; just needs to be mounted)
  - `app/api/copilotkit/route.ts` (replace `'default-session'` with the workflow id)
- Backend
  - `app/main.py:64-73` (replace stub upload with real S3 put)
  - `app/api/workflows_routes.py:106-132` (drop the placeholder URI fallback)
- Infra
  - `agentcore-demo-test1-backend/docker-compose.yml` (add `AGENT_*_RUNTIME_URL` + `NEXT_PUBLIC_*` env to `nextjs-frontend`; pin Temporal `1.22`)

---

*End of report.*
