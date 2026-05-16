# Repository Strategy: agentcore-demo-test1

**Date:** 2026-05-14
**Classification:** Architecture Decision — Repository Layout
**Applies To:** All development teams (backend, frontend, infrastructure)

---

## Table of Contents

1. [Decision Summary](#1-decision-summary)
2. [Repository Boundaries](#2-repository-boundaries)
3. [Why Separate Repos?](#3-why-separate-repos)
4. [Repository: `agentcore-demo-test1-backend`](#4-repository-agentcore-demo-test1-backend)
5. [Repository: `agentcore-demo-test1-frontend`](#5-repository-agentcore-demo-test1-frontend)
6. [Repository: `agentcore-demo-test1-infra`](#6-repository-agentcore-demo-test1-infra)
7. [Repository: `agentcore-demo-test1-docs`](#7-repository-agentcore-demo-test1-docs)
8. [Shared Resources](#8-shared-resources)
9. [Dockerfile Strategy](#9-dockerfile-strategy)
10. [CI/CD Pipeline Separation](#10-cicd-pipeline-separation)
11. [Cross-Repo Development Workflow](#11-cross-repo-development-workflow)

---

## 1. Decision Summary

| Question | Answer |
|---|---|
| **Monorepo or Polyrepo?** | **Polyrepo** — separate repositories for backend, frontend, infrastructure, and documentation |
| **Rationale** | Independent deployment cadence, technology isolation, access control, clear ownership |
| **Shared code?** | API contracts shared via OpenAPI specs + generated clients (no shared source tree) |
| **Coordination** | Git submodules for pinned API contracts; GitHub/GitLab issues for cross-repo tracking |

**Key Principle:** Each repository owns its entire lifecycle — from local development to production deployment. No repository depends on another's build artifacts at development time.

---

## 2. Repository Boundaries

```
agentcore-demo-test1/
├── agentcore-demo-test1-backend/     # Python FastAPI + Temporal (owns all Python code)
├── agentcore-demo-test1-frontend/    # Next.js 14 + CopilotKit (owns all TypeScript code)
├── agentcore-demo-test1-infra/       # Terraform/CDK + Docker Compose + AWS configs
└── agentcore-demo-test1-docs/        # Markdown docs + architecture diagrams + runbooks
```

**What does NOT go in code repos:**
- Prompt files (Gate 00-06) → `resources/` in docs repo
- Deliverables (0-9) → `resources/` in docs repo
- Architecture guides → `resources/` in docs repo
- ZIP archives → `.gitignore` in all repos

---

## 3. Why Separate Repos?

### 3.1 Independent Deployment Cadence

| Repository | Deployment Frequency | Trigger | Target Environment |
|---|---|---|---|
| `backend` | Per-merge to `main` | Git push → GitHub Actions → Docker build → EC2 deploy | EC2 (FastAPI + Temporal Worker via docker-compose) |
| `frontend` | Per-merge to `main` | Git push → GitHub Actions → Docker build → EC2 deploy | EC2 (Next.js container via docker-compose) |
| `infra` | On-demand / scheduled | Manual trigger → Terraform apply | AWS (all services) |
| `docs` | Per-merge to `main` | Git push → GitHub Pages | GitHub Pages |

**Scenario:** A frontend hotfix (CSS bug) should deploy in 2 minutes without touching the backend. A backend agent update should not trigger a frontend rebuild.

### 3.2 Technology Isolation

| Repository | Language | Runtime | Package Manager | Build Tool |
|---|---|---|---|---|
| `backend` | Python 3.11+ | uv + .venv | `uv pip` | `pytest`, `ruff` |
| `frontend` | TypeScript 5.3+ | Node.js 20+ | `pnpm` | `next build`, `vitest` |
| `infra` | HCL / Python | Terraform 1.7+ / CDK | `terraform` | `terraform plan` |
| `docs` | Markdown | — | — | `mkdocs` (optional) |

No shared build system. No shared dependency tree. Each repo uses its native tooling.

### 3.3 Access Control & Ownership

| Repository | Primary Owner | Reviewers | Deployment Approval |
|---|---|---|---|
| `backend` | Backend Lead | Architect, QA Lead | Backend Lead |
| `frontend` | Frontend Lead | UX Lead, QA Lead | Frontend Lead |
| `infra` | DevOps Lead | Architect, Security | DevOps Lead + Architect |
| `docs` | Tech Writer | All leads | Any lead |

### 3.4 Avoiding Monorepo Anti-Patterns

A monorepo would require:
- A unified build system (Bazel/Nx/Turborepo) → adds complexity
- Cross-language dependency management → fragile
- Shared CI pipeline → slow (backend tests run on frontend changes)
- Shared `node_modules` / `.venv` → version conflicts
- Blurred ownership → "someone else will fix it"

**Decision:** Avoid these costs. The four repos are truly independent.

---

## 4. Repository: `agentcore-demo-test1-backend`

### 4.1 Purpose
All Python code for the backend: FastAPI server, Temporal workflow definitions, Temporal worker, activity implementations, OpenTelemetry instrumentation, evidence pack builder, S3 handlers.

### 4.2 Directory Structure

```
agentcore-demo-test1-backend/
├── .gitignore
├── .python-version              # "3.11"
├── pyproject.toml               # uv project config + deps
├── uv.lock                      # uv lockfile
├── Dockerfile                   # FastAPI + Worker (see Section 9)
├── docker-compose.yml           # Local dev: FastAPI + Temporal + PostgreSQL
├── README.md
├── pytest.ini
├── .env.example
├── .env.local                   # gitignored — local secrets
│
├── src/
│   ├── __init__.py
│   ├── main.py                  # FastAPI app entry point
│   ├── otel_setup.py            # OpenTelemetry tracer config
│   ├── config.py                # Pydantic Settings (env vars)
│   │
│   ├── api/
│   │   ├── __init__.py
│   │   ├── router.py            # Main API router
│   │   ├── workflows.py         # /api/workflows/* endpoints
│   │   ├── uploads.py           # /api/uploads/* endpoints
│   │   ├── evidence.py          # /api/evidence/* endpoints
│   │   └── dependencies.py      # Mock auth, DB session, OTel tracer
│   │
│   ├── temporal/
│   │   ├── __init__.py
│   │   ├── client.py            # Temporal client singleton
│   │   ├── worker.py            # Temporal worker entry point
│   │   ├── workflow/
│   │   │   ├── __init__.py
│   │   │   └── brd_workflow.py  # @workflow.defn + HITL signal loop
│   │   └── activities/
│   │       ├── __init__.py
│   │       ├── agent_01_transcriber.py   # Agent 1 activity
│   │       ├── agent_02_drafter.py       # Agent 2 activity
│   │       └── agent_03_reviewer.py      # Agent 3 activity
│   │
│   ├── services/
│   │   ├── __init__.py
│   │   ├── s3_handler.py        # S3 claim-check operations
│   │   ├── evidence_pack.py     # ALCOA+ evidence pack builder
│   │   ├── sse_bridge.py        # Temporal → Browser SSE bridge
│   │   └── db.py                # PostgreSQL connection pool
│   │
│   └── models/
│       ├── __init__.py
│       ├── workflow.py          # Pydantic models for workflow API
│       ├── payload.py           # V5 payload 8-block model
│       └── enums.py             # Status enums, Phase enums
│
├── agents/
│   ├── __init__.py
│   ├── transcriber/
│   │   ├── main.py              # @app.entrypoint for Agent 1
│   │   ├── strands.py           # Strands agent definition
│   │   ├── tools.py             # Transcriber tools (Transcribe, etc.)
│   │   └── requirements.txt     # Agent 1 deps (separate from backend)
│   ├── drafter/
│   │   ├── main.py              # @app.entrypoint for Agent 2
│   │   ├── strands.py           # Strands agent definition
│   │   ├── tools.py             # Drafter tools (HITL, etc.)
│   │   └── requirements.txt     # Agent 2 deps
│   └── reviewer/
│       ├── main.py              # @app.entrypoint for Agent 3
│       ├── stategraph.py        # LangGraph StateGraph (4 nodes)
│       ├── nodes.py             # review_compliance, review_quality, review_pii, compile_report
│       └── requirements.txt     # Agent 3 deps
│
├── tests/
│   ├── __init__.py
│   ├── conftest.py              # Shared pytest fixtures
│   ├── unit/
│   │   ├── test_payload.py      # V5 payload validation tests
│   │   ├── test_workflow.py     # Workflow logic tests
│   │   └── test_otel.py         # OTel instrumentation tests
│   ├── integration/
│   │   ├── test_api.py          # FastAPI endpoint tests
│   │   └── test_temporal.py     # Temporal workflow tests
│   └── e2e/
│       └── test_full_brd_flow.py # End-to-end BRD generation
│
└── scripts/
    ├── dev_server.sh            # uv run uvicorn src.main:app --reload
    ├── worker.sh                # uv run python -m src.temporal.worker
    ├── agent_deploy.sh          # agentcore configure + deploy wrapper
    └── db_migrate.sh            # Database migration runner
```

### 4.3 Key Rules
- **Python 3.11+ only** — set in `.python-version`
- **uv + .venv** — no global pip, no conda, no poetry
- **Agent code isolation** — each agent has its own `requirements.txt` (deployed separately via S3 ZIP)
- **No ECR** — agents deploy via S3 ZIP (`agentcore configure/deploy`)
- **Backend Docker image** — FastAPI + Worker only (not agents)

---

## 5. Repository: `agentcore-demo-test1-frontend`

### 5.1 Purpose
All TypeScript/React code for the frontend: Next.js 14+ App Router, CopilotKit AG-UI integration, canvas state machine, dual-stream event handling, Tailwind + shadcn/ui components.

### 5.2 Directory Structure

```
agentcore-demo-test1-frontend/
├── .gitignore
├── package.json                 # pnpm workspace + deps
├── pnpm-lock.yaml
├── Dockerfile                   # Multi-stage Next.js build (see Section 9)
├── docker-compose.yml           # Local dev: Next.js dev server
├── next.config.js
├── tailwind.config.ts
├── tsconfig.json
├── README.md
├── .env.example
├── .env.local                   # gitignored — local API endpoints
│
├── src/
│   ├── app/
│   │   ├── layout.tsx           # Root layout with providers
│   │   ├── page.tsx             # Landing page
│   │   ├── workspace/
│   │   │   └── [id]/
│   │   │       └── page.tsx     # Main workspace (canvas + chat)
│   │   └── api/
│   │       └── upload/
│   │           └── route.ts     # Pre-signed S3 URL generation
│   │
│   ├── components/
│   │   ├── ui/                  # shadcn/ui components (Button, Card, Dialog, etc.)
│   │   ├── canvas/
│   │   │   ├── CanvasStateMachine.tsx   # Zustand state merger
│   │   │   ├── WorkflowProgress.tsx     # Progress bar + phase indicator
│   │   │   ├── TranscriptPanel.tsx      # Transcript display
│   │   │   ├── BRDPreview.tsx           # BRD markdown preview
│   │   │   ├── ReviewPanel.tsx          # Agent 3 review findings
│   │   │   └── EvidencePack.tsx         # Download + ALCOA+ badge
│   │   ├── chat/
│   │   │   ├── ChatPanel.tsx
│   │   │   ├── MessageBubble.tsx
│   │   │   └── ToolCallCard.tsx
│   │   └── hitl/
│   │       └── ClarificationQuestionCard.tsx
│   │
│   ├── hooks/
│   │   ├── useAgent.ts          # CopilotKit useAgent hook
│   │   ├── useWorkflowStream.ts # Custom SSE → Temporal stream
│   │   ├── useCanvasState.ts    # Zustand canvas state
│   │   └── useUpload.ts         # S3 pre-signed upload
│   │
│   ├── lib/
│   │   ├── copilotkit.ts        # CopilotKit provider config
│   │   ├── api.ts               # FastAPI client (generated from OpenAPI)
│   │   ├── s3.ts                # S3 upload utilities
│   │   └── utils.ts             # cn() helper, formatters
│   │
│   ├── types/
│   │   ├── workflow.ts          # Workflow type definitions
│   │   ├── stream.ts            # AG-UI + Workflow stream event types
│   │   └── payload.ts           # V5 payload TypeScript types
│   │
│   └── store/
│       └── canvasStore.ts       # Zustand store definition
│
├── public/
│   └── assets/
│       └── logo.svg
│
├── tests/
│   ├── unit/
│   │   ├── CanvasStateMachine.test.tsx
│   │   └── payload.test.ts
│   ├── integration/
│   │   └── api.test.ts
│   └── e2e/
│       └── brd-flow.spec.ts     # Playwright E2E test
│
└── scripts/
    └── dev.sh                   # pnpm dev (Next.js 3000)
```

### 5.3 Key Rules
- **Node.js 20+**, **pnpm only** — no npm, no yarn
- **Next.js 14+ App Router** — no Pages Router
- **No global packages** — all tools in `devDependencies`
- **TypeScript strict mode** — no `any` without explicit exception

---

## 6. Repository: `agentcore-demo-test1-infra`

### 6.1 Purpose
All infrastructure-as-code: Terraform modules for AWS resources, Docker Compose for local development, CI/CD pipeline definitions, IAM policy templates, environment variable templates.

### 6.2 Directory Structure

```
agentcore-demo-test1-infra/
├── .gitignore
├── README.md
├── docker-compose.yml           # Local dev stack (Temporal + PostgreSQL)
│
├── terraform/
│   ├── modules/
│   │   ├── vpc/                 # VPC, subnets, security groups
│   │   ├── rds/                 # PostgreSQL RDS instances
│   │   ├── s3/                  # S3 buckets + policies
│   │   ├── ec2/                 # EC2 (ADOT Collector)
│   │   ├── iam/                 # IAM roles + policies
│   │   ├── cloudwatch/          # Log groups + dashboards + alarms
│   │   └── bedrock/             # AgentCore runtime config
│   ├── environments/
│   │   ├── dev/
│   │   │   ├── main.tf
│   │   │   ├── variables.tf
│   │   │   └── terraform.tfvars
│   │   └── prod/
│   │       ├── main.tf
│   │       ├── variables.tf
│   │       └── terraform.tfvars
│   └── backend.tf               # S3 + DynamoDB remote state
│
├── github-actions/              # CI/CD workflow templates
│   ├── backend-ci.yml
│   ├── frontend-ci.yml
│   └── infra-deploy.yml
│
├── iam-policies/
│   ├── ec2-adot-collector.json
│   ├── agentcore-runtime.json
│   ├── temporal-worker.json
│   └── s3-least-privilege.json
│
└── env-templates/
    ├── .env.backend.example
    └── .env.frontend.example
```

---

## 7. Repository: `agentcore-demo-test1-docs`

### 7.1 Purpose
All documentation, deliverables, prompts, architecture diagrams, and runbooks. This is the **authoritative source** for all non-code project knowledge.

### 7.2 Directory Structure

```
agentcore-demo-test1-docs/
├── .gitignore
├── README.md
├── Makefile                     # Build targets (optional)
│
├── resources/                   # All docs from this project
│   ├── ARCHITECTURE_DATAFLOW_GUIDE.md
│   ├── Deliverable_0_PROJECT_CONTEXT.md
│   ├── Deliverable_1_Infrastructure_Cost_Report.md
│   ├── ... (Deliverable_2 through Deliverable_9)
│   ├── Prompt_00_Infra_Bootstrap.md
│   ├── Prompt_01_AgentCore_Deployment.md
│   ├── ... (Prompt_02 through Prompt_06)
│   ├── DEVELOPER_ONBOARDING.md
│   ├── DEMO_GELISTIRME_KILAVUZU.md
│   ├── PERSISTENT_FILESYSTEM_GUIDE.md
│   ├── PHASE_PLAN.md
│   ├── V5_IMPLEMENTATION_PLAN.md
│   ├── V5_TELEMETRY_INTEGRATION_ANALYSIS.md
│   ├── REPO_STRATEGY.md         # This file
│   ├── GIT_WORKFLOW.md          # Git workflow guide
│   ├── DELIVERABLE_MAPPING.md
│   ├── PROMPT_TESTING_SECTIONS.md
│   ├── architecture_infographic.png
│   ├── dataflow_diagram.png
│   ├── telemetry_flow_detailed.png
│   └── agent_integration_full.png
│
├── runbooks/
│   ├── incident-response.md
│   ├── deployment-rollback.md
│   └── cost-optimization.md
│
├── decisions/                   # Architecture Decision Records (ADRs)
│   ├── ADR-001-hybrid-architecture.md
│   ├── ADR-002-dual-stream-a2a.md
│   ├── ADR-003-s3-zip-deployment.md
│   └── ADR-004-agentcore-persistent-filesystems.md
│
└── meetings/
    └── weekly-standup-template.md
```

---

## 8. Shared Resources

### 8.1 API Contract Sharing

The backend exposes an **OpenAPI 3.1 spec** at `/openapi.json`. The frontend generates its TypeScript client from this spec.

**Workflow:**
1. Backend PR adds a new endpoint → updates `openapi.json`
2. Backend PR is merged to `main`
3. Frontend team runs `pnpm generate-api` which fetches `openapi.json` from staging
4. Frontend PR uses the new generated client types

**No shared source tree.** The OpenAPI spec is the contract.

### 8.2 Environment Variable Templates

The `infra` repo owns `.env.*.example` templates. Each repo copies the relevant template to `.env.local` and fills in secrets.

| Repository | Template Source | Secret Management |
|---|---|---|
| `backend` | `infra/env-templates/.env.backend.example` | AWS Secrets Manager (prod), `.env.local` (dev) |
| `frontend` | `infra/env-templates/.env.frontend.example` | `.env.local` only (no secrets in browser) |
| `infra` | Terraform variables + `.tfvars` | AWS Secrets Manager, Terraform Cloud |

### 8.3 Cross-Repo Issue Tracking

Use a **GitHub/GitLab project board** spanning all four repos. Labels:
- `repo:backend`, `repo:frontend`, `repo:infra`, `repo:docs`
- `cross-repo` — for changes affecting multiple repos
- `breaking-change` — for API contract changes

---

## 9. Dockerfile Strategy

### 9.1 Backend Dockerfile (`agentcore-demo-test1-backend/Dockerfile`)

```dockerfile
# ---------------------------------------------------------------------------
# agentcore-demo-test1-backend/Dockerfile
# Stage 1: Builder — install dependencies with uv
# Stage 2: Runtime — FastAPI server + Temporal worker
# ---------------------------------------------------------------------------

# ---------- STAGE 1: Builder ----------
FROM ghcr.io/astral-sh/uv:python3.11-bookworm-slim AS builder

WORKDIR /app

# Copy project files
COPY pyproject.toml uv.lock ./
COPY src/ ./src/
COPY agents/ ./agents/

# Create virtualenv and install dependencies
RUN uv venv /app/.venv && \
    uv pip install --no-cache -e ".[dev]"

# ---------- STAGE 2: Runtime ----------
FROM python:3.11-slim-bookworm AS runtime

WORKDIR /app

# Copy virtualenv from builder
COPY --from=builder /app/.venv /app/.venv
ENV PATH="/app/.venv/bin:$PATH"

# Copy source code
COPY --from=builder /app/src ./src
COPY --from=builder /app/agents ./agents

# Non-root user
RUN useradd -m -u 1000 appuser && chown -R appuser:appuser /app
USER appuser

# Health check
HEALTHCHECK --interval=30s --timeout=5s --start-period=5s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')"

# Default: FastAPI server (override CMD for worker)
EXPOSE 8000
CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000", "--loop", "uvloop", "--http", "httptools"]
```

### 9.2 Frontend Dockerfile (`agentcore-demo-test1-frontend/Dockerfile`)

```dockerfile
# ---------------------------------------------------------------------------
# agentcore-demo-test1-frontend/Dockerfile
# Stage 1: Dependencies — pnpm install
# Stage 2: Builder — next build
# Stage 3: Runtime — next start (standalone output)
# ---------------------------------------------------------------------------

# ---------- STAGE 1: Dependencies ----------
FROM node:20-slim AS deps
WORKDIR /app

# Install pnpm
RUN corepack enable && corepack prepare pnpm@9.0.0 --activate

COPY package.json pnpm-lock.yaml ./
RUN pnpm install --frozen-lockfile

# ---------- STAGE 2: Builder ----------
FROM node:20-slim AS builder
WORKDIR /app

RUN corepack enable && corepack prepare pnpm@9.0.0 --activate

COPY --from=deps /app/node_modules ./node_modules
COPY . .

# Build with standalone output (no Node.js needed at runtime)
ENV NEXT_TELEMETRY_DISABLED=1
RUN pnpm build

# ---------- STAGE 3: Runtime ----------
FROM node:20-slim AS runtime
WORKDIR /app

ENV NODE_ENV=production
ENV NEXT_TELEMETRY_DISABLED=1

# Copy standalone output
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
COPY --from=builder /app/public ./public

# Non-root user
RUN useradd -m -u 1000 appuser && chown -R appuser:appuser /app
USER appuser

EXPOSE 3000
ENV PORT=3000
ENV HOSTNAME="0.0.0.0"
CMD ["node", "server.js"]
```

### 9.3 Agent Docker — NOT USED

**There is NO Dockerfile for agents.** Agents deploy via S3 ZIP using `agentcore configure/deploy`. See `PERSISTENT_FILESYSTEM_GUIDE.md` for details.

```bash
# Correct way to deploy Agent 1
cd agents/transcriber
agentcore configure -e main.py --protocol HTTP --alias agent_1_transcriber-v1
agentcore deploy
```

### 9.4 Docker Compose (Local Dev)

Each repo has its own `docker-compose.yml` for local development. The backend `docker-compose.yml` includes Temporal Server + PostgreSQL. The frontend `docker-compose.yml` is Next.js dev server only.

To run the full stack locally:
```bash
# Terminal 1: Infrastructure (Temporal + PostgreSQL)
cd agentcore-demo-test1-infra && docker-compose up

# Terminal 2: Backend
cd agentcore-demo-test1-backend && uv run uvicorn src.main:app --reload

# Terminal 3: Temporal Worker
cd agentcore-demo-test1-backend && uv run python -m src.temporal.worker

# Terminal 4: Frontend
cd agentcore-demo-test1-frontend && pnpm dev
```

---

## 10. CI/CD Pipeline Separation

### 10.1 Backend CI (`.github/workflows/backend-ci.yml`)

```yaml
# Runs on: agentcore-demo-test1-backend repo
# Triggers: push to main, pull_request

name: Backend CI
on:
  push:
    branches: [main, dev]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v3
        with:
          version: "0.5.0"
      - run: uv python install 3.11
      - run: uv sync --all-extras --dev
      - run: uv run pytest tests/ -v --cov=src --cov-report=xml
      - run: uv run ruff check src/
      - run: uv run ruff format --check src/

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: docker build -t agentcore-demo-test1-backend:${{ github.sha }} .
      # On main: SCP image tarball to EC2 + docker compose pull on the host.
      # No ECR: agents deploy via `agentcore configure/deploy` (S3 ZIP) per PCD §15.1;
      # backend/frontend images live on the EC2 host directly.
```

### 10.2 Frontend CI (`.github/workflows/frontend-ci.yml`)

```yaml
# Runs on: agentcore-demo-test1-frontend repo
# Triggers: push to main, pull_request

name: Frontend CI
on:
  push:
    branches: [main, dev]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v3
        with:
          version: 9
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'pnpm'
      - run: pnpm install --frozen-lockfile
      - run: pnpm lint
      - run: pnpm type-check
      - run: pnpm test
      - run: pnpm build
```

### 10.3 Infra Deploy (`.github/workflows/infra-deploy.yml`)

```yaml
# Runs on: agentcore-demo-test1-infra repo
# Triggers: manual (workflow_dispatch) or scheduled

name: Infra Deploy
on:
  workflow_dispatch:
    inputs:
      environment:
        type: choice
        options: [dev, prod]

jobs:
  plan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
      - run: terraform init
      - run: terraform plan -var-file=environments/${{ inputs.environment }}/terraform.tfvars
```

---

## 11. Cross-Repo Development Workflow

### 11.1 Starting a New Feature

1. Create a cross-repo tracking issue in the **docs** repo with label `cross-repo`
2. Create branches in each affected repo:
   ```bash
   # Backend
   cd agentcore-demo-test1-backend
   git checkout -b feature/ABC-123-hitl-timeout
   
   # Frontend (if UI changes needed)
   cd agentcore-demo-test1-frontend
   git checkout -b feature/ABC-123-hitl-timeout
   ```
3. Develop independently in each repo
4. Open PRs in each repo, referencing the cross-repo issue
5. Merge in dependency order: backend → frontend (frontend depends on backend API)

### 11.2 API Contract Changes

1. Backend PR includes `openapi.json` update
2. Backend PR merged first
3. Frontend team regenerates API client:
   ```bash
   cd agentcore-demo-test1-frontend
   pnpm generate-api   # Fetches openapi.json from staging
   ```
4. Frontend PR uses new types

### 11.3 Hotfix Process

1. Branch from `main` in the affected repo only:
   ```bash
   git checkout -b hotfix/critical-bug
   ```
2. Fix, test, PR, merge to `main`
3. Deploy independently
4. Cherry-pick to `dev` if needed:
   ```bash
   git checkout dev
   git cherry-pick <hotfix-commit>
   ```

### 11.4 Release Coordination

All four repos tag releases independently with **semver**:
- `backend`: `v1.2.3`
- `frontend`: `v2.1.0`
- `infra`: `v1.0.4`
- `docs`: `v2024.05.14`

**Release compatibility matrix** (documented in docs repo):

| Backend | Frontend | Infra | Compatible |
|---|---|---|---|
| v1.2.x | v2.1.x | v1.0.x | Yes |
| v1.3.0 | v2.0.x | v1.0.x | No (API breaking change) |

---

## 12. Summary

| Aspect | Decision |
|---|---|
| **Repo count** | 4 separate repos |
| **Backend** | `agentcore-demo-test1-backend` — Python, uv, FastAPI, Temporal |
| **Frontend** | `agentcore-demo-test1-frontend` — TypeScript, pnpm, Next.js, CopilotKit |
| **Infra** | `agentcore-demo-test1-infra` — Terraform, Docker Compose, CI/CD |
| **Docs** | `agentcore-demo-test1-docs` — Markdown, diagrams, runbooks, deliverables |
| **Shared code** | None — API contract via OpenAPI spec |
| **Agents** | S3 ZIP deploy (NO Docker, NO ECR) |
| **CI/CD** | Independent per repo |
| **Branching** | `main` + `dev` + feature branches (see GIT_WORKFLOW.md) |
