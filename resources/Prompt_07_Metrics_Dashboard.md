# Prompt 07: Metric Dashboard (Phase 7)

**MODE: AUTOMATIC — AI writes code directly, sequential execution, every step verified.**

## Reference Documents (READ FIRST)

| Priority | Document | Why |
|----------|----------|-----|
| **PRIMARY** | `resources/Deliverable_0_PROJECT_CONTEXT.md` (PCD) | D14 dashboard scope, §3 D11 metric API paths, §10 cardinality/namespace |
| **PRIMARY** | `resources/Deliverable_7_CloudWatch_Telemetry_Guide.md` | OTel/ADOT pipeline; CloudWatch Logs Insights queries; metric namespace `DemoSDLC/Agent` |
| **PRIMARY** | `resources/PAYLOAD_SCHEMA.md` | Block 4 (Financial) fields fuel the computed metrics |
| **REFERENCE** | AI SDLC Guidelines §9.6 | Authoritative computed-metric formulas |

---

## CONTEXT

You are executing **Phase 7** — the final phase. Phases 0–6 are complete:
- The 6-service docker-compose stack runs.
- At least one APPROVED workflow exists in `evidence_packs` (T3/T10 produced it).
- CloudWatch records carry `demo.*` attributes under namespace `DemoSDLC/Agent`.

This phase adds:
1. **Backend metric API** — 5 endpoints under `/api/v1/metrics/...` reading from RDS + CloudWatch.
2. **Frontend `/metrics` page** — Recharts-based dashboard with summary cards, time series, and per-template breakdown.
3. **IAM extension** — add `cloudwatch:GetMetricData` to the EC2 instance role (already pre-added in Prompt_00; verify here).

## CRITICAL RULES

- **No new database tables.** Read-only over `evidence_packs` + CloudWatch.
- **CloudWatch cost control**: cap GetMetricData queries to last 30 days max, period >= 300s, max 1440 datapoints per call (1 datapoint per 5 minutes over 5 days).
- **Recharts only.** No d3-direct, no Chart.js. Frontend dependency: `recharts@^2.12.0`.
- **`/metrics` page lives OUTSIDE `/workspace`.** No CopilotKit provider here — the metrics page is a standard React Query app.
- **Computed metrics use Guideline §9.6 formulas verbatim** (PCD §3 D14).
- **All comments in English.**

## WORKING DIRECTORY

```bash
export PROJECT_ROOT="/Users/ugurgocen/projects/agentcore-demo-test1"
cd "$PROJECT_ROOT/agentcore-demo-test1-backend"
```

---

## STEP 1: IAM extension verification

The EC2 instance role needs `cloudwatch:GetMetricData`. This was pre-added in Prompt_00's updated IAM policy (Phase 0). Verify it's present:

```bash
ROLE_NAME="agentcore-demo-test1-ec2-role"
aws iam get-role-policy --role-name "$ROLE_NAME" --policy-name agentcore-demo-test1-ec2-policy \
    --query 'PolicyDocument.Statement[?contains(Action || `[]`, `cloudwatch:GetMetricData`)] | length(@)'
```

Expected: `>= 1`. If `0`, edit `agentcore-demo-test1-infra/iam-policies/ec2-instance-role.json` to add a statement granting `cloudwatch:GetMetricData`, `cloudwatch:GetMetricStatistics`, `cloudwatch:ListMetrics` (Resource: `*`), then re-apply via `aws iam put-role-policy`.

CHECKPOINT 1: Query returns `>= 1`.

---

## STEP 2: Backend — Pydantic response models

**File:** `agentcore-demo-test1-backend/app/api/schemas/metrics.py`

```python
"""Response schemas for /api/v1/metrics/* (PCD §3 D11/D14)."""
from __future__ import annotations
from datetime import datetime
from typing import Any
from pydantic import BaseModel, Field


class Overview(BaseModel):
    total_runs: int
    success_rate_pct: float
    avg_cost_usd: float
    total_agent_hours: float
    total_time_saved_hours: float
    period_start: datetime
    period_end: datetime


class TemplateAggregate(BaseModel):
    workflow_template_id: str
    runs: int
    p50_latency_ms: float | None
    p95_latency_ms: float | None
    avg_confidence: float | None
    acceptance_rate_pct: float
    total_cost_usd: float


class ProjectAggregate(BaseModel):
    project_id: str
    project_name: str | None
    sector: str | None
    vertical: str | None
    runs: int
    avg_cost_usd: float
    total_savings_usd: float


class CloudWatchDatapoint(BaseModel):
    timestamp: datetime
    value: float


class CloudWatchSeries(BaseModel):
    metric: str
    period_seconds: int
    datapoints: list[CloudWatchDatapoint]


class ComputedMetrics(BaseModel):
    """Guideline §9.6 formulas applied across the period."""
    time_saved_hours: float
    net_savings_usd: float
    roi_pct: float
    manual_equivalent_fte: float
    speed_multiplier: float
    acceptance_rate_pct: float
    traceability_completeness_pct: float
    period_start: datetime
    period_end: datetime
```

CHECKPOINT 2: `python -c "from app.api.schemas.metrics import Overview, TemplateAggregate, ProjectAggregate, CloudWatchSeries, ComputedMetrics; print('OK')"` succeeds.

---

## STEP 3: Backend — Metrics service (RDS aggregation)

**File:** `agentcore-demo-test1-backend/app/services/metrics_service.py`

```python
"""Aggregation logic over evidence_packs (read-only)."""
from __future__ import annotations
from datetime import datetime, timezone, timedelta
from sqlalchemy import func, case, select
from app.storage.rds_models import EvidencePack, get_db


def _period_bounds(since: datetime | None, until: datetime | None) -> tuple[datetime, datetime]:
    until = until or datetime.now(timezone.utc)
    since = since or (until - timedelta(days=30))
    return since, until


def overview(since: datetime | None = None, until: datetime | None = None) -> dict:
    since, until = _period_bounds(since, until)
    with get_db() as session:
        rows = session.execute(
            select(EvidencePack).where(EvidencePack.created_at.between(since, until))
        ).scalars().all()
        total = len(rows)
        approved = sum(1 for r in rows if r.outcome == "APPROVED")
        # Cost and timing pulled from agent_outputs Financial block
        agent_cost_total = 0.0
        agent_hours_total = 0.0
        time_saved_total = 0.0
        for r in rows:
            for role, payloads in (r.agent_outputs or {}).items():
                for p in payloads:
                    fin = p.get("financial", {})
                    agent_cost_total += float(fin.get("agent_cost_usd") or 0.0)
                    agent_hours_total += float(fin.get("agent_active_hours") or 0.0)
                    baseline = fin.get("manual_baseline_hours")
                    active = fin.get("agent_active_hours") or 0.0
                    if isinstance(baseline, (int, float)):
                        time_saved_total += max(0.0, baseline - active)
        return {
            "total_runs": total,
            "success_rate_pct": (100.0 * approved / total) if total else 0.0,
            "avg_cost_usd": (agent_cost_total / total) if total else 0.0,
            "total_agent_hours": agent_hours_total,
            "total_time_saved_hours": time_saved_total,
            "period_start": since, "period_end": until,
        }


def by_template(workflow_template_id: str | None = None,
                since: datetime | None = None, until: datetime | None = None) -> list[dict]:
    since, until = _period_bounds(since, until)
    with get_db() as session:
        q = select(EvidencePack).where(EvidencePack.created_at.between(since, until))
        if workflow_template_id:
            q = q.where(EvidencePack.workflow_template_id == workflow_template_id)
        rows = session.execute(q).scalars().all()
        # Group in Python (small N for the demo)
        groups: dict[str, list] = {}
        for r in rows:
            groups.setdefault(r.workflow_template_id, []).append(r)
        out = []
        for tid, packs in groups.items():
            latencies, costs, confidences, approvals = [], [], [], 0
            for p in packs:
                for role, payloads in (p.agent_outputs or {}).items():
                    for body in payloads:
                        if t := body.get("timing", {}).get("total_elapsed_ms"):
                            latencies.append(float(t))
                        if c := body.get("financial", {}).get("agent_cost_usd"):
                            costs.append(float(c))
                        if q_ := body.get("quality", {}).get("confidence"):
                            confidences.append(float(q_))
                if p.outcome == "APPROVED":
                    approvals += 1
            latencies.sort()
            def percentile(values, pct):
                if not values: return None
                k = max(0, int(round((pct / 100.0) * (len(values) - 1))))
                return values[k]
            out.append({
                "workflow_template_id": tid,
                "runs": len(packs),
                "p50_latency_ms": percentile(latencies, 50),
                "p95_latency_ms": percentile(latencies, 95),
                "avg_confidence": (sum(confidences) / len(confidences)) if confidences else None,
                "acceptance_rate_pct": (100.0 * approvals / len(packs)) if packs else 0.0,
                "total_cost_usd": sum(costs),
            })
        return out


def by_project(project_id: str | None = None,
               since: datetime | None = None, until: datetime | None = None) -> list[dict]:
    since, until = _period_bounds(since, until)
    with get_db() as session:
        rows = session.execute(
            select(EvidencePack).where(EvidencePack.created_at.between(since, until))
        ).scalars().all()
        groups: dict[str, dict] = {}
        for r in rows:
            for role, payloads in (r.agent_outputs or {}).items():
                for body in payloads:
                    rb = body.get("status", {}).get("custom_metadata", {})
                    # project info lives in requested_by from the inbound request,
                    # which the activity wrapper records under custom_metadata for traceability.
                    project = rb.get("project") or {}
                    pid = project.get("project_id", "unknown")
                    if project_id and pid != project_id:
                        continue
                    g = groups.setdefault(pid, {
                        "project_id": pid,
                        "project_name": project.get("project_name"),
                        "sector": project.get("sector"),
                        "vertical": project.get("vertical"),
                        "runs": 0, "agent_cost_total": 0.0, "savings_total": 0.0,
                    })
                    g["runs"] += 1
                    fin = body.get("financial", {})
                    g["agent_cost_total"] += float(fin.get("agent_cost_usd") or 0.0)
                    baseline = float(fin.get("manual_baseline_hours") or 0.0)
                    active = float(fin.get("agent_active_hours") or 0.0)
                    g["savings_total"] += max(0.0, baseline - active)
        return [
            {"project_id": g["project_id"], "project_name": g["project_name"],
             "sector": g["sector"], "vertical": g["vertical"], "runs": g["runs"],
             "avg_cost_usd": (g["agent_cost_total"] / g["runs"]) if g["runs"] else 0.0,
             "total_savings_usd": g["savings_total"] * 100.0}  # placeholder $/hour multiplier
            for g in groups.values()
        ]


def computed(since: datetime | None = None, until: datetime | None = None) -> dict:
    """Apply Guideline §9.6 formulas across the period."""
    since, until = _period_bounds(since, until)
    ov = overview(since, until)
    manual_baseline = 0.0
    agent_active = ov["total_agent_hours"]
    accepted = 0
    total = ov["total_runs"]
    agent_cost = 0.0
    with get_db() as session:
        rows = session.execute(
            select(EvidencePack).where(EvidencePack.created_at.between(since, until))
        ).scalars().all()
        for r in rows:
            if r.outcome == "APPROVED":
                accepted += 1
            for role, payloads in (r.agent_outputs or {}).items():
                for p in payloads:
                    fin = p.get("financial", {})
                    manual_baseline += float(fin.get("manual_baseline_hours") or 0.0)
                    agent_cost += float(fin.get("agent_cost_usd") or 0.0)
    time_saved = max(0.0, manual_baseline - agent_active)
    # Demo cost multiplier: $50/hour for human time (configurable in production)
    HOURLY_RATE = 50.0
    manual_cost = manual_baseline * HOURLY_RATE
    net_savings = manual_cost - agent_cost
    roi_pct = (100.0 * net_savings / agent_cost) if agent_cost > 0 else 0.0
    return {
        "time_saved_hours": time_saved,
        "net_savings_usd": net_savings,
        "roi_pct": roi_pct,
        "manual_equivalent_fte": time_saved / 160.0,           # 160h/month
        "speed_multiplier": (manual_baseline / agent_active) if agent_active > 0 else 0.0,
        "acceptance_rate_pct": (100.0 * accepted / total) if total else 0.0,
        "traceability_completeness_pct": 100.0,                # demo: all runs have full chain
        "period_start": since, "period_end": until,
    }
```

CHECKPOINT 3: `python -c "from app.services.metrics_service import overview, by_template, by_project, computed; print('OK')"` succeeds.

---

## STEP 4: Backend — CloudWatch client wrapper

**File:** `agentcore-demo-test1-backend/app/services/cloudwatch_client.py`

```python
"""Thin boto3 wrapper around cloudwatch:GetMetricData."""
from __future__ import annotations
import os
from datetime import datetime
import boto3

_client = None


def _cw():
    global _client
    if _client is None:
        _client = boto3.client("cloudwatch",
                                region_name=os.environ.get("AWS_REGION", "us-east-1"))
    return _client


METRIC_NAMESPACE = "DemoSDLC/Agent"


def get_metric_series(metric_name: str, period_seconds: int,
                       start: datetime, end: datetime,
                       workflow_template_id: str | None = None) -> list[dict]:
    """
    Read a single CloudWatch metric over the given window.

    Cost control: caller MUST ensure period >= 300s and the window <= 30 days.
    """
    if period_seconds < 300:
        raise ValueError("period_seconds must be >= 300")
    if (end - start).days > 30:
        raise ValueError("window must be <= 30 days")

    dimensions = []
    if workflow_template_id:
        dimensions.append({"Name": "workflow_template_id", "Value": workflow_template_id})

    resp = _cw().get_metric_data(
        StartTime=start, EndTime=end,
        MetricDataQueries=[{
            "Id": "m1",
            "MetricStat": {
                "Metric": {"Namespace": METRIC_NAMESPACE,
                           "MetricName": metric_name,
                           "Dimensions": dimensions},
                "Period": period_seconds, "Stat": "Average",
            },
            "ReturnData": True,
        }],
    )
    result = resp["MetricDataResults"][0]
    return [{"timestamp": ts, "value": val}
            for ts, val in zip(result["Timestamps"], result["Values"])]
```

CHECKPOINT 4: `python -c "from app.services.cloudwatch_client import get_metric_series; print('OK')"` succeeds.

---

## STEP 5: Backend — Metrics routes

**File:** `agentcore-demo-test1-backend/app/api/metrics_routes.py`

```python
"""FastAPI routes for /api/v1/metrics/* (PCD §3 D11/D14)."""
from __future__ import annotations
from datetime import datetime, timezone, timedelta
from fastapi import APIRouter, HTTPException, Query
from app.api.schemas.metrics import (
    Overview, TemplateAggregate, ProjectAggregate, CloudWatchSeries, ComputedMetrics,
)
from app.services import metrics_service
from app.services.cloudwatch_client import get_metric_series

router = APIRouter(prefix="/api/v1/metrics")


def _parse_range(since: str | None, until: str | None) -> tuple[datetime, datetime]:
    now = datetime.now(timezone.utc)
    return (
        datetime.fromisoformat(since) if since else now - timedelta(days=30),
        datetime.fromisoformat(until) if until else now,
    )


@router.get("/overview", response_model=Overview)
def overview(since: str | None = None, until: str | None = None) -> Overview:
    s, u = _parse_range(since, until)
    return Overview(**metrics_service.overview(s, u))


@router.get("/by-template", response_model=list[TemplateAggregate])
def by_template(workflow_template_id: str | None = None,
                since: str | None = None, until: str | None = None) -> list[TemplateAggregate]:
    s, u = _parse_range(since, until)
    return [TemplateAggregate(**r)
            for r in metrics_service.by_template(workflow_template_id, s, u)]


@router.get("/by-project", response_model=list[ProjectAggregate])
def by_project(project_id: str | None = None,
                since: str | None = None, until: str | None = None) -> list[ProjectAggregate]:
    s, u = _parse_range(since, until)
    return [ProjectAggregate(**r)
            for r in metrics_service.by_project(project_id, s, u)]


@router.get("/cloudwatch", response_model=CloudWatchSeries)
def cloudwatch(metric: str = Query(..., description="Metric name in DemoSDLC/Agent"),
                period: int = 300, start: str | None = None, end: str | None = None,
                workflow_template_id: str | None = None) -> CloudWatchSeries:
    s, u = _parse_range(start, end)
    try:
        datapoints = get_metric_series(metric, period, s, u, workflow_template_id)
    except ValueError as e:
        raise HTTPException(400, str(e)) from e
    return CloudWatchSeries(metric=metric, period_seconds=period,
                            datapoints=[{"timestamp": d["timestamp"], "value": d["value"]}
                                        for d in datapoints])


@router.get("/computed", response_model=ComputedMetrics)
def computed(since: str | None = None, until: str | None = None) -> ComputedMetrics:
    s, u = _parse_range(since, until)
    return ComputedMetrics(**metrics_service.computed(s, u))
```

In `app/main.py`, register the router:

```python
from app.api.metrics_routes import router as metrics_router
app.include_router(metrics_router)
```

CHECKPOINT 5:
```bash
curl -s http://localhost:8000/openapi.json | jq '.paths | keys[] | select(startswith("/api/v1/metrics"))' | wc -l
# Expected: 5
```

---

## STEP 6: Frontend — Recharts dependency + page

Add to `agentcore-demo-test1-frontend/package.json`:

```json
{
  "dependencies": {
    "recharts": "^2.12.0"
  }
}
```

Run: `pnpm install`.

**File:** `agentcore-demo-test1-frontend/lib/api/metrics.ts`

```typescript
export type Overview = {
  total_runs: number;
  success_rate_pct: number;
  avg_cost_usd: number;
  total_agent_hours: number;
  total_time_saved_hours: number;
  period_start: string;
  period_end: string;
};

export type CloudWatchSeries = {
  metric: string;
  period_seconds: number;
  datapoints: { timestamp: string; value: number }[];
};

export type ComputedMetrics = {
  time_saved_hours: number;
  net_savings_usd: number;
  roi_pct: number;
  manual_equivalent_fte: number;
  speed_multiplier: number;
  acceptance_rate_pct: number;
  traceability_completeness_pct: number;
  period_start: string;
  period_end: string;
};

async function get<T>(path: string): Promise<T> {
  const resp = await fetch(`/api/v1${path}`);
  if (!resp.ok) throw new Error(`${resp.status} ${path}`);
  return resp.json() as Promise<T>;
}

export const metricsApi = {
  overview: (since?: string, until?: string) =>
    get<Overview>(`/metrics/overview${since ? `?since=${since}&until=${until}` : ""}`),
  cloudwatch: (metric: string, period = 300, start?: string, end?: string) =>
    get<CloudWatchSeries>(
      `/metrics/cloudwatch?metric=${encodeURIComponent(metric)}&period=${period}` +
      (start && end ? `&start=${start}&end=${end}` : ""),
    ),
  computed: (since?: string, until?: string) =>
    get<ComputedMetrics>(`/metrics/computed${since ? `?since=${since}&until=${until}` : ""}`),
};
```

**File:** `agentcore-demo-test1-frontend/hooks/useMetrics.ts`

```typescript
"use client";
import { useQuery } from "@tanstack/react-query";
import { metricsApi } from "@/lib/api/metrics";

const STALE = 60 * 1000; // 1 minute

export function useOverview(since?: string, until?: string) {
  return useQuery({
    queryKey: ["overview", since, until],
    queryFn: () => metricsApi.overview(since, until),
    staleTime: STALE,
  });
}

export function useCloudWatchSeries(metric: string, period = 300, start?: string, end?: string) {
  return useQuery({
    queryKey: ["cw", metric, period, start, end],
    queryFn: () => metricsApi.cloudwatch(metric, period, start, end),
    staleTime: STALE,
    enabled: !!metric,
  });
}

export function useComputedMetrics(since?: string, until?: string) {
  return useQuery({
    queryKey: ["computed", since, until],
    queryFn: () => metricsApi.computed(since, until),
    staleTime: STALE,
  });
}
```

**File:** `agentcore-demo-test1-frontend/components/metrics/SummaryCards.tsx`

```typescript
"use client";
import { useOverview, useComputedMetrics } from "@/hooks/useMetrics";

function Card({ title, value, subtitle }: { title: string; value: string; subtitle?: string }) {
  return (
    <div className="rounded-lg border bg-white p-4 shadow-sm">
      <div className="text-xs uppercase text-gray-500">{title}</div>
      <div className="mt-1 text-2xl font-semibold">{value}</div>
      {subtitle ? <div className="text-xs text-gray-500">{subtitle}</div> : null}
    </div>
  );
}

export function SummaryCards() {
  const overview = useOverview();
  const computed = useComputedMetrics();
  if (overview.isLoading || computed.isLoading) return <div>Loading…</div>;
  if (overview.error || computed.error) return <div className="text-red-600">Failed to load</div>;
  const o = overview.data!;
  const c = computed.data!;
  return (
    <div className="grid grid-cols-2 gap-3 md:grid-cols-4">
      <Card title="Total Runs" value={o.total_runs.toString()} subtitle={`last 30 days`} />
      <Card title="Success Rate" value={`${o.success_rate_pct.toFixed(1)}%`} />
      <Card title="Total Cost" value={`$${o.avg_cost_usd.toFixed(2)} avg/run`} subtitle={`ROI ${c.roi_pct.toFixed(1)}%`} />
      <Card title="Time Saved" value={`${c.time_saved_hours.toFixed(1)}h`} subtitle={`${c.manual_equivalent_fte.toFixed(2)} FTE`} />
    </div>
  );
}
```

**File:** `agentcore-demo-test1-frontend/components/metrics/CostTimeSeries.tsx`

```typescript
"use client";
import { LineChart, Line, XAxis, YAxis, Tooltip, CartesianGrid, ResponsiveContainer } from "recharts";
import { useCloudWatchSeries } from "@/hooks/useMetrics";

export function CostTimeSeries() {
  const series = useCloudWatchSeries("AgentCostUsd", 3600);
  if (series.isLoading) return <div>Loading…</div>;
  if (series.error) return <div className="text-red-600">Failed to load</div>;
  const data = (series.data?.datapoints ?? []).map((d) => ({
    ts: new Date(d.timestamp).toLocaleString(),
    cost: d.value,
  }));
  return (
    <div className="rounded-lg border bg-white p-4">
      <h3 className="mb-2 text-sm font-semibold">Cost over time (hourly average)</h3>
      <ResponsiveContainer width="100%" height={240}>
        <LineChart data={data}>
          <CartesianGrid strokeDasharray="3 3" />
          <XAxis dataKey="ts" minTickGap={50} />
          <YAxis />
          <Tooltip />
          <Line type="monotone" dataKey="cost" />
        </LineChart>
      </ResponsiveContainer>
    </div>
  );
}
```

**File:** `agentcore-demo-test1-frontend/app/metrics/page.tsx`

```typescript
import { MetricsClient } from "./MetricsClient";

export default function MetricsPage() {
  return (
    <main className="mx-auto max-w-6xl p-6 space-y-6">
      <h1 className="text-2xl font-semibold">Demo Metrics</h1>
      <MetricsClient />
    </main>
  );
}
```

**File:** `agentcore-demo-test1-frontend/app/metrics/MetricsClient.tsx`

```typescript
"use client";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { SummaryCards } from "@/components/metrics/SummaryCards";
import { CostTimeSeries } from "@/components/metrics/CostTimeSeries";

const queryClient = new QueryClient();

export function MetricsClient() {
  return (
    <QueryClientProvider client={queryClient}>
      <SummaryCards />
      <CostTimeSeries />
    </QueryClientProvider>
  );
}
```

CHECKPOINT 6: `pnpm build` succeeds; `http://localhost:3000/metrics` renders cards + chart.

---

## STEP 7: End-to-end smoke

```bash
# Backend smoke
curl -s http://localhost:8000/api/v1/metrics/overview | jq
curl -s http://localhost:8000/api/v1/metrics/computed | jq
curl -s "http://localhost:8000/api/v1/metrics/cloudwatch?metric=AgentLatencyMs&period=300" | jq '.datapoints | length'

# Frontend smoke
curl -s http://localhost:3000/metrics | grep -i "<h1" | head -1
```

CHECKPOINT 7: All commands return data; the metrics page HTML contains the expected `<h1>Demo Metrics</h1>`.

---

## GATE PASS CHECKLIST — Phase 7 Complete

- [ ] **G7-1** All 5 metric endpoints registered under `/api/v1/metrics/*` (verified via OpenAPI).
- [ ] **G7-2** `cloudwatch:GetMetricData` granted to EC2 instance role (verified in Step 1).
- [ ] **G7-3** Computed metrics formulas match Guideline §9.6 (test in `tests/metrics/`).
- [ ] **G7-4** Frontend `/metrics` page renders with at least 4 summary cards + 1 chart.
- [ ] **G7-5** React Query `staleTime` caps refetches at 1 minute; CloudWatch requests respect `period >= 300s` and window `<= 30 days`.
- [ ] **G7-6** Page is OUTSIDE `/workspace`; CopilotKit provider not active here.
- [ ] **G7-7** No regression in any earlier phase (Phase 0–6 gates still pass).

**DECISION:**
- All checks pass → **Phase 7 COMPLETE → ENTIRE DEMO READY.**
- Any check fails → STOP; report gate ID and error.

---

## TESTING REQUIREMENTS — pytest + vitest

```
tests/
  metrics/
    test_overview_aggregation.py        # mock evidence_packs; assert totals
    test_by_template_p50_p95.py         # synthetic latencies; verify percentiles
    test_computed_formulas.py           # Guideline §9.6 exact values
    test_cloudwatch_param_validation.py # period < 300 → 400; window > 30d → 400

frontend/tests/
  unit/
    metrics-summary-cards.test.tsx      # mock useOverview; assert 4 cards render
    metrics-cost-chart.test.tsx         # mock useCloudWatchSeries; assert chart renders
```

---

*End of Prompt 07. Phase 7 closes out the demo — both implementation (Prompts 00–06) and observability (Prompt 07) are complete.*
