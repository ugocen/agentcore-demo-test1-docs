# V5 Telemetry, AG-UI, Temporal Entegrasyon ve Claim-Check
## Kapsamli Teknik Analiz Raporu

**Tarih:** 2026-05-12
**Kapsam:** 4 Teknik Sorunun Derinlemesine Analizi
**Dokuman Kaynagi:** Mevcut 24 V5 dosyasi incelendi

---

## SORU 1: Telemetry Nereden Nasil Toplaniyor, Nerede Kaydediliyor? Backend FastAPI'ye Gidiyor mu?

### KISA CEVAP: HAYIR. Agent telemetry'si FastAPI'den GECMEZ. Iki bagimsiz paralel akis vardir.

### Tam Akis Diyagrami

```
================================================================================
KAYNAK KATMAN              TRANSPORT              HEDEF
================================================================================

1. AGENT TELEMETRY (3 Agent Paralel)
   ----------
   Agent 1 (Transcriber)    OTLP gRPC :4317   --->   ADOT Collector (EC2)
   Agent 2 (Drafter)        OTLP gRPC :4317   --->   ADOT Collector (EC2)
   Agent 3 (Reviewer)       OTLP gRPC :4317   --->   ADOT Collector (EC2)
                                |
                                v
                    +---------------------------+
                    |   ADOT Collector (EC2)    |
                    |   - awsxray exporter      |
                    |   - awsemf exporter       |
                    |   - awscloudwatchlogs     |
                    +------------+--------------+
                                 |
            +--------------------+--------------------+
            |                    |                    |
            v                    v                    v
    CloudWatch X-Ray    CloudWatch GenAI    CloudWatch Logs
    (Traces)            Dashboard           (/aws/agentcore/...)


2. FASTAPI BACKEND TELEMETRY (Bagimsiz)
   ----------
   FastAPI App              OTLP gRPC :4317   --->   ADOT Collector (EC2)
   (opentelemetry-instrumentation-fastapi)
                                |
                                v
                    +---------------------------+
                    |   ADOT Collector (EC2)    |
                    +------------+--------------+
                                 |
            +--------------------+--------------------+
            v                    v                    v
    CloudWatch X-Ray    CloudWatch GenAI    CloudWatch Logs
    (Traces)            Dashboard           (/aws/fastapi/...)


3. TEMPORAL SERVER TELEMETRY (Bagimsiz)
   ----------
   Temporal Server          OTLP gRPC :4317   --->   ADOT Collector (EC2)
   (Built-in OTel support)


4. TRACE BAGLAMI (Hepsi ayni trace_id'yi paylasir)
   ----------
   Browser (traceparent header uretir)
      |
      +---> FastAPI (traceparent'i parse eder, span acar)
      |       +---> Temporal Client (W3C propagate)
      |               +---> Temporal Activity
      |                       +---> boto3.invoke_agent_runtime()
      |                               +---> AgentCore (span acar)
      |                                       +---> Bedrock InvokeModel
      |
      +---> AgentCore SSE (AG-UI stream, traceparent ile)

================================================================================
```

### Agent Tarafinda Telemetry Nasil Calisiyor?

**Dosya:** `agents/common/otel_setup.py`
```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.trace_exporter import OTLPSpanExporter

def setup_tracer(agent_name: str):
    resource = Resource.create({
        SERVICE_NAME: f"agentcore-demo-test1-{agent_name}",
        SERVICE_VERSION: "1.0.0",
        DEPLOYMENT_ENVIRONMENT: "demo",
    })
    provider = TracerProvider(resource=resource)
    otlp_exporter = OTLPSpanExporter(endpoint="http://localhost:4318", insecure=True)
    provider.add_span_processor(BatchSpanProcessor(otlp_exporter))
    trace.set_tracer_provider(provider)
    return trace.get_tracer(f"agent-{agent_name}")
```

**Calisma Adimlari:**

| Adim | Ne Olur | Kodda Nerede |
|------|---------|-------------|
| 1 | `setup_tracer("transcriber")` cagrildi | `agent.py` en ustunde |
| 2 | `TracerProvider` olusturuldu | `otel_setup.py` icinde |
| 3 | `OTLPSpanExporter` `localhost:4318`'e baglandi | ADOT Collector daemon |
| 4 | `BatchSpanProcessor` span'leri toplu gonderiyor | 1sn timeout ile |
| 5 | `tracer.start_as_current_span("agent-run")` ile span acildi | `@app.entrypoint` icinde |
| 6 | `jnj.*` attributeler span'e eklendi | `jnj.trace_id`, `jnj.workflow_id`, etc. |
| 7 | `span.add_event("tool_call", {...})` ile tool call kaydedildi | Tool calistiktan sonra |
| 8 | `span.add_event("payload_emitted", {...})` ile payload kaydedildi | `build_complete_payload` sonrasi |
| 9 | Span otomatik kapatildi (context manager) | Fonksiyon return ettiginde |
| 10 | `BatchSpanProcessor` span'i ADOT Collector'a gonderdi | Async, otomatik |

**ONEMLI:** Agent'lar `localhost:4318`'e (ADOT Collector) dogrudan baglanir. Hicbir sey FastAPI'den gecmez.

### FastAPI Tarafinda Telemetry Nasil Calisiyor?

**Dosya:** `backend/app/telemetry/tracing.py`
```python
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor

def setup_tracing(app):
    FastAPIInstrumentor.instrument_app(app)
```

Bu, FastAPI'yi OTEL ile auto-instrument eder. Her HTTP request otomatik olarak bir span olusturur. FastAPI'nin span'leri de ayni ADOT Collector'a (`localhost:4317` veya `4318`) gider ama **agent'larinkinden bagimsizdir**.

### Trace Baglami Nasil Paylasiliyor? (W3C Trace Propagation)

| Katman | Islem |
|--------|-------|
| Browser | `traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01` header'i uretir |
| FastAPI | `tracing.py` > `get_trace_id()` bu header'i parse eder |
| FastAPI -> Temporal | W3C propagator ile trace_id'i gRPC metadata'ya ekler |
| Temporal -> Activity | `activity.info()` icinde trace_id bulunur |
| Activity -> AgentCore | `boto3.invoke_agent_runtime()` `trace_id` parametresi ile cagrilir |
| AgentCore -> Agent | `context.get("trace_id", ...)` ile agent'in span'ine yazilir |
| Agent -> ADOT | Ayni trace_id ile span ADOT'a gider |

**Sonuc:** Tum katmanlar ayni `trace_id`'yi (ornegin `4bf92f3577b34da6a3ce929d0e0e4736`) paylasir ama **farkli span'ler** uretir. CloudWatch X-Ray'de bu trace_id ile TUM katmanlarin span'lerini birlestirip end-to-end goruntuleyebilirsiniz.

### Hangi Telemetri Verileri Gidiyor?

**Agent Span'leri:**
- `agent-run` (ana span) — agent'in tam calisma suresi
- `tool_call` (event) — tool adi, sure, input, status
- `payload_emitted` (event) — payload ozeti, confidence, cost
- `jnj.trace_id`, `jnj.workflow_id`, `jnj.agent_run_id`, `jnj.step_id` (attribute)
- `jnj.step_type`, `jnj.tier`, `jnj.tenant_id` (attribute)

**FastAPI Span'leri:**
- `http_request` (auto) — endpoint, method, status code, duration
- `jnj.workflow_id` (attribute) — manual eklenir

### CloudWatch'a Nasil Kaydediliyor?

**ADOT Collector Config** (`infra/adot/collector-config.yaml`):
```yaml
receivers:
  otlp:
    protocols:
      grpc: { endpoint: 0.0.0.0:4317 }
      http: { endpoint: 0.0.0.0:4318 }
exporters:
  awsxray: {}        # -> X-Ray traces
  awsemf:            # -> CloudWatch Metrics/GenAI Dashboard
    namespace: "AISDLC/Agent"
    log_group_name: "/aws/agentcore/agentcore-demo-test1"
  awscloudwatchlogs: # -> CloudWatch Logs
    log_group_name: "/aws/agentcore/agentcore-demo-test1"
service:
  pipelines:
    traces:    [otlp -> batch -> awsxray]
    metrics:   [otlp -> batch -> awsemf]
    logs:      [otlp -> batch -> awscloudwatchlogs]
```

**CloudWatch GenAI Dashboard** otomatik olarak ADOT'un EMF ciktilarini tuketir. Manuel EMF log yazmaniza GEREK YOK.

---

## SORU 2: Agent'larimizin AG-UI ile Calismasi Icin Ek Islem Gerekecek mi?

### KISA CEVAP: EVET. 5 spesifik ek islem gerekiyor.

### Gereken Islemler

| # | Islem | Detay | Zorluk |
|---|-------|-------|--------|
| 1 | `bedrock-agentcore` paketini yuklemek | `uv pip install bedrock-agentcore` (.venv icinde) | Kolay |
| 2 | `BedrockAgentCoreApp()` instance olusturmak | `app = BedrockAgentCoreApp()` | Kolay |
| 3 | `@app.entrypoint` decorator ile entrypoint tanimlamak | `@app.entrypoint def invoke(payload, context)` | Kolay |
| 4 | 5 endpoint'i implement etmek | `/invoke`, `/metadata`, `/capabilities`, `/metrics`, `/health` | Orta |
| 5 | Framework'e ozel agent kodu yazmak | Strands (Agent 1, 2) veya LangGraph (Agent 3) | Orta |

### Ornek: Agent 1 (Transcriber) Icin Tam Kod

**Dosya:** `agents/agent_1_transcriber/agent.py`

```python
# ========== 1. IMPORTLAR ==========
from strands import Agent, tool
from bedrock_agentcore.runtime import BedrockAgentCoreApp
from common.payload_builder import build_complete_payload, validate_payload
from common.otel_setup import setup_tracer

# ========== 2. TOOL TANIMLAMA ==========
@tool
def transcribe_audio_tool(audio_uri: str) -> dict:
    """Transcribe audio via Amazon Transcribe."""
    import boto3
    client = boto3.client("transcribe")
    # ... transcription logic ...
    return {"transcript": transcript, "language": language}

# ========== 3. STRANDS AGENT ==========
strands_agent = Agent(
    model="anthropic.claude-3-5-sonnet-20241022-v2:0",
    system_prompt=("You are a transcription agent..."),
    tools=[transcribe_audio_tool],
)

# ========== 4. BEDROCKAGENTCOREAPP ==========
app = BedrockAgentCoreApp()

@app.entrypoint
def invoke(payload: dict, context: dict) -> dict:
    """POST /invoke — V5 endpoint."""
    # context'ten trace_id, workflow_id vb. al
    trace_id = context.get("trace_id", "unknown")
    workflow_id = context.get("workflow_id", "unknown")
    
    # OTel span ac
    with tracer.start_as_current_span("agent-run", attributes={...}):
        # Tool calistir
        result = transcribe_audio_tool(audio_uri)
        # V5 payload olustur
        payload_dict = build_complete_payload(...)
        validate_payload(payload_dict)
    
    return {"response": result, "payload": payload_dict}

# ========== 5. DIGER 4 ENDPOINT ==========
@app.get("/metadata")
def metadata():
    return {"agent_name": "transcriber", "framework": "Strands", "tier": "native"}

@app.get("/capabilities")
def capabilities():
    return {"tools": [{"name": "transcribe_audio_tool", ...}]}

@app.get("/metrics")
def metrics():
    from prometheus_client import generate_latest, CollectorRegistry
    return generate_latest(CollectorRegistry())

@app.get("/health")
def health():
    return {"status": "ok"}
```

### Farkli Framework'ler Icin Fark

| Framework | Entrypoint Yapisi | Fark |
|-----------|------------------|------|
| **Strands** (Agent 1, 2) | `@app.entrypoint` + `Agent()` | Tool'lar `@tool` decorator ile |
| **LangGraph** (Agent 3) | `@app.entrypoint` + `StateGraph` | Node'lar fonksiyon, `graph.invoke()` cagriliyor |
| **CrewAI** | `@app.entrypoint` + `Crew` | Task + Agent tanimlari |

**LangGraph Ornegi (Agent 3):**
```python
from langgraph.graph import StateGraph, END
from bedrock_agentcore.runtime import BedrockAgentCoreApp

# 4-node StateGraph
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

app = BedrockAgentCoreApp()

@app.entrypoint
def invoke(payload: dict, context: dict) -> dict:
    initial_state = {"messages": [...], "brd_text": ...}
    result = reviewer_graph.invoke(initial_state)  # LangGraph native cagri
    payload_dict = build_complete_payload(...)
    return {"response": result["review_report"], "payload": payload_dict}
```

### AG-UI Event'leri Ne Zaman ve Nasil Gonderiliyor?

`@app.entrypoint` decorator'u AGENT'TEN otomatik olarak AG-UI event'leri uretir:

| Event | Ne Zaman | Kim Uretir |
|-------|----------|------------|
| `TEXT_MESSAGE_START` | Agent cevap yazmaya basladiginda | `BedrockAgentCoreApp` otomatik |
| `TEXT_MESSAGE_CONTENT` | Her text chunk uretildiginde | LLM streaming + AG-UI |
| `TEXT_MESSAGE_END` | Cevap tamamlandiginda | `BedrockAgentCoreApp` otomatik |
| `TOOL_CALL_START` | Tool cagrilmaya basladiginda | `@tool` wrapper otomatik |
| `TOOL_CALL_END` | Tool tamamlandiginda | `@tool` wrapper otomatik |
| `STATE_DELTA` | State degistiginde | `BedrockAgentCoreApp` otomatik |
| `INTERRUPT` | HITL clarification gerektiginde | `request_clarification` tool |
| `CUSTOM` | Ozel event'ler | `app.emit_custom_event()` ile |

**NOT:** AG-UI event'leri `BedrockAgentCoreApp` tarafindan OTOMATIK uretilir. Sizin ayri bir kod yazmaniza gerek yok. Ama `INTERRUPT` gibi ozel event'ler icin `request_clarification` gibi tool'lari tanimlamaniz gerekir.

---

## SORU 3: Agent'larimizin Temporal ile Calismasi Icin Ek Islem Gerekecek mi?

### KISA CEVAP: Agent kodunda HAYIR, ama Temporal Worker'da EVET. Agent'lar dogrudan Temporal SDK kullanmaz.

### Tam Mimari

```
================================================================================
TEMPORAL ENTGREASYON AKISI
================================================================================

Browser (Frontend)
    |
    | POST /api/workflows (FastAPI REST)
    v
FastAPI Backend (port 8000)
    |
    | gRPC (port 7233)
    v
Temporal Server (port 7233)
    |
    | Task Queue: "brd-task-queue"
    v
Temporal Worker (ayri process/container)
    |
    | @activity.defn fonksiyonu calisir
    v
Activity: invoke_agent_core_activity()
    |
    | boto3.client("bedrock-agentcore-runtime")
    | .invoke_agent_runtime(
    |     session_id=workflow_id,
    |     qualifier="DEFAULT",
    |     input_text=...,
    |     session_state={
    |         "trace_id": trace_id,
    |         "workflow_id": workflow_id,
    |         "previous_output": previous_output,
    |         "round_num": round_num,
    |     }
    | )
    v
AWS AgentCore Runtime (port 8080)
    |
    | X-Amzn-Bedrock-AgentCore-Runtime-Session-Id: workflow_id
    v
Firecracker MicroVM (spun up)
    |
    v
Agent (@app.entrypoint calisir)
    |
    v
Sonuc doner -> Activity -> Workflow -> Signal/Diger Agent

================================================================================
```

### Agent Tarafinda NE YAPMAMIZ GEREKMIYOR

Agent'lar Temporal SDK'dan habersizdir. Agent'lar sadece:
1. `@app.entrypoint` ile gelen payload'i isler
2. `context.get("trace_id")` ve `context.get("workflow_id")` ile baglam alir
3. `build_complete_payload()` ile V5 payload olusturur
4. Sonuc doner

Agent KODUNDA hicbir `@activity.defn`, `workflow.execute_activity`, veya Temporal import YOK.

### Temporal Worker Tarafinda NE YAPMAMIZ GEREKIR

**Dosya:** `backend/app/temporal/activities.py`

```python
from temporalio import activity
import boto3

AGENT_RUNTIME_ARN_MAP = {
    "analyst":   "arn:aws:bedrock-agentcore:...:agent/transcriber",
    "architect": "arn:aws:bedrock-agentcore:...:agent/drafter",
    "reviewer":  "arn:aws:bedrock-agentcore:...:agent/reviewer",
}

@activity.defn
async def invoke_agent_core_activity(input_data: dict) -> dict:
    """
    Temporal Activity: AgentCore agent'ini cagirir.
    Bu fonksiyon Temporal Worker'da calisir.
    """
    workflow_id = input_data["workflow_id"]
    agent_role  = input_data["agent_role"]
    prompt      = input_data["prompt"]
    previous    = input_data.get("previous_output")
    round_num   = input_data.get("round_num", 1)

    # boto3 ile AgentCore Runtime'a cagri
    client = boto3.client("bedrock-agentcore-runtime")

    # Claim-check: buyuk payload varsa S3'e yaz
    session_state = {
        "trace_id": input_data.get("trace_id"),
        "workflow_id": workflow_id,
        "previous_output": previous,
        "round_num": round_num,
    }
    agent_request = _maybe_claim_check(session_state)

    # AgentCore Runtime'u cagir
    response = client.invoke_agent_runtime(
        agentRuntimeArn=AGENT_RUNTIME_ARN_MAP[agent_role],
        sessionId=workflow_id,          # session_id = workflow_id
        qualifier="DEFAULT",
        inputText=prompt,
        sessionState=agent_request,
    )

    # Agent sonucunu parse et
    result = json.loads(response["completion"])

    # Agent'in V5 payload'ini al
    agent_payload = result.get("payload", {})

    return {
        "content": result.get("response", ""),
        "needs_clarification": result.get("needs_clarification", False),
        "clarification_question": result.get("clarification_question", ""),
        "verdict": agent_payload.get("status", {}).get("status_code"),
        "feedback": agent_payload.get("status", {}).get("message", ""),
        "agent_payload": agent_payload,
    }
```

### Temporal Workflow Nasil Agent'lari Sirayla Cagiriyor?

**Dosya:** `backend/app/temporal/workflows.py`

```python
@workflow.defn(sandbox=False)
class BRDWorkflow:

    @workflow.run
    async def run(self, wf_input: WorkflowInput):

        for round_num in range(1, wf_input.max_rounds + 1):

            # === Agent 1: ANALYST ===
            analyst_result = await workflow.execute_activity(
                invoke_agent_core_activity,
                {"workflow_id": wf_id, "agent_role": "analyst", ...},
                start_to_close_timeout=timedelta(minutes=10),
            )

            # Eger clarification gerekirse bekle
            if analyst_result.get("needs_clarification"):
                clarification = await self._wait_for_clarification(...)

            # === Agent 2: ARCHITECT ===
            architect_result = await workflow.execute_activity(
                invoke_agent_core_activity,
                {"workflow_id": wf_id, "agent_role": "architect",
                 "previous_output": analyst_result["content"], ...},
                start_to_close_timeout=timedelta(minutes=10),
            )

            # === Agent 3: REVIEWER ===
            reviewer_result = await workflow.execute_activity(
                invoke_agent_core_activity,
                {"workflow_id": wf_id, "agent_role": "reviewer",
                 "previous_output": architect_result["content"], ...},
                start_to_close_timeout=timedelta(minutes=10),
            )

            if reviewer_result["verdict"] == "APPROVE":
                break
            # Yoksa loop devam eder (self-correction)
```

### A2A (Agent-to-Agent) Nasil Calisiyor?

A2A, agent'larin birbirleriyle KONUSMAMASI anlamina gelir. Agent'lar sadece:
1. Kendi `@app.entrypoint`'ine gelen payload'i isler
2. Sonuc doner
3. Temporal workflow bu sonucu alir, diger agent'a parametre olarak gecirir

**A2A Akisi:**
```
Agent 1 calisir (transcribe)
    -> Sonuc: {"transcript": "...", "language": "en-US"}
    -> Temporal workflow bunu alir
    -> Agent 2'nin inputuna ekler: {"previous_output": transcript}
Agent 2 calisir (draft)
    -> Sonuc: {"draft": "# BRD\n..."}
    -> Temporal workflow bunu alir
    -> Agent 3'un inputuna ekler: {"previous_output": draft}
Agent 3 calisir (review)
    -> Sonuc: {"review": "APPROVED" veya "CHANGES_REQUESTED"}
    -> Temporal workflow karar verir
```

### HITL (Human-in-the-Loop) Nasil Calisiyor?

HITL de Temporal signals uzerinden calisir:

```python
# Workflow'ta sinyal bekleme
await workflow.wait_condition(
    lambda: self._clarification_response is not None,
    timeout=timedelta(minutes=15),
)

# Frontend'den sinyal gonderme (FastAPI uzerinden)
POST /api/workflows/{workflow_id}/signal
Body: {"signal_name": "clarification_response", "data": {"response_text": "Cevap..."}}
```

### Self-Correction (V5 Ozelligi)

| Durum | Akis |
|-------|------|
| Reviewer "APPROVE" der | Workflow tamamlanir |
| Reviewer "CHANGES_REQUESTED" der | `self_correction_count += 1` |
| `self_correction_count < 2` | Loop devam eder, Agent 2 tekrar calisir |
| `self_correction_count >= 2` | HITL escalation — clarification sorusu sorulur |

---

## SORU 4: Claim-Check Sistemi V5'te Nasil Calisiyor? AgentCore Persistent Filesystems Nerede Devreye Giriyor?

### KISA CEVAP: Claim-check pattern'i KORUNUR ama agent'lar artik boto3 S3 API yerine AgentCore Persistent Filesystems (S3 Files mount) uzerinden POSIX file operations kullanir. Temporal Activity'ler (EC2) hala boto3 ile S3'e yazip okur.

### Mimari: Iki Taraf, Iki API

```
================================================================================
CLAIM-CHECK SISTEMI — DUAL API MIMARISI
================================================================================

EC2 (Temporal Activity)                         microVM (Agent)
| boto3.client("s3").put_object()               | open("/mnt/agentcore/...", "w")
| boto3.client("s3").get_object()               | open("/mnt/agentcore/...", "r")
|                                               |
|  _maybe_claim_check()                         |  write_claim_check()
|  _resolve_claim_check()                       |  read_claim_check()
|       |                                       |       |
+-------+---------------------------------------+-------+
        |                                       |
        v                                       v
   +----+------------------------------------+----+
   |         S3  (claim-check bucket)             |
   |   claim-checks/uuid-123.json                 |
   |   claim-checks/uuid-456.json                 |
   +----------------------------------------------+
        |
        v
   S3 Files Access Point  <-- NFSv4.2 over TLS (port 2049)
        |
        v
   AgentCore Runtime mounts this as /mnt/agentcore/claim-checks

================================================================================
```

| Taraf | Nerede Calisir | API | Hangi Fonksiyon |
|-------|---------------|-----|----------------|
| **Temporal Activity** | EC2 | boto3 S3 | `_maybe_claim_check()`, `_resolve_claim_check()` |
| **Agent** | microVM | POSIX file ops | `write_claim_check()`, `read_claim_check()` |
| **Evidence Pack** | EC2 | boto3 S3 | `_resolve_claim_check()` |

### Neden Iki Farkli API?

| Taraf | Neden Bu API? |
|-------|--------------|
| **EC2 (Temporal)** | EC2'de S3 Files mount yok. boto3 en verimli yontem. |
| **microVM (Agent)** | AgentCore Runtime S3 Files'i otomatik mount eder. POSIX (`open`, `read`, `write`) daha basit ve dogrudan. |
| **Referanslar** | Her iki taraf da ayni S3 bucket'i kullanir. Claim-check referans dict (`_claim_check`, `bucket`, `key`) ortaktir. |

### EC2 Tarafi: Temporal Activity (boto3) — Degismez

```python
# backend/app/temporal/activities.py
# Bu kod EC2'de calisir, boto3 ile S3'e yazar.

import boto3

S3_CLAIM_CHECK_BUCKET = "agentcore-demo-test1-claimcheck-123456789"

def _maybe_claim_check(payload: dict) -> dict:
    """EC2: Payload > 256 KiB ise S3'e yaz, claim-check referansi don."""
    payload_bytes = json.dumps(payload).encode("utf-8")
    if len(payload_bytes) <= 256 * 1024:
        return payload  # Inline

    s3 = boto3.client("s3", region_name="us-east-1")
    key = f"claim-checks/{uuid.uuid4()}.json"
    s3.put_object(Bucket=S3_CLAIM_CHECK_BUCKET, Key=key, Body=payload_bytes)

    return {
        "_claim_check": True,
        "bucket": S3_CLAIM_CHECK_BUCKET,
        "key": key,
        "original_size": len(payload_bytes),
        "mount_path": "/mnt/agentcore/claim-checks",  # agent icin
    }
```

### Agent Tarafi: microVM (POSIX File Operations) — YENI

```python
# agents/common/claim_check_io.py
# Bu kod agent'in microVM'inde calisir. S3 Files mount edilmistir.

import json
import os
from pathlib import Path

# /mnt/agentcore/claim-checks = S3 Files mount noktasi
MOUNT = Path(os.environ.get("CLAIM_CHECK_MOUNT", "/mnt/agentcore/claim-checks"))

def write_claim_check(payload: dict) -> str:
    """microVM: S3 Files mount uzerine JSON yazar. POSIX file ops."""
    file_name = f"claim-checks/{uuid.uuid4()}.json"
    file_path = MOUNT / file_name
    file_path.parent.mkdir(parents=True, exist_ok=True)

    with open(file_path, "w", encoding="utf-8") as f:
        json.dump(payload, f)

    return file_name  # relative path (Temporal reference'ta kullanilir)


def read_claim_check(reference: dict) -> dict:
    """microVM: S3 Files mount uzerinden claim-check okur. POSIX file ops."""
    key = reference.get("key", "")
    file_path = MOUNT / key if not key.startswith("/") else Path(key)

    with open(file_path, "r", encoding="utf-8") as f:
        return json.load(f)
```

### AgentCore Persistent Filesystems — S3 Files Nasil Yapilandirilir?

**1. S3 Files File System + Access Point Olustur:**
```bash
# Claim-check bucket'in S3 Files access point'i
aws s3api create-access-point \
  --bucket agentcore-demo-test1-claimcheck-123456789 \
  --name agentcore-demo-test1-claim-checks-ap \
  --vpc-configuration VpcId=vpc-xxxxxxxxxxxxx
```

**2. Agent Runtime Filesystem Konfigurasyonu:**
```json
{
  "filesystemConfigurations": [
    {
      "s3Files": {
        "fileSystemArn": "arn:aws:s3-files:us-east-1:123456789012:file-system/fs-xxx",
        "accessPointArn": "arn:aws:s3-files:us-east-1:123456789012:access-point/fsap-xxx",
        "mountPath": "/mnt/agentcore/claim-checks",
        "readOnly": false
      }
    }
  ]
}
```

**3. Deploy:**
```bash
agentcore configure -e agent.py --protocol AGUI \
  --filesystem-configurations file://fs-config.json
agentcore deploy
```

**4. IAM Permissions (S3 Files icin):**
```json
{
  "Sid": "S3FilesAgentAccess",
  "Effect": "Allow",
  "Action": [
    "s3files:ClientMount",
    "s3files:ClientWrite",
    "s3files:GetAccessPoint"
  ],
  "Resource": "arn:aws:s3-files:*:*:file-system/*",
  "Condition": {
    "StringEquals": {
      "s3files:AccessPointArn": "arn:aws:s3-files:us-east-1:123456789012:access-point/fsap-xxx"
    }
  }
}
```

### Temporal'a Ne Bilgisi Gidiyor?

**Temporal Event History'ye kaydedilen bilgiler:**

| Bilgi | Nerede | Boyut |
|-------|--------|-------|
| `workflow_id` | `WorkflowExecutionStarted` event'inde | Kucuk |
| `agent_role` | Activity input'ta | Kucuk |
| `session_id` (= workflow_id) | Activity input'ta | Kucuk |
| `prompt` | Activity input'ta | Kucuk-Medium |
| `previous_output` | Activity input'ta | **Buyuk olabilir** |
| `trace_id` | Activity input'ta | Kucuk |
| `round_num` | Activity input'ta | Kucuk |
| `_claim_check` referansi | Activity input/output'ta | Kucuk (sadece S3 URI) |

### Hangi Durumlarda Claim-Check Devreye Girer?

| Senaryo | previous_output Boyutu | Claim-Check? |
|---------|----------------------|-------------|
| Agent 1 -> Agent 2 | Transcript (50 KB) | HAYIR |
| Agent 2 -> Agent 3 | BRD Markdown (200 KB) | HAYIR |
| Agent 2 -> Agent 3 | Uzun BRD (500 KB) | **EVET** |
| HITL context | Kisa clarification (5 KB) | HAYIR |
| Uzun audio + metadata | > 256 KiB | **EVET** |

### S3 Files — Onemli Teknik Detaylar

| Ozellik | Deger |
|---------|-------|
| **Protokol** | NFSv4.2 over TLS (port 2049) |
| **Mount path** | `/mnt/agentcore/claim-checks` (konfigurEdilebilir) |
| **Max file size** | 48 TiB |
| **Consistency** | Close-to-open (close() sonrasi S3'e sync) |
| **Hard links** | Desteklenmez |
| **S3 Glacier** | Desteklenmez |
| **Encryption** | TLS uzerinden (transit), SSE-S3 (rest) |
| **Bidirectional sync** | S3 bucket <-> mount arasi otomatik |
| **VPC gereksinimi** | Evet, subnet'ler mount target AZ ile overlap etmeli |

### Claim-Check Bucket

| Ozellik | Deger |
|---------|-------|
| Bucket ismi | `agentcore-demo-test1-claimcheck-{account-id}` |
| Key prefix | `claim-checks/{uuid}.json` |
| Content-Type | `application/json` |
| Lifecycle | 7 gun otomatik silme (demo) |
| Encryption | SSE-S3 |
| Access | S3 Files Access Point uzerinden |

---

## OZET TABLO: 4 Sorunun Karsilastirmali Cevaplari

| Soru | Kisa Cevap | Kim Ekler | Ne Ekler | Nereye Gider |
|------|-----------|-----------|----------|-------------|
| **1. Telemetry** | FastAPI'ye GECMEZ | Her katman kendi ekler | OTel span + `jnj.*` attr | ADOT Collector -> CloudWatch |
| **2. AG-UI** | EVET, ek islem gerekir | Developer (siz) | `@app.entrypoint` + 5 endpoint | AgentCore -> Browser (SSE) |
| **3. Temporal** | Agent'da HAYIR, Worker'da EVET | Developer (siz) | `@activity.defn` + `boto3.invoke_agent_runtime()` | Temporal -> AgentCore |
| **4. Claim-Check** | 256 KiB threshold | Otomatik | `_maybe_claim_check()` + S3 | S3 -> Temporal (referans) |

---

*Rapor Sonu*
