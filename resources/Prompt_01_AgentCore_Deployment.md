# Prompt 01: AgentCore Deployment — Phase 1 (Minimal Agents)

**MODE: AUTOMATIC — AI writes code directly, no user interaction needed**

## Reference Documents (READ THESE FIRST)

Before writing any code, read the following documents in `resources/` for the exact patterns and skeletons to follow:

| Priority | Document | Why Read It |
|----------|----------|-------------|
| **PRIMARY** | `resources/Deliverable_2_Reference_Strands_Agent_Code.md` | **AGENT CODE SKELETONS.** This is the MOST IMPORTANT reference. Contains the exact `BedrockAgentCoreApp` + `@app.entrypoint` pattern for all 3 agents, plus the V5 payload builder template. Read Sections 1 (Transcriber), 2 (Drafter), and 3 (Reviewer) before writing any agent code. |
| **REFERENCE** | `resources/Deliverable_0_PROJECT_CONTEXT.md` | Agent architecture decisions, V5 payload schema (Section 6), and tech stack (Section 11). Read Section 6 for the exact 8-block payload structure every agent must emit. |
| **REFERENCE** | `resources/Deliverable_6_AWS_Operations_Guide.md` | S3 ZIP deployment procedure (Sections 3 and 5). Read for the exact `agentcore configure/deploy` commands and IAM permissions needed. |
| **REFERENCE** | `resources/PERSISTENT_FILESYSTEM_GUIDE.md` | S3 Filesystem (AgentCore Persistent Filesystems) setup for claim-check pattern. Read if implementing the POSIX file-based claim-check (vs boto3 S3 API). |

> **How to find these documents:** They are in the `resources/` folder of the `agentcore-demo-test1-docs` repo (or the `resources/` folder if docs are copied locally). All prompt and deliverable files follow the naming convention: `Prompt_NN_Descriptive_Name.md` and `Deliverable_N_Descriptive_Name.md`.

---

## YOU ARE A LOW-THINKING EXECUTOR. DO NOT IMPROVISE. DO NOT SKIP STEPS.

Your job: Execute every step sequentially. Do not make decisions. Do not optimize. Do not add features.
If a step says "create file X with content Y", create EXACTLY that file with EXACTLY that content.
If a step fails, STOP and report the exact error. Do not proceed.

---

## CRITICAL RULES (NON-NEGOTIABLE)

- **Deployment method: `direct_code_deploy` (the toolkit default).** You submit Python source; the toolkit handles ZIP → S3 → CodeBuild → container image → runtime registration. You do NOT write a Dockerfile or push to ECR yourself. The runtime IS container-backed under the hood (verifiable via `agentcore status`); earlier drafts said "no container, no ECR" which described the DEVELOPER experience, not the implementation. Canonical sequence (script-friendly, no interactive prompts):
  ```bash
  agentcore configure -e main.py --protocol HTTP --non-interactive
  agentcore deploy --env S3_BUCKET_ARTIFACTS="$S3_BUCKET_ARTIFACTS" --auto-update-on-conflict
  ```
  Subsequent code edits → re-run `agentcore deploy --auto-update-on-conflict`. To verify state: `agentcore status`. To tear down: `agentcore destroy`.
- **Python environment isolation:** NEVER use global `pip install`. ALWAYS use `uv` with `.venv`:
  ```bash
  uv venv .venv
  source .venv/bin/activate
  uv pip install -r requirements.txt
  ```
- **Node.js environment isolation:** NEVER use `npm install -g`. ALWAYS install locally via `npm install`.
- **All code comments MUST be written in English only.**
- **Git workflow:** After every GATE PASS, commit and push:
  ```bash
  git add -A
  git commit -m "feat: Phase 1 Gate X passed — description"
  git push origin main
  ```
  Keep `.gitignore` and `requirements.txt` updated at all times.
- If ANY step fails, STOP and report the exact error. Do NOT proceed.

---

## CONTEXT

You are building the FIRST phase of a multi-phase project called `agentcore-demo-test1`.
This project deploys 3 agents (2 Strands + 1 LangGraph) to AWS Bedrock AgentCore via the starter toolkit's **`direct_code_deploy` mode** (the default).

- **Phase 1 (THIS PROMPT)**: Get 3 minimal agents deployed and responding to `/ping` and `/invocations`. NOTHING ELSE.
- **Phase 2**: Add telemetry, 8-block payloads, and tool integrations.
- **Phase 3+**: HITL via Temporal, A2A, frontend integration.

**CRITICAL**: Phase 1 agents return placeholder 8-block payloads (per PCD §6) — full business logic + OTel arrives in Phase 2. Phase 1 succeeds when `agentcore status` shows all three runtimes Active and `invoke_agent_runtime` returns a valid placeholder payload.

### The `direct_code_deploy` flow

```
Developer code (main.py + requirements.txt + pyproject.toml)
        |
        v
  agentcore configure -e main.py --protocol HTTP --non-interactive
        |  (writes .bedrock_agentcore.yaml in the agent directory)
        v
  agentcore deploy --env KEY=VAL --auto-update-on-conflict
        |
        v
  Toolkit zips source → uploads to S3 staging bucket
                       (bedrock-agentcore-codebuild-sources-<acct>-<region>)
        |
        v
  AWS CodeBuild builds an ARM64 container image
        |
        v
  AgentCore Runtime registers the agent (Firecracker MicroVM per invocation)
        |
        v
  /invocations (external) → /invoke (internal, port 8080)
  /ping (external) → auto-handled
  /metadata, /capabilities, /metrics, /health (internal)
```

**Why S3 ZIP?**
- No Dockerfile to write or maintain
- No ECR repository to create or manage
- CodeBuild handles the ARM64 build automatically
- Faster iteration: change code → `agentcore configure` → `agentcore deploy`
- This is the RECOMMENDED method by AWS as of 2026

### What You Need Before Starting

1. AWS CLI configured. Verify with:
   ```bash
   aws sts get-caller-identity
   ```
   If this fails, STOP.

2. `bedrock-agentcore-starter-toolkit` installed (via uv, inside .venv):
   ```bash
   source .venv/bin/activate
   uv pip install bedrock-agentcore-starter-toolkit
   agentcore --version
   ```

3. S3 Files filesystem and access point created (from Phase 0). Verify:
   ```bash
   aws s3files describe-file-systems --query 'FileSystems[*].FileSystemId'
   ```

---

## STEP 0: AgentCore Bootstrap (First-Time Setup)

**This step runs ONLY once**, the first time you use AgentCore in a new AWS account.

### 0.1: Verify AWS CLI and Region

```bash
aws configure get region
# Must output: us-east-1

aws sts get-caller-identity --query Account --output text
# Saves your account ID for later steps
```

### 0.2: Install AgentCore Starter Toolkit

```bash
cd /Users/ugurgocen/projects/agentcore-demo-test1  # project root
uv venv .venv
source .venv/bin/activate
uv pip install bedrock-agentcore-starter-toolkit
```

Verify:
```bash
agentcore --version
# Expected: agentcore version x.y.z
```

**CHECKPOINT 0.1**: `agentcore --version` prints a version number.
If FAIL → STOP. Check that `.venv` is activated. Re-run `uv pip install`.

### 0.3: First-run auto-provisioning (handled by `agentcore deploy`)

> **There is no `agentcore bootstrap` command.** This section previously called a fictional `agentcore bootstrap --region us-east-1`. Skip that step entirely.
>
> What actually happens: the **first time** you run `agentcore deploy` (Step 6 below), the toolkit auto-provisions:
> 1. An S3 staging bucket `bedrock-agentcore-codebuild-sources-<account>-<region>`
> 2. A CodeBuild project for ARM64 image builds
> 3. An IAM execution role named `AmazonBedrockAgentCoreSDKRuntime-...`
>
> These are created lazily on demand, not in a separate bootstrap step. Your IAM principal needs `iam:CreateRole`, `iam:PutRolePolicy`, `s3:CreateBucket`, and `codebuild:CreateProject` permissions at the time of the first deploy (the EC2 instance role created in Phase 0 has these).

**CHECKPOINT 0.2 (revised):** Skip — there is nothing to verify in this step. The auto-provisioned resources appear after Step 6 (first deploy).

### 0.4: AgentCore Persistent Filesystem — DEFERRED (Optional)

> **Important correction (2026-05-15):** Earlier drafts of this prompt called fictional CLI commands `aws bedrock-agentcore create-filesystem` and `--filesystem-configurations` on `agentcore configure`. Neither exists. The real API surface is:
>
> | Operation | Real command |
> |---|---|
> | Create S3 Files filesystem | `aws s3files create-file-system` (separate service) |
> | Create access point | `aws s3files create-access-point` |
> | Create EFS filesystem (alternative) | `aws efs create-file-system` |
> | **Attach** filesystem to an existing agent runtime | `aws bedrock-agentcore-control update-agent-runtime --filesystem-configurations '[...]'` (**control-plane**, not data-plane) |
>
> Note the namespace difference: `bedrock-agentcore-control` (control plane, attach) vs `bedrock-agentcore` (data plane, invoke + memory only). The `agentcore configure` toolkit command has **no** filesystem flag — attachment is a separate post-deploy step.

**For Phase 1 and Phase 2, the filesystem is NOT required.** Agents use boto3 S3 directly for any IO they need (Transcribe output, BRD draft persistence, review report). The POSIX mount is purely an ergonomics/performance optimization for very large (>1 MB) payloads.

**If you want to enable the mount later** (e.g. before Phase 6 stress testing with realistic payloads), follow `PERSISTENT_FILESYSTEM_GUIDE.md`. The summarized flow is:

```bash
# 1. Create the S3 Files filesystem on top of the existing claimcheck bucket.
FS_ID=$(aws s3files create-file-system \
    --name "agentcore-demo-test1-claimcheck-fs" \
    --backing-bucket "$S3_BUCKET_CLAIMCHECK" \
    --query 'FileSystem.FileSystemId' --output text)

# 2. Create a POSIX access point.
AP_ID=$(aws s3files create-access-point \
    --file-system-id "$FS_ID" \
    --name "claimcheck-ap" \
    --posix-user Uid=1000,Gid=1000 \
    --query 'AccessPoint.AccessPointId' --output text)

# 3. Attach to an already-deployed runtime via the control plane.
aws bedrock-agentcore-control update-agent-runtime \
    --agent-runtime-id "$AGENTCORE_ID_TRANSCRIBER" \
    --filesystem-configurations "[{
      \"s3FilesAccessPoint\": {
        \"accessPointArn\": \"arn:aws:s3files:us-east-1:$ACCOUNT_ID:access-point/$AP_ID\",
        \"mountPath\": \"/mnt/sdlc-payloads-claimcheck\",
        \"readOnly\": false
      }
    }]"
```

**Skip this section for now.** Proceed to Step 1.

**CHECKPOINT 0.3 (revised):** No filesystem creation required for Phase 1. Just confirm `agentcore --version` works on both Mac and EC2.

---

## STEP 1: Create Project Skeleton and Git Repository

### 1.1: Create Directory Structure

```bash
cd /Users/ugurgocen/projects/agentcore-demo-test1

mkdir -p agents/{agent_1_transcriber,agent_2_drafter,agent_3_reviewer,common}
mkdir -p infra/scripts
mkdir -p tests/agents
```

### 1.2: Initialize Git Repository

```bash
cd /Users/ugurgocen/projects/agentcore-demo-test1
git init
GITHUB_USER="${GITHUB_USER:-ugocen}"
git remote add origin "git@github.com:${GITHUB_USER}/agentcore-demo-test1-backend.git" 2>/dev/null \
    || git remote set-url origin "git@github.com:${GITHUB_USER}/agentcore-demo-test1-backend.git"
```

### 1.3: Create `.gitignore`

```bash
cat > .gitignore <<'GITIGNORE'
# Python
__pycache__/
*.py[cod]
*$py.class
*.so
.Python
.venv/
venv/
env/
*.egg-info/
dist/
build/
*.egg
.pytest_cache/
.mypy_cache/
.coverage
htmlcov/

# Node.js
node_modules/
.next/
*.log
npm-debug.log*

# Environment
.env
.env.local
.env.*.local

# IDE
.vscode/
.idea/
*.swp
*.swo
*~

# OS
.DS_Store
Thumbs.db

# AWS
.vpc_env
.ec2_env
*.pem

# Temporary
*.tmp
*.bak
GITIGNORE
```

### 1.4: Commit Skeleton

```bash
git add -A
git commit -m "chore: initialize project skeleton with .gitignore and agent directories"
git push origin main
```

---

## STEP 2: Shared payload-builder stub

Before writing the agents, create the shared placeholder builder. The full implementation lands in Phase 2; here it returns a valid-shaped 8-block payload with `"Unavailable"` for fields the placeholder can't compute.

### 2a: Create `agents/common/__init__.py`

```python
# Marker file so `agents.common` is importable.
```

### 2b: Create `agents/common/payload_builder_stub.py`

```python
"""
Phase 1 placeholder for the 8-block payload builder.

Returns a structurally valid payload (all 8 top-level keys, Status echoes
inbound identifiers) with "Unavailable" for fields a placeholder cannot
compute. Replaced by the full implementation in Phase 2.

Reference: PAYLOAD_SCHEMA.md §3 (top-level shape), §4 (per-block fields).
"""
from __future__ import annotations
import time
from typing import Any


UNAVAILABLE = "Unavailable"


def build_placeholder_payload(
    request: dict[str, Any],
    agent_id: str,
    agent_version: str,
    started_at_ms: int,
    ended_at_ms: int | None = None,
) -> dict[str, Any]:
    """
    Build an 8-block placeholder response.

    The Status block echoes the inbound identifiers per PCD §7 / PAYLOAD_SCHEMA §4.
    All other blocks contain "Unavailable" markers; the full builder fills them in
    Phase 2.
    """
    end_ms = ended_at_ms if ended_at_ms is not None else int(time.time() * 1000)
    return {
        "status": {
            "code": "success",
            "http_status": 200,
            "message": f"{agent_id} placeholder invocation",
            "trace_id": request.get("trace_id", UNAVAILABLE),
            "workflow_template_id": request.get("workflow_template_id", UNAVAILABLE),
            "workflow_id": request.get("workflow_id", UNAVAILABLE),
            "agent_run_id": request.get("agent_run_id", UNAVAILABLE),
            "step_id": request.get("step_id", UNAVAILABLE),
            "error_code": None,
            "error_category": "none",
            "retryable": False,
            "custom_metadata": {"phase": "1-placeholder"},
        },
        "resources": {
            "cost_reporting_model": "token_based",
            "token_based": {
                "model_used": UNAVAILABLE,
                "tokens": {"input": 0, "output": 0, "total_outer": 0},
                "internal_llm_calls": 0,
                "total_tokens_all_calls": 0,
            },
        },
        "timing": {
            "total_elapsed_ms": max(0, end_ms - started_at_ms),
            "queue_duration_ms": 0,
            "agent_active_ms": max(0, end_ms - started_at_ms),
            "waiting_for_tools_ms": 0,
            "waiting_for_human_ms": 0,
            "timing_breakdown": UNAVAILABLE,
            "critical_path": UNAVAILABLE,
        },
        "financial": {
            "task_type": request.get("task", {}).get("task_type", UNAVAILABLE),
            "complexity_tier": request.get("task", {}).get("complexity_tier", "simple"),
            "manual_baseline_hours": UNAVAILABLE,
            "agent_active_hours": 0.0,
            "human_review_hours_estimated": None,
            "agent_cost_usd": 0.0,
            "retry_cost_usd": 0.0,
            "currency": "usd",
        },
        "artifacts": [],
        "quality": {
            "confidence": 0.0,
            "initial_attempt_success": True,
            "total_attempts": 1,
            "self_corrections": 0,
            "quality_concerns": [],
            "unsupported_claims": [],
            "completeness_assessment": "minimal",
        },
        "tool_calls": {
            "calls": [],
            "tool_summary": {
                "total_tool_calls": 0,
                "total_tool_duration_ms": 0,
                "tools_as_pct_of_elapsed": 0,
            },
        },
        "risk": {
            "pii_detected": False,
            "pii_filtered_count": 0,
            "secrets_detected": False,
            "policy_violations": 0,
            "sensitivity_compliant": True,
            "missing_inputs": [],
            "unsupported_claims": [],
            "compliance_checks": [],
        },
    }
```

---

## STEP 3: Agent 1 — Transcriber (Strands)

### 3a: Create `agents/agent_1_transcriber/main.py`

Five endpoints per PCD §8 (`/invoke`, `/health`, `/metadata`, `/capabilities`, `/metrics`). Returns a valid 8-block placeholder payload that echoes identifiers in Block 1.

```python
"""
Agent 1: Transcriber (Strands) — S3 ZIP deployed to AgentCore.

Phase 1 scope: skeleton with all five required endpoints and a placeholder
8-block response. Real AWS Transcribe integration arrives in Phase 2.
"""
from __future__ import annotations
import time
from bedrock_agentcore.runtime import BedrockAgentCoreApp
from agents.common.payload_builder_stub import build_placeholder_payload

AGENT_ID = "agent_1_transcriber"
AGENT_VERSION = "1.0.0"

app = BedrockAgentCoreApp()


@app.entrypoint
def invoke(request: dict, context: dict | None = None) -> dict:
    """Entrypoint for AgentCore Runtime (= POST /invoke per PCD §8)."""
    started_at_ms = int(time.time() * 1000)
    # Phase 1: no business logic — just acknowledge and return the placeholder shape.
    return build_placeholder_payload(
        request=request,
        agent_id=AGENT_ID,
        agent_version=AGENT_VERSION,
        started_at_ms=started_at_ms,
    )


# /health, /metadata, /capabilities, /metrics are mounted on the underlying
# Starlette/FastAPI app exposed by BedrockAgentCoreApp.
@app.fastapi.get("/health")
def health():
    return {"status": "healthy", "agent_id": AGENT_ID, "version": AGENT_VERSION}


@app.fastapi.get("/metadata")
def metadata():
    return {
        "agent_id": AGENT_ID,
        "agent_version": AGENT_VERSION,
        "framework": "strands",
        "tier": "first_party",
        "owner": "agentcore-demo-test1-team",
        "supported_task_types": ["audio_transcription"],
    }


@app.fastapi.get("/capabilities")
def capabilities():
    return {
        "inputs": {"artifact_refs": ["s3://*/*.mp3", "s3://*/*.wav", "s3://*/*.webm", "s3://*/*.m4a"]},
        "outputs": {"artifact_type": "design_document", "artifact_subtype": "transcript", "format": "json"},
        "tools": ["aws-transcribe"],
        "execution_modes": ["invoke"],
    }


@app.fastapi.get("/metrics")
def metrics():
    # Phase 1: trivial in-process counters; ADOT pipeline added in Phase 2.
    return {"requests_total": 0, "errors_total": 0}
```

### 3b: Create `agents/agent_1_transcriber/requirements.txt`

```text
strands-agents
bedrock-agentcore
```

### 3c: Create `agents/agent_1_transcriber/pyproject.toml`

```toml
[project]
name = "agent_1_transcriber"
version = "1.0.0"
requires-python = ">=3.11"
dependencies = ["strands-agents", "bedrock-agentcore"]
```

**GATE**: All 3 files exist and are non-empty. `main.py` registers exactly 5 endpoints (`/invoke` via `@app.entrypoint` + 4 `/health`/`/metadata`/`/capabilities`/`/metrics`).

---

## STEP 4: Agent 2 — Drafter (Strands)

### 4a: Create `agents/agent_2_drafter/main.py`

```python
"""
Agent 2: BRD Drafter (Strands) — S3 ZIP deployed to AgentCore.

Phase 1 scope: skeleton with all five required endpoints and a placeholder
8-block response. Real Bedrock + HITL clarification logic arrives in Phase 2.
"""
from __future__ import annotations
import time
from bedrock_agentcore.runtime import BedrockAgentCoreApp
from agents.common.payload_builder_stub import build_placeholder_payload

AGENT_ID = "agent_2_drafter"
AGENT_VERSION = "1.0.0"

app = BedrockAgentCoreApp()


@app.entrypoint
def invoke(request: dict, context: dict | None = None) -> dict:
    started_at_ms = int(time.time() * 1000)
    return build_placeholder_payload(
        request=request,
        agent_id=AGENT_ID,
        agent_version=AGENT_VERSION,
        started_at_ms=started_at_ms,
    )


@app.fastapi.get("/health")
def health():
    return {"status": "healthy", "agent_id": AGENT_ID, "version": AGENT_VERSION}


@app.fastapi.get("/metadata")
def metadata():
    return {
        "agent_id": AGENT_ID,
        "agent_version": AGENT_VERSION,
        "framework": "strands",
        "tier": "first_party",
        "owner": "agentcore-demo-test1-team",
        "supported_task_types": ["brd_generation"],
    }


@app.fastapi.get("/capabilities")
def capabilities():
    return {
        "inputs": {"artifact_refs": ["s3://*/transcripts/*.json"], "human_input": "supported"},
        "outputs": {"artifact_type": "design_document", "artifact_subtype": "functional_spec", "format": "md"},
        "tools": ["aws-bedrock"],
        "execution_modes": ["invoke", "resume_with_human_input"],
    }


@app.fastapi.get("/metrics")
def metrics():
    return {"requests_total": 0, "errors_total": 0, "clarifications_total": 0}
```

### 4b: Create `agents/agent_2_drafter/requirements.txt`

```text
strands-agents
bedrock-agentcore
```

### 4c: Create `agents/agent_2_drafter/pyproject.toml`

```toml
[project]
name = "agent_2_drafter"
version = "1.0.0"
requires-python = ">=3.11"
dependencies = ["strands-agents", "bedrock-agentcore"]
```

**GATE**: All 3 files exist. Endpoint count = 5.

---

## STEP 5: Agent 3 — Reviewer (LangGraph 4-node)

### 5a: Create `agents/agent_3_reviewer/nodes.py`

The four-node review graph (per PCD §15.3). Phase 1 has placeholder bodies; full implementations land in Phase 2.

```python
"""LangGraph nodes for the Reviewer agent (Phase 1 placeholders)."""
from typing import TypedDict


class ReviewerState(TypedDict, total=False):
    brd_markdown: str
    quality_findings: list[dict]
    pii_findings: list[dict]
    policy_findings: list[dict]
    final_report: str


def analyze_quality(state: ReviewerState) -> dict:
    """Phase 1 placeholder. Phase 2 calls Bedrock with quality rubric."""
    return {"quality_findings": []}


def scan_pii(state: ReviewerState) -> dict:
    """Phase 1 placeholder. Phase 2 runs regex + (optional) Comprehend PII detect."""
    return {"pii_findings": []}


def check_policy(state: ReviewerState) -> dict:
    """Phase 1 placeholder. Phase 2 evaluates against policy rules."""
    return {"policy_findings": []}


def generate_report(state: ReviewerState) -> dict:
    """Compile findings into a markdown report."""
    return {"final_report": "# Review Report\n\nPhase 1 placeholder.\n"}
```

### 5b: Create `agents/agent_3_reviewer/main.py`

```python
"""
Agent 3: BRD Reviewer (LangGraph 4-node) — S3 ZIP deployed to AgentCore.

Phase 1 scope: skeleton with all five required endpoints and a placeholder
8-block response. The LangGraph compiles successfully but every node is a
placeholder; real review logic arrives in Phase 2.
"""
from __future__ import annotations
import time
from bedrock_agentcore.runtime import BedrockAgentCoreApp
from langgraph.graph import StateGraph, END
from agents.common.payload_builder_stub import build_placeholder_payload
from agents.agent_3_reviewer.nodes import (
    ReviewerState, analyze_quality, scan_pii, check_policy, generate_report,
)

AGENT_ID = "agent_3_reviewer"
AGENT_VERSION = "1.0.0"

# Build the 4-node graph at import time. Nodes return partial state dicts
# (idiomatic LangGraph) so the reducer merges them correctly.
_builder = StateGraph(ReviewerState)
_builder.add_node("analyze_quality", analyze_quality)
_builder.add_node("scan_pii", scan_pii)
_builder.add_node("check_policy", check_policy)
_builder.add_node("generate_report", generate_report)
_builder.set_entry_point("analyze_quality")
_builder.add_edge("analyze_quality", "scan_pii")
_builder.add_edge("scan_pii", "check_policy")
_builder.add_edge("check_policy", "generate_report")
_builder.add_edge("generate_report", END)
graph = _builder.compile()

app = BedrockAgentCoreApp()


@app.entrypoint
def invoke(request: dict, context: dict | None = None) -> dict:
    started_at_ms = int(time.time() * 1000)
    # Phase 1: do not actually run the graph against arbitrary input.
    # The graph is wired so Phase 2 can swap nodes for real implementations.
    return build_placeholder_payload(
        request=request,
        agent_id=AGENT_ID,
        agent_version=AGENT_VERSION,
        started_at_ms=started_at_ms,
    )


@app.fastapi.get("/health")
def health():
    return {"status": "healthy", "agent_id": AGENT_ID, "version": AGENT_VERSION, "graph_nodes": 4}


@app.fastapi.get("/metadata")
def metadata():
    return {
        "agent_id": AGENT_ID,
        "agent_version": AGENT_VERSION,
        "framework": "langgraph",
        "tier": "first_party",
        "owner": "agentcore-demo-test1-team",
        "supported_task_types": ["brd_review"],
    }


@app.fastapi.get("/capabilities")
def capabilities():
    return {
        "inputs": {"artifact_refs": ["s3://*/brd/*.md"]},
        "outputs": {"artifact_type": "quality_document", "artifact_subtype": "review_report", "format": "md"},
        "tools": ["aws-bedrock"],
        "execution_modes": ["invoke"],
        "graph_nodes": ["analyze_quality", "scan_pii", "check_policy", "generate_report"],
    }


@app.fastapi.get("/metrics")
def metrics():
    return {"requests_total": 0, "errors_total": 0, "nodes_executed_total": 0}
```

### 5c: Create `agents/agent_3_reviewer/requirements.txt`

```text
langgraph
langchain-aws
bedrock-agentcore
```

### 5d: Create `agents/agent_3_reviewer/pyproject.toml`

```toml
[project]
name = "agent_3_reviewer"
version = "1.0.0"
requires-python = ">=3.11"
dependencies = ["langgraph", "langchain-aws", "bedrock-agentcore"]
```

**GATE**: All 4 files exist (main.py, nodes.py, requirements.txt, pyproject.toml). Endpoint count = 5. Graph compiles at import time (4 nodes wired linearly).

---

## STEP 5: Commit Agent Code and Sync Requirements

### 5.1: Update Root `requirements.txt` (Phase 1 minimal)

```bash
cd /Users/ugurgocen/projects/agentcore-demo-test1

# Phase 1 root-level deps only. Per PCD §4 rule 10, agents NEVER import temporalio.
# OpenTelemetry is added in Phase 2 (Prompt_02). Backend deps land in Phase 3.
cat > requirements.txt <<'REQEOF'
# Agent dependencies (union of all 3 agent requirements.txt files)
strands-agents
langgraph
langchain-aws
bedrock-agentcore

# Phase 1 dev tools
boto3
pydantic
pytest
REQEOF
```

### 5.2: Install in .venv

```bash
source .venv/bin/activate
uv pip install -r requirements.txt
```

### 5.3: Commit and Push

```bash
git add -A
git commit -m "feat(agents): add 3 minimal agents (2 Strands + 1 LangGraph) for S3 ZIP deploy

- Agent 1: Transcriber (Strands)
- Agent 2: BRD Drafter (Strands)
- Agent 3: BRD Reviewer (LangGraph StateGraph, 4 nodes)
- All use BedrockAgentCoreApp with @app.entrypoint
- S3 ZIP deployment (no ECR, no Docker)
- Filesystem config for S3 Files mount"
git push origin main
```

---

## STEP 6: Deploy Agent 1

Make sure `.vpc_env` is sourced so `$S3_BUCKET_ARTIFACTS` and `$S3_BUCKET_CLAIMCHECK` are set.

### 6.1: Configure (non-interactive)

```bash
cd /Users/ugurgocen/projects/agentcore-demo-test1/agentcore-demo-test1-backend/agents/agent_1_transcriber
source /Users/ugurgocen/projects/agentcore-demo-test1/agentcore-demo-test1-infra/scripts/.vpc_env

agentcore configure \
    --entrypoint main.py \
    --name agent_1_transcriber \
    --protocol HTTP \
    --deployment-type direct_code_deploy \
    --region us-east-1 \
    --non-interactive
```

Flag notes:
- `--non-interactive` / `-ni` skips all toolkit prompts; required for scripted/CI use.
- `--deployment-type direct_code_deploy` / `-dt direct_code_deploy` is the default; spelled out for explicitness. Use `container` only if you need a custom Dockerfile.
- `--protocol HTTP` is the canonical protocol per PCD §15.1.
- The command writes `.bedrock_agentcore.yaml` in the current directory — commit this file.

What `configure` does:
1. Validates `main.py` defines `BedrockAgentCoreApp` + `@app.entrypoint`.
2. Writes `.bedrock_agentcore.yaml` with the deployment config.
3. Does NOT yet upload code — `deploy` does that.

### 6.2: Deploy

```bash
agentcore deploy \
    --env "S3_BUCKET_ARTIFACTS=${S3_BUCKET_ARTIFACTS}" \
    --env "S3_BUCKET_CLAIMCHECK=${S3_BUCKET_CLAIMCHECK}" \
    --env "TRANSCRIBE_DATA_ACCESS_ROLE_ARN=${TRANSCRIBE_DATA_ACCESS_ROLE_ARN}" \
    --auto-update-on-conflict
```

Flag notes:
- `--env KEY=VAL` can be repeated; each becomes an environment variable inside the MicroVM at invocation time.
- `--auto-update-on-conflict` / `-auc` allows re-deploying over an existing runtime (without it, second deploy fails if the runtime already exists).
- `--region` / `-r` defaults to whatever `configure` set; override if needed.

What `deploy` does on first run:
1. Auto-provisions the S3 source bucket `bedrock-agentcore-codebuild-sources-${ACCOUNT_ID}-us-east-1`.
2. Auto-provisions a CodeBuild project, IAM execution role.
3. Zips the agent directory, uploads to S3.
4. CodeBuild builds an ARM64 container image.
5. Registers the AgentCore Runtime resource and binds the env vars.
Subsequent runs (`--auto-update-on-conflict`) reuse all of the above.

### 6.3: Verify

```bash
# Status of THIS agent
agentcore status

# Or list all agents in the region
aws bedrock-agentcore-control list-agent-runtimes --region us-east-1 \
    --query 'agentRuntimeSummaries[?starts_with(agentRuntimeName, `agent-`)].[agentRuntimeName,status]' --output table

# Pull the runtime ARN to .vpc_env for downstream phases
ARN=$(aws bedrock-agentcore-control list-agent-runtimes --region us-east-1 \
    --query "agentRuntimeSummaries[?agentRuntimeName=='agent_1_transcriber'].agentRuntimeArn | [0]" --output text)
echo "AGENTCORE_ARN_TRANSCRIBER=$ARN" >> /Users/ugurgocen/projects/agentcore-demo-test1/agentcore-demo-test1-infra/scripts/.vpc_env
```

**CHECKPOINT 6.1:** `agentcore status` reports state `READY` (or `ACTIVE` on older toolkit versions) and the runtime ARN is appended to `.vpc_env`.
If FAIL → `agentcore status --verbose` for the build log path; check the CodeBuild project logs in CloudWatch.

---

## STEP 7: Deploy Agent 2

```bash
cd /Users/ugurgocen/projects/agentcore-demo-test1/agentcore-demo-test1-backend/agents/agent_2_drafter
source /Users/ugurgocen/projects/agentcore-demo-test1/agentcore-demo-test1-infra/scripts/.vpc_env

agentcore configure \
    --entrypoint main.py --name agent_2_drafter \
    --protocol HTTP --deployment-type direct_code_deploy \
    --region us-east-1 --non-interactive

agentcore deploy \
    --env "S3_BUCKET_ARTIFACTS=${S3_BUCKET_ARTIFACTS}" \
    --env "S3_BUCKET_CLAIMCHECK=${S3_BUCKET_CLAIMCHECK}" \
    --env "BEDROCK_MODEL_ID=anthropic.claude-sonnet-4-6" \
    --env "SELF_CORRECTION_CAP=3" \
    --auto-update-on-conflict

ARN=$(aws bedrock-agentcore-control list-agent-runtimes --region us-east-1 \
    --query "agentRuntimeSummaries[?agentRuntimeName=='agent_2_drafter'].agentRuntimeArn | [0]" --output text)
echo "AGENTCORE_ARN_DRAFTER=$ARN" >> /Users/ugurgocen/projects/agentcore-demo-test1/agentcore-demo-test1-infra/scripts/.vpc_env
```

**CHECKPOINT 7.1:** `agentcore status` shows `agent_2_drafter` READY; ARN exported.

---

## STEP 8: Deploy Agent 3

```bash
cd /Users/ugurgocen/projects/agentcore-demo-test1/agentcore-demo-test1-backend/agents/agent_3_reviewer
source /Users/ugurgocen/projects/agentcore-demo-test1/agentcore-demo-test1-infra/scripts/.vpc_env

agentcore configure \
    --entrypoint main.py --name agent_3_reviewer \
    --protocol HTTP --deployment-type direct_code_deploy \
    --region us-east-1 --non-interactive

agentcore deploy \
    --env "S3_BUCKET_ARTIFACTS=${S3_BUCKET_ARTIFACTS}" \
    --env "BEDROCK_MODEL_ID=anthropic.claude-sonnet-4-6" \
    --auto-update-on-conflict

ARN=$(aws bedrock-agentcore-control list-agent-runtimes --region us-east-1 \
    --query "agentRuntimeSummaries[?agentRuntimeName=='agent_3_reviewer'].agentRuntimeArn | [0]" --output text)
echo "AGENTCORE_ARN_REVIEWER=$ARN" >> /Users/ugurgocen/projects/agentcore-demo-test1/agentcore-demo-test1-infra/scripts/.vpc_env
```

**CHECKPOINT 8.1:** All three agents READY; three ARNs in `.vpc_env`.

---

## STEP 8.5: Set retention on vended log groups (one-time, post-first-deploy)

AWS auto-created the AgentCore Runtime vended log groups during deploy but with **"Never expire" retention** (default). Set 7-day retention to match the rest of our log infrastructure (PCD §10):

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
for AGENT_NAME in $(aws bedrock-agentcore-control list-agent-runtimes --region us-east-1 \
        --query 'agentRuntimes[*].agentRuntimeName' --output text); do
    # Real pattern observed in deployment: /aws/bedrock-agentcore/runtimes/<agent_id>-<random>-<qualifier>
    aws logs describe-log-groups \
        --log-group-name-prefix "/aws/bedrock-agentcore/runtimes/${AGENT_NAME}-" \
        --query 'logGroups[*].logGroupName' --output text | tr '\t' '\n' | while read LG; do
        aws logs put-retention-policy --log-group-name "$LG" --retention-in-days 7
        echo "Set retention=7 on $LG"
    done
done
```

**CHECKPOINT 8.5:** All AgentCore vended log groups have `retentionInDays=7`. Verify:
```bash
aws logs describe-log-groups --log-group-name-prefix "/aws/bedrock-agentcore/runtimes/" \
    --query 'logGroups[?retentionInDays != `7`].[logGroupName,retentionInDays]' --output table
# Expected: empty result (no groups with retention != 7).
```

---

## STEP 9: Verify All 3 Agents Respond

### 9.1: Test `/ping` for All Agents

```python
"""Phase 1 smoke test — invokes each agent and validates the 8-block shape.
Run from EC2 (or your Mac with AWS creds): python scripts/smoke_invoke.py
"""
import os, json, time, boto3

REGION = os.environ.get("AWS_REGION", "us-east-1")
ACCOUNT_ID = boto3.client("sts").get_caller_identity()["Account"]
client = boto3.client("bedrock-agentcore-runtime", region_name=REGION)

AGENT_ARNS = {
    "agent_1_transcriber": f"arn:aws:bedrock-agentcore:{REGION}:{ACCOUNT_ID}:runtime/agent_1_transcriber",
    "agent_2_drafter":     f"arn:aws:bedrock-agentcore:{REGION}:{ACCOUNT_ID}:runtime/agent_2_drafter",
    "agent_3_reviewer":    f"arn:aws:bedrock-agentcore:{REGION}:{ACCOUNT_ID}:runtime/agent_3_reviewer",
}

REQUIRED_BLOCKS = {"status", "resources", "timing", "financial", "artifacts", "quality", "tool_calls", "risk"}

ts = int(time.time())
test_request = {
    "trace_id": f"TRC-smoke-{ts}",
    "workflow_template_id": "brd-from-audio-v1",
    "workflow_id": f"wf-smoke-{ts}",
    "agent_run_id": f"run-smoke-{ts}",
    "step_id": "step-001",
    "step_sequence": 1,
    "parent_run_id": f"wf-smoke-{ts}",
    "task_id": "TASK-SMOKE",
    "agent_id": "smoke-test",
    "agent_version": "0.0.0",
    "requested_by": {"user_id": "demo-user-001", "email": "demo@local", "roles": ["business_analyst"],
                     "group": {"group_id": "sap-mm"}, "project": {"project_id": "proj-demo"}},
    "execution_context": {"environment": "dev", "data_classification": "INTERNAL"},
    "task": {"task_type": "smoke", "complexity_tier": "simple"},
    "inputs": {"artifact_refs": []},
    "execution_options": {"dry_run": True, "timeout_seconds": 60},
    "actor_id": "demo-user-001", "session_id": f"sess-smoke-{ts}",
}

failures = 0
for name, arn in AGENT_ARNS.items():
    try:
        resp = client.invoke_agent_runtime(
            agentRuntimeArn=arn,
            runtimeSessionId=test_request["session_id"],
            qualifier="DEFAULT",
            payload=json.dumps(test_request).encode(),
        )
        body = json.loads(b"".join(c for c in resp.get("response", [])).decode())
        missing = REQUIRED_BLOCKS - set(body.keys())
        if missing:
            print(f"✗ {name}: missing blocks {missing}"); failures += 1
        elif body["status"]["workflow_id"] != test_request["workflow_id"]:
            print(f"✗ {name}: workflow_id not echoed"); failures += 1
        else:
            print(f"✓ {name}: 8-block valid, identifiers echoed")
    except Exception as e:
        print(f"✗ {name}: FAILED — {e}"); failures += 1

raise SystemExit(failures)
```

### 9.2: Quick CLI sanity check

```bash
# Replace <ACCOUNT_ID> via the .vpc_env (ACCOUNT_ID is exported there) or:
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
REGION="${AWS_DEFAULT_REGION:-us-east-1}"

for AGENT in agent_1_transcriber agent_2_drafter agent_3_reviewer; do
    echo "=== ${AGENT} ==="
    aws bedrock-agentcore-runtime invoke-agent-runtime \
      --agent-runtime-arn "arn:aws:bedrock-agentcore:${REGION}:${ACCOUNT_ID}:runtime/${AGENT}" \
      --runtime-session-id "smoke-$(date +%s)" \
      --qualifier DEFAULT \
      --payload "$(printf '%s' '{"trace_id":"TRC-cli","workflow_template_id":"brd-from-audio-v1","workflow_id":"wf-cli","agent_run_id":"run-cli","step_id":"step-001"}')" \
      --region $REGION
done
```

**GATE 1**: All 3 agents return a response (not an error). If PASS → commit and proceed to Phase 2.

### 9.3: Commit and Push (Gate 1 Pass)

```bash
cd /Users/ugurgocen/projects/agentcore-demo-test1
git add -A
git commit -m "feat(deploy): deploy 3 agents via S3 ZIP — all /ping and /invoke verified

- Agent 1 (transcriber): S3 ZIP deploy, ACTIVE
- Agent 2 (drafter): S3 ZIP deploy, ACTIVE
- Agent 3 (reviewer): S3 ZIP deploy, ACTIVE
- AgentCore bootstrap: completed
- S3 Files mount: /mnt/agentcore/claim-checks
- Deployment method: S3 ZIP (no ECR, no Docker)"
git push origin main
```

---

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| `agentcore: command not found` | .venv not activated | `source .venv/bin/activate` |
| `agentcore configure` fails | Missing `main.py` | Check file exists in current directory |
| `agentcore deploy` fails | IAM permissions for first-run auto-provisioning | Caller IAM needs `iam:CreateRole`, `iam:PutRolePolicy`, `s3:CreateBucket`, `codebuild:CreateProject` (auto-create on first deploy) |
| Agent shows `INACTIVE` | Runtime crash | Check CloudWatch `/aws/bedrock-agentcore/runtimes/<agent>` logs |
| `invoke_agent_runtime` returns 404 | Wrong ARN | Source `.vpc_env`; verify `AGENTCORE_ARN_TRANSCRIBER` etc. set |
| Want filesystem mount later | Not in Phase 1 | See `PERSISTENT_FILESYSTEM_GUIDE.md` — use `aws s3files` + `aws bedrock-agentcore-control update-agent-runtime --filesystem-configurations` |

---

## File Inventory

| # | File | Purpose |
|---|------|---------|
| 1 | `agents/agent_1_transcriber/main.py` | Transcriber entrypoint |
| 2 | `agents/agent_1_transcriber/requirements.txt` | Transcriber deps |
| 3 | `agents/agent_1_transcriber/pyproject.toml` | Transcriber package |
| 4 | `agents/agent_2_drafter/main.py` | Drafter entrypoint |
| 5 | `agents/agent_2_drafter/requirements.txt` | Drafter deps |
| 6 | `agents/agent_2_drafter/pyproject.toml` | Drafter package |
| 7 | `agents/agent_3_reviewer/main.py` | Reviewer entrypoint (LangGraph) |
| 8 | `agents/agent_3_reviewer/requirements.txt` | Reviewer deps |
| 9 | `agents/agent_3_reviewer/pyproject.toml` | Reviewer package |
| 10 | `requirements.txt` | Root dependencies (merged) |
| 11 | `.gitignore` | Git ignore rules |
| — | (filesystem files removed — deferred to PERSISTENT_FILESYSTEM_GUIDE.md) | — |

---

## What Was NOT Done (Correct — Phase 1 Scope)

The following are INTENTIONALLY excluded from Phase 1. They will be added in later phases:

- No telemetry emission
- No 8-block V5 payload
- No tool implementations (only placeholders)
- No HITL logic
- No A2A protocol
- No frontend integration
- No Dockerfile (S3 ZIP replaces it)
- No ECR (S3 ZIP replaces it)

---

## TESTING REQUIREMENTS — pytest Agent Verification

Every gate checkpoint MUST have a corresponding pytest test. Create:

```
tests/
  agents/
    conftest.py              # bedrock-agentcore-runtime client, AGENT_ARN fixtures
    test_gate_1_ping.py      # test_agent_1_ping, test_agent_2_ping, test_agent_3_ping
    test_gate_1_invoke.py    # test_agent_1_invoke, test_agent_2_invoke, test_agent_3_invoke
    test_deploy_method.py    # test_no_dockerfile, test_no_ecr_repo, test_s3_zip_exists
```

Key test — verify S3 ZIP deployment (NOT ECR):
```python
def test_no_dockerfile_in_agent_dirs():
    """Agents use S3 ZIP — no Dockerfile should exist."""
    for agent in ["agent_1_transcriber", "agent_2_drafter", "agent_3_reviewer"]:
        dockerfile = Path(f"agents/{agent}/Dockerfile")
        assert not dockerfile.exists(), f"Dockerfile found in {agent} — use S3 ZIP"

def test_no_ecr_repo():
    """No ECR repository should exist for agents."""
    ecr = boto3.client("ecr", region_name="us-east-1")
    repos = ecr.describe_repositories().get("repositories", [])
    agent_repos = [r for r in repos if "agent-" in r["repositoryName"]]
    assert len(agent_repos) == 0, f"ECR repos found: {agent_repos}"

def test_codebuild_sources_bucket_exists_after_first_deploy():
    """After the first agentcore deploy, the auto-provisioned source bucket appears.

    Run this test AFTER Step 6 (first deploy), not before.
    """
    s3 = boto3.client("s3", region_name="us-east-1")
    buckets = [b["Name"] for b in s3.list_buckets()["Buckets"]]
    # Canonical name pattern auto-created by the starter toolkit
    matches = [b for b in buckets if b.startswith("bedrock-agentcore-codebuild-sources-")]
    assert len(matches) > 0, ("No bedrock-agentcore-codebuild-sources-* bucket found; "
                              "did `agentcore deploy` complete successfully?")
```

Run: `pytest tests/agents/ -v`

---

## Appendix: Programmatic deployment via boto3 (`bedrock-agentcore-control`)

The CLI flow above (`agentcore configure` + `agentcore deploy`) is the easiest path on a developer machine. For CI/CD pipelines, GitHub Actions, or anywhere you want a hands-off deployment, the same operations are available via boto3 against the **control-plane** client `bedrock-agentcore-control`. The starter toolkit just wraps these APIs.

### A.1 What the toolkit does on first run (auto-provisioning)

1. **S3 bucket** for ZIP staging: `bedrock-agentcore-codebuild-sources-<account>-<region>` (created with `s3:CreateBucket`).
2. **CodeBuild project** that builds the ARM64 image from the staged ZIP.
3. **IAM execution role** for the runtime (`AmazonBedrockAgentCoreRuntimeRole-<random>`), trust policy: `bedrock-agentcore.amazonaws.com`.
4. **Agent runtime** resource: `bedrock-agentcore-control:create-agent-runtime` with `containerConfiguration.codeBuildProjectName` pointing at #2.

If you need to skip the toolkit (e.g. corporate environment that already has the role + bucket), use the snippet below.

### A.2 Programmatic deploy (Python + boto3)

```python
"""Deploy an agent to AgentCore without the starter toolkit.

Prereqs:
  - .vpc_env exports ACCOUNT_ID, AWS_REGION, S3_BUCKET_CODE, EC2_ROLE_ARN
  - Agent source is already packaged as a ZIP at $ZIP_PATH (use `zip -r agent.zip .` from the agent dir)
  - An IAM role exists with permissions for bedrock-runtime + the agent's tools (re-use the EC2 role for the demo)
"""
from __future__ import annotations
import os, sys, time
import boto3

REGION = os.environ["AWS_REGION"]
ACCOUNT_ID = os.environ["ACCOUNT_ID"]
S3_BUCKET_CODE = os.environ["S3_BUCKET_CODE"]
EXECUTION_ROLE_ARN = os.environ["EC2_ROLE_ARN"]  # demo simplification: re-use EC2 role
ZIP_PATH = sys.argv[1]                            # e.g. "build/agent_1_transcriber.zip"
AGENT_NAME = sys.argv[2]                          # e.g. "agent_1_transcriber"

s3 = boto3.client("s3", region_name=REGION)
control = boto3.client("bedrock-agentcore-control", region_name=REGION)

# 1) Upload the ZIP
s3_key = f"agents/{AGENT_NAME}/v1.0.0/{AGENT_NAME}.zip"
s3.upload_file(ZIP_PATH, S3_BUCKET_CODE, s3_key)
zip_uri = f"s3://{S3_BUCKET_CODE}/{s3_key}"
print(f"[uploaded] {zip_uri}")

# 2) Create or update the agent runtime
try:
    resp = control.create_agent_runtime(
        agentRuntimeName=AGENT_NAME,
        roleArn=EXECUTION_ROLE_ARN,
        networkConfiguration={"networkMode": "PUBLIC"},
        protocolConfiguration={"serverProtocol": "HTTP"},
        artifactSource={"s3Bucket": S3_BUCKET_CODE, "s3Key": s3_key},
        # filesystemConfigurations=[...]  # OPTIONAL; see PERSISTENT_FILESYSTEM_GUIDE.md
    )
    runtime_arn = resp["agentRuntimeArn"]
    print(f"[created] {runtime_arn}")
except control.exceptions.ConflictException:
    runtime = control.get_agent_runtime(agentRuntimeName=AGENT_NAME)
    runtime_arn = runtime["agentRuntimeArn"]
    control.update_agent_runtime(
        agentRuntimeId=runtime["agentRuntimeId"],
        artifactSource={"s3Bucket": S3_BUCKET_CODE, "s3Key": s3_key},
    )
    print(f"[updated] {runtime_arn}")

# 3) Poll until ACTIVE
deadline = time.time() + 600
while time.time() < deadline:
    rt = control.get_agent_runtime(agentRuntimeName=AGENT_NAME)
    status = rt["status"]
    print(f"  status={status}")
    if status == "ACTIVE":
        print(f"[ready] ARN={runtime_arn}")
        break
    if status in {"FAILED", "DELETING"}:
        print(f"[failed] {rt.get('statusReason')}")
        sys.exit(1)
    time.sleep(10)
```

Use it from a shell script that loops over the three agents:

```bash
for AGENT in agent_1_transcriber agent_2_drafter agent_3_reviewer; do
    cd "$PROJECT_ROOT/agentcore-demo-test1-backend/agents/${AGENT}"
    rm -f build.zip && zip -r build.zip . -x '*.pyc' '__pycache__/*'
    python "$PROJECT_ROOT/scripts/deploy_agent.py" build.zip "${AGENT//_/-}"
done
```

This is the **programmatic equivalent** of `agentcore configure` + `agentcore deploy`. The toolkit is preferred for local dev; this script is the form CI/CD uses.

### A.3 Bootstrap concerns

There is **no** separate `agentcore bootstrap` command. Both the toolkit and the boto3 path auto-provision the S3 bucket and CodeBuild project on first deploy (the toolkit creates `bedrock-agentcore-codebuild-sources-...`; the boto3 path uses the `S3_BUCKET_CODE` you created in Phase 0). The execution role you pass via `roleArn` must already exist; for the demo we re-use the EC2 instance role (Phase 0 §6) for simplicity.

---

## Next Phase

When ALL 3 agents respond to `/ping` and `/invocations` successfully:
- Report Phase 1 as COMPLETE
- Proceed to **Phase 2: Agent Payloads & Telemetry** using `Prompt_02_Agent_Telemetry.md`
- Reference docs: `Deliverable_2_Reference_Strands_Agent_Code.md` (payload builder skeleton)
