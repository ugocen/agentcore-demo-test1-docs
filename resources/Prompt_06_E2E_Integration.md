# Prompt 06: End-to-End Integration (Phase 6)

**MODE: AUTOMATIC — AI writes test scripts and runs them sequentially.**

## Reference Documents (READ FIRST)

| Priority | Document | Why |
|----------|----------|-----|
| **PRIMARY** | `resources/Deliverable_0_PROJECT_CONTEXT.md` (PCD) | All frozen decisions D1–D14; demo simplifications matrix §12 |
| **PRIMARY** | `resources/PAYLOAD_SCHEMA.md` | 8-block schema for response validation |
| **PRIMARY** | `resources/PROMPT_TESTING_SECTIONS.md` | Test directory layout per phase |
| **REFERENCE** | All earlier Prompts (00–05) — this phase validates their work |

---

## CONTEXT

You are executing **Phase 6** of an 8-phase build. Phases 0–5 are complete:

- **Phase 0**: AWS infrastructure (VPC, S3, RDS, EC2, AgentCore CLI bootstrap).
- **Phase 1**: 3 agents deployed to AgentCore via S3 ZIP; each exposes 5 endpoints; smoke-tested.
- **Phase 2**: Agents emit OTel spans with `demo.*` attributes; ADOT Collector configured.
- **Phase 3**: FastAPI backend running; `evidence_packs` table created (canonical D10 schema); 6-service docker-compose stack up.
- **Phase 4**: `BrdFromAudioWorkflow` runs end-to-end; HITL signals work; Workflow Stream (Pattern A or B per D6) bridges to FastAPI SSE.
- **Phase 5**: Next.js frontend; mic recording + file upload; `/workspace/[wfId]` page renders chat + canvas; approval/clarification UI works.

This phase **validates the integrated system** with 11 sequential tests (T1–T11). After T11 passes, Phase 7 (Metric Dashboard) can begin.

---

## WORKING DIRECTORY (canonical per PCD D5)

Tests run from the **backend repo** because they import the same modules the workflow uses:

```bash
# Mac development environment (primary):
export PROJECT_ROOT="/Users/ugurgocen/projects/agentcore-demo-test1"
cd "$PROJECT_ROOT/agentcore-demo-test1-backend"

# EC2 instance environment (alternative — when SSH'd into the demo host):
# export PROJECT_ROOT="/home/ec2-user/agentcore-demo-test1"
# cd "$PROJECT_ROOT/agentcore-demo-test1-backend"
```

All paths below are relative to `$PROJECT_ROOT/agentcore-demo-test1-backend`.

---

## CRITICAL RULES

- **Tests validate, they don't create.** All AWS resources, code, and configuration exist from Phases 0–5. Tests assert behavior; they do not provision.
- **No `--protocol AGUI`** anywhere (the canonical deployment uses `--protocol HTTP` per PCD §15.1).
- **8-block payload assertions** check canonical block names per PAYLOAD_SCHEMA §3: `status / resources / timing / financial / artifacts / quality / tool_calls / risk`. No alternate names.
- **Workflow outcomes** are from the canonical BRDState enum (PCD App D): `PENDING`, `IN_PROGRESS`, `AWAITING_HUMAN`, `APPROVED`, `REJECTED`, `FAILED`, `MAX_ITERATIONS`.
- **CloudWatch verification** queries for `demo.*` attributes (PCD D1). No `jnj.*` references.
- **macOS / Linux date compatibility**: scripts must work on both.

---

## STEP 0: Test fixtures

Create the test fixture generator (deterministic audio sample) once:

**File:** `tests/e2e/seed_test_audio.py`

```python
"""Generate a deterministic 3-second WAV for E2E tests.

The file is uploaded once at the start of the test run and re-used by every
test that needs an audio input. The content (a 440 Hz sine wave) is the same
on every run so artifact hashes can be compared across runs if needed.
"""
from __future__ import annotations
import math
import struct
import wave
from pathlib import Path

OUTPUT = Path(__file__).parent / "fixtures" / "smoke-3sec.wav"
OUTPUT.parent.mkdir(exist_ok=True)


def main() -> None:
    sample_rate = 16000  # Transcribe-friendly
    duration_seconds = 3.0
    frequency = 440.0
    amplitude = 16000

    n_frames = int(sample_rate * duration_seconds)
    with wave.open(str(OUTPUT), "wb") as wav:
        wav.setnchannels(1)
        wav.setsampwidth(2)
        wav.setframerate(sample_rate)
        for i in range(n_frames):
            sample = int(amplitude * math.sin(2 * math.pi * frequency * i / sample_rate))
            wav.writeframes(struct.pack("<h", sample))
    print(f"Wrote {OUTPUT} ({OUTPUT.stat().st_size} bytes)")


if __name__ == "__main__":
    main()
```

Run once before T1:
```bash
mkdir -p tests/e2e/fixtures
uv run python tests/e2e/seed_test_audio.py
ls -lh tests/e2e/fixtures/smoke-3sec.wav
```

**CHECKPOINT 0:** `tests/e2e/fixtures/smoke-3sec.wav` exists, ~96 KB.

---

## STEP 1: Orchestrator script

**File:** `scripts/e2e_test.sh`

```bash
#!/usr/bin/env bash
# End-to-end test orchestrator for the agentcore-demo-test1 system.
# Runs T1–T11 sequentially; stops on first failure; prints summary.
set -uo pipefail

PROJECT_ROOT="${PROJECT_ROOT:-/Users/ugurgocen/projects/agentcore-demo-test1}"
BACKEND_ROOT="${PROJECT_ROOT}/agentcore-demo-test1-backend"
FRONTEND_URL="${FRONTEND_URL:-http://localhost:3000}"
BACKEND_URL="${BACKEND_URL:-http://localhost:8000}"
TEMPORAL_UI_URL="${TEMPORAL_UI_URL:-http://localhost:8081}"
LOG_FILE="${BACKEND_ROOT}/tests/e2e/e2e-$(date +%s).log"

mkdir -p "$(dirname "$LOG_FILE")"

GREEN="\033[0;32m"; RED="\033[0;31m"; YEL="\033[0;33m"; NC="\033[0m"
PASSED=0
FAILED=0

run() {
    local name="$1"; shift
    echo ""
    echo -e "${YEL}===> $name${NC}"
    if "$@" >>"$LOG_FILE" 2>&1; then
        echo -e "${GREEN}PASS${NC}: $name"
        PASSED=$((PASSED+1))
    else
        echo -e "${RED}FAIL${NC}: $name (see $LOG_FILE)"
        FAILED=$((FAILED+1))
    fi
}

cd "$BACKEND_ROOT" || { echo "Cannot cd to $BACKEND_ROOT"; exit 2; }

# Fixture
if [ ! -f tests/e2e/fixtures/smoke-3sec.wav ]; then
    uv run python tests/e2e/seed_test_audio.py
fi

TESTS=(
    "T01_container_health"
    "T02_service_endpoints"
    "T03_happy_path_upload"
    "T04_happy_path_mic"
    "T05_hitl_clarification"
    "T06_self_correction_cap"
    "T07_rejection"
    "T08_revision_cycle"
    "T09_cloudwatch_demo_attributes"
    "T10_evidence_pack_validation"
    "T11_clean_restart"
)
for t in "${TESTS[@]}"; do
    run "$t" uv run pytest "tests/e2e/test_${t}.py" -v --tb=short
done

echo ""
echo "===== E2E Summary ====="
echo "  Passed: $PASSED"
echo "  Failed: $FAILED"
echo "  Log:    $LOG_FILE"
if [ "$FAILED" -gt 0 ]; then exit 1; fi
```

Make executable: `chmod +x scripts/e2e_test.sh`.

---

## STEP 2: T1 — Container health

All 6 (or 7 with Pattern B) docker-compose services are Up.

**File:** `tests/e2e/test_T01_container_health.py`

```python
import subprocess
import json


REQUIRED = {"temporal-server", "temporal-ui", "temporal-worker",
            "fastapi", "nextjs-frontend", "adot-collector"}


def _running_services() -> set[str]:
    out = subprocess.check_output(
        ["docker", "compose", "ps", "--format", "json"]
    ).decode()
    return {json.loads(line)["Service"] for line in out.splitlines() if line.strip()}


def test_required_services_running():
    running = _running_services()
    missing = REQUIRED - running
    assert not missing, f"Missing services: {missing}"
```

---

## STEP 3: T2 — Service endpoints

```python
# tests/e2e/test_T02_service_endpoints.py
import os
import requests

BACKEND = os.environ.get("BACKEND_URL", "http://localhost:8000")
FRONTEND = os.environ.get("FRONTEND_URL", "http://localhost:3000")
TEMPORAL_UI = os.environ.get("TEMPORAL_UI_URL", "http://localhost:8081")


def test_fastapi_health():
    r = requests.get(f"{BACKEND}/health", timeout=5)
    assert r.status_code == 200
    assert r.json().get("status") in {"ok", "healthy"}


def test_fastapi_db_health():
    r = requests.get(f"{BACKEND}/health/db", timeout=5)
    assert r.status_code == 200


def test_frontend_landing():
    r = requests.get(FRONTEND, timeout=5)
    assert r.status_code == 200


def test_temporal_ui_reachable():
    r = requests.get(TEMPORAL_UI, timeout=5)
    assert r.status_code == 200
```

---

## STEP 4: T3 — Happy path via file upload

Upload the seeded WAV, start a workflow, approve at the end, expect APPROVED outcome.

```python
# tests/e2e/test_T03_happy_path_upload.py
import os
import time
import requests
from pathlib import Path

BACKEND = os.environ.get("BACKEND_URL", "http://localhost:8000")
FIXTURE = Path(__file__).parent / "fixtures" / "smoke-3sec.wav"


def test_happy_path_via_upload():
    # 1) Upload audio
    with FIXTURE.open("rb") as f:
        up = requests.post(f"{BACKEND}/api/v1/uploads/audio",
                           files={"file": (FIXTURE.name, f, "audio/wav")},
                           timeout=30)
    assert up.status_code == 200, up.text
    s3_uri = up.json()["s3_uri"]
    assert s3_uri.startswith("s3://")

    # 2) Start workflow
    start = requests.post(f"{BACKEND}/api/v1/workflows",
                          json={"audio_s3_uri": s3_uri, "persona_id": "ba-sap-mm"},
                          timeout=10)
    assert start.status_code == 202, start.text
    wf_id = start.json()["workflow_id"]

    # 3) Poll until AWAITING_HUMAN (approval gate) or terminal
    deadline = time.time() + 180
    state = None
    while time.time() < deadline:
        st = requests.get(f"{BACKEND}/api/v1/workflows/{wf_id}", timeout=5).json()
        state = st["outcome"]
        if state in {"AWAITING_HUMAN", "APPROVED", "REJECTED", "FAILED", "MAX_ITERATIONS"}:
            break
        time.sleep(2)
    assert state == "AWAITING_HUMAN", f"unexpected outcome {state}"

    # 4) Approve
    requests.post(f"{BACKEND}/api/v1/workflows/{wf_id}/signal/approval",
                  json={"decision": "approve"}, timeout=5)

    # 5) Poll until APPROVED
    deadline = time.time() + 30
    while time.time() < deadline:
        st = requests.get(f"{BACKEND}/api/v1/workflows/{wf_id}", timeout=5).json()
        if st["outcome"] == "APPROVED":
            # Save id for T10 (Evidence Pack validation)
            Path("tests/e2e/.last_approved_wf").write_text(wf_id)
            return
        time.sleep(1)
    raise AssertionError(f"Workflow did not reach APPROVED, last: {st}")
```

---

## STEP 5: T4 — Happy path via simulated mic recording

```python
# tests/e2e/test_T04_happy_path_mic.py
import os
import subprocess
import requests
from pathlib import Path

BACKEND = os.environ.get("BACKEND_URL", "http://localhost:8000")
SOURCE = Path(__file__).parent / "fixtures" / "smoke-3sec.wav"
WEBM = Path(__file__).parent / "fixtures" / "smoke-3sec.webm"


def _ensure_webm():
    if WEBM.exists():
        return
    subprocess.check_call([
        "ffmpeg", "-y", "-i", str(SOURCE), "-c:a", "libopus", "-b:a", "32k", str(WEBM),
    ])


def test_happy_path_via_mic_simulation():
    _ensure_webm()
    with WEBM.open("rb") as f:
        up = requests.post(f"{BACKEND}/api/v1/uploads/audio",
                           files={"file": (WEBM.name, f, "audio/webm")},
                           timeout=30)
    assert up.status_code == 200, up.text
    assert up.json()["content_type"] in {"audio/webm", "audio/ogg"}
    # Start (same flow as T3); we just verify the upload path accepts webm here.
    start = requests.post(f"{BACKEND}/api/v1/workflows",
                          json={"audio_s3_uri": up.json()["s3_uri"], "persona_id": "ba-sap-mm"},
                          timeout=10)
    assert start.status_code == 202
```

---

## STEP 6: T5 — HITL clarification round

```python
# tests/e2e/test_T05_hitl_clarification.py
import os
import time
import requests
from pathlib import Path

BACKEND = os.environ.get("BACKEND_URL", "http://localhost:8000")
FIXTURE = Path(__file__).parent / "fixtures" / "smoke-3sec.wav"


def test_hitl_clarification_round():
    with FIXTURE.open("rb") as f:
        up = requests.post(f"{BACKEND}/api/v1/uploads/audio",
                           files={"file": (FIXTURE.name, f, "audio/wav")}, timeout=30)
    s3_uri = up.json()["s3_uri"]
    start = requests.post(f"{BACKEND}/api/v1/workflows",
                          json={"audio_s3_uri": s3_uri, "persona_id": "ba-sap-mm"})
    wf_id = start.json()["workflow_id"]

    # Wait for at least one hitl exchange (drafter clarification) or terminal
    deadline = time.time() + 180
    saw_clarification = False
    st = {}
    while time.time() < deadline:
        st = requests.get(f"{BACKEND}/api/v1/workflows/{wf_id}", timeout=5).json()
        if st["hitl_exchange_count"] >= 1:
            saw_clarification = True
            break
        if st["outcome"] in {"APPROVED", "REJECTED", "FAILED", "MAX_ITERATIONS"}:
            return  # Non-deterministic — for the seed audio, drafter may not need clarification
        time.sleep(2)

    if saw_clarification:
        requests.post(f"{BACKEND}/api/v1/workflows/{wf_id}/signal/clarification",
                      json={"rationale": "The approver is the Procurement Manager."})
        deadline = time.time() + 120
        while time.time() < deadline:
            st = requests.get(f"{BACKEND}/api/v1/workflows/{wf_id}", timeout=5).json()
            if st["outcome"] in {"AWAITING_HUMAN", "APPROVED", "REJECTED", "FAILED", "MAX_ITERATIONS"}:
                return
            time.sleep(2)
        raise AssertionError("Workflow did not advance after clarification")
```

---

## STEP 7: T6 — Self-correction cap escalation (PCD D8)

```python
# tests/e2e/test_T06_self_correction_cap.py
import os
import time
import requests
import pytest
from pathlib import Path

BACKEND = os.environ.get("BACKEND_URL", "http://localhost:8000")
FIXTURE = Path(__file__).parent / "fixtures" / "smoke-3sec.wav"


@pytest.mark.skipif(
    os.environ.get("DEMO_FORCE_DRAFTER_CLARIFICATION") != "true",
    reason="Set DEMO_FORCE_DRAFTER_CLARIFICATION=true on temporal-worker to run.",
)
def test_self_correction_cap_escalation():
    """After 3 failed Drafter attempts, the workflow must terminate MAX_ITERATIONS."""
    with FIXTURE.open("rb") as f:
        up = requests.post(f"{BACKEND}/api/v1/uploads/audio",
                           files={"file": (FIXTURE.name, f, "audio/wav")}, timeout=30)
    s3_uri = up.json()["s3_uri"]
    start = requests.post(f"{BACKEND}/api/v1/workflows",
                          json={"audio_s3_uri": s3_uri, "persona_id": "ba-sap-mm"})
    wf_id = start.json()["workflow_id"]

    # Respond to 3 clarifications with text that keeps confidence below threshold
    for _ in range(3):
        deadline = time.time() + 120
        while time.time() < deadline:
            st = requests.get(f"{BACKEND}/api/v1/workflows/{wf_id}", timeout=5).json()
            if st["outcome"] == "MAX_ITERATIONS":
                return
            if st["hitl_exchange_count"] >= 1:
                requests.post(f"{BACKEND}/api/v1/workflows/{wf_id}/signal/clarification",
                              json={"rationale": "unclear"})
                break
            time.sleep(2)

    deadline = time.time() + 60
    while time.time() < deadline:
        st = requests.get(f"{BACKEND}/api/v1/workflows/{wf_id}", timeout=5).json()
        if st["outcome"] == "MAX_ITERATIONS":
            return
        time.sleep(2)
    raise AssertionError("Workflow did not reach MAX_ITERATIONS")
```

---

## STEP 8: T7 — Rejection at approval

```python
# tests/e2e/test_T07_rejection.py
import os, time, requests
from pathlib import Path

BACKEND = os.environ.get("BACKEND_URL", "http://localhost:8000")
FIXTURE = Path(__file__).parent / "fixtures" / "smoke-3sec.wav"


def test_rejection_at_approval_step():
    with FIXTURE.open("rb") as f:
        up = requests.post(f"{BACKEND}/api/v1/uploads/audio",
                           files={"file": (FIXTURE.name, f, "audio/wav")}, timeout=30)
    s3_uri = up.json()["s3_uri"]
    start = requests.post(f"{BACKEND}/api/v1/workflows",
                          json={"audio_s3_uri": s3_uri, "persona_id": "ba-sap-mm"})
    wf_id = start.json()["workflow_id"]

    # Wait for AWAITING_HUMAN
    deadline = time.time() + 240
    st = {}
    while time.time() < deadline:
        st = requests.get(f"{BACKEND}/api/v1/workflows/{wf_id}", timeout=5).json()
        if st["outcome"] == "AWAITING_HUMAN":
            break
        if st["outcome"] in {"APPROVED", "REJECTED", "FAILED", "MAX_ITERATIONS"}:
            return  # earlier terminal
        time.sleep(2)

    requests.post(f"{BACKEND}/api/v1/workflows/{wf_id}/signal/approval",
                  json={"decision": "reject"})

    deadline = time.time() + 30
    while time.time() < deadline:
        st = requests.get(f"{BACKEND}/api/v1/workflows/{wf_id}", timeout=5).json()
        if st["outcome"] == "REJECTED":
            return
        time.sleep(1)
    raise AssertionError(f"Workflow did not reach REJECTED, last: {st}")
```

---

## STEP 9: T8 — Revise → REJECTED (demo simplification)

```python
# tests/e2e/test_T08_revision_cycle.py
import os, time, requests
from pathlib import Path

BACKEND = os.environ.get("BACKEND_URL", "http://localhost:8000")
FIXTURE = Path(__file__).parent / "fixtures" / "smoke-3sec.wav"


def test_revise_terminates_as_rejected():
    with FIXTURE.open("rb") as f:
        up = requests.post(f"{BACKEND}/api/v1/uploads/audio",
                           files={"file": (FIXTURE.name, f, "audio/wav")}, timeout=30)
    s3_uri = up.json()["s3_uri"]
    start = requests.post(f"{BACKEND}/api/v1/workflows",
                          json={"audio_s3_uri": s3_uri, "persona_id": "ba-sap-mm"})
    wf_id = start.json()["workflow_id"]

    deadline = time.time() + 240
    st = {}
    while time.time() < deadline:
        st = requests.get(f"{BACKEND}/api/v1/workflows/{wf_id}", timeout=5).json()
        if st["outcome"] == "AWAITING_HUMAN":
            break
        time.sleep(2)

    requests.post(f"{BACKEND}/api/v1/workflows/{wf_id}/signal/approval",
                  json={"decision": "revise", "rationale": "Section 3 unclear"})

    deadline = time.time() + 30
    while time.time() < deadline:
        st = requests.get(f"{BACKEND}/api/v1/workflows/{wf_id}", timeout=5).json()
        if st["outcome"] == "REJECTED":
            return
        time.sleep(1)
    raise AssertionError(f"Workflow did not terminate after revise; outcome={st.get('outcome')}")
```

---

## STEP 10: T9 — CloudWatch `demo.*` attribute verification

```python
# tests/e2e/test_T09_cloudwatch_demo_attributes.py
import os
import time
import boto3
import pytest

REGION = os.environ.get("AWS_REGION", "us-east-1")
client = boto3.client("logs", region_name=REGION)


SHARED_LOG_GROUP = "/agentcore-demo-test1/agent-logs"


@pytest.mark.parametrize("agent_id", [
    "agent_1_transcriber",
    "agent_2_drafter",
    "agent_3_reviewer",
])
def test_demo_attributes_in_logs(agent_id):
    """Verify each agent's records appear in the shared log group, filtered by demo.agent_id."""
    end_time = int(time.time() * 1000)
    start_time = end_time - 30 * 60 * 1000  # last 30 minutes
    resp = client.start_query(
        logGroupName=SHARED_LOG_GROUP,
        startTime=start_time, endTime=end_time,
        queryString=(
            f'fields @message, demo.agent_id, demo.workflow_id '
            f'| filter demo.agent_id = "{agent_id}" '
            f'| limit 5'
        ),
    )
    query_id = resp["queryId"]
    deadline = time.time() + 30
    result = {}
    while time.time() < deadline:
        result = client.get_query_results(queryId=query_id)
        if result["status"] in {"Complete", "Failed", "Cancelled"}:
            break
        time.sleep(1)
    assert result["status"] == "Complete", result
    assert len(result["results"]) > 0, f"No demo.agent_id={agent_id} records in {SHARED_LOG_GROUP}"


def test_metric_namespace_exists():
    cw = boto3.client("cloudwatch", region_name=REGION)
    metrics = cw.list_metrics(Namespace="DemoSDLC/Agent")
    assert len(metrics.get("Metrics", [])) > 0, "DemoSDLC/Agent namespace empty"
```

---

## STEP 11: T10 — Evidence Pack JSON validation

```python
# tests/e2e/test_T10_evidence_pack_validation.py
import os
import requests
import pytest
from pathlib import Path

BACKEND = os.environ.get("BACKEND_URL", "http://localhost:8000")
REQUIRED_BLOCKS = {"status", "resources", "timing", "financial",
                   "artifacts", "quality", "tool_calls", "risk"}


def test_evidence_pack_has_full_8block_per_agent():
    """Read the workflow_id saved by T3; assert every agent_outputs entry is full 8-block."""
    saved = Path("tests/e2e/.last_approved_wf")
    if not saved.exists():
        pytest.skip("T3 did not record a workflow_id; run test_T03 first.")
    wf_id = saved.read_text().strip()

    st = requests.get(f"{BACKEND}/api/v1/workflows/{wf_id}").json()
    outputs = st["agent_outputs_summary"]
    assert "transcriber" in outputs and outputs["transcriber"] >= 1
    assert "drafter" in outputs and outputs["drafter"] >= 1
    assert "reviewer" in outputs and outputs["reviewer"] >= 1

    # Deep check via direct DB read
    from app.storage.rds_models import EvidencePack, get_db
    with get_db() as session:
        pack = session.query(EvidencePack).filter_by(workflow_id=wf_id).one()
        for role, payloads in (pack.agent_outputs or {}).items():
            for payload in payloads:
                missing = REQUIRED_BLOCKS - set(payload.keys())
                assert not missing, f"{role} payload missing: {missing}"
                # Identifier echo check (PAYLOAD_SCHEMA §1)
                assert payload["status"]["workflow_id"] == wf_id
                assert payload["status"]["trace_id"], "trace_id not echoed"
                assert payload["status"]["workflow_template_id"] == "brd-from-audio-v1"
```

---

## STEP 12: T11 — Clean restart

```python
# tests/e2e/test_T11_clean_restart.py
import os
import subprocess
import time
import requests

BACKEND = os.environ.get("BACKEND_URL", "http://localhost:8000")


def test_docker_compose_down_then_up_recovers():
    subprocess.check_call(["docker", "compose", "down"])
    time.sleep(2)
    subprocess.check_call(["docker", "compose", "up", "-d"])

    deadline = time.time() + 90
    while time.time() < deadline:
        try:
            r = requests.get(f"{BACKEND}/health", timeout=2)
            if r.status_code == 200:
                return
        except Exception:
            pass
        time.sleep(2)
    raise AssertionError("FastAPI did not come back online after clean restart")
```

---

## GATE PASS CHECKLIST — Phase 6 Complete

- [ ] **G6-1** Run `./scripts/e2e_test.sh`; summary shows `Failed: 0`.
- [ ] **G6-2** T9 confirms `demo.*` attributes in CloudWatch Logs for all 3 agent groups.
- [ ] **G6-3** T10 confirms every agent's 8-block payload echoes inbound identifiers (Block 1 Status).
- [ ] **G6-4** T11 confirms `docker compose down && up` recovers fully.
- [ ] **G6-5** At least one APPROVED workflow exists in the `evidence_packs` table.
- [ ] **G6-6** No `--protocol AGUI`, `BRDDemo/Orchestrator`, `jnj.*`, or `cloudwatch_emf` in any tested file.
- [ ] **G6-7** Happy path (T3) total wall-clock ≤ 5 minutes for a 3-second audio sample.

**DECISION:**
- All checks pass → Phase 6 COMPLETE; proceed to `Prompt_07_Metrics_Dashboard.md`.
- Any check fails → STOP; report failing test ID and the e2e log file path.

---

## TESTING REQUIREMENTS — pytest layout

```
tests/
  e2e/
    fixtures/
      smoke-3sec.wav     # generated by seed_test_audio.py
      smoke-3sec.webm    # transcoded by T4
    .last_approved_wf    # written by T3, read by T10
    seed_test_audio.py
    test_T01_container_health.py
    test_T02_service_endpoints.py
    test_T03_happy_path_upload.py
    test_T04_happy_path_mic.py
    test_T05_hitl_clarification.py
    test_T06_self_correction_cap.py
    test_T07_rejection.py
    test_T08_revision_cycle.py
    test_T09_cloudwatch_demo_attributes.py
    test_T10_evidence_pack_validation.py
    test_T11_clean_restart.py
```

Run: `./scripts/e2e_test.sh` (orchestrator) or `uv run pytest tests/e2e/ -v` (direct).

---

*End of Prompt 06. Phase 6 closes out the implementation work; Phase 7 (Metrics Dashboard) is the final deliverable.*
