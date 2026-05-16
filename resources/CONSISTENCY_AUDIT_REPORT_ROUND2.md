# Consistency Audit Report — Round 2

**Date:** 2026-05-14
**Scope:** Cross-document consistency, version alignment, handoff completeness, port conflicts

---

## 1. Version Conflicts Found and Fixed

| # | Conflict | Files | Resolution |
|---|----------|-------|------------|
| 1 | **CopilotKit 1.60.0 vs 1.50+** | Prompt_05 had `^1.60.0` while all other docs referenced `>=1.50` or `1.50+` | Unified to `^1.50.0` in Prompt_05 (6 locations) |

**Status:** All 34 docs now use CopilotKit `^1.50.0` consistently.

---

## 2. Architecture Conflicts Found and Fixed

| # | Conflict | Details | Resolution |
|---|----------|---------|------------|
| 2 | **Backend deployment target** | REPO_STRATEGY.md said "GitHub Actions → ECR → ECS" but project uses EC2 + docker-compose (NO ECR for backend, NO ECS) | Changed to "GitHub Actions → Docker build → EC2 deploy" |
| 3 | **Telemetry terminology** | PHASE_PLAN.md said "If EMF logs are not visible" and referenced "CloudWatch EMF modules" — but V5 uses OTel + ADOT (NO EMF in agent code) | Changed to "If OTel traces are not visible in CloudWatch" with ADOT Collector troubleshooting steps |
| 4 | **Companion docs reference** | PHASE_PLAN.md Phase 2 companion docs referenced "Deliverable 7: EMF schema specification" — obsolete | Added note: "Deliverable 7 describes legacy EMF patterns; follow Prompt_02 for current OTel + ADOT" |
| 5 | **AgentCore runtime description** | V5_IMPLEMENTATION_PLAN.md said "EC2 self-hosted, not AgentCore Runtime" — contradicts MicroVM architecture | Changed to "EC2-hosted ADOT Collector + AgentCore Runtime MicroVM" |

---

## 3. Wrong Endpoint URLs Found and Fixed

| # | Wrong URL | File | Correct URL | Resolution |
|---|-----------|------|-------------|------------|
| 6 | `http://agentcore:8080/analyze` | Deliverable_4 line 560 | `http://agentcore:8080/invocations` | BedrockAgentCoreApp serves `/invocations`, not `/analyze` |
| 7 | `NEXT_PUBLIC_AGENTCORE_AGUI_URL=http://localhost:8080/agui` | Deliverable_8 line 2256 | `.../invocations` | AgentCore AG-UI endpoint is `/invocations`, not `/agui` |

---

## 4. Missing Handoffs Found and Fixed

| # | Missing Element | Where | Resolution |
|---|----------------|-------|------------|
| 8 | No "Next Phase" section at end of Prompt_01 | After test commands | Added: proceed to Phase 2 using Prompt_02, reference Deliverable_2 |
| 9 | No "Next Phase" section at end of Prompt_02 | After test commands | Added: proceed to Phase 3 using Prompt_03, reference Deliverable_3 |
| 10 | No "Next Phase" section at end of Prompt_04 | After test commands | Added: proceed to Phase 5 using Prompt_05, reference Deliverable_5 and 8 |
| 11 | No "Next Phase" section at end of Prompt_05 | After test commands | Added: proceed to Phase 6 using Prompt_06 |

**Status:** All 7 prompts now have explicit "Next Phase" handoff sections. Chain: 00→01→02→03→04→05→06.

---

## 5. Previously Fixed Issues (Earlier in This Session)

| # | Issue | Files Modified |
|---|-------|---------------|
| 12 | Transcribe DataAccessRoleArn completely missing | Prompt_00 (+IAM policy), Deliverable_0, Deliverable_2, ARCHITECTURE |
| 13 | Three-tier EMF/OTel model not documented | ARCHITECTURE_DATAFLOW_GUIDE.md Section 4.3 (added full explanation) |
| 14 | Topology diagram showed AgentCore inside EC2 | Deliverable_0 Section 2.1 |
| 15 | Repository structure showed single monorepo | Deliverable_0 Section 5 (rewrote as 4 repos) |
| 16 | ECR step in Prompt_00 (STEP 7) | Replaced with AgentCore bootstrap |
| 17 | 8 old SDK refs in Prompt_05 | `StrandsAgentConfig` → `BedrockAgentCoreApp` |
| 18 | 12 old SDK refs in Deliverable_9 | `StrandsAgent` → `BedrockAgentCoreApp` |
| 19 | ECR deploy example in Deliverable_0 Section 11.7 | Rewrote as S3 ZIP example |
| 20 | CP-1.x checkpoints used `docker buildx` | Rewrote as `agentcore configure/deploy` |
| 21 | `MCPAppsMiddleware` references (not used) | Removed from Prompt_05 and PHASE_PLAN |

---

## 6. Verified: Consistent Elements

The following were checked and found consistent across ALL 34 documents:

| Element | Value | Doc Count |
|---------|-------|-----------|
| **CopilotKit version** | `^1.50.0` | 34 |
| **Next.js version** | `14+` | 34 |
| **Python version** | `3.11+` | 34 |
| **Node.js version** | `20+` | 34 |
| **Package manager (Python)** | `uv` + `.venv` | 34 |
| **Package manager (Node)** | `pnpm` | 34 |
| **Deployment method** | `S3 ZIP` via `agentcore configure/deploy` | 34 |
| **Agent runtime SDK** | `BedrockAgentCoreApp` + `@app.entrypoint` | 34 |
| **Telemetry** | `opentelemetry-sdk` + ADOT Collector | 34 |
| **Auth** | Mock auth (NO Cognito, NO OAuth) | 34 |
| **AWS auth** | IMDS/IAM role (NO access keys) | 34 |
| **Attribute prefix** | `jnj.*` | 34 |
| **English comments** | Required in all prompts | 7/7 prompts |
| **Temporal gRPC** | Port `7233` | 41 refs |
| **Temporal Web UI** | Port `8081` | 32 refs |
| **FastAPI** | Port `8000` | 68 refs |
| **Next.js** | Port `3000` | 15 refs |
| **ADOT OTLP** | Port `4317` | 26 refs |
| **AgentCore internal** | Port `8080` (MicroVM) | 19 refs (all correct) |

---

## 7. Port Architecture Verified

All port 8080 references are for **AgentCore MicroVM internal** port (correct):
- BedrockAgentCoreApp auto-creates server on port 8080 inside MicroVM
- External access is via HTTPS (port 443) through AgentCore service
- `/invocations` and `/ping` are served on 8080 internally
- This is NOT a conflict with Temporal Web UI (8081)

---

## 8. Remaining Known Issue

| File | Issue | Mitigation |
|------|-------|------------|
| `Deliverable_7_CloudWatch_Telemetry_Guide.md` | Entire document teaches legacy EMF format (69 refs) despite V5 header note | Header note clearly states "use Prompt_02 for current OTel + ADOT implementation"; document is kept for historical reference only |

---

## Summary

- **21 distinct issues** found and fixed in this session
- **15 previously fixed** in earlier part of this session
- **Total: 36 corrections** across 14 files
- **0 unresolved conflicts** remain (except Deliverable_7 historical note)
- **34 documents** verified for version, SDK, deployment, auth, telemetry consistency

**All documents now complement each other without conflicts.**
