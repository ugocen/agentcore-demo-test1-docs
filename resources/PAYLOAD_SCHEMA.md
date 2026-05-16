# PAYLOAD_SCHEMA — Canonical 8-Block Reference

**Document ID:** DELIV-PAYLOAD-SCHEMA
**Version:** 1.0
**Date:** 2026-05-15
**Status:** BASELINE — referenced by every Prompt and Deliverable
**Upstream:** `AI_SDLC_Agent_Development_Guidelines_v5.docx` Sections 3.4, 7, 8, 9, 16, Appendix B
**Companion:** `Deliverable_0_PROJECT_CONTEXT.md` (PCD)

This document is the single canonical reference for:

1. The **agent request schema** (inbound `/invoke`)
2. The **8-block response schema** (outbound `/invoke` return value)
3. The **identifier hierarchy** with strict propagation rules
4. The **error and status code dictionaries**
5. **Worked examples** for each of the three demo agents

If any Prompt or Deliverable disagrees with this document, this document wins. Validation logic in `agents/common/payload_builder.py` enforces every rule below.

---

## Table of Contents

1. [Identifier Hierarchy](#1-identifier-hierarchy)
2. [Inbound Request Schema](#2-inbound-request-schema)
3. [Outbound 8-Block Response Schema](#3-outbound-8-block-response-schema)
4. [Block-by-Block Field Reference](#4-block-by-block-field-reference)
5. [Status Code Dictionary](#5-status-code-dictionary)
6. [Error Code Dictionary](#6-error-code-dictionary)
7. [Step Type Enum](#7-step-type-enum)
8. [HITL Re-Invocation Contract](#8-hitl-re-invocation-contract)
9. [Validation Rules](#9-validation-rules)
10. [Worked Examples per Agent](#10-worked-examples-per-agent)
11. [Naming Conventions](#11-naming-conventions)

---

## 1. Identifier Hierarchy

Per Guideline §3.4. Identifiers are layered; each layer creates specific ones and is forbidden from inventing identifiers a higher layer should have provided.

| Identifier | Created by | Example | Scope |
|---|---|---|---|
| `trace_id` | Platform (FastAPI workflow-start handler) | `TRC-9981` | End-to-end across all layers (W3C `traceparent`) |
| `workflow_template_id` | Platform (Catalog publish) | `brd-from-audio-v1` | Bounded; safe for low-cardinality dimension |
| `workflow_id` | Platform (workflow start) | `wf-2026-05-15-001` | One orchestration execution; high-cardinality |
| `agent_run_id` | Orchestrator (per Activity) | `run-001` | One agent invocation; high-cardinality |
| `step_id` | Orchestrator (per logical step) | `step-001` | Smallest atomic unit; high-cardinality |
| `step_sequence` | Orchestrator | `1`, `2`, `3`... | Monotonically increasing within workflow |
| `parent_run_id` | Orchestrator | `wf-2026-05-15-001` | Parent context (usually = `workflow_id`) |
| `task_id` | Orchestrator (per request) | `TASK-BRD-001` | A2A protocol task identifier |
| `agent_id` | Agent (own identity) | `agent_1_transcriber` | Agent's catalog identifier |
| `agent_version` | Agent (own version) | `1.0.0` | Semver of the agent code |
| `actor_id` | Platform (= `requested_by.user_id`) | `demo-user-001` | Used for AgentCore Memory isolation |
| `session_id` | Platform (= `workflow_id`) | `sess-2026-05-15-001` | Used for AgentCore MicroVM isolation |

**Propagation rules:**

1. **Agents NEVER invent** `trace_id`, `workflow_template_id`, `workflow_id`, `agent_run_id`, `step_id`, `parent_run_id`, `task_id`. They receive these from the inbound request and echo them in Block 1 (Status).
2. **W3C `traceparent`** carries `trace_id` through every HTTP, gRPC, and boto3 call. The value of `trace_id` in the payload MUST equal the value encoded in `traceparent`.
3. **Workflow Template ID is mandatory in every aggregated query.** Block 1 echoes it because dashboards group by template.
4. **High-cardinality identifiers** (`trace_id`, `workflow_id`, `agent_run_id`, `step_id`, `user_id`) appear at the EMF root, NEVER in `CloudWatchMetrics.Dimensions`. The ADOT Collector enforces this.

---

## 2. Inbound Request Schema

This is exactly what the platform passes to an agent's `POST /invoke`. The agent's role is to read these fields, populate them in the response Block 1, propagate `trace_id` on every outbound call, and use `actor_id` + `session_id` for memory isolation.

```json
{
  "trace_id": "TRC-9981",
  "workflow_template_id": "brd-from-audio-v1",
  "workflow_id": "wf-2026-05-15-001",
  "agent_run_id": "run-001",
  "step_id": "step-001",
  "step_sequence": 1,
  "parent_run_id": "wf-2026-05-15-001",
  "task_id": "TASK-BRD-001",
  "agent_id": "agent_1_transcriber",
  "agent_version": "1.0.0",

  "requested_by": {
    "user_id": "demo-user-001",
    "email": "demo@local",
    "roles": ["business_analyst"],
    "group": {
      "group_id": "sap-mm",
      "group_type": "module",
      "group_name": "SAP MM"
    },
    "project": {
      "project_id": "proj-demo",
      "project_name": "Demo BRD",
      "system": "DEMO",
      "sector": "MT",
      "vertical": "ERP"
    }
  },

  "execution_context": {
    "environment": "dev",
    "data_classification": "INTERNAL"
  },

  "task": {
    "task_type": "audio_transcription",
    "complexity_tier": "simple",
    "instruction": "Transcribe the meeting audio.",
    "output_format": "json",
    "template_id": null
  },

  "inputs": {
    "text": null,
    "artifact_refs": ["s3://agentcore-demo-test1-audio-uploads-<acct>/uploads/2026-05-15/meeting.mp3"],
    "metadata": {}
  },

  "execution_options": {
    "dry_run": false,
    "require_human_approval": true,
    "max_attempts": 3,
    "timeout_seconds": 900,
    "quality_threshold": 0.85
  },

  "actor_id": "demo-user-001",
  "session_id": "sess-2026-05-15-001",

  "human_input": null,
  "human_approved": null
}
```

**Field notes:**

- `execution_context.environment` ∈ `pre-dev | dev | qa | prod`.
- `execution_context.data_classification` ∈ `PUBLIC | INTERNAL | CONFIDENTIAL | RESTRICTED`.
- `task.task_type` for the demo: `audio_transcription` (Agent 1), `brd_generation` (Agent 2), `brd_review` (Agent 3).
- `task.complexity_tier` ∈ `simple | medium | complex | very_complex`. Drives Block 4 `manual_baseline_hours` lookup.
- `requested_by.project.sector` and `vertical` are controlled enums in production (governed by the Catalog). For the demo: `sector ∈ {MT, IM, CBT}`, `vertical ∈ {ERP, Non-ERP, Data, MLL, Other}`.
- `human_input` and `human_approved` are populated only on re-invocations after a HITL response. See §8.

---

## 3. Outbound 8-Block Response Schema

Every `/invoke` returns ALL 8 blocks. No partial payloads. Fields the agent genuinely cannot fill use the string `"Unavailable"` (or `null` where typed; this document calls out the right choice per field).

Top-level shape:

```json
{
  "status":    { ... Block 1 ... },
  "resources": { ... Block 2 ... },
  "timing":    { ... Block 3 ... },
  "financial": { ... Block 4 ... },
  "artifacts": [ ... Block 5 array ... ],
  "quality":   { ... Block 6 ... },
  "tool_calls":{ "calls": [...], "tool_summary": {...} },
  "risk":      { ... Block 8 ... }
}
```

**Order does not matter** for JSON parsing, but `agents/common/payload_builder.py` always emits them in the order above for visual consistency in CloudWatch.

---

## 4. Block-by-Block Field Reference

### Block 1 — Status

**Purpose:** Outcome record. Always carries the identifier echo so downstream consumers can correlate without external joins.

```json
{
  "status": {
    "code": "success",
    "http_status": 200,
    "message": "Transcription completed successfully.",

    "trace_id": "TRC-9981",
    "workflow_template_id": "brd-from-audio-v1",
    "workflow_id": "wf-2026-05-15-001",
    "agent_run_id": "run-001",
    "step_id": "step-001",

    "error_code": null,
    "error_category": "none",
    "retryable": false,

    "custom_metadata": {
      "transcribe_job_name": "agentcore-demo-test1-wf-2026-05-15-001",
      "language_detected": "en-US"
    }
  }
}
```

**Required (every agent, every call):** `code`, `http_status`, `message`, `trace_id`, `workflow_template_id`, `workflow_id`, `agent_run_id`, `step_id`.
**Failure case adds:** `error_code` (string), `error_category` (enum, §6), `retryable` (bool).
**Optional:** `custom_metadata` (≤4 KB free-form dict).

### Block 2 — Resources

**Purpose:** Cost reporting. The agent declares its model via `cost_reporting_model` and populates the matching sub-object.

**`cost_reporting_model` ∈ `token_based | vendor_cost | fixed_subscription | both`.**

**`token_based` example (used by Drafter and Reviewer, both call Bedrock):**

```json
{
  "resources": {
    "cost_reporting_model": "token_based",
    "token_based": {
      "model_used": "anthropic.claude-sonnet-4-6",
      "tokens": {
        "input": 4200,
        "output": 8100,
        "total_outer": 12300
      },
      "internal_llm_calls": 3,
      "total_tokens_all_calls": 26600
    }
  }
}
```

`tokens.total_outer` = what an outside observer would see at the request/response boundary.
`total_tokens_all_calls` = sum across every internal LLM call (initial → self-correct → finalize). These two numbers ARE typically different and BOTH must be reported (Guideline §8.3.6).

**`vendor_cost` example (used by Transcriber for AWS Transcribe):**

```json
{
  "resources": {
    "cost_reporting_model": "vendor_cost",
    "vendor_cost": {
      "unit": "audio_minutes",
      "amount": 3.0,
      "pricing_model": "per_minute",
      "usd_equivalent": 0.072
    }
  }
}
```

`usd_equivalent` is the normalized USD figure; downstream cost roll-ups read it without knowing the vendor unit.

**`both` example (advanced; not used in demo):**

```json
{
  "resources": {
    "cost_reporting_model": "both",
    "token_based": { ... },
    "vendor_cost": { ... }
  }
}
```

### Block 3 — Timing

```json
{
  "timing": {
    "total_elapsed_ms": 408000,
    "queue_duration_ms": 3000,
    "agent_active_ms": 285000,
    "waiting_for_tools_ms": 98000,
    "waiting_for_human_ms": 0,
    "timing_breakdown": {
      "reasoning_ms": 120000,
      "tool_calls_ms": 98000,
      "formatting_ms": 52000,
      "overhead_ms": 15000
    },
    "critical_path": "reasoning (42%) → tool_calls (34%) → formatting (18%) → overhead (6%)"
  }
}
```

- `total_elapsed_ms` = wall-clock from request received to response returned.
- `waiting_for_human_ms` is non-zero only for HITL steps. Sums to workflow-level human-wait aggregate.
- `timing_breakdown` is optional but recommended; if omitted set to `"Unavailable"`.

### Block 4 — Financial

```json
{
  "financial": {
    "task_type": "audio_transcription",
    "complexity_tier": "simple",
    "manual_baseline_hours": 0.5,
    "agent_active_hours": 0.08,
    "human_review_hours_estimated": null,
    "agent_cost_usd": 0.072,
    "retry_cost_usd": 0.0,
    "currency": "usd"
  }
}
```

- `manual_baseline_hours` comes from a `task_type × complexity_tier` lookup table in `agents/common/baselines.py`.
- `human_review_hours_estimated` is `null` until the human loop closes; the platform fills it in.
- `agent_cost_usd` = either `token_based.tokens` × Bedrock unit price, or `vendor_cost.usd_equivalent`.
- `currency` is always `"usd"` (lowercase).

### Block 5 — Artifacts

Array (possibly empty). Each entry is one output the agent produced.

```json
{
  "artifacts": [
    {
      "artifact_id": "art-brd-wf001-v1.1",
      "artifact_type": "design_document",
      "artifact_subtype": "functional_spec",
      "artifact_format": "md",
      "artifact_size_bytes": 24800,
      "artifact_title": "Demo BRD — Meeting 2026-05-15",
      "content_ref": "s3://agentcore-demo-test1-artifacts-<acct>/brd/wf001/v1.1.md",
      "content_hash": "sha256:9f8a3c2e1b...",
      "versioning": {
        "version": "1.1",
        "version_sequence": 2,
        "is_new_artifact": false,
        "previous_version": "1.0",
        "previous_version_ref": "s3://agentcore-demo-test1-artifacts-<acct>/brd/wf001/v1.0.md",
        "change_summary": "Added vendor approval role per HITL clarification."
      },
      "timestamps": {
        "created_at": "2026-05-15T14:06:48Z",
        "generation_started_at": "2026-05-15T13:59:48Z",
        "generation_duration_ms": 408000
      },
      "template_used": "brd-template-v1",
      "template_adherence_pct": 96,
      "sections_completed": 12,
      "sections_total": 12,
      "information_classification": "INTERNAL",
      "retention_policy_ref": "demo-retention-design-document"
    }
  ]
}
```

**`artifact_type` controlled enum:** `user_story | design_document | code | migration_code | test_script | quality_document | break_fix | other`.

For demo:
- Transcriber → `artifact_type: "design_document"`, `artifact_subtype: "transcript"`, `artifact_format: "json"`
- Drafter → `artifact_type: "design_document"`, `artifact_subtype: "functional_spec"`, `artifact_format: "md"`
- Reviewer → `artifact_type: "quality_document"`, `artifact_subtype: "review_report"`, `artifact_format: "md"`

**`versioning.is_new_artifact`:** `true` on initial creation; `false` on regenerations (then `previous_version_ref` is mandatory).

**`retention_policy_ref`:** points to a policy ID, not a raw duration. Demo uses `demo-retention-{type}` placeholders.

### Block 6 — Quality

```json
{
  "quality": {
    "confidence": 0.94,
    "initial_attempt_success": false,
    "total_attempts": 3,
    "self_corrections": 1,
    "quality_concerns": ["Custom table field mapping resolved on 2nd attempt"],
    "unsupported_claims": [],
    "completeness_assessment": "all_sections_complete"
  }
}
```

- `total_attempts` counts initial generation + self-correction attempts. Capped at 3 (D8) before mandatory HITL escalation.
- `self_corrections` = `total_attempts - 1` (the first try is not a correction).
- `quality_concerns` is a list of free-text notes (kept short; for audit, not telemetry).
- `unsupported_claims` is a list of claims the agent could not back up; usually empty for happy paths.
- `completeness_assessment` ∈ `all_sections_complete | mostly_complete | partial | minimal`.

### Block 7 — Tool Calls

```json
{
  "tool_calls": {
    "calls": [
      {
        "sequence": 1,
        "mcp_server": "aws-transcribe",
        "tool": "start_transcription_job",
        "duration_ms": 340,
        "downstream_system_time_ms": 280,
        "mcp_overhead_ms": 60,
        "status": "success",
        "data_classification": "INTERNAL",
        "result_summary": {"job_name": "...", "size_bytes": 4200}
      }
    ],
    "tool_summary": {
      "total_tool_calls": 1,
      "total_tool_duration_ms": 340,
      "tools_as_pct_of_elapsed": 28
    }
  }
}
```

**Demo `mcp_server` values (not real MCP servers; conceptual labels):**
- `aws-transcribe` — Transcriber agent
- `aws-bedrock` — Drafter and Reviewer (LLM calls)
- `aws-s3` — Any agent reading/writing artifacts directly

In a real MCP-enabled environment, `mcp_server` would name the actual MCP server (`mcp-sap-rfc`, `mcp-jira`, etc.).

### Block 8 — Risk

```json
{
  "risk": {
    "pii_detected": false,
    "pii_filtered_count": 0,
    "secrets_detected": false,
    "policy_violations": 0,
    "sensitivity_compliant": true,
    "missing_inputs": [],
    "unsupported_claims": [],
    "compliance_checks": [
      {"framework": "SOX", "check": "audit_trail_complete", "passed": true}
    ]
  }
}
```

- `pii_detected` is the boolean result of the agent's own PII scan (regex + optional vendor service). `pii_filtered_count` records the number of redactions applied.
- `secrets_detected` checks for AWS keys, API tokens, etc. in inputs/outputs.
- `policy_violations` is the count of policy hits; `compliance_checks` enumerates them.
- For the demo, Agent 3 (Reviewer) is the primary populator of risk findings; Agents 1 and 2 still emit Block 8 with `pii_detected: false, pii_filtered_count: 0, ...` after a quick scan.

---

## 5. Status Code Dictionary

`status.code` enum (Guideline App B):

| code | Meaning | Typical next step |
|---|---|---|
| `success` | Task completed successfully | Workflow proceeds to next step |
| `partial_success` | Output produced with warnings | Workflow may proceed; warnings logged |
| `partial_failure` | Some sub-steps failed; degraded result | Workflow may escalate or proceed |
| `failed` | Task failed completely | Workflow either retries or fails |
| `requires_human_review` | HITL required | Workflow opens `hitl_review`/`hitl_approval` step and waits |
| `blocked` | Permission/dependency/policy block | Workflow surfaces a blocker; no retry helpful |
| `cancelled` | Cancelled (signal or orchestrator decision) | Workflow terminates with CANCELLED outcome |
| `timeout` | Task exceeded `execution_options.timeout_seconds` | Workflow retries per RetryPolicy or fails |

`status.http_status` should match the semantic of `code`: 200 for success, 206 for partial_*, 400 for input_validation failures, 401/403 for authorization failures, 422 for quality_gate, 500 for unknown.

---

## 6. Error Code Dictionary

`status.error_category` enum (Guideline App B):

| error_category | Example error_code values | retryable? |
|---|---|---|
| `input_validation` | `MISSING_REQUIRED_FIELD`, `INVALID_ARTIFACT_REF` | No |
| `authorization` | `USER_NOT_AUTHORIZED`, `TOOL_SCOPE_DENIED` | No |
| `tool_failure` | `MCP_SERVER_TIMEOUT`, `TRANSCRIBE_JOB_FAILED`, `BEDROCK_INVOKE_FAILED` | Yes (transient) |
| `llm_failure` | `MODEL_RATE_LIMIT`, `MODEL_TIMEOUT`, `MODEL_CONTEXT_OVERFLOW` | Yes |
| `quality_gate` | `CONFIDENCE_BELOW_THRESHOLD`, `UNSUPPORTED_CLAIMS_FOUND`, `COMPLETENESS_BELOW_THRESHOLD` | No after D8 cap; then HITL |
| `policy_violation` | `PII_POLICY_BLOCK`, `SECRET_DETECTED`, `SENSITIVITY_VIOLATION` | No |
| `dependency_unavailable` | `BEDROCK_UNAVAILABLE`, `S3_UNAVAILABLE`, `RDS_UNAVAILABLE` | Yes |
| `timeout` | `AGENT_TIMEOUT`, `WORKFLOW_TIMEOUT` | Workflow-dependent |
| `claim_check_failure` | `CLAIM_CHECK_UPLOAD_FAILED`, `CLAIM_CHECK_HASH_MISMATCH`, `CLAIM_CHECK_NOT_FOUND` | Yes (transient) |
| `evidence_pack_input` | `EVIDENCE_PACK_INPUT_INCOMPLETE` | No; escalate to orchestrator |
| `none` | (success path; no error) | n/a |
| `unknown` | `UNCLASSIFIED_ERROR` | Manual triage |

Always set `error_category` even on success (`"none"`). Always set `retryable` on failures.

---

## 7. Step Type Enum

Per Guideline §16. Used by the orchestrator when logging step metadata to CloudWatch.

| step_type | When emitted |
|---|---|
| `agent_action` | Pure agent invocation (no HITL) |
| `hitl_question` | Agent asks for clarification |
| `hitl_review` | Reviewer-style review request (user reviews a draft) |
| `hitl_approval` | Mandatory approval gate before write/publish |
| `hitl_branch_decision` | User picks among several paths |
| `tool_call` | A tool invocation outside an agent (rare; usually nested inside agent_action) |
| `claim_check_io` | Large payload upload/download to/from claim-check bucket |
| `revise` | Self-correction attempt (D8); new step_id per attempt; same agent_run_id when invocation is logically continuous |

---

## 8. HITL Re-Invocation Contract

When the workflow re-invokes an agent after a human responds, the inbound request payload includes two additional fields:

```json
{
  "human_input": {
    "responder_user_id": "demo-user-001",
    "responded_at": "2026-05-15T14:12:30Z",
    "decision": "approve | revise | reject | answer",
    "rationale": "Free text from the reviewer",
    "structured_response": null,
    "previous_step_id": "step-004"
  },
  "human_approved": true
}
```

- `decision`:
  - For `hitl_question`: `"answer"` (with `rationale` containing the answer text).
  - For `hitl_review`: `"approve" | "revise" | "reject"`.
  - For `hitl_approval`: `"approve" | "reject"`.
- `human_approved` is a convenience bool: `true` iff `decision == "approve"`.
- `structured_response` is reserved for future forms with structured fields (e.g., a marked-up document); null for now.

**Agent behavior on re-invocation:**
1. Read `human_input.decision`.
2. If `decision == "answer"`: continue reasoning using `rationale` as the answer; do NOT ask the same question again.
3. If `decision == "revise"`: regenerate with `rationale` as feedback; bump `version_sequence`; new `step_id` with `step_type=revise`.
4. If `decision == "reject"`: return Status `code=failed`, `error_category=quality_gate`, `retryable=false`; the workflow will terminate with REJECTED.
5. If `decision == "approve"`: return Status `code=success` and finalize the artifact.

---

## 9. Validation Rules

Implemented in `agents/common/payload_builder.py` `validate_payload(payload, tier="first_party")`:

| Rule | Failure error |
|---|---|
| All 8 top-level keys present | `EVIDENCE_PACK_INPUT_INCOMPLETE` — `missing_blocks: [...]` |
| `status.code` in the Status Code enum | `INVALID_STATUS_CODE` |
| `status.trace_id`, `workflow_id`, `workflow_template_id`, `agent_run_id`, `step_id` all non-null | `MISSING_REQUIRED_FIELD` — `field: status.<name>` |
| `resources.cost_reporting_model` in {token_based, vendor_cost, fixed_subscription, both} | `INVALID_COST_REPORTING_MODEL` |
| Matching sub-object exists for declared cost_reporting_model | `MISSING_RESOURCES_SUBOBJECT` |
| `financial.currency == "usd"` | `INVALID_CURRENCY` (demo only allows USD) |
| If `artifacts` non-empty: each entry has `content_ref` (s3://), `content_hash` (sha256:), `versioning.version` | `INVALID_ARTIFACT` |
| `quality.total_attempts <= 3` (D8) | `SELF_CORRECTION_CAP_EXCEEDED` — should never trigger; orchestrator escalates to HITL first |
| `tool_calls.calls` is a list (possibly empty); `tool_calls.tool_summary` is an object | `INVALID_TOOL_CALLS_SHAPE` |
| `risk` has all 8 sub-fields (pii_detected, pii_filtered_count, secrets_detected, policy_violations, sensitivity_compliant, missing_inputs, unsupported_claims, compliance_checks) | `INCOMPLETE_RISK_BLOCK` |

Validation runs:
- In the agent before returning the response (defense in depth)
- In `app/temporal/activities/invoke_agent_runtime.py` immediately after receiving the agent's response
- In E2E tests (Phase 6) against persisted Evidence Packs

A validation failure produces an error Status block:

```json
{
  "status": {
    "code": "partial_failure",
    "http_status": 422,
    "error_code": "EVIDENCE_PACK_INPUT_INCOMPLETE",
    "error_category": "evidence_pack_input",
    "retryable": false,
    "message": "Missing blocks: ['risk']",
    "trace_id": "...", "workflow_template_id": "...", "workflow_id": "...",
    "agent_run_id": "...", "step_id": "..."
  },
  ...
}
```

---

## 10. Worked Examples per Agent

### 10.1 Agent 1 — Transcriber (Strands)

**Inbound `task`:**
```json
{
  "task_type": "audio_transcription",
  "complexity_tier": "simple",
  "instruction": "Transcribe the audio with auto language detection.",
  "output_format": "json",
  "template_id": null
}
```

**Outbound 8-block (success):**
```json
{
  "status": {
    "code": "success", "http_status": 200,
    "message": "Transcribed 3 segments in 12 seconds.",
    "trace_id": "TRC-9981", "workflow_template_id": "brd-from-audio-v1",
    "workflow_id": "wf-2026-05-15-001", "agent_run_id": "run-001", "step_id": "step-001",
    "error_code": null, "error_category": "none", "retryable": false,
    "custom_metadata": {
      "transcribe_job_name": "agentcore-demo-test1-wf-2026-05-15-001",
      "language_detected": "en-US",
      "segments_count": 47
    }
  },
  "resources": {
    "cost_reporting_model": "vendor_cost",
    "vendor_cost": {
      "unit": "audio_minutes", "amount": 3.0,
      "pricing_model": "per_minute", "usd_equivalent": 0.072
    }
  },
  "timing": {
    "total_elapsed_ms": 12500,
    "queue_duration_ms": 1200,
    "agent_active_ms": 11300,
    "waiting_for_tools_ms": 9800,
    "waiting_for_human_ms": 0,
    "timing_breakdown": {"tool_calls_ms": 9800, "formatting_ms": 800, "overhead_ms": 700},
    "critical_path": "tool_calls (87%) → formatting (7%) → overhead (6%)"
  },
  "financial": {
    "task_type": "audio_transcription", "complexity_tier": "simple",
    "manual_baseline_hours": 0.5, "agent_active_hours": 0.003,
    "human_review_hours_estimated": null,
    "agent_cost_usd": 0.072, "retry_cost_usd": 0.0, "currency": "usd"
  },
  "artifacts": [
    {
      "artifact_id": "art-transcript-wf001-v1.0",
      "artifact_type": "design_document",
      "artifact_subtype": "transcript",
      "artifact_format": "json",
      "artifact_size_bytes": 8420,
      "artifact_title": "Transcript — wf-2026-05-15-001",
      "content_ref": "s3://agentcore-demo-test1-artifacts-<acct>/transcripts/wf001/v1.0.json",
      "content_hash": "sha256:1a2b3c...",
      "versioning": {
        "version": "1.0", "version_sequence": 1, "is_new_artifact": true,
        "previous_version": null, "previous_version_ref": null,
        "change_summary": "Initial transcription"
      },
      "timestamps": {
        "created_at": "2026-05-15T14:00:12Z",
        "generation_started_at": "2026-05-15T13:59:59Z",
        "generation_duration_ms": 12500
      },
      "template_used": null, "template_adherence_pct": "Unavailable",
      "sections_completed": "Unavailable", "sections_total": "Unavailable",
      "information_classification": "INTERNAL",
      "retention_policy_ref": "demo-retention-transcript"
    }
  ],
  "quality": {
    "confidence": 0.94,
    "initial_attempt_success": true,
    "total_attempts": 1,
    "self_corrections": 0,
    "quality_concerns": [],
    "unsupported_claims": [],
    "completeness_assessment": "all_sections_complete"
  },
  "tool_calls": {
    "calls": [
      {
        "sequence": 1, "mcp_server": "aws-transcribe", "tool": "start_transcription_job",
        "duration_ms": 9800, "downstream_system_time_ms": 9600, "mcp_overhead_ms": 200,
        "status": "success", "data_classification": "INTERNAL",
        "result_summary": {"job_name": "agentcore-demo-test1-wf-2026-05-15-001", "duration_seconds": 180}
      }
    ],
    "tool_summary": {"total_tool_calls": 1, "total_tool_duration_ms": 9800, "tools_as_pct_of_elapsed": 78}
  },
  "risk": {
    "pii_detected": false, "pii_filtered_count": 0, "secrets_detected": false,
    "policy_violations": 0, "sensitivity_compliant": true,
    "missing_inputs": [], "unsupported_claims": [],
    "compliance_checks": []
  }
}
```

### 10.2 Agent 2 — Drafter (Strands, with HITL)

**Inbound on initial call:**
```json
{
  "task": {
    "task_type": "brd_generation", "complexity_tier": "medium",
    "instruction": "Generate a BRD from the transcript.",
    "output_format": "md", "template_id": "brd-template-v1"
  },
  "inputs": {
    "artifact_refs": ["s3://.../transcripts/wf001/v1.0.json"]
  },
  "human_input": null
}
```

**Outbound — clarification needed:**
```json
{
  "status": {
    "code": "requires_human_review", "http_status": 202,
    "message": "Clarification required: vendor approval role unspecified.",
    "trace_id": "TRC-9981", "workflow_template_id": "brd-from-audio-v1",
    "workflow_id": "wf-2026-05-15-001", "agent_run_id": "run-002", "step_id": "step-002",
    "error_code": null, "error_category": "none", "retryable": false,
    "custom_metadata": {
      "review_summary": {
        "reason": "Section 3.2 mentions vendor approval but role unspecified",
        "agent_confidence": 0.62,
        "draft_decision": null,
        "supporting_artifacts": []
      }
    }
  },
  "resources": {
    "cost_reporting_model": "token_based",
    "token_based": {
      "model_used": "anthropic.claude-sonnet-4-6",
      "tokens": {"input": 2400, "output": 1200, "total_outer": 3600},
      "internal_llm_calls": 1,
      "total_tokens_all_calls": 3600
    }
  },
  "timing": {
    "total_elapsed_ms": 4200, "queue_duration_ms": 200, "agent_active_ms": 4000,
    "waiting_for_tools_ms": 3800, "waiting_for_human_ms": 0,
    "timing_breakdown": {"reasoning_ms": 200, "tool_calls_ms": 3800},
    "critical_path": "tool_calls (90%) → reasoning (5%)"
  },
  "financial": {
    "task_type": "brd_generation", "complexity_tier": "medium",
    "manual_baseline_hours": 16.0, "agent_active_hours": 0.001,
    "human_review_hours_estimated": null,
    "agent_cost_usd": 0.018, "retry_cost_usd": 0.0, "currency": "usd"
  },
  "artifacts": [],
  "quality": {
    "confidence": 0.62, "initial_attempt_success": false,
    "total_attempts": 1, "self_corrections": 0,
    "quality_concerns": ["Vendor approval role unspecified in source transcript"],
    "unsupported_claims": [],
    "completeness_assessment": "partial"
  },
  "tool_calls": {
    "calls": [
      {"sequence": 1, "mcp_server": "aws-bedrock", "tool": "invoke_model",
       "duration_ms": 3800, "downstream_system_time_ms": 3700, "mcp_overhead_ms": 100,
       "status": "success", "data_classification": "INTERNAL",
       "result_summary": {"tokens_in": 2400, "tokens_out": 1200}}
    ],
    "tool_summary": {"total_tool_calls": 1, "total_tool_duration_ms": 3800, "tools_as_pct_of_elapsed": 90}
  },
  "risk": {
    "pii_detected": false, "pii_filtered_count": 0, "secrets_detected": false,
    "policy_violations": 0, "sensitivity_compliant": true,
    "missing_inputs": ["vendor_approval_role"], "unsupported_claims": [],
    "compliance_checks": []
  }
}
```

The workflow then opens a `hitl_question` step, waits for the user's answer, and re-invokes Agent 2 with `human_input.decision = "answer"` and `human_input.rationale = "<user's answer>"`.

### 10.3 Agent 3 — Reviewer (LangGraph 4-node)

**Inbound:**
```json
{
  "task": {
    "task_type": "brd_review", "complexity_tier": "medium",
    "instruction": "Review the BRD for quality, PII, and policy compliance.",
    "output_format": "md", "template_id": "brd-review-template-v1"
  },
  "inputs": {
    "artifact_refs": ["s3://.../brd/wf001/v1.1.md"]
  }
}
```

**Outbound (after 4-node graph: analyze_quality → scan_pii → check_policy → generate_report):**

Status block returns `code: "requires_human_review"` because the workflow always emits a `hitl_approval` step after the Reviewer, even when the review is clean.

`risk.pii_detected` and `risk.policy_violations` are populated by the LangGraph nodes; the review report itself (with severity, suggestions) goes into `artifacts[]` as a `quality_document`.

### 10.4 Failure path example (Drafter, after 3 attempts)

Block 1 on the 4th attempt (cap exceeded — the orchestrator should never have called the agent again, but defensive failure path):

```json
{
  "status": {
    "code": "failed", "http_status": 422,
    "error_code": "SELF_CORRECTION_CAP_EXCEEDED",
    "error_category": "quality_gate",
    "retryable": false,
    "message": "Self-correction cap (3 attempts) exhausted; orchestrator should escalate.",
    "trace_id": "...", "workflow_template_id": "...", "workflow_id": "...",
    "agent_run_id": "...", "step_id": "..."
  },
  ...
  "quality": {"confidence": 0.71, "initial_attempt_success": false,
              "total_attempts": 3, "self_corrections": 2,
              "quality_concerns": ["Repeated CONFIDENCE_BELOW_THRESHOLD"],
              "unsupported_claims": [], "completeness_assessment": "partial"}
}
```

---

## 11. Naming Conventions

### 11.1 Filesystem & Code

| Concept | Convention | Example |
|---|---|---|
| Repo name | hyphen-case | `agentcore-demo-test1-backend` |
| Python directory | snake_case | `agents/agent_1_transcriber/` |
| Python file | snake_case | `payload_builder.py`, `invoke_agent_runtime.py` |
| Python class | PascalCase | `BrdFromAudioWorkflow`, `MetricsService` |
| Python function | snake_case | `build_payload`, `validate_payload` |
| TypeScript file | PascalCase for components, camelCase for hooks/lib | `ChatPanel.tsx`, `useMicRecorder.ts`, `mockAuth.ts` |
| TypeScript type | PascalCase | `WorkflowStatus`, `EightBlockPayload` |

### 11.2 Identifiers in API/Payload

| Concept | Convention | Example |
|---|---|---|
| `agent_id` | hyphen-case | `agent_1_transcriber` |
| `workflow_template_id` | hyphen-case | `brd-from-audio-v1` |
| `workflow_id` | `wf-` prefix + UUID/sequence | `wf-2026-05-15-001` |
| `trace_id` | `TRC-` prefix + W3C hex | `TRC-9981a2b3c4...` |
| `step_id` | `step-` prefix + sequence | `step-001` |
| `agent_run_id` | `run-` prefix + sequence | `run-001` |
| `task_id` | `TASK-<DOMAIN>-` prefix | `TASK-BRD-001` |
| `artifact_id` | `art-<type-prefix>-` | `art-brd-wf001-v1.1` |

### 11.3 Database

| Concept | Convention | Example |
|---|---|---|
| Database name | snake_case | `agentcore_demo_test1`, `temporal`, `temporal_visibility` |
| Table name | snake_case plural | `evidence_packs`, `workflow_runs` |
| Column name | snake_case | `agent_outputs`, `created_at` |
| Enum value | UPPER_SNAKE_CASE for state enums, snake_case for type enums | `BRDState.APPROVED`, `step_type='agent_action'` |

### 11.4 AWS Resources

| Resource | Convention | Example |
|---|---|---|
| S3 bucket | hyphen-case | `agentcore-demo-test1-audio-uploads-${ACCOUNT_ID}` |
| RDS DB instance | hyphen-case | `agentcore-demo-test1-db` |
| EC2 instance Name tag | hyphen-case | `agentcore-demo-test1-ec2` |
| IAM role | hyphen-case | `agentcore-demo-test1-ec2-role` |
| IAM policy | hyphen-case | `agentcore-demo-test1-ec2-policy` |
| CloudWatch log group | path-style, single shared group for agents | `/agentcore-demo-test1/agent-logs` (filter by `demo.agent_id` attribute) |
| CloudWatch metric namespace | PascalCase/PascalCase | `DemoSDLC/Agent` |
| OTel span attribute | dotted, demo-prefixed | `demo.workflow_id`, `demo.actor_id` |
| Project tag | `Project=agentcore-demo-test1` on every resource | — |

### 11.5 API Paths

All routes under `/api/v1/`. See PCD §3 D11 for the canonical list.

---

**END OF DOCUMENT**

*If a Prompt or Deliverable contradicts this schema, treat this schema as authoritative and file an issue.*
