# V5 Implementation Plan — AgentCore demo test 1

## Verified Research Findings

### Finding 1: AgentCore Endpoints (Q1 Resolution)

**AgentCore Runtime service contract** (external-facing):
- `POST /invocations` — REQUIRED, primary agent interaction endpoint
- `GET /ping` — REQUIRED, health check for async agents
- Port 8080, ARM64 (Graviton)

**V5 agent framework contract** (internal):
- `POST /invoke` — V5 primary endpoint
- `GET /metadata` — V5 required
- `GET /capabilities` — V5 required
- `GET /metrics` — V5 required
- `GET /health` — V5 required

**How they connect:** `BedrockAgentCoreApp` from `bedrock-agentcore` SDK:
- `@app.entrypoint` decorator = the `/invoke` handler per V5
- `BedrockAgentCoreApp` auto-creates HTTP server on port 8080
- Auto-implements `/invocations` endpoint (maps to `@app.entrypoint`)
- Auto-implements `/ping` endpoint (AgentCore health check)
- Developer adds `/metadata`, `/capabilities`, `/metrics`, `/health` manually

**Answer to Q1:** Use `BedrockAgentCoreApp` with `@app.entrypoint`. This gives you `/invocations` (AgentCore) + `/ping` (AgentCore) automatically. You then add `/metadata`, `/capabilities`, `/metrics`, `/health` for V5 compliance.

### Finding 2: Direct Code Deployment (S3 Zip) — CONFIRMED

**From AWS docs:** "Direct Code Deploy Deployment (RECOMMENDED) — Suitable for most use cases, no Docker required."

Flow:
```
1. agentcore configure -e main.py
   → Creates .bedrock_agentcore.yaml
   → Auto-creates S3 bucket for code storage
   → Auto-creates IAM execution role
2. agentcore deploy
   → Packages Python code as ZIP
   → Uploads ZIP to S3
   → AWS CodeBuild builds ARM64 container (cloud)
   → Deploys to AgentCore Runtime
   → Returns Runtime ARN
```

**No local Docker needed.** No manual ECR push. No Dockerfile.

### Finding 3: OpenTelemetry / ADOT — CONFIRMED

**From AWS docs and blogs:**
- `aws-opentelemetry-distro` — auto-instruments boto3, HTTP requests
- AgentCore built-in observability via `.bedrock_agentcore.yaml`
- CloudWatch GenAI Observability dashboard — traces, spans, metrics
- ADOT pre-configured in AgentCore Runtime (auto-detection)
- Strands SDK: `trace_attributes` in Agent constructor for custom attributes
- Session context: `session_id`, `request_id` from AgentCore Runtime context

**For our demo (EC2-hosted ADOT Collector + AgentCore Runtime MicroVM):**
- Install `aws-opentelemetry-distro` on EC2
- Configure ADOT Collector (config YAML)
- Agent code uses `opentelemetry-api/sdk` directly
- Collector sends to CloudWatch (traces, metrics, logs)

---

## Files to Delete (5 items)

| File/Folder | Reason |
|---|---|
| `telemetry-sdk/` (entire folder, 15+ files) | V5 Section 9.5: "standard opentelemetry-sdk" only |
| `SDK_INTEGRATION_GUIDE.md` | No SDK to integrate |
| `Prompt_02b_Telemetry_SDK.md` | No SDK phase |
| `MCP_SERVER_GUIDE.md` | MCP module was SDK-dependent |
| `v2_to_v3_changes.md` | Obsolete |

## Files to Rewrite (Major Changes)

| File | Changes |
|---|---|
| `Prompt_02_Agent_Telemetry.md` | V5 payload builder (raw dict), OTel spans, no SDK EMF, ADOT Collector |
| `Deliverable_2_Reference_Strands_Agent_Code.md` | V5 payload, LangGraph agent 3, BedrockAgentCoreApp, new endpoints |
| `Deliverable_3_Implementation_Prompts.md` | V5 prompts, S3 deploy, no SDK |

## Files to Update (Medium Changes)

| File | Changes |
|---|---|
| `PHASE_PLAN.md` | Remove SDK/ECR/Docker, add S3 zip, OTel, ADOT |
| `Prompt_01_AgentCore_Deployment.md` | S3 zip via `agentcore configure/deploy`, no Dockerfile |
| `Prompt_03_Backend_Foundation.md` | OTel middleware, V5 endpoints, `bedrock-agentcore` imports |
| `Prompt_04_Temporal_Workflow.md` | OTel spans, A2A via Temporal signals |
| `Prompt_05_Frontend.md` | AG-UI/A2A separation, no SDK |
| `Prompt_06_E2E_Integration.md` | OTel trace verification, S3 deploy verification |
| `ARCHITECTURE_DATAFLOW_GUIDE.md` | ADOT Collector, S3 zip, AG-UI/A2A separation |
| `DEMO_GELISTIRME_KILAVUZU.md` | Full V5 update, Turkish |
| `Deliverable_0_PROJECT_CONTEXT.md` | V5 decisions: no SDK, S3 zip, OTel, mock auth |

## Files to Update (Minor Changes)

| File | Changes |
|---|---|
| `Deliverable_1_Infrastructure_Cost_Report.md` | Remove ECR cost, remove Docker |
| `Deliverable_4_Temporal_Operations_Guide.md` | A2A pattern via signals |
| `Deliverable_5_CopilotKit_AGUI_Guide.md` | AG-UI event types, A2A separation |
| `Deliverable_6_AWS_Operations_Guide.md` | S3 zip deploy procedure |
| `Deliverable_7_CloudWatch_Telemetry_Guide.md` | OTel → ADOT → CloudWatch |
| `Deliverable_8_Frontend_Architecture_Guide.md` | AG-UI/A2A separation |
| `Deliverable_9_Multi_Framework_AGUI_Guide.md` | Minor |
| `DELIVERABLE_MAPPING.md` | Remove SDK |

## Images to Update

| Image | Changes |
|---|---|
| `architecture_infographic.png` | Remove SDK, add ADOT Collector, S3 zip, AG-UI/A2A split |
| `dataflow_diagram.png` | OTel → ADOT → CloudWatch, V5 payload, AG-UI/A2A split |

---

## V5 Payload Schema (Section 7) — Exact Fields

### Block 1: Status
- `code`: string ("success", "error", etc.)
- `http_status`: integer
- `message`: string
- `trace_id`: string
- `workflow_template_id`: string
- `workflow_id`: string
- `agent_run_id`: string
- `step_id`: string
- `parent_run_id`: string ("Unavailable" if none)
- `tier`: "native" (all 3 agents — custom-built)
- `step_type`: enum — `agent_action`, `human_review`, `human_approval`, `human_question`, `tool_call`, `claim_check`
- `custom_metadata`: dict, 4KB limit
- `tenant_id`: string ("demo-tenant" for demo)

### Block 2: Resources
- `cost_reporting_model`: "token_based" | "vendor_cost" | "fixed_subscription" | "both"
- `token_based`: {model_used, tier, tokens{input, output, total_outer}, internal_llm_calls, total_tokens_all_calls}
- `vendor_cost`: {vendor_name, cost_per_request_usd, requests_made, total_cost_usd}
- `fixed_subscription`: {plan_name, monthly_cost_usd, period_start, period_end}

### Block 3: Timing
- `total_elapsed_ms`: integer
- `queue_duration_ms`: integer (0 if unknown)
- `agent_active_ms`: integer
- `waiting_for_tools_ms`: integer (0 if unknown)
- `waiting_for_human_ms`: integer (0 for non-HITL)
- `timing_breakdown`: {reasoning_ms, tool_calls_ms, formatting_ms}
- `critical_path`: string

### Block 4: Financial
- `task_type`: string
- `complexity_tier`: string
- `manual_baseline_hours`: float
- `agent_active_hours`: float
- `human_review_hours_estimated`: float or null
- `agent_cost_usd`: float
- `retry_cost_usd`: float (0.0 if no retry)
- `currency`: "USD" (uppercase per V5)
- `retention_policy_ref`: string

### Block 5: Artifacts
- List of: {artifact_id, artifact_type, artifact_subtype, artifact_format, artifact_size_bytes, artifact_title, content_ref, content_hash, versioning{version, version_sequence, is_new_artifact, previous_version, previous_version_ref, change_summary}, timestamps{created_at, generation_started_at, generation_duration_ms}, template_used, template_adherence_pct, sections_completed, sections_total, information_classification, retention_policy_ref}

### Block 6: Quality
- `confidence`: float (0-1)
- `initial_attempt_success`: boolean
- `total_attempts`: integer
- `self_corrections`: integer
- `quality_concerns`: list of strings
- `concerns`: list of strings
- `unsupported_claims`: list of {claim_text, reason, location, severity}
- `defects_found_later`: integer
- `template_adherence_pct`: float (0-100)
- `completeness_assessment`: string

### Block 7: Tool Calls
- List of: {sequence, server, operation, duration_ms, status, result_summary, downstream_system_time_ms, mcp_overhead_ms, attempt}

### Block 8: Risk
- `pii_detected`: boolean
- `pii_filtered_count`: integer
- `secrets_detected`: boolean
- `policy_violations`: list of {policy_id, severity, description, rule_id, category}
- `sensitivity_compliant`: boolean
- `compliance_checks`: list of {framework, check, passed}
- `compliance_classification`: {pass, fail, not_applicable}

---

## Telemetry Architecture (V5 Section 9)

```
Agent Code (opentelemetry-api/sdk)
  → aws-opentelemetry-distro (auto-instrumentation)
  → stdout / file / UDP
  → ADOT Collector (EC2 daemon/sidecar)
    → CloudWatch Metrics (via EMF exporter)
    → CloudWatch Traces (via X-Ray exporter)
    → CloudWatch Logs (via logs exporter)
    → CloudWatch GenAI Observability Dashboard
```

**Required packages:**
```
opentelemetry-api
opentelemetry-sdk
opentelemetry-distro[aws]
opentelemetry-exporter-otlp
```

**ADOT Collector config (EC2):**
```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318
processors:
  batch:
exporters:
  awsxray:
  awsemf:
    namespace: "AISDLC/Agent"
  awscloudwatchlogs:
    log_group_name: "/aws/agentcore/agentcore-demo-test1"
service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [awsxray]
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [awsemf]
    logs:
      receivers: [otlp]
      processors: [batch]
      exporters: [awscloudwatchlogs]
```

---

## Deployment: S3 Zip (No Container)

### For Each Agent:
```bash
cd agents/agent_1_transcriber

# 1. Configure (creates .bedrock_agentcore.yaml + S3 bucket + IAM role)
agentcore configure -e main.py --protocol AGUI -r us-east-1

# 2. Deploy (packages ZIP → S3 → CodeBuild ARM64 → Runtime)
agentcore deploy

# 3. Note the Runtime ARN from output
```

### Manual ZIP (Without Starter Toolkit, for understanding):
```bash
cd agent-directory
uv venv .venv
source .venv/bin/activate
uv pip install -r requirements.txt --target=./package
cp main.py ./package/
cd package && zip -r ../agent.zip . && cd ..
aws s3 cp agent.zip s3://agentcore-demo-test1-agents/agent-1-transcriber.zip
# Then create Runtime via AWS Console or CLI
```

### Agent `main.py` Pattern:
```python
from bedrock_agentcore.runtime import BedrockAgentCoreApp

app = BedrockAgentCoreApp()

@app.entrypoint  # This = /invoke (V5) → auto-mapped to /invocations (AgentCore)
def invoke(payload, context):
    """Primary agent execution endpoint."""
    session_id = context.get("session_id", "unknown")
    # ... agent logic ...
    return {"response": result, "payload": eight_block_payload}

# V5 additional endpoints
@app.get("/metadata")
def metadata():
    return {"agent_name": "transcriber", "version": "1.0.0",
            "model": "anthropic.claude-3-5-sonnet-20241022-v2:0",
            "endpoints": ["/invoke", "/metadata", "/capabilities", "/metrics", "/health"]}

@app.get("/capabilities")
def capabilities():
    return {"tools": [{"name": "transcribe_audio", "description": "..."}]}

@app.get("/metrics")
def metrics():
    return generate_latest(prometheus_registry)  # OTel Prometheus format

@app.get("/health")
def health():
    return {"status": "ok"}
# /invocations and /ping are auto-created by BedrockAgentCoreApp
```

---

## AG-UI vs A2A Separation

### AG-UI (Agent-User Interface Protocol)
- **Scope:** Frontend ↔ Agent (real-time user interaction)
- **Transport:** SSE (text/event-stream) or WebSocket
- **Library:** CopilotKit React SDK
- **Use in demo:** Chat streaming, tool call rendering, HITL interrupts, canvas state
- **Events:** RUN_STARTED/FINISHED, TEXT_MESSAGE_*, TOOL_CALL_*, STATE_*, INTERRUPT

### A2A (Agent-to-Agent Protocol)
- **Scope:** Agent ↔ Agent (inter-agent communication)
- **Transport:** HTTP/REST + JSON
- **Use in demo:** Temporal workflow signals between agents
- **Pattern:** Agent 1 completes → Temporal signal → Agent 2 starts
- **V5 Section 6.6:** A2A is the preferred protocol for agent↔agent

### Separation in Architecture:
```
Frontend (CopilotKit)
  ├── AG-UI SSE ←→ AgentCore Runtime ←→ Agent MicroVM
  │     (chat, streaming, HITL, state sync)
  │
  └── REST ←→ FastAPI ←→ Temporal Server
        (workflow lifecycle, signals)
        Temporal Signals = A2A equivalent in our architecture
```

---

## Mock Auth (Demo Simplification)

**Production:** Azure Entra ID + PingID MFA (V5 Section 15.1.1)
**Demo:** Mock auth — `MOCK_USER` constant

```python
# backend/app/auth/mock_user.py
MOCK_USER = {
    "user_id": "demo-user-001",
    "project_id": "PROJ-DEFAULT-123",
    "roles": ["functional_engineer"],
}
# Production: Replace with Entra ID + PingID per V5 Section 15.1.1
```

---

## Execution Order

1. Delete 5 items (SDK folder + 4 files)
2. Rewrite 3 files (Prompt_02, Deliverable_2, Deliverable_3)
3. Update 9 files (PHASE_PLAN, Prompt_01, 03-06, ARCHITECTURE, DEMO_GUIDE, Deliverable_0)
4. Update 8 reference deliverables (minor)
5. Update 2 images
6. Final ZIP package
