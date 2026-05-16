# Deliverable 7: CloudWatch Telemetry Guide (V5 — OTel + ADOT)

**Date:** 2026-05-14
**Version:** 5.0 (V5-compliant)
**Scope:** OpenTelemetry instrumentation, ADOT Collector configuration, CloudWatch GenAI Dashboard, alarms, cost control

---

> **ARCHITECTURE NOTE:** This document describes the V5 telemetry stack: `opentelemetry-sdk` → OTLP gRPC → ADOT Collector → CloudWatch (Logs + Metrics + X-Ray). The agent developer NEVER touches EMF JSON or calls `PutMetricData`. All metrics are automatically extracted from OTel spans by the ADOT Collector. See `ARCHITECTURE_DATAFLOW_GUIDE.md` Section 4.3 for the Three-Tier Observability Model.

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Log Groups](#2-log-groups)
3. [ADOT Collector Configuration](#3-adot-collector-configuration)
4. [OTel Instrumentation in Agent Code](#4-otel-instrumentation-in-agent-code)
5. [CloudWatch Metrics (Auto-Extracted from Spans)](#5-cloudwatch-metrics-auto-extracted-from-spans)
6. [CloudWatch GenAI Dashboard](#6-cloudwatch-genai-dashboard)
7. [Logs Insights Queries](#7-logs-insights-queries)
8. [Alarms](#8-alarms)
9. [Trace Propagation Verification](#9-trace-propagation-verification)
10. [Cost Control](#10-cost-control)
11. [Tech Reference](#11-tech-reference)
12. [Quick Start Checklist](#12-quick-start-checklist)

---

## 1. Architecture Overview

### 1.1 The Three-Tier Observability Model

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    THREE-TIER OBSERVABILITY MODEL                           │
├──────────────────┬──────────────────────────────────┬─────────────────────┤
│   TIER 1         │         TIER 2                   │    TIER 3           │
│ Infrastructure   │     Business Metadata            │  Enforcement &      │
│   Telemetry      │                                  │   Formatting        │
├──────────────────┼──────────────────────────────────┼─────────────────────┤
│                  │                                  │                     │
│ Bedrock AgentCore│ Developer attaches jnj.* attrs   │ ADOT Collector on   │
│ auto-emits       │ to OTel spans using standard     │ EC2 transforms:     │
│ vended logs +    │ opentelemetry-sdk (NO custom     │                     │
│ X-Ray traces     │ SDK, NO hand-crafted EMF)        │ • Span → EMF      │
│                  │                                  │ • Dimension assembly│
│ InvokeAgentRuntime│ span.set_attribute(             │ • Cardinality       │
│ ExecuteTool      │   "jnj.workflow_id", "wf-001")   │   enforcement       │
│ CreateEvent      │                                  │ • Governance gate   │
│                  │                                  │   (reject missing   │
│ NO agent code    │ NO EMF formatting in agent code  │   mandatory attrs)  │
│ required         │                                  │                     │
│                  │                                  │ Writes to:          │
│ → CloudWatch     │ → OTLP gRPC → ADOT Collector    │  • CloudWatch Logs  │
│    vended logs   │                                  │  • GenAI Dashboard  │
│                  │                                  │  • X-Ray traces     │
└──────────────────┴──────────────────────────────────┴─────────────────────┘
```

| Tier | What It Carries | Who Owns It | Code Required? |
|------|----------------|-------------|----------------|
| **T1: Infrastructure** | `InvokeAgentRuntime`, `ExecuteTool`, `CreateEvent` in Memory | Bedrock AgentCore | **NONE** — auto-emits vended logs to `/aws/bedrock-agentcore/runtimes/<agent_id>-<random>-<qualifier>` |
| **T2: Business Metadata** | `jnj.*` attributes: `jnj.workflow_id`, `jnj.error_category`, `jnj.step_id` | Agent developer | **STANDARD OTel ONLY** — `span.set_attribute()` |
| **T3: Enforcement** | Span→EMF transform, `_aws` block assembly, cardinality enforcement | ADOT Collector (platform) | **NONE** — configured centrally |

### 1.2 Data Flow

```
Python Agent Code (opentelemetry-sdk)
  └── TracerProvider → BatchSpanProcessor
      └── OTLPSpanExporter (gRPC, port 4317)
          └── ADOT Collector (EC2, docker-compose)
              ├── Span-to-EMF transformation (TIER 3 — ADOT)
              │   ├── Extract jnj.* attributes from each span
              │   ├── Assemble EMF _aws block (auto)
              │   ├── Enforce cardinality discipline (auto)
              │   └── Governance gate: reject spans missing jnj.workflow_id (auto)
              ├── CloudWatch Logs (structured OTel + EMF)
              ├── CloudWatch GenAI Dashboard (pre-built)
              └── X-Ray (distributed traces)
```

### 1.3 Agent Developer Surface Area

The agent developer's ONLY responsibility is TIER 2: attach business attributes to OTel spans. Everything else is platform-managed.

```python
from opentelemetry import trace

tracer = trace.get_tracer(__name__)

def execute_agent_turn(agent_logic_func, workflow_id: str, step_id: str):
    """Canonical Day-1 pattern — attach business attributes, NO EMF formatting."""
    with tracer.start_as_current_span("AgentExecution") as span:
        # Required enterprise attributes (TIER 2 — developer responsibility)
        span.set_attribute("jnj.workflow_id", workflow_id)
        span.set_attribute("jnj.agent_run_id", f"agent-1-run-{seq}")
        span.set_attribute("jnj.step_id", step_id)
        span.set_attribute("jnj.error_category", "none")
        try:
            return agent_logic_func()
        except Exception as e:
            span.set_attribute("jnj.error_category", "system_failure")
            span.record_exception(e)
            raise
```

---

## 2. Log Groups

### 2.1 Log Group Architecture (canonical per PCD §10)

Two name spaces are in play:

**Pre-created in Phase 0 (Prompt_00 Step 6) — 4 groups, all with 7-day retention:**

| Log Group | Source | Contents |
|---|---|---|
| `/agentcore-demo-test1/agent-logs` | ADOT → all 3 agents' spans/logs | OTel span events + business logs. Per-agent filtering done in queries via `filter demo.agent_id = "agent_X_..."` (every record carries that attribute) |
| `/agentcore-demo-test1/adot-collector` | ADOT self-diagnostics | Pipeline stats, dropped spans, governance-gate hits |
| `/agentcore-demo-test1/fastapi` | FastAPI container + EMF metric records | App logs + EMF JSON for metric extraction |
| `/agentcore-demo-test1/temporal-worker` | Temporal Worker container | Worker activity logs + workflow spans |

> **Why not 3 per-agent log groups?** Each log record already carries `demo.agent_id`. AWS additionally auto-creates per-agent **vended** log groups (next table) for container stdout. A third layer of per-agent ADOT-routed groups would mean 3 extra retentions / IAM ARNs / exporters with zero operational benefit — cross-agent correlation is trivial when everything is in one group.

**Auto-created on first agent invocation (do NOT pre-create):**

| Log Group | Source | Contents |
|---|---|---|
| `/aws/bedrock-agentcore/runtimes/<agent_id>-<random>-<qualifier>` | Bedrock AgentCore (TIER 1, vended) | Native `InvokeAgentRuntime`, `ExecuteTool`, `CreateEvent` events + container runtime stdout/stderr + OTel exporter logs. AWS provisions these — agents do not write to them. Example real log group: `/aws/bedrock-agentcore/runtimes/agent_1_transcriber-ZgDOxI4k9f-DEFAULT`. Log streams inside: `YYYY/MM/DD/[runtime-logs]` (container stdout, date-partitioned) and `otel-rt-logs` (OTel pipeline). **Retention defaults to "Never expire" — set retention manually with `aws logs put-retention-policy` after first deploy or costs accumulate.** |

AgentCore vended logs join with our `demo.*`-tagged ADOT logs on `trace_id` for end-to-end correlation.

### 2.2 Create Log Groups

These commands run in Phase 0 (Prompt_00 Step 6); they appear here for reference / re-running:

```bash
for GROUP in \
  /agentcore-demo-test1/agent-logs \
  /agentcore-demo-test1/adot-collector \
  /agentcore-demo-test1/fastapi \
  /agentcore-demo-test1/temporal-worker; do

    aws logs create-log-group --log-group-name "$GROUP" 2>/dev/null || echo "Log group exists: $GROUP"
    aws logs put-retention-policy --log-group-name "$GROUP" --retention-in-days 7
    echo "Created/verified: $GROUP (7-day retention)"
done
```

### 2.3 Verify

```bash
aws logs describe-log-groups \
  --query "logGroups[?starts_with(logGroupName, '/agentcore/') || starts_with(logGroupName, '/agentcore-demo-test1/')].{Name:logGroupName,Retention:retentionInDays}" \
  --output table

# EXPECTED: 6 log groups, each with retentionInDays = 7
```

### 2.4 CloudWatch Logs Insights: Cross-Log-Group Query

OTel traces span multiple log groups. Use **Cross-Log-Group Insights** to trace a single `workflow_id` across all sources.

```sql
-- Cross-log-group trace: find a single workflow across all sources
fields @timestamp, @logGroup, @message
| parse @message '"workflow_id": "*"' as workflow_id
| parse @message '"trace_id": "*"' as trace_id
| filter workflow_id = "wf-brd-20250115-001"
| stats count(*) as event_count by trace_id, @logGroup
| display trace_id, @logGroup, event_count
| sort trace_id asc
```

**Expected:** Each `trace_id` appears in MULTIPLE log groups (FastAPI, Temporal Worker, AgentCore vended logs).

---

## 3. ADOT Collector Configuration

### 3.1 Architecture

The ADOT Collector runs as a Docker container on the EC2 instance. It receives OTel spans via OTLP gRPC (port 4317), transforms them, and exports to CloudWatch.

```
Agent (Python opentelemetry-sdk)
    ↓ OTLP gRPC :4317
ADOT Collector (EC2, Docker)
    ↓ span-to-EMF transformation
    ↓ cardinality enforcement
    ↓ governance gating
CloudWatch (Logs + Metrics + X-Ray)
```

### 3.2 Collector Config (`adot-config.yaml`)

```yaml
# /Users/ugurgocen/projects/agentcore-demo-test1/agentcore-demo-test1-infra/adot-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 1s
    send_batch_size: 1024

  # Governance gate: reject spans missing mandatory jnj.* attributes
  filter/governance:
    spans:
      exclude:
        match_type: strict
        # Spans WITHOUT jnj.workflow_id are dropped (logged as warning)
        # This enforces the Day-1 rule: every span must have jnj.workflow_id

exporters:
  awscloudwatchlogs:
    log_group_name: /agentcore-demo-test1/fastapi
    log_stream_name: otel-spans
    region: us-east-1

  awsxray:
    region: us-east-1

  # CloudWatch EMF exporter — auto-extracts metrics from OTel data points.
  # Per PCD §10, the canonical namespace is DemoSDLC/Agent (was DemoSDLC/Agent
  # in earlier drafts; do not use the old name).
  #
  # IMPORTANT — what becomes a metric:
  #   metric_name_selectors operate on NUMERIC OTel data points, not on string
  #   attributes. So `demo.workflow_id` (a string) is NOT a metric; it lands at
  #   the EMF root as a searchable field. Numeric span events emitted as OTel
  #   metric instruments (gauges/counters/histograms) ARE metrics, with their
  #   instrument name (e.g. "AgentLatencyMs", "TokensInput") becoming the metric
  #   name. The dimensions list below is the LOW-cardinality fields the EMF
  #   exporter is allowed to materialize (see PCD §10 cardinality discipline).
  awsemf:
    log_group_name: /agentcore-demo-test1/fastapi
    log_stream_name: otel-metrics
    namespace: DemoSDLC/Agent
    region: us-east-1
    dimension_rollup_option: NoDimensionRollup
    metric_declarations:
      - dimensions: [["agent_id", "task_type", "tier", "environment", "workflow_template_id"]]
        metric_name_selectors:
          - "AgentLatencyMs"          # Histogram, emitted from agent_active_ms
          - "AgentCostUsd"             # Gauge, emitted from financial.agent_cost_usd
          - "TokensInput"              # Counter, gen_ai.usage.input_tokens
          - "TokensOutput"             # Counter, gen_ai.usage.output_tokens
          - "TotalAttempts"            # Gauge, quality.total_attempts (self-correction cap)
      - dimensions: [["workflow_template_id", "environment"]]
        metric_name_selectors:
          - "WorkflowOutcome"          # Counter, +1 per terminal outcome
          - "HitlRoundCount"           # Histogram, number of HITL rounds per workflow
          - "WorkflowDurationMs"        # Histogram, total_elapsed_ms aggregated to workflow
    # HIGH-cardinality fields (demo.trace_id, demo.workflow_id, demo.agent_run_id,
    # demo.step_id, demo.actor_id) appear at the EMF JSON root via the
    # attributes/demo processor (defined in Prompt_02) — never in Dimensions.

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

### 3.3 Docker Compose Entry

Add to `docker-compose.yml` on EC2:

```yaml
  adot-collector:
    image: public.ecr.aws/aws-observability/aws-otel-collector:v0.39.0
    command: ["--config=/etc/otel-agent-config.yaml"]
    volumes:
      - ./adot-config.yaml:/etc/otel-agent-config.yaml
    ports:
      - "4317:4317"   # OTLP gRPC
      - "4318:4318"   # OTLP HTTP
      - "55679:55679" # Health check
    environment:
      - AWS_REGION=us-east-1
    depends_on:
      - temporal-server
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:55679/health"]
      interval: 30s
      timeout: 5s
      retries: 3
```

### 3.4 Python Tracer Setup (`otel_setup.py`)

```python
# src/otel_setup.py — Initialize OTel tracer with OTLP export to ADOT

from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.resources import Resource
import os

OTEL_ENDPOINT = os.getenv("OTEL_EXPORTER_OTLP_ENDPOINT", "http://localhost:4317")

def init_tracer(service_name: str = "agentcore-demo-test1"):
    """Initialize the global TracerProvider with OTLP export to ADOT Collector."""
    resource = Resource.create({"service.name": service_name})
    provider = TracerProvider(resource=resource)
    trace.set_tracer_provider(provider)

    # OTLP gRPC exporter → ADOT Collector (port 4317)
    otlp_exporter = OTLPSpanExporter(
        endpoint=OTEL_ENDPOINT,
        insecure=True,  # Local network (EC2 loopback), TLS not needed
    )
    processor = BatchSpanProcessor(otlp_exporter)
    provider.add_span_processor(processor)

    return trace.get_tracer(__name__)
```

### 3.5 Environment Variables (`.env` or docker-compose)

```bash
# OTel configuration — set these in docker-compose or .env
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
OTEL_SERVICE_NAME=agentcore-demo-test1
OTEL_TRACES_EXPORTER=otlp
OTEL_METRICS_EXPORTER=otlp
OTEL_LOGS_EXPORTER=otlp

# AWS credentials via IMDS (EC2 instance role — NO access keys)
AWS_REGION=us-east-1
```

---

## 4. OTel Instrumentation in Agent Code

### 4.1 Required `jnj.*` Attributes (Mandatory)

Every span emitted by agent code MUST contain these attributes. The ADOT Collector governance gate drops spans missing `jnj.workflow_id`.

| Attribute | Type | Source | Example |
|---|---|---|---|
| `jnj.workflow_id` | string | Temporal workflow execution | `"wf-brd-20250115-001"` |
| `jnj.agent_run_id` | string | Agent execution sequence | `"agent-1-run-003"` |
| `jnj.step_id` | string | Workflow phase | `"phase_2_draft"` |
| `jnj.agent_name` | string | Agent identifier | `"Agent1_Transcriber"` |
| `jnj.error_category` | string | Error classification | `"none"`, `"validation"`, `"system_failure"` |

### 4.2 Instrumentation Pattern (Every Agent)

```python
from opentelemetry import trace

tracer = trace.get_tracer(__name__)

def run_agent_with_telemetry(workflow_id: str, input_data: dict) -> dict:
    """Execute agent logic with full V5 telemetry.

    This is the ONLY instrumentation code the agent developer writes.
    All EMF formatting, metric extraction, and CloudWatch writing is
    handled by the ADOT Collector (TIER 3).
    """
    with tracer.start_as_current_span("AgentExecution") as span:
        # --- REQUIRED: jnj.* attributes (TIER 2) ---
        span.set_attribute("jnj.workflow_id", workflow_id)
        span.set_attribute("jnj.agent_run_id", generate_run_id())
        span.set_attribute("jnj.step_id", get_current_phase())
        span.set_attribute("agent_id", "agent_1_transcriber")
        span.set_attribute("jnj.error_category", "none")

        # --- OPTIONAL: Business metrics as span attributes ---
        # The ADOT Collector auto-extracts these as CloudWatch metrics
        span.set_attribute("AgentLatencyMs", 0)  # Will be updated on completion
        span.set_attribute("TokensInput", 0)   # Will be updated on completion
        span.set_attribute("AgentCostUsd", 0.0)    # Will be updated on completion

        # --- Execute agent logic ---
        try:
            start_time = time.time()
            result = execute_agent_logic(input_data)
            elapsed_ms = int((time.time() - start_time) * 1000)

            # Update metrics post-execution
            span.set_attribute("AgentLatencyMs", elapsed_ms)
            span.set_attribute("TokensInput", result.get("tokens_used", 0))
            span.set_attribute("AgentCostUsd", result.get("cost_usd", 0.0))

            # Emit V5 8-block payload as span event (see Deliverable 2)
            span.add_event("v5_payload", {"payload": build_v5_payload(result)})

            return result

        except Exception as e:
            span.set_attribute("jnj.error_category", classify_error(e))
            span.record_exception(e)
            raise
```

### 4.3 Manual Span Creation (Outside `@app.entrypoint`)

If you need to instrument a specific function outside the main agent handler:

```python
def transcribe_audio(audio_uri: str, workflow_id: str) -> str:
    """Instrumented Transcribe call with OTel span."""
    with tracer.start_as_current_span("TranscribeAudio") as span:
        span.set_attribute("jnj.workflow_id", workflow_id)
        span.set_attribute("agent_id", "agent_1_transcriber")
        span.set_attribute("jnj.error_category", "none")
        span.set_attribute("audio.uri", audio_uri)

        try:
            result = call_transcribe_api(audio_uri)
            span.set_attribute("transcription.job_id", result["job_id"])
            span.set_attribute("transcription.language", result["language"])
            return result["text"]
        except Exception as e:
            span.set_attribute("jnj.error_category", "aws_api_failure")
            span.record_exception(e)
            raise
```

---

## 5. CloudWatch Metrics — what is emitted, by whom, where it lands

### 5.1 Three sources of metric data

| Source | Emits | Lands at |
|---|---|---|
| **AgentCore Runtime (vended, automatic)** | `Invocations`, `SystemErrors`, `UserErrors`, `Latency`, `vCPUHours`, `GBHours` per agent ARN | CloudWatch namespace `AWS/Bedrock-AgentCore` — NOT under our control. AWS auto-emits these per invocation. |
| **Agent OTel instruments (developer code)** | Custom metric instruments via `meter.create_histogram("AgentLatencyMs")` etc. | OTLP → ADOT Collector → CloudWatch namespace `DemoSDLC/Agent` (via `awsemf` exporter). |
| **Span attributes that ALSO are numeric metric instruments** | `gen_ai.usage.input_tokens` / `output_tokens` (OpenTelemetry GenAI semantic convention) | Same path as above when the SDK exposes them as metric instruments rather than just span attributes. |

> **String attributes are NOT metrics.** `demo.workflow_id`, `demo.actor_id`, `demo.error_category` are searchable fields in the EMF JSON root — they appear as queryable columns in CloudWatch Logs Insights, but they do not create CloudWatch metric time series. Only numeric metric instruments do.

### 5.2 What the demo agents actually emit (Phase 2 instrumentation)

`agents/common/otel_setup.py` (Prompt_02) creates a `Meter` per agent and registers three histograms + one counter:

```python
from opentelemetry import metrics

meter = metrics.get_meter("agentcore-demo-test1-<agent>")

# Numeric instruments — these become CloudWatch metrics via ADOT awsemf
AGENT_LATENCY_MS  = meter.create_histogram("AgentLatencyMs",  unit="ms")
TOKENS_INPUT      = meter.create_counter ("TokensInput",      unit="1")
TOKENS_OUTPUT     = meter.create_counter ("TokensOutput",     unit="1")
AGENT_COST_USD    = meter.create_histogram("AgentCostUsd",    unit="USD")
WORKFLOW_OUTCOME  = meter.create_counter ("WorkflowOutcome",  unit="1")
```

These are recorded inside `agent_execution_span()` once the 8-block response is built:

```python
elapsed = ended_at_ms - started_at_ms
attrs = {"agent_id": AGENT_ID, "task_type": task_type,
         "tier": "first_party", "environment": "dev",
         "workflow_template_id": request["workflow_template_id"]}
AGENT_LATENCY_MS.record(elapsed, attributes=attrs)
TOKENS_INPUT.add(tokens_in, attributes=attrs)
TOKENS_OUTPUT.add(tokens_out, attributes=attrs)
AGENT_COST_USD.record(agent_cost_usd, attributes=attrs)
```

The `attributes` dict is the **LOW-cardinality** dimension set (PCD §10) — bounded values that are safe to materialize as CloudWatch metric dimensions.

### 5.3 Canonical Metric Definitions

| Metric Name | Unit | Type | Source | Dimensions | Used By |
|---|---|---|---|---|---|
| `AgentLatencyMs` | Milliseconds | Histogram | Computed from `agent_active_ms` in Block 3 | `agent_id`, `task_type`, `tier`, `environment`, `workflow_template_id` | Phase 7 dashboard p50/p95/p99 |
| `TokensInput` | Count | Counter | Block 2 `tokens.input` | Same | Phase 7 dashboard token consumption |
| `TokensOutput` | Count | Counter | Block 2 `tokens.output` | Same | Phase 7 dashboard token consumption |
| `AgentCostUsd` | USD | Histogram | Block 4 `agent_cost_usd` | Same | Phase 7 dashboard cost over time |
| `WorkflowOutcome` | Count | Counter (+1 per terminal) | BRDState transition in workflow | `workflow_template_id`, `environment`, `outcome={APPROVED,REJECTED,FAILED,MAX_ITERATIONS}` | Phase 7 success rate |
| `HitlRoundCount` | Count | Histogram | Temporal workflow on completion | `workflow_template_id`, `environment` | Phase 7 HITL frequency |

In addition, **AgentCore native vended metrics** (no code required):

| Metric Name | Source | Notes |
|---|---|---|
| `Invocations` | AWS auto | Per agent ARN |
| `Latency` | AWS auto | End-to-end MicroVM round-trip |
| `SystemErrors` / `UserErrors` | AWS auto | 5xx / 4xx counts |
| `vCPUHours` / `GBHours` | AWS auto | Billing telemetry |

### 5.4 Verify metrics appear in CloudWatch

```bash
# Our custom namespace
aws cloudwatch list-metrics --namespace DemoSDLC/Agent --output table

# AgentCore native namespace
aws cloudwatch list-metrics --namespace AWS/Bedrock-AgentCore --output table

# Read a single metric's recent statistics
aws cloudwatch get-metric-statistics \
  --namespace DemoSDLC/Agent \
  --metric-name AgentLatencyMs \
  --dimensions Name=agent_id,Value=agent_1_transcriber \
              Name=workflow_template_id,Value=brd-from-audio-v1 \
  --start-time "$(date -u -v-1H +%Y-%m-%dT%H:%M:%SZ 2>/dev/null || date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ)" \
  --end-time   "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  --period 300 \
  --statistics Average Maximum
```

---

## 6. CloudWatch GenAI Dashboard

### 6.1 GenAI Dashboard (Pre-built by AWS)

AWS provides a **GenAI Dashboard** that automatically visualizes OTel traces from Bedrock AgentCore. No manual dashboard JSON needed — it auto-discovers metrics.

**Access:** CloudWatch Console → Insights → GenAI Dashboard

### 6.2 Custom Dashboard (Optional)

If you need a custom dashboard beyond the GenAI Dashboard:

```bash
# Create dashboard with key tiles
aws cloudwatch put-dashboard \
  --dashboard-name BRDDemo-Operations \
  --dashboard-body '{
    "widgets": [
      {
        "type": "metric",
        "x": 0, "y": 0, "width": 12, "height": 6,
        "properties": {
          "title": "Average Latency by Agent (ms)",
          "metrics": [
            ["DemoSDLC/Agent", "AgentLatencyMs", "agent_id", "agent_1_transcriber", {"stat": "Average"}],
            ["...", "Agent2_Drafter", {"stat": "Average"}],
            ["...", "Agent3_Reviewer", {"stat": "Average"}]
          ],
          "period": 60,
          "view": "timeSeries"
        }
      },
      {
        "type": "metric",
        "x": 12, "y": 0, "width": 12, "height": 6,
        "properties": {
          "title": "Token Consumption",
          "metrics": [
            ["DemoSDLC/Agent", "TokensInput", {"stat": "Sum"}]
          ],
          "period": 3600,
          "view": "timeSeries",
          "annotations": {
            "horizontal": [{"value": 100000, "label": "Budget Alert", "color": "#d62728"}]
          }
        }
      },
      {
        "type": "metric",
        "x": 0, "y": 6, "width": 12, "height": 6,
        "properties": {
          "title": "Agent Cost (USD)",
          "metrics": [
            ["DemoSDLC/Agent", "AgentCostUsd", {"stat": "Sum"}]
          ],
          "period": 3600,
          "view": "timeSeries"
        }
      },
      {
        "type": "metric",
        "x": 12, "y": 6, "width": 12, "height": 6,
        "properties": {
          "title": "Workflow Outcomes",
          "metrics": [
            ["BRDDemo/Workflow", "WorkflowOutcome", "Status", "APPROVED", {"stat": "Sum"}],
            ["...", "REJECTED", {"stat": "Sum"}],
            ["...", "FAILED", {"stat": "Sum"}]
          ],
          "period": 3600,
          "view": "timeSeries"
        }
      }
    ]
  }'
```

---

## 7. Logs Insights Queries

### 7.1 Query: "Show me the full event timeline of workflow X"

```sql
-- Cross-log-group: trace a single workflow across all sources
fields @timestamp, @logGroup, @message
| parse @message '"workflow_id": "*"' as workflow_id
| parse @message '"trace_id": "*"' as trace_id
| parse @message '"agent_name": "*"' as agent_name
| parse @message '"status": "*"' as status
| filter workflow_id = "wf-brd-20250115-001"
| display @timestamp, agent_name, status, @logGroup
| sort @timestamp asc
```

### 7.2 Query: "Find all workflows where Agent 2 took more than 30 seconds"

```sql
-- Target: /agentcore-demo-test1/fastapi
fields @timestamp, workflow_id, agent_name, RunDuration
| filter agent_name = "Agent2_Drafter" and RunDuration > 30000
| display workflow_id, RunDuration, @timestamp
| sort RunDuration desc
```

### 7.3 Query: "Top 10 most expensive workflows last week"

```sql
-- Target: /agentcore-demo-test1/fastapi
fields @timestamp, workflow_id, CostUSD
| stats sum(CostUSD) as workflow_total_cost by workflow_id
| sort workflow_total_cost desc
| limit 10
```

### 7.4 Query: "Workflows that failed with PII findings"

```sql
-- Target: /agentcore-demo-test1/fastapi
fields @timestamp, workflow_id, status, policy_violations
| filter status = "FAILED" and policy_violations like /PII/
| display @timestamp, workflow_id, status, policy_violations
| sort @timestamp desc
```

### 7.5 Query: "Average tokens per workflow grouped by region"

```sql
-- Target: /agentcore-demo-test1/fastapi
fields @timestamp, region, TokenUsage
| stats avg(TokenUsage) as avg_tokens, count(*) as count by region
| sort avg_tokens desc
```

---

## 8. Alarms

### 8.1 Alarm Strategy

| Failure Mode | Alarm | Severity |
|---|---|---|
| Agent producing errors | High Error Rate Per Agent | P1-Critical |
| Workflow stuck | Stuck Workflow Detection | P1-Critical |
| Runaway LLM consumption | Token Budget Exhaustion | P2-High |
| Unexpected cost spike | Cost Spike Detection | P2-High |

### 8.2 Alarm Definitions

```bash
# A1 — High Error Rate Per Agent
aws cloudwatch put-metric-alarm \
  --alarm-name BRDDemo-Agent1-HighErrorRate \
  --alarm-description "Agent1_Transcriber: error rate > 10% over 5 min" \
  --metric-name RunDuration \
  --namespace DemoSDLC/Agent \
  --statistic Average \
  --dimensions Name=jnj.agent_name,Value=Agent1_Transcriber \
  --period 300 \
  --evaluation-periods 1 \
  --threshold 60000 \
  --comparison-operator GreaterThanThreshold \
  --treat-missing-data notBreaching

# A2 — Stuck Workflow Detection
aws cloudwatch put-metric-alarm \
  --alarm-name BRDDemo-StuckWorkflow \
  --alarm-description "Workflow stuck: no completion within 10 minutes" \
  --metric-name RunDuration \
  --namespace DemoSDLC/Agent \
  --statistic Average \
  --period 300 \
  --evaluation-periods 2 \
  --threshold 600000 \
  --comparison-operator GreaterThanThreshold

# A3 — Token Budget Exhaustion
aws cloudwatch put-metric-alarm \
  --alarm-name BRDDemo-TokenBudget \
  --alarm-description "Hourly token usage > 100,000" \
  --metric-name TokenUsage \
  --namespace DemoSDLC/Agent \
  --statistic Sum \
  --period 3600 \
  --evaluation-periods 1 \
  --threshold 100000 \
  --comparison-operator GreaterThanThreshold

# A4 — Cost Spike Detection
aws cloudwatch put-metric-alarm \
  --alarm-name BRDDemo-CostSpike \
  --alarm-description "Hourly cost > $5.00" \
  --metric-name CostUSD \
  --namespace DemoSDLC/Agent \
  --statistic Sum \
  --period 3600 \
  --evaluation-periods 1 \
  --threshold 5.0 \
  --comparison-operator GreaterThanThreshold

# Composite: Any Critical Issue
aws cloudwatch put-composite-alarm \
  --alarm-name BRDDemo-Critical-Any \
  --alarm-rule "ALARM(BRDDemo-Agent1-HighErrorRate) OR ALARM(BRDDemo-StuckWorkflow) OR ALARM(BRDDemo-TokenBudget) OR ALARM(BRDDemo-CostSpike)"
```

---

## 9. Trace Propagation Verification

### 9.1 The Full Trace Chain

```
Browser (React)
  ↓ traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
  ↓ HTTP POST /api/v1/workflows

FastAPI Gateway
  ↓ Extracts traceparent, continues span
  ↓ Starts workflow via Temporal SDK

Temporal Server
  ↓ Persists trace context in workflow history
  ↓ Schedules activity

Temporal Worker
  ↓ Receives activity task with trace context
  ↓ Propagates to agent invocation

AgentCore MicroVM (Firecracker)
  ↓ BedrockAgentCoreApp creates child span
  ↓ Calls Bedrock (auto-instrumented)

Bedrock (AWS)
  ↓ X-Ray segment automatically generated
  ↓ Response returns
```

### 9.2 Verify Trace Propagation

```bash
# List recent traces in X-Ray
aws xray get-trace-summaries \
  --start-time $(date -d '1 hour ago' +%s) \
  --end-time $(date +%s) \
  --filter-expression 'service("agentcore-demo-test1")'

# Verify a specific trace spans multiple services
aws xray batch-get-traces \
  --trace-ids 'TRACE_ID_HERE'
# EXPECTED: Segments include FastAPI, Temporal, AgentCore, Bedrock
```

### 9.3 Cross-Log-Group Trace Verification

```sql
-- Verify each trace_id appears in multiple log groups
fields @timestamp, @logGroup
| parse @message '"trace_id": "*"' as trace_id
| stats count(*) as group_count by trace_id
| filter group_count < 3
# Empty result = ALL traces propagated correctly
# Non-empty = broken propagation (investigate)
```

---

## 10. Cost Control

### 10.1 CloudWatch Cost Components (OTel Stack)

| Component | Pricing Model | This Project |
|---|---|---|
| **Log Ingestion** | $0.50/GB ingested | Primary cost driver (OTel spans + application logs) |
| **Log Storage** | $0.03/GB/month | Low with 7-day retention |
| **Metrics** | $0.01 per 1,000 metrics | Auto-extracted by ADOT (free extraction, pay for storage) |
| **Metric Storage** | $0.01/metric/month (first 10K free) | ~20 metrics × 10 dimension combos = ~200 |
| **Dashboards** | $3/dashboard/month (first 3 free) | 1 GenAI + 1 custom = $3/month |
| **Alarms** | $0.10/alarm/month | 4 alarms + 1 composite = $0.50/month |
| **X-Ray Traces** | $5 per 1M traces stored | ~1K traces/day = $0.15/month |
| **ADOT Collector** | $0 (open source, runs on your EC2) | No per-collector charge |

### 10.2 Cost Comparison: EMF (Legacy) vs OTel + ADOT (V5)

| Aspect | Legacy EMF (manual) | V5 OTel + ADOT |
|---|---|---|
| **Metric API calls** | `PutMetricData` per metric (~$0.01/1K) | **FREE** — ADOT auto-extracts |
| **Log formatting** | Manual EMF JSON construction | **NONE** — standard OTel spans |
| **Developer effort** | High (EMF structure, cardinality) | **LOW** — `span.set_attribute()` only |
| **Governance** | Manual review | **Automatic** — ADOT drops non-compliant spans |
| **Error surface** | High (EMF syntax errors) | **LOW** (type-safe Python) |
| **Total monthly cost** | ~$15-25 | **~$8-15** (40% lower) |

### 10.3 Cost Optimization

```bash
# 7-day retention (demo)
aws logs put-retention-policy \
  --log-group-name /agentcore-demo-test1/fastapi \
  --retention-in-days 7

# 10% trace sampling (demo — NOT production)
OTEL_TRACES_SAMPLER=traceidratio
OTEL_TRACES_SAMPLER_ARG=0.1

# Filter DEBUG logs (emit INFO+ only)
# Set in Python logging config or OTel log level
```

### 10.4 Cost Control Checklist

- [ ] Retention is 7 days (demo) — `put-retention-policy --retention-in-days 7`
- [ ] Trace sampling is 10% (demo) — `OTEL_TRACES_SAMPLER=traceidratio; OTEL_TRACES_SAMPLER_ARG=0.1`
- [ ] DEBUG logs disabled — OTel log level `INFO`
- [ ] No high-cardinality dimensions in `jnj.*` attributes
- [ ] Logs Insights queries use time-boxed filters

---

## 11. Tech Reference

### 11.1 Official Documentation

| Topic | URL |
|---|---|
| OpenTelemetry Python SDK | https://opentelemetry.io/docs/languages/python/ |
| AWS Distro for OpenTelemetry (ADOT) | https://aws-otel.github.io/docs/getting-started/collector |
| ADOT Collector CloudWatch Exporter | https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/exporter/awsemfexporter |
| CloudWatch GenAI Dashboard | https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-GenAI-Dashboard.html |
| CloudWatch Logs Insights | https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/AnalyzingLogData.html |
| CloudWatch Alarms | https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/AlarmThatSendsEmail.html |
| AWS X-Ray | https://docs.aws.amazon.com/xray/latest/devguide/aws-xray.html |
| W3C Trace Context | https://www.w3.org/TR/trace-context/ |

### 11.2 Glossary

| Term | Definition |
|---|---|
| **OTel** | OpenTelemetry — open-source observability framework for traces, metrics, logs |
| **ADOT** | AWS Distro for OpenTelemetry — AWS-managed distribution of the OTel Collector |
| **OTLP** | OpenTelemetry Protocol — gRPC/HTTP protocol for exporting telemetry data |
| **Span** | A single operation within a trace (e.g., `Agent1_Transcriber.run()`) |
| **Trace** | End-to-end request flow across multiple services, identified by `trace_id` |
| **Attribute** | Key-value pair attached to a span (e.g., `jnj.workflow_id = "wf-001"`) |
| **Collector** | ADOT process that receives OTLP data, processes it, exports to backends |
| **EMF** | Embedded Metric Format — CloudWatch metric format; **auto-generated by ADOT**, NOT written by developer |
| **Cardinality** | Number of unique values in a dimension set |
| **Governance Gate** | ADOT filter that drops spans missing mandatory attributes |

### 11.3 Related Documents

| Document | Why Read It |
|---|---|
| `ARCHITECTURE_DATAFLOW_GUIDE.md` Section 4.3 | Three-tier observability model in detail |
| `Prompt_02_Agent_Telemetry.md` | Step-by-step implementation guide for agent instrumentation |
| `Deliverable_2_Reference_Strands_Agent_Code.md` Section 4 | V5 payload builder (raw dict, 25 fields) |
| `V5_TELEMETRY_INTEGRATION_ANALYSIS.md` | Deep OTel + ADOT architecture analysis |

---

## 12. Quick Start Checklist

```
[ ] Day 1: Create 5 log groups with 7-day retention
[ ] Day 1: Deploy ADOT Collector (docker-compose on EC2)
[ ] Day 1: Configure `OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317`
[ ] Day 1: Verify ADOT health: `curl http://localhost:55679/health`
[ ] Day 2: Add `jnj.*` attributes to agent spans (Deliverable 2 Section 4)
[ ] Day 2: Verify spans appear in X-Ray console
[ ] Day 2: Verify OTel traces visible in CloudWatch Logs Insights
[ ] Day 3: Instrument all 3 agents with full `jnj.*` attribute set
[ ] Day 3: Verify CloudWatch Metrics console shows DemoSDLC/Agent namespace
[ ] Day 3: Check GenAI Dashboard renders correctly
[ ] Day 4: Create custom dashboard (optional — GenAI covers most needs)
[ ] Day 4: Deploy 4 alarms + 1 composite alarm
[ ] Day 5: Run trace propagation verification (cross-log-group query)
[ ] Day 5: Run cost estimation — establish baseline
[ ] Day 6: Tune alarm thresholds based on observed patterns
[ ] Day 7: Document any anomalies
```

---

## Appendix A: Metrics Namespace Design

```
BRDDemo/
├── Agents/                          ← Per-agent operational metrics
│   ├── RunDuration (ms)             ← jnj.workflow_id, jnj.agent_name
│   ├── TokenUsage (count)           ← jnj.workflow_id, jnj.agent_name
│   ├── CostUSD (dollars)          ← jnj.workflow_id, jnj.agent_name
│   └── QualityScore (0-100)       ← jnj.workflow_id, jnj.agent_name
│
└── Workflow/                        ← End-to-end workflow metrics
    ├── WorkflowOutcome (count)    ← Status (APPROVED/REJECTED/TERMINATED/FAILED)
    ├── HITLRoundCount (count)     ← jnj.workflow_id
    └── WorkflowDuration (ms)      ← jnj.workflow_id, Status
```

All metrics are **auto-extracted by ADOT** from OTel span attributes. **No manual EMF, no `PutMetricData`.**

---

## Appendix B: Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| No metrics in CloudWatch | ADOT Collector not running | `docker ps` → check `adot-collector` container |
| Spans not appearing | Wrong OTLP endpoint | Verify `OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317` |
| Missing `jnj.workflow_id` | Governance gate dropped span | Add `span.set_attribute("jnj.workflow_id", ...)` |
| High CloudWatch cost | No trace sampling | Set `OTEL_TRACES_SAMPLER=traceidratio; OTEL_TRACES_SAMPLER_ARG=0.1` |
| Traces broken (1 log group only) | Trace context not propagated | Check W3C `traceparent` header at each hop |
| GenAI Dashboard empty | No vended logs from AgentCore | Verify Bedrock AgentCore permissions for CloudWatch |

---

*End of Deliverable 7: CloudWatch Telemetry Guide (V5)*

*Version 5.0 — All EMF code examples replaced with OTel + ADOT patterns. ADOT Collector auto-generates EMF from OTel spans. Agent developer surface: `span.set_attribute()` only.*
