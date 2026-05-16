# Deliverable 8: Frontend Architecture Guide

## Coexistence of Agentic UI (CopilotKit/AG-UI) and Standard React in a Next.js Application

---

**Document ID:** DELIV-008-ARCH  
**Version:** 1.0  
**Date:** 2025-01-20  
**Status:** BASELINE  
**Classification:** Architecture Decision Record + Implementation Guide  
**Scope:** Frontend architecture for dual-realm (Agentic + Standard) React application  
**Upstream Dependencies:** DELIV-000-PCD, DELIV-005-CopilotKit+AG-UI  
**Target:** Next.js 14+ App Router, React 18+, Tailwind CSS, CopilotKit >=1.50
> **V5 (2026-05-12):** Mock auth only (`demo-user-001`). No Cognito/OAuth integration. Dual-stream architecture: AG-UI (AgentCore → Browser direct) + Workflow Stream (Temporal → FastAPI → Browser). A2A is backend-only via Temporal signals.  

---

## Table of Contents

1. [Two Realms of the Frontend](#1-two-realms-of-the-frontend)
2. [Where Each Realm Begins and Ends](#2-where-each-realm-begins-and-ends)
3. [Provider Layout Strategy](#3-provider-layout-strategy)
4. [Routing Strategy](#4-routing-strategy)
5. [Data Fetching — Two Different Patterns](#5-data-fetching--two-different-patterns)
6. [Shared State Between Realms](#6-shared-state-between-realms)
7. [When NOT to Use CopilotKit — Decision Framework](#7-when-not-to-use-copilotkit--decision-framework)
8. [Component Library Strategy](#8-component-library-strategy)
9. [Testing Strategy for Each Realm](#9-testing-strategy-for-each-realm)
10. [Refresh Behavior in Each Realm](#10-refresh-behavior-in-each-realm)
11. [Key Gotchas & Conflict Resolution](#11-key-gotchas--conflict-resolution)
12. [Tech Reference](#12-tech-reference)

---

## 1. Two Realms of the Frontend

The application is divided into **two architectural realms** that coexist in the same Next.js codebase. They share the same build pipeline, CSS framework, and deployment target, but they follow fundamentally different data-fetching patterns, state models, and user interaction paradigms.

### 1.1 Realm Overview Diagram

```
+--------------------------------------------------------------------------+
|                         NEXT.JS APP ROUTER                               |
|                                                                          |
|  +-----------------------------+    +----------------------------------+ |
|  |     STANDARD REALM          |    |        AGENTIC REALM             | |
|  |     (Non-Agentic UI)        |    |        (CopilotKit + AG-UI)      | |
|  |                             |    |                                  | |
|  |  Pages:                     |    |  Pages:                          | |
|  |  - /  (landing)             |    |  - /workspace/[wfId]             | |
|  |  - /dashboard               |    |                                  | |
|  |  - /settings                |    |  Surfaces:                       | |
|  |                             |    |  - CopilotChat sidebar           | |
|  |  Components:                |    |  - Canvas (generative cards)     | |
|  |  - MetricTile               |    |  - HITL question dialogs         | |
|  |  - SidebarHistory           |    |  - Evidence pack viewer          | |
|  |  - DataTable                |    |                                  | |
|  |  - UploadForm               |    |  Components:                     | |
|  |                             |    |  - TranscriptCard                | |
|  |  Data: REST GET/POST        |    |  - DraftPreviewCard              | |
|  |  Cache: React Query / SWR   |    |  - ReviewReportCard              | |
|  |                             |    |  - HITLResponseCard              | |
|  |  State: Local + Context     |    |                                  | |
|  |                             |    |  Data: AG-UI SSE (direct)        | |
|  |                             |    |        Workflow Stream SSE       | |
|  |                             |    |  Cache: None — streams ARE data  | |
|  |                             |    |                                  | |
|  |                             |    |  State: CanvasProvider + hooks   | |
|  +-----------------------------+    +----------------------------------+ |
|                                                                          |
|  SHARED LAYER (both realms use):                                         |
|  - components/ui/  (Button, Card, Tile — shared primitives)              |
|  - Tailwind CSS utility classes                                          |
|  - React Context for cross-realm bits (workflow_id, user identity)       |
|  - Same Next.js Layout root (but different providers)                    |
|                                                                          |
+--------------------------------------------------------------------------+
```

### 1.2 Standard Realm (Non-Agentic UI)

The **Standard Realm** is conventional React. It follows patterns every React developer knows:

- **Data fetching:** REST API calls to FastAPI (`fetch`, `axios`, React Query, SWR)
- **State management:** Local component state (`useState`) + shared React Context
- **Rendering:** Static JSX driven by fetched data
- **Interactivity:** Form submissions, button clicks, navigation via `next/link`
- **Caching:** React Query caches REST responses; SWR provides stale-while-revalidate
- **Philosophy:** User views pre-computed data; agent is NOT involved

**Pages in this realm:**

| Page | Route | Purpose |
|------|-------|---------|
| Landing Page | `/` | Marketing page, upload entry point, feature overview |
| Dashboard | `/dashboard` | Workflow history metrics, completion stats |
| Settings | `/settings` | User preferences, configuration (mock identity) |
| Workflow List | `/workflows` | Tabular list of past workflow executions |

### 1.3 Agentic Realm (CopilotKit + AG-UI)

The **Agentic Realm** is where the AI agent lives. It follows a fundamentally different model:

- **Data fetching:** TWO Server-Sent Event (SSE) streams — AG-UI stream from AgentCore, Workflow Stream from FastAPI
- **State management:** Stream-driven state — events from SSE streams ARE the state mutations
- **Rendering:** Generative UI — components rendered dynamically based on agent tool calls
- **Interactivity:** Conversational (chat), Human-in-the-Loop (HITL) question/response cycles
- **Caching:** NO React Query — streams are ephemeral and cannot be cached meaningfully
- **Philosophy:** User interacts WITH the agent; agent generates content and drives UI

**Pages in this realm:**

| Page | Route | Purpose |
|------|-------|---------|
| Workspace | `/workspace/[wfId]` | Main agent interaction surface — chat + canvas |

### 1.4 Why Two Realms?

The realms are separated because they solve different problems:

| Aspect | Standard Realm | Agentic Realm |
|--------|---------------|---------------|
| **User goal** | View data, navigate, configure | Collaborate with agent, review generated content |
| **Data source** | Pre-computed database records | Real-time agent-generated content |
| **Latency model** | Tolerates 100-500ms fetch | Requires <100ms stream updates |
| **State model** | Fetch → cache → render | Stream → event → state mutation → render |
| **Interaction** | Click → load → view | Chat → think → generate → HITL → approve |
| **CopilotKit?** | No — adds unnecessary overhead | Yes — provides AG-UI integration |

**Critical insight:** Not every page that displays agent-related data belongs in the Agentic Realm. The Dashboard page shows workflow completion statistics (which come from agent runs), but it is a **Standard Realm page** because the data is pre-computed and stored in RDS. There is no agent interaction on that page.

---

## 2. Where Each Realm Begins and Ends

This section provides a **file-by-file map** of the repository. Every file is tagged with its realm ownership. This is the single source of truth for where CopilotKit hooks may (and may not) be used.

### 2.1 File-to-Realm Map

```
my-app/
|
|-- app/                                    # Next.js App Router
|   |-- layout.tsx                          # STANDARD — root layout (NO CopilotKit)
|   |-- page.tsx                            # STANDARD — landing page
|   |-- globals.css                         # SHARED — Tailwind + global styles
|   |
|   |-- dashboard/
|   |   |-- page.tsx                        # STANDARD — metrics dashboard
|   |
|   |-- settings/
|   |   |-- page.tsx                        # STANDARD — user configuration
|   |
|   |-- workflows/
|   |   |-- page.tsx                        # STANDARD — workflow history list
|   |
|   |-- workspace/
|   |   |-- [wfId]/
|   |   |   |-- layout.tsx                  # HYBRID — CopilotKit provider HERE
|   |   |   |-- page.tsx                    # AGENTIC — workspace main page
|   |   |   |-- loading.tsx                 # SHARED — loading skeleton
|   |
|-- components/
|   |-- ui/                                 # SHARED — primitive UI components
|   |   |-- Button.tsx
|   |   |-- Card.tsx
|   |   |-- Tile.tsx
|   |   |-- Input.tsx
|   |   |-- Badge.tsx
|   |   |-- Skeleton.tsx
|   |   |-- Dialog.tsx
|   |
|   |-- standard/                           # STANDARD ONLY
|   |   |-- MetricTile.tsx                  # Dashboard metric display
|   |   |-- SidebarHistory.tsx              # Conversation history sidebar
|   |   |-- DataTable.tsx                   # Workflow list table
|   |   |-- UploadForm.tsx                  # Audio upload form
|   |   |-- WorkflowStatusBadge.tsx         # Status indicator
|   |
|   |-- agentic/                            # AGENTIC ONLY
|   |   |-- TranscriptCard.tsx              # Generated transcript display
|   |   |-- DraftPreviewCard.tsx            # BRD draft preview
|   |   |-- ReviewReportCard.tsx            # Review report display
|   |   |-- HITLResponseCard.tsx            # Human-in-the-Loop dialog
|   |   |-- EvidencePackViewer.tsx          # ALCOA+ evidence viewer
|   |   |-- Canvas.tsx                      # Main canvas container
|   |   |-- CanvasStateMachine.tsx          # Canvas state management
|   |
|   |-- shared/                             # CROSS-REALM shared
|   |   |-- WorkflowNav.tsx                 # Navigate to workspace link
|   |   |-- StatusBadge.tsx                 # Shared status display
|   |
|-- hooks/
|   |-- standard/                           # STANDARD ONLY
|   |   |-- useDashboardMetrics.ts          # REST fetch + React Query
|   |   |-- useWorkflowList.ts              # REST fetch + React Query
|   |   |-- useUpload.ts                    # POST to FastAPI
|   |
|   |-- agentic/                            # AGENTIC ONLY
|   |   |-- useAGUIStream.ts                # AG-UI SSE from AgentCore
|   |   |-- useWorkflowStream.ts            # Workflow Stream SSE from FastAPI
|   |   |-- useCanvasState.ts               # Canvas state from dual streams
|   |   |-- useHITLResponse.ts              # HITL response submission
|   |
|-- providers/
|   |-- CopilotKitProvider.tsx              # AGENTIC — route-scoped provider
|   |-- CanvasProvider.tsx                  # AGENTIC — canvas state provider
|   |-- MetricsProvider.tsx                 # STANDARD — metrics context
|   |-- SharedProvider.tsx                  # SHARED — cross-realm context
|
|-- lib/
|   |-- api.ts                              # STANDARD — REST client (FastAPI)
|   |-- agui-client.ts                      # AGENTIC — AG-UI event parser
|   |-- constants.ts                        # SHARED — config, user identity
|   |-- utils.ts                            # SHARED — helpers
|
|-- types/
|   |-- api.ts                              # STANDARD — REST response types
|   |-- agui.ts                             # AGENTIC — AG-UI event types
|   |-- canvas.ts                           # AGENTIC — canvas state types
|   |-- workflow.ts                         # SHARED — workflow domain types
```

### 2.2 Realm Enforcement Rules

These rules are **non-negotiable** and are enforced via code review and (ideally) custom ESLint rules:

| Rule | Enforcement |
|------|-------------|
| CopilotKit hooks (`useAgent`, `useCopilotChat`, `useFrontendTool`, `renderAndWaitForResponse`) **MUST NOT** be imported in `/components/standard/`, `/hooks/standard/`, or standard pages | ESLint `no-restricted-imports` |
| React Query hooks (`useQuery`, `useMutation`) **MUST NOT** be imported in `/components/agentic/`, `/hooks/agentic/`, or workspace pages | ESLint `no-restricted-imports` |
| Components in `/components/ui/` **MUST NOT** import from either realm — they are pure presentation | Code review |
| `CopilotKit` provider **MUST NOT** wrap the root layout — only workspace route | Code review + runtime check |
| AG-UI event types **MUST NOT** leak into standard page props | Type-only imports OK, runtime imports banned |

### 2.3 Code Example: ESLint Configuration for Realm Boundaries

```typescript
// ============================================================================
// .eslintrc.json — Realm Boundary Enforcement
// ============================================================================
// These rules prevent cross-realm imports that would couple the two
// architectural models and create maintenance nightmares.
// ============================================================================

{
  "rules": {
    // Prevent CopilotKit hooks from leaking into standard realm
    "no-restricted-imports": ["error", {
      "paths": [{
        "name": "@copilotkit/react-core",
        "message": "CopilotKit hooks are AGENTIC-ONLY. They may only be used in /components/agentic/, /hooks/agentic/, or /workspace/ pages. See FRONTEND_ARCHITECTURE_GUIDE.md §2.",
        "allowImportNames": []  // Block all named imports
      }, {
        "name": "@copilotkit/react-ui",
        "message": "CopilotKit UI components are AGENTIC-ONLY. See FRONTEND_ARCHITECTURE_GUIDE.md §2."
      }],
      "zones": [
        // Only enforce in standard realm files
        {
          "target": "./components/standard/**",
          "from": ["@copilotkit/react-core", "@copilotkit/react-ui"]
        },
        {
          "target": "./hooks/standard/**",
          "from": ["@copilotkit/react-core", "@copilotkit/react-ui"]
        },
        {
          "target": "./app/page.tsx",
          "from": ["@copilotkit/react-core", "@copilotkit/react-ui"]
        },
        {
          "target": "./app/dashboard/**",
          "from": ["@copilotkit/react-core", "@copilotkit/react-ui"]
        }
      ]
    }]
  }
}
```

---

## 3. Provider Layout Strategy

The provider tree is **deliberately asymmetric**. CopilotKit wraps ONLY the workspace route — not the entire application. This is a critical architectural decision, not an oversight.

### 3.1 Why CopilotKit Is Route-Scoped, Not App-Scoped

Placing `CopilotKit` at the root layout would cause three problems:

1. **Performance:** CopilotKit initializes its WebSocket/SSE connection on mount. On the landing page, this would create an unnecessary connection to AgentCore for every visitor.
2. **Tailwind CSS Conflict:** CopilotKit injects global styles (see Section 11) that conflict with Tailwind utilities. Limiting the scope limits the blast radius.
3. **Semantic Correctness:** CopilotKit is an **agentic UI runtime**. It has no meaning on pages where no agent interaction occurs. Root-level providers should be universal (auth, theme, analytics), not domain-specific (agent chat).

### 3.2 Provider Tree Diagram

```
+------------------------------------------------------------------+
|                     ROOT LAYOUT (app/layout.tsx)                   |
|                     === STANDARD REALM ===                         |
|                                                                    |
|  <html>                                                            |
|    <body>                                                          |
|      <SharedProvider>        ← workflow_id, user identity, theme   |
|        <MetricsProvider>     ← dashboard metrics context           |
|          {children}          ← page content renders here           |
|        </MetricsProvider>                                          |
|      </SharedProvider>                                             |
|    </body>                                                         |
|  </html>                                                           |
|                                                                    |
|  When route = /workspace/[wfId]:                                   |
|                                                                    |
|  children = <WorkspaceLayout>  (app/workspace/[wfId]/layout.tsx)   |
|                                                                    |
|  +----------------------------------------------------------------+|
|  |              WORKSPACE LAYOUT (agentic boundary)               ||
|  |              === AGENTIC REALM ===                             ||
|  |                                                                ||
|  |  <CopilotKitProvider>    ← AG-UI runtime, useAgent context     ||
|  |    <CanvasProvider>      ← dual-stream canvas state machine    ||
|  |      <WorkspacePage>     ← chat + canvas UI                   ||
|  |        <CopilotSidebar>  ← CopilotKit chat sidebar            ||
|  |        <Canvas>          ← generative cards container         ||
|  |      </WorkspacePage>                                         ||
|  |    </CanvasProvider>                                           ||
|  |  </CopilotKitProvider>                                         ||
|  |                                                                ||
|  |  [CopilotKit + Tailwind styles scoped to workspace]           ||
|  +----------------------------------------------------------------+|
+------------------------------------------------------------------+
```

### 3.3 Code: Root Layout (Standard Realm)

```tsx
// ============================================================================
// app/layout.tsx — Root Layout
// ============================================================================
// NO CopilotKit here. This layout is 100% standard React.
// CopilotKit is injected at the workspace route level only.
// ============================================================================

import type { Metadata } from "next";
import { Inter } from "next/font/google";
import "./globals.css";
import { SharedProvider } from "@/providers/SharedProvider";
import { MetricsProvider } from "@/providers/MetricsProvider";

const inter = Inter({ subsets: ["latin"] });

export const metadata: Metadata = {
  title: "AgentCore demo test 1 Automation",
  description: "AI-powered business requirements document generation",
};

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <body className={inter.className}>
        {/* SharedProvider: cross-realm state (user identity, workflow nav) */}
        <SharedProvider>
          {/* MetricsProvider: standard realm analytics context */}
          <MetricsProvider>
            {children}
          </MetricsProvider>
        </SharedProvider>
      </body>
    </html>
  );
}
```

### 3.4 Code: Workspace Layout (Agentic Realm Boundary)

```tsx
// ============================================================================
// app/workspace/[wfId]/layout.tsx — Workspace Layout
// ============================================================================
// This is the CRITICAL boundary. CopilotKit is introduced HERE, not above.
// It wraps ONLY workspace pages. When the user navigates away,
// CopilotKit unmounts completely — connection closed, context destroyed.
//
// The workflow_id from the URL parameter becomes the AG-UI session_id,
// shared with AgentCore for microVM session isolation.
// ============================================================================

"use client";

import { CopilotKit } from "@copilotkit/react-core";
import "@copilotkit/react-ui/styles.css";  // See Section 11 for Tailwind conflict
import { CanvasProvider } from "@/providers/CanvasProvider";
import { useParams } from "next/navigation";

export default function WorkspaceLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  const params = useParams();
  const workflowId = params.wfId as string;

  return (
    <CopilotKit
      runtimeUrl={process.env.NEXT_PUBLIC_AGENTCORE_AGUI_URL}
      agent="content-generation-agent"
      // The workflow_id IS the session_id — this enables microVM session
      // sharing across the multi-agent workflow (Transcriber → Drafter → Reviewer)
      headers={{
        "X-Workflow-Id": workflowId,
        "X-Session-Id": workflowId,  // AgentCore uses this for microVM isolation
      }}
      showDevConsole={process.env.NODE_ENV === "development"}
    >
      {/* CanvasProvider: dual-stream state machine merges AG-UI + Workflow events */}
      <CanvasProvider workflowId={workflowId}>
        <div className="flex h-screen w-full">
          {/* Main content area — canvas + generative cards */}
          <main className="flex-1 overflow-auto">
            {children}
          </main>
        </div>
      </CanvasProvider>
    </CopilotKit>
  );
}
```

### 3.5 Code: Workspace Page

```tsx
// ============================================================================
// app/workspace/[wfId]/page.tsx — Workspace Page
// ============================================================================
// The actual workspace UI. Uses CopilotKit hooks (useAgent, useFrontendTool)
// and Canvas state. All imports here are AGENTIC realm.
// ============================================================================

"use client";

import { CopilotSidebar } from "@copilotkit/react-ui";
import { useAgent } from "@copilotkit/react-core";
import { Canvas } from "@/components/agentic/Canvas";
import { useCanvasState } from "@/hooks/agentic/useCanvasState";
import { useEffect } from "react";

export default function WorkspacePage() {
  // useAgent: primary AG-UI connection hook (v1.50+)
  // Connects directly to AgentCore SSE, streams all AG-UI events
  const agent = useAgent({
    id: "strands-agent",
  });

  // Canvas state merges AG-UI stream + Workflow Stream into unified view
  const { canvasState, isLoading } = useCanvasState();

  // Auto-start agent when workspace loads
  useEffect(() => {
    if (agent.status === "idle") {
      agent.run();
    }
  }, [agent]);

  return (
    <div className="flex h-full">
      {/* Canvas: main content area showing generative UI cards */}
      <div className="flex-1 p-6">
        <Canvas state={canvasState} isLoading={isLoading} />
      </div>

      {/* CopilotSidebar: chat interface provided by CopilotKit */}
      <CopilotSidebar
        labels={{
          title: "BRD Assistant",
          placeholder: "Type your message...",
        }}
      />
    </div>
  );
}
```

### 3.6 Unmount Behavior on Navigation Away

When the user navigates from `/workspace/[wfId]` to `/dashboard`:

1. `WorkspacePage` unmounts
2. `CanvasProvider` unmounts → cleans up both SSE subscriptions
3. `CopilotKit` provider unmounts → **closes AG-UI SSE connection**
4. Agentic realm state is **destroyed** (intentionally — AG-UI events are ephemeral)
5. Dashboard mounts under `MetricsProvider` — standard realm takes over

```
User clicks "Dashboard" link:

/workspace/123 ──► /dashboard

AG-UI SSE:        CLOSED (connection terminated)
Workflow Stream:  CLOSED (connection terminated)
Canvas state:     DESTROYED
Chat messages:    PERSISTED in RDS (restored on next workspace visit)
Agent status:     RESET (will re-run on next workspace load)
```

---

## 4. Routing Strategy

### 4.1 Route Map

```
+----------------+----------------------------+--------------------------+
| Route          | Realm                      | CopilotKit?              |
+----------------+----------------------------+--------------------------+
| /              | Standard — Landing         | NO                       |
| /dashboard     | Standard — Metrics         | NO                       |
| /workflows     | Standard — History List    | NO                       |
| /settings      | Standard — Configuration   | NO                       |
| /workspace/    | Standard (redirect to /)   | NO                       |
| /workspace/:id | Agentic — Workspace        | YES                      |
+----------------+----------------------------+--------------------------+
```

### 4.2 Navigation Patterns

**Standard → Standard** (React Router, no state loss):
```tsx
// components/standard/Sidebar.tsx
import Link from "next/link";

// Simple navigation — no special handling needed
<Link href="/dashboard" className="nav-link">
  Dashboard
</Link>
<Link href="/workflows" className="nav-link">
  Workflows
</Link>
<Link href="/settings" className="nav-link">
  Settings
</Link>
```

**Standard → Agentic** (full context switch):
```tsx
// components/shared/WorkflowNav.tsx
"use client";

import { useRouter } from "next/navigation";
import Button from "@/components/ui/Button";

interface WorkflowNavProps {
  workflowId: string;
}

export function WorkflowNav({ workflowId }: WorkflowNavProps) {
  const router = useRouter();

  const handleEnterWorkspace = () => {
    // router.push triggers navigation to workspace
    // CopilotKit provider will mount, CanvasProvider initializes,
    // chat messages are restored from RDS
    router.push(`/workspace/${workflowId}`);
  };

  return (
    <Button
      variant="primary"
      onClick={handleEnterWorkspace}
    >
      Open Workspace
    </Button>
  );
}
```

**Agentic → Standard** (CopilotKit unmounts):
```tsx
// components/agentic/WorkspaceHeader.tsx
import { useRouter } from "next/navigation";
import Button from "@/components/ui/Button";

export function WorkspaceHeader() {
  const router = useRouter();

  const handleExit = () => {
    // Navigating away from workspace:
    // 1. AG-UI SSE closes
    // 2. Canvas state is destroyed
    // 3. Chat messages remain in RDS (persisted)
    // 4. User returns to standard realm
    router.push("/workflows");
  };

  return (
    <header className="flex items-center justify-between p-4 border-b">
      <h1>BRD Workspace</h1>
      <Button variant="secondary" onClick={handleExit}>
        Back to Workflows
      </Button>
    </header>
  );
}
```

### 4.3 State Persistence on Navigation

| State | Persisted? | Location | Restored How? |
|-------|-----------|----------|---------------|
| Chat messages | YES | RDS `chat_messages` table | REST GET on workspace load |
| Workflow status | YES | Temporal + RDS | REST GET `/api/workflows/{id}` |
| Canvas/card content | PARTIAL | Workflow state (Temporal) | Hydrated from Workflow Stream |
| AG-UI event stream | **NO** | Nowhere (ephemeral) | **NOT restored** (see Section 10) |
| Draft markdown | YES | S3 (claim-check) | REST GET via pre-signed URL |
| Evidence pack | YES | S3 | REST GET via pre-signed URL |

### 4.4 Dynamic Route Segments

The workspace route uses a **dynamic segment** `[wfId]`:

```
/workspace/[wfId]/page.tsx
```

The `wfId` parameter serves three purposes:

1. **URL identity** — unique workspace URL for each workflow
2. **AG-UI session_id** — passed to AgentCore as `X-Session-Id` header for microVM isolation
3. **Data fetch key** — used to load chat history from RDS and workflow state from Temporal

```tsx
// Multiple workflows = multiple independent workspace instances
// Each has its own CopilotKit mount, own SSE connections, own Canvas state

/workspace/abc-123   → CopilotKit mount #1, session=abc-123
/workspace/def-456   → CopilotKit mount #2, session=def-456
/workspace/ghi-789   → CopilotKit mount #3, session=ghi-789
```

---

## 5. Data Fetching — Two Different Patterns

The most fundamental difference between the two realms is **how they fetch data**. Standard realm uses REST. Agentic realm uses SSE streams. These patterns do not mix.

### 5.1 Pattern Comparison

```
+----------------------------+------------------------------------------+
|      STANDARD REALM        |           AGENTIC REALM                  |
|      (REST + Cache)        |          (SSE Streams)                   |
+----------------------------+------------------------------------------+
|                            |                                          |
|  Request → Response        |  Connect → Stream → Event → Event → ...  |
|  (single round-trip)       |  (persistent connection)                 |
|                            |                                          |
|  GET /api/metrics          |  SSE /agui/stream                        |
|  ↓                         |  event: agent_message                    |
|  { metrics: [...] }        |  event: tool_call                        |
|  ↓                         |  event: generative_ui                    |
|  Cache in React Query      |  event: hitl_question                    |
|  ↓                         |  ...                                     |
|  Render                    |  ↓                                       |
|                            |  Each event → state mutation → re-render |
+----------------------------+------------------------------------------+
```

### 5.2 Standard Realm: REST + React Query

Standard pages fetch data via REST and cache with React Query (TanStack Query).

```tsx
// ============================================================================
// hooks/standard/useDashboardMetrics.ts — Standard Realm Data Fetching
// ============================================================================
// Uses React Query for caching, stale-while-revalidate, error handling.
// This is conventional React — familiar to every frontend developer.
// ============================================================================

import { useQuery } from "@tanstack/react-query";
import { apiClient } from "@/lib/api";

interface DashboardMetrics {
  totalWorkflows: number;
  completedWorkflows: number;
  averageDuration: number;
  hitlRoundsAverage: number;
  recentActivity: ActivityEvent[];
}

const DASHBOARD_QUERY_KEY = "dashboard-metrics";

export function useDashboardMetrics() {
  return useQuery<DashboardMetrics>({
    queryKey: [DASHBOARD_QUERY_KEY],
    queryFn: async () => {
      const response = await apiClient.get("/api/metrics/dashboard");
      return response.data;
    },
    // Stale-while-revalidate: show cached data immediately, re-fetch in background
    staleTime: 30_000,     // Data considered fresh for 30 seconds
    refetchInterval: 60_000, // Poll for updates every 60 seconds
    retry: 3,              // Retry failed requests 3 times
  });
}
```

```tsx
// ============================================================================
// hooks/standard/useWorkflowList.ts — Paginated List with React Query
// ============================================================================

import { useQuery } from "@tanstack/react-query";
import { apiClient } from "@/lib/api";

interface WorkflowListItem {
  workflow_id: string;
  status: "running" | "completed" | "failed" | "terminated";
  created_at: string;
  completed_at: string | null;
  current_agent: string;
}

export function useWorkflowList(page: number = 1, limit: number = 20) {
  return useQuery<WorkflowListItem[]>({
    queryKey: ["workflow-list", page, limit],
    queryFn: async () => {
      const response = await apiClient.get("/api/workflows", {
        params: { page, limit },
      });
      return response.data.workflows;
    },
    staleTime: 15_000,
  });
}
```

```tsx
// ============================================================================
// hooks/standard/useUpload.ts — Mutation (POST) Pattern
// ============================================================================

import { useMutation } from "@tanstack/react-query";
import { apiClient } from "@/lib/api";
import { useRouter } from "next/navigation";

interface UploadResponse {
  workflow_id: string;
  status: string;
}

export function useUpload() {
  const router = useRouter();

  return useMutation<UploadResponse, Error, File>({
    mutationFn: async (audioFile: File) => {
      const formData = new FormData();
      formData.append("audio", audioFile);

      const response = await apiClient.post("/api/upload", formData, {
        headers: { "Content-Type": "multipart/form-data" },
      });
      return response.data;
    },
    onSuccess: (data) => {
      // After upload, navigate to workspace where CopilotKit takes over
      router.push(`/workspace/${data.workflow_id}`);
    },
  });
}
```

### 5.3 Agentic Realm: AG-UI SSE Stream

Agentic realm uses a **direct SSE connection** to AgentCore. There is no caching layer — the stream IS the data source.

```tsx
// ============================================================================
// hooks/agentic/useAGUIStream.ts — AG-UI SSE from AgentCore
// ============================================================================
// Connects DIRECTLY to AgentCore's AG-UI endpoint via SSE.
// Parses AG-UI protocol events and exposes them as React state.
// NO caching — events are ephemeral and consumed in real-time.
// ============================================================================

import { useEffect, useRef, useState, useCallback } from "react";
import { AGUIEvent } from "@/types/agui";

interface UseAGUIStreamOptions {
  workflowId: string;
  agentId: string;
  onEvent?: (event: AGUIEvent) => void;
}

interface UseAGUIStreamReturn {
  events: AGUIEvent[];
  status: "idle" | "connecting" | "streaming" | "error" | "closed";
  error: Error | null;
  connect: () => void;
  disconnect: () => void;
}

export function useAGUIStream({
  workflowId,
  agentId,
  onEvent,
}: UseAGUIStreamOptions): UseAGUIStreamReturn {
  const [events, setEvents] = useState<AGUIEvent[]>([]);
  const [status, setStatus] = useState<UseAGUIStreamReturn["status"]>("idle");
  const [error, setError] = useState<Error | null>(null);
  const eventSourceRef = useRef<EventSource | null>(null);

  const disconnect = useCallback(() => {
    if (eventSourceRef.current) {
      eventSourceRef.current.close();
      eventSourceRef.current = null;
      setStatus("closed");
    }
  }, []);

  const connect = useCallback(() => {
    // Close any existing connection
    disconnect();

    setStatus("connecting");
    setError(null);

    // Build SSE URL with session_id = workflow_id
    const url = new URL(
      `${process.env.NEXT_PUBLIC_AGENTCORE_AGUI_URL}/stream`
    );
    url.searchParams.append("agent_id", agentId);
    url.searchParams.append("session_id", workflowId);

    const es = new EventSource(url.toString());
    eventSourceRef.current = es;

    es.onopen = () => {
      setStatus("streaming");
    };

    es.onmessage = (msg) => {
      try {
        const event: AGUIEvent = JSON.parse(msg.data);
        setEvents((prev) => [...prev, event]);
        onEvent?.(event);
      } catch (e) {
        console.error("Failed to parse AG-UI event:", msg.data, e);
      }
    };

    es.onerror = (e) => {
      setError(new Error("AG-UI SSE connection failed"));
      setStatus("error");
      es.close();
    };
  }, [workflowId, agentId, disconnect, onEvent]);

  // Auto-connect on mount, cleanup on unmount
  useEffect(() => {
    connect();
    return () => disconnect();
  }, [connect, disconnect]);

  return { events, status, error, connect, disconnect };
}
```

### 5.4 Agentic Realm: Workflow Stream SSE

The **second SSE stream** comes from FastAPI and carries workflow-level events.

```tsx
// ============================================================================
// hooks/agentic/useWorkflowStream.ts — Workflow Stream SSE from FastAPI
// ============================================================================
// Bridges Temporal Workflow Streams to the frontend via FastAPI SSE.
// Carries orchestration events: HITL questions, status transitions,
// Evidence Pack completion notifications.
// ============================================================================

import { useEffect, useRef, useState, useCallback } from "react";
import { WorkflowEvent } from "@/types/workflow";

interface UseWorkflowStreamOptions {
  workflowId: string;
  onEvent?: (event: WorkflowEvent) => void;
}

interface UseWorkflowStreamReturn {
  events: WorkflowEvent[];
  status: "idle" | "connecting" | "streaming" | "error" | "closed";
  error: Error | null;
}

export function useWorkflowStream({
  workflowId,
  onEvent,
}: UseWorkflowStreamOptions): UseWorkflowStreamReturn {
  const [events, setEvents] = useState<WorkflowEvent[]>([]);
  const [status, setStatus] = useState<UseWorkflowStreamReturn["status"]>("idle");
  const [error, setError] = useState<Error | null>(null);
  const esRef = useRef<EventSource | null>(null);

  useEffect(() => {
    const url = `${process.env.NEXT_PUBLIC_FASTAPI_URL}/api/workflows/${workflowId}/stream`;

    setStatus("connecting");
    const es = new EventSource(url);
    esRef.current = es;

    es.onopen = () => setStatus("streaming");

    es.addEventListener("hitl_question", (msg) => {
      const event: WorkflowEvent = { type: "hitl_question", ...JSON.parse(msg.data) };
      setEvents((prev) => [...prev, event]);
      onEvent?.(event);
    });

    es.addEventListener("status_change", (msg) => {
      const event: WorkflowEvent = { type: "status_change", ...JSON.parse(msg.data) };
      setEvents((prev) => [...prev, event]);
      onEvent?.(event);
    });

    es.addEventListener("evidence_pack", (msg) => {
      const event: WorkflowEvent = { type: "evidence_pack", ...JSON.parse(msg.data) };
      setEvents((prev) => [...prev, event]);
      onEvent?.(event);
    });

    es.onerror = (e) => {
      setError(new Error("Workflow Stream SSE failed"));
      setStatus("error");
    };

    return () => {
      es.close();
      esRef.current = null;
    };
  }, [workflowId, onEvent]);

  return { events, status, error };
}
```

### 5.5 Dual-Stream Merge in CanvasProvider

The `CanvasProvider` merges both streams into a unified canvas state:

```tsx
// ============================================================================
// providers/CanvasProvider.tsx — Dual-Stream State Machine
// ============================================================================
// Merges AG-UI stream (agent thinking + tool calls) + Workflow Stream
// (orchestration events) into a single coherent canvas state.
// This is the heart of the agentic realm's state management.
// ============================================================================

"use client";

import { createContext, useContext, useReducer, ReactNode, useCallback } from "react";
import { useAGUIStream } from "@/hooks/agentic/useAGUIStream";
import { useWorkflowStream } from "@/hooks/agentic/useWorkflowStream";
import { CanvasState, CanvasAction } from "@/types/canvas";

interface CanvasProviderProps {
  workflowId: string;
  children: ReactNode;
}

interface CanvasContextValue {
  state: CanvasState;
  isLoading: boolean;
  dispatch: React.Dispatch<CanvasAction>;
}

const CanvasContext = createContext<CanvasContextValue | null>(null);

function canvasReducer(state: CanvasState, action: CanvasAction): CanvasState {
  switch (action.type) {
    case "AGUI_EVENT":
      // Process AG-UI event: agent messages, tool calls, generative UI
      return processAGUIEvent(state, action.payload);

    case "WORKFLOW_EVENT":
      // Process workflow event: status changes, HITL questions
      return processWorkflowEvent(state, action.payload);

    case "RESTORE_HISTORY":
      // Restore chat history from RDS on workspace load
      return { ...state, messages: action.payload.messages };

    case "USER_RESPONSE":
      // User submitted HITL response
      return { ...state, pendingHITL: null };

    case "RESET":
      return getInitialCanvasState();

    default:
      return state;
  }
}

export function CanvasProvider({ workflowId, children }: CanvasProviderProps) {
  const [state, dispatch] = useReducer(canvasReducer, getInitialCanvasState());

  // Stream 1: AG-UI from AgentCore (direct)
  const aguiStream = useAGUIStream({
    workflowId,
    agentId: "strands-agent",
    onEvent: (event) => dispatch({ type: "AGUI_EVENT", payload: event }),
  });

  // Stream 2: Workflow Stream from FastAPI (Temporal bridge)
  const workflowStream = useWorkflowStream({
    workflowId,
    onEvent: (event) => dispatch({ type: "WORKFLOW_EVENT", payload: event }),
  });

  const isLoading =
    aguiStream.status === "connecting" ||
    workflowStream.status === "connecting";

  return (
    <CanvasContext.Provider value={{ state, isLoading, dispatch }}>
      {children}
    </CanvasContext.Provider>
  );
}

export function useCanvasState(): CanvasContextValue {
  const ctx = useContext(CanvasContext);
  if (!ctx) throw new Error("useCanvasState must be used within CanvasProvider");
  return ctx;
}
```

### 5.6 Why React Query Is BANNED from the Agentic Realm

| Concern | REST + React Query | SSE Streams |
|---------|-------------------|-------------|
| Cache semantics | "fetch then cache" makes sense | "stream then cache" is nonsensical — events are temporal |
| Stale-while-revalidate | Perfect for semi-static data | Impossible — stream events are ordered and time-sensitive |
| Retry logic | Automatic retry with backoff | Manual — must reconnect and resume |
| Query keys | Simple string-based invalidation | Events don't have keys |
| Background refetch | Fetches new data silently | Would require re-subscribing to SSE |

**The streams ARE the data source.** Caching an SSE stream would mean caching a sequence of time-ordered events, which is meaningless on replay. The meaningful state is the **canvas state** derived from those events — and that is managed by `CanvasProvider`, not React Query.

---

## 6. Shared State Between Realms

Both realms share a small amount of state. This is intentionally minimal — most state is realm-specific.

### 6.1 Shared State Registry

| State | Type | Owned By | Access Pattern |
|-------|------|----------|----------------|
| `workflow_id` | URL parameter | Browser | Read-only, derived from `useParams()` |
| User identity (mock) | Config constant | `constants.ts` | Read-only, no auth system |
| Recent conversations list | REST API | Standard reads, Agentic updates | REST GET (standard), restored on workspace load (agentic) |
| Theme preference | LocalStorage | Shared | `useTheme()` hook |
| Navigation intent | React Context | Shared | `router.push()` + Link |

### 6.2 Code: SharedProvider

```tsx
// ============================================================================
// providers/SharedProvider.tsx — Cross-Realm Shared Context
// ============================================================================
// Holds minimal state shared between standard and agentic realms.
// This provider wraps the ENTIRE app (in root layout) because its
// contents are needed by both realms.
//
// RULE: If a piece of state is needed by both realms, it goes here.
// Everything else belongs in realm-specific providers.
// ============================================================================

"use client";

import { createContext, useContext, ReactNode } from "react";
import { useQuery } from "@tanstack/react-query";
import { apiClient } from "@/lib/api";
import { MOCK_USER } from "@/lib/constants";

interface ConversationListItem {
  workflow_id: string;
  title: string;
  last_message_at: string;
  status: string;
}

interface SharedContextValue {
  // User identity (mock — no real auth in demo)
  user: typeof MOCK_USER;

  // Recent conversations (shared — sidebar shows in standard realm,
  // workspace loads the selected one in agentic realm)
  recentConversations: ConversationListItem[];
  isLoadingConversations: boolean;

  // Utility: navigate to a workflow's workspace
  getWorkspaceUrl: (workflowId: string) => string;
}

const SharedContext = createContext<SharedContextValue | null>(null);

export function SharedProvider({ children }: { children: ReactNode }) {
  // Fetch recent conversations — used by sidebar (standard) and
  // workspace load (agentic needs the list to populate thread selector)
  const { data: conversations, isLoading } = useQuery<ConversationListItem[]>({
    queryKey: ["recent-conversations"],
    queryFn: async () => {
      const response = await apiClient.get("/api/conversations/recent");
      return response.data.conversations;
    },
    staleTime: 30_000,
  });

  const value: SharedContextValue = {
    user: MOCK_USER,
    recentConversations: conversations ?? [],
    isLoadingConversations: isLoading,
    getWorkspaceUrl: (workflowId: string) => `/workspace/${workflowId}`,
  };

  return (
    <SharedContext.Provider value={value}>
      {children}
    </SharedContext.Provider>
  );
}

export function useShared(): SharedContextValue {
  const ctx = useContext(SharedContext);
  if (!ctx) throw new Error("useShared must be used within SharedProvider");
  return ctx;
}
```

### 6.3 Code: Constants (Mock Identity)

```tsx
// ============================================================================
// lib/constants.ts — Shared Configuration Constants
// ============================================================================
// Constants needed by both realms. No secrets here — env vars for those.
// ============================================================================

// Mock user identity for demo (no authentication system)
export const MOCK_USER = {
  id: "demo-user-001",
  name: "Demo User",
  email: "demo@example.com",
  role: "business_analyst" as const,
};

// HITL configuration
export const HITL_CONFIG = {
  maxRounds: 5,
  timeoutMinutes: 15,
};

// UI configuration
export const UI_CONFIG = {
  sidebarWidth: "320px",
  maxChatHistory: 100,
  defaultMessagePageSize: 20,
};
```

### 6.4 Why No Global Store (Zustand/Redux)

The shared state is small enough that React Context is sufficient:

- **3 shared values** (user, conversation list, workspace URL builder)
- **No complex cross-cutting concerns** (no auth, no permissions, no feature flags)
- **No frequent updates** (conversation list changes on workflow completion, not every render)
- **No action dispatching** (shared state is mostly read-only)

If the app grows to >10 shared values with complex update logic, consider Zustand or Jotai. For now, Context is the right call.

---

## 7. When NOT to Use CopilotKit — Decision Framework

This is the most important decision in the architecture. CopilotKit is powerful but **not universally applicable**. Using it where it doesn't belong adds latency, cost, and complexity.

### 7.1 The Heuristic

```
+----------------------------+     +----------------------------+
|  IS THE USER...            |     |  IS THE USER...            |
|                            |     |                            |
|  Asking the agent for      |     |  Viewing pre-computed      |
|  data / help / generation? |     |  data / navigating /       |
|                            |     |  configuring?              |
|  Examples:                 |     |                            |
|  "Draft a BRD"             |     |  Examples:                 |
|  "Review this transcript"  |     |  "See completion stats"    |
|  "Fill in missing reqs"    |     |  "Browse past workflows"   |
|  "Approve this draft"      |     |  "Change my settings"      |
|                            |     |                            |
|  → YES: Use CopilotKit     |     |  → YES: Use standard React |
|    (Agentic Realm)         |     |    (Standard Realm)        |
+----------------------------+     +----------------------------+
```

### 7.2 Concrete Examples from the Demo

| Feature | Realm | Rationale |
|---------|-------|-----------|
| **HITL chat (clarifying questions)** | Agentic | User is conversing WITH the agent. Agent generates questions dynamically. |
| **BRD draft preview** | Agentic | Content is agent-generated, user provides feedback inline. |
| **Review report display** | Agentic | Report is agent-generated, user approves/rejects. |
| **Transcript viewer** | Agentic | Transcript is agent-generated from audio, user reviews. |
| **Dashboard metrics** | Standard | Data is pre-computed in RDS. Agent adds nothing. |
| **Workflow history list** | Standard | Tabular data from RDS. Standard React + REST is simpler. |
| **Audio upload form** | Standard | One-shot form POST. No agent interaction needed. |
| **Settings page** | Standard | Configuration forms. No agent involvement. |

### 7.3 The "Dashboard Could Be Agentic" Anti-Pattern

A tempting but wrong design: "What if the dashboard used a Generative UI card driven by an agent? The agent could summarize the metrics in natural language!"

**Why we DON'T do this:**

1. **Latency:** Agent-generated summaries add 1-3 seconds of LLM latency. Pre-computed metrics render instantly.
2. **Token cost:** Every dashboard view would consume LLM tokens for no functional gain.
3. **Reliability:** Agent might hallucinate metric interpretations. Pre-computed numbers are deterministic.
4. **Complexity:** Generative UI cards require tool definitions, rendering logic, error handling. Standard `<MetricTile>` is 20 lines.
5. **User value:** A business analyst viewing completion percentages doesn't need AI narration. They need the numbers, fast.

### 7.4 Decision Matrix

```
+----------------------------+-------------------+-------------------+
| Criterion                  | Use CopilotKit    | Use Standard React|
+----------------------------+-------------------+-------------------+
| User converses with agent  | YES               | No                |
| Content is agent-generated | YES               | No                |
| HITL (human-in-the-loop)   | YES               | Never             |
| Real-time stream updates   | YES               | No                |
| Data is pre-computed       | No                | YES               |
| Form submission            | No                | YES               |
| Tabular data display       | No                | YES               |
| Navigation / browsing      | No                | YES               |
| Latency-sensitive          | Avoid             | YES               |
| Token cost matters         | Avoid             | YES               |
+----------------------------+-------------------+-------------------+
```

### 7.5 The One Exception: "Agent-Assisted Dashboard"

Future evolution: A dashboard widget that lets the user **ask** the agent about their metrics ("Why did my HITL round count spike last week?"). This hybrid feature would:

- Use **standard React** for the metric tiles (static display)
- Embed a **small CopilotKit chat panel** (agentic) for Q&A about the metrics
- Both coexist on the same page, but the CopilotKit provider is scoped to the chat panel, not the whole page

This is the correct pattern for agent-assisted standard pages: **scoped CopilotKit**, not page-level conversion.

---

## 8. Component Library Strategy

Components are organized into three directories by ownership. This separation prevents cross-realm coupling and makes it obvious where a new component belongs.

### 8.1 Directory Structure

```
components/
|
|-- ui/                 # SHARED — pure presentation primitives
|   |-- Button.tsx      #    No CopilotKit, no REST calls, no business logic
|   |-- Card.tsx        #    Props-only, Tailwind styled
|   |-- Tile.tsx        #    Used by both MetricTile (standard) and TranscriptCard (agentic)
|   |-- Input.tsx
|   |-- Badge.tsx
|   |-- Skeleton.tsx
|   |-- Dialog.tsx
|   |-- Separator.tsx
|   |-- Tooltip.tsx
|
|-- standard/           # STANDARD ONLY — business logic + REST
|   |-- MetricTile.tsx  #    Reads from useDashboardMetrics (React Query)
|   |-- SidebarHistory.tsx  # Reads from useShared (conversation list)
|   |-- DataTable.tsx   #    Paginated table, REST-powered
|   |-- UploadForm.tsx  #    POST to /api/upload
|   |-- WorkflowStatusBadge.tsx
|   |-- DashboardGrid.tsx
|   |-- WorkflowFilters.tsx
|
|-- agentic/            # AGENTIC ONLY — CopilotKit + SSE logic
|   |-- TranscriptCard.tsx    # Renders from AG-UI tool_call events
|   |-- DraftPreviewCard.tsx  # Renders markdown from agent output
|   |-- ReviewReportCard.tsx  # Displays agent-generated review
|   |-- HITLResponseCard.tsx  # Uses useFrontendTool + renderAndWaitForResponse
|   |-- EvidencePackViewer.tsx
|   |-- Canvas.tsx            # Container: merges dual-stream state
|   |-- CanvasStateMachine.tsx # State reducer for canvas
|   |-- AgentMessageBubble.tsx # Custom chat message renderer
|   |-- ToolCallIndicator.tsx  # Shows agent tool execution status
|
|-- shared/             # CROSS-REALM — used by both
    |-- WorkflowNav.tsx # Link component that enters workspace
    |-- StatusBadge.tsx # Status display (shared visual language)
    |-- LoadingState.tsx # Loading skeleton used by both realms
    |-- ErrorBoundary.tsx # Error handling for both realms
```

### 8.2 Shared UI Primitives Example

```tsx
// ============================================================================
// components/ui/Card.tsx — Shared Primitive
// ============================================================================
// NO CopilotKit imports. NO REST hooks. Pure presentation.
// Used by both MetricTile (standard) and DraftPreviewCard (agentic).
// ============================================================================

import { ReactNode } from "react";

interface CardProps {
  children: ReactNode;
  className?: string;
  variant?: "default" | "outlined" | "elevated";
  onClick?: () => void;
}

export function Card({ children, className = "", variant = "default", onClick }: CardProps) {
  const variantStyles = {
    default: "bg-white border border-gray-200 rounded-lg shadow-sm",
    outlined: "bg-transparent border-2 border-gray-300 rounded-lg",
    elevated: "bg-white rounded-lg shadow-md hover:shadow-lg transition-shadow",
  };

  return (
    <div
      className={`${variantStyles[variant]} p-4 ${className}`}
      onClick={onClick}
      role={onClick ? "button" : undefined}
      tabIndex={onClick ? 0 : undefined}
    >
      {children}
    </div>
  );
}

export function CardHeader({ children, className = "" }: { children: ReactNode; className?: string }) {
  return <div className={`flex items-center justify-between mb-3 ${className}`}>{children}</div>;
}

export function CardTitle({ children, className = "" }: { children: ReactNode; className?: string }) {
  return <h3 className={`text-lg font-semibold text-gray-900 ${className}`}>{children}</h3>;
}

export function CardContent({ children, className = "" }: { children: ReactNode; className?: string }) {
  return <div className={`text-sm text-gray-700 ${className}`}>{children}</div>;
}
```

### 8.3 Standard Realm Component Example

```tsx
// ============================================================================
// components/standard/MetricTile.tsx — Standard-Only Component
// ============================================================================
// Uses React Query for data fetching. NO CopilotKit hooks.
// Uses shared Card primitive from components/ui/.
// ============================================================================

import { Card, CardHeader, CardTitle, CardContent } from "@/components/ui/Card";
import { useDashboardMetrics } from "@/hooks/standard/useDashboardMetrics";
import { Skeleton } from "@/components/ui/Skeleton";

interface MetricTileProps {
  label: string;
  metricKey: "totalWorkflows" | "completedWorkflows" | "averageDuration" | "hitlRoundsAverage";
}

export function MetricTile({ label, metricKey }: MetricTileProps) {
  const { data, isLoading } = useDashboardMetrics();

  if (isLoading) {
    return (
      <Card>
        <Skeleton className="h-4 w-24 mb-2" />
        <Skeleton className="h-8 w-16" />
      </Card>
    );
  }

  const value = data?.[metricKey] ?? 0;
  const formatted = metricKey === "averageDuration"
    ? `${value.toFixed(1)} min`
    : value.toLocaleString();

  return (
    <Card variant="elevated">
      <CardHeader>
        <CardTitle className="text-sm font-medium text-gray-500">{label}</CardTitle>
      </CardHeader>
      <CardContent>
        <p className="text-3xl font-bold text-gray-900">{formatted}</p>
      </CardContent>
    </Card>
  );
}
```

### 8.4 Agentic Realm Component Example

```tsx
// ============================================================================
// components/agentic/DraftPreviewCard.tsx — Agentic-Only Component
// ============================================================================
// Uses CopilotKit hooks (renderAndWaitForResponse for HITL approval).
// Uses shared Card primitive.
// Renders agent-generated markdown via react-markdown.
// ============================================================================

"use client";

import { useState } from "react";
import { renderAndWaitForResponse } from "@copilotkit/react-core";
import ReactMarkdown from "react-markdown";
import { Card, CardHeader, CardTitle, CardContent } from "@/components/ui/Card";
import Button from "@/components/ui/Button";
import Badge from "@/components/ui/Badge";

interface DraftPreviewCardProps {
  draftMarkdown: string;
  workflowId: string;
  onApproved: () => void;
  onRejected: () => void;
}

export function DraftPreviewCard({
  draftMarkdown,
  workflowId,
  onApproved,
  onRejected,
}: DraftPreviewCardProps) {
  const [isExpanded, setIsExpanded] = useState(false);

  // renderAndWaitForResponse: blocks agent execution until user clicks
  // Approve or Reject. This is the HITL pattern.
  const { render, status } = renderAndWaitForResponse({
    action: (
      <div className="flex gap-2 mt-4">
        <Button variant="primary" onClick={() => {}}>
          Approve Draft
        </Button>
        <Button variant="secondary" onClick={() => {}}>
          Request Changes
        </Button>
      </div>
    ),
    waitForResponse: true,
  });

  return (
    <Card variant="elevated" className="max-w-2xl">
      <CardHeader>
        <div className="flex items-center gap-2">
          <CardTitle>BRD Draft</CardTitle>
          <Badge variant="info">Agent Generated</Badge>
        </div>
      </CardHeader>
      <CardContent>
        <div
          className={`prose prose-sm max-w-none ${isExpanded ? "" : "max-h-64 overflow-hidden"}`}
        >
          <ReactMarkdown>{draftMarkdown}</ReactMarkdown>
        </div>

        {!isExpanded && (
          <button
            onClick={() => setIsExpanded(true)}
            className="text-blue-600 hover:text-blue-800 text-sm mt-2"
          >
            Show more...
          </button>
        )}

        {/* HITL action buttons — agent waits for this response */}
        {status === "waiting" && render}
      </CardContent>
    </Card>
  );
}
```

### 8.5 Tailwind CSS Sharing

Both realms use the **same Tailwind configuration**. There is no realm-specific theme:

```javascript
// tailwind.config.ts — shared across both realms
import type { Config } from "tailwindcss";

const config: Config = {
  content: [
    "./app/**/*.{js,ts,jsx,tsx,mdx}",
    "./components/**/*.{js,ts,jsx,tsx,mdx}",
  ],
  theme: {
    extend: {
      colors: {
        // Custom brand colors — shared
        brand: {
          50: "#eff6ff",
          500: "#3b82f6",
          600: "#2563eb",
          900: "#1e3a5f",
        },
      },
    },
  },
  plugins: [],
};

export default config;
```

**Exception:** CopilotKit injects its own CSS (see Section 11) which may conflict with Tailwind utilities. This is managed via CSS scoping, not separate configs.

---

## 9. Testing Strategy for Each Realm

Testing strategies differ fundamentally because the two realms interact with fundamentally different data sources.

### 9.1 Testing Philosophy

```
+-------------------------+-------------------------+
|    STANDARD REALM       |     AGENTIC REALM       |
|    (REST mocks)         |     (Stream mocks)      |
+-------------------------+-------------------------+
|                         |                         |
| Mock REST responses     | Mock SSE event streams  |
| Assert rendered output  | Assert hook outputs     |
| Test user interactions  | Test stream handling    |
| (clicks, form submits)  | (events → state)        |
|                         |                         |
| Tools: Jest + RTL       | Tools: Jest + RTL +     |
|        msw (Mock SW)    |        custom SSE mock  |
+-------------------------+-------------------------+
```

### 9.2 Standard Realm Testing

```tsx
// ============================================================================
// __tests__/components/MetricTile.test.tsx — Standard Component Test
// ============================================================================
// Uses MSW to mock REST responses. Tests rendering based on fetched data.
// ============================================================================

import { render, screen } from "@testing-library/react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { rest } from "msw";
import { setupServer } from "msw/node";
import { MetricTile } from "@/components/standard/MetricTile";

// Mock REST server
const server = setupServer(
  rest.get("/api/metrics/dashboard", (req, res, ctx) => {
    return res(
      ctx.json({
        totalWorkflows: 42,
        completedWorkflows: 38,
        averageDuration: 12.5,
        hitlRoundsAverage: 2.3,
        recentActivity: [],
      })
    );
  })
);

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

function createWrapper() {
  const queryClient = new QueryClient({
    defaultOptions: { queries: { retry: false } },
  });
  return ({ children }: { children: React.ReactNode }) => (
    <QueryClientProvider client={queryClient}>{children}</QueryClientProvider>
  );
}

describe("MetricTile", () => {
  it("renders loading state initially", () => {
    render(<MetricTile label="Total Workflows" metricKey="totalWorkflows" />, {
      wrapper: createWrapper(),
    });
    expect(screen.getByRole("status")).toBeInTheDocument(); // Skeleton
  });

  it("renders metric value after fetch", async () => {
    render(<MetricTile label="Total Workflows" metricKey="totalWorkflows" />, {
      wrapper: createWrapper(),
    });
    expect(await screen.findByText("42")).toBeInTheDocument();
    expect(screen.getByText("Total Workflows")).toBeInTheDocument();
  });

  it("formats duration with units", async () => {
    render(<MetricTile label="Avg Duration" metricKey="averageDuration" />, {
      wrapper: createWrapper(),
    });
    expect(await screen.findByText("12.5 min")).toBeInTheDocument();
  });
});
```

### 9.3 Agentic Realm Testing

Testing agentic components requires **mocking SSE event streams** and asserting on hook outputs.

```tsx
// ============================================================================
// __tests__/hooks/useAGUIStream.test.ts — AG-UI Stream Hook Test
// ============================================================================
// Creates a mock EventSource that emits AG-UI events programmatically.
// Asserts that the hook correctly parses and exposes events.
// ============================================================================

import { renderHook, waitFor } from "@testing-library/react";
import { useAGUIStream } from "@/hooks/agentic/useAGUIStream";
import { AGUIEvent } from "@/types/agui";

// Mock EventSource globally
class MockEventSource {
  onmessage: ((msg: MessageEvent) => void) | null = null;
  onopen: (() => void) | null = null;
  onerror: ((e: Event) => void) | null = null;
  readyState = 0;

  constructor(public url: string) {
    // Auto-trigger open on next tick
    setTimeout(() => {
      this.readyState = 1;
      this.onopen?.();
    }, 0);
  }

  close() {
    this.readyState = 2;
  }

  // Test helper: simulate incoming event
  simulateMessage(data: AGUIEvent) {
    this.onmessage?.(new MessageEvent("message", {
      data: JSON.stringify(data),
    }));
  }

  // Test helper: simulate error
  simulateError() {
    this.onerror?.(new Event("error"));
  }
}

// Replace global EventSource
(global as any).EventSource = MockEventSource;

describe("useAGUIStream", () => {
  it("connects and transitions to streaming state", async () => {
    const { result } = renderHook(() =>
      useAGUIStream({
        workflowId: "test-wf-123",
        agentId: "test-agent",
      })
    );

    // Should start connecting
    expect(result.current.status).toBe("connecting");

    // After EventSource opens, should be streaming
    await waitFor(() => {
      expect(result.current.status).toBe("streaming");
    });
  });

  it("collects AG-UI events", async () => {
    const { result } = renderHook(() =>
      useAGUIStream({
        workflowId: "test-wf-123",
        agentId: "test-agent",
      })
    );

    await waitFor(() => expect(result.current.status).toBe("streaming"));

    // Simulate agent message event
    const mockEvent: AGUIEvent = {
      type: "agent_message",
      content: "I've analyzed the transcript and identified 3 gaps.",
      timestamp: Date.now(),
    };

    // Access the mock EventSource instance and emit
    const esInstance = (global.EventSource as any).instances?.[0];
    esInstance?.simulateMessage(mockEvent);

    await waitFor(() => {
      expect(result.current.events).toHaveLength(1);
      expect(result.current.events[0].content).toBe(
        "I've analyzed the transcript and identified 3 gaps."
      );
    });
  });

  it("disconnects on unmount", () => {
    const { unmount } = renderHook(() =>
      useAGUIStream({
        workflowId: "test-wf-123",
        agentId: "test-agent",
      })
    );

    unmount();

    // EventSource.close() should have been called
    // Verify via mock tracking
  });
});
```

### 9.4 CopilotKit Hook Testing

CopilotKit provides recommended patterns for testing hooks that depend on its context:

```tsx
// ============================================================================
// __tests__/components/DraftPreviewCard.test.tsx — Generative UI Card Test
// ============================================================================
// Wraps component in CopilotKit test provider. Mocks renderAndWaitForResponse.
// ============================================================================

import { render, screen, fireEvent } from "@testing-library/react";
import { CopilotKit } from "@copilotkit/react-core";
import { DraftPreviewCard } from "@/components/agentic/DraftPreviewCard";

// Mock renderAndWaitForResponse
jest.mock("@copilotkit/react-core", () => ({
  ...jest.requireActual("@copilotkit/react-core"),
  renderAndWaitForResponse: jest.fn(() => ({
    render: <div data-testid="hitl-actions">Approve / Reject</div>,
    status: "waiting",
    respond: jest.fn(),
  })),
}));

function renderWithCopilotKit(ui: React.ReactElement) {
  return render(
    <CopilotKit
      runtimeUrl="http://test-runtime"
      agent="test-agent"
    >
      {ui}
    </CopilotKit>
  );
}

describe("DraftPreviewCard", () => {
  it("renders markdown content", () => {
    renderWithCopilotKit(
      <DraftPreviewCard
        draftMarkdown="# BRD Draft\n\n## Section 1\nContent here."
        workflowId="test-wf-123"
        onApproved={jest.fn()}
        onRejected={jest.fn()}
      />
    );

    expect(screen.getByText("BRD Draft")).toBeInTheDocument();
    expect(screen.getByText("Section 1")).toBeInTheDocument();
    expect(screen.getByText("Content here.")).toBeInTheDocument();
  });

  it("shows agent-generated badge", () => {
    renderWithCopilotKit(
      <DraftPreviewCard
        draftMarkdown="# Test"
        workflowId="test-wf-123"
        onApproved={jest.fn()}
        onRejected={jest.fn()}
      />
    );

    expect(screen.getByText("Agent Generated")).toBeInTheDocument();
  });

  it("expands collapsed content on click", () => {
    renderWithCopilotKit(
      <DraftPreviewCard
        draftMarkdown={"# " + "x".repeat(1000)} // Long content
        workflowId="test-wf-123"
        onApproved={jest.fn()}
        onRejected={jest.fn()}
      />
    );

    const expandButton = screen.getByText("Show more...");
    fireEvent.click(expandButton);
    expect(screen.queryByText("Show more...")).not.toBeInTheDocument();
  });
});
```

### 9.5 Integration Testing: Dual Streams

```tsx
// ============================================================================
// __tests__/providers/CanvasProvider.test.tsx — Dual-Stream Integration
// ============================================================================
// Tests that AG-UI events and Workflow Stream events are correctly merged
// into the unified canvas state.
// ============================================================================

import { renderHook, waitFor } from "@testing-library/react";
import { CanvasProvider, useCanvasState } from "@/providers/CanvasProvider";

describe("CanvasProvider dual-stream merge", () => {
  it("merges agent message + workflow status change", async () => {
    const wrapper = ({ children }: { children: React.ReactNode }) => (
      <CanvasProvider workflowId="test-wf-123">{children}</CanvasProvider>
    );

    const { result } = renderHook(() => useCanvasState(), { wrapper });

    // Wait for streams to connect
    await waitFor(() => {
      expect(result.current.isLoading).toBe(false);
    });

    // Simulate AG-UI event: agent sends a message
    // (In real test, mock EventSource to emit this)

    // Simulate Workflow Stream event: status changes to "hitl_required"
    // (In real test, mock second EventSource to emit this)

    // Assert: canvas state reflects BOTH events
    // - Message appears in chat history
    // - Status indicator shows HITL pending
  });
});
```

### 9.6 Testing Summary by Realm

| Test Type | Standard Realm | Agentic Realm |
|-----------|---------------|---------------|
| **Unit tests** | Component rendering with mocked REST | Hook behavior with mocked SSE |
| **Integration tests** | Form submission → REST → re-render | Dual-stream → CanvasProvider → UI update |
| **E2E tests** | Upload → navigate → verify dashboard | Full workflow: upload → chat → HITL → approve |
| **Mocking** | MSW (Mock Service Worker) for REST | Custom EventSource mock for SSE |
| **Key assertions** | `expect(screen.getByText(...))` | `expect(hookResult.current.events)` |

---

## 10. Refresh Behavior in Each Realm

How each realm handles page refresh (F5) is fundamentally different.

### 10.1 Standard Realm Refresh

Standard pages refresh conventionally:

```
User presses F5 on /dashboard:

1. Browser reloads /dashboard
2. Next.js re-renders page
3. React Query fetches fresh data from REST API
4. Dashboard renders with latest data
5. No special handling needed — standard Next.js + React Query behavior
```

```tsx
// Standard pages: React Query handles everything
// No special refresh logic required

// hooks/standard/useDashboardMetrics.ts
export function useDashboardMetrics() {
  return useQuery<...>({
    queryKey: ["dashboard-metrics"],
    queryFn: fetchMetrics,
    // On refresh: React Query automatically re-fetches
    // If cache is stale, shows cached data immediately, fetches in background
    // If no cache (hard refresh), shows loading skeleton until fetch completes
    staleTime: 30_000,
  });
}
```

### 10.2 Agentic Realm (Workspace) Refresh

Workspace refresh is **more complex** because AG-UI events are ephemeral:

```
User presses F5 on /workspace/abc-123:

1. Browser reloads /workspace/abc-123
2. Next.js re-renders workspace page
3. CopilotKit provider mounts → new AG-UI SSE connection
4. CanvasProvider mounts → new Workflow Stream SSE connection
5. CRITICAL: AG-UI events from the PREVIOUS session are GONE (ephemeral)
   - Agent thinking steps: LOST
   - Tool call visualization: LOST
   - Streaming tokens: LOST
6. Chat messages are RESTORED from RDS:
   - GET /api/workflows/abc-123/messages
   - Messages rendered in chat sidebar
7. Canvas state is HYDRATED from Workflow Stream:
   - Current workflow status (running, completed, etc.)
   - Current agent (Transcriber, Drafter, Reviewer)
   - Pending HITL question (if any)
   - Draft content URL (if generated)
8. AG-UI stream starts FRESH from current agent state
   - New streaming events going forward
   - Historical streaming events NOT replayed
```

### 10.3 Code: Workspace Refresh Handling

```tsx
// ============================================================================
// app/workspace/[wfId]/page.tsx — Refresh State Restoration
// ============================================================================
// On refresh (mount), this component:
// 1. Loads chat history from RDS (REST call)
// 2. Restores canvas state from workflow state (Workflow Stream)
// 3. Starts fresh AG-UI stream (ephemeral, no replay)
// ============================================================================

"use client";

import { useEffect } from "react";
import { useParams } from "next/navigation";
import { useQuery } from "@tanstack/react-query";
import { useAgent } from "@copilotkit/react-core";
import { CopilotSidebar } from "@copilotkit/react-ui";
import { Canvas } from "@/components/agentic/Canvas";
import { useCanvasState } from "@/hooks/agentic/useCanvasState";
import { apiClient } from "@/lib/api";

interface ChatMessage {
  id: string;
  role: "user" | "assistant" | "system";
  content: string;
  timestamp: string;
  workflow_id: string;
}

export default function WorkspacePage() {
  const params = useParams();
  const workflowId = params.wfId as string;

  // AG-UI connection (fresh on every mount — no history)
  const agent = useAgent({ id: "strands-agent" });

  // Canvas state from dual streams
  const { state: canvasState, dispatch } = useCanvasState();

  // RESTORE: Load chat history from RDS on mount
  // This is NOT an SSE stream — it's a one-time REST fetch
  const { data: chatHistory, isLoading: historyLoading } = useQuery<ChatMessage[]>({
    queryKey: ["chat-history", workflowId],
    queryFn: async () => {
      const response = await apiClient.get(`/api/workflows/${workflowId}/messages`);
      return response.data.messages;
    },
    staleTime: Infinity, // Don't re-fetch — new messages come via SSE
  });

  // Restore chat history into canvas state
  useEffect(() => {
    if (chatHistory && chatHistory.length > 0) {
      dispatch({
        type: "RESTORE_HISTORY",
        payload: { messages: chatHistory },
      });
    }
  }, [chatHistory, dispatch]);

  // Auto-start agent if workflow is still running
  useEffect(() => {
    if (agent.status === "idle" && canvasState.workflowStatus === "running") {
      agent.run();
    }
  }, [agent, canvasState.workflowStatus]);

  return (
    <div className="flex h-full">
      <div className="flex-1 p-6">
        <Canvas
          state={canvasState}
          isLoading={historyLoading || canvasState.isLoading}
        />
      </div>
      <CopilotSidebar
        defaultOpen={true}
        labels={{ title: "BRD Assistant", placeholder: "Type your message..." }}
      />
    </div>
  );
}
```

### 10.4 What Is Lost vs. Preserved on Refresh

| Data | Persisted? | Storage | Restored? |
|------|-----------|---------|-----------|
| Chat messages (text) | **YES** | RDS `chat_messages` | YES — REST GET |
| Workflow status | **YES** | Temporal + RDS | YES — Workflow Stream |
| BRD draft content | **YES** | S3 (claim-check) | YES — pre-signed URL |
| Review report | **YES** | S3 (claim-check) | YES — pre-signed URL |
| Evidence pack | **YES** | S3 | YES — pre-signed URL |
| Agent thinking steps | **NO** | None (ephemeral) | **NO** |
| Tool call visualizations | **NO** | None (ephemeral) | **NO** |
| Streaming token output | **NO** | None (ephemeral) | **NO** |
| HITL response form state | **NO** | Component state | **NO** (user re-enters) |
| Canvas scroll position | **NO** | Component state | **NO** |
| Chat sidebar open/closed | **NO** | Component state | **NO** (defaults to open) |

### 10.5 Design Decision: AG-UI Events Are Ephemeral

The ephemeral nature of AG-UI events is **by design**, not a bug:

- **Agent thinking steps** are intermediate outputs — they have no value after the final response is generated
- **Streaming tokens** are visual effects — persisting them would waste storage
- **Tool call visualizations** are ephemeral feedback — the tool results are what matter

What matters (chat messages, draft content, workflow status) IS persisted. The streaming "experience" is not.

---

## 11. Key Gotchas & Conflict Resolution

### 11.1 CopilotKit + Tailwind CSS Conflict (Issue #1857)

**Problem:** CopilotKit injects global CSS that conflicts with Tailwind utility classes. Symptoms include:

- Button styles overridden by CopilotKit defaults
- Layout shifts when CopilotKit components mount
- Font size inconsistencies in chat bubbles

**Root Cause:** `@copilotkit/react-ui/styles.css` sets global styles that conflict with Tailwind's utility-first approach.

**Solutions (in order of preference):**

```css
/* ============================================================================
 * globals.css — Tailwind + CopilotKit Conflict Resolution
 * ============================================================================
 * Strategy: Import CopilotKit styles FIRST, then override with Tailwind
 * utilities using @layer and !important where necessary.
 * ============================================================================ */

/* 1. Tailwind directives (always first) */
@tailwind base;
@tailwind components;
@tailwind utilities;

/* 2. CopilotKit styles — imported with layer isolation */
/* Option A: Import and scope to workspace container */
@layer components {
  .copilotkit-workspace {
    @import "@copilotkit/react-ui/styles.css";
  }
}

/* Option B: Override CopilotKit globals with Tailwind utilities */
/* (Use this if Option A causes specificity issues) */
@layer utilities {
  /* Reset CopilotKit button styles to use Tailwind */
  .copilot-kit-button {
    @apply px-4 py-2 rounded-md font-medium !important;
  }

  /* Ensure chat container uses Tailwind spacing */
  .copilot-kit-chat-container {
    @apply flex flex-col gap-2 !important;
  }
}
```

```tsx
// Alternative: CSS Module approach for workspace
// app/workspace/[wfId]/layout.tsx

import "./workspace-copilotkit-override.css";  // Override styles scoped to workspace

// workspace-copilotkit-override.css:
// .copilotkit-workspace .copilot-kit-button { @apply btn-primary; }
```

**Recommendation:** Use CSS containment to scope CopilotKit styles to the workspace container only. Do NOT let CopilotKit styles leak to standard realm pages.

### 11.2 Two SSE Streams Need Separate Cleanup

**Problem:** If both EventSource connections aren't properly cleaned up, they leak on unmount.

**Solution:** Each hook manages its own EventSource with `useRef` + `useEffect` cleanup:

```tsx
// Each stream hook follows this pattern:
useEffect(() => {
  const es = new EventSource(url);
  esRef.current = es;

  // ... event handlers ...

  return () => {
    es.close();        // CLOSE the connection
    esRef.current = null;  // CLEAR the ref
  };
}, [dependency]);  // Re-create ONLY when dependency changes
```

**Common mistake:** Creating a new EventSource on every render without cleanup:

```tsx
// WRONG: Creates infinite EventSources
useEffect(() => {
  const es = new EventSource(url);
  es.onmessage = ...;
  // No cleanup! Connection leaks on every re-render.
});

// CORRECT: One EventSource per mount, cleaned up on unmount
useEffect(() => {
  const es = new EventSource(url);
  es.onmessage = ...;
  return () => es.close();
}, [url]);  // Dependency array ensures single creation
```

### 11.3 Workspace Page Must Restore State from RDS on Load

**Problem:** If chat history is not loaded from RDS on workspace mount, the chat sidebar appears empty after refresh.

**Solution:** The workspace page ALWAYS fetches chat history via REST on mount (see Section 10). This is a **blocking fetch** — the workspace shows a loading state until messages are restored.

```tsx
// Show skeleton until chat history is restored
if (historyLoading) {
  return <WorkspaceSkeleton />;
}

// Only render full workspace after history is loaded
return <FullWorkspace ... />;
```

### 11.4 AG-UI Events Cannot Be Replayed

**Problem:** Developers expect to "replay" the AG-UI stream on refresh. This is impossible.

**Solution:** Accept the ephemeral nature of AG-UI events. Design the UX so that:

- Chat messages (text) are sufficient for context — thinking steps are nice-to-have
- Workflow status from Temporal provides enough state to reconstruct the canvas
- If thinking steps are critical, log them to CloudWatch (not the frontend)

### 11.5 Route-Scoped vs. App-Scoped Provider

**Problem:** Placing `CopilotKit` at the root layout causes it to initialize on every page, including the landing page.

**Solution:** Only mount `CopilotKit` inside `/workspace/[wfId]/layout.tsx`. Use a **route group** if needed:

```
app/
|-- layout.tsx              # NO CopilotKit (standard pages)
|-- page.tsx                # Landing page
|-- (workspace)/            # Route group (no URL prefix)
|   |-- workspace/
|   |   |-- [wfId]/
|   |   |   |-- layout.tsx  # CopilotKit HERE
```

### 11.6 Mock User Identity Is Not Auth

**Problem:** `MOCK_USER` in `constants.ts` is NOT a substitute for authentication.

**Reality:** This demo uses a hardcoded mock identity. In production:

- Replace `MOCK_USER` with a real auth system (NextAuth, Clerk, etc.)
- Pass auth tokens to FastAPI via `Authorization` header
- Pass session tokens to AgentCore via `X-Session-Id` header
- The architecture supports this — SharedProvider is the injection point

---

## 12. Tech Reference

### 12.1 Verified URLs

| Technology | URL | Used For |
|------------|-----|----------|
| Next.js App Router | https://nextjs.org/docs/app | Routing, layouts, SSR |
| CopilotKit | https://docs.copilotkit.ai | Agentic UI hooks, chat components |
| AG-UI Protocol | https://docs.ag-ui.com/ | SSE event format, agent communication |
| React Query | https://tanstack.com/query/latest | REST data caching (standard realm) |
| react-markdown | https://github.com/remarkjs/react-markdown | Markdown rendering in agent cards |
| Tailwind CSS | https://tailwindcss.com | Utility-first styling |

### 12.2 CopilotKit Hook Reference (Agentic Realm)

| Hook | Source | Purpose | When to Use |
|------|--------|---------|-------------|
| `useAgent` | `@copilotkit/react-core` | AG-UI connection, event streaming | Always in workspace — primary agent hook |
| `useCopilotChat` | `@copilotkit/react-core` | Chat message management | For custom chat UIs (instead of CopilotSidebar) |
| `useFrontendTool` | `@copilotkit/react-core` | Register frontend-callable tools | HITL responses, button actions |
| `renderAndWaitForResponse` | `@copilotkit/react-core` | Block agent until user acts | Approval/rejection flows |
| `CopilotSidebar` | `@copilotkit/react-ui` | Pre-built chat sidebar | Default chat UI |
| `CopilotChat` | `@copilotkit/react-ui` | Standalone chat component | Custom layout chat |

### 12.3 Package Versions

```json
{
  "dependencies": {
    "next": "^14.0.0",
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "@copilotkit/react-core": "^1.50.0",
    "@copilotkit/react-ui": "^1.50.0",
    "@tanstack/react-query": "^5.0.0",
    "react-markdown": "^9.0.0",
    "tailwindcss": "^3.4.0"
  }
}
```

### 12.4 Environment Variables

```bash
# .env.local — Frontend Environment Configuration

# AG-UI direct connection to AgentCore (agentic realm)
NEXT_PUBLIC_AGENTCORE_AGUI_URL=http://localhost:8080/invocations

# FastAPI REST + SSE bridge (both realms)
NEXT_PUBLIC_FASTAPI_URL=http://localhost:8000

# Mock user for demo (standard + agentic shared)
NEXT_PUBLIC_MOCK_USER_ID=demo-user-001

# Dev mode features
NEXT_PUBLIC_ENABLE_DEV_CONSOLE=true
```

---

## Appendix A: Quick Decision Checklist

When adding a new feature, use this checklist:

```
[ ] Does the feature require user conversation with an agent?
    → YES: Agentic realm (/workspace, CopilotKit)
    → NO:  Continue...

[ ] Does the feature display agent-generated content?
    → YES: Agentic realm (generative UI cards)
    → NO:  Continue...

[ ] Does the feature require HITL (human-in-the-loop)?
    → YES: Agentic realm (renderAndWaitForResponse)
    → NO:  Continue...

[ ] Does the feature consume real-time SSE streams?
    → YES: Agentic realm (dual-stream pattern)
    → NO:  Continue...

[ ] Is the data pre-computed and stored in RDS/S3?
    → YES: Standard realm (REST + React Query)
    → NO:  Continue...

[ ] Is the feature a form, table, or navigation element?
    → YES: Standard realm (standard React)
    → NO:  Re-evaluate — most features fit above categories
```

---

## Appendix B: Architecture at a Glance

```
+------------------------------------------------------------------+
|                        NEXT.JS 14+ APP                           |
|                                                                  |
|  +-----------------------+    +-------------------------------+ |
|  |   STANDARD REALM      |    |      AGENTIC REALM           | |
|  |                       |    |                               | |
|  |  /                    |    |  /workspace/[wfId]            | |
|  |  /dashboard           |    |                               | |
|  |  /workflows           |    |  CopilotKit Provider          | |
|  |  /settings            |    |    → useAgent                 | |
|  |                       |    |    → useFrontendTool          | |
|  |  React Query (cache)  |    |    → renderAndWaitForResponse | |
|  |  REST GET/POST        |    |                               | |
|  |  components/standard/ |    |  CanvasProvider               | |
|  |                       |    |    → AG-UI SSE stream         | |
|  |                       |    |    → Workflow Stream SSE      | |
|  |                       |    |    → Unified canvas state     | |
|  |                       |    |                               | |
|  |                       |    |  components/agentic/          | |
|  |                       |    |    → TranscriptCard           | |
|  |                       |    |    → DraftPreviewCard         | |
|  |                       |    |    → HITLResponseCard         | |
|  +-----------------------+    +-------------------------------+ |
|                                                                  |
|  SHARED:                                                         |
|  - components/ui/ (Button, Card, Tile)                          |
|  - Tailwind CSS                                                  |
|  - SharedProvider (user, conversations)                         |
|  - workflow_id (URL param)                                       |
+------------------------------------------------------------------+
```

---

*End of Deliverable 8: Frontend Architecture Guide*
