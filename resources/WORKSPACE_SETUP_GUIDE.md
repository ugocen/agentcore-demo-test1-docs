# Workspace Setup Guide: agentcore-demo-test1

**Date:** 2026-05-14
**Applies To:** VS Code and Antigravity
**Scope:** How to open and work with 4 separate repositories

---

## Table of Contents

1. [The Short Answer](#1-the-short-answer)
2. [VS Code: Multi-Root Workspace](#2-vs-code-multi-root-workspace)
3. [Antigravity: Multi-Project Setup](#3-antigravity-multi-project-setup)
4. [Terminal Workflow](#4-terminal-workflow)
5. [Directory Layout on Your Machine](#5-directory-layout-on-your-machine)
6. [Working with Cross-Repo Changes](#6-working-with-cross-repo-changes)
7. [Quick Reference](#7-quick-reference)

---

## 1. The Short Answer

**Use a SINGLE window with MULTI-ROOT workspace.**

Do NOT open 4 separate VS Code/Antigravity windows. Both editors support adding multiple folders to a single workspace. This gives you:
- One window, all 4 repos visible
- Cross-repo search (find in all repos at once)
- Unified terminal (switch between repos with `cd`)
- Single editor session for the entire project

---

## 2. VS Code: Multi-Root Workspace

### Step 1: Create Parent Directory

```bash
# Create a parent folder that HOLDS all 4 repos (it is NOT a git repo itself)
mkdir -p /Users/ugurgocen/projects/agentcore-demo-test1
cd /Users/ugurgocen/projects/agentcore-demo-test1

# Clone all 4 repos (or create them during Day 0)
git clone git@github.com:YOUR_GITHUB_USERNAME/agentcore-demo-test1-backend.git
git clone git@github.com:YOUR_GITHUB_USERNAME/agentcore-demo-test1-frontend.git
git clone git@github.com:YOUR_GITHUB_USERNAME/agentcore-demo-test1-infra.git
git clone git@github.com:YOUR_GITHUB_USERNAME/agentcore-demo-test1-docs.git

# Verify structure — 4 separate .git directories
ls -la */.git
# Should show: backend/.git, frontend/.git, infra/.git, docs/.git
```

### Step 2: Create Multi-Root Workspace File

```bash
cd /Users/ugurgocen/projects/agentcore-demo-test1

# Create the workspace file
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
      "**/*.egg-info": true
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
  },
  "extensions": {
    "recommendations": [
      "ms-python.python",
      "ms-python.vscode-pylance",
      "bradlc.vscode-tailwindcss",
      "esbenp.prettier-vscode",
      "dbaeumer.vscode-eslint",
      "hashicorp.terraform"
    ]
  },
  "launch": {
    "version": "0.2.0",
    "configurations": [
      {
        "name": "Backend: FastAPI",
        "type": "debugpy",
        "request": "launch",
        "module": "uvicorn",
        "args": ["src.main:app", "--reload", "--port", "8000"],
        "cwd": "${workspaceFolder:BACKEND}",
        "python": "${workspaceFolder:BACKEND}/.venv/bin/python"
      },
      {
        "name": "Frontend: Next.js",
        "type": "node",
        "request": "launch",
        "runtimeExecutable": "pnpm",
        "runtimeArgs": ["dev"],
        "cwd": "${workspaceFolder:FRONTEND}"
      }
    ]
  }
}
EOF
```

### Step 3: Open in VS Code

```bash
# From the parent directory:
cd /Users/ugurgocen/projects/agentcore-demo-test1
code agentcore-demo-test1.code-workspace

# Result: ONE VS Code window with 4 root folders in Explorer:
#   BACKEND/
#   FRONTEND/
#   INFRA/
#   DOCS/
```

### What You See in VS Code

```
EXPLORER
├── BACKEND/                    ← agentcore-demo-test1-backend
│   ├── src/
│   ├── agents/
│   ├── tests/
│   └── pyproject.toml
├── FRONTEND/                   ← agentcore-demo-test1-frontend
│   ├── src/
│   ├── package.json
│   └── next.config.js
├── INFRA/                      ← agentcore-demo-test1-infra
│   ├── terraform/
│   └── docker-compose.yml
└── DOCS/                       ← agentcore-demo-test1-docs
    ├── resources/
    │   ├── Prompt_00_*.md
    │   ├── Deliverable_*.md
    │   └── ...
    └── runbooks/
```

### Cross-Repo Search

Press `Ctrl+Shift+F` (or `Cmd+Shift+F` on Mac) — searches ALL 4 repos simultaneously. Results are grouped by repo name.

### Per-Repo Terminal

Open a terminal with `` Ctrl+` ``. Use `cd` to switch:

```bash
# Terminal 1: Backend work
cd /Users/ugurgocen/projects/agentcore-demo-test1/agentcore-demo-test1-backend
source .venv/bin/activate
uv run uvicorn src.main:app --reload

# Terminal 2: Frontend work
cd /Users/ugurgocen/projects/agentcore-demo-test1/agentcore-demo-test1-frontend
pnpm dev

# Terminal 3: Infra work
cd /Users/ugurgocen/projects/agentcore-demo-test1/agentcore-demo-test1-infra
docker-compose up

# Terminal 4: Docs
cd /Users/ugurgocen/projects/agentcore-demo-test1/agentcore-demo-test1-docs
```

### VS Code Status Bar Shows Active Repo

The status bar at the bottom of VS Code shows which git repo your current file belongs to:
- `main (BACKEND)` — when editing backend files
- `dev (FRONTEND)` — when editing frontend files
- You can switch branches per-repo from the status bar

---

## 3. Antigravity: Multi-Project Setup

Antigravity (Cursor/VS Code fork) supports the same multi-root workspace as VS Code.

### Method 1: Open Folder with All Repos

```bash
cd /Users/ugurgocen/projects/agentcore-demo-test1
# Open the PARENT directory (NOT individual repos)
antigravity .

# Then File → Add Folder to Workspace for each repo
```

### Method 2: Use the Same `.code-workspace` File

Antigravity reads VS Code workspace files natively:

```bash
cd /Users/ugurgocen/projects/agentcore-demo-test1
antigravity agentcore-demo-test1.code-workspace
```

### Antigravity-Specific Tips

- **Composer** (AI chat) sees ALL 4 repos simultaneously — you can ask "find all uses of `BedrockAgentCoreApp`" and it searches all repos
- **Tab Groups**: Create one tab group per repo for organized editing
- **@ Symbol Reference**: `Ctrl+Click` on symbols works cross-repo if you have TypeScript/Python language servers configured

---

## 4. Terminal Workflow

### Recommended: One Terminal Window, Multiple Tabs

Use a terminal multiplexer or multiple tabs:

```bash
# === Tab 1: Backend (always active during development)
cd /Users/ugurgocen/projects/agentcore-demo-test1/agentcore-demo-test1-backend
git checkout dev
git pull origin dev
source .venv/bin/activate
uv run uvicorn src.main:app --reload        # FastAPI on :8000

# === Tab 2: Temporal Worker (always active)
cd /Users/ugurgocen/projects/agentcore-demo-test1/agentcore-demo-test1-backend
source .venv/bin/activate
uv run python -m src.temporal.worker         # Worker polls Temporal

# === Tab 3: Frontend (active in FAZ 5+)
cd /Users/ugurgocen/projects/agentcore-demo-test1/agentcore-demo-test1-frontend
git checkout dev
git pull origin dev
pnpm dev                                     # Next.js on :3000

# === Tab 4: Infrastructure (on-demand)
cd /Users/ugurgocen/projects/agentcore-demo-test1/agentcore-demo-test1-infra
docker-compose up                            # Temporal Server + PostgreSQL

# === Tab 5: Agent Deployment (on-demand)
cd /Users/ugurgocen/projects/agentcore-demo-test1/agentcore-demo-test1-backend/agents/transcriber
agentcore configure -e main.py --protocol AGUI --alias transcriber-v1
agentcore deploy

# === Tab 6: Git / General
cd /Users/ugurgocen/projects/agentcore-demo-test1
# Use for cross-repo git operations, status checks, etc.
```

### Using tmux (Advanced)

```bash
# Create a named session
tmux new-session -s agentcore-demo -n backend
tmux new-window -n worker
tmux new-window -n frontend
tmux new-window -n infra
tmux new-window -n deploy

# Attach: tmux attach -t agentcore-demo
# Switch windows: Ctrl+B, then 0-4
```

---

## 5. Directory Layout on Your Machine

```
~/projects/                     # Your projects root
└── agentcore-demo-test1/       # PARENT DIRECTORY (NOT a git repo)
    ├── agentcore-demo-test1.code-workspace   # VS Code workspace file
    │
    ├── agentcore-demo-test1-backend/         # Git repo 1
    │   ├── .git/
    │   ├── .venv/               # Python virtualenv (gitignored)
    │   ├── src/
    │   ├── agents/
    │   ├── tests/
    │   ├── pyproject.toml
    │   └── Dockerfile           # FastAPI + Worker ONLY
    │
    ├── agentcore-demo-test1-frontend/        # Git repo 2
    │   ├── .git/
    │   ├── node_modules/        # pnpm packages (gitignored)
    │   ├── .next/               # Build output (gitignored)
    │   ├── src/
    │   ├── package.json
    │   └── Dockerfile
    │
    ├── agentcore-demo-test1-infra/           # Git repo 3
    │   ├── .git/
    │   ├── terraform/
    │   ├── docker-compose.yml
    │   └── iam-policies/
    │
    └── agentcore-demo-test1-docs/            # Git repo 4
        ├── .git/
        ├── resources/           # All deliverables, prompts, guides
        │   ├── Prompt_00_*.md
        │   ├── Prompt_01_*.md
        │   ├── ...
        │   ├── Deliverable_*.md
        │   ├── GIT_WORKFLOW.md
        │   ├── REPO_STRATEGY.md
        │   └── ...
        ├── runbooks/
        └── decisions/
```

**Key rule:** The parent directory `/Users/ugurgocen/projects/agentcore-demo-test1/` is NOT a git repo. It is just a container. Each subdirectory is an independent git repo with its own `.git/`, `main` branch, `dev` branch, and commit history.

---

## 6. Working with Cross-Repo Changes

### Scenario: Adding a new API endpoint + UI component

```bash
# === VS Code Explorer shows all 4 repos ===

# 1. Create feature branch in backend
# Click BACKEND/ in Explorer → Terminal → New Terminal
cd agentcore-demo-test1-backend
git checkout -b feature/new-endpoint

# Edit: BACKEND/src/api/workflows.py
# Edit: BACKEND/src/models/workflow.py
# Commit from VS Code Source Control panel (shows "BACKEND" context)

# 2. Create matching branch in frontend
# Click FRONTEND/ in Explorer → Terminal → New Terminal
cd agentcore-demo-test1-frontend
git checkout -b feature/new-endpoint

# Edit: FRONTEND/src/hooks/useWorkflow.ts
# Commit from VS Code Source Control panel (shows "FRONTEND" context)

# 3. Open PRs (one per repo) from GitHub web interface or gh CLI
```

### VS Code Source Control Panel

The Source Control panel (`Ctrl+Shift+G`) shows ALL 4 repos:

```
SOURCE CONTROL
├── BACKEND    [dev ▼]  +3 ~1 -0  (3 staged, 1 modified)
│   ├── M  src/api/workflows.py
│   └── A  tests/test_new_endpoint.py
├── FRONTEND  [feature/new ▼]  +1 ~2 -0
│   ├── M  src/hooks/useWorkflow.ts
│   └── M  src/components/NewPanel.tsx
├── INFRA     [main ▼]  +0 ~0 -0
└── DOCS      [main ▼]  +0 ~0 -0
```

Each repo has its own commit box, staging area, and branch selector.

---

## 7. Quick Reference

| Task | Command/Action |
|------|---------------|
| **Open workspace** | `code agentcore-demo-test1.code-workspace` |
| **Switch repo in terminal** | `cd ../agentcore-demo-test1-frontend` |
| **Search all repos** | `Ctrl+Shift+F` (VS Code search) |
| **See git status all repos** | VS Code Source Control panel |
| **Run backend** | `uv run uvicorn src.main:app --reload` (in BACKEND tab) |
| **Run frontend** | `pnpm dev` (in FRONTEND tab) |
| **Run worker** | `uv run python -m src.temporal.worker` (in WORKER tab) |
| **Deploy agent** | `agentcore configure -e main.py --protocol AGUI && agentcore deploy` (in DEPLOY tab) |
| **Commit backend** | Stage in Source Control → type message → Ctrl+Enter |
| **Commit frontend** | Stage in Source Control → type message → Ctrl+Enter |
| **Switch branch** | Click branch name in Status Bar (bottom left) |

---

*End of Workspace Setup Guide*
