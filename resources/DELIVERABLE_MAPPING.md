# Deliverable-to-Phase Mapping Document

**Version:** 2.0
**Date:** 2026-05-15
**Purpose:** Bridge the original deliverables (architecture/reference material) with the executable 8-phase plan (Prompts 00–07). Use to understand which deliverable feeds which phase.

> **Authoritative documents:** PCD (`Deliverable_0_PROJECT_CONTEXT.md`) for decisions and architecture; `PAYLOAD_SCHEMA.md` for the 8-block contract; `PHASE_PLAN.md` for phase sequencing; this file for cross-referencing.

---

## 1. Overview

The project uses an **8-phase**, checkpoint-driven build plan. Each phase has a dedicated, self-contained prompt (`Prompt_00_*.md` through `Prompt_07_*.md`). The original deliverables provide **reference material**: code skeletons, architectural rationale, operational walkthroughs.

**Hierarchy:**
1. **PCD** (`Deliverable_0`) — frozen decisions D1–D14, the architectural baseline.
2. **PAYLOAD_SCHEMA** — canonical 8-block contract; any conflict is resolved in its favor.
3. **PHASE_PLAN** — phase sequencing, gates, dependencies.
4. **Prompt_NN** — atomic, executable instructions for one phase.
5. **Deliverable_N** — deep-dive reference material for that phase.

If a Prompt or Deliverable contradicts PCD or PAYLOAD_SCHEMA, the Prompt/Deliverable is wrong.

---

## 2. Phase-to-Deliverable Mapping Table

| Phase | Prompt File | Primary Deliverables | Reference Deliverables | What to Read |
|---|---|---|---|---|
| 0 | `Prompt_00_Infra_Bootstrap.md` | `Deliverable_6_AWS_Operations_Guide.md` | PCD §2 (topology), PCD §5 (infra repo structure) | All AWS setup scripts; 2-AZ subnet group rules; IMDSv2 |
| 1 | `Prompt_01_AgentCore_Deployment.md` | `Deliverable_2_Reference_Strands_Agent_Code.md`, `Deliverable_6_AWS_Operations_Guide.md` §11 | PCD §6 (8-block), `PAYLOAD_SCHEMA.md` | Agent skeleton patterns; `BedrockAgentCoreApp` + 5 endpoints; S3 ZIP deploy |
| 2 | `Prompt_02_Agent_Telemetry.md` | `Deliverable_7_CloudWatch_Telemetry_Guide.md`, `PAYLOAD_SCHEMA.md` | PCD §10 (telemetry model), Guideline §9.5 | OTel boilerplate, `demo.*` attributes, ADOT config, governance filter |
| 3 | `Prompt_03_Backend_Foundation.md` | PCD §5 (backend repo structure), `Deliverable_4_Temporal_Operations_Guide.md` §1 | PCD §3 D10 (evidence_packs schema), D11 (API paths), D12 (DB creds) | FastAPI skeleton, SQLAlchemy 2.0, Alembic migration, docker-compose 6-service stack |
| 4 | `Prompt_04_Temporal_Workflow.md` | `Deliverable_4_Temporal_Operations_Guide.md` | PCD §11 (Workflow Streams Pattern A/B), `PAYLOAD_SCHEMA.md` §8 (HITL re-invocation) | Workflow definition, generic `invoke_agent_runtime` activity, signal handlers, Pattern A/B |
| 5 | `Prompt_05_Frontend.md` | `Deliverable_5_CopilotKit_AGUI_Guide.md`, `Deliverable_8_Frontend_Architecture_Guide.md` | PCD §3 D13 (audio input), Guideline §16 (HITL UX) | Next.js 14 App Router, CopilotKit provider scope, `useMicRecorder`, AG-UI client |
| 6 | `Prompt_06_E2E_Integration.md` | `PROMPT_TESTING_SECTIONS.md` | All previous Prompts + Deliverables | 11 sequential E2E tests, CloudWatch verification, Evidence Pack validation |
| 7 | `Prompt_07_Metrics_Dashboard.md` | `Deliverable_7_CloudWatch_Telemetry_Guide.md` (dashboard section) | PCD §3 D14, Guideline §9.6 (computed metrics) | `/metrics` page, 5 backend endpoints, Recharts, `get_metric_data` integration |

### Reading Guide

- **Primary Deliverables:** Contain the exact code, scripts, or instructions the phase prompt implements. These are the most important references.
- **Reference Deliverables:** Architectural context, decision rationale, operational background.
- **What to Read:** Sections most relevant to the phase.

---

## 3. Key Content Index per Deliverable

### Deliverable 0 — PCD (`Deliverable_0_PROJECT_CONTEXT.md`)

| Phase | Sections | Why it matters |
|---|---|---|
| 0 | §2 (topology), §5 (infra repo), §3 D11/D12 (paths, DB) | Confirms infra choices, repo layout, credentials strategy |
| 1 | §6 (8-block), §7 (identifiers), §8 (5 endpoints) | What every agent must implement |
| 2 | §10 (telemetry model), §3 D1 (`demo.*` prefix) | OTel attribute taxonomy |
| 3 | §5 (backend repo), §3 D10 (evidence_packs schema) | DB schema, repo structure |
| 4 | §11 (Workflow Streams), §3 D6 (Pattern A/B) | Streams strategy and feature flag |
| 5 | §3 D13 (audio input), §9 (HITL model), §12 (demo simplifications) | UX scope and HITL contract |
| 6 | §12 (demo simplifications matrix) | What is in scope vs production |
| 7 | §3 D14 (metric dashboard), §10 (cardinality discipline) | Dashboard scope and query patterns |
| All | §4 (non-negotiable rules), §3 D1–D14 | Cross-cutting rules |

### Deliverable 1 — Infrastructure Cost Report

| Phase | Sections | Why it matters |
|---|---|---|
| 0, 1 | Per-resource pricing, eu-central-1 vs us-east-1 | Budget reference and region comparison |
| — | Cost-saving scenarios | Optional: scale-down patterns |

### Deliverable 2 — Reference Strands/LangGraph Agent Code

| Phase | Sections | Why it matters |
|---|---|---|
| 1 | Agent skeletons (`BedrockAgentCoreApp` + `@app.entrypoint`) | Minimal agent structure |
| 2 | `common/payload_builder.py`, `common/otel_setup.py` patterns | Shared modules pattern |
| 2 | Agent 3 LangGraph 4-node skeleton | Review graph: analyze_quality, scan_pii, check_policy, generate_report |
| 4 | Agent invocation pattern (boto3 from activity) | How Temporal activities call agents |

### Deliverable 3 — Historical Implementation Prompts

**Status:** Superseded by Prompts 00–07. Read only if you need historical context on earlier design decisions.

### Deliverable 4 — Temporal Operations Guide

| Phase | Sections | Why it matters |
|---|---|---|
| 4 | All sections | Workflow patterns, activities, signals/updates/queries, **Pattern A vs Pattern B Streams**, determinism rules, retry policies, troubleshooting |

### Deliverable 5 — CopilotKit AG-UI Guide

| Phase | Sections | Why it matters |
|---|---|---|
| 5 | All sections | CopilotKit hooks (`useAgent`, `useCoAgent`, `useCopilotAction`), AG-UI HttpAgent, generative UI cards, dual-response HITL pattern |

### Deliverable 6 — AWS Operations Guide

| Phase | Sections | Why it matters |
|---|---|---|
| 0 | All sections | Step-by-step AWS setup from a blank account: VPC (2 AZs), S3 (us-east-1 quirk), RDS subnet group, IAM least-privilege, EC2 + Docker, AgentCore CLI |
| 1 | §11 (AgentCore deployment) | S3 ZIP deploy walkthrough |
| — | Teardown section | Cleanup procedure |

### Deliverable 7 — CloudWatch Telemetry Guide

| Phase | Sections | Why it matters |
|---|---|---|
| 2 | All sections | OTel + ADOT architecture, three-tier observability, governance gate, EMF schema (auto-generated by ADOT) |
| 7 | Dashboard query patterns, alarms | `/metrics` page query templates |

> **Note:** The Guideline §9.5 and PCD §10 emphasise that the developer's surface area is OTel spans, NOT manual EMF JSON. ADOT does the EMF transformation.

### Deliverable 8 — Frontend Architecture Guide

| Phase | Sections | Why it matters |
|---|---|---|
| 5 | All sections | Dual-realm (Agentic vs Standard), CopilotKit provider scope (workspace route, NOT root), shared state, testing strategy, CSS conflict resolution |

### Deliverable 9 — Multi-Framework AG-UI Guide

| Phase | Sections | Why it matters |
|---|---|---|
| — | All sections | Future reference for adding LangGraph/CrewAI/etc. AG-UI adapters; not a Phase 6 requirement for the demo |

---

## 4. Document Set Summary

| Document | Type | When to use |
|---|---|---|
| `Deliverable_0_PROJECT_CONTEXT.md` | Reference (authoritative) | Architecture, decisions D1–D14, baseline |
| `PAYLOAD_SCHEMA.md` | Reference (authoritative) | 8-block schema, identifiers, error codes |
| `PHASE_PLAN.md` | Master plan | Phase sequencing, gates |
| `APPLY_PLAN.md` | Meta | Documentation cleanup execution plan (this whole rewrite effort) |
| `Prompt_00_..._07_*.md` | Implementation prompts | Phase execution |
| `Deliverable_1_..._9_*.md` | Reference material | Deep dive per topic |
| `ARCHITECTURE_DATAFLOW_GUIDE.md` | Reference (deep architecture) | When you need detailed component interaction diagrams |
| `REPO_STRATEGY.md` | Reference (engineering process) | Polyrepo, CI/CD posture |
| `GIT_WORKFLOW.md` | Reference (engineering process) | Branches, commits, PRs |
| `DEVELOPER_ONBOARDING.md` | Reference (onboarding) | New developer setup |
| `DEMO_GELISTIRME_KILAVUZU.md` | Reference (Turkish dev guide) | Localized walkthrough |
| `PERSISTENT_FILESYSTEM_GUIDE.md` | Reference (AgentCore FS) | POSIX claim-check details |
| `V5_IMPLEMENTATION_PLAN.md` | Reference (historical) | Migration history |
| `V5_TELEMETRY_INTEGRATION_ANALYSIS.md` | Reference (Turkish telemetry deep dive) | OTel + Temporal + claim-check interplay |
| `WORKSPACE_SETUP_GUIDE.md` | Reference (IDE setup) | VS Code multi-root workspace |
| `PROJECT_STARTER.md` | Reference (bootstrap) | One-shot 4-repo creation |
| `FILE_STATUS_REPORT.md` | Reference (meta) | Freshness status per file |
| `DELIVERABLE_MAPPING.md` | This file | Which deliverable feeds which phase |
| `DOCUMENT_AUDIT_REPORT.md` | Reference (historical) | Round 1 audit |
| `CONSISTENCY_AUDIT_REPORT_ROUND2.md` | Reference (historical) | Round 2 audit |
| `CONSISTENCY_AUDIT_REPORT_ROUND3.md` | Reference (current) | Round 3 audit produced by P13 |

---

## 5. Quick Reference: Deliverable by Phase

| Phase | Must-Have (Primary) | Nice-to-Have (Reference) |
|---|---|---|
| 0 | Deliverable 6 | Deliverable 0, Deliverable 1 |
| 1 | Deliverable 2, Deliverable 6 §11 | Deliverable 0, PAYLOAD_SCHEMA |
| 2 | Deliverable 7, PAYLOAD_SCHEMA | Deliverable 0, Deliverable 2 |
| 3 | PCD §5, §3 D10/D11/D12 | Deliverable 4 §1 |
| 4 | Deliverable 4, PAYLOAD_SCHEMA §8 | PCD §11 |
| 5 | Deliverable 5, Deliverable 8 | Deliverable 0 §3 D13 |
| 6 | PROMPT_TESTING_SECTIONS | All previous |
| 7 | Deliverable 7 (dashboard), PCD §3 D14 | PAYLOAD_SCHEMA §10 |

---

## 6. How to Use the Prompts

Each phase prompt is **self-contained** for execution. However, for maximum effectiveness:

1. **Read PCD first** to understand the architectural baseline and frozen decisions.
2. **Read PAYLOAD_SCHEMA** to internalize the 8-block contract.
3. **Read PHASE_PLAN** to understand the phase sequencing.
4. **Open the Prompt for your current phase** alongside its Primary Deliverable(s).
5. **Execute the prompt step by step**, running each Checkpoint command.
6. **Verify the Exit Gate** before moving to the next phase.

If you are using an AI coding assistant (Claude Code, Cursor, Gemini), upload:
- PCD (always)
- PAYLOAD_SCHEMA (Phases 1, 2, 4)
- The current Prompt
- The Primary Deliverable for that phase

That set is enough for the assistant to execute the phase without needing the rest of the document tree.

---

*End of Mapping Document.*
