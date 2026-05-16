# Project Starter: agentcore-demo-test1

**Purpose:** Run this in an EMPTY directory to create the complete 4-repo workspace structure.
**When to run:** Once, at the very beginning, before any other prompt.
**What it does:** Creates repos, workspace file, and directory skeleton. Then tells you where to copy documents.

---

## STEP 1: Validate — You Are in an Empty Directory

```bash
# RUN THIS FIRST. If it prints anything other than "EMPTY", STOP.
ls -A
# EXPECTED OUTPUT: (nothing — directory is completely empty)
```

**If the directory is NOT empty:** Create a new empty directory and `cd` into it:
```bash
mkdir ~/agentcore-demo-test1
cd ~/agentcore-demo-test1
```

---

## STEP 2: Create the 4 Repository Directories

```bash
# Create all 4 repos as subdirectories (NOT nested git repos)
mkdir -p agentcore-demo-test1-backend
mkdir -p agentcore-demo-test1-frontend
mkdir -p agentcore-demo-test1-infra
mkdir -p agentcore-demo-test1-docs
mkdir -p agentcore-demo-test1-docs/resources
mkdir -p agentcore-demo-test1-docs/runbooks
mkdir -p agentcore-demo-test1-docs/decisions

# Verify structure
find . -maxdepth 2 -type d | sort
# EXPECTED OUTPUT:
# .
# ./agentcore-demo-test1-backend
# ./agentcore-demo-test1-docs
# ./agentcore-demo-test1-docs/decisions
# ./agentcore-demo-test1-docs/resources
# ./agentcore-demo-test1-docs/runbooks
# ./agentcore-demo-test1-frontend
# ./agentcore-demo-test1-infra
```

---

## STEP 3: Create the VS Code / Antigravity Workspace File

```bash
cat > agentcore-demo-test1.code-workspace << 'EOF'
{
  "folders": [
    {
      "path": "./agentcore-demo-test1-backend",
      "name": "BACKEND"
    },
    {
      "path": "./agentcore-demo-test1-frontend",
      "name": "FRONTEND"
    },
    {
      "path": "./agentcore-demo-test1-infra",
      "name": "INFRA"
    },
    {
      "path": "./agentcore-demo-test1-docs",
      "name": "DOCS"
    }
  ],
  "settings": {
    "files.exclude": {
      "**/__pycache__": true,
      "**/.venv": true,
      "**/node_modules": true,
      "**/.next": true,
      "**/.terraform": true,
      "**/*.egg-info": true,
      "**/.git": false
    },
    "search.exclude": {
      "**/uv.lock": true,
      "**/pnpm-lock.yaml": true,
      "**/.git": true
    },
    "python.defaultInterpreterPath": "./agentcore-demo-test1-backend/.venv/bin/python",
    "python.analysis.extraPaths": [
      "./agentcore-demo-test1-backend/src"
    ],
    "typescript.tsdk": "./agentcore-demo-test1-frontend/node_modules/typescript/lib"
  }
}
EOF
```

---

## STEP 4: Create Root `.gitignore`

```bash
cat > .gitignore << 'EOF'
# Root .gitignore — the parent directory is NOT a git repo,
# but this file ensures IDEs don't index build artifacts.

# Python
__pycache__/
*.py[cod]
.venv/
venv/
*.egg-info/

# Node.js
node_modules/
.pnpm-store/
.next/

# IDE
.vscode/
.idea/

# OS
.DS_Store
Thumbs.db

# AWS
*.zip
.env.local
*.pem
EOF
```

---

## STEP 5: Create a Placeholder README

```bash
cat > README.md << 'EOF'
# AgentCore demo test 1

Multi-repo workspace for the Audio-to-BRD automation demo.

## Repositories

| Repo | Purpose | Tech |
|------|---------|------|
| `agentcore-demo-test1-backend/` | FastAPI + Temporal + Agent code | Python 3.11, uv, FastAPI, Temporal |
| `agentcore-demo-test1-frontend/` | Next.js web UI + CopilotKit | TypeScript, Next.js 14, pnpm |
| `agentcore-demo-test1-infra/` | Terraform + Docker Compose + CI/CD | Terraform, Docker |
| `agentcore-demo-test1-docs/` | All documentation, prompts, deliverables | Markdown |

## Development Phases

1. Phase 0: AWS Infrastructure (`Prompt_00` — INTERACTIVE)
2. Phase 1: Agent Deployment (`Prompt_01` — AUTOMATIC)
3. Phase 2: Telemetry (`Prompt_02` — AUTOMATIC)
4. Phase 3: Backend (`Prompt_03` — AUTOMATIC)
5. Phase 4: Workflow (`Prompt_04` — AUTOMATIC)
6. Phase 5: Frontend (`Prompt_05` — AUTOMATIC)
7. Phase 6: E2E (`Prompt_06` — AUTOMATIC)

## Workspace Setup

Open `agentcore-demo-test1.code-workspace` in VS Code or Antigravity:
```bash
code agentcore-demo-test1.code-workspace
# or
antigravity agentcore-demo-test1.code-workspace
```

## Resources

All deliverables, prompts, and guides are in `agentcore-demo-test1-docs/resources/`.

See `REPO_STRATEGY.md` and `GIT_WORKFLOW.md` in that folder for architecture decisions.
EOF
```

---

## STEP 6: Create Repo-Level `.gitignore` Files

```bash
# Backend
cat > agentcore-demo-test1-backend/.gitignore << 'EOF'
__pycache__/
*.py[cod]
.venv/
venv/
*.egg-info/
.pytest_cache/
.coverage/
.env.local
*.zip
.DS_Store
resources/
EOF

# Frontend
cat > agentcore-demo-test1-frontend/.gitignore << 'EOF'
node_modules/
.pnpm-store/
.next/
*.log
.env.local
.DS_Store
resources/
EOF

# Infra
cat > agentcore-demo-test1-infra/.gitignore << 'EOF'
.terraform/
*.tfstate*
*.tfvars
.env.local
.DS_Store
resources/
EOF

# Docs (resources/ is NOT gitignored here — this IS the docs repo)
cat > agentcore-demo-test1-docs/.gitignore << 'EOF'
*.zip
.DS_Store
EOF
```

---

## STEP 7: Verify Structure

```bash
ls -la
echo "---"
find . -maxdepth 2 -type f | sort
```

**EXPECTED OUTPUT:**
```
total 24
drwxr-xr-x  6 user user 4096 ...
drwxr-xr-x  2 user user 4096 ... agentcore-demo-test1-backend
drwxr-xr-x  2 user user 4096 ... agentcore-demo-test1-docs
drwxr-xr-x  2 user user 4096 ... agentcore-demo-test1-frontend
drwxr-xr-x  2 user user 4096 ... agentcore-demo-test1-infra
-rw-r--r--  1 user user  ... .gitignore
-rw-r--r--  1 user user  ... README.md
-rw-r--r--  1 user user  ... agentcore-demo-test1.code-workspace

./.gitignore
./README.md
./agentcore-demo-test1-backend/.gitignore
./agentcore-demo-test1-docs/.gitignore
./agentcore-demo-test1-frontend/.gitignore
./agentcore-demo-test1-infra/.gitignore
./agentcore-demo-test1.code-workspace
```

---

## STEP 8: Open Workspace in Editor

```bash
# VS Code:
code agentcore-demo-test1.code-workspace

# Antigravity (Cursor):
antigravity agentcore-demo-test1.code-workspace
```

You will see 4 folders in the Explorer:
- **BACKEND** — agentcore-demo-test1-backend/
- **FRONTEND** — agentcore-demo-test1-frontend/
- **INFRA** — agentcore-demo-test1-infra/
- **DOCS** — agentcore-demo-test1-docs/

---

## STEP 9: Copy All Documents to `resources/`

**This is a MANUAL step. The AI cannot do this for you because the documents come from a separate source.**

Copy ALL the following files into `agentcore-demo-test1-docs/resources/`:

### Prompt Files (7 files)
```
Prompt_00_Infra_Bootstrap.md
Prompt_01_AgentCore_Deployment.md
Prompt_02_Agent_Telemetry.md
Prompt_03_Backend_Foundation.md
Prompt_04_Temporal_Workflow.md
Prompt_05_Frontend.md
Prompt_06_E2E_Integration.md
```

### Deliverable Files (10 files)
```
Deliverable_0_PROJECT_CONTEXT.md
Deliverable_1_Infrastructure_Cost_Report.md
Deliverable_2_Reference_Strands_Agent_Code.md
Deliverable_3_Implementation_Prompts.md
Deliverable_4_Temporal_Operations_Guide.md
Deliverable_5_CopilotKit_AGUI_Guide.md
Deliverable_6_AWS_Operations_Guide.md
Deliverable_7_CloudWatch_Telemetry_Guide.md
Deliverable_8_Frontend_Architecture_Guide.md
Deliverable_9_Multi_Framework_AGUI_Guide.md
```

### Guide Files (11 files)
```
ARCHITECTURE_DATAFLOW_GUIDE.md
REPO_STRATEGY.md
GIT_WORKFLOW.md
DEMO_GELISTIRME_KILAVUZU.md
DEVELOPER_ONBOARDING.md
PERSISTENT_FILESYSTEM_GUIDE.md
PHASE_PLAN.md
V5_IMPLEMENTATION_PLAN.md
V5_TELEMETRY_INTEGRATION_ANALYSIS.md
WORKSPACE_SETUP_GUIDE.md
DELIVERABLE_MAPPING.md
PROMPT_TESTING_SECTIONS.md
DOCUMENT_AUDIT_REPORT.md
CONSISTENCY_AUDIT_REPORT_ROUND2.md
```

### After copying, verify:
```bash
ls agentcore-demo-test1-docs/resources/ | wc -l
# EXPECTED: 34 (or more if additional guides exist)
```

---

## STEP 10: Start Phase 0

Now that the workspace is ready and all documents are in `resources/`, proceed to **Phase 0**:

1. Open the workspace in your editor (VS Code or Antigravity)
2. Read `resources/Prompt_00_Infra_Bootstrap.md`
3. Follow its instructions — it will guide you through AWS infrastructure setup

**Each prompt file has a "Reference Documents" section at the top** that tells you exactly which deliverables to read before executing that phase.

---

## Quick Reference: Phase-to-Prompt-to-Documents Map

| Phase | Prompt File | Primary Documents to Read | What They Contain |
|-------|-------------|--------------------------|-------------------|
| 0 | `Prompt_00_Infra_Bootstrap.md` | `Deliverable_1` (Section A scripts), `Deliverable_6` (Section 2 prerequisites) | AWS shell scripts, IAM setup |
| 1 | `Prompt_01_AgentCore_Deployment.md` | `Deliverable_2` (agent code skeletons), `Deliverable_6` (Sections 3, 5) | BedrockAgentCoreApp pattern, S3 ZIP deploy |
| 2 | `Prompt_02_Agent_Telemetry.md` | `Deliverable_2` (Section 4 payload builder), `Deliverable_7` (concepts only) | 8-block payload, OTel setup |
| 3 | `Prompt_03_Backend_Foundation.md` | `Deliverable_3` (Prompt 1 backend scaffold) | FastAPI structure, docker-compose |
| 4 | `Prompt_04_Temporal_Workflow.md` | `Deliverable_4` (Temporal patterns), `Deliverable_3` (Prompt 2 workflow) | HITL signals, A2A, ContinueAsNew |
| 5 | `Prompt_05_Frontend.md` | `Deliverable_5` (CopilotKit hooks), `Deliverable_8` (realm separation) | useAgent, dual-stream, AG-UI |
| 6 | `Prompt_06_E2E_Integration.md` | `Deliverable_0` (complete spec), `Deliverable_9` (E2E checklist) | Full workflow, CloudWatch verify |

**Architecture reference (read once, use throughout):**
- `Deliverable_0_PROJECT_CONTEXT.md` — The single source of truth for all decisions
- `ARCHITECTURE_DATAFLOW_GUIDE.md` — Complete system architecture with all 21 interactions

---

*End of Project Starter*
