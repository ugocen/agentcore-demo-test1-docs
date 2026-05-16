# Document Audit Report — agentcore-demo-test1

**Date:** 2026-05-14
**Auditor:** Systematic review of all 30 .md files + 4 .png files
**Scope:** Identify stale references, incorrect architecture descriptions, and obsolete patterns

---

## 1. Executive Summary

### Critical Issues Found and Fixed

| # | File | Issue | Severity | Status |
|---|------|-------|----------|--------|
| 1 | `Deliverable_0_PROJECT_CONTEXT.md` | Topology diagram showed AgentCore Runtime INSIDE EC2 as Docker container | **CRITICAL** | **FIXED** |
| 2 | `Deliverable_0_PROJECT_CONTEXT.md` | Repository structure showed single monorepo (not 4 repos) | **CRITICAL** | **FIXED** |
| 3 | `Deliverable_0_PROJECT_CONTEXT.md` | Agent directories included Dockerfile (S3 ZIP, no Dockerfile) | **CRITICAL** | **FIXED** |
| 4 | `Deliverable_0_PROJECT_CONTEXT.md` | Emission point referenced `emf_logger.py` (OTel + ADOT, not EMF) | **CRITICAL** | **FIXED** |
| 5 | `Deliverable_0_PROJECT_CONTEXT.md` | Tech Reference used `create_strands_app()` + `StrandsAgent` (old SDK) | **CRITICAL** | **FIXED** |
| 6 | `Deliverable_0_PROJECT_CONTEXT.md` | Companion Documents listed obsolete filenames | **HIGH** | **FIXED** |
| 7 | `Prompt_00_Infra_Bootstrap.md` | STEP 7: Created ECR repositories (not needed for S3 ZIP) | **CRITICAL** | **FIXED** |
| 8 | `Prompt_00_Infra_Bootstrap.md` | GATE 8 checked ECR repos (replaced with AgentCore bootstrap check) | **HIGH** | **FIXED** |
| 9 | `Prompt_00_Infra_Bootstrap.md` | Cleanup script deleted ECR repos (removed) | **MEDIUM** | **FIXED** |
| 10 | `Prompt_05_Frontend.md` | 8 references to `StrandsAgentConfig` / `StrandsAgent` / `create_strands_app` | **CRITICAL** | **FIXED** |
| 11 | `Prompt_05_Frontend.md` | CLI API list included `MCPAppsMiddleware` (not used) | **HIGH** | **FIXED** |
| 12 | `PHASE_PLAN.md` | Phase 0 scope included "Create ECR repository" | **HIGH** | **FIXED** |
| 13 | `PHASE_PLAN.md` | Phase 1 scope: "Build containers, push ECR" (S3 ZIP needed) | **CRITICAL** | **FIXED** |
| 14 | `PHASE_PLAN.md` | Phase 1 Exit Gate: "images pushed to ECR" (ZIP deploy needed) | **CRITICAL** | **FIXED** |
| 15 | `PHASE_PLAN.md` | Final decision: "manuel Dockerfile + ECR" (S3 ZIP needed) | **CRITICAL** | **FIXED** |

### Additional Issues Found in Deep Review (Second Pass)

| # | File | Issue | Severity | Status |
|---|------|-------|----------|--------|
| 16 | `Deliverable_0_PROJECT_CONTEXT.md` | Section 11.7: ECR deploy code example (`docker build` + `docker push` + `--image-uri`) | **CRITICAL** | **FIXED** |
| 17 | `Deliverable_0_PROJECT_CONTEXT.md` | Section 11.7: "Deployment requires pre-built ARM64 container in ECR" in gotchas | **CRITICAL** | **FIXED** |
| 18 | `Deliverable_9_Multi_Framework_AGUI_Guide.md` | 12 references to `StrandsAgent` / `create_strands_app` / `ag_ui_strands` | **CRITICAL** | **FIXED** |
| 19 | `Deliverable_9_Multi_Framework_AGUI_Guide.md` | ECR URI deploy example (`--image-uri "$ECR_URI/..."`) | **CRITICAL** | **FIXED** |
| 20 | `Deliverable_7_CloudWatch_Telemetry_Guide.md` | 69 EMF references vs 22 OTel; entire doc is EMF-focused despite V5 note | **HIGH** | **KNOWN ISSUE** |

**Total: 20 issues found, 19 fixed, 1 known issue documented.**

---

## 2. File-by-File Status

### PROMPT FILES (Active Development Instructions)

These are the files given to the AI coding assistant during development.

| File | Status | Issues Found | Used In Phase |
|------|--------|--------------|---------------|
| `Prompt_00_Infra_Bootstrap.md` | **UPDATED** | ECR step replaced with AgentCore bootstrap | FAZ 0 |
| `Prompt_01_AgentCore_Deployment.md` | **CLEAN** | S3 ZIP correctly specified throughout | FAZ 1 |
| `Prompt_02_Agent_Telemetry.md` | **CLEAN** | OTel + ADOT correctly specified | FAZ 2 |
| `Prompt_03_Backend_Foundation.md` | **CLEAN** | Backend Dockerfile correctly scoped (infra only, not agents) | FAZ 3 |
| `Prompt_04_Temporal_Workflow.md` | **CLEAN** | S3 ZIP correctly specified | FAZ 4 |
| `Prompt_05_Frontend.md` | **UPDATED** | 8 old SDK references fixed (`StrandsAgentConfig` → `BedrockAgentCoreApp`) | FAZ 5 |
| `Prompt_06_E2E_Integration.md` | **CLEAN** | S3 ZIP correctly specified | FAZ 6 |

### DELIVERABLE FILES (Reference Documentation)

Deep context documents uploaded alongside prompts for architecture rationale.

| File | Status | Issues Found | When to Read |
|------|--------|--------------|--------------|
| `Deliverable_0_PROJECT_CONTEXT.md` | **UPDATED** | 6 critical issues fixed (topology, repos, SDK, EMF, docs) | All phases (decisions, payload ref, architecture) |
| `Deliverable_1_Infrastructure_Cost_Report.md` | **CLEAN** | No ECR cost; S3/Secrets Manager correct | Budget review |
| `Deliverable_2_Reference_Strands_Agent_Code.md` | **CLEAN** | Uses `BedrockAgentCoreApp` + `@app.entrypoint` correctly | Agent code skeletons |
| `Deliverable_3_Implementation_Prompts.md` | **CLEAN** | V5-aligned: S3 ZIP, no ECR, no Cognito, opentelemetry-sdk | Backend skeleton |
| `Deliverable_4_Temporal_Operations_Guide.md` | **CLEAN** | No stale references found | Temporal patterns |
| `Deliverable_5_CopilotKit_AGUI_Guide.md` | **CLEAN** | V5 note present; AG-UI/A2A separation correct | Frontend hooks |
| `Deliverable_6_AWS_Operations_Guide.md` | **CLEAN** | S3 ZIP deploy, no ECR references | IAM, deployment |
| `Deliverable_7_CloudWatch_Telemetry_Guide.md` | **KNOWN ISSUE** | 69 EMF refs vs 22 OTel; V5 header note present but content is EMF-focused. Use Prompt_02 for OTel guidance | CloudWatch architecture |
| `Deliverable_8_Frontend_Architecture_Guide.md` | **CLEAN** | V5 note present; mock auth, dual-stream correct | Realm separation |
| `Deliverable_9_Multi_Framework_AGUI_Guide.md` | **UPDATED** | 12 old SDK refs + 1 ECR deploy ref fixed | Framework integrations |

### GUIDE FILES (Process & Architecture)

| File | Status | Notes |
|------|--------|-------|
| `ARCHITECTURE_DATAFLOW_GUIDE.md` | **UPDATED** | AgentCore MicroVM clarification added (Section 1, Layer 4) |
| `REPO_STRATEGY.md` | **NEW (this session)** | 4-repo layout, Dockerfiles, CI/CD |
| `GIT_WORKFLOW.md` | **NEW (this session)** | Branch strategy, daily process, repo setup |
| `DEMO_GELISTIRME_KILAVUZU.md` | **UPDATED** | 4-repo structure, dev branch workflow, `resources/` explanation |
| `PHASE_PLAN.md` | **UPDATED** | ECR→S3 ZIP, container→ZIP deploy, all stale refs fixed |
| `DEVELOPER_ONBOARDING.md` | **CLEAN** | IMDS, uv, .venv correctly specified |
| `PERSISTENT_FILESYSTEM_GUIDE.md` | **CLEAN** | S3 ZIP deployment correctly stated |
| `V5_IMPLEMENTATION_PLAN.md` | **MOSTLY CLEAN** | Lists needed updates; some "EC2 self-hosted" wording needs review |
| `V5_TELEMETRY_INTEGRATION_ANALYSIS.md` | **CLEAN** | No stale references found |
| `DELIVERABLE_MAPPING.md` | **CLEAN** | Correctly marks ECR as legacy |
| `PROMPT_TESTING_SECTIONS.md` | **CLEAN** | Pytest requirements only |
| `WORKSPACE_SETUP_GUIDE.md` | **NEW** | VS Code + Antigravity multi-root workspace |

---

## 3. Architecture Decisions Verified

## 3. Architecture Decisions Verified

### Correct (No Changes Needed)

| Decision | Files Verified |
|----------|---------------|
| S3 ZIP deployment (NO ECR, NO Docker) | Prompt_01, Prompt_02, Prompt_03, Prompt_04, Prompt_06, Deliverable_2, Deliverable_6, PERSISTENT_FILESYSTEM_GUIDE |
| `bedrock-agentcore` SDK + `@app.entrypoint` | Deliverable_2, Prompt_01 (correct), ARCHITECTURE_DATAFLOW_GUIDE |
| OTel + ADOT Collector (NO custom SDK, NO manual EMF) | Prompt_02, Deliverable_0 (fixed), ARCHITECTURE_DATAFLOW_GUIDE |
| IMDS for AWS auth (NO access keys) | DEVELOPER_ONBOARDING, Deliverable_6, all prompts |
| Mock auth (NO Cognito, NO OAuth) | Deliverable_0, all prompts |
| `jnj.*` attribute prefix | Deliverable_0, Prompt_02 |
| 4 separate repos (polyrepo) | REPO_STRATEGY, GIT_WORKFLOW, DEMO_GELISTIRME_KILAVUZU |
| uv + .venv (NO global pip) | All prompts, DEVELOPER_ONBOARDING |
| English-only comments | All prompts |
| AgentCore Persistent Filesystems (S3 Files) | PERSISTENT_FILESYSTEM_GUIDE |

### Fixed (Were Incorrect)

| Decision | Was | Now | Fixed In |
|----------|-----|-----|----------|
| AgentCore location | Docker container on EC2 | AWS managed service, Firecracker MicroVM | Deliverable_0, ARCHITECTURE_DATAFLOW_GUIDE |
| Repository layout | Single monorepo | 4 separate repos | Deliverable_0, REPO_STRATEGY, GIT_WORKFLOW, DEMO_GELISTIRME_KILAVUZU |
| Agent deployment artifact | Dockerfile + ECR push | S3 ZIP via `agentcore configure/deploy` | Prompt_00, PHASE_PLAN, Deliverable_0 |
| Agent runtime SDK | `create_strands_app()` + `StrandsAgent` | `BedrockAgentCoreApp` + `@app.entrypoint` | Deliverable_0, Prompt_05 |
| Telemetry emission | `emf_logger.py` manual EMF | OTel span events → ADOT → CloudWatch | Deliverable_0 |
| Infra bootstrap step | Create ECR repositories | AgentCore toolkit bootstrap | Prompt_00 |

---

## 4. How Files Are Used During Development

### Phase 0: AWS Infrastructure

```
You give AI: Prompt_00_Infra_Bootstrap.md
You give AI (reference): Deliverable_6_AWS_Operations_Guide.md, PERSISTENT_FILESYSTEM_GUIDE.md
AI output: Shell scripts in infra/scripts/, .env files, docker-compose.yml
```

### Phase 1: Agent Deployment

```
You give AI: Prompt_01_AgentCore_Deployment.md
You give AI (reference): Deliverable_2_Reference_Strands_Agent_Code.md
AI output: agents/{transcriber,drafter,reviewer}/main.py, requirements.txt
AI runs: agentcore configure -e main.py --protocol AGUI && agentcore deploy
```

### Phase 2: Telemetry

```
You give AI: Prompt_02_Agent_Telemetry.md
You give AI (reference): Deliverable_0 (payload ref), Deliverable_7 (CloudWatch)
AI output: V5 payload builder (raw dict), OTel instrumentation in agents
```

### Phase 3: Backend

```
You give AI: Prompt_03_Backend_Foundation.md
You give AI (reference): Deliverable_0 (decisions, repo structure)
AI output: FastAPI app, Temporal client/worker scaffold, docker-compose
```

### Phase 4: Workflow

```
You give AI: Prompt_04_Temporal_Workflow.md
You give AI (reference): Deliverable_4 (Temporal patterns)
AI output: BRD workflow, activities, HITL signal loop, A2A
```

### Phase 5: Frontend

```
You give AI: Prompt_05_Frontend.md
You give AI (reference): Deliverable_5 (CopilotKit), Deliverable_8 (architecture)
AI output: Next.js app, CopilotKit components, dual-stream hooks
```

### Phase 6: E2E

```
You give AI: Prompt_06_E2E_Integration.md
You give AI (reference): All deliverables
AI output: E2E tests, CloudWatch verification
```

---

## 5. Files That Are NOT Used During Development

These files exist for **human reference only**. The AI coding assistant does NOT read them during code generation:

| File | Purpose | Audience |
|------|---------|----------|
| `REPO_STRATEGY.md` | Architecture decision record: why 4 repos | Team leads, architects |
| `GIT_WORKFLOW.md` | Git process: branches, commits, PRs | All developers |
| `DEVELOPER_ONBOARDING.md` | Day-0 environment setup | New team members |
| `DEMO_GELISTIRME_KILAVUZU.md` | Turkish step-by-step guide | Turkish-speaking developers |
| `V5_IMPLEMENTATION_PLAN.md` | V5 migration checklist (historical) | Architects |
| `V5_TELEMETRY_INTEGRATION_ANALYSIS.md` | Deep telemetry architecture analysis | Backend developers |
| `DELIVERABLE_MAPPING.md` | Which deliverable to read for which phase | Developers navigating docs |
| `PROMPT_TESTING_SECTIONS.md` | Summary of test requirements per prompt | QA engineers |
| `ARCHITECTURE_DATAFLOW_GUIDE.md` | Complete system architecture | Architects, senior developers |
| All 4 `.png` images | Visual architecture references | All (for quick understanding) |

---

## 6. Remaining Review Needed

The following files have **residual concerns** that should be reviewed before production use:

| File | Issue | Priority |
|------|-------|----------|
| `Deliverable_7_CloudWatch_Telemetry_Guide.md` | Entire document teaches EMF format (69 refs). V5 uses OTel + ADOT. Header has V5 note but content is obsolete. **Do NOT use this for telemetry implementation** — use Prompt_02 instead. | **HIGH** |
| `V5_IMPLEMENTATION_PLAN.md` | Some "EC2 self-hosted" wording may need alignment with MicroVM architecture | Low |

All other files have been reviewed and are current.

---

*End of Audit Report*
