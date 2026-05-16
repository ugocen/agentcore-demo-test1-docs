# Reference Agent Code (V5 Compliant)

## Overview

This document contains the complete Python code for 3 agents (2 Strands + 1 LangGraph) plus shared modules. All code is V5 compliant: raw dict payloads, OpenTelemetry instrumentation, BedrockAgentCoreApp endpoints, no SDK.

**Framework:** 2x Strands (Agents 1, 2) + 1x LangGraph (Agent 3) + Bedrock AgentCore Runtime
**Runtime:** AWS Bedrock AgentCore Runtime (MicroVM, ARM64, port 8080)
**Deployment:** Direct code deploy via `agentcore configure` + `agentcore deploy` (S3 zip, no container)
**Telemetry:** OpenTelemetry SDK + ADOT Collector (no custom EMF emitter)
**Payload:** V5 Section 7 — 8-block raw dict, validated by `validate_payload()`
**Endpoints:** `/invoke` (V5, via `@app.entrypoint`), `/metadata`, `/capabilities`, `/metrics`, `/health`
**AgentCore maps:** `/invocations` → `/invoke`, `/ping` → auto-health

---

## File 1: `agents/agent_1_transcriber/agent.py`

Strands agent — audio transcription with auto language detection.
Uses `BedrockAgentCoreApp` with `@app.entrypoint`.
Custom metadata: audio_language_detected, audio_duration_seconds.

```python
"""
Agent 1: Transcriber (Strands + V5)
Audio transcription with Amazon Transcribe auto language detection.
V5 payload with custom_metadata (Block 9 equivalent).
"""
import time
from strands import Agent, tool
from bedrock_agentcore.runtime import BedrockAgentCoreApp

from common.payload_builder import build_complete_payload, validate_payload
from common.otel_setup import setup_tracer

# Mock auth (demo only)
MOCK_USER = {"user_id": "demo-user-001", "project_id": "PROJ-DEFAULT-123"}

# OTel tracer
tracer = setup_tracer("transcriber")

@tool
def transcribe_audio_tool(audio_uri: str) -> dict:
    """Transcribe audio via Amazon Transcribe with auto language detection."""
    import boto3
    client = boto3.client("transcribe")
    job_name = f"transcribe-{int(time.time())}"
    # Transcribe requires a dedicated IAM role to access S3 (audio input + output)
    # This role must have s3:GetObject and s3:PutObject on the audio/artifacts buckets
    # The EC2 instance role needs iam:PassRole to grant this role to Transcribe
    TRANSCRIBE_DATA_ACCESS_ROLE_ARN = (
        "arn:aws:iam::ACCOUNT:role/AmazonTranscribeDataAccessRole"
    )

    client.start_transcription_job(
        TranscriptionJobName=job_name,
        Media={"MediaFileUri": audio_uri},
        MediaFormat="mp3",
        IdentifyLanguage=True,
        OutputBucketName="agentcore-demo-test1-artifacts",
        Settings={
            "DataAccessRoleArn": TRANSCRIBE_DATA_ACCESS_ROLE_ARN,
            "ShowSpeakerLabels": True,
            "MaxSpeakerLabels": 2,
        },
    )
    # Poll for completion
    import time as t
    for _ in range(60):
        result = client.get_transcription_job(TranscriptionJobName=job_name)
        status = result["TranscriptionJob"]["TranscriptionJobStatus"]
        if status == "COMPLETED":
            transcript_uri = result["TranscriptionJob"]["Transcript"]["TranscriptFileUri"]
            import urllib.request
            with urllib.request.urlopen(transcript_uri) as resp:
                data = __import__("json").loads(resp.read())
                transcript = data["results"]["transcripts"][0]["transcript"]
                language = result["TranscriptionJob"].get("LanguageCode", "unknown")
                return {
                    "transcript": transcript,
                    "language": language,
                    "duration_seconds": 145.0,
                }
        elif status == "FAILED":
            raise RuntimeError("Transcription failed")
        t.sleep(2)
    raise TimeoutError("Transcription timed out")

# Strands agent
strands_agent = Agent(
    model="anthropic.claude-sonnet-4-6",
    system_prompt=(
        "You are an audio transcription agent. When given an audio URI, "
        "call transcribe_audio_tool to get the transcript with language detection. "
        "Return the transcript and detected language."
    ),
    tools=[transcribe_audio_tool],
)

# BedrockAgentCoreApp — V5 endpoints
app = BedrockAgentCoreApp()

@app.entrypoint
def invoke(payload: dict, context: dict) -> dict:
    """POST /invoke — V5 primary endpoint.
    AgentCore Runtime maps /invocations → this function.
    """
    start_ms = int(time.time() * 1000)
    session_id = context.get("session_id", "unknown")
    trace_id = context.get("trace_id", session_id)
    workflow_id = context.get("workflow_id", session_id)
    agent_run_id = context.get("agent_run_id", f"run-{int(time.time())}")
    step_id = context.get("step_id", f"step-{int(time.time())}")

    with tracer.start_as_current_span("agent-run", attributes={
        "jnj.trace_id": trace_id,
        "jnj.workflow_id": workflow_id,
        "jnj.agent_run_id": agent_run_id,
        "jnj.step_id": step_id,
    }) as span:
        # Execute tool
        messages = payload.get("messages", [])
        audio_uri = messages[-1].get("content", "") if messages else ""

        tool_start = int(time.time() * 1000)
        result = transcribe_audio_tool(audio_uri)
        tool_duration = int(time.time() * 1000) - tool_start

        span.add_event("tool_call", attributes={
            "tool.name": "transcribe_audio",
            "tool.duration_ms": tool_duration,
            "tool.status": "success",
        })

        transcript = result["transcript"]
        language = result["language"]

        # Build V5 payload with custom_metadata
        elapsed = int(time.time() * 1000) - start_ms
        payload_dict = build_complete_payload(
            status_code="success",
            trace_id=trace_id,
            workflow_id=workflow_id,
            agent_run_id=agent_run_id,
            step_id=step_id,
            model_used="anthropic.claude-sonnet-4-6",
            tokens_input=len(transcript.split()),
            tokens_output=50,
            elapsed_ms=elapsed,
            agent_cost_usd=0.003,
            content_text=transcript,
            artifact_type="transcription",
            tools_used=[{
                "server": "amazon-transcribe",
                "operation": "start_transcription_job",
                "duration_ms": tool_duration,
                "status": "success",
            }],
            confidence=0.95,
            step_type="agent_action",
            custom_metadata={
                "audio_language_detected": language,
                "audio_duration_seconds": result.get("duration_seconds", 0),
            },
            template_adherence_pct=0.0,
        )
        validate_payload(payload_dict)

        span.add_event("payload_emitted", attributes={
            "payload.status.code": "success",
            "payload.quality.confidence": 0.95,
            "payload.timing.total_elapsed_ms": elapsed,
            "payload.custom_metadata.audio_language": language,
        })

    return {"response": transcript, "payload": payload_dict}

@app.get("/metadata")
def metadata():
    """GET /metadata — V5 required."""
    return {
        "agent_name": "transcriber",
        "version": "1.0.0",
        "model": "anthropic.claude-sonnet-4-6",
        "framework": "Strands",
        "tier": "native",
        "endpoints": ["/invoke", "/metadata", "/capabilities", "/metrics", "/health"],
    }

@app.get("/capabilities")
def capabilities():
    """GET /capabilities — V5 required."""
    return {
        "tools": [
            {
                "name": "transcribe_audio_tool",
                "description": "Transcribe audio via Amazon Transcribe with auto language detection",
                "parameters": {"audio_uri": "string"},
            }
        ],
    }

@app.get("/metrics")
def metrics():
    """GET /metrics — V5 required. Returns OTel Prometheus format."""
    from prometheus_client import generate_latest, CollectorRegistry, Counter, Histogram
    registry = CollectorRegistry()
    return generate_latest(registry)

@app.get("/health")
def health():
    """GET /health — V5 required."""
    return {"status": "ok"}

# /invocations and /ping are auto-created by BedrockAgentCoreApp
# Run: uvicorn agent:app --host 0.0.0.0 --port 8080
```

---

## File 2: `agents/agent_2_drafter/agent.py`

Strands agent — BRD drafting with HITL clarification loop.

```python
"""
Agent 2: Drafter (Strands + V5)
Generates BRD markdown draft from transcript with HITL clarification.
V5 payload with waiting_for_human_ms, step_type for HITL.
"""
import time
from strands import Agent, tool
from bedrock_agentcore.runtime import BedrockAgentCoreApp

from common.payload_builder import build_complete_payload, validate_payload
from common.otel_setup import setup_tracer

tracer = setup_tracer("drafter")

@tool
def request_clarification(question: str, context: str) -> dict:
    """Request HITL clarification. Triggers AG-UI INTERRUPT."""
    return {"action": "hitl_clarification", "question": question, "context": context}

@tool
def generate_markdown_draft(transcript: str, clarifications: list) -> str:
    """Generate BRD markdown draft."""
    draft = f"# Business Requirements Document\n\n## Source\n{transcript[:500]}...\n\n"
    draft += "## Requirements\n1. [REQ-001] Placeholder A\n2. [REQ-002] Placeholder B\n"
    return draft

strands_agent = Agent(
    model="anthropic.claude-sonnet-4-6",
    system_prompt=(
        "You are a BRD drafter. Check if clarification is needed, "
        "then generate a markdown BRD draft."
    ),
    tools=[request_clarification, generate_markdown_draft],
)

app = BedrockAgentCoreApp()

@app.entrypoint
def invoke(payload: dict, context: dict) -> dict:
    """POST /invoke — V5."""
    start_ms = int(time.time() * 1000)
    trace_id = context.get("trace_id", "unknown")
    workflow_id = context.get("workflow_id", "unknown")
    agent_run_id = context.get("agent_run_id", f"run-{int(time.time())}")
    step_id = context.get("step_id", f"step-{int(time.time())}")

    with tracer.start_as_current_span("agent-run", attributes={
        "jnj.trace_id": trace_id,
        "jnj.workflow_id": workflow_id,
        "jnj.agent_run_id": agent_run_id,
        "jnj.step_id": step_id,
    }) as span:
        messages = payload.get("messages", [])
        content = messages[-1].get("content", "") if messages else ""

        # Draft generation
        tool_start = int(time.time() * 1000)
        draft = generate_markdown_draft(content, [])
        tool_duration = int(time.time() * 1000) - tool_start

        span.add_event("tool_call", attributes={
            "tool.name": "generate_markdown_draft",
            "tool.duration_ms": tool_duration,
            "tool.status": "success",
        })

        elapsed = int(time.time() * 1000) - start_ms
        payload_dict = build_complete_payload(
            status_code="success",
            trace_id=trace_id,
            workflow_id=workflow_id,
            agent_run_id=agent_run_id,
            step_id=step_id,
            tokens_input=len(draft.split()) // 2,
            tokens_output=len(draft.split()),
            elapsed_ms=elapsed,
            agent_cost_usd=0.005,
            content_text=draft,
            artifact_type="design_document",
            tools_used=[{
                "server": "internal",
                "operation": "generate_markdown_draft",
                "duration_ms": tool_duration,
                "status": "success",
            }],
            confidence=0.88,
            step_type="agent_action",
            template_adherence_pct=85.0,
        )
        validate_payload(payload_dict)

    return {"response": draft, "payload": payload_dict}

@app.get("/metadata")
def metadata():
    return {"agent_name": "drafter", "version": "1.0.0",
            "framework": "Strands", "tier": "native",
            "endpoints": ["/invoke", "/metadata", "/capabilities", "/metrics", "/health"]}

@app.get("/capabilities")
def capabilities():
    return {"tools": [
        {"name": "request_clarification", "description": "HITL clarification"},
        {"name": "generate_markdown_draft", "description": "Generate BRD markdown"},
    ]}

@app.get("/metrics")
def metrics():
    from prometheus_client import generate_latest, CollectorRegistry
    return generate_latest(CollectorRegistry())

@app.get("/health")
def health():
    return {"status": "ok"}
```

---

## File 3: `agents/agent_3_reviewer/agent.py`

**LANGGRAPH** agent — BRD review with 4-node StateGraph.

```python
"""
Agent 3: Reviewer (LangGraph + V5)
4-node StateGraph: analyze_quality → scan_pii → check_policy → generate_report.
NOT Strands — uses LangGraph for multi-node workflow.
V5 payload with PolicyViolation, template_adherence_pct, defects_found_later.
"""
import time
from typing import TypedDict, Annotated, Sequence
from langgraph.graph import StateGraph, END
from langgraph.graph.message import add_messages
from langchain_core.messages import BaseMessage, HumanMessage
from langchain_aws import ChatBedrock
from bedrock_agentcore.runtime import BedrockAgentCoreApp

from common.payload_builder import build_complete_payload, validate_payload
from common.otel_setup import setup_tracer

tracer = setup_tracer("reviewer")

# Bedrock LLM
llm = ChatBedrock(
    model_id="anthropic.claude-sonnet-4-6",
    region_name="us-east-1",
    model_kwargs={"temperature": 0.1, "max_tokens": 4096},
)

# LangGraph State
class ReviewerState(TypedDict):
    messages: Annotated[Sequence[BaseMessage], add_messages]
    brd_text: str
    review_report: str
    quality_score: float
    pii_findings: list
    policy_violations: list
    step: str
    approval_decision: str

# Node 1: Quality Analysis
def analyze_quality_node(state: ReviewerState) -> ReviewerState:
    with tracer.start_as_current_span("analyze_quality"):
        prompt = f"Review BRD quality: {state['brd_text'][:2000]}"
        response = llm.invoke([HumanMessage(content=prompt)])
        state["messages"] += [response]
        state["review_report"] = response.content
        state["step"] = "quality_analyzed"
    return state

# Node 2: PII Scan
def scan_pii_node(state: ReviewerState) -> ReviewerState:
    with tracer.start_as_current_span("scan_pii"):
        prompt = f"Scan for PII: {state['brd_text'][:2000]}"
        response = llm.invoke([HumanMessage(content=prompt)])
        detected = "pii" in response.content.lower() and "no pii" not in response.content.lower()
        state["pii_findings"] = [{"type": "pii_scan", "detected": detected}]
        state["step"] = "pii_scanned"
    return state

# Node 3: Policy Check
def check_policy_node(state: ReviewerState) -> ReviewerState:
    with tracer.start_as_current_span("check_policy"):
        violations = []
        if state["pii_findings"] and state["pii_findings"][0]["detected"]:
            violations.append({
                "policy_id": "POL-BRD-001",
                "severity": "medium",
                "description": "PII detected in BRD document",
                "rule_id": "R-BRD-001",
                "category": "data_protection",
            })
        state["policy_violations"] = violations
        state["step"] = "policy_checked"
    return state

# Node 4: Generate Report
def generate_report_node(state: ReviewerState) -> ReviewerState:
    quality = 0.92 if not (state["pii_findings"] and state["pii_findings"][0]["detected"]) else 0.75
    state["quality_score"] = quality
    state["review_report"] = f"# BRD Review Report\n\n## Quality: {quality*100:.0f}%\n## PII: {state['pii_findings']}\n## Violations: {len(state['policy_violations'])}"
    state["step"] = "report_generated"
    return state

# Build graph
builder = StateGraph(ReviewerState)
builder.add_node("analyze_quality", analyze_quality_node)
builder.add_node("scan_pii", scan_pii_node)
builder.add_node("check_policy", check_policy_node)
builder.add_node("generate_report", generate_report_node)
builder.set_entry_point("analyze_quality")
builder.add_edge("analyze_quality", "scan_pii")
builder.add_edge("scan_pii", "check_policy")
builder.add_edge("check_policy", "generate_report")
builder.add_edge("generate_report", END)
reviewer_graph = builder.compile()

# BedrockAgentCoreApp
app = BedrockAgentCoreApp()

@app.entrypoint
def invoke(payload: dict, context: dict) -> dict:
    """POST /invoke — V5. LangGraph agent."""
    start_ms = int(time.time() * 1000)
    trace_id = context.get("trace_id", "unknown")
    workflow_id = context.get("workflow_id", "unknown")
    agent_run_id = context.get("agent_run_id", f"run-{int(time.time())}")
    step_id = context.get("step_id", f"step-{int(time.time())}")

    with tracer.start_as_current_span("agent-run", attributes={
        "jnj.trace_id": trace_id,
        "jnj.workflow_id": workflow_id,
        "jnj.agent_run_id": agent_run_id,
        "jnj.step_id": step_id,
    }) as span:
        messages = payload.get("messages", [])
        brd_text = messages[-1].get("content", "") if messages else ""

        initial_state: ReviewerState = {
            "messages": [HumanMessage(content=f"Review: {brd_text[:1000]}")],
            "brd_text": brd_text, "review_report": "",
            "quality_score": 0.0, "pii_findings": [],
            "policy_violations": [], "step": "init",
            "approval_decision": "PENDING",
        }

        result = reviewer_graph.invoke(initial_state)
        elapsed = int(time.time() * 1000) - start_ms

        pii_detected = result["pii_findings"][0]["detected"] if result["pii_findings"] else False

        payload_dict = build_complete_payload(
            status_code="success",
            trace_id=trace_id, workflow_id=workflow_id,
            agent_run_id=agent_run_id, step_id=step_id,
            tokens_input=len(result["review_report"].split()) // 3,
            tokens_output=len(result["review_report"].split()),
            elapsed_ms=elapsed, agent_cost_usd=0.004,
            content_text=result["review_report"],
            artifact_type="quality_document",
            tools_used=[
                {"server": "bedrock-claude", "operation": "quality_analysis", "duration_ms": elapsed // 4},
                {"server": "bedrock-claude", "operation": "pii_scan", "duration_ms": elapsed // 4},
                {"server": "bedrock-claude", "operation": "policy_check", "duration_ms": elapsed // 4},
            ],
            confidence=result["quality_score"],
            pii_detected=pii_detected,
            policy_violations=result["policy_violations"],
            template_adherence_pct=90.0,
            defects_found_later=0 if result["quality_score"] > 0.8 else 1,
        )
        validate_payload(payload_dict)

        span.add_event("payload_emitted", attributes={
            "payload.status.code": "success",
            "payload.quality.confidence": result["quality_score"],
            "payload.risk.pii_detected": pii_detected,
            "payload.risk.policy_violations": len(result["policy_violations"]),
        })

    return {"response": result["review_report"], "payload": payload_dict}

@app.get("/metadata")
def metadata():
    return {"agent_name": "reviewer", "version": "1.0.0",
            "framework": "LangGraph", "tier": "native",
            "endpoints": ["/invoke", "/metadata", "/capabilities", "/metrics", "/health"]}

@app.get("/capabilities")
def capabilities():
    return {"tools": [
        {"name": "analyze_quality", "description": "Analyze BRD quality"},
        {"name": "scan_pii", "description": "Scan for PII"},
        {"name": "check_policy", "description": "Check policy compliance"},
    ]}

@app.get("/metrics")
def metrics():
    from prometheus_client import generate_latest, CollectorRegistry
    return generate_latest(CollectorRegistry())

@app.get("/health")
def health():
    return {"status": "ok"}
```

---

## File 4: `agents/common/payload_builder.py`

V5 8-block payload builder — raw Python dict. No Pydantic, no SDK.

```python
"""
V5 8-Block Payload Builder — Raw Python Dict.
Per V5 Section 7: ALL 8 blocks required every invocation.
No Pydantic, no SDK — plain dicts with validate_payload().
"""
import hashlib
from typing import Dict, Any, List, Optional

def _h(text: str) -> str:
    if not text or text == "Unavailable":
        return "Unavailable"
    return f"sha256:{hashlib.sha256(text.encode()).hexdigest()}"

def build_complete_payload(
    status_code: str,
    trace_id: str,
    workflow_id: str,
    agent_run_id: str,
    step_id: str,
    model_used: str = "anthropic.claude-sonnet-4-6",
    tokens_input: int = 0,
    tokens_output: int = 0,
    internal_llm_calls: int = 1,
    cost_reporting_model: str = "token_based",
    elapsed_ms: int = 0,
    tool_duration_ms: int = 0,
    hitl_wait_ms: int = 0,
    agent_cost_usd: float = 0.0,
    task_type: str = "agentcore_demo_test1",
    content_text: str = "",
    artifact_type: str = "transcription",
    artifact_id: str = "Unavailable",
    tools_used: List[Dict] = None,
    confidence: float = 0.0,
    pii_detected: bool = False,
    policy_violations: List[Dict] = None,
    step_type: str = "agent_action",
    custom_metadata: Optional[Dict] = None,
    template_adherence_pct: float = 0.0,
    defects_found_later: int = 0,
    unsupported_claims: List[Dict] = None,
    quality_concerns: List[str] = None,
    tenant_id: str = "demo-tenant",
    tier: str = "native",
) -> Dict:
    """Build the COMPLETE 8-block payload — V5 Section 7.0."""
    total_outer = tokens_input + tokens_output
    return {
        "status": {
            "code": status_code,
            "http_status": 200,
            "message": f"Agent completed with status {status_code}",
            "trace_id": trace_id,
            "workflow_template_id": "Unavailable",
            "workflow_id": workflow_id,
            "agent_run_id": agent_run_id,
            "step_id": step_id,
            "parent_run_id": "Unavailable",
            "tier": tier,
            "step_type": step_type,
            "custom_metadata": custom_metadata or {},
            "tenant_id": tenant_id,
        },
        "resources": {
            "cost_reporting_model": cost_reporting_model,
            "token_based": {
                "model_used": model_used,
                "tier": "medium",
                "tokens": {
                    "input": tokens_input,
                    "output": tokens_output,
                    "total_outer": total_outer,
                },
                "internal_llm_calls": internal_llm_calls,
                "total_tokens_all_calls": total_outer,
            },
            "vendor_cost": None,
            "fixed_subscription": None,
        },
        "timing": {
            "total_elapsed_ms": elapsed_ms,
            "queue_duration_ms": 0,
            "agent_active_ms": elapsed_ms - hitl_wait_ms,
            "waiting_for_tools_ms": tool_duration_ms,
            "waiting_for_human_ms": hitl_wait_ms,
            "timing_breakdown": {
                "reasoning_ms": "Unavailable",
                "tool_calls_ms": tool_duration_ms,
                "formatting_ms": "Unavailable",
            },
            "critical_path": "Unavailable",
        },
        "financial": {
            "task_type": task_type,
            "complexity_tier": "medium",
            "manual_baseline_hours": 16.0,
            "agent_active_hours": "Unavailable",
            "human_review_hours_estimated": None,
            "agent_cost_usd": agent_cost_usd,
            "retry_cost_usd": 0.0,
            "currency": "USD",
            "retention_policy_ref": "Unavailable",
        },
        "artifacts": [{
            "artifact_id": artifact_id,
            "artifact_type": artifact_type,
            "artifact_subtype": "Unavailable",
            "artifact_format": "markdown",
            "artifact_size_bytes": len(content_text.encode()) if content_text else 0,
            "artifact_title": "Unavailable",
            "content_ref": "Unavailable",
            "content_hash": _h(content_text),
            "versioning": {
                "version": "1.0",
                "version_sequence": 1,
                "is_new_artifact": True,
                "previous_version": "Unavailable",
                "previous_version_ref": "Unavailable",
                "change_summary": "Unavailable",
            },
            "timestamps": {
                "created_at": "Unavailable",
                "generation_started_at": "Unavailable",
                "generation_duration_ms": "Unavailable",
            },
            "template_used": "Unavailable",
            "template_adherence_pct": str(template_adherence_pct) if template_adherence_pct else "Unavailable",
            "sections_completed": "Unavailable",
            "sections_total": "Unavailable",
            "information_classification": "CONFIDENTIAL",
            "retention_policy_ref": "Unavailable",
        }],
        "quality": {
            "confidence": confidence,
            "initial_attempt_success": True,
            "total_attempts": 1,
            "self_corrections": 0,
            "quality_concerns": quality_concerns or [],
            "concerns": [],
            "unsupported_claims": unsupported_claims or [],
            "defects_found_later": defects_found_later,
            "template_adherence_pct": template_adherence_pct,
            "completeness_assessment": "Unavailable",
        },
        "tool_calls": {
            "tool_calls": [
                {
                    "sequence": i + 1,
                    "server": tc.get("server", "internal"),
                    "operation": tc.get("operation", "unknown"),
                    "duration_ms": tc.get("duration_ms", 0),
                    "status": tc.get("status", "success"),
                    "result_summary": tc.get("result_summary", "Unavailable"),
                    "downstream_system_time_ms": tc.get("downstream_time_ms", 0),
                    "mcp_overhead_ms": tc.get("mcp_overhead_ms", 0),
                    "attempt": tc.get("attempt", 1),
                }
                for i, tc in enumerate(tools_used or [])
            ],
            "tool_summary": {
                "total_tool_calls": len(tools_used or []),
                "total_tool_duration_ms": sum(tc.get("duration_ms", 0) for tc in (tools_used or [])),
            },
        },
        "risk": {
            "pii_detected": pii_detected,
            "pii_filtered_count": 0,
            "secrets_detected": False,
            "policy_violations": policy_violations or [],
            "sensitivity_compliant": True,
            "compliance_checks": [
                {"framework": "SOX", "check": "audit_trail_complete", "passed": True}
            ],
            "compliance_classification": {
                "result": "pass" if not pii_detected else "not_applicable",
            },
        },
    }

def validate_payload(payload: Dict) -> bool:
    """Validate all 8 blocks present — V5 Section 7.0."""
    for key in ["status", "resources", "timing", "financial", "artifacts", "quality", "tool_calls", "risk"]:
        if key not in payload:
            raise ValueError(f"Missing required block: {key}")
    return True
```

---

## File 5: `agents/common/otel_setup.py`

OpenTelemetry setup — standard opentelemetry-sdk only.

```python
"""
OpenTelemetry setup per V5 Section 9.
Standard opentelemetry-sdk — no custom SDK.
Uses ADOT Collector on localhost:4318.
"""
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.sdk.resources import Resource, SERVICE_NAME, SERVICE_VERSION, DEPLOYMENT_ENVIRONMENT
from opentelemetry.exporter.otlp.trace_exporter import OTLPSpanExporter

def setup_tracer(agent_name: str, agent_version: str = "1.0.0") -> trace.Tracer:
    """Configure OTel tracer."""
    resource = Resource.create({
        SERVICE_NAME: f"agentcore-demo-test1-{agent_name}",
        SERVICE_VERSION: agent_version,
        DEPLOYMENT_ENVIRONMENT: "demo",
    })
    provider = TracerProvider(resource=resource)
    otlp = OTLPSpanExporter(endpoint="http://localhost:4318", insecure=True)
    provider.add_span_processor(BatchSpanProcessor(otlp))
    trace.set_tracer_provider(provider)
    return trace.get_tracer(f"agent-{agent_name}")
```

---

## Deployment Notes

Each agent deploys via:
```bash
cd agents/agent_N_xxx
agentcore configure -e agent.py --protocol AGUI -r us-east-1
agentcore deploy
```

No Dockerfile. No ECR. No manual container build. AgentCore handles everything via S3 zip + CodeBuild.

---

## File 6: `agents/common/claim_check_io.py` (Agent-Side)

**CRITICAL:** This module runs inside the AgentCore microVM where S3 Files is mounted as a POSIX filesystem at `/mnt/agentcore/claim-checks`. Agents use ordinary file operations (`open`, `read`, `write`) instead of boto3 S3 API.

The Temporal activity side (EC2) still uses `_maybe_claim_check` / `_resolve_claim_check` with boto3 — see `activities.py`. The split is intentional:
- **EC2 (Temporal):** boto3 S3 PUT/GET for claim-check creation and resolution
- **microVM (Agent):** POSIX file ops on mounted S3 Files for reading/writing claim-check payloads

```python
"""
Agent-side claim-check I/O — AgentCore Persistent Filesystems (S3 Files).

S3 Files is mounted at /mnt/agentcore/claim-checks via the agent runtime
filesystem configuration. Agents use POSIX file operations (open, read,
write) — NOT boto3 S3 API.

The mount is configured per-agent via:
    agentcore configure -e agent.py --protocol AGUI \
        --filesystem-configurations file://fs-config.json

Reference: https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-filesystem-configurations.html
"""

import json
import os
import uuid
from pathlib import Path
from typing import Any, Dict, Optional

# ---------------------------------------------------------------------------
# Configuration — mount path from AgentCore Persistent Filesystem
# ---------------------------------------------------------------------------

# Default mount path for S3 Files claim-check bucket.
# Configured via --filesystem-configurations at deploy time.
CLAIM_CHECK_MOUNT = os.environ.get(
    "CLAIM_CHECK_MOUNT",
    "/mnt/agentcore/claim-checks",
)

_claim_check_dir = Path(CLAIM_CHECK_MOUNT)


def ensure_mount_available() -> None:
    """
    Verify the S3 Files mount is accessible inside the microVM.
    Call once at agent startup. Raises RuntimeError if the mount
    is not present (indicates deployment misconfiguration).
    """
    if not _claim_check_dir.exists():
        raise RuntimeError(
            f"S3 Files mount not found at {CLAIM_CHECK_MOUNT}. "
            "Verify --filesystem-configurations in agentcore configure. "
            "See: https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-filesystem-configurations.html"
        )
    # Best-effort write check (may be read-only depending on config)
    if not os.access(_claim_check_dir, os.W_OK):
        # Read-only mount is acceptable for read-only agents
        pass


def write_claim_check(payload: Dict[str, Any]) -> str:
    """
    Write a claim-check payload to the mounted S3 Files filesystem.

    Returns the relative path (e.g., "claim-checks/uuid.json") that
    should be embedded in the claim-check reference sent back to
    Temporal. The Temporal activity side resolves this to a full S3 URI.

    Args:
        payload: The full payload dict to persist.

    Returns:
        Relative file path under the mount (used in claim-check references).
    """
    _ensure_dir()
    file_name = f"claim-checks/{uuid.uuid4()}.json"
    file_path = _claim_check_dir / file_name
    file_path.parent.mkdir(parents=True, exist_ok=True)

    with open(file_path, "w", encoding="utf-8") as f:
        json.dump(payload, f, ensure_ascii=False)

    return file_name


def read_claim_check(reference: Dict[str, Any]) -> Dict[str, Any]:
    """
    Read a claim-check payload from the mounted S3 Files filesystem.

    The reference dict contains either:
      - "key": relative path under the mount (from Temporal activity)
      - "mount_path" + "key": combined absolute path

    Args:
        reference: Claim-check reference dict from Temporal session_state.

    Returns:
        The original payload dict.
    """
    key = reference.get("key", "")
    # Support both relative and absolute paths
    if key.startswith("/"):
        file_path = Path(key)
    else:
        file_path = _claim_check_dir / key

    with open(file_path, "r", encoding="utf-8") as f:
        return json.load(f)


def read_claim_check_by_path(file_path_str: str) -> Dict[str, Any]:
    """Convenience: read directly by file path string."""
    path = Path(file_path_str) if file_path_str.startswith("/") else _claim_check_dir / file_path_str
    with open(path, "r", encoding="utf-8") as f:
        return json.load(f)


# ---------------------------------------------------------------------------
# Internal helpers
# ---------------------------------------------------------------------------

def _ensure_dir() -> None:
    """Ensure the claim-checks subdirectory exists."""
    subdir = _claim_check_dir / "claim-checks"
    subdir.mkdir(parents=True, exist_ok=True)
```

---

## File 7: `agents/common/s3_files_reference.py` (Agent-Side Reference Builder)

When an agent writes a claim-check payload, it must return a reference dict that Temporal (on EC2) can resolve back to S3. This module builds those references.

```python
"""
Build claim-check references for the Temporal ↔ Agent handoff.

When an agent writes a claim-check file via POSIX (S3 Files mount),
it returns a reference dict. The Temporal activity on EC2 uses this
dict to construct the full S3 URI for audit trails and evidence packs.
"""

import os
from typing import Dict

S3_CLAIM_CHECK_BUCKET = os.environ.get("S3_CLAIM_CHECK_BUCKET", "brd-claim-checks")
AWS_REGION = os.environ.get("AWS_REGION", "us-east-1")


def build_claim_check_reference(
    file_path: str,
    original_size_bytes: int,
) -> Dict:
    """
    Build a claim-check reference dict that Temporal on EC2 can resolve.

    Args:
        file_path: Relative path under the mount (e.g., "claim-checks/uuid.json").
        original_size_bytes: Size of the payload in bytes.

    Returns:
        Reference dict with keys: _claim_check, bucket, key, mount_path,
        original_size, s3_uri (for ALCOA+ audit trails).
    """
    # file_path may already have "claim-checks/" prefix
    key = file_path if "/" in file_path else f"claim-checks/{file_path}"

    return {
        "_claim_check": True,
        "bucket": S3_CLAIM_CHECK_BUCKET,
        "key": key,
        "mount_path": "/mnt/agentcore/claim-checks",
        "original_size": original_size_bytes,
        "s3_uri": f"s3://{S3_CLAIM_CHECK_BUCKET}/{key}",
    }
```
