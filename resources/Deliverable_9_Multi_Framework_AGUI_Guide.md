# Deliverable 9: Multi-Framework AG-UI Strategy Guide

## How to Mix, Match, and Migrate Agent Frameworks Without Rewriting Your Frontend

**Document ID:** DELIV-009  
**Version:** 1.0  
**Date:** 2025-01-28  
**Status:** BASELINE  
**Classification:** Architecture Strategy Guide  
**Scope:** Framework-agnostic AG-UI protocol design, per-framework adapter libraries, multi-framework routing patterns, event compatibility, migration paths, and protocol disambiguation
> **V5 (2026-05-12):** This demo uses 2 Strands agents + 1 LangGraph agent (Agent 3 Reviewer = LangGraph StateGraph with 4 nodes). BedrockAgentCoreApp with `@app.entrypoint` handles AG-UI streaming. S3 zip deployment — no containers. A2A via Temporal signals — NOT direct agent calls.  

---

## Table of Contents

1. [AG-UI is Framework-Agnostic by Design](#1-ag-ui-is-framework-agnostic-by-design)
2. [Per-Framework Adapter Libraries](#2-per-framework-adapter-libraries)
3. [How Routing Works When Multiple Frameworks Are in Play](#3-how-routing-works-when-multiple-frameworks-are-in-play)
4. [How Does the Frontend Know Which Framework?](#4-how-does-the-frontend-know-which-framework)
5. [Event Compatibility Across Adapters -- The Catch](#5-event-compatibility-across-adapters--the-catch)
6. [Migration Path -- Switching a Single Agent's Framework](#6-migration-path--switching-a-single-agents-framework)
7. [CrewAI / LangGraph Quickstart for Future Projects](#7-crewai--langgraph-quickstart-for-future-projects)
8. [A2A vs AG-UI vs MCP -- Disambiguation](#8-a2a-vs-ag-ui-vs-mcp--disambiguation)
9. [Production Considerations](#9-production-considerations)
10. [Tech Reference](#10-tech-reference)

---

## 1. AG-UI is Framework-Agnostic by Design

### 1.1 The Core Principle

AG-UI (Agent-User Interface) is an **open wire protocol**, not a framework. It specifies a set of event types, a JSON envelope format, and a Server-Sent Events (SSE) transport mechanism. Any agent runtime that can emit JSON lines over an HTTP SSE stream can speak AG-UI. Any frontend that can consume SSE and parse JSON can understand AG-UI.

This means:

| What AG-UI Specifies | What AG-UI Does NOT Specify |
|---|---|
| Event type names (`TEXT_MESSAGE_CONTENT`, `TOOL_CALL_START`, etc.) | Which framework produced the event |
| JSON envelope shape (`{"type": "...", "payload": {...}}`) | How the agent is implemented internally |
| SSE transport over HTTP | The programming language of the agent |
| Standard payload fields per event type | Deployment topology (container, serverless, local) |

### 1.2 On-the-Wire Identity

A `TEXT_MESSAGE_CONTENT` event from a **Strands** agent looks **identically** to one from a **LangGraph** agent, a **CrewAI** crew, or a **Microsoft** agent framework runtime. The frontend cannot distinguish the source framework from the event payload alone -- and that is intentional.

```json
// ============================================================================
// EXAMPLE: Two events, same wire format, different frameworks behind them.
// The frontend receives both and renders them the same way.
// ============================================================================

// Event from Strands agent (via BedrockAgentCoreApp @app.entrypoint — V5)
// The agent emits AG-UI TEXT_MESSAGE_CONTENT events through BedrockAgentCoreApp
// which handles serialization automatically. No manual adapter needed.
{
  "type": "TEXT_MESSAGE_CONTENT",
  "payload": {
    "messageId": "msg-strands-001",
    "content": "Analyzing the audio transcript for business requirements...",
    "delta": "requirements..."
  }
}

// Event from LangGraph agent (native AG-UI emission)
{
  "type": "TEXT_MESSAGE_CONTENT",
  "payload": {
    "messageId": "msg-langgraph-042",
    "content": "Analyzing the audio transcript for business requirements...",
    "delta": "requirements..."
  }
}

// Event from CrewAI agent (via ag-ui-crewai adapter)
{
  "type": "TEXT_MESSAGE_CONTENT",
  "payload": {
    "messageId": "msg-crewai-007",
    "content": "Analyzing the audio transcript for business requirements...",
    "delta": "requirements..."
  }
}
```

### 1.3 Frontend Immunity to Framework Changes

Because the frontend speaks AG-UI -- not framework-specific APIs -- a framework swap behind an AG-UI adapter requires **zero frontend code changes**. Your investment in:

- CopilotKit `useAgent` hook configuration
- Custom Generative UI card components (`TranscriptCard`, `DraftPreviewCard`, `ReviewReportCard`)
- Canvas state machine logic
- Thread management and sidebar history
- Tailwind styling and theme integration

...is **fully preserved** across framework swaps. The only code that changes is the adapter layer and the agent implementation itself.

### 1.4 The Adapter as Abstraction Boundary

```
+-------------------------------------------------------------+
|                    FRONTEND (Next.js + CopilotKit)           |
|                                                              |
|  useAgent() <-- AG-UI SSE stream --> Generic event handlers  |
|  TranscriptCard, DraftPreviewCard, ReviewReportCard          |
|                                                              |
|  [NO FRAMEWORK-SPECIFIC CODE]                                |
+---------------------------+----------------------------------+
                            |  AG-UI Protocol (SSE / JSON)
                            v
+---------------------------+----------------------------------+
|                    ADAPTER LAYER                             |
|                                                              |
|  Strands:  bedrock_agentcore.BedrockAgentCoreApp  (this demo — V5) |
|  LangGraph: Built-in emit_ag_ui_event()      (native)        |
|  CrewAI:    ag_ui_crewai.AGUIStream          (wrapper)       |
|  Microsoft: agent_framework.ag_ui bridge     (official)      |
|  ...                                                         |
|                                                              |
|  [FRAMEWORK-SPECIFIC CODE -- ISOLATED HERE]                  |
+---------------------------+----------------------------------+
                            |  Internal framework calls
                            v
+---------------------------+----------------------------------+
|                    AGENT RUNTIME                             |
|                                                              |
|  Strands Agent  |  LangGraph StateGraph  |  CrewAI Crew     |
|  (AWS Bedrock)  |  (Python/JS)           |  (multi-agent)   |
|                                                              |
|  [AGENT LOGIC -- REPLACEABLE INDEPENDENTLY]                  |
+-------------------------------------------------------------+
```

---

## 2. Per-Framework Adapter Libraries

### 2.1 Adapter Support Matrix

The following table lists all known AG-UI adapter implementations as of January 2025. Check the official AG-UI protocol repository for the latest additions.

| Framework | Adapter Package / Mechanism | Maintainer | Status | Notes |
|---|---|---|---|---|
| **AWS Strands** | `bedrock-agentcore` | AWS Official | **Stable** | V5: `BedrockAgentCoreApp` + `@app.entrypoint` — what we use in this demo. S3 ZIP deploy. Agent 1 & 2 use Strands SDK for tool definition only; runtime is always `BedrockAgentCoreApp`. |
| **LangGraph** | Native AG-UI emission | LangChain/LangGraph Official | **Stable** | Built into `langgraph>=0.2.x`. No separate package. Call `emit_ag_ui_event()` inside node functions. |
| **CrewAI** | `ag-ui-crewai` | Community + AWS supported | **Beta** | `AGUIStream` wrapper around `Crew.kickoff()`. Emits events for each task execution. |
| **Microsoft Agent Framework** | `agent_framework.ag_ui` | Microsoft Official | **Preview** | Built into the Microsoft Agent Framework SDK. Exports `AgUiEventStream` class. |
| **AG2 / AutoGen** | `AGUIStream` wrapper | Community | **Experimental** | Wraps `ConversableAgent.run()`. Emits `TEXT_MESSAGE_*` for each agent turn in group chat. |
| **Google ADK** | Native via `Agentic UI` module | Google Official | **Preview** | Google's Agent Development Kit includes `agentic_ui` module with native AG-UI emission. |
| **Mastra** | Built-in AG-UI support | Mastra Official | **Stable** | Native AG-UI streaming built into Mastra framework. Zero-config after `mastra init`. |
| **OpenAI Agents** | AG-UI SDK compatible | OpenAI + Community | **Stable** | OpenAI Agents SDK outputs compatible events via `ag-ui-sdk` bridge package. |
| **LlamaIndex** | `llama-ag-ui` | Community | **Experimental** | Adapter for LlamaIndex agent workflows. Event coverage varies by agent type. |
| **Agno** | `agno-ag-ui` | Community | **Experimental** | Lightweight adapter for Agno (formerly PhiData) agents. |
| **Pydantic AI** | Native AG-UI support | Pydantic Official | **Preview** | Planned native support in Pydantic AI roadmap. Check latest releases. |

### 2.2 Adapter Maturity Levels

```
Stable    -> Production-ready, documented, tested against AG-UI spec compliance suite
Beta      -> Functional, API may change, limited edge-case handling
Preview   -> Available, may have gaps in event coverage, active development
Experimental -> Community-maintained, varying completeness, best for prototyping
```

### 2.3 How Adapters Work Internally

Every AG-UI adapter performs the same three functions:

1. **Intercept agent lifecycle** -- hook into the framework's execution model (Strands `Agent.invoke()`, LangGraph `StateGraph.invoke()`, CrewAI `Crew.kickoff()`, etc.)
2. **Translate to AG-UI events** -- convert framework-native execution events to AG-UI event types
3. **Emit over SSE** -- serialize to JSON lines and stream over HTTP Server-Sent Events

```python
# ============================================================================
# CONCEPTUAL: What every AG-UI adapter does internally
# ============================================================================
# Pseudocode showing the universal adapter pattern:

class BaseAgUiAdapter:
    """All adapters implement this conceptual interface."""

    def __init__(self, agent_runtime, config: dict):
        self.runtime = agent_runtime
        self.config = config

    async def run(self, user_input: str) -> AsyncIterator[dict]:
        # 1. Emit RUN_STARTED
        yield {"type": "RUN_STARTED", "payload": {"runId": str(uuid4())}}

        # 2. Intercept framework execution and translate events
        async for framework_event in self.runtime.execute(user_input):
            agui_event = self._translate(framework_event)
            yield agui_event

        # 3. Emit RUN_FINISHED
        yield {"type": "RUN_FINISHED", "payload": {"runId": ...}}

    def _translate(self, framework_event: Any) -> dict:
        """Framework-specific translation. Each adapter overrides this."""
        ...
```

---

## 3. How Routing Works When Multiple Frameworks Are in Play

When a workflow involves agents from different frameworks (e.g., Strands for transcription, LangGraph for reasoning, CrewAI for multi-agent review), three routing patterns are available.

### 3.1 Pattern A: Sequential Separate Containers (Recommended)

**What we use in this demo.** Each agent runs in its own AgentCore Runtime container (or other deployment target), possibly using a different framework. The frontend switches the `runtime-id` (or endpoint URL) as the workflow progresses through phases.

```
+---------------------------------------------------------------+
|                      NEXT.JS FRONTEND                          |
|                                                                |
|   useAgent({ id: "transcriber" })  -->  Runtime A (Strands)   |
|   useAgent({ id: "drafter" })      -->  Runtime B (Strands)   |
|   useAgent({ id: "reviewer" })     -->  Runtime C (LangGraph) |
|                                                                |
|   Frontend switches runtime-id by workflow phase.              |
|   Each runtime is a separate microVM or container.              |
+---------------------------------------------------------------+

+-----------+    +-----------+    +-----------+
| AgentCore |    | AgentCore |    | AgentCore |
| Runtime A | -> | Runtime B | -> | Runtime C |
| (Strands) |    | (Strands) |    |(LangGraph)|
| :8080     |    | :8082     |    | :8084     |
+-----------+    +-----------+    +-----------+
```

**Pros:**
- Each agent is independently versioned, scaled, and restarted
- Framework upgrades affect only one container
- Local development can mix dockerized and locally-running agents
- Clean failure isolation: one agent crash doesn't affect others

**Cons:**
- Higher baseline resource usage (N containers for N agents)
- Slightly higher inter-agent latency (HTTP hops between containers)
- More endpoints to manage in infrastructure

**Frontend Implementation:**

```typescript
// ============================================================================
// Pattern A Frontend: Switch runtime by workflow phase
// ============================================================================

import { useAgent } from "@copilotkit/react-core";

function WorkflowOrchestrator({ phase }: { phase: "transcribe" | "draft" | "review" }) {
  // Map phase to runtime endpoint
  const runtimeConfig = {
    transcriber: { id: "transcriber", runtimeUrl: process.env.NEXT_PUBLIC_TRANSCRIBER_URL },
    drafter:     { id: "drafter",     runtimeUrl: process.env.NEXT_PUBLIC_DRAFTER_URL },
    reviewer:    { id: "reviewer",    runtimeUrl: process.env.NEXT_PUBLIC_REVIEWER_URL },
  }[phase];

  const agent = useAgent({
    id: runtimeConfig.id,
    name: `${phase}-agent`,
  });

  // Agent events render identically regardless of which runtime is behind.
  // TranscriptCard handles AG-UI events from Strands OR LangGraph OR CrewAI.
  return <TranscriptCard events={agent.events} status={agent.status} />;
}
```

### 3.2 Pattern B: Multi-Agent Inside One Container

One container runs an orchestrator that internally delegates to agents of different frameworks. From the frontend's perspective, there is only **one AG-UI session**.

```
+---------------------------------------------------------------+
|                    SINGLE CONTAINER (:8080)                    |
|                                                                |
|   +----------------+     +---------+     +---------+          |
|   | Orchestrator   | --> | Agent A |     | Agent B |          |
|   | (Microsoft WF  |     |(Strands)|     |(CrewAI) |          |
|   |  WorkflowBuilder|    +---------+     +---------+          |
|   |  add_handoff)  |                                          |
|   +-------+--------+                                          |
|           |                                                    |
|           v  Single AG-UI SSE stream                          |
+-----------+---------------------------------------------------+
            |
+-----------v---------------------------------------------------+
|   FRONTEND: useAgent({ id: "orchestrator" }) -- ONE endpoint  |
+---------------------------------------------------------------+
```

**Best for:**
- Tight latency requirements (no inter-container HTTP hops)
- Simplified infrastructure (one runtime to manage)
- Workflows where agents share in-memory state

**Implementation sketch (Microsoft Agent Framework):**

```python
# ============================================================================
# Pattern B: Microsoft WorkflowBuilder orchestrating multiple frameworks
# ============================================================================
from agent_framework import WorkflowBuilder
from bedrock_agentcore.runtime import BedrockAgentCoreApp  # V5 runtime (Strands tool definition)
from ag_ui_crewai import AGUIStream                          # CrewAI adapter

builder = WorkflowBuilder()

# Add a Strands-based transcription agent (V5: BedrockAgentCoreApp + @app.entrypoint)
builder.add_agent(
    name="transcriber",
    runtime=BedrockAgentCoreApp(),  # Entrypoint defined in agent's main.py
)

# Add a CrewAI-based review crew
builder.add_agent(
    name="review_crew",
    runtime=AGUIStream(review_crew),
)

# Add handoff logic
builder.add_handoff("transcriber", "review_crew", condition="transcript_complete")

# Single AG-UI stream from the orchestrator
app = builder.compile_ag_ui_app(port=8080)
```

**Cons:**
- Tighter coupling between agents (shared process)
- Harder to scale individual agents independently
- Python version and dependency conflicts between frameworks in same environment

### 3.3 Pattern C: Cross-Framework via Temporal

Temporal orchestrates a workflow where each activity invokes a different agent framework. Each agent runs in its own container with its own adapter. The frontend sees **three separate AG-UI sessions** coordinated by Temporal signals.

```
+---------------------------------------------------------------+
|  FRONTEND: Three AG-UI sessions + one Workflow Stream          |
|                                                                |
|  useAgent({ id: "transcriber" }) --> AgentCore :8080 (Strands) |
|  useAgent({ id: "drafter" })     --> AgentCore :8082 (Strands) |
|  useAgent({ id: "reviewer" })    --> AgentCore :8084 (CrewAI)  |
|  Workflow Stream SSE             --> FastAPI :8000 (Temporal)  |
+---------------------------------------------------------------+
                            |
                     Temporal gRPC
                            |
+---------------------------------------------------------------+
|  TEMPORAL WORKFLOW                                           |
|                                                                |
|  Activity 1: Invoke Transcriber (Strands)  --> Return result  |
|  Activity 2: Invoke Drafter (Strands)      --> Return result  |
|  Activity 3: Invoke Reviewer (CrewAI)      --> Return result  |
|                                                                |
|  Each activity = separate AgentCore invocation                 |
|  Session isolation via workflow_id as session_id               |
+---------------------------------------------------------------+
```

**This is the hybrid architecture our demo uses.** The Temporal workflow coordinates the multi-agent pipeline while AG-UI streams provide real-time frontend visibility into each agent's execution.

**Key coordination mechanism:**

```python
# ============================================================================
// Pattern C: Temporal workflow invoking different frameworks per activity
// ============================================================================

from temporalio import workflow

@workflow.defn
class AudioToBRDWorkflow:
    @workflow.run
    async def run(self, audio_uri: str) -> dict:
        # Activity 1: Strands transcriber
        transcript = await workflow.execute_activity(
            transcribe_audio,
            args=(audio_uri,),
            start_to_close_timeout=timedelta(minutes=5),
        )

        # Activity 2: Strands drafter with HITL
        draft = await workflow.execute_activity(
            generate_draft,
            args=(transcript,),
            start_to_close_timeout=timedelta(minutes=15),
        )

        # Activity 3: CrewAI reviewer (different framework!)
        review = await workflow.execute_activity(
            run_review_crew,   # <-- Calls CrewAI container
            args=(draft,),
            start_to_close_timeout=timedelta(minutes=10),
        )

        return {"status": "COMPLETE", "draft": draft, "review": review}
```

### 3.4 Pattern Comparison

| Dimension | Pattern A: Separate | Pattern B: Single Container | Pattern C: Temporal |
|---|---|---|---|
| **Isolation** | High (separate processes) | Low (shared process) | High (separate invocations) |
| **Latency** | Medium (HTTP hops) | Low (in-process) | Higher (Temporal overhead) |
| **Scalability** | Per-agent scaling | Scale as unit | Per-activity scaling |
| **Framework mixing** | Easy (any framework per container) | Harder (dep conflicts) | Easy (any framework per activity) |
| **Frontend complexity** | Medium (switch runtime-id) | Low (single endpoint) | High (multi-session + WF stream) |
| **Failure recovery** | Container restart | Full restart | Temporal retry per activity |
| **Our demo uses** | **Yes** | No | Orchestration layer |
| **Recommendation** | **Default choice** | Tight latency needs | Complex long-running workflows |

---

## 4. How Does the Frontend Know Which Framework?

### 4.1 It Doesn't -- And It Doesn't Need To

The frontend speaks AG-UI. It parses event types, not framework names. A `TEXT_MESSAGE_CONTENT` event renders as text whether it came from Strands, LangGraph, CrewAI, or a hand-rolled Python script. The `TOOL_CALL_START` event shows a tool execution spinner regardless of which framework invoked the tool.

```typescript
// ============================================================================
// FRONTEND: Framework-agnostic event handling
// This code works identically for Strands, LangGraph, CrewAI, and any other
// framework that speaks AG-UI.
// ============================================================================

import { useAgent } from "@copilotkit/react-core";

function AgentChatPanel({ agentId }: { agentId: string }) {
  const { events, status, messages } = useAgent({ id: agentId });

  // This handler works for ANY framework behind the agentId.
  // Strands? LangGraph? CrewAI? The code is identical.
  useEffect(() => {
    if (!events) return;
    const unsub = events.subscribe((event: AgentEvent) => {
      switch (event.type) {
        case "TEXT_MESSAGE_CONTENT":
          // Renders text -- framework-agnostic
          appendTextChunk(event.payload.messageId, event.payload.delta);
          break;
        case "TOOL_CALL_START":
          // Shows tool spinner -- framework-agnostic
          showToolExecution(event.payload.toolCallId, event.payload.toolName);
          break;
        case "STATE_DELTA":
          // Applies JSON patch -- framework-agnostic
          applyStatePatch(event.payload.patch);
          break;
        case "RUN_FINISHED":
          // Marks completion -- framework-agnostic
          setPhaseComplete(true);
          break;
      }
    });
    return () => unsub();
  }, [events]);

  // Component rendering is 100% framework-independent
  return (
    <div>
      <StatusIndicator status={status} />
      <MessageList messages={messages} />
      <ToolCallPanel events={events} />
    </div>
  );
}
```

### 4.2 Optional: Framework Annotation for Observability

While not required for rendering, adapters **may** include a `framework` field in `RAW` events for debugging and observability dashboards. This is purely informational and should never drive frontend logic.

```json
// ============================================================================
// OPTIONAL: Framework annotation for observability only
// The frontend MAY log this but MUST NOT branch rendering logic on it.
// ============================================================================

{
  "type": "RAW",
  "payload": {
    "framework": "crewai",
    "version": "0.80.0",
    "raw_event": { "task": "review", "agent": "senior_reviewer" }
  }
}
```

**Rule:** If you find yourself writing `if (framework === "langgraph") { ... }` in your frontend, you have broken the AG-UI abstraction. The correct place for framework-specific logic is in the adapter layer, not the UI.

---

## 5. Event Compatibility Across Adapters -- The Catch

### 5.1 The Universal vs. Framework-Specific Divide

AG-UI event **types** are standardized. However, the **sequence** of events emitted during an agent run varies by framework. This is the primary integration risk when mixing frameworks.

### 5.2 Universally Emitted Events (Safe to Rely On)

These events are emitted by **all** mature adapters. Build your frontend logic around these:

| Event Type | Direction | Guaranteed By | Use In Frontend For |
|---|---|---|---|
| `RUN_STARTED` | Agent --> Frontend | All adapters | Run initialization, status = "thinking" |
| `RUN_FINISHED` | Agent --> Frontend | All adapters | Run completion, status = "idle", cleanup |
| `RUN_ERROR` | Agent --> Frontend | All adapters | Error display, status = "idle" + toast |
| `TEXT_MESSAGE_START` | Agent --> Frontend | All adapters | Message bubble creation |
| `TEXT_MESSAGE_CONTENT` | Agent --> Frontend | All adapters | Token streaming into message bubble |
| `TEXT_MESSAGE_END` | Agent --> Frontend | All adapters | Message finalization, markdown render |
| `TOOL_CALL_START` | Agent --> Frontend | All adapters | Tool execution spinner |
| `TOOL_CALL_END` | Agent --> Frontend | All adapters | Tool result display, hide spinner |
| `STATE_DELTA` | Agent --> Frontend | All adapters | Partial result streaming (JSON Patch) |

### 5.3 Framework-Specific Events (Treat as Enhancements)

These events are emitted by **some** adapters. Use them to enhance UX when available, but do not require them:

| Event Type | Emitters | Meaning | Frontend Handling |
|---|---|---|---|
| `STEP_STARTED` | LangGraph (every graph node), Mastra | Individual computation step began | Optional progress indicator |
| `STEP_FINISHED` | LangGraph, Mastra | Computation step completed | Optional progress update |
| `MESSAGES_DELTA` | LangGraph | Message list state change | Optional: supplement TEXT_MESSAGE_* |
| `CUSTOM_EVENT` | LangGraph | User-defined graph events | Optional: custom card rendering |
| `THREAD_CREATED` | LangGraph | New conversation thread | Optional: sidebar update |

### 5.4 The LangGraph Sequence Difference (Detailed)

LangGraph emits `STEP_STARTED`/`STEP_FINISHED` for **every node** in the state graph. A single agent turn may generate 4-6 step events as the graph traverses nodes.

```
# ============================================================================
# EVENT SEQUENCE COMPARISON: Same logical operation, different frameworks
# ============================================================================

# Strands agent running one tool call:
# (clean, minimal -- what our demo expects)
RUN_STARTED
  TEXT_MESSAGE_START
  TEXT_MESSAGE_CONTENT  (x N chunks)
  TEXT_MESSAGE_END
  TOOL_CALL_START
  TOOL_CALL_END
RUN_FINISHED

# LangGraph agent running equivalent tool call:
# (verbose, includes step boundaries for every graph node)
RUN_STARTED
  STEP_STARTED  { type: "__start__" }          <-- LangGraph-specific
  STEP_FINISHED { stepId: "..." }
  STEP_STARTED  { type: "agent" }                <-- LangGraph-specific
    TEXT_MESSAGE_START
    TEXT_MESSAGE_CONTENT  (x N chunks)
    TEXT_MESSAGE_END
  STEP_FINISHED { stepId: "..." }
  STEP_STARTED  { type: "tools" }                <-- LangGraph-specific
    TOOL_CALL_START
    TOOL_CALL_END
  STEP_FINISHED { stepId: "..." }
  STEP_STARTED  { type: "__end__" }              <-- LangGraph-specific
  STEP_FINISHED { stepId: "..." }
RUN_FINISHED
```

### 5.5 Best Practice: Defensive Event Handling

```typescript
// ============================================================================
// DEFENSIVE: Frontend event handler that works across ALL frameworks
// Handles both minimal (Strands) and verbose (LangGraph) event sequences.
// ============================================================================

function useFrameworkAgnosticHandler(events: AgentEventStream) {
  const [activeSteps, setActiveSteps] = useState(0);
  const [toolCall, setToolCall] = useState<ToolCallInfo | null>(null);

  useEffect(() => {
    if (!events) return;

    const unsub = events.subscribe((event: AgentEvent) => {
      switch (event.type) {
        // === UNIVERSAL: Always handle ===
        case "RUN_STARTED":
          setStatus("thinking");
          break;
        case "RUN_FINISHED":
        case "RUN_ERROR":
          setStatus("idle");
          setActiveSteps(0);
          setToolCall(null);
          break;

        case "TEXT_MESSAGE_START":
          createMessageBubble(event.payload.messageId);
          break;
        case "TEXT_MESSAGE_CONTENT":
          appendToBubble(event.payload.messageId, event.payload.delta);
          break;
        case "TEXT_MESSAGE_END":
          finalizeBubble(event.payload.messageId);
          break;

        case "TOOL_CALL_START":
          setToolCall({ id: event.payload.toolCallId, name: event.payload.toolName });
          break;
        case "TOOL_CALL_END":
          setToolCall(null);
          break;

        // === FRAMEWORK-SPECIFIC: Enhance only, never require ===
        case "STEP_STARTED":
          // LangGraph/Mastra: show optional step indicator
          setActiveSteps(prev => prev + 1);
          if (event.payload.type === "tools") {
            setStatus("working");  // Agent is in tool-execution phase
          }
          break;
        case "STEP_FINISHED":
          setActiveSteps(prev => Math.max(0, prev - 1));
          break;

        // === UNKNOWN: Log and ignore ===
        default:
          console.debug(`[AG-UI] Unhandled event type: ${event.type}`);
          break;
      }
    });

    return () => unsub();
  }, [events]);

  return { activeSteps, toolCall, status };
}
```

### 5.6 Testing Event Sequence Compatibility

When introducing a new framework adapter, run this compatibility checklist:

```bash
# ============================================================================
# EVENT COMPATIBILITY TEST SCRIPT
# Run against each framework adapter before deployment.
# ============================================================================

#!/bin/bash

AG_UI_ENDPOINT="${1:-http://localhost:8080/invocations}"
TEST_PROMPT="Generate a business requirements document for a mobile banking app"

echo "Testing AG-UI event sequence compatibility..."
echo "Endpoint: $AG_UI_ENDPOINT"
echo ""

# Capture event sequence
curl -N -X POST "$AG_UI_ENDPOINT" \
  -H "Content-Type: application/json" \
  -d "{\"message\": \"$TEST_PROMPT\"}" \
  2>/dev/null | while read -r line; do
    echo "$line" | jq -r '.type' 2>/dev/null
  done | tee /tmp/ag_ui_events.log

echo ""
echo "=== COMPATIBILITY REPORT ==="

# Check universal events
echo -n "RUN_STARTED: "; grep -c "RUN_STARTED" /tmp/ag_ui_events.log || echo "MISSING -- CRITICAL"
echo -n "RUN_FINISHED: "; grep -c "RUN_FINISHED" /tmp/ag_ui_events.log || echo "MISSING -- CRITICAL"
echo -n "TEXT_MESSAGE_START: "; grep -c "TEXT_MESSAGE_START" /tmp/ag_ui_events.log || echo "MISSING -- CRITICAL"
echo -n "TEXT_MESSAGE_CONTENT: "; grep -c "TEXT_MESSAGE_CONTENT" /tmp/ag_ui_events.log || echo "MISSING -- CRITICAL"
echo -n "TEXT_MESSAGE_END: "; grep -c "TEXT_MESSAGE_END" /tmp/ag_ui_events.log || echo "MISSING -- CRITICAL"
echo -n "TOOL_CALL_START: "; grep -c "TOOL_CALL_START" /tmp/ag_ui_events.log || echo "none (OK if no tools called)"
echo -n "TOOL_CALL_END: "; grep -c "TOOL_CALL_END" /tmp/ag_ui_events.log || echo "none (OK if no tools called)"

# Check optional events
echo -n "STEP_STARTED: "; grep -c "STEP_STARTED" /tmp/ag_ui_events.log || echo "none (optional enhancement)"
echo -n "STEP_FINISHED: "; grep -c "STEP_FINISHED" /tmp/ag_ui_events.log || echo "none (optional enhancement)"

echo ""
echo "Adapter is COMPATIBLE if all CRITICAL events are present."
```

---

## 6. Migration Path -- Switching a Single Agent's Framework

### 6.1 The Six-Step Migration

One of the most powerful features of the AG-UI architecture is the ability to swap an agent's underlying framework with **zero frontend changes**. Here is the complete migration procedure:

#### Step 1: Replace Agent Code, Install New Framework

```bash
# ============================================================================
# STEP 1: Replace agent implementation
# ============================================================================

# --- BEFORE (Strands V4 -- NOT used in this demo) ---
# agents/agent_2_drafter/agent.py
from strands import Agent, tool
from bedrock_agentcore.runtime import BedrockAgentCoreApp  # V5 runtime

@tool
def analyze_gaps(transcript: str) -> str:
    ...

agent = Agent(name="drafter", tools=[analyze_gaps], ...)

app = BedrockAgentCoreApp()

@app.entrypoint
def handler(request):
    # Agent logic here
    return {"status": "success", "output": result}

# --- AFTER (LangGraph) ---
# agents/agent_2_drafter/agent.py
from langgraph.graph import StateGraph, END
from langgraph.prebuilt import ToolNode
from langgraph.ag_ui import emit_ag_ui_event  # Native AG-UI emission

def drafter_node(state: DrafterState):
    emit_ag_ui_event("TEXT_MESSAGE_START", {"messageId": state["msg_id"]})
    # ... agent logic ...
    emit_ag_ui_event("TEXT_MESSAGE_END", {"messageId": state["msg_id"]})
    return state

builder = StateGraph(DrafterState)
builder.add_node("drafter", drafter_node)
builder.add_node("tools", ToolNode([analyze_gaps]))
builder.add_edge("__start__", "drafter")
builder.add_conditional_edges("drafter", should_call_tools, {"tools": "tools", END: END})
builder.add_edge("tools", "drafter")

app = builder.compile_ag_ui_app(port=8080)  # Native AG-UI app export
```

#### Step 2: Rebuild Container (ARM64, Port 8080, /invocations)

```dockerfile
# ============================================================================
# STEP 2: Dockerfile changes -- only dependency layer changes
# ============================================================================

FROM --platform=linux/arm64 public.ecr.aws/amazonlinux/amazonlinux:2023

# --- BEFORE: Strands dependencies ---
# RUN pip install strands bedrock-agentcore  # V5 runtime SDK (NOT ag_ui_strands)

# --- AFTER: LangGraph dependencies ---
RUN pip install langgraph langgraph-ag-ui langchain-aws

# Application code (agent.py) is mounted at runtime
COPY agent.py /app/agent.py
WORKDIR /app
EXPOSE 8080
CMD ["python", "agent.py"]
```

```bash
# V5: S3 ZIP deploy — NO docker build, NO ECR push
# Developer code → ZIP → S3 → CodeBuild (ARM64) → AgentCore Runtime

agentcore configure -e agent.py --protocol AGUI \
  --filesystem-configurations file://fs-config.json
agentcore deploy --name drafter-langgraph
```

#### Step 3: Deploy via S3 ZIP (NO ECR, NO `launch-runtime`)

```bash
# ============================================================================
# STEP 3: Deploy via agentcore CLI (S3 ZIP — same as current workflow)
# ============================================================================

# V5: agentcore configure creates ZIP → uploads to S3 → CodeBuild builds ARM64
agentcore configure -e agent.py --protocol AGUI \
  --alias drafter-langgraph \
  --filesystem-configurations file://fs-config.json

agentcore deploy

# No ECR URI needed. No docker build. No aws bedrock-agentcore launch-runtime.
# AgentCore handles MicroVM provisioning automatically.
```

#### Step 4: Update Runtime ARN Environment Variable

```bash
# ============================================================================
# STEP 4: Point FastAPI to the new runtime
# ============================================================================

# .env file or AWS Systems Manager Parameter Store
# --- BEFORE ---
AGENT_2_RUNTIME_ARN="arn:aws:bedrock-agentcore:us-east-1:123456789:runtime/drafter-strands"

# --- AFTER ---
AGENT_2_RUNTIME_ARN="arn:aws:bedrock-agentcore:us-east-1:123456789:runtime/drafter-langgraph"

# Restart FastAPI (reads env var at startup)
docker compose restart fastapi
```

#### Step 5: Frontend Code -- Zero Changes

```typescript
// ============================================================================
// STEP 5: FRONTEND REQUIRES NO CHANGES
// The same useAgent hook, the same TranscriptCard, the same everything.
// ============================================================================

// This code is IDENTICAL before and after the framework migration.
// The AG-UI protocol makes the swap transparent.
const agent = useAgent({
  id: "drafter",
  name: "BRD Drafter",
});

return <DraftPreviewCard events={agent.events} status={agent.status} />;
```

#### Step 6: Test AG-UI Event Sequence

```bash
# ============================================================================
# STEP 6: Validate event sequence is acceptable to existing cards
# Run the compatibility test script from Section 5.6
# ============================================================================

./test_ag_ui_compatibility.sh \
  "https://agentcore.example.com/runtimes/drafter-langgraph/invocations"

# Expected output: All CRITICAL events present.
# If STEP_STARTED/STEP_FINISHED appear, verify cards handle them gracefully.
```

### 6.2 Migration Risk Matrix

| Risk | Likelihood | Mitigation |
|---|---|---|
| Missing `TEXT_MESSAGE_*` events | Low (all adapters emit these) | Run compatibility test script |
| Extra `STEP_*` events confuse cards | Medium (LangGraph emits many) | Defensive event handler (Section 5.5) |
| Different token chunking behavior | Medium | Cards should aggregate by `messageId`, not assume chunk count |
| `STATE_DELTA` format differences | Low (all use RFC 6902 JSON Patch) | Validate patch structure in test |
| Tool call naming conventions differ | Low | Adapter maps to AG-UI standard fields |
| Timing: slower/faster event emission | Medium | Frontend already handles variable streaming speed |

---

## 7. CrewAI / LangGraph Quickstart for Future Projects

### 7.1 LangGraph "Hello AG-UI" Skeleton (30 lines)

```python
#!/usr/bin/env python3
# ============================================================================
# LangGraph + Native AG-UI Emission -- Minimal Quickstart
# ============================================================================
# Dependencies:  uv venv .venv && source .venv/bin/activate && uv pip install langgraph langgraph-ag-ui langchain-aws
# Verified:      https://github.com/langchain-ai/langgraph
# ============================================================================

from typing import TypedDict
from langgraph.graph import StateGraph, END
from langgraph.ag_ui import emit_ag_ui_event, AgUiApp
from langchain_aws import ChatBedrock

# --- 1. Define state ---
class HelloState(TypedDict):
    message: str
    count: int

# --- 2. Define graph node with AG-UI emission ---
model = ChatBedrock(model_id="anthropic.claude-sonnet-4-6")

def hello_node(state: HelloState) -> HelloState:
    """A node that greets the user with AG-UI streaming."""
    msg_id = f"msg-{state['count']}"

    # Emit AG-UI text message events
    emit_ag_ui_event("TEXT_MESSAGE_START", {"messageId": msg_id})

    response = model.invoke([("human", state["message"])])
    content = response.content

    # Stream content (in real apps, use streaming model and chunk here)
    emit_ag_ui_event("TEXT_MESSAGE_CONTENT", {
        "messageId": msg_id,
        "content": content,
        "delta": content,
    })

    emit_ag_ui_event("TEXT_MESSAGE_END", {"messageId": msg_id})

    return {"message": content, "count": state["count"] + 1}

# --- 3. Build graph ---
builder = StateGraph(HelloState)
builder.add_node("hello", hello_node)
builder.add_edge("__start__", "hello")
builder.add_edge("hello", END)

# --- 4. Export as AG-UI app (SSE server) ---
app: AgUiApp = builder.compile_ag_ui_app(port=8080)

if __name__ == "__main__":
    app.run()
```

### 7.2 LangGraph with Tools and State Streaming

```python
#!/usr/bin/env python3
# ============================================================================
# LangGraph + AG-UI with Tools and STATE_DELTA -- Production Skeleton
# ============================================================================

from typing import TypedDict, Annotated
from langgraph.graph import StateGraph, END
from langgraph.prebuilt import ToolNode
from langgraph.ag_ui import emit_ag_ui_event, AgUiApp
from langchain_aws import ChatBedrock
from langchain_core.tools import tool
import operator

# --- 1. State with aggregation ---
class AgentState(TypedDict):
    messages: Annotated[list, operator.add]
    draft: str
    status: str

# --- 2. Tools ---
@tool
def search_docs(query: str) -> str:
    """Search internal documentation."""
    return f"Found 3 results for: {query}"

@tool
def save_draft(content: str) -> str:
    """Save a draft document."""
    return f"Draft saved: {len(content)} chars"

tools = [search_docs, save_draft]
tool_node = ToolNode(tools)
model = ChatBedrock(model_id="anthropic.claude-sonnet-4-6").bind_tools(tools)

# --- 3. Nodes with AG-UI emission ---
def agent_node(state: AgentState) -> AgentState:
    """Main agent node with streaming AG-UI events."""
    msg_id = f"agent-{len(state['messages'])}"

    emit_ag_ui_event("STEP_STARTED", {"stepId": msg_id, "type": "agent"})
    emit_ag_ui_event("TEXT_MESSAGE_START", {"messageId": msg_id})

    response = model.invoke(state["messages"])

    emit_ag_ui_event("TEXT_MESSAGE_CONTENT", {
        "messageId": msg_id,
        "content": response.content,
        "delta": response.content,
    })
    emit_ag_ui_event("TEXT_MESSAGE_END", {"messageId": msg_id})
    emit_ag_ui_event("STEP_FINISHED", {"stepId": msg_id})

    # Stream draft state as it builds up
    emit_ag_ui_event("STATE_DELTA", {
        "patch": [{"op": "add", "path": "/draft", "value": response.content}]
    })

    return {"messages": [response], "draft": response.content}

def should_continue(state: AgentState) -> str:
    """Route to tools if tool calls present, else end."""
    last_msg = state["messages"][-1]
    if last_msg.tool_calls:
        return "tools"
    return END

# --- 4. Build and export ---
builder = StateGraph(AgentState)
builder.add_node("agent", agent_node)
builder.add_node("tools", tool_node)
builder.add_edge("__start__", "agent")
builder.add_conditional_edges("agent", should_continue, {"tools": "tools", END: END})
builder.add_edge("tools", "agent")

app: AgUiApp = builder.compile_ag_ui_app(port=8080)

if __name__ == "__main__":
    app.run()
```

### 7.3 CrewAI "Hello AG-UI" Skeleton (30 lines)

```python
#!/usr/bin/env python3
# ============================================================================
# CrewAI + AGUIStream Wrapper -- Minimal Quickstart
# ============================================================================
# Dependencies:  uv venv .venv && source .venv/bin/activate && uv pip install crewai ag-ui-crewai langchain-aws
# Verified:      Check ag-ui-protocol/ag-ui repo for latest ag-ui-crewai
# ============================================================================

from crewai import Agent, Crew, Task
from ag_ui_crewai import AGUIStream  # The AG-UI adapter wrapper
from langchain_aws import ChatBedrock

# --- 1. Define agents ---
llm = ChatBedrock(model_id="anthropic.claude-sonnet-4-6")

researcher = Agent(
    role="Research Analyst",
    goal="Find key information about the given topic",
    backstory="Expert at research and analysis.",
    llm=llm,
    verbose=True,
)

writer = Agent(
    role="Technical Writer",
    goal="Write a concise summary",
    backstory="Expert at technical writing.",
    llm=llm,
    verbose=True,
)

# --- 2. Define tasks ---
research_task = Task(
    description="Research the topic: {topic}",
    expected_output="A list of 5 key points",
    agent=researcher,
)

write_task = Task(
    description="Write a summary based on the research",
    expected_output="A 200-word summary",
    agent=writer,
    context=[research_task],
)

# --- 3. Create crew ---
crew = Crew(
    agents=[researcher, writer],
    tasks=[research_task, write_task],
    verbose=True,
)

# --- 4. Wrap with AG-UI stream and export ---
agui_stream = AGUIStream(crew)
app = agui_stream.create_app(port=8080)

if __name__ == "__main__":
    # Kickoff with topic parameter
    app.run(kickoff_params={"topic": "AI in healthcare"})
```

### 7.4 CrewAI Multi-Agent with AG-UI State Streaming

```python
#!/usr/bin/env python3
# ============================================================================
# CrewAI + AGUIStream with STATE_DELTA -- Production Skeleton
# ============================================================================

from crewai import Agent, Crew, Task, Process
from ag_ui_crewai import AGUIStream
from langchain_aws import ChatBedrock
import asyncio

llm = ChatBedrock(model_id="anthropic.claude-sonnet-4-6")

# --- 1. Define specialized agents ---
gap_analyzer = Agent(
    role="Gap Analysis Specialist",
    goal="Identify missing requirements in transcripts",
    backstory="Expert at finding gaps in business requirements documents.",
    llm=llm,
)

requirements_writer = Agent(
    role="BRD Author",
    goal="Write comprehensive business requirements",
    backstory="Expert technical writer for enterprise BRDs.",
    llm=llm,
)

reviewer = Agent(
    role="Quality Reviewer",
    goal="Review BRD for completeness and accuracy",
    backstory="Senior analyst who reviews BRDs against quality standards.",
    llm=llm,
)

# --- 2. Define tasks with state streaming ---
analyze_task = Task(
    description="Analyze transcript for gaps: {transcript}",
    expected_output="JSON list of identified gaps with severity",
    agent=gap_analyzer,
)

draft_task = Task(
    description="Write BRD draft addressing all gaps",
    expected_output="Complete Markdown BRD document",
    agent=requirements_writer,
    context=[analyze_task],
)

review_task = Task(
    description="Review BRD for quality and completeness",
    expected_output="Quality score and review report",
    agent=reviewer,
    context=[draft_task],
)

# --- 3. Create crew with sequential process ---
crew = Crew(
    agents=[gap_analyzer, requirements_writer, reviewer],
    tasks=[analyze_task, draft_task, review_task],
    process=Process.sequential,
    verbose=True,
)

# --- 4. Wrap with AG-UI stream ---
# AGUIStream emits:
#   - RUN_STARTED / RUN_FINISHED for the entire crew
#   - STEP_STARTED / STEP_FINISHED for each task
#   - TEXT_MESSAGE_* for each agent's output
#   - STATE_DELTA with partial results after each task

agui_stream = AGUIStream(
    crew,
    state_config={
        "emit_intermediate": True,   # Emit STATE_DELTA after each task
        "include_task_outputs": True, # Include task results in STATE_DELTA
    }
)

app = agui_stream.create_app(port=8080)

if __name__ == "__main__":
    app.run(kickoff_params={"transcript": "We need a mobile banking app..."})
```

---

## 8. A2A vs AG-UI vs MCP -- Disambiguation

Three protocols are frequently confused. They serve **completely different purposes** and are **complementary**, not competing.

### 8.1 Protocol Purpose Matrix

| Protocol | Full Name | Direction | Purpose | Scope |
|---|---|---|---|---|
| **AG-UI** | Agent-User Interface | Agent --> Frontend (human) | Stream agent execution events to a user-facing UI | Agent talks to human via UI |
| **A2A** | Agent-to-Agent | Agent --> Agent | Structured communication between autonomous agents | Agent talks to agent |
| **MCP** | Model Context Protocol | Agent --> Tool/Resource | Connect agents to external tools, data sources, APIs | Agent talks to tool |

### 8.2 Visual Differentiation

```
+---------------------------------------------------------------+
|                    HUMAN USER (Frontend)                       |
+---------------+-----------------------------------------------+
                |  AG-UI Protocol (SSE / JSON)
                v
+---------------+-----------------------------------------------+
|   AGENT A (e.g., Strands Drafter)                            |
|   - Emits AG-UI events to frontend                            |
|   - Uses MCP to call tools                                    |
|   - Uses A2A to communicate with Agent B                      |
+-------+---------------+-----------------------+---------------+
        |               |                       |
        | AG-UI         | MCP                   | A2A
        | (to UI)       | (to tools)            | (to Agent B)
        v               v                       v
+-------+-------+  +----+----------------+  +--+--------------+
|  Next.js      |  |  External Tools     |  |  AGENT B       |
|  + CopilotKit |  |  - File systems     |  |  (e.g., Reviewer|
|               |  |  - Databases        |  |   on LangGraph) |
|  Renders:     |  |  - APIs             |  |                |
|  - Chat       |  |  - Search           |  |  Receives A2A  |
|  - Tool cards |  |                     |  |  task requests |
|  - State      |  |  stdio / HTTP       |  |                |
+---------------+  +---------------------+  +----------------+
```

### 8.3 AG-UI: Agent-to-User

```json
// AG-UI event: Agent tells the user what it's doing
{
  "type": "TEXT_MESSAGE_CONTENT",
  "payload": {
    "messageId": "msg-001",
    "content": "I'm analyzing the audio transcript now...",
    "delta": "analyzing..."
  }
}
```

**Key URLs:**
- Protocol docs: https://docs.ag-ui.com/
- GitHub: https://github.com/ag-ui-protocol/ag-ui
- Our demo: Strands via `BedrockAgentCoreApp` streaming to CopilotKit `useAgent`

### 8.4 A2A: Agent-to-Agent

```json
// A2A message: Agent A asks Agent B to perform a task
{
  "jsonrpc": "2.0",
  "method": "tasks/send",
  "params": {
    "id": "task-review-001",
    "sessionId": "sess-abc-123",
    "message": {
      "role": "user",
      "parts": [{"type": "text", "text": "Please review this BRD draft for completeness."}]
    }
  }
}
```

A2A handles **delegation**: one agent asks another to do work and gets back a structured result. It uses JSON-RPC 2.0 over HTTP or Server-Sent Events. A2A defines task lifecycle (submitted, working, input-required, completed, cancelled), artifact delivery, and push notifications.

**Key URLs:**
- Protocol spec: https://a2a-protocol.org/latest/
- Use case: Agent A (Drafter) asks Agent B (Reviewer) to review a document

### 8.5 MCP: Model Context Protocol

```json
// MCP request: Agent asks a tool for data
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "read_file",
    "arguments": {
      "path": "/docs/requirements.md"
    }
  }
}
```

MCP handles **tool access**: the agent connects to external capabilities (file systems, databases, APIs) through a standardized interface. MCP servers expose resources (readable data), tools (callable functions), and prompts (reusable templates).

**Key URLs:**
- Protocol spec: https://modelcontextprotocol.io/
- Use case: Agent reads from S3, queries a database, calls an external API

### 8.6 When to Use Which

| Scenario | Protocol to Use | Why |
|---|---|---|
| Stream agent progress to a web UI | **AG-UI** | Native SSE streaming, UI-optimized events |
| Agent asks another agent to do work | **A2A** | Structured task delegation, lifecycle management |
| Agent reads a file or calls an API | **MCP** | Standardized tool interface, security boundaries |
| Frontend needs real-time updates | **AG-UI** | Designed for human-visible streaming |
| Multi-agent workflow needs orchestration | **A2A + Temporal** | A2A for agent comms, Temporal for durability |
| Agent needs database access | **MCP** | Resource abstraction, access control |
| User asks agent a question in chat | **AG-UI** | Bidirectional chat via TEXT_MESSAGE_* |

### 8.7 Our Demo's Protocol Usage

| Protocol | Used In Our Demo | How |
|---|---|---|
| **AG-UI** | **Yes -- primary** | Strands agents emit AG-UI events via `BedrockAgentCoreApp`, consumed by CopilotKit `useAgent` |
| **A2A** | Stub only | A2A envelope classes defined in `common/a2a_envelope.py` but not actively used. Available for future multi-agent delegation. |
| **MCP** | Indirectly | boto3 calls to Transcribe, Bedrock, S3 are MCP-equivalent tool calls, but via AWS SDK directly rather than MCP protocol. |

---

## 9. Production Considerations

### 9.1 Dependency Drift

Each framework has its own dependency tree. When running multiple frameworks in the same environment (Pattern B), conflicts are inevitable.

| Framework | Key Dependencies | Known Conflicts |
|---|---|---|
| Strands | `strands`, `bedrock-agentcore`, `boto3>=1.34` | None major |
| LangGraph | `langgraph>=0.2`, `langchain>=0.3`, `langchain-aws` | LangChain version pinning is strict; multiple LC versions cannot coexist |
| CrewAI | `crewai>=0.80`, `pydantic>=2.0` | CrewAI pins specific Pydantic versions; conflicts with older FastAPI |
| Microsoft | `agent-framework>=1.0` | Relatively self-contained |
| Mastra | `mastra` (JS/TS runtime) | **JavaScript/TypeScript only** -- cannot mix in Python container |

**Mitigation:** Use **Pattern A (separate containers)** for framework mixing. Each container has its own virtual environment and dependency tree. This is the primary reason Pattern A is recommended.

```dockerfile
# Anti-pattern: mixing frameworks in one environment
# pip install strands langgraph bedrock-agentcore  # V5 runtime

# Correct: separate containers
# Container 1: pip install strands bedrock-agentcore  # V5 runtime
# Container 2: pip install langgraph langgraph-ag-ui langchain-aws
# Container 3: pip install crewai ag-ui-crewai
```

### 9.2 Python Version Differences

| Framework | Minimum Python | Recommended | Notes |
|---|---|---|---|
| Strands | 3.9 | 3.11 | Tested on 3.11 for this demo |
| LangGraph | 3.9 | 3.11 | Python 3.12 support varies by release |
| CrewAI | 3.10 | 3.11 | Some features require 3.10+ match statements |
| Microsoft Agent Framework | 3.10 | 3.11 | Preview SDK, check latest requirements |
| Mastra | N/A (JS/TS) | Node 20+ | **Not Python** -- requires separate JS runtime |

**Mitigation:** Standardize on **Python 3.11** across all containers. Use Amazon Linux 2023 base image (ships with Python 3.11).

```dockerfile
FROM --platform=linux/arm64 public.ecr.aws/amazonlinux/amazonlinux:2023
RUN dnf install -y python3.11 python3.11-pip
RUN python3.11 -m pip install --upgrade pip
```

### 9.3 Observability Format Differences

Each framework has different default logging and tracing behavior. Our 8-block telemetry payload normalizes across them.

| Framework | Default Logging | 8-Block Payload | CloudWatch EMF |
|---|---|---|---|
| Strands | SDK default (configurable) | We emit manually | We emit manually via boto3 |
| LangGraph | Structured JSON via LangSmith | Must add manually | Must add manually |
| CrewAI | Verbose text logging | Must add manually | Must add manually |

**Best practice:** Keep the 8-block payload emitter (`common/payload_builder.py`) as a **shared library** used by all frameworks. When migrating, the payload builder code does not change -- only the framework invocation wrapper changes.

```python
# ============================================================================
# Framework-agnostic payload emission -- used identically across all frameworks
# ============================================================================
from common.payload_builder import build_telemetry_payload

# Strands, LangGraph, CrewAI -- all use the same function
payload = build_telemetry_payload(
    trace_id=trace_id,
    workflow_id=workflow_id,
    agent_run_id=agent_run_id,
    status="COMPLETE",
    resources={"input_tokens": 1500, "output_tokens": 800},
    timing={"duration_ms": 4500},
    artifacts=[{"type": "draft_md", "hash": sha256(draft)}],
    quality={"completeness_score": 0.92},
    tool_calls=[{"tool": "analyze_gaps", "duration_ms": 1200}],
    risk={"pii_detected": False},
)
```

### 9.4 Cost Profiles (Token Verbosity Varies)

Different frameworks send different amounts of context/tokens to the LLM, affecting Bedrock costs.

| Framework | Token Efficiency | Notes | Cost Impact |
|---|---|---|---|
| Strands | High | Minimal system prompt overhead | Baseline |
| LangGraph | Medium | Checkpoint serialization adds context | +10-20% vs Strands |
| CrewAI | Lower | Multi-agent delegation adds rounds | +30-50% vs single agent |
| CrewAI (hierarchical) | Lowest | Manager agent adds full context per delegation | +50-100% |

**Mitigation:** Token costs are typically small compared to infrastructure. Monitor with the 8-block payload's `Financial` block. Set CloudWatch alarms on `input_tokens + output_tokens` per workflow.

### 9.5 Security Review Per Framework

| Dimension | Strands | LangGraph | CrewAI |
|---|---|---|---|
| Code execution risk | Low (tool definitions required) | Medium (graph can execute arbitrary Python) | Medium (agents can use delegated tools) |
| Prompt injection defense | Input validation via tool schemas | Depends on implementation | Depends on guardrails config |
| Secret handling | IAM roles (no env keys) | Must configure explicitly | Must configure explicitly |
| Sandboxing | AgentCore microVM isolation | Container-level only | Container-level only |
| Audit trail | 8-block payload + EMF | Must implement manually | Must implement manually |

**Recommendation:** All frameworks should run in **AgentCore microVMs** or equivalent sandboxed containers. Never run multi-agent frameworks with network access in an unsandboxed process.

### 9.6 Deployment Checklist for Multi-Framework

```markdown
- [ ] Each framework in its own container (Pattern A)
- [ ] Python 3.11 standardized across all containers
- [ ] ARM64 builds for AgentCore Runtime compatibility
- [ ] Port 8080 + /invocations endpoint on each container
- [ ] IAM role per runtime (least privilege)
- [ ] 8-block telemetry payload emitted by all agents
- [ ] W3C traceparent propagation verified end-to-end
- [ ] AG-UI event compatibility test passed (Section 5.6)
- [ ] CloudWatch EMF metrics flowing from all runtimes
- [ ] Health check (/ping) responding on all containers
- [ ] Temporal activity retries configured per framework timeout
- [ ] Frontend defensive event handler in place (Section 5.5)
- [ ] Security review completed for each framework's tool execution model
```

---

## 10. Tech Reference

### 10.1 AG-UI Protocol

| Resource | URL |
|---|---|
| AG-UI Protocol Documentation | https://docs.ag-ui.com/ |
| AG-UI Protocol GitHub | https://github.com/ag-ui-protocol/ag-ui |
| Event Types Reference (17 types) | https://www.copilotkit.ai/blog/master-the-17-ag-ui-event-types-for-building-agents-the-right-way |
| CopilotKit AG-UI Backend Docs | https://docs.copilotkit.ai/backend/ag-ui |

### 10.2 Framework Adapters

| Framework | Adapter Package | Documentation |
|---|---|---|
| AWS Strands | `bedrock-agentcore` | https://docs.aws.amazon.com/bedrock/latest/userguide/agentcore-runtime-sdk.html |
| LangGraph | Native (`langgraph.ag_ui`) | https://langchain-ai.github.io/langgraph/ |
| CrewAI | `ag-ui-crewai` | Check https://github.com/ag-ui-protocol/ag-ui for releases |
| Microsoft Agent Framework | `agent_framework.ag_ui` | Microsoft Agent Framework preview docs |
| Mastra | Built-in | https://mastra.ai/docs |
| OpenAI Agents | `ag-ui-sdk` compatible | OpenAI Agents SDK documentation |

### 10.3 Related Protocols

| Protocol | URL |
|---|---|
| A2A Protocol (Agent-to-Agent) | https://a2a-protocol.org/latest/ |
| MCP (Model Context Protocol) | https://modelcontextprotocol.io/ |

### 10.4 AWS Services

| Service | URL | Purpose |
|---|---|---|
| AWS Bedrock AgentCore | https://docs.aws.amazon.com/bedrock-agentcore/ | Runtime for containerized agents |
| Amazon Bedrock | https://docs.aws.amazon.com/bedrock/latest/userguide/ | Foundation model access |
| Amazon Transcribe | https://docs.aws.amazon.com/transcribe/ | Audio transcription |
| Amazon CloudWatch | https://docs.aws.amazon.com/cloudwatch/ | Observability and EMF metrics |

### 10.5 Frontend

| Technology | URL | Purpose |
|---|---|---|
| CopilotKit | https://copilotkit.ai | React SDK for AG-UI streaming |
| Next.js | https://nextjs.org | React framework (App Router) |
| Temporal TypeScript SDK | https://docs.temporal.io/dev-guide/typescript | Workflow client in frontend (optional) |

### 10.6 Orchestration

| Technology | URL | Purpose |
|---|---|---|
| Temporal | https://temporal.io | Durable workflow orchestration |
| Temporal Python SDK | https://docs.temporal.io/dev-guide/python | Workflow and activity definitions |

---

## Appendix A: Quick-Reference Decision Tree

```
Starting a new project with AG-UI?
|
+-- Single framework for all agents?
|   +-- Yes: Use that framework's adapter (Table in Section 2)
|   +-- No (mixing frameworks): Continue...
|
+-- Do agents need tight state sharing?
|   +-- Yes: Consider Pattern B (single container, orchestrator)
|   +-- No: Continue...
|
+-- Is latency critical (< 100ms between agents)?
|   +-- Yes: Pattern B (single container) or Pattern A with local networking
|   +-- No: Continue...
|
+-- Are agents long-running with human-in-the-loop?
|   +-- Yes: Pattern C (Temporal orchestration) -- what our demo uses
|   +-- No: Continue...
|
+-- RECOMMENDED: Pattern A (separate containers per agent)
   +-- Each agent independently deployable
   +-- Mix frameworks freely
   +-- Frontend switches runtime-id by phase
   +-- Maximum maintainability
```

---

## Appendix B: Glossary

| Term | Definition |
|---|---|
| **AG-UI** | Agent-User Interface protocol. Open standard for streaming agent events to frontend UIs. |
| **A2A** | Agent-to-Agent protocol. JSON-RPC-based communication between autonomous agents. |
| **MCP** | Model Context Protocol. Standard for connecting agents to external tools and resources. |
| **Adapter** | Framework-specific library that translates framework events to AG-UI events. |
| **SSE** | Server-Sent Events. HTTP-based push technology used by AG-UI for streaming. |
| **Pattern A** | Separate containers per agent (recommended). |
| **Pattern B** | Single container with internal orchestrator. |
| **Pattern C** | Temporal-orchestrated cross-framework workflow. |
| **8-block payload** | Standardized telemetry format (Status, Resources, Timing, Financial, Artifacts, Quality, Tool Calls, Risk). |
| **EMF** | Embedded Metric Format. CloudWatch structured logging format for metrics. |

---

*Document generated for AgentCore demo test 1 Automation Demo. Cross-reference with Deliverable 5 (CopilotKit AG-UI Guide) for frontend integration details and Deliverable 4 (Temporal Operations Guide) for orchestration patterns.*
