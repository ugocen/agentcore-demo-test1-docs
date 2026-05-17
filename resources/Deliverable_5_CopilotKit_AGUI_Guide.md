# Deliverable 5: CopilotKit + AG-UI Integration & Reusability Guide

## Comprehensive Integration Guide for Next.js 14+ App Router + CopilotKit >=1.50 + AG-UI Protocol + AWS Bedrock AgentCore

> **V5 (2026-05-12):** AG-UI and A2A are clearly separated — AG-UI handles user-facing streaming (this guide), A2A handles agent-to-agent communication via Temporal signals (backend-only, invisible to frontend). Mock auth (`demo-user-001`) — NO Cognito, NO OAuth.

---

## Table of Contents

1. [CopilotKit Architecture in this Demo](#1-copilotkit-architecture-in-this-demo)
2. [AG-UI Protocol Deep Dive](#2-ag-ui-protocol-deep-dive)
3. [Custom HttpAgent for AgentCore](#3-custom-httpagent-for-agentcore)
4. [Coordinating AG-UI Events with Temporal Signals](#4-coordinating-ag-ui-events-with-temporal-signals)
5. [Generative UI Cards Library](#5-generative-ui-cards-library)
6. [Two SSE Streams Pattern](#6-two-sse-streams-pattern)
7. [Threads & Sidebar History](#7-threads--sidebar-history)
8. [Page Refresh & State Restoration](#8-page-refresh--state-restoration)
9. [CopilotKit + Tailwind Conflict Workarounds](#9-copilotkit--tailwind-conflict-workarounds)
10. [Reusability Patterns for Future Projects](#10-reusability-patterns-for-future-projects)
11. [Tech Reference](#11-tech-reference)

---

## 1. CopilotKit Architecture in this Demo

### Overview

CopilotKit serves as the primary React SDK for integrating AG-UI protocol streaming into our Next.js frontend. It provides hooks, providers, and UI components that consume AG-UI event streams and render them as interactive chat and canvas experiences.

### Key Components and Their Roles

| Component | Source | Role in Demo |
|-----------|--------|-------------|
| `CopilotKit` provider | `@copilotkit/react-core` | Root provider wrapping the app, holds agent runtime config |
| `useAgent` | `@copilotkit/react-core` | **Primary hook** — connects to AG-UI endpoint, streams events, manages run lifecycle (v1.50+) |
| `useCoAgent` | `@copilotkit/react-core` | Legacy hook for LangGraph-specific agents; NOT used in our AgentCore setup |
| `useCopilotChat` | `@copilotkit/react-core` | Manages chat messages, sendMessage, status |
| `useFrontendTool` | `@copilotkit/react-core` | Registers frontend-callable tools for Static Generative UI (Controlled) pattern |
| `renderAndWaitForResponse` | `@copilotkit/react-core` | Blocks agent execution until user interacts with rendered UI |
| `CopilotSidebar` | `@copilotkit/react-ui` | Pre-built sidebar with chat + canvas (optional) |
| `CopilotChat` | `@copilotkit/react-ui` | Chat bubble component |

### Data Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              NEXT.JS FRONTEND                           │
│                                                                         │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────────────────┐  │
│  │  CopilotKit  │    │   useAgent   │    │    useFrontendTool       │  │
│  │  Provider    │◄───│   Hook       │◄───│    + renderAndWaitFor    │  │
│  │              │    │              │    │    Response              │  │
│  └──────┬───────┘    └──────┬───────┘    └─────────────┬────────────┘  │
│         │                   │                          │               │
│         ▼                   ▼                          ▼               │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                     GENERATIVE UI CARDS                          │   │
│  │  (TranscriptCard, DraftPreviewCard, ReviewReportCard, etc.)      │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│         │                                                              │
│         ▼                                                              │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                     CANVAS STATE MACHINE                         │   │
│  │  (Merges AG-UI stream + Workflow Stream into unified state)      │   │
│  └─────────────────────────────────────────────────────────────────┘   │
└─────────┬─────────────────────────────────────────────────────────────┘
          │
          │  HTTPS / SSE
          │
┌─────────▼─────────────────────────────────────────────────────────────┐
│                         AWS BEDROCK AGENTCORE                          │
│                                                                        │
│  ┌─────────────────────┐      ┌───────────────────────────────┐       │
│  │  AG-UI Endpoint     │      │   Runtime: Strands Agent      │       │
│  │  (Server-Sent       │◄─────│   (microVM per invocation)    │       │
│  │   Events)           │      │                               │       │
│  └─────────────────────┘      └───────────────────────────────┘       │
│                                                                        │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │  Session Sharing: X-Amzn-Bedrock-AgentCore-Runtime-Session-Id  │  │
│  │  = workflow_id (shares microVM across multi-agent workflow)     │  │
│  └─────────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────────┘
          │
          │  Temporal gRPC
          ▼
┌────────────────────────────────────────────────────────────────────────┐
│                      TEMPORAL ORCHESTRATION                            │
│                                                                        │
│  ┌──────────────────┐        ┌──────────────────────────┐             │
│  │  Workflow Stream │        │  Temporal Signals         │             │
│  │  (SSE from       │        │  (HITL responses)         │             │
│  │   FastAPI)       │        │                           │             │
│  └──────────────────┘        └──────────────────────────┘             │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### Why `useAgent` Over `useCoAgent`

As of CopilotKit v1.50+, `useAgent` is the **AG-UI-native connection hook**. It replaces `useCoAgent` for all non-LangGraph agent runtimes:

- `useAgent` speaks raw AG-UI protocol — it understands all 16+ event types natively
- `useCoAgent` is LangGraph-specific — it expects LangGraph's state format and checkpointing
- Our AgentCore runtime emits pure AG-UI events, so `useAgent` is the correct match
- `useAgent` returns: `{ agent, setState, run, stop, status, messages, ... }`

```typescript
// WRONG — useCoAgent expects LangGraph-specific configuration
const agent = useCoAgent({
  name: "strands-agent",        // LangGraph agent name
  // ... LangGraph-specific options
});

// CORRECT — useAgent speaks AG-UI protocol natively
const agent = useAgent({
  id: "my-strands-agent",        // Agent identifier
  // AG-UI compatible configuration
});
```

### Provider Setup in `layout.tsx`

```tsx
// ============================================================================
// app/layout.tsx — CopilotKit Provider Configuration
// ============================================================================
// The CopilotKit provider MUST wrap all components that use CopilotKit hooks.
// We place it at the root layout level so the entire app has access.
// ============================================================================

import { CopilotKit } from "@copilotkit/react-core";
import "@copilotkit/react-ui/styles.css";  // Default styles (see Section 9 for Tailwind conflicts)

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <CopilotKit
          // The runtimeUrl points to our AG-UI-compatible endpoint.
          // In our architecture, this is handled by the custom HttpAgent
          // (see Section 3), NOT by a direct CopilotKit runtime URL.
          runtimeUrl={process.env.NEXT_PUBLIC_AGENTCORE_AGUI_URL}
          // Agent-specific configuration
          agent="content-generation-agent"
          // Enable generative UI for frontend tools
          showDevConsole={process.env.NODE_ENV === "development"}
        >
          {children}
        </CopilotKit>
      </body>
    </html>
  );
}
```

### useAgent Hook Full API

```typescript
// ============================================================================
// useAgent Hook — Primary interface for AG-UI streaming
// ============================================================================

import { useAgent } from "@copilotkit/react-core";

function MyAgentComponent() {
  const {
    // Core interaction
    run,           // (message: string, props?: RunProps) => void — starts agent run
    stop,          // () => void — aborts current run
    reset,         // () => void — clears agent state

    // Status
    status,        // "idle" | "thinking" | "working" — derived from AG-UI lifecycle events

    // Messages (chat history)
    messages,      // Message[] — accumulated chat messages from TEXT_MESSAGE_* events

    // Raw AG-UI events (for advanced use)
    events,        // AgentEvent[] — raw stream of all AG-UI events

    // State management
    setState,      // (state: any) => void — updates agent state (emits STATE_DELTA)
    state,         // any — current agent state

    // HITL
    respond,       // (response: string) => void — respond to HITL prompt
    renderWaitForResponse, // boolean — true when agent is waiting for HITL

    // Error handling
    error,         // Error | null — any stream error
  } = useAgent({
    id: "content-generation-agent",     // Unique agent identifier
    name: "Content Generation Agent",    // Display name
  });

  // Start a run
  const handleStart = () => {
    run("Generate a blog post about AI", {
      // Optional: pass initial state
      initialState: { topic: "AI", tone: "professional" },
      // Optional: frontend tools available for this run
      frontendTools: ["approveContent", "requestChanges"],
    });
  };

  return (
    <div>
      <button onClick={handleStart} disabled={status !== "idle"}>
        {status === "idle" ? "Start" : status === "thinking" ? "Thinking..." : "Working..."}
      </button>
      <button onClick={stop} disabled={status === "idle"}>Stop</button>
    </div>
  );
}
```

---

## 2. AG-UI Protocol Deep Dive

### What is AG-UI?

AG-UI (Agent-User Interface) is an open protocol for streaming events between AI agents and frontend applications. It defines a standardized set of event types that any agent runtime can emit and any frontend can consume. CopilotKit v1.50+ is an AG-UI-native consumer.

**Key References:**
- Protocol docs: https://docs.ag-ui.com/
- Event types reference: https://www.copilotkit.ai/blog/master-the-17-ag-ui-event-types-for-building-agents-the-right-way
- Backend integration: https://docs.copilotkit.ai/backend/ag-ui

### Event Categories

#### 2.1 Lifecycle Events

These events bracket the entire agent execution (RUN) and individual steps within it (STEP).

| Event Type | Direction | Payload | Purpose |
|-----------|-----------|---------|---------|
| `RUN_STARTED` | Agent → Frontend | `{ runId: string, threadId?: string }` | Signals the beginning of a complete agent run |
| `RUN_FINISHED` | Agent → Frontend | `{ runId: string }` | Signals successful completion of the run |
| `RUN_ERROR` | Agent → Frontend | `{ runId: string, error: string }` | Signals the run failed with an error |
| `STEP_STARTED` | Agent → Frontend | `{ stepId: string, type: string }` | Individual step (tool call, LLM call) began |
| `STEP_FINISHED` | Agent → Frontend | `{ stepId: string }` | Individual step completed |

```typescript
// ============================================================================
// Lifecycle Event Handler — useAgent status derives from these events
// ============================================================================

// RUN_STARTED  → status transitions from "idle" → "thinking"
// STEP_STARTED → status may transition to "working" (if tool call)
// STEP_FINISHED → status may return to "thinking"
// RUN_FINISHED → status transitions to "idle"
// RUN_ERROR    → status transitions to "idle" + error is set

// Example handler:
useEffect(() => {
  if (!agent.events) return;

  const unsubscribe = agent.events.subscribe((event: AgentEvent) => {
    switch (event.type) {
      case "RUN_STARTED":
        console.log(`[AG-UI] Run ${event.runId} started`);
        setWorkflowId(event.runId);  // Capture for session management
        break;
      case "RUN_FINISHED":
        console.log(`[AG-UI] Run ${event.runId} finished`);
        // Trigger any post-run cleanup
        break;
      case "RUN_ERROR":
        console.error(`[AG-UI] Run ${event.runId} error:`, event.error);
        toast.error(`Agent error: ${event.error}`);
        break;
      case "STEP_STARTED":
        console.log(`[AG-UI] Step ${event.stepId} (${event.type}) started`);
        break;
      case "STEP_FINISHED":
        console.log(`[AG-UI] Step ${event.stepId} finished`);
        break;
    }
  });

  return () => unsubscribe();
}, [agent.events]);
```

#### 2.2 Text Message Events

Text messages are streamed in three parts: START → CONTENT chunks → END. This enables real-time token streaming to the UI.

| Event Type | Direction | Payload | Purpose |
|-----------|-----------|---------|---------|
| `TEXT_MESSAGE_START` | Agent → Frontend | `{ messageId: string }` | New message begins |
| `TEXT_MESSAGE_CONTENT` | Agent → Frontend | `{ messageId: string, content: string, delta?: string }` | Token chunk (can be many) |
| `TEXT_MESSAGE_END` | Agent → Frontend | `{ messageId: string }` | Message complete |

```typescript
// ============================================================================
// Text Message Streaming — CopilotKit assembles these automatically
// ============================================================================
// You typically don't handle these manually — useAgent.messages already
// assembles them. But for custom rendering, you may want raw events.

// Example: Custom streaming text renderer
function StreamingText({ events }: { events: AgentEvent[] }) {
  const [textChunks, setTextChunks] = useState<Map<string, string>>(new Map());

  useEffect(() => {
    events.forEach((event) => {
      if (event.type === "TEXT_MESSAGE_CONTENT") {
        setTextChunks((prev) => {
          const next = new Map(prev);
          const existing = next.get(event.messageId) || "";
          next.set(event.messageId, existing + (event.delta || event.content));
          return next;
        });
      }
    });
  }, [events]);

  return (
    <div>
      {Array.from(textChunks.entries()).map(([id, text]) => (
        <p key={id}>{text}</p>
      ))}
    </div>
  );
}
```

#### 2.3 Tool Call Events

Tool calls are bracketed by START/END events. The tool call itself contains the function name and arguments.

| Event Type | Direction | Payload | Purpose |
|-----------|-----------|---------|---------|
| `TOOL_CALL_START` | Agent → Frontend | `{ callId: string, name: string, arguments: Record<string, any> }` | Agent is calling a tool |
| `TOOL_CALL_END` | Agent → Frontend | `{ callId: string, result?: any }` | Tool call completed, may include result |

```typescript
// ============================================================================
// Tool Call Event Handler — Detect when specific tools are invoked
// ============================================================================

useEffect(() => {
  if (!agent.events) return;

  agent.events.subscribe((event: AgentEvent) => {
    if (event.type === "TOOL_CALL_START") {
      console.log(`[AG-UI] Tool call: ${event.name}(${JSON.stringify(event.arguments)})`);

      // Track tool calls for analytics/debugging
      analytics.track("agent_tool_call", {
        tool_name: event.name,
        run_id: currentRunId,
      });
    }

    if (event.type === "TOOL_CALL_END") {
      console.log(`[AG-UI] Tool call ${event.callId} completed`);
    }
  });
}, [agent.events]);
```

#### 2.4 State Management Events

These are the most powerful events for building reactive UIs. They allow the agent to push structured state updates that the frontend can render as components.

| Event Type | Direction | Payload | Purpose |
|-----------|-----------|---------|---------|
| `STATE_SNAPSHOT` | Agent → Frontend | `{ state: any }` | Full state replacement |
| `STATE_DELTA` | Agent → Frontend | `{ patch: JSONPatch[] }` | RFC 6902 JSON Patch — partial update |

**JSON Patch format (RFC 6902):**
```typescript
// STATE_DELTA uses RFC 6902 JSON Patch for efficient incremental updates
// This is the same patch format used by React DevTools and many state libraries

type JSONPatchOp =
  | { op: "add"; path: string; value: any }      // Add new value at path
  | { op: "remove"; path: string }                // Remove value at path
  | { op: "replace"; path: string; value: any }   // Replace value at path
  | { op: "move"; path: string; from: string }    // Move value from → to
  | { op: "copy"; path: string; from: string }    // Copy value from → to
  | { op: "test"; path: string; value: any };     // Test value (for conditional patches)

// Example: Agent emits state delta as it builds a blog post
const exampleDelta = {
  type: "STATE_DELTA",
  patch: [
    { op: "replace", path: "/status", value: "generating_outline" },
    { op: "add", path: "/sections", value: [
      { title: "Introduction", content: "...", approved: false }
    ]},
  ],
};

// Frontend applies patches using a library like `fast-json-patch`
import * as jsonpatch from "fast-json-patch";

function applyStateDelta(currentState: any, patch: JSONPatch[]) {
  // fast-json-patch applies RFC 6902 patches in-place
  const result = JSON.parse(JSON.stringify(currentState)); // Deep clone
  jsonpatch.applyPatch(result, patch);
  return result;
}
```

#### 2.5 Special/Meta Events

| Event Type | Direction | Payload | Purpose |
|-----------|-----------|---------|---------|
| `CUSTOM` | Bidirectional | `{ name: string, payload: any }` | Application-specific custom events |
| `RAW` | Agent → Frontend | `{ data: any }` | Unstructured/raw data passthrough |
| `ERROR` | Agent → Frontend | `{ message: string, code?: string }` | Stream-level error (not run error) |

```typescript
// ============================================================================
// CUSTOM events are the key to our hybrid architecture
// We use them to embed workflow metadata in the AG-UI stream
// ============================================================================

// Agent emits CUSTOM event with workflow_id for cross-reference
const customEvent = {
  type: "CUSTOM",
  name: "workflow_metadata",
  payload: {
    workflow_id: "wf_abc123",
    temporal_workflow_id: "temporal-wf-456",
    agent_name: "content-generator",
  },
};

// Frontend captures workflow_id from CUSTOM event to subscribe
// to the second SSE stream (Workflow Stream from FastAPI)
useEffect(() => {
  agent.events?.subscribe((event: AgentEvent) => {
    if (event.type === "CUSTOM" && event.name === "workflow_metadata") {
      const { workflow_id } = event.payload;
      // Now we know the workflow_id → subscribe to Workflow Stream
      workflowStream.connect(workflow_id);
    }
  });
}, [agent.events]);
```

#### 2.6 HITL (Human-in-the-Loop) Events

| Event Type | Direction | Payload | Purpose |
|-----------|-----------|---------|---------|
| `INTERRUPT` | Agent → Frontend | `{ prompt: string, metadata?: any }` | Agent needs human input before continuing |
| `RESPONSE` | Frontend → Agent | `{ response: string }` | Human's response to INTERRUPT |

```typescript
// ============================================================================
// HITL Flow — Agent interrupts, frontend renders UI, user responds
// ============================================================================

// 1. Agent emits INTERRUPT when it needs approval/clarification
const interruptEvent = {
  type: "INTERRUPT",
  prompt: "Please review and approve the generated content.",
  metadata: {
    content_id: "section_1",
    requires_approval: true,
  },
};

// 2. Frontend detects interrupt and renders HITL UI
const { renderWaitForResponse, respond } = useAgent({ ... });

// In JSX:
{renderWaitForResponse && (
  <ApprovalCard
    content={currentDraft}
    onApprove={() => respond("APPROVED")}
    onReject={(feedback) => respond(`REJECTED: ${feedback}`)}
  />
)}

// 3. respond() sends RESPONSE event back to agent via AG-UI stream
//    AND simultaneously sends Temporal Signal (see Section 4)
```

### Complete Event Type Reference Table

```typescript
// ============================================================================
// AG-UI Event Type Discriminated Union — Full TypeScript Definition
// Copy this into your project for type-safe event handling
// ============================================================================

export type AgentEvent =
  // === Lifecycle Events ===
  | { type: "RUN_STARTED"; runId: string; threadId?: string; timestamp: number }
  | { type: "RUN_FINISHED"; runId: string; timestamp: number }
  | { type: "RUN_ERROR"; runId: string; error: string; timestamp: number }
  | { type: "STEP_STARTED"; stepId: string; stepType: string; timestamp: number }
  | { type: "STEP_FINISHED"; stepId: string; timestamp: number }

  // === Text Message Events ===
  | { type: "TEXT_MESSAGE_START"; messageId: string; timestamp: number }
  | { type: "TEXT_MESSAGE_CONTENT"; messageId: string; content?: string; delta?: string; timestamp: number }
  | { type: "TEXT_MESSAGE_END"; messageId: string; timestamp: number }

  // === Tool Call Events ===
  | { type: "TOOL_CALL_START"; callId: string; name: string; arguments: Record<string, any>; timestamp: number }
  | { type: "TOOL_CALL_END"; callId: string; result?: any; timestamp: number }

  // === State Management Events ===
  | { type: "STATE_SNAPSHOT"; state: any; timestamp: number }
  | { type: "STATE_DELTA"; patch: JSONPatchOp[]; timestamp: number }

  // === HITL Events ===
  | { type: "INTERRUPT"; prompt: string; metadata?: any; timestamp: number }
  | { type: "RESPONSE"; response: string; timestamp: number }

  // === Special/Meta Events ===
  | { type: "CUSTOM"; name: string; payload: any; timestamp: number }
  | { type: "RAW"; data: any; timestamp: number }
  | { type: "ERROR"; message: string; code?: string; timestamp: number };

// JSON Patch Operation (RFC 6902)
export type JSONPatchOp =
  | { op: "add"; path: string; value: any }
  | { op: "remove"; path: string }
  | { op: "replace"; path: string; value: any }
  | { op: "move"; path: string; from: string }
  | { op: "copy"; path: string; from: string }
  | { op: "test"; path: string; value: any };
```

---

## 3. Custom HttpAgent for AgentCore

### Why We Need Custom HttpAgent

The `@ag-ui/client` package provides a base `HttpAgent` class that handles AG-UI protocol communication over HTTP/SSE. However, AWS Bedrock AgentCore requires:

1. **Session ID header** — `X-Amzn-Bedrock-AgentCore-Runtime-Session-Id` for microVM sharing across multi-agent invocations
2. **
3. **Custom URL pattern** — AgentCore Runtime endpoint URLs differ from standard AG-UI URLs

### Step-by-Step Subclassing

```typescript
// ============================================================================
// agentcore-agui-client.ts — Custom HttpAgent for AWS Bedrock AgentCore
// ============================================================================
// This file subclasses HttpAgent from @ag-ui/client to inject AgentCore-specific
// headers and URL patterns. It is the critical bridge between CopilotKit's
// useAgent hook and the AWS Bedrock AgentCore runtime.
//
// FILE LOCATION: /lib/agentcore/agentcore-agui-client.ts
// DEPENDENCIES:  @ag-ui/client, @aws-sdk/client-
// ============================================================================

import { HttpAgent, AgentRunConfig, AgentEvent } from "@ag-ui/client";
import { EventEmitter } from "events";

// ---------------------------------------------------------------------------
// Interface: AgentCore-specific configuration options
// Extends the base AgentRunConfig with AgentCore-specific fields
// ---------------------------------------------------------------------------
export interface AgentCoreRunConfig extends AgentRunConfig {
  /** The workflow ID — used as the AgentCore Runtime Session ID for microVM sharing */
  workflowId: string;
  /**
  /** AWS region where AgentCore Runtime is deployed */
  region: string;
  /** AgentCore Runtime ID — each Strands agent has its own runtime */
  runtimeId: string;
  /** AWS Account ID */
  accountId: string;
  /** Optional: qualifier (default: "DEFAULT") */
  qualifier?: string;
}

// ---------------------------------------------------------------------------
// AgentCoreHttpAgent: Subclassed HttpAgent for Bedrock AgentCore
// ---------------------------------------------------------------------------
export class AgentCoreHttpAgent extends HttpAgent {
  // Store config for use in requestInit override
  private coreConfig: AgentCoreRunConfig;

  // Event emitter for local event distribution
  private eventEmitter = new EventEmitter();

  constructor(config: AgentCoreRunConfig) {
    // Initialize base HttpAgent with the AgentCore endpoint URL
    // The URL follows the pattern:
    // https://bedrock-agentcore.<region>.amazonaws.com/runtimes/{runtime-id}/invocations?accountId={account-id}&qualifier=DEFAULT
    const url = buildAgentCoreUrl(config);

    super({
      ...config,
      url,
      // Enable SSE streaming (required for AG-UI event stream)
      streaming: true,
    });

    this.coreConfig = config;
  }

  // =========================================================================
  // CRITICAL OVERRIDE: requestInit
  // This method builds the RequestInit object for each fetch call.
  // We override it to inject AgentCore-specific headers.
  // =========================================================================
  protected async requestInit(
    body?: Record<string, any>
  ): Promise<RequestInit> {
    // Get base RequestInit from parent class
    const init = await super.requestInit(body);

    // Ensure headers object exists
    const headers = new Headers(init.headers || {});

    // -----------------------------------------------------------------------
    // HEADER 1: Session ID for microVM sharing
    // This is CRITICAL: the session ID must be the workflow_id so that
    // all agent invocations within the same workflow share the same microVM.
    // Without this, each agent invocation spins up a fresh microVM,
    // destroying any shared state (filesystem, environment variables).
    // -----------------------------------------------------------------------
    headers.set(
      "X-Amzn-Bedrock-AgentCore-Runtime-Session-Id",
      this.coreConfig.workflowId
    );

    // -----------------------------------------------------------------------
    // HEADER 2:
    // AgentCore endpoints are protected by
    // freshly obtained (check expiry!) before each request.
    // -----------------------------------------------------------------------
    headers.set(
      "Authorization",
      `Bearer ${this.coreConfig.
    );

    // -----------------------------------------------------------------------
    // HEADER 3: Content-Type for AG-UI protocol
    // The AgentCore endpoint expects JSON payloads with AG-UI event types.
    // -----------------------------------------------------------------------
    headers.set("Content-Type", "application/json");

    // -----------------------------------------------------------------------
    // HEADER 4: Accept for SSE streaming
    // text/event-stream tells the server to stream responses.
    // -----------------------------------------------------------------------
    headers.set("Accept", "text/event-stream");

    return {
      ...init,
      headers,
    };
  }

  // =========================================================================
  // As-built pattern (2026-05-17): per-request CopilotRuntime + header forwarding
  // =========================================================================
  //
  // The reference implementation above shows the AG-UI client wiring up the
  // session id on every direct fetch. The CopilotKit runtime route used in
  // this demo (`app/api/copilotkit/route.ts`) cannot reuse the same
  // approach because `CopilotRuntime({ agents: { ... } })` accepts only a
  // STATIC `HttpAgent` per agent — its `headers` object is captured at
  // construction time. To still get per-workflow session isolation, the
  // route builds a fresh `CopilotRuntime` on every POST and reads the
  // workflow id from a custom header set by the React provider:
  //
  //     // app/workspace/[wfId]/layout.tsx
  //     <CopilotKit
  //       runtimeUrl={COPILOT_CONFIG.runtimeUrl}
  //       agent={COPILOT_CONFIG.agentName}
  //       headers={{ 'x-workflow-id': wfId }}
  //     >...</CopilotKit>
  //
  //     // app/api/copilotkit/route.ts
  //     export const POST = async (req: NextRequest) => {
  //       const wf = req.headers.get('x-workflow-id') || '';
  //       // AgentCore requires runtimeSessionId of length >= 33; pad short ids
  //       // (used by the synthetic /workspace/new bootstrap, where wf === 'new').
  //       const sessionId = wf.length >= 33 ? wf
  //         : `fallback-session-${wf || crypto.randomUUID()}`.padEnd(33, '0');
  //       const runtime = new CopilotRuntime({
  //         agents: {
  //           strands_transcriber: new HttpAgent({
  //             url: process.env.AGENT_1_RUNTIME_URL!,
  //             headers: { 'X-Amzn-Bedrock-AgentCore-Runtime-Session-Id': sessionId },
  //           }),
  //           // ... drafter + reviewer ...
  //         },
  //       });
  //       const { handleRequest } = copilotRuntimeNextJSAppRouterEndpoint({
  //         runtime, serviceAdapter: new ExperimentalEmptyAdapter(),
  //         endpoint: '/api/copilotkit',
  //       });
  //       return handleRequest(req);
  //     };
  //
  // Net effect: each `/api/copilotkit` request that originates from inside
  // a `/workspace/[wfId]` page lands on the AgentCore microVM bound to that
  // workflow id. Requests from outside a workspace (no header) fall back to
  // a generated id so the chat at least loads instead of throwing.
  //
  // Also at module load the route logs a `console.warn` if any of
  // `AGENT_1/2/3_RUNTIME_URL` is empty — `HttpAgent` silently accepts an
  // empty URL and the failure would otherwise only appear deep inside a
  // CopilotKit fetch with no clear cause.

  // =========================================================================
  // Event Stream Access
  // Override to intercept events for local distribution and logging
  // =========================================================================
  public async *stream(input: string): AsyncGenerator<AgentEvent, void, unknown> {
    try {
      console.log(`[AgentCoreHttpAgent] Starting stream for workflow ${this.coreConfig.workflowId}`);

      for await (const event of super.stream(input)) {
        // Emit locally for any subscribers (analytics, logging, etc.)
        this.eventEmitter.emit("event", event);

        // Log events in development
        if (process.env.NODE_ENV === "development") {
          console.log(`[AgentCoreHttpAgent] Event: ${event.type}`,
            event.type === "TEXT_MESSAGE_CONTENT" ? "(content chunk)" : ""
          );
        }

        yield event;
      }

      console.log(`[AgentCoreHttpAgent] Stream completed for workflow ${this.coreConfig.workflowId}`);
    } catch (error) {
      console.error(`[AgentCoreHttpAgent] Stream error:`, error);
      throw error;
    }
  }

  // =========================================================================
  // Public API for event subscription (analytics, custom handlers)
  // =========================================================================
  public onEvent(callback: (event: AgentEvent) => void): () => void {
    this.eventEmitter.on("event", callback);
    return () => this.eventEmitter.off("event", callback);
  }
}

// ---------------------------------------------------------------------------
// URL Builder: Constructs the AgentCore Runtime endpoint URL
// ---------------------------------------------------------------------------
function buildAgentCoreUrl(config: AgentCoreRunConfig): string {
  const {
    region,
    runtimeId,
    accountId,
    qualifier = "DEFAULT",
  } = config;

  // The AgentCore Runtime URL pattern:
  // https://bedrock-agentcore.<region>.amazonaws.com/runtimes/{runtime-id}/invocations?accountId={account-id}&qualifier=DEFAULT
  const url = new URL(
    `https://bedrock-agentcore.${region}.amazonaws.com/runtimes/${runtimeId}/invocations`
  );
  url.searchParams.set("accountId", accountId);
  url.searchParams.set("qualifier", qualifier);

  return url.toString();
}

// ---------------------------------------------------------------------------
// Factory Function: Creates AgentCoreHttpAgent with auth
// This is the recommended entry point — handles token refresh transparently.
// ---------------------------------------------------------------------------
export async function createAgentCoreHttpAgent(
  params: Omit<AgentCoreRunConfig, "
    /** Function that returns a fresh
    get
  }
): Promise<AgentCoreHttpAgent> {
  // Obtain fresh token (handles refresh if needed)
  const

  const config: AgentCoreRunConfig = {
    ...params,
  };

  return new AgentCoreHttpAgent(config);
}

// ---------------------------------------------------------------------------
// React Hook: useAgentCoreHttpAgent
// Wraps the factory in a React-friendly hook with automatic cleanup
// ---------------------------------------------------------------------------
import { useState, useEffect, useCallback, useRef } from "react";

export function useAgentCoreHttpAgent(
  config: Omit<AgentCoreRunConfig, "
    get
  }
) {
  const [agent, setAgent] = useState<AgentCoreHttpAgent | null>(null);
  const [isReady, setIsReady] = useState(false);
  const [error, setError] = useState<Error | null>(null);

  // Use a ref to track the current agent for cleanup
  const agentRef = useRef<AgentCoreHttpAgent | null>(null);

  const initialize = useCallback(async () => {
    try {
      setError(null);
      const newAgent = await createAgentCoreHttpAgent(config);

      // Clean up previous agent if exists
      if (agentRef.current) {
        agentRef.current.dispose?.();
      }

      agentRef.current = newAgent;
      setAgent(newAgent);
      setIsReady(true);
    } catch (err) {
      setError(err instanceof Error ? err : new Error(String(err)));
      setIsReady(false);
    }
  }, [config.workflowId, config.runtimeId, config.region, config.accountId]);

  useEffect(() => {
    initialize();

    // Cleanup on unmount
    return () => {
      if (agentRef.current) {
        agentRef.current.dispose?.();
      }
    };
  }, [initialize]);

  return { agent, isReady, error, reinitialize: initialize };
}
```

### Usage in a React Component

```typescript
// ============================================================================
// Example: Using the custom AgentCoreHttpAgent with CopilotKit
// ============================================================================

import { useAgent } from "@copilotkit/react-core";
import { useAgentCoreHttpAgent } from "@/lib/agentcore/agentcore-agui-client";
import { useAuth } from "@/hooks/use-auth";  // Your

function ContentGenerationPage() {
  const { getIdToken, isAuthenticated } = useAuth();
  const [workflowId, setWorkflowId] = useState<string>("");

  // Create our custom AgentCore HTTP agent
  const { agent: coreAgent, isReady: coreReady } = useAgentCoreHttpAgent({
    workflowId,                          // Session ID = workflow_id
    runtimeId: "arn:aws:bedrock:us-east-1:123456789:runtime/my-agent",
    region: "us-east-1",
    accountId: "123456789",
    get
  });

  // CopilotKit's useAgent hook connects to the AG-UI stream
  const agent = useAgent({
    id: "content-generation-agent",
    name: "Content Generation Agent",
    // Pass our custom HTTP agent as the transport layer
    httpAgent: coreAgent ?? undefined,
  });

  const handleStart = async (topic: string) => {
    // Generate a new workflow ID
    const newWorkflowId = `wf_${Date.now()}_${Math.random().toString(36).slice(2)}`;
    setWorkflowId(newWorkflowId);

    // Start the run — CopilotKit will use our custom HttpAgent
    agent.run(`Generate content about: ${topic}`, {
      initialState: { topic, status: "idle" },
    });
  };

  if (!isAuthenticated) return <LoginPrompt />;
  if (!coreReady) return <LoadingSpinner />;

  return (
    <div>
      <AgentStatus status={agent.status} />
      <Canvas state={agent.state} />
      <ChatInterface messages={agent.messages} onSend={handleStart} />
    </div>
  );
}
```

---

## 4. Coordinating AG-UI Events with Temporal Signals (Hybrid Pattern)

### The Dual-Response Problem

In our hybrid architecture, Human-in-the-Loop (HITL) interactions require **two simultaneous responses**:

1. **AG-UI `respond()`** — Unblocks the agent's AG-UI event stream so it can continue emitting events
2. **Temporal Signal** — Sends the human's decision to the Temporal workflow orchestrating multi-agent execution

If only one succeeds, the system enters an inconsistent state.

### Detailed Flow Diagram

```
┌──────────┐     INTERRUPT event      ┌──────────────┐
│  Agent   │ ───────────────────────► │   Frontend   │
│ (microVM)│   "Approve this draft?"  │   (Canvas)   │
└──────────┘                          └──────┬───────┘
                                             │
                                             ▼
                                    ┌──────────────────┐
                                    │  Render Approval │
                                    │     Card UI      │
                                    └──────┬───────────┘
                                           │
                              User clicks "Approve"
                                           │
                                           ▼
                          ┌─────────────────────────────────┐
                          │        DUAL RESPONSE            │
                          │  ┌─────────────────────────┐    │
                          │  │ 1. respond("APPROVED")  │    │
                          │  │    → AG-UI stream       │    │
                          │  │    → Unblocks agent     │    │
                          │  └─────────────────────────┘    │
                          │  ┌─────────────────────────┐    │
                          │  │ 2. signalWorkflow({     │    │
                          │  │      decision: "APPROVE" │    │
                          │  │    })                   │    │
                          │  │    → Temporal gRPC      │    │
                          │  │    → Advances workflow  │    │
                          │  └─────────────────────────┘    │
                          └─────────────────────────────────┘
                                           │
                              ┌────────────┴────────────┐
                              ▼                         ▼
                        ┌──────────┐              ┌──────────────┐
                        │  Agent   │              │   Temporal   │
                        │ Continues│              │  Workflow    │
                        │  Stream  │              │  Advances    │
                        └──────────┘              └──────────────┘
```

### Implementation: The Dual-Response Function

```typescript
// ============================================================================
// hooks/useDualResponse.ts — Dual-response pattern for HITL
// ============================================================================
// This hook coordinates AG-UI respond() with Temporal Signal to ensure
// both systems receive the human's decision. Includes failure handling
// and idempotency protection.
//
// FILE LOCATION: /hooks/use-dual-response.ts
// ============================================================================

import { useCallback, useState, useRef } from "react";
import { useAgent } from "@copilotkit/react-core";
import { TemporalClient } from "@/lib/temporal/client";

export interface DualResponseConfig {
  /** Temporal workflow ID to signal */
  temporalWorkflowId: string;
  /** Temporal namespace */
  temporalNamespace: string;
  /** Signal name for HITL responses */
  hitlSignalName?: string;
}

export interface DualResponseResult {
  /** Overall success (both succeeded or recovered) */
  success: boolean;
  /** AG-UI respond() result */
  aguiSuccess: boolean;
  /** Temporal Signal result */
  temporalSuccess: boolean;
  /** Any error that occurred */
  error?: string;
}

export function useDualResponse(config: DualResponseConfig) {
  const agent = useAgent({ ... });  // Your agent config
  const [isResponding, setIsResponding] = useState(false);
  const [lastResult, setLastResult] = useState<DualResponseResult | null>(null);

  // Use a ref to prevent duplicate responses for the same interrupt
  const responseLock = useRef<string | null>(null);

  const temporal = new TemporalClient({
    namespace: config.temporalNamespace,
  });

  /**
   * Send a dual-response: AG-UI respond() + Temporal Signal simultaneously.
   * Uses Promise.allSettled so one failure doesn't prevent the other.
   */
  const sendDualResponse = useCallback(
    async (decision: string, metadata?: Record<string, any>): Promise<DualResponseResult> => {
      const interruptId = agent.currentInterruptId || "unknown";

      // Idempotency: prevent double-submission for same interrupt
      if (responseLock.current === interruptId) {
        console.warn("[DualResponse] Duplicate response blocked for interrupt:", interruptId);
        return { success: true, aguiSuccess: true, temporalSuccess: true };
      }
      responseLock.current = interruptId;

      setIsResponding(true);

      try {
        // -------------------------------------------------------------------
        // STEP 1: Fire both requests simultaneously
        // We use Promise.allSettled() so that even if one fails, the other
        // still executes. This is critical for partial failure recovery.
        // -------------------------------------------------------------------
        const [aguiResult, temporalResult] = await Promise.allSettled([
          // Request 1: AG-UI respond()
          // This unblocks the agent's event stream.
          Promise.resolve(agent.respond(decision)),

          // Request 2: Temporal Signal
          // This advances the multi-agent workflow.
          temporal.signalWorkflow({
            workflowId: config.temporalWorkflowId,
            signalName: config.hitlSignalName || "humanDecision",
            data: {
              decision,
              metadata,
              timestamp: Date.now(),
              interruptId,
            },
          }),
        ]);

        // -------------------------------------------------------------------
        // STEP 2: Analyze results
        // -------------------------------------------------------------------
        const aguiSuccess = aguiResult.status === "fulfilled";
        const temporalSuccess = temporalResult.status === "fulfilled";

        // Log individual failures for debugging
        if (!aguiSuccess) {
          console.error("[DualResponse] AG-UI respond() failed:", aguiResult.reason);
        }
        if (!temporalSuccess) {
          console.error("[DualResponse] Temporal Signal failed:", temporalResult.reason);
        }

        // -------------------------------------------------------------------
        // STEP 3: Handle failure modes
        // -------------------------------------------------------------------
        if (aguiSuccess && temporalSuccess) {
          // PERFECT: Both succeeded
          const result: DualResponseResult = {
            success: true,
            aguiSuccess: true,
            temporalSuccess: true,
          };
          setLastResult(result);
          return result;
        }

        if (aguiSuccess && !temporalSuccess) {
          // MODE 1: AG-UI succeeded, Temporal failed
          // Agent is unblocked but workflow is stuck.
          // Strategy: Show warning, allow retry of Temporal signal only.
          const result: DualResponseResult = {
            success: false,
            aguiSuccess: true,
            temporalSuccess: false,
            error: `Temporal Signal failed: ${temporalResult.reason}`,
          };
          setLastResult(result);
          return result;
        }

        if (!aguiSuccess && temporalSuccess) {
          // MODE 2: AG-UI failed, Temporal succeeded
          // Workflow advanced but agent stream is still blocked.
          // Strategy: Agent will likely timeout/retry; show error to user.
          const result: DualResponseResult = {
            success: false,
            aguiSuccess: false,
            temporalSuccess: true,
            error: `AG-UI respond() failed: ${aguiResult.reason}`,
          };
          setLastResult(result);
          return result;
        }

        // MODE 3: Both failed — worst case
        const result: DualResponseResult = {
          success: false,
          aguiSuccess: false,
          temporalSuccess: false,
          error: `Both failed. AG-UI: ${aguiResult.reason}; Temporal: ${temporalResult.reason}`,
        };
        setLastResult(result);
        return result;

      } catch (unexpectedError) {
        // Catastrophic failure — both likely failed
        const result: DualResponseResult = {
          success: false,
          aguiSuccess: false,
          temporalSuccess: false,
          error: `Unexpected error: ${unexpectedError}`,
        };
        setLastResult(result);
        return result;
      } finally {
        setIsResponding(false);
      }
    },
    [agent, config.temporalWorkflowId, config.temporalNamespace]
  );

  /**
   * Retry a failed Temporal Signal only (AG-UI already succeeded).
   */
  const retryTemporalSignal = useCallback(
    async (decision: string, metadata?: Record<string, any>) => {
      try {
        await temporal.signalWorkflow({
          workflowId: config.temporalWorkflowId,
          signalName: config.hitlSignalName || "humanDecision",
          data: { decision, metadata, timestamp: Date.now() },
        });
        setLastResult((prev) =>
          prev ? { ...prev, temporalSuccess: true, success: true, error: undefined } : null
        );
        return true;
      } catch (error) {
        console.error("[DualResponse] Retry failed:", error);
        return false;
      }
    },
    [config.temporalWorkflowId]
  );

  return {
    sendDualResponse,
    retryTemporalSignal,
    isResponding,
    lastResult,
  };
}
```

### Failure Mode UX

```tsx
// ============================================================================
// components/DualResponseStatus.tsx — UX for each failure mode
// ============================================================================

function DualResponseStatus({ result }: { result: DualResponseResult | null }) {
  if (!result) return null;

  if (result.success) {
    return (
      <div className="p-3 bg-green-50 border border-green-200 rounded-md">
        <p className="text-green-800">Response sent successfully</p>
      </div>
    );
  }

  // Mode 1: AG-UI OK, Temporal failed — show retry button
  if (result.aguiSuccess && !result.temporalSuccess) {
    return (
      <div className="p-3 bg-yellow-50 border border-yellow-200 rounded-md">
        <p className="text-yellow-800">
          Agent received your response, but workflow update failed.
        </p>
        <button
          onClick={() => retryTemporalSignal(result.decision)}
          className="mt-2 px-3 py-1 bg-yellow-500 text-white rounded"
        >
          Retry Workflow Update
        </button>
      </div>
    );
  }

  // Mode 2: AG-UI failed, Temporal OK — show warning
  if (!result.aguiSuccess && result.temporalSuccess) {
    return (
      <div className="p-3 bg-orange-50 border border-orange-200 rounded-md">
        <p className="text-orange-800">
          Workflow updated, but agent display may be stale.
          The page will auto-recover.
        </p>
      </div>
    );
  }

  // Mode 3: Both failed — show full error
  return (
    <div className="p-3 bg-red-50 border border-red-200 rounded-md">
      <p className="text-red-800">
        Failed to send response. Please refresh and try again.
      </p>
      <p className="text-red-600 text-sm mt-1">{result.error}</p>
    </div>
  );
}
```

### Idempotency Considerations

```typescript
// ============================================================================
// Idempotency Strategy
// ============================================================================
// 1. RESPONSE_LOCK: Frontend uses a ref to prevent double-clicks
// 2. INTERRUPT_ID: Each INTERRUPT event has a unique ID; responses
//    are keyed to it. Duplicate responses with same ID are ignored.
// 3. TEMPORAL: Temporal Signals are naturally idempotent — sending
//    the same signal twice is a no-op if the workflow has already
//    processed it (use workflow state checks).
// 4. AG-UI: The respond() call is idempotent on the server side
//    for a given interrupt session.
```

---

## 5. Generative UI Cards Library

### Static Generative UI (Controlled) Pattern

Our demo uses the **Static Generative UI (Controlled)** pattern. Unlike dynamic generative UI where the agent decides what component to render, in the controlled pattern:

1. The frontend registers specific tools using `useFrontendTool`
2. The agent calls these tools by name via TOOL_CALL_START/END events
3. The frontend renders the matching component
4. For HITL, `renderAndWaitForResponse` blocks until the user interacts

**Reference:** https://www.copilotkit.ai/blog/the-developer-s-guide-to-generative-ui-in-2026

### Complete Card Library

#### TranscriptCard.tsx

```tsx
// ============================================================================
// components/generative-ui/TranscriptCard.tsx
// ============================================================================
// Displays a generated transcript with expandable sections.
// Follows the Static Generative UI (Controlled) pattern: the agent emits
// a STATE_SNAPSHOT or TOOL_CALL with transcript data, and this card
// renders it in a structured format.
//
// USAGE: Agent calls "display_transcript" tool → frontend renders this card
// FILE: components/generative-ui/TranscriptCard.tsx
// ============================================================================

import React, { useState } from "react";

// ---------------------------------------------------------------------------
// Data Interface: The shape of transcript data from the agent
// ---------------------------------------------------------------------------
export interface TranscriptSection {
  /** Unique identifier for this section */
  id: string;
  /** Section heading (e.g., "Introduction", "Key Points") */
  heading: string;
  /** Timestamp or position marker */
  timestamp?: string;
  /** The transcript text content */
  content: string;
  /** Speaker identifier, if applicable */
  speaker?: string;
  /** Confidence score 0-1 (optional, for review workflows) */
  confidence?: number;
}

export interface TranscriptCardProps {
  /** The transcript title/topic */
  title: string;
  /** Array of transcript sections */
  sections: TranscriptSection[];
  /** Overall confidence or quality score */
  overallConfidence?: number;
  /** Callback when user approves the transcript */
  onApprove?: (sections: TranscriptSection[]) => void;
  /** Callback when user requests edits */
  onRequestEdits?: (sectionId: string, feedback: string) => void;
  /** Whether this card is in review mode (shows approve/reject buttons) */
  isReviewMode?: boolean;
  /** Optional CSS class */
  className?: string;
}

/**
 * TranscriptCard: Displays a structured transcript with sections.
 *
 * ARCHITECTURE NOTE: This is a "Controlled" generative UI component.
 * The agent does NOT decide what component to render. Instead, the agent
 * emits structured data (STATE_SNAPSHOT with transcript sections), and
 * the frontend maps that data to this pre-registered component.
 *
 * This approach gives the frontend full control over UX and ensures
 * consistent rendering regardless of which agent produced the data.
 */
export function TranscriptCard({
  title,
  sections,
  overallConfidence,
  onApprove,
  onRequestEdits,
  isReviewMode = false,
  className = "",
}: TranscriptCardProps) {
  // Track which sections are expanded
  const [expandedSections, setExpandedSections] = useState<Set<string>>(
    // Default: expand the first section
    new Set(sections.length > 0 ? [sections[0].id] : [])
  );

  // Track edit feedback per section
  const [editFeedback, setEditFeedback] = useState<Record<string, string>>({});

  const toggleSection = (id: string) => {
    setExpandedSections((prev) => {
      const next = new Set(prev);
      if (next.has(id)) {
        next.delete(id);
      } else {
        next.add(id);
      }
      return next;
    });
  };

  const handleApproveAll = () => {
    onApprove?.(sections);
  };

  return (
    <div className={`bg-white border border-gray-200 rounded-lg shadow-sm ${className}`}>
      {/* Header */}
      <div className="px-4 py-3 border-b border-gray-100 flex items-center justify-between">
        <div>
          <h3 className="text-lg font-semibold text-gray-900">{title}</h3>
          {overallConfidence !== undefined && (
            <div className="flex items-center mt-1">
              <span className="text-xs text-gray-500 mr-2">
                Confidence: {Math.round(overallConfidence * 100)}%
              </span>
              <div className="w-20 h-1.5 bg-gray-200 rounded-full overflow-hidden">
                <div
                  className="h-full bg-green-500 rounded-full"
                  style={{ width: `${overallConfidence * 100}%` }}
                />
              </div>
            </div>
          )}
        </div>
        <span className="text-xs text-gray-400">{sections.length} sections</span>
      </div>

      {/* Sections */}
      <div className="divide-y divide-gray-100">
        {sections.map((section) => (
          <div key={section.id} className="px-4 py-3">
            {/* Section Header (clickable to expand) */}
            <button
              onClick={() => toggleSection(section.id)}
              className="w-full flex items-center justify-between text-left hover:bg-gray-50 -mx-4 -my-3 px-4 py-3 transition-colors"
            >
              <div className="flex items-center">
                {/* Expand/collapse chevron */}
                <svg
                  className={`w-4 h-4 mr-2 text-gray-400 transition-transform ${
                    expandedSections.has(section.id) ? "rotate-90" : ""
                  }`}
                  fill="none"
                  stroke="currentColor"
                  viewBox="0 0 24 24"
                >
                  <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M9 5l7 7-7 7" />
                </svg>
                <span className="font-medium text-gray-800">{section.heading}</span>
                {section.speaker && (
                  <span className="ml-2 text-xs text-gray-400">({section.speaker})</span>
                )}
              </div>
              {section.confidence !== undefined && (
                <span
                  className={`text-xs px-2 py-0.5 rounded-full ${
                    section.confidence > 0.8
                      ? "bg-green-100 text-green-700"
                      : section.confidence > 0.5
                      ? "bg-yellow-100 text-yellow-700"
                      : "bg-red-100 text-red-700"
                  }`}
                >
                  {Math.round(section.confidence * 100)}%
                </span>
              )}
            </button>

            {/* Expanded Content */}
            {expandedSections.has(section.id) && (
              <div className="mt-2 pl-6 pr-2">
                {section.timestamp && (
                  <span className="text-xs text-gray-400 mb-1 block">{section.timestamp}</span>
                )}
                <p className="text-sm text-gray-700 leading-relaxed whitespace-pre-wrap">
                  {section.content}
                </p>

                {/* Edit feedback input (review mode only) */}
                {isReviewMode && onRequestEdits && (
                  <div className="mt-3">
                    <textarea
                      placeholder="Request edits for this section..."
                      className="w-full text-sm border border-gray-300 rounded-md px-3 py-2 focus:ring-2 focus:ring-blue-500 focus:border-transparent"
                      rows={2}
                      value={editFeedback[section.id] || ""}
                      onChange={(e) =>
                        setEditFeedback((prev) => ({
                          ...prev,
                          [section.id]: e.target.value,
                        }))
                      }
                    />
                    {editFeedback[section.id]?.trim() && (
                      <button
                        onClick={() => {
                          onRequestEdits(section.id, editFeedback[section.id]);
                          setEditFeedback((prev) => ({ ...prev, [section.id]: "" }));
                        }}
                        className="mt-1 text-xs px-3 py-1 bg-gray-800 text-white rounded hover:bg-gray-700"
                      >
                        Send Edit Request
                      </button>
                    )}
                  </div>
                )}
              </div>
            )}
          </div>
        ))}
      </div>

      {/* Footer with approve button (review mode) */}
      {isReviewMode && onApprove && (
        <div className="px-4 py-3 border-t border-gray-100 flex justify-end">
          <button
            onClick={handleApproveAll}
            className="px-4 py-2 bg-green-600 text-white rounded-md hover:bg-green-700 transition-colors"
          >
            Approve All Sections
          </button>
        </div>
      )}
    </div>
  );
}
```

#### DraftPreviewCard.tsx

```tsx
// ============================================================================
// components/generative-ui/DraftPreviewCard.tsx
// ============================================================================
// Displays a preview of generated content (blog post, email, etc.) with
// formatting, metadata, and action buttons. Uses the Controlled pattern
// where the agent pushes STATE_SNAPSHOT with draft data.
//
// USAGE: Agent calls "display_draft" tool → frontend renders this card
// FILE: components/generative-ui/DraftPreviewCard.tsx
// ============================================================================

import React, { useState } from "react";

export interface DraftMetadata {
  wordCount: number;
  readingTimeMinutes: number;
  tone: string;
  targetAudience?: string;
  seoKeywords?: string[];
  generatedAt: string;
}

export interface DraftPreviewCardProps {
  /** Draft title */
  title: string;
  /** Draft body content (supports markdown-like formatting) */
  content: string;
  /** Draft metadata */
  metadata: DraftMetadata;
  /** Revision number */
  revision?: number;
  /** Whether this is the latest revision */
  isLatest?: boolean;
  /** Callback: user approves this draft */
  onApprove: () => void;
  /** Callback: user requests changes */
  onRequestChanges: (feedback: string) => void;
  /** Callback: user wants to see previous revision */
  onViewPrevious?: () => void;
  className?: string;
}

/**
 * DraftPreviewCard: Renders a generated content draft with full preview.
 *
 * CONTROLLED PATTERN: The frontend registers this component as the handler
 * for "display_draft" tool calls. The agent emits:
 *   { tool: "display_draft", arguments: { title, content, metadata, revision } }
 * The frontend then renders this card with those props.
 */
export function DraftPreviewCard({
  title,
  content,
  metadata,
  revision = 1,
  isLatest = true,
  onApprove,
  onRequestChanges,
  onViewPrevious,
  className = "",
}: DraftPreviewCardProps) {
  const [feedback, setFeedback] = useState("");
  const [showFeedbackForm, setShowFeedbackForm] = useState(false);

  // Simple markdown-like rendering: **bold**, *italic*, ### headings
  const renderFormattedContent = (text: string) => {
    const lines = text.split("\n");
    return lines.map((line, i) => {
      // Heading: ### Heading
      if (line.startsWith("### ")) {
        return (
          <h4 key={i} className="text-lg font-semibold text-gray-900 mt-4 mb-2">
            {line.replace("### ", "")}
          </h4>
        );
      }
      // Heading: ## Heading
      if (line.startsWith("## ")) {
        return (
          <h3 key={i} className="text-xl font-semibold text-gray-900 mt-5 mb-3">
            {line.replace("## ", "")}
          </h3>
        );
      }
      // Bullet point
      if (line.startsWith("- ")) {
        return (
          <li key={i} className="ml-5 text-sm text-gray-700 leading-relaxed">
            {line.replace("- ", "")}
          </li>
        );
      }
      // Bold: **text**
      const bolded = line.replace(/\*\*(.+?)\*\*/g, "<strong>$1</strong>");
      // Italic: *text*
      const italicized = bolded.replace(/\*(.+?)\*/g, "<em>$1</em>");

      return (
        <p
          key={i}
          className="text-sm text-gray-700 leading-relaxed mb-2"
          dangerouslySetInnerHTML={{ __html: italicized }}
        />
      );
    });
  };

  return (
    <div className={`bg-white border border-gray-200 rounded-lg shadow-sm overflow-hidden ${className}`}>
      {/* Header with metadata */}
      <div className="bg-gray-50 px-4 py-3 border-b border-gray-200">
        <div className="flex items-center justify-between">
          <h3 className="text-lg font-semibold text-gray-900">{title}</h3>
          <div className="flex items-center gap-2">
            {revision > 1 && (
              <span className="text-xs bg-blue-100 text-blue-700 px-2 py-0.5 rounded-full">
                Rev {revision}
              </span>
            )}
            {isLatest && (
              <span className="text-xs bg-green-100 text-green-700 px-2 py-0.5 rounded-full">
                Latest
              </span>
            )}
          </div>
        </div>

        {/* Metadata bar */}
        <div className="flex flex-wrap gap-3 mt-2 text-xs text-gray-500">
          <span>{metadata.wordCount.toLocaleString()} words</span>
          <span>{metadata.readingTimeMinutes} min read</span>
          <span>Tone: {metadata.tone}</span>
          {metadata.targetAudience && <span>Audience: {metadata.targetAudience}</span>}
        </div>

        {/* SEO keywords */}
        {metadata.seoKeywords && metadata.seoKeywords.length > 0 && (
          <div className="flex flex-wrap gap-1 mt-2">
            {metadata.seoKeywords.map((kw) => (
              <span
                key={kw}
                className="text-xs bg-purple-100 text-purple-700 px-2 py-0.5 rounded"
              >
                {kw}
              </span>
            ))}
          </div>
        )}
      </div>

      {/* Content body */}
      <div className="px-4 py-4 max-h-96 overflow-y-auto">
        {renderFormattedContent(content)}
      </div>

      {/* Action footer */}
      <div className="px-4 py-3 border-t border-gray-200 bg-gray-50">
        {showFeedbackForm ? (
          <div className="space-y-2">
            <textarea
              placeholder="Describe the changes you want..."
              className="w-full text-sm border border-gray-300 rounded-md px-3 py-2 focus:ring-2 focus:ring-blue-500"
              rows={3}
              value={feedback}
              onChange={(e) => setFeedback(e.target.value)}
              autoFocus
            />
            <div className="flex gap-2">
              <button
                onClick={() => {
                  if (feedback.trim()) {
                    onRequestChanges(feedback);
                    setFeedback("");
                    setShowFeedbackForm(false);
                  }
                }}
                className="px-4 py-2 bg-blue-600 text-white text-sm rounded-md hover:bg-blue-700"
              >
                Submit Feedback
              </button>
              <button
                onClick={() => setShowFeedbackForm(false)}
                className="px-4 py-2 bg-gray-200 text-gray-700 text-sm rounded-md hover:bg-gray-300"
              >
                Cancel
              </button>
            </div>
          </div>
        ) : (
          <div className="flex items-center justify-between">
            <div className="flex gap-2">
              <button
                onClick={onApprove}
                className="px-4 py-2 bg-green-600 text-white text-sm rounded-md hover:bg-green-700 flex items-center gap-1"
              >
                <svg className="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                  <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M5 13l4 4L19 7" />
                </svg>
                Approve
              </button>
              <button
                onClick={() => setShowFeedbackForm(true)}
                className="px-4 py-2 bg-white border border-gray-300 text-gray-700 text-sm rounded-md hover:bg-gray-50 flex items-center gap-1"
              >
                <svg className="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                  <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M11 5H6a2 2 0 00-2 2v11a2 2 0 002 2h11a2 2 0 002-2v-5m-1.414-9.414a2 2 0 112.828 2.828L11.828 15H9v-2.828l8.586-8.586z" />
                </svg>
                Request Changes
              </button>
            </div>
            {revision > 1 && onViewPrevious && (
              <button
                onClick={onViewPrevious}
                className="text-xs text-gray-500 hover:text-gray-700 underline"
              >
                View Previous
              </button>
            )}
          </div>
        )}
      </div>
    </div>
  );
}
```

#### ReviewReportCard.tsx

```tsx
// ============================================================================
// components/generative-ui/ReviewReportCard.tsx
// ============================================================================
// Displays a structured review report with issues, suggestions, and
// severity indicators. Used in the content review phase of the workflow.
//
// USAGE: Agent calls "display_review_report" tool → frontend renders this card
// FILE: components/generative-ui/ReviewReportCard.tsx
// ============================================================================

import React, { useState } from "react";

export interface ReviewIssue {
  /** Unique issue ID */
  id: string;
  /** Issue category */
  category: "grammar" | "factual" | "style" | "seo" | "tone" | "other";
  /** Severity level */
  severity: "critical" | "warning" | "suggestion";
  /** Location in the content (e.g., "Section 2, paragraph 3") */
  location: string;
  /** Issue description */
  description: string;
  /** Suggested fix */
  suggestion: string;
  /** Whether this issue has been addressed */
  resolved?: boolean;
}

export interface ReviewReportCardProps {
  /** Report title */
  title: string;
  /** The content being reviewed */
  contentPreview: string;
  /** List of issues found */
  issues: ReviewIssue[];
  /** Overall quality score 0-100 */
  overallScore: number;
  /** Summary text from the reviewer agent */
  summary: string;
  /** Callback: mark an issue as resolved */
  onResolveIssue?: (issueId: string) => void;
  /** Callback: approve the report and proceed */
  onApprove: () => void;
  /** Callback: send feedback on the report */
  onSendFeedback?: (feedback: string) => void;
  className?: string;
}

/**
 * ReviewReportCard: Structured content review with issue tracking.
 *
 * PATTERN: Controlled generative UI. The review agent emits a STATE_SNAPSHOT
 * containing the review data. The frontend maps this to the ReviewReportCard.
 * Issue resolution state is managed locally (useState) and can be synced back
 * to the agent via onResolveIssue callback.
 */
export function ReviewReportCard({
  title,
  contentPreview,
  issues,
  overallScore,
  summary,
  onResolveIssue,
  onApprove,
  onSendFeedback,
  className = "",
}: ReviewReportCardProps) {
  const [resolvedIds, setResolvedIds] = useState<Set<string>>(
    new Set(issues.filter((i) => i.resolved).map((i) => i.id))
  );
  const [selectedCategory, setSelectedCategory] = useState<string>("all");
  const [feedback, setFeedback] = useState("");

  const categories = ["all", ...new Set(issues.map((i) => i.category))];

    selectedCategory === "all" ? issues : issues.filter((i) => i.category === selectedCategory);

  const handleResolve = (issueId: string) => {
    setResolvedIds((prev) => {
      const next = new Set(prev);
      if (next.has(issueId)) {
        next.delete(issueId);
      } else {
        next.add(issueId);
      }
      return next;
    });
    onResolveIssue?.(issueId);
  };

  const severityConfig = {
    critical: { color: "bg-red-100 text-red-700 border-red-200", icon: "🔴" },
    warning: { color: "bg-yellow-100 text-yellow-700 border-yellow-200", icon: "🟡" },
    suggestion: { color: "bg-blue-100 text-blue-700 border-blue-200", icon: "🔵" },
  };

  const scoreColor =
    overallScore >= 80 ? "text-green-600" : overallScore >= 50 ? "text-yellow-600" : "text-red-600";

  return (
    <div className={`bg-white border border-gray-200 rounded-lg shadow-sm ${className}`}>
      {/* Header with score */}
      <div className="px-4 py-4 border-b border-gray-200">
        <div className="flex items-center justify-between">
          <h3 className="text-lg font-semibold text-gray-900">{title}</h3>
          <div className="flex items-center gap-3">
            <div className="text-center">
              <span className={`text-2xl font-bold ${scoreColor}`}>{overallScore}</span>
              <span className="text-xs text-gray-400 block">/100</span>
            </div>
          </div>
        </div>

        {/* Issue counts by severity */}
        <div className="flex gap-3 mt-3">
          {(["critical", "warning", "suggestion"] as const).map((sev) => {
            const count = issues.filter((i) => i.severity === sev).length;
            return (
              <span
                key={sev}
                className={`text-xs px-2 py-1 rounded-full border ${severityConfig[sev].color}`}
              >
                {severityConfig[sev].icon} {count} {sev}
              </span>
            );
          })}
        </div>

        {/* Summary */}
        <p className="text-sm text-gray-600 mt-3 leading-relaxed">{summary}</p>
      </div>

      {/* Category filter */}
      <div className="px-4 py-2 border-b border-gray-100 flex gap-2 overflow-x-auto">
        {categories.map((cat) => (
          <button
            key={cat}
            onClick={() => setSelectedCategory(cat)}
            className={`text-xs px-3 py-1 rounded-full capitalize transition-colors ${
              selectedCategory === cat
                ? "bg-gray-800 text-white"
                : "bg-gray-100 text-gray-600 hover:bg-gray-200"
            }`}
          >
            {cat}
            {cat !== "all" && (
              <span className="ml-1 opacity-60">
                ({issues.filter((i) => i.category === cat).length})
              </span>
            )}
          </button>
        ))}
      </div>

      {/* Issues list */}
      <div className="max-h-80 overflow-y-auto divide-y divide-gray-100">
          <div
            key={issue.id}
            className={`px-4 py-3 transition-opacity ${
              resolvedIds.has(issue.id) ? "opacity-50 bg-gray-50" : ""
            }`}
          >
            <div className="flex items-start gap-3">
              {/* Resolve checkbox */}
              {onResolveIssue && (
                <button
                  onClick={() => handleResolve(issue.id)}
                  className={`mt-0.5 w-5 h-5 rounded border flex items-center justify-center flex-shrink-0 ${
                    resolvedIds.has(issue.id)
                      ? "bg-green-500 border-green-500 text-white"
                      : "border-gray-300 hover:border-gray-400"
                  }`}
                >
                  {resolvedIds.has(issue.id) && (
                    <svg className="w-3 h-3" fill="currentColor" viewBox="0 0 20 20">
                      <path
                        fillRule="evenodd"
                        d="M16.707 5.293a1 1 0 010 1.414l-8 8a1 1 0 01-1.414 0l-4-4a1 1 0 011.414-1.414L8 12.586l7.293-7.293a1 1 0 011.414 0z"
                        clipRule="evenodd"
                      />
                    </svg>
                  )}
                </button>
              )}

              <div className="flex-1">
                <div className="flex items-center gap-2">
                  <span
                    className={`text-xs px-2 py-0.5 rounded border ${severityConfig[issue.severity].color}`}
                  >
                    {issue.severity}
                  </span>
                  <span className="text-xs text-gray-400 capitalize">{issue.category}</span>
                  <span className="text-xs text-gray-400">{issue.location}</span>
                </div>
                <p className="text-sm text-gray-800 mt-1">{issue.description}</p>
                <p className="text-sm text-gray-500 mt-1 italic">Suggested: {issue.suggestion}</p>
              </div>
            </div>
          </div>
        ))}
      </div>

      {/* Footer actions */}
      <div className="px-4 py-3 border-t border-gray-200 bg-gray-50 space-y-3">
        {onSendFeedback && (
          <div className="flex gap-2">
            <input
              type="text"
              placeholder="Additional feedback on this review..."
              className="flex-1 text-sm border border-gray-300 rounded-md px-3 py-2"
              value={feedback}
              onChange={(e) => setFeedback(e.target.value)}
              onKeyDown={(e) => {
                if (e.key === "Enter" && feedback.trim()) {
                  onSendFeedback(feedback);
                  setFeedback("");
                }
              }}
            />
            <button
              onClick={() => {
                if (feedback.trim()) {
                  onSendFeedback(feedback);
                  setFeedback("");
                }
              }}
              className="px-3 py-2 bg-gray-800 text-white text-sm rounded-md"
            >
              Send
            </button>
          </div>
        )}

        <div className="flex justify-between items-center">
          <span className="text-xs text-gray-500">
            {resolvedIds.size} of {issues.length} issues resolved
          </span>
          <button
            onClick={onApprove}
            className="px-4 py-2 bg-blue-600 text-white text-sm rounded-md hover:bg-blue-700"
          >
            Approve & Continue
          </button>
        </div>
      </div>
    </div>
  );
}
```

#### ApprovalCard.tsx

```tsx
// ============================================================================
// components/generative-ui/ApprovalCard.tsx
// ============================================================================
// The simplest HITL card — presents content and asks for Approve/Reject.
// Uses renderAndWaitForResponse pattern to block agent execution.
//
// USAGE: Agent interrupts with "awaiting_approval" → frontend renders this card
//        User clicks Approve or Reject → respond() called → agent continues
// FILE: components/generative-ui/ApprovalCard.tsx
// ============================================================================

import React, { useState } from "react";

export interface ApprovalCardProps {
  /** What is being approved (e.g., "blog post section", "final draft") */
  subject: string;
  /** Content to approve (truncated preview or full content) */
  content: string;
  /** Context or notes about what the user should check */
  approvalNotes?: string;
  /** Maximum characters to show before truncation */
  previewLength?: number;
  /** Callback: user approves (passes to respond()) */
  onApprove: () => void;
  /** Callback: user rejects with feedback (passes to respond()) */
  onReject: (reason: string) => void;
  /** Whether a response is being sent */
  isSubmitting?: boolean;
  className?: string;
}

/**
 * ApprovalCard: Minimal HITL card for binary approve/reject decisions.
 *
 * HITL PATTERN: This card is rendered when the agent emits an INTERRUPT event.
 * It uses renderAndWaitForResponse under the hood — the agent is blocked
 * until onApprove or onReject is called, which triggers respond().
 *
 * CRITICAL: This card does NOT auto-submit. It requires explicit user action.
 * The agent remains in the "waiting" state until respond() is called.
 */
export function ApprovalCard({
  subject,
  content,
  approvalNotes,
  previewLength = 500,
  onApprove,
  onReject,
  isSubmitting = false,
  className = "",
}: ApprovalCardProps) {
  const [showFullContent, setShowFullContent] = useState(false);
  const [rejectReason, setRejectReason] = useState("");
  const [showRejectForm, setShowRejectForm] = useState(false);

  const truncated = content.length > previewLength && !showFullContent;
  const displayContent = truncated ? content.slice(0, previewLength) + "..." : content;

  return (
    <div className={`bg-white border border-gray-200 rounded-lg shadow-sm ${className}`}>
      {/* Header */}
      <div className="px-4 py-3 border-b border-gray-200 bg-blue-50">
        <div className="flex items-center gap-2">
          <svg className="w-5 h-5 text-blue-600" fill="none" stroke="currentColor" viewBox="0 0 24 24">
            <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M9 12l2 2 4-4m6 2a9 9 0 11-18 0 9 9 0 0118 0z" />
          </svg>
          <h3 className="font-semibold text-gray-900">Approval Required: {subject}</h3>
        </div>
        {approvalNotes && (
          <p className="text-sm text-gray-600 mt-1">{approvalNotes}</p>
        )}
      </div>

      {/* Content preview */}
      <div className="px-4 py-3">
        <div className="bg-gray-50 rounded-md p-3 max-h-64 overflow-y-auto">
          <p className="text-sm text-gray-700 whitespace-pre-wrap leading-relaxed">
            {displayContent}
          </p>
          {truncated && (
            <button
              onClick={() => setShowFullContent(true)}
              className="text-xs text-blue-600 hover:underline mt-2"
            >
              Show full content ({content.length.toLocaleString()} chars)
            </button>
          )}
        </div>
      </div>

      {/* Reject feedback form */}
      {showRejectForm && (
        <div className="px-4 py-3 border-t border-gray-100">
          <label className="text-sm font-medium text-gray-700 block mb-1">
            Reason for rejection:
          </label>
          <textarea
            className="w-full text-sm border border-gray-300 rounded-md px-3 py-2 focus:ring-2 focus:ring-red-500"
            rows={2}
            placeholder="What needs to be changed?"
            value={rejectReason}
            onChange={(e) => setRejectReason(e.target.value)}
            autoFocus
          />
        </div>
      )}

      {/* Action buttons */}
      <div className="px-4 py-3 border-t border-gray-200 flex gap-2">
        <button
          onClick={onApprove}
          disabled={isSubmitting || showRejectForm}
          className="flex-1 px-4 py-2 bg-green-600 text-white rounded-md hover:bg-green-700 disabled:opacity-50 flex items-center justify-center gap-2"
        >
          {isSubmitting ? (
            <>
              <LoadingSpinner size={16} />
              Processing...
            </>
          ) : (
            <>
              <svg className="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M5 13l4 4L19 7" />
              </svg>
              Approve
            </>
          )}
        </button>

        {showRejectForm ? (
          <button
            onClick={() => {
              if (rejectReason.trim()) {
                onReject(rejectReason);
              }
            }}
            disabled={isSubmitting || !rejectReason.trim()}
            className="flex-1 px-4 py-2 bg-red-600 text-white rounded-md hover:bg-red-700 disabled:opacity-50"
          >
            Confirm Rejection
          </button>
        ) : (
          <button
            onClick={() => setShowRejectForm(true)}
            disabled={isSubmitting}
            className="flex-1 px-4 py-2 bg-white border border-gray-300 text-gray-700 rounded-md hover:bg-gray-50 disabled:opacity-50 flex items-center justify-center gap-2"
          >
            <svg className="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
              <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M6 18L18 6M6 6l12 12" />
            </svg>
            Reject
          </button>
        )}
      </div>
    </div>
  );
}

// Small inline spinner for loading states
function LoadingSpinner({ size = 16 }: { size?: number }) {
  return (
    <svg className="animate-spin" width={size} height={size} viewBox="0 0 24 24" fill="none">
      <circle className="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" strokeWidth="4" />
      <path
        className="opacity-75"
        fill="currentColor"
        d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"
      />
    </svg>
  );
}
```

#### ClarificationQuestionCard.tsx

```tsx
// ============================================================================
// components/generative-ui/ClarificationQuestionCard.tsx
// ============================================================================
// When the agent needs more information, it interrupts with a question.
// This card renders the question with quick-reply options or a free-text
// input. Uses renderAndWaitForResponse pattern.
//
// USAGE: Agent interrupts with "ask_clarification" → frontend renders this
//        User answers → respond(answer) → agent continues
// FILE: components/generative-ui/ClarificationQuestionCard.tsx
// ============================================================================

import React, { useState } from "react";

export interface ClarificationOption {
  /** Option label shown to user */
  label: string;
  /** Value sent to agent when selected */
  value: string;
}

export interface ClarificationQuestionCardProps {
  /** The question being asked */
  question: string;
  /** Context explaining why this info is needed */
  context?: string;
  /** Pre-defined quick-reply options (optional) */
  options?: ClarificationOption[];
  /** Whether to allow free-text answer alongside options */
  allowFreeText?: boolean;
  /** Placeholder for free-text input */
  freeTextPlaceholder?: string;
  /** Callback: user selected an option or typed an answer */
  onAnswer: (answer: string) => void;
  /** Whether response is being submitted */
  isSubmitting?: boolean;
  className?: string;
}

/**
 * ClarificationQuestionCard: HITL card for agent questions.
 *
 * PATTERN: renderAndWaitForResponse. The agent emits an INTERRUPT with
 * clarification metadata. The frontend renders this card. The user answers
 * via quick-reply buttons or free text. The answer is sent via respond().
 *
 * QUICK-REPLY OPTIMIZATION: Providing pre-defined options dramatically
 * reduces user friction compared to always requiring typed input. The agent
 * can suggest options based on the conversation context.
 */
export function ClarificationQuestionCard({
  question,
  context,
  options,
  allowFreeText = true,
  freeTextPlaceholder = "Type your answer...",
  onAnswer,
  isSubmitting = false,
  className = "",
}: ClarificationQuestionCardProps) {
  const [freeText, setFreeText] = useState("");
  const [selectedOption, setSelectedOption] = useState<string | null>(null);

  const handleOptionClick = (value: string) => {
    setSelectedOption(value);
    onAnswer(value);
  };

  const handleFreeTextSubmit = () => {
    if (freeText.trim()) {
      onAnswer(freeText.trim());
    }
  };

  return (
    <div className={`bg-white border border-amber-200 rounded-lg shadow-sm ${className}`}>
      {/* Header */}
      <div className="px-4 py-3 border-b border-amber-100 bg-amber-50">
        <div className="flex items-center gap-2">
          <svg className="w-5 h-5 text-amber-600" fill="none" stroke="currentColor" viewBox="0 0 24 24">
            <path
              strokeLinecap="round"
              strokeLinejoin="round"
              strokeWidth={2}
              d="M8.228 9c.549-1.165 2.03-2 3.772-2 2.21 0 4 1.343 4 3 0 1.4-1.278 2.575-3.006 2.907-.542.104-.994.54-.994 1.093m0 3h.01M21 12a9 9 0 11-18 0 9 9 0 0118 0z"
            />
          </svg>
          <h3 className="font-semibold text-gray-900">Question</h3>
        </div>
      </div>

      {/* Question body */}
      <div className="px-4 py-4">
        <p className="text-gray-900 font-medium">{question}</p>
        {context && <p className="text-sm text-gray-500 mt-2">{context}</p>}

        {/* Quick-reply options */}
        {options && options.length > 0 && (
          <div className="flex flex-wrap gap-2 mt-4">
            {options.map((option) => (
              <button
                key={option.value}
                onClick={() => handleOptionClick(option.value)}
                disabled={isSubmitting}
                className={`px-4 py-2 rounded-full text-sm border transition-all ${
                  selectedOption === option.value
                    ? "bg-blue-600 text-white border-blue-600"
                    : "bg-white text-gray-700 border-gray-300 hover:border-blue-400 hover:bg-blue-50"
                } disabled:opacity-50`}
              >
                {option.label}
              </button>
            ))}
          </div>
        )}

        {/* Free-text input */}
        {allowFreeText && (
          <div className="mt-4 flex gap-2">
            <input
              type="text"
              placeholder={freeTextPlaceholder}
              className="flex-1 text-sm border border-gray-300 rounded-md px-3 py-2 focus:ring-2 focus:ring-blue-500 focus:border-transparent"
              value={freeText}
              onChange={(e) => setFreeText(e.target.value)}
              onKeyDown={(e) => {
                if (e.key === "Enter" && !e.shiftKey) {
                  e.preventDefault();
                  handleFreeTextSubmit();
                }
              }}
              disabled={isSubmitting}
            />
            <button
              onClick={handleFreeTextSubmit}
              disabled={isSubmitting || !freeText.trim()}
              className="px-4 py-2 bg-blue-600 text-white text-sm rounded-md hover:bg-blue-700 disabled:opacity-50"
            >
              {isSubmitting ? "Sending..." : "Send"}
            </button>
          </div>
        )}
      </div>
    </div>
  );
}
```

### useFrontendTool + renderAndWaitForResponse Integration

```tsx
// ============================================================================
// hooks/useGenerativeUI.ts — Registers all frontend tools with CopilotKit
// ============================================================================

import { useFrontendTool, useAgent } from "@copilotkit/react-core";
import { useState, useCallback } from "react";

export function useGenerativeUI() {
  const agent = useAgent({ ... });
  const [activeCard, setActiveCard] = useState<string | null>(null);
  const [cardProps, setCardProps] = useState<any>(null);

  // Register "display_transcript" tool
  useFrontendTool({
    name: "display_transcript",
    description: "Display a transcript in the UI for user review",
    parameters: {
      title: { type: "string", description: "Transcript title" },
      sections: {
        type: "array",
        description: "Array of transcript sections",
        items: {
          type: "object",
          properties: {
            id: { type: "string" },
            heading: { type: "string" },
            content: { type: "string" },
            speaker: { type: "string" },
            confidence: { type: "number" },
          },
        },
      },
    },
    handler: async (args) => {
      setActiveCard("transcript");
      setCardProps(args);
      // renderAndWaitForResponse blocks until user interacts
      const userResponse = await renderAndWaitForResponse({
        message: "Please review the transcript and approve or request changes.",
      });
      return userResponse;
    },
  });

  // Register "display_draft" tool
  useFrontendTool({
    name: "display_draft",
    description: "Display a content draft for user approval",
    parameters: {
      title: { type: "string" },
      content: { type: "string" },
      metadata: { type: "object" },
      revision: { type: "number" },
    },
    handler: async (args) => {
      setActiveCard("draft");
      setCardProps(args);
      const userResponse = await renderAndWaitForResponse({
        message: "Please review and approve or request changes.",
      });
      return userResponse;
    },
  });

  // Register "display_review_report" tool
  useFrontendTool({
    name: "display_review_report",
    description: "Display a review report with issues to resolve",
    parameters: {
      title: { type: "string" },
      issues: { type: "array" },
      overallScore: { type: "number" },
      summary: { type: "string" },
    },
    handler: async (args) => {
      setActiveCard("review");
      setCardProps(args);
      const userResponse = await renderAndWaitForResponse({
        message: "Please review the report and approve or provide feedback.",
      });
      return userResponse;
    },
  });

  // Register "ask_clarification" tool
  useFrontendTool({
    name: "ask_clarification",
    description: "Ask the user a clarifying question",
    parameters: {
      question: { type: "string" },
      context: { type: "string" },
      options: { type: "array" },
    },
    handler: async (args) => {
      setActiveCard("clarification");
      setCardProps(args);
      const userResponse = await renderAndWaitForResponse({
        message: args.question,
      });
      return userResponse;
    },
  });

  return { activeCard, cardProps, agent };
}
```

---

## 6. Two SSE Streams Pattern

### Architecture Overview

Our frontend subscribes to **two independent Server-Sent Event streams**:

| Stream | Source | Content | Hook | Purpose |
|--------|--------|---------|------|---------|
| **AG-UI Stream** | AgentCore Runtime (via `@ag-ui/client`) | AG-UI events (text, tools, state) | `useAgent` | Real-time agent→frontend communication |
| **Workflow Stream** | FastAPI backend | Workflow lifecycle events | `useWorkflowStream` | Cross-agent orchestration status |

### Why Two Streams (and Why Not Merge on Backend)

1. **Different protocols**: AG-UI speaks AG-UI protocol; Workflow Stream speaks our custom JSON format
2. **Different sources**: AG-UI comes from AgentCore Runtime; Workflow Stream from our FastAPI
3. **Different lifetimes**: AG-UI is per-agent-invocation; Workflow Stream is per-workflow (multi-agent)
4. **Separation of concerns**: Merging on backend would couple the systems, reducing resilience
5. **

### Implementation: useWorkflowStream Hook

```typescript
// ============================================================================
// hooks/useWorkflowStream.ts — SSE client for Workflow Stream
// ============================================================================
// Subscribes to the FastAPI Workflow Stream endpoint for real-time
// workflow lifecycle updates (agent started, completed, failed, etc.).
//
// FILE: hooks/useWorkflowStream.ts
// ============================================================================

import { useState, useEffect, useRef, useCallback } from "react";

// ---------------------------------------------------------------------------
// Workflow Event Types — Custom protocol from FastAPI backend
// ---------------------------------------------------------------------------
export interface WorkflowEvent {
  /** Event type */
  type: "AGENT_STARTED" | "AGENT_COMPLETED" | "AGENT_FAILED" | "WORKFLOW_STARTED" | "WORKFLOW_COMPLETED" | "WORKFLOW_FAILED" | "STATE_UPDATE";
  /** Workflow ID */
  workflowId: string;
  /** Timestamp (ISO 8601) */
  timestamp: string;
  /** Agent name (for agent events) */
  agentName?: string;
  /** Agent index in sequence */
  agentIndex?: number;
  /** Total agents in workflow */
  totalAgents?: number;
  /** Error message (for failed events) */
  error?: string;
  /** Workflow state snapshot */
  state?: Record<string, any>;
  /** Current stage label */
  stage?: string;
}

export interface WorkflowStreamState {
  /** Current workflow stage */
  stage: string;
  /** Completed agent names */
  completedAgents: string[];
  /** Current agent (if any) */
  currentAgent: string | null;
  /** Overall progress 0-100 */
  progress: number;
  /** Latest error (if any) */
  error: string | null;
  /** Full workflow state */
  workflowState: Record<string, any>;
}

export interface UseWorkflowStreamReturn {
  /** Current workflow state derived from stream events */
  state: WorkflowStreamState;
  /** Whether connected to the stream */
  isConnected: boolean;
  /** Connection error */
  error: Error | null;
  /** Manually connect to a workflow */
  connect: (workflowId: string) => void;
  /** Disconnect from stream */
  disconnect: () => void;
}

/**
 * useWorkflowStream: SSE client for workflow lifecycle events.
 *
 * ARCHITECTURE: This hook creates an EventSource connection to the FastAPI
 * Workflow Stream endpoint. It maintains local state derived from the stream
 * events and provides a unified view of multi-agent workflow progress.
 */
export function useWorkflowStream(
  /** Base URL for the workflow stream API */
  apiBaseUrl: string
): UseWorkflowStreamReturn {
  // Local state derived from workflow events
  const [state, setState] = useState<WorkflowStreamState>({
    stage: "idle",
    completedAgents: [],
    currentAgent: null,
    progress: 0,
    error: null,
    workflowState: {},
  });

  const [isConnected, setIsConnected] = useState(false);
  const [error, setError] = useState<Error | null>(null);

  // Use refs for EventSource and current workflow ID
  const eventSourceRef = useRef<EventSource | null>(null);
  const workflowIdRef = useRef<string | null>(null);

  /**
   * Connect to the workflow stream for a given workflow ID.
   * Automatically disconnects from any previous stream.
   */
  const connect = useCallback(
    (workflowId: string) => {
      // Disconnect from previous stream if any
      disconnect();

      workflowIdRef.current = workflowId;

      // Reset state
      setState({
        stage: "connecting",
        completedAgents: [],
        currentAgent: null,
        progress: 0,
        error: null,
        workflowState: {},
      });

      // Create EventSource connection
      // The endpoint is a GET endpoint that returns SSE events
      const url = `${apiBaseUrl}/api/workflows/${workflowId}/stream`;

      try {
        const es = new EventSource(url);
        eventSourceRef.current = es;

        // Connection opened
        es.onopen = () => {
          console.log(`[WorkflowStream] Connected to ${workflowId}`);
          setIsConnected(true);
          setError(null);
        };

        // Message received
        es.onmessage = (event) => {
          try {
            const data: WorkflowEvent = JSON.parse(event.data);
            handleWorkflowEvent(data);
          } catch (parseError) {
            console.error("[WorkflowStream] Failed to parse event:", parseError);
          }
        };

        // Error occurred
        es.onerror = (err) => {
          console.error("[WorkflowStream] SSE error:", err);
          setError(new Error("Workflow stream connection failed"));
          setIsConnected(false);
        };
      } catch (err) {
        setError(err instanceof Error ? err : new Error(String(err)));
      }
    },
    [apiBaseUrl]
  );

  /**
   * Disconnect from the current stream.
   */
  const disconnect = useCallback(() => {
    if (eventSourceRef.current) {
      eventSourceRef.current.close();
      eventSourceRef.current = null;
    }
    workflowIdRef.current = null;
    setIsConnected(false);
  }, []);

  /**
   * Process incoming workflow events and update local state.
   */
  const handleWorkflowEvent = useCallback((event: WorkflowEvent) => {
    switch (event.type) {
      case "WORKFLOW_STARTED":
        setState((prev) => ({
          ...prev,
          stage: event.stage || "in_progress",
          progress: 0,
        }));
        break;

      case "AGENT_STARTED":
        setState((prev) => ({
          ...prev,
          currentAgent: event.agentName || null,
          stage: event.stage || `${event.agentName}_running`,
        }));
        break;

      case "AGENT_COMPLETED":
        setState((prev) => {
          const completed = [...prev.completedAgents, event.agentName || ""].filter(Boolean);
          const total = event.totalAgents || 1;
          return {
            ...prev,
            completedAgents: completed,
            currentAgent: null,
            progress: Math.round((completed.length / total) * 100),
            stage: event.stage || `${event.agentName}_completed`,
          };
        });
        break;

      case "AGENT_FAILED":
        setState((prev) => ({
          ...prev,
          currentAgent: null,
          error: event.error || "Agent failed",
          stage: `${event.agentName}_failed`,
        }));
        break;

      case "WORKFLOW_COMPLETED":
        setState((prev) => ({
          ...prev,
          progress: 100,
          stage: "completed",
          currentAgent: null,
        }));
        // Auto-disconnect on completion
        disconnect();
        break;

      case "WORKFLOW_FAILED":
        setState((prev) => ({
          ...prev,
          error: event.error || "Workflow failed",
          stage: "failed",
          currentAgent: null,
        }));
        break;

      case "STATE_UPDATE":
        setState((prev) => ({
          ...prev,
          workflowState: { ...prev.workflowState, ...(event.state || {}) },
        }));
        break;
    }
  }, [disconnect]);

  // Cleanup on unmount
  useEffect(() => {
    return () => {
      disconnect();
    };
  }, [disconnect]);

  return { state, isConnected, error, connect, disconnect };
}
```

### Merging Both Streams in the Canvas Component

```tsx
// ============================================================================
// components/Canvas.tsx — Unified canvas merging both SSE streams
// ============================================================================

import React, { useEffect, useMemo } from "react";
import { useAgent } from "@copilotkit/react-core";
import { useWorkflowStream } from "@/hooks/useWorkflowStream";
import { TranscriptCard, DraftPreviewCard, ReviewReportCard, ApprovalCard, ClarificationQuestionCard } from "./generative-ui";

interface CanvasProps {
  workflowId?: string;
}

/**
 * Canvas: The main display area that merges AG-UI events with Workflow Stream
 * events into a unified user experience.
 *
 * STREAM MERGING STRATEGY:
 * - AG-UI Stream (via useAgent): Provides real-time agent output (text, state, HITL)
 * - Workflow Stream (via useWorkflowStream): Provides orchestration status
 * - The Canvas component merges these into a single coherent view
 */
export function Canvas({ workflowId }: CanvasProps) {
  // Stream 1: AG-UI events from the agent
  const agent = useAgent({ id: "content-generation-agent" });

  // Stream 2: Workflow lifecycle events from FastAPI
  const workflow = useWorkflowStream(process.env.NEXT_PUBLIC_API_URL!);

  // Connect to workflow stream when workflowId is available
  useEffect(() => {
    if (workflowId) {
      workflow.connect(workflowId);
    }
    return () => workflow.disconnect();
  }, [workflowId]);

  // Derive unified state from both streams
  const unifiedState = useMemo(() => {
    // Priority: AG-UI state takes precedence for content,
    // Workflow state provides orchestration context
    return {
      // Content from AG-UI
      agentState: agent.state,
      messages: agent.messages,
      status: agent.status,
      isWaitingForInput: agent.renderWaitForResponse,

      // Orchestration from Workflow Stream
      workflowProgress: workflow.state.progress,
      workflowStage: workflow.state.stage,
      completedAgents: workflow.state.completedAgents,
      currentAgent: workflow.state.currentAgent,
      workflowError: workflow.state.error,

      // Combined error
      error: agent.error || workflow.state.error,
    };
  }, [agent, workflow.state]);

  return (
    <div className="flex flex-col h-full">
      {/* Progress bar from Workflow Stream */}
      {workflow.state.progress > 0 && (
        <div className="w-full h-1 bg-gray-200">
          <div
            className="h-full bg-blue-600 transition-all duration-500"
            style={{ width: `${workflow.state.progress}%` }}
          />
        </div>
      )}

      {/* Workflow stage indicator */}
      <div className="px-4 py-2 bg-gray-50 border-b text-sm text-gray-600 flex items-center gap-2">
        <span className="font-medium">Stage:</span> {unifiedState.workflowStage}
        {unifiedState.currentAgent && (
          <>
            <span className="text-gray-300">|</span>
            <span>Current: {unifiedState.currentAgent}</span>
          </>
        )}
        {workflow.isConnected && (
          <span className="ml-auto flex items-center gap-1 text-green-600 text-xs">
            <span className="w-2 h-2 bg-green-500 rounded-full animate-pulse" />
            Live
          </span>
        )}
      </div>

      {/* Main content area — renders generative UI cards based on agent state */}
      <div className="flex-1 overflow-y-auto p-4 space-y-4">
        {/* Render based on active card from useGenerativeUI */}
        {agent.state?.activeCard === "transcript" && (
          <TranscriptCard {...agent.state.cardProps} />
        )}

        {agent.state?.activeCard === "draft" && (
          <DraftPreviewCard {...agent.state.cardProps} />
        )}

        {agent.state?.activeCard === "review" && (
          <ReviewReportCard {...agent.state.cardProps} />
        )}

        {/* HITL cards rendered conditionally */}
        {agent.renderWaitForResponse && agent.state?.hitlType === "approval" && (
          <ApprovalCard
            subject={agent.state.hitlSubject}
            content={agent.state.contentToApprove}
            onApprove={() => agent.respond("APPROVED")}
            onReject={(reason) => agent.respond(`REJECTED: ${reason}`)}
          />
        )}

        {agent.renderWaitForResponse && agent.state?.hitlType === "clarification" && (
          <ClarificationQuestionCard
            question={agent.state.question}
            options={agent.state.options}
            onAnswer={(answer) => agent.respond(answer)}
          />
        )}

        {/* Fallback: raw messages */}
        {agent.messages.length > 0 && !agent.state?.activeCard && (
          <div className="space-y-2">
            {agent.messages.map((msg) => (
              <div key={msg.id} className={`p-3 rounded-lg ${msg.isUser ? "bg-blue-50 ml-8" : "bg-gray-50 mr-8"}`}>
                <p className="text-sm text-gray-800">{msg.content}</p>
              </div>
            ))}
          </div>
        )}
      </div>

      {/* Status bar */}
      <div className="px-4 py-2 border-t bg-gray-50 text-xs text-gray-500 flex justify-between">
        <span>Agent: {agent.status}</span>
        <span>Progress: {workflow.state.progress}%</span>
      </div>
    </div>
  );
}
```

---

## 7. Threads & Sidebar History

### Lightweight Threads Implementation

Our demo uses a lightweight threads implementation backed by an RDS `chat_messages` table. Each thread is a conversation between the user and the agent, stored persistently.

### Database Schema

```sql
-- ============================================================================
-- RDS Schema for Chat Messages (Thread History)
-- ============================================================================

CREATE TABLE chat_threads (
    thread_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         TEXT NOT NULL,
    workflow_id     TEXT,              -- Links thread to Temporal workflow
    title           TEXT,              -- Auto-generated from first message
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW(),
    status          TEXT DEFAULT 'active',  -- active, archived, deleted
    metadata        JSONB DEFAULT '{}'
);

CREATE INDEX idx_threads_user ON chat_threads(user_id, updated_at DESC);
CREATE INDEX idx_threads_workflow ON chat_threads(workflow_id);

CREATE TABLE chat_messages (
    message_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    thread_id       UUID NOT NULL REFERENCES chat_threads(thread_id) ON DELETE CASCADE,
    role            TEXT NOT NULL CHECK (role IN ('user', 'assistant', 'system', 'tool')),
    content         TEXT NOT NULL,
    -- AG-UI event metadata for reconstruction
    event_type      TEXT,              -- TEXT_MESSAGE_START/CONTENT/END, etc.
    event_metadata  JSONB DEFAULT '{}',
    -- For tool calls
    tool_name       TEXT,
    tool_arguments  JSONB,
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_messages_thread ON chat_messages(thread_id, created_at);
```

### React Components

```tsx
// ============================================================================
// components/ThreadSidebar.tsx — Collapsible sidebar with thread history
// ============================================================================

import React, { useState, useEffect } from "react";

export interface ChatThread {
  threadId: string;
  title: string;
  updatedAt: string;
  messageCount: number;
  status: string;
}

interface ThreadSidebarProps {
  userId: string;
  activeThreadId?: string;
  onSelectThread: (threadId: string) => void;
  onNewThread: () => void;
  onDeleteThread: (threadId: string) => void;
}

export function ThreadSidebar({
  userId,
  activeThreadId,
  onSelectThread,
  onNewThread,
  onDeleteThread,
}: ThreadSidebarProps) {
  const [threads, setThreads] = useState<ChatThread[]>([]);
  const [isLoading, setIsLoading] = useState(true);
  const [isCollapsed, setIsCollapsed] = useState(false);

  // Fetch threads on mount
  useEffect(() => {
    fetchThreads();
  }, [userId]);

  const fetchThreads = async () => {
    setIsLoading(true);
    try {
      const res = await fetch(`/api/threads?userId=${userId}`);
      const data = await res.json();
      setThreads(data.threads);
    } catch (err) {
      console.error("Failed to fetch threads:", err);
    } finally {
      setIsLoading(false);
    }
  };

  if (isCollapsed) {
    return (
      <div className="w-10 border-r bg-gray-50 flex flex-col items-center py-4">
        <button
          onClick={() => setIsCollapsed(false)}
          className="p-2 hover:bg-gray-200 rounded"
          title="Expand sidebar"
        >
          <svg className="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
            <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M13 5l7 7-7 7M5 5l7 7-7 7" />
          </svg>
        </button>
      </div>
    );
  }

  return (
    <div className="w-64 border-r bg-gray-50 flex flex-col">
      {/* Header */}
      <div className="px-3 py-3 border-b flex items-center justify-between">
        <h2 className="font-semibold text-gray-800">History</h2>
        <button
          onClick={() => setIsCollapsed(true)}
          className="p-1 hover:bg-gray-200 rounded"
        >
          <svg className="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
            <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M11 19l-7-7 7-7m8 14l-7-7 7-7" />
          </svg>
        </button>
      </div>

      {/* New thread button */}
      <div className="px-3 py-2">
        <button
          onClick={onNewThread}
          className="w-full px-3 py-2 bg-blue-600 text-white rounded-md text-sm hover:bg-blue-700 flex items-center justify-center gap-2"
        >
          <svg className="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
            <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M12 4v16m8-8H4" />
          </svg>
          New Chat
        </button>
      </div>

      {/* Thread list */}
      <div className="flex-1 overflow-y-auto">
        {isLoading ? (
          <div className="p-4 text-center text-sm text-gray-400">Loading...</div>
        ) : threads.length === 0 ? (
          <div className="p-4 text-center text-sm text-gray-400">No conversations yet</div>
        ) : (
          threads.map((thread) => (
            <button
              key={thread.threadId}
              onClick={() => onSelectThread(thread.threadId)}
              className={`w-full text-left px-3 py-2 hover:bg-gray-100 border-b border-gray-100 group ${
                activeThreadId === thread.threadId ? "bg-blue-50 border-l-4 border-l-blue-600" : "border-l-4 border-l-transparent"
              }`}
            >
              <div className="flex items-center justify-between">
                <span className="text-sm font-medium text-gray-800 truncate flex-1">
                  {thread.title || "Untitled Chat"}
                </span>
                <button
                  onClick={(e) => {
                    e.stopPropagation();
                    onDeleteThread(thread.threadId);
                  }}
                  className="opacity-0 group-hover:opacity-100 p-1 hover:bg-red-100 rounded text-gray-400 hover:text-red-600"
                >
                  <svg className="w-3 h-3" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                    <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M6 18L18 6M6 6l12 12" />
                  </svg>
                </button>
              </div>
              <div className="flex items-center justify-between mt-1">
                <span className="text-xs text-gray-400">{thread.messageCount} messages</span>
                <span className="text-xs text-gray-400">
                  {new Date(thread.updatedAt).toLocaleDateString()}
                </span>
              </div>
            </button>
          ))
        )}
      </div>
    </div>
  );
}
```

### Restoring Thread History into CopilotKit

```tsx
// ============================================================================
// hooks/useThreadHistory.ts — Loads thread messages into CopilotKit
// ============================================================================

import { useState, useEffect, useCallback } from "react";
import { useCopilotChat } from "@copilotkit/react-core";

export interface ThreadMessage {
  messageId: string;
  role: "user" | "assistant";
  content: string;
  createdAt: string;
}

/**
 * useThreadHistory: Fetches persisted thread messages and injects them
 * into CopilotKit's chat state so the user sees their full conversation history.
 */
export function useThreadHistory(threadId: string | undefined) {
  const chat = useCopilotChat();
  const [isLoading, setIsLoading] = useState(false);

  const loadHistory = useCallback(async () => {
    if (!threadId) {
      chat.setMessages([]); // Clear messages for new thread
      return;
    }

    setIsLoading(true);
    try {
      const res = await fetch(`/api/threads/${threadId}/messages`);
      const data: { messages: ThreadMessage[] } = await res.json();

      // Convert to CopilotKit message format and set
      const copilotMessages = data.messages.map((msg) => ({
        id: msg.messageId,
        content: msg.content,
        role: msg.role,
        createdAt: msg.createdAt,
      }));

      chat.setMessages(copilotMessages);
    } catch (err) {
      console.error("Failed to load thread history:", err);
    } finally {
      setIsLoading(false);
    }
  }, [threadId]);

  useEffect(() => {
    loadHistory();
  }, [loadHistory]);

  return { isLoading, reload: loadHistory };
}
```

---

## 8. Page Refresh & State Restoration

### What Survives a Refresh

| Data | Storage | Survives Refresh? | Recovery Method |
|------|---------|-------------------|-----------------|
| Chat history | RDS `chat_messages` | **Yes** | `useThreadHistory` loads from DB |
| Thread list | RDS `chat_threads` | **Yes** | `ThreadSidebar` fetches from API |
| Workflow state | RDS + Temporal | **Partial** | Workflow Stream reconnects |
| Canvas content | RDS (agent state snapshots) | **Yes** | `STATE_SNAPSHOT` replay |
| In-flight tokens | Memory only | **No** | Lost — stream resets |
| Ephemeral UI | React state | **No** | Regenerated from agent state |
| Agent run status | Memory only | **No** | Run must be restarted |
| Workflow progress | Workflow Stream | **Recovered** | Reconnect to stream |

### Recovery Procedure

```tsx
// ============================================================================
// components/RefreshRecovery.tsx — Post-refresh state restoration
// ============================================================================

import { useEffect, useState } from "react";
import { useAgent } from "@copilotkit/react-core";
import { useWorkflowStream } from "@/hooks/useWorkflowStream";
import { useThreadHistory } from "@/hooks/useThreadHistory";

/**
 * RefreshRecovery: Orchestrates state restoration after a page refresh.
 *
 * RECOVERY STRATEGY:
 * 1. If URL has ?threadId=xxx → load that thread's chat history
 * 2. If URL has ?workflowId=xxx → reconnect to Workflow Stream
 * 3. If agent was mid-run → inform user they need to restart
 */
export function RefreshRecovery({
  threadId,
  workflowId,
}: {
  threadId?: string;
  workflowId?: string;
}) {
  const agent = useAgent({ ... });
  const workflow = useWorkflowStream("...");
  const [recoveryStatus, setRecoveryStatus] = useState<"recovering" | "done" | "needs_restart">("recovering");

  // Step 1: Load thread history
  useThreadHistory(threadId);

  // Step 2: Reconnect to workflow stream
  useEffect(() => {
    if (workflowId) {
      workflow.connect(workflowId);
      setRecoveryStatus("done");
    }
  }, [workflowId]);

  // Step 3: Check if we need to restart
  useEffect(() => {
    if (agent.status !== "idle" && !workflowId) {
      // Agent was running but we have no workflow context
      setRecoveryStatus("needs_restart");
    }
  }, [agent.status, workflowId]);

  if (recoveryStatus === "needs_restart") {
    return (
      <div className="p-4 bg-yellow-50 border border-yellow-200 rounded-md">
        <p className="text-yellow-800 font-medium">Session Interrupted</p>
        <p className="text-yellow-700 text-sm mt-1">
          The page was refreshed during an active session.
          Your chat history is preserved, but the current task needs to be restarted.
        </p>
        <button
          onClick={() => agent.reset()}
          className="mt-2 px-4 py-2 bg-yellow-600 text-white text-sm rounded hover:bg-yellow-700"
        >
          Restart Task
        </button>
      </div>
    );
  }

  return null; // Recovery is transparent when successful
}
```

---

## 9. CopilotKit + Tailwind Conflict Workarounds

### The Issue (GitHub #1857)

CopilotKit's default UI components (`CopilotSidebar`, `CopilotChat`) ship with their own CSS that conflicts with Tailwind CSS utility classes. Symptoms include:
- CopilotKit styles overriding Tailwind utilities
- Component layout breaking
- Typography inconsistencies
- Button styling conflicts

### Solutions (in order of preference)

#### Option 1: Headless UI Mode (Recommended)

Use CopilotKit's **headless mode** — consume the hooks directly and build your own UI with Tailwind:

```tsx
// ============================================================================
// Headless UI Mode: Use CopilotKit hooks, build UI with Tailwind
// ============================================================================

// DO NOT import: import "@copilotkit/react-ui/styles.css";
// Instead, build all UI yourself using Tailwind:

import { useAgent, useCopilotChat, useFrontendTool } from "@copilotkit/react-core";

// Custom chat UI built with Tailwind — zero CopilotKit CSS conflicts
function CustomChatInterface() {
  const agent = useAgent({ ... });
  const chat = useCopilotChat();
  const [input, setInput] = useState("");

  return (
    <div className="flex flex-col h-full bg-white border border-gray-200 rounded-lg">
      {/* Messages */}
      <div className="flex-1 overflow-y-auto p-4 space-y-4">
        {chat.messages.map((msg) => (
          <div key={msg.id} className={`flex ${msg.isUser ? "justify-end" : "justify-start"}`}>
            <div className={`max-w-[80%] px-4 py-2 rounded-lg ${
              msg.isUser ? "bg-blue-600 text-white" : "bg-gray-100 text-gray-800"
            }`}>
              {msg.content}
            </div>
          </div>
        ))}
      </div>

      {/* Input */}
      <div className="p-4 border-t">
        <div className="flex gap-2">
          <input
            value={input}
            onChange={(e) => setInput(e.target.value)}
            onKeyDown={(e) => e.key === "Enter" && chat.sendMessage(input)}
            className="flex-1 border rounded-lg px-4 py-2 focus:ring-2 focus:ring-blue-500"
            placeholder="Type a message..."
          />
          <button
            onClick={() => chat.sendMessage(input)}
            className="px-4 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700"
          >
            Send
          </button>
        </div>
      </div>
    </div>
  );
}
```

#### Option 2: CSS Isolation

If you need CopilotKit's pre-built components, isolate them:

```tsx
// Wrap CopilotKit UI in a shadow DOM or iframe to isolate styles
// Or use Tailwind's important modifier in your config:

// tailwind.config.ts
export default {
  // Force Tailwind utilities to win specificity battles
  important: '#app',
  // ... rest of config
};
```

#### Option 3: Custom CSS Overrides (Demo Workaround)

Our demo uses targeted CSS to fix the most common conflicts:

```css
/* ============================================================================
   styles/copilotkit-tailwind-fixes.css
   Targeted fixes for CopilotKit + Tailwind conflicts
   ============================================================================ */

/* Override CopilotKit's button styles with Tailwind-compatible ones */
.copilot-kit button {
  @apply px-4 py-2 rounded-md;
}

.copilot-kit button.copilot-kit-primary {
  @apply bg-blue-600 text-white hover:bg-blue-700;
}

/* Fix CopilotKit input to match Tailwind form styles */
.copilot-kit input[type="text"],
.copilot-kit textarea {
  @apply border border-gray-300 rounded-md px-3 py-2 focus:ring-2 focus:ring-blue-500;
}

/* Fix message bubble colors */
.copilot-kit .message-user {
  @apply bg-blue-600 text-white;
}

.copilot-kit .message-assistant {
  @apply bg-gray-100 text-gray-800;
}

/* Ensure CopilotKit doesn't override our Tailwind typography */
.copilot-kit * {
  font-family: inherit;
}
```

#### Option 4: Style Precedence Hacks

```tsx
// Force Tailwind classes to take precedence with !important
<div className="!bg-blue-600 !text-white !rounded-lg !p-4">
  {/* CopilotKit styles can't override these */}
</div>
```

---

## 10. Reusability Patterns for Future Projects

### Extractable Starter Kit

The following components and hooks are **framework-agnostic** and can be extracted as a starter kit for any CopilotKit + AG-UI + AgentCore project:

#### Core Framework Files (Copy to any project)

| File | Location | Notes |
|------|----------|-------|
| `AgentCoreHttpAgent` | `lib/agentcore/agentcore-agui-client.ts` | Works with any Strands agent on AgentCore |
| `useAgentCoreHttpAgent` | `lib/agentcore/agentcore-agui-client.ts` | React hook wrapper |
| `useWorkflowStream` | `hooks/useWorkflowStream.ts` | Generic SSE workflow stream consumer |
| `useDualResponse` | `hooks/use-dual-response.ts` | Any HITL requiring dual-response |
| `useThreadHistory` | `hooks/useThreadHistory.ts` | Generic thread history loader |
| `TranscriptCard` | `components/generative-ui/TranscriptCard.tsx` | Reusable with any transcript data |
| `DraftPreviewCard` | `components/generative-ui/DraftPreviewCard.tsx` | Reusable for any content draft |
| `ReviewReportCard` | `components/generative-ui/ReviewReportCard.tsx` | Reusable for any review workflow |
| `ApprovalCard` | `components/generative-ui/ApprovalCard.tsx` | Generic binary HITL pattern |
| `ClarificationQuestionCard` | `components/generative-ui/ClarificationQuestionCard.tsx` | Generic Q&A HITL pattern |
| `Canvas` | `components/Canvas.tsx` | Template for dual-stream merging |
| `ThreadSidebar` | `components/ThreadSidebar.tsx` | Generic thread history sidebar |

#### Project-Specific Files (Customize per project)

| File | Why it's project-specific |
|------|--------------------------|
| `useAuth` (
| Tool handlers in `useGenerativeUI` | Domain-specific tools |
| Agent state shape | Different for each agent workflow |
| Database schema | May vary per project |
| Workflow event types | Project-specific workflow stages |
| Temporal client config | Namespace/workflow naming |

### Starter Kit File Structure

```
copilotkit-agui-starter/
  lib/
    agentcore/
      agentcore-agui-client.ts    # HttpAgent subclass + factory + hook
  hooks/
    use-workflow-stream.ts        # SSE workflow stream consumer
    use-dual-response.ts          # HITL dual-response pattern
    use-thread-history.ts         # Thread history loader
  components/
    generative-ui/
      TranscriptCard.tsx          # Reusable transcript display
      DraftPreviewCard.tsx        # Reusable draft preview
      ReviewReportCard.tsx        # Reusable review report
      ApprovalCard.tsx            # Generic approval HITL
      ClarificationQuestionCard.tsx # Generic Q&A HITL
    Canvas.tsx                    # Dual-stream canvas template
    ThreadSidebar.tsx             # Thread history sidebar
  styles/
    copilotkit-tailwind-fixes.css  # Tailwind conflict fixes
```

---

## 11. Tech Reference

### CopilotKit Resources

| Resource | URL |
|----------|-----|
| Official Docs | https://docs.copilotkit.ai |
| useAgent hook | `@copilotkit/react-core` — AG-UI native (v1.50+) |
| useCoAgent hook | `@copilotkit/react-core` — LangGraph-specific |
| useFrontendTool | `@copilotkit/react-core` — Register frontend tools |
| useCopilotChat | `@copilotkit/react-core` — Chat message management |
| renderAndWaitForResponse | `@copilotkit/react-core` — HITL blocking call |
| useHumanInTheLoop | https://docs.copilotkit.ai/reference/v2/hooks/useHumanInTheLoop |
| React UI Components | `@copilotkit/react-ui` — Pre-built chat/sidebar |

### AG-UI Protocol Resources

| Resource | URL |
|----------|-----|
| AG-UI Docs | https://docs.ag-ui.com/ |
| Event Types Reference | https://www.copilotkit.ai/blog/master-the-17-ag-ui-event-types-for-building-agents-the-right-way |
| Backend Integration | https://docs.copilotkit.ai/backend/ag-ui |
| @ag-ui/client | `npm install @ag-ui/client` — HttpAgent base class |

### AWS Strands + AgentCore Resources

| Resource | URL |
|----------|-----|
| Strands + AG-UI Announcement | https://www.copilotkit.ai/blog/aws-strands-agents-now-compatible-with-ag-ui |
| AgentCore AG-UI Endpoint | https://www.copilotkit.ai/blog/aws-announces-dedicated-ag-ui-endpoint-in-agentcore-and-fast-template-for-building-fullstack-agents |
| Strands Tutorial | https://dev.to/copilotkit/easily-build-a-frontend-for-your-aws-strands-agents-using-ag-ui-in-30-minutes-42ji |
| Strands + AgentCore Setup | https://medium.com/@luke.marrai/setting-up-copilotkit-and-ag-ui-with-strands-agents-and-amazon-bedrock-agentcore-e1059e0dfc3e |

### Generative UI Resources

| Resource | URL |
|----------|-----|
| Generative UI Guide 2026 | https://www.copilotkit.ai/blog/the-developer-s-guide-to-generative-ui-in-2026 |

### Version Pins

```json
{
  "dependencies": {
    "@copilotkit/react-core": "^1.50.0",
    "@copilotkit/react-ui": "^1.50.0",
    "@ag-ui/client": "^0.1.0",
    "fast-json-patch": "^3.1.1",
    "temporalio": "^1.11.0"
  }
}
```

### Key Gotchas Checklist

- [ ] `useAgent` (not `useCoAgent`) for AgentCore runtimes
- [ ] Session ID = `workflow_id` for microVM sharing
- [ ]
- [ ] AG-UI events are ephemeral — persist what matters to RDS
- [ ] Always use `Promise.allSettled` for dual-response pattern
- [ ] Handle AG-UI stream errors gracefully — they can drop mid-run
- [ ] Workflow Stream auto-reconnect on disconnect (EventSource does this)
- [ ] Thread history must be loaded before CopilotKit chat initializes
- [ ] Headless UI mode avoids all Tailwind conflicts
- [ ] Each Strands agent is a separate AgentCore Runtime deployment
- [ ] `renderAndWaitForResponse` must be called within a `useFrontendTool` handler

---

*End of Deliverable 5: CopilotKit + AG-UI Integration & Reusability Guide*
