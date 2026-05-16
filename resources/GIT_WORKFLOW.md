# Git Workflow: agentcore-demo-test1

**Date:** 2026-05-14
**Classification:** Development Process — Mandatory
**Applies To:** All repositories (`backend`, `frontend`, `infra`, `docs`)

---

## Table of Contents

1. [Branch Strategy](#1-branch-strategy)
2. [Repository Setup (Day 0)](#2-repository-setup-day-0)
3. [Daily Development Process](#3-daily-development-process)
4. [Commit Rules](#4-commit-rules)
5. [Pull Request Process](#5-pull-request-process)
6. [Gate-Based Commit Points](#6-gate-based-commit-points)
7. [Release Process](#7-release-process)
8. [Emergency Hotfix](#8-emergency-hotfix)
9. [Multi-Repo Coordination](#9-multi-repo-coordination)

---

## 1. Branch Strategy

### 1.1 Branch Types

```
main              Production-ready code. Protected — requires PR + review.
  |
  +-- dev         Integration branch. All feature branches merge here first.
  |   |
  |   +-- feature/ABC-123-description    Individual feature branches
  |   +-- feature/ABC-456-description
  |   +-- bugfix/ABC-789-description
  |   +-- refactor/module-name
  |
  +-- hotfix/critical-description        Emergency fixes from main
```

### 1.2 Branch Rules

| Branch | Protection | PR Required | Reviews | CI Must Pass |
|---|---|---|---|---|
| `main` | Enforced | Yes (2 reviewers) | 2 | Yes |
| `dev` | Enforced | Yes (1 reviewer) | 1 | Yes |
| `feature/*` | None | No | 0 | No (but recommended) |
| `hotfix/*` | None | Yes (to `main`) | 2 | Yes |

### 1.3 Branch Naming

```bash
# Features
feature/ABC-123-add-hitl-timeout      # Issue tracker ID + description
feature/ABC-456-improve-otel-spans

# Bug fixes
bugfix/ABC-789-fix-payload-validation
bugfix/critical-memory-leak

# Refactors
refactor/workflow-state-machine
refactor/split-activities-module

# Documentation (docs repo only)
docs/update-architecture-diagram

# Infrastructure (infra repo only)
infra/add-cloudwatch-alarm
```

---

## 2. Repository Setup (Day 0)

### 2.1 Create the Backend Repository

```bash
# Step 1: Create directory
mkdir agentcore-demo-test1-backend
cd agentcore-demo-test1-backend

# Step 2: Initialize git
git init

# Step 3: Create .gitignore
cat > .gitignore << 'EOF'
# Python
__pycache__/
*.py[cod]
.venv/
venv/

# uv.lock IS committed (reproducible builds — see PCD §5).
# Do NOT add uv.lock to .gitignore.

# Build
build/
dist/

# IDE
.idea/
.vscode/

# OS
.DS_Store

# AWS
*.zip
.env.local
EOF

# Step 4: Create initial directory structure
mkdir -p src/api src/temporal/workflow src/temporal/activities \
         src/services src/models \
         agents/transcriber agents/drafter agents/reviewer \
         tests/unit tests/integration tests/e2e \
         scripts

# Step 5: Create pyproject.toml
cat > pyproject.toml << 'EOF'
[project]
name = "agentcore-demo-test1-backend"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
    "fastapi>=0.115.0",
    "uvicorn[standard]>=0.32.0",
    "temporalio>=1.9.0",
    "boto3>=1.35.0",
    "psycopg[binary]>=3.2.0",
    "pydantic>=2.9.0",
    "pydantic-settings>=2.5.0",
    "opentelemetry-api>=1.28.0",
    "opentelemetry-sdk>=1.28.0",
    "opentelemetry-instrumentation-fastapi>=0.49b0",
    "opentelemetry-exporter-otlp>=1.28.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.3.0",
    "pytest-asyncio>=0.24.0",
    "pytest-cov>=6.0.0",
    "ruff>=0.7.0",
    "httpx>=0.27.0",
]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"
EOF

# Step 6: Create .python-version
echo "3.11" > .python-version

# Step 7: Initial commit
git add .
git commit -m "chore: initial project structure

- FastAPI + Temporal scaffold
- Agent directories (transcriber, drafter, reviewer)
- uv + .venv configuration
- pytest + ruff dev tooling

Refs: Gate-00"

# Step 8: Create remote (GitHub/GitLab) and push
git remote add origin git@github.com:${GITHUB_USER:-ugocen}/agentcore-demo-test1-backend.git
git branch -M main
git push -u origin main

# Step 9: Create dev branch
git checkout -b dev
git push -u origin dev
```

### 2.2 Create the Frontend Repository

```bash
mkdir agentcore-demo-test1-frontend
cd agentcore-demo-test1-frontend

# Initialize
git init

# .gitignore
cat > .gitignore << 'EOF'
node_modules/
.pnpm-store/
.next/
*.log
.env.local
.DS_Store
EOF

# package.json (minimal — use create-next-app in practice)
cat > package.json << 'EOF'
{
  "name": "agentcore-demo-test1-frontend",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "next lint",
    "type-check": "tsc --noEmit",
    "test": "vitest run"
  }
}
EOF

# Create directory structure
mkdir -p src/app/workspace/\[id\] src/app/api/upload \
         src/components/ui src/components/canvas src/components/chat src/components/hitl \
         src/hooks src/lib src/types src/store \
         tests/unit tests/integration tests/e2e \
         scripts public/assets

# Initial commit
git add .
git commit -m "chore: initial project structure

- Next.js 14+ App Router scaffold
- CopilotKit integration directories
- Canvas + Chat component structure
- pnpm + TypeScript configuration

Refs: Gate-00"

git remote add origin git@github.com:${GITHUB_USER:-ugocen}/agentcore-demo-test1-frontend.git
git branch -M main
git push -u origin main
git checkout -b dev
git push -u origin dev
```

### 2.3 Create the Docs Repository (for resources/)

```bash
mkdir agentcore-demo-test1-docs
cd agentcore-demo-test1-docs

git init

# .gitignore — resources/ is NOT gitignored here (this IS the docs repo)
cat > .gitignore << 'EOF'
*.zip
.DS_Store
EOF

mkdir -p resources runbooks decisions meetings

git add .
git commit -m "chore: initial docs structure

- resources/ directory for deliverables, prompts, guides
- runbooks/ for operational procedures
- decisions/ for ADRs

Refs: Setup"

git remote add origin git@github.com:${GITHUB_USER:-ugocen}/agentcore-demo-test1-docs.git
git branch -M main
git push -u origin main
```

### 2.4 Repository Summary After Setup

| Repository | URL | Default Branch | Branches |
|---|---|---|---|
| Backend | `agentcore-demo-test1-backend` | `main` | `main`, `dev` |
| Frontend | `agentcore-demo-test1-frontend` | `main` | `main`, `dev` |
| Infra | `agentcore-demo-test1-infra` | `main` | `main`, `dev` |
| Docs | `agentcore-demo-test1-docs` | `main` | `main` |

---

## 3. Daily Development Process

### 3.1 Morning: Start of Day

```bash
# 1. Pull latest changes from dev
cd agentcore-demo-test1-backend
git checkout dev
git pull origin dev

# 2. Sync dependencies (only if pyproject.toml changed)
uv sync --all-extras --dev

# 3. Run tests to verify clean state
uv run pytest tests/unit -q
```

### 3.2 During Development: Feature Work

```bash
# 1. Create feature branch from dev
git checkout dev
git checkout -b feature/ABC-123-add-clarification-timeout

# 2. Write code + tests
# Edit files...

# 3. Run tests continuously
uv run pytest tests/unit/test_workflow.py -v

# 4. Commit frequently (see Section 4 for commit format)
git add src/temporal/workflow/brd_workflow.py
git commit -m "feat: add 15-minute HITL timeout timer

- asyncio.Timer for clarification timeout
- Escalate to proceed_with_best_effort on expiry
- Configurable via CLARIFICATION_TIMEOUT_SECONDS env var

Refs: ABC-123"

# 5. Push branch to remote
git push -u origin feature/ABC-123-add-clarification-timeout
```

### 3.3 End of Day

```bash
# 1. Ensure all changes are committed
git status

# 2. Push current branch
git push

# 3. Create PR if feature is complete
# (See Section 5)
```

### 3.4 Switching Between Repos

When working on a cross-repo feature (e.g., backend API change + frontend UI update):

```bash
# Use consistent branch names across repos
cd agentcore-demo-test1-backend
git checkout -b feature/ABC-123-unified-search

cd ../agentcore-demo-test1-frontend
git checkout -b feature/ABC-123-unified-search

# Work in both repos independently
# Each repo has its own PR
# Both PRs reference the same issue: ABC-123
```

---

## 4. Commit Rules

### 4.1 Commit Message Format

```
<type>: <subject>

<body>

Refs: <issue-id>
```

### 4.2 Commit Types

| Type | Use For | Example |
|---|---|---|
| `feat` | New feature | `feat: add HITL timeout timer` |
| `fix` | Bug fix | `fix: correct payload validation logic` |
| `test` | Tests only | `test: add workflow state machine tests` |
| `refactor` | Code restructure | `refactor: split monolithic activity module` |
| `docs` | Documentation | `docs: update architecture diagram` |
| `chore` | Maintenance | `chore: update dependencies` |
| `infra` | Infrastructure | `infra: add CloudWatch alarm` |
| `agent` | Agent code | `agent: add PII detection tool to reviewer` |

### 4.3 Commit Body Rules

- Each commit must be self-contained (compiles/passes tests)
- Explain WHAT and WHY, not HOW (the code shows HOW)
- Reference the issue tracker ID

### 4.4 Example Commits

```bash
# Good commit — clear, self-contained
git commit -m "feat: emit V5 payload after transcription

- Build 8-block payload with Status, Resources, Timing
- Add validate_payload() helper for schema checking
- Emit payload as OpenTelemetry span event
- Add unit test for payload structure validation

Refs: ABC-456"

# Bad commit — vague, no context
git commit -m "update code"
```

### 4.5 Commit Frequency

| Scenario | Frequency | Rule |
|---|---|---|
| Normal development | Every 15-30 minutes | Commit working units |
| Complex refactor | Logical checkpoints | Commit when tests pass |
| End of workday | Always commit | Never leave uncommitted work |
| Gate completion | Immediate | Commit right after gate passes |

---

## 5. Pull Request Process

### 5.1 Creating a PR

```bash
# 1. Push branch
git push -u origin feature/ABC-123-description

# 2. Open PR via CLI (GitHub CLI)
gh pr create \
  --base dev \
  --title "feat: ABC-123 add clarification timeout" \
  --body "## Changes
- Add 15-minute HITL timeout timer
- Escalate to proceed_with_best_effort on expiry
- Add unit tests for timeout logic

## Testing
- [x] Unit tests pass
- [x] Integration tests pass
- [x] Manual testing completed

## Related
Refs: ABC-123"
```

### 5.2 PR Checklist

Every PR must include:

- [ ] **Title** follows format: `type: description`
- [ ] **Description** explains WHAT, WHY, and HOW
- [ ] **Tests** added or updated
- [ ] **CI passes** (all checks green)
- [ ] **Reviewers** assigned (1 for dev, 2 for main)
- [ ] **Issue reference** in description

### 5.3 PR Review Rules

| Target Branch | Required Reviews | Who Can Approve |
|---|---|---|
| `dev` | 1 | Any team member |
| `main` | 2 | Tech Lead + relevant domain expert |
| `hotfix/*` | 2 | Tech Lead + relevant domain expert |

### 5.4 Merging

```bash
# After PR is approved and CI passes:
# Use "Squash and Merge" for feature branches
# Use "Create a Merge Commit" for dev → main

# After merge, delete branch
git branch -d feature/ABC-123-description
git push origin --delete feature/ABC-123-description
```

---

## 6. Gate-Based Commit Points

Each Gate in the development process has **mandatory commit points**. These are non-negotiable — they ensure the codebase is always in a known, working state.

### 6.1 Commit Points by Gate

| Gate | Commit Point | Branch | Commit Message Pattern |
|---|---|---|---|
| Gate-00 | After infra bootstrap | `dev` | `chore: Gate-00 infra bootstrap complete` |
| Gate-01 | After AgentCore deploy | `dev` | `agent: Gate-01 AgentCore S3 ZIP deploy` |
| Gate-02 | After telemetry wired | `dev` | `feat: Gate-02 OTel + ADOT telemetry` |
| Gate-03 | After backend foundation | `dev` | `feat: Gate-03 FastAPI + Temporal scaffold` |
| Gate-04 | After workflow works | `dev` | `feat: Gate-04 Temporal workflow + HITL` |
| Gate-05 | After frontend works | `dev` | `feat: Gate-05 Next.js + CopilotKit UI` |
| Gate-06 | After E2E passes | `dev` | `test: Gate-06 E2E integration complete` |
| Pre-release | All gates done | `main` | `release: v0.1.0 all gates complete` |

### 6.2 Gate Commit Procedure

```bash
# Example: Gate-03 completion
cd agentcore-demo-test1-backend
git checkout dev

# Ensure all changes are staged
git status

# Run full test suite
uv run pytest tests/ -v

# Commit with gate marker
git add .
git commit -m "feat: Gate-03 backend foundation complete

- FastAPI app with workflow endpoints
- Temporal client + worker scaffold
- S3 claim-check handler
- OpenTelemetry instrumentation
- All unit tests passing

Gate: 03/06
Refs: DEMO-PLAN"

# Push to dev
git push origin dev

# Tag the gate (optional but recommended)
git tag -a gate-03 -m "Gate 03 complete: Backend foundation"
git push origin gate-03
```

### 6.3 Gate Tag Convention

```bash
# Lightweight tags for gate milestones
git tag gate-00  # Infra bootstrap
git tag gate-01  # AgentCore deploy
git tag gate-02  # Telemetry
git tag gate-03  # Backend foundation
git tag gate-04  # Workflow
git tag gate-05  # Frontend
git tag gate-06  # E2E integration

# Push all tags
git push origin --tags
```

---

## 7. Release Process

### 7.1 Versioning

All repos use **Semantic Versioning** (`MAJOR.MINOR.PATCH`):
- **MAJOR**: Breaking API changes
- **MINOR**: New features, backward compatible
- **PATCH**: Bug fixes

### 7.2 Release Steps

```bash
# Step 1: Ensure dev is stable
git checkout dev
git pull origin dev
uv run pytest tests/ -v  # All tests pass

# Step 2: Merge dev → main
git checkout main
git pull origin main
git merge --no-ff dev -m "release: v0.2.0 merge dev → main"

# Step 3: Version bump (edit version in pyproject.toml / package.json)
vim pyproject.toml  # Update version = "0.2.0"
git add pyproject.toml
git commit -m "chore: bump version to 0.2.0"

# Step 4: Tag release
git tag -a v0.2.0 -m "Release v0.2.0

Features:
- HITL clarification timeout
- Improved OTel span attributes
- Frontend evidence pack download

Full changelog: https://github.com/${GITHUB_USER:-ugocen}/agentcore-demo-test1-backend/compare/v0.1.0...v0.2.0"

# Step 5: Push
git push origin main
git push origin v0.2.0

# Step 6: Deploy (backend)
# CI/CD pipeline triggers on tag push
```

### 7.3 Release Checklist

- [ ] `dev` branch all tests passing
- [ ] `CHANGELOG.md` updated
- [ ] Version bumped in config file
- [ ] `main` branch merged from `dev`
- [ ] Git tag created and pushed
- [ ] CI/CD pipeline successful
- [ ] Staging deployment verified
- [ ] Production deployment approved

---

## 8. Emergency Hotfix

### 8.1 Hotfix Procedure

```bash
# Step 1: Create hotfix branch from main
git checkout main
git checkout -b hotfix/critical-memory-leak

# Step 2: Fix the bug (minimal change)
vim src/services/sse_bridge.py

# Step 3: Test
cd tests/unit && uv run pytest test_sse_bridge.py -v

# Step 4: Commit
git add src/services/sse_bridge.py tests/unit/test_sse_bridge.py
git commit -m "fix: resolve memory leak in SSE bridge

- Close unterminated generator on client disconnect
- Add cleanup handler for stale connections

Refs: HOTFIX-001"

# Step 5: Push and create PR to main
git push -u origin hotfix/critical-memory-leak
gh pr create --base main --title "hotfix: critical memory leak" --body "Emergency fix..."

# Step 6: After merge to main, cherry-pick the specific hotfix commit to dev
git checkout dev
HOTFIX_SHA=$(git rev-parse main)        # SHA of the merged hotfix tip on main
git cherry-pick "$HOTFIX_SHA"           # Apply that exact commit to dev

# Step 7: Clean up
git branch -d hotfix/critical-memory-leak
```

### 8.2 Hotfix Rules

- Hotfix branches ALWAYS branch from `main`, never from `dev`
- Hotfix PRs target `main` directly (bypass `dev`)
- After merge to `main`, cherry-pick the commit to `dev`
- 2 required reviewers for all hotfix PRs
- Post-incident review required within 48 hours

---

## 9. Multi-Repo Coordination

### 9.1 Cross-Repo Feature Workflow

When a feature touches multiple repos (e.g., new API endpoint + new UI component):

```bash
# === Backend Repo ===
cd agentcore-demo-test1-backend
git checkout dev
git checkout -b feature/ABC-123-new-endpoint

# Edit: src/api/workflows.py (new endpoint)
# Edit: src/models/workflow.py (new model)
# Edit: tests/unit/test_api.py (new test)

git add .
git commit -m "feat: add workflow retry endpoint

- POST /api/workflows/{id}/retry
- Validates workflow state before retry
- Returns new run_id on success

Refs: ABC-123"
git push -u origin feature/ABC-123-new-endpoint
gh pr create --base dev --title "feat: ABC-123 workflow retry"

# === Frontend Repo ===
cd ../agentcore-demo-test1-frontend
git checkout dev
git checkout -b feature/ABC-123-new-endpoint

# Edit: src/components/canvas/RetryButton.tsx (new component)
# Edit: src/hooks/useWorkflow.ts (add retry mutation)
# Edit: tests/unit/RetryButton.test.tsx

git add .
git commit -m "feat: add workflow retry UI

- RetryButton component with confirmation dialog
- useWorkflow hook retry mutation
- Toast notification on retry success

Depends-On: agentcore-demo-test1-backend#45
Refs: ABC-123"
git push -u origin feature/ABC-123-new-endpoint
gh pr create --base dev --title "feat: ABC-123 retry button"

# Merge backend PR FIRST (creates the API)
# Then merge frontend PR (uses the API)
```

### 9.2 Dependency Order

```
Backend API change  ──┐
                      ├──→ Backend PR merges FIRST
Infra change          ──┘
                      ↓
Frontend change  ─────→ Frontend PR merges SECOND
                      (depends on backend API being live)
```

### 9.3 Cross-Repo Issue References

Use the `Depends-On:` trailer in commit messages:

```bash
git commit -m "feat: add evidence pack download

- PDF generation endpoint
- ALCOA+ compliance metadata

Depends-On: agentcore-demo-test1-backend#45
Refs: ABC-456"
```

---

## 10. Quick Reference Card

### Daily Commands

```bash
# Start of day
git checkout dev && git pull origin dev

# New feature
git checkout -b feature/ID-description
# ... work ...
git commit -m "feat: description

Body...

Refs: ID"
git push -u origin feature/ID-description

# End of day
git status && git push
```

### Emergency Commands

```bash
# Undo last commit (keep changes)
git reset --soft HEAD~1

# Undo last commit (discard changes)
git reset --hard HEAD~1

# Stash work temporarily
git stash push -m "WIP: description"
git stash pop

# Check what changed
git diff --stat
git log --oneline -10

# Fix last commit message
git commit --amend
```

### Gate Completion Checklist

```bash
# At each gate boundary:
[ ] All unit tests pass:     uv run pytest tests/unit -q
[ ] All integration tests:   uv run pytest tests/integration -q
[ ] Type checking:           (uv run mypy src/  OR  pnpm type-check)
[ ] Linting:                 uv run ruff check src/
[ ] Requirements updated:    (cat requirements.txt matches pyproject.toml)
[ ] Commit with gate marker: git commit -m "... Gate: 0N/06"
[ ] Pushed to dev branch:    git push origin dev
[ ] Tagged (optional):       git tag -a gate-0N -m "..."
```

---

*End of Git Workflow Guide*
