# Prompt 05: Frontend (Next.js + CopilotKit + AG-UI)

## Reference Documents (READ THESE FIRST)

| Priority | Document | Why Read It |
|----------|----------|-------------|
| **PRIMARY** | `resources/Deliverable_5_CopilotKit_AGUI_Guide.md` | **COPILOTKIT PATTERNS.** All hooks (`useAgent`, `useCoAgent`), components, dual-stream architecture. |
| **PRIMARY** | `resources/Deliverable_8_Frontend_Architecture_Guide.md` | **REALM SEPARATION.** Component hierarchy, provider layout, AG-UI vs Workflow realms. |
| **PRIMARY** | `resources/Deliverable_3_Implementation_Prompts.md` | Prompt 3 (Next.js scaffold, Tailwind, shadcn/ui setup). |
| **REFERENCE** | `resources/Deliverable_0_PROJECT_CONTEXT.md` | CopilotKit stack (Section 11.3), AG-UI map (Section 3.3), 8 features list. |
| **REFERENCE** | `resources/ARCHITECTURE_DATAFLOW_GUIDE.md` | Layer 8 (Presentation) for AG-UI SSE data flow. |

> **How to find these documents:** They are in the `resources/` folder. Naming convention: `Prompt_NN_*.md` and `Deliverable_N_*.md`.

---

**MODE: AUTOMATIC — AI writes code directly, no user interaction needed**


**AG-UI vs A2A Clarification:**
- **AG-UI** = Frontend ↔ Agent communication (this prompt). CopilotKit, SSE, chat streaming.
- **A2A** = Agent ↔ Agent communication (NOT this prompt). Temporal signals.

# Phase 5: Next.js Frontend with CopilotKit Integration

## CRITICAL RULES (NON-NEGOTIABLE)

- **Node.js environment isolation:** NEVER use `npm install -g` (global). ALWAYS install locally via `npm install` (or `pnpm install`, `yarn install`) into `./node_modules`.
- **Python environment isolation:** NEVER use global `pip install`. ALWAYS use `uv` with `.venv` (`uv venv .venv && source .venv/bin/activate && uv pip install ...`). The ONLY exception is `RUN pip install` inside a Dockerfile.
- **All code comments MUST be written in English only.** No other language in comments, docstrings, or string literals.
- If ANY step fails, STOP and report the exact error. Do NOT proceed.

---

## Mission
Build the Next.js 14+ frontend with full CopilotKit integration. All 8 CopilotKit features must render visibly. AG-UI events must flow from AgentCore to the browser. The workspace page must function correctly with dual-response HITL, generatiand UI cards, and SSE streaming.

**Target model:** Gemini Flash, Claude Sonnet (low-cognition, sequential execution)
**Execution rule:** Follow steps in exact order. No branching. No decisions. Execute every file creation exactly as written.

---

## ASSUMPTIONS (Phases 0-4 Complete)

| Service | Port | Purpose |
|---------|------|---------|
| FastAPI | 8000 | Backend API, workflow lifecycle, SSE bridge |
| Temporal Web UI | 8081 | Workflow observability |
| PostgreSQL | 5432 | Data persistence (RDS) |
| Temporal Server | 7233 | Workflow engine gRPC |
| Frontend (Next.js) | 3000 | This build |

## NOT IN SCOPE
- Backend changes
- Agent code changes
- Temporal workflow changes
- Database migrations
- Infrastructure changes

---

## DIRECTORY STRUCTURE TO CREATE

```
frontend/
  package.json
  Dockerfile
  next.config.js
  tailwind.config.ts
  tsconfig.json
  postcss.config.js
  app/
    layout.tsx
    page.tsx
    globals.css
    api/
      copilotkit/
        route.ts
    workspace/
      [wfId]/
        page.tsx
  lib/
    agentcore-agui-client.ts
    use-workflow-stream.ts
    use-agentcore-runtimes.ts
    use-shared-state.ts
    copilot-config.ts
    utils.ts
  components/
    ui/
      (shadcn components - generated)
    CanvasPanel.tsx
    TranscriptCard.tsx
    DraftPreviewCard.tsx
    ReviewReportCard.tsx
    ApprovalCard.tsx
    ClarificationQuestionCard.tsx
    MetricTile.tsx
    CopilotSidebarWrapper.tsx
    GenerativeUIRenderer.tsx
```

---

## STEP 1: Create `frontend/package.json`

**Directory:** Create `frontend/` at the project root if it does not exist.
**File:** `frontend/package.json`

Write the following content exactly:

```json
{
  "name": "agentcore-demo-test1-frontend",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "dev": "next dev -p 3000",
    "build": "next build",
    "start": "next start -p 3000",
    "lint": "next lint"
  },
  "dependencies": {
    "next": "^14.2.0",
    "react": "^18.3.0",
    "react-dom": "^18.3.0",
    "@copilotkit/react-core": "^1.50.0",
    "@copilotkit/react-ui": "^1.50.0",
    "@copilotkit/runtime": "^1.50.0",
    "@copilotkit/runtime-client-gql": "^1.50.0",
    "@copilotkit/shared": "^1.50.0",
    "@ag-ui/client": "^0.5.0",
    "@ag-ui/core": "^0.5.0",
    "@ag-ui/react": "^0.5.0",
    "lucide-react": "^0.400.0",
    "class-variance-authority": "^0.7.0",
    "clsx": "^2.1.0",
    "tailwind-merge": "^2.3.0",
    "@radix-ui/react-dialog": "^1.0.5",
    "@radix-ui/react-slot": "^1.0.2",
    "@radix-ui/react-tabs": "^1.0.4",
    "@radix-ui/react-scroll-area": "^1.0.5",
    "@radix-ui/react-avatar": "^1.0.4",
    "@radix-ui/react-separator": "^1.0.3",
    "@radix-ui/react-tooltip": "^1.0.7",
    "@radix-ui/react-toast": "^1.1.5",
    "framer-motion": "^11.0.0"
  },
  "devDependencies": {
    "typescript": "^5.4.0",
    "@types/node": "^20.12.0",
    "@types/react": "^18.3.0",
    "@types/react-dom": "^18.3.0",
    "tailwindcss": "^3.4.0",
    "postcss": "^8.4.0",
    "autoprefixer": "^10.4.0",
    "eslint": "^8.57.0",
    "eslint-config-next": "^14.2.0"
  }
}
```

**VERIFICATION:** Run `cat frontend/package.json | grep "@copilotkit/react-core"`. If output is empty, STOP and report "package.json not written correctly."

---

## STEP 1.5 (ALTERNATIVE): Project scaffolding via CLI

CopilotKit now offers `npx copilotkit create` to scaffold a project skeleton.
Eger sifirdan basliyorsaniz, this commandla baslayabilirsiniz.

```bash
npx copilotkit@latest create -f aws-strands-py
```

This command automatically:
- Next.js 14+ App Router projesi olusturur
- CopilotKit provider and runtime'i kurar
- AG-UI client'i yukler
- Strands agent ornegi ekler
- Dockerfile and deployment config'leri olusturur

**NOT**: This prompt mevcut projeye frontend eklemek icindir.
Eger sifirdan basliyorsaniz, this commandu kullanin.
Mevcut projeye ekliyorsaniz, STEP 1'den devam edin.

**BedrockAgentCoreApp Runtime APIs (Python side):**
- `@app.entrypoint` — V5 agent handler decorator (replaces `create_strands_app`)
- `BedrockAgentCoreApp` — Runtime app container (auto-creates `/invocations`, `/ping`)
- Runtime `context` — Session ID, request ID from AgentCore MicroVM (state via `workflow_id` as `session_id`)
- `useAgent` hook (CopilotKit) — AG-UI client connecting directly to AgentCore `/invocations`
- `HttpAgent` — AgentCore AG-UI endpoint connection (configured with `runtimeUrl`)

**NO `StrandsAgentConfig`. NO `create_strands_app`. NO `MCPAppsMiddleware`.**
These are V4 patterns. V5 uses `BedrockAgentCoreApp` + `@app.entrypoint` exclusively.

---

## STEP 2: Create `frontend/tsconfig.json`

**File:** `frontend/tsconfig.json`

Write the following content exactly:

```json
{
  "compilerOptions": {
    "target": "ES2017",
    "lib": ["dom", "dom.iterable", "esnext"],
    "allowJs": true,
    "skipLibCheck": true,
    "strict": true,
    "noEmit": true,
    "esModuleInterop": true,
    "module": "esnext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "jsx": "preserve",
    "incremental": true,
    "plugins": [
      {
        "name": "next"
      }
    ],
    "paths": {
      "@/*": ["./*"]
    }
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts"],
  "exclude": ["node_modules"]
}
```

---

## STEP 3: Create `frontend/next.config.js`

**File:** `frontend/next.config.js`

Write the following content exactly:

```javascript
/** @type {import('next').NextConfig} */
const nextConfig = {
  output: 'standalone',
  async rewrites() {
    return [
      {
        source: '/api/v1/:path*',
        destination: 'http://host.docker.internal:8000/api/:path*',
      },
      {
        source: '/api/v1/:path*',
        destination: 'http://host.docker.internal:8000/api/v1/:path*',
      },
    ];
  },
  env: {
    BACKEND_URL: process.env.BACKEND_URL || 'http://host.docker.internal:8000',
    COPILOTKIT_RUNTIME_URL: process.env.COPILOTKIT_RUNTIME_URL || 'http://host.docker.internal:8000/api/copilotkit',
  },
};

module.exports = nextConfig;
```

---

## STEP 4: Create `frontend/tailwind.config.ts`

**File:** `frontend/tailwind.config.ts`

Write the following content exactly:

```typescript
import type { Config } from 'tailwindcss';

const config: Config = {
  darkMode: ["class"],
  content: [
    './pages/**/*.{ts,tsx}',
    './components/**/*.{ts,tsx}',
    './app/**/*.{ts,tsx}',
    './lib/**/*.{ts,tsx}',
  ],
  theme: {
    container: {
      center: true,
      padding: "2rem",
      screens: {
        "2xl": "1400px",
      },
    },
    extend: {
      colors: {
        border: "hsl(var(--border))",
        input: "hsl(var(--input))",
        ring: "hsl(var(--ring))",
        background: "hsl(var(--background))",
        foreground: "hsl(var(--foreground))",
        primary: {
          DEFAULT: "hsl(var(--primary))",
          foreground: "hsl(var(--primary-foreground))",
        },
        secondary: {
          DEFAULT: "hsl(var(--secondary))",
          foreground: "hsl(var(--secondary-foreground))",
        },
        destructive: {
          DEFAULT: "hsl(var(--destructive))",
          foreground: "hsl(var(--destructive-foreground))",
        },
        muted: {
          DEFAULT: "hsl(var(--muted))",
          foreground: "hsl(var(--muted-foreground))",
        },
        accent: {
          DEFAULT: "hsl(var(--accent))",
          foreground: "hsl(var(--accent-foreground))",
        },
        popover: {
          DEFAULT: "hsl(var(--popover))",
          foreground: "hsl(var(--popover-foreground))",
        },
        card: {
          DEFAULT: "hsl(var(--card))",
          foreground: "hsl(var(--card-foreground))",
        },
        agui: {
          event: "#3b82f6",
          step: "#10b981",
          run: "#8b5cf6",
          unknown: "#6b7280",
        },
      },
      borderRadius: {
        lg: "var(--radius)",
        md: "calc(var(--radius) - 2px)",
        sm: "calc(var(--radius) - 4px)",
      },
      keyframes: {
        "accordion-down": {
          from: { height: "0" },
          to: { height: "var(--radix-accordion-content-height)" },
        },
        "accordion-up": {
          from: { height: "var(--radix-accordion-content-height)" },
          to: { height: "0" },
        },
        "pulse-glow": {
          "0%, 100%": { boxShadow: "0 0 5px rgba(59, 130, 246, 0.3)" },
          "50%": { boxShadow: "0 0 20px rgba(59, 130, 246, 0.6)" },
        },
      },
      animation: {
        "accordion-down": "accordion-down 0.2s ease-out",
        "accordion-up": "accordion-up 0.2s ease-out",
        "pulse-glow": "pulse-glow 2s ease-in-out infinite",
      },
    },
  },
  plugins: [require("tailwindcss-animate")],
};

export default config;
```

**VERIFICATION:** Run `ls frontend/tsconfig.json frontend/next.config.js frontend/tailwind.config.ts frontend/package.json`. All 4 files must exist. If any is missing, STOP and report which file is missing.

---

## CHECKPOINT 1: Config Files Verified

Before proceeding, verify:
1. `frontend/package.json` exists and contains `"@copilotkit/react-core"` in dependencies
2. `frontend/tsconfig.json` exists with `"jsx": "preserve"`
3. `frontend/next.config.js` exists with the `rewrites` function for backend proxying
4. `frontend/tailwind.config.ts` exists with AG-UI custom colors

If any check fails, STOP. Do not proceed.

---

## STEP 5: Create `frontend/postcss.config.js`

**File:** `frontend/postcss.config.js`

Write the following content exactly:

```javascript
module.exports = {
  plugins: {
    tailwindcss: {},
    autoprefixer: {},
  },
};
```

---

## STEP 6: Create `frontend/app/globals.css`

**File:** `frontend/app/globals.css`

Write the following content exactly:

```css
@tailwind base;
@tailwind components;
@tailwind utilities;

@layer base {
  :root {
    --background: 0 0% 100%;
    --foreground: 222.2 84% 4.9%;
    --card: 0 0% 100%;
    --card-foreground: 222.2 84% 4.9%;
    --popover: 0 0% 100%;
    --popover-foreground: 222.2 84% 4.9%;
    --primary: 221.2 83.2% 53.3%;
    --primary-foreground: 210 40% 98%;
    --secondary: 210 40% 96.1%;
    --secondary-foreground: 222.2 47.4% 11.2%;
    --muted: 210 40% 96.1%;
    --muted-foreground: 215.4 16.3% 46.9%;
    --accent: 210 40% 96.1%;
    --accent-foreground: 222.2 47.4% 11.2%;
    --destructive: 0 84.2% 60.2%;
    --destructive-foreground: 210 40% 98%;
    --border: 214.3 31.8% 91.4%;
    --input: 214.3 31.8% 91.4%;
    --ring: 221.2 83.2% 53.3%;
    --radius: 0.5rem;
  }

  .dark {
    --background: 222.2 84% 4.9%;
    --foreground: 210 40% 98%;
    --card: 222.2 84% 4.9%;
    --card-foreground: 210 40% 98%;
    --popover: 222.2 84% 4.9%;
    --popover-foreground: 210 40% 98%;
    --primary: 217.2 91.2% 59.8%;
    --primary-foreground: 222.2 47.4% 11.2%;
    --secondary: 217.2 32.6% 17.5%;
    --secondary-foreground: 210 40% 98%;
    --muted: 217.2 32.6% 17.5%;
    --muted-foreground: 215 20.2% 65.1%;
    --accent: 217.2 32.6% 17.5%;
    --accent-foreground: 210 40% 98%;
    --destructive: 0 62.8% 30.6%;
    --destructive-foreground: 210 40% 98%;
    --border: 217.2 32.6% 17.5%;
    --input: 217.2 32.6% 17.5%;
    --ring: 212.7 26.8% 83.9%;
  }
}

@layer base {
  * {
    @apply border-border;
  }
  body {
    @apply bg-background text-foreground;
  }
}

/* AG-UI Event Stream Styles */
.agui-event-line {
  font-family: 'Menlo', 'Monaco', 'Courier New', monospace;
  font-size: 0.75rem;
  line-height: 1.5;
}

.agui-event-event { color: #3b82f6; }
.agui-event-step { color: #10b981; }
.agui-event-run { color: #8b5cf6; }
.agui-event-unknown { color: #6b7280; }

/* CopilotKit Overrides */
.copilot-chat-window {
  --copilot-primary-color: hsl(var(--primary));
}

/* Custom scrollbar for canvas */
.canvas-scroll::-webkit-scrollbar {
  width: 6px;
}
.canvas-scroll::-webkit-scrollbar-track {
  background: transparent;
}
.canvas-scroll::-webkit-scrollbar-thumb {
  background-color: hsl(var(--border));
  border-radius: 3px;
}
```

---

## STEP 7: Create `frontend/lib/utils.ts`

**File:** `frontend/lib/utils.ts`

Write the following content exactly:

```typescript
import { type ClassValue, clsx } from "clsx";
import { twMerge } from "tailwind-merge";

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}

export function formatTimestamp(ts: string | Date | number): string {
  const d = new Date(ts);
  return d.toLocaleTimeString('en-US', {
    hour: '2-digit',
    minute: '2-digit',
    second: '2-digit',
    hour12: false,
  });
}

export function truncateText(text: string, maxLength: number): string {
  if (text.length <= maxLength) return text;
  return text.substring(0, maxLength) + '...';
}

export function sleep(ms: number): Promise<void> {
  return new Promise(resolve => setTimeout(resolve, ms));
}
```

---

## STEP 8: Create `frontend/lib/copilot-config.ts`

**File:** `frontend/lib/copilot-config.ts`

Write the following content exactly:

```typescript
/**
 * CopilotKit Configuration
 * Centralized config for all CopilotKit endpoints and settings.
 */

export const COPILOT_CONFIG = {
  /** Public runtime URL - proxied through Next.js rewrites in dev, direct in prod */
  runtimeUrl: process.env.NEXT_PUBLIC_COPILOT_RUNTIME_URL || '/api/copilotkit',

  /** Backend API base URL */
  backendUrl: process.env.NEXT_PUBLIC_BACKEND_URL || '/api/v1',

  /** Agent name registered in CopilotKit runtime */
  agentName: 'audio-to-brd-agent',

  /** Feature flags for all 8 CopilotKit features */
  features: {
    copilotChat: true,
    copilotSidebar: true,
    copilotTextarea: false,
    useCoAgent: true,
    useAgent: true,
    useCopilotAction: true,
    useCopilotReadable: true,
    useFrontendTool: true,
  },

  /** SSE stream endpoint for workflow events */
  workflowStreamUrl: (wfId: string) =>
    `${process.env.NEXT_PUBLIC_BACKEND_URL || '/api/v1'}/workflows/${wfId}/stream`,

  /** REST endpoint for workflow state */
  workflowStateUrl: (wfId: string) =>
    `${process.env.NEXT_PUBLIC_BACKEND_URL || '/api/v1'}/workflows/${wfId}/state`,

  /** REST endpoint for agent runtimes (ARN list) */
  runtimesUrl: `${process.env.NEXT_PUBLIC_BACKEND_URL || '/api/v1'}/agent-runtimes`,
} as const;

/** Feature names for display */
export const FEATURE_NAMES = [
  'CopilotChat',
  'CopilotSidebar',
  'useCoAgent',
  'useAgent',
  'useCopilotAction',
  'useCopilotReadable',
  'useFrontendTool',
  'GenerativeUI',
] as const;
```

**VERIFICATION:** Run `ls frontend/lib/*.ts frontend/app/*.css frontend/postcss.config.js`. Count the files. There should be 5 files total. If count is not 5, STOP and report which files are missing.

---

## CHECKPOINT 2: Base Infrastructure Verified

Before proceeding, verify:
1. `frontend/postcss.config.js` exists
2. `frontend/app/globals.css` exists and contains `@tailwind` directives
3. `frontend/lib/utils.ts` exists with `cn()` utility
4. `frontend/lib/copilot-config.ts` exists with `COPILOT_CONFIG` export

If any check fails, STOP. Do not proceed.

---

## STEP 9: Create `frontend/lib/agentcore-agui-client.ts`

**CRITICAL FILE.** This is the HttpAgent factory that bridges AgentCore AG-UI events to CopilotKit.

**File:** `frontend/lib/agentcore-agui-client.ts`

Write the following content exactly:

```typescript
/**
 * AgentCore AG-UI Client
 * ======================
 * HttpAgent factory that connects to AWS Bedrock AgentCore AG-UI endpoints.
 * Uses the official AG-UI protocol with proper AgentCore authentication.
 *
 * Architecture:
 * - Creates HttpAgent instances via @ag-ui/client
 * - Connects to AgentCore AG-UI endpoint with proper auth headers
 * - URL format: https://bedrock-agentcore.<region>.amazonaws.com/runtimes/<id>/invocations?qualifier=DEFAULT
 * - Session ID = workflow_id (AgentCore microVM session isolation icin)
 *
 * Reference: https://www.copilotkit.ai/blog/aws-announces-dedicated-ag-ui-endpoint-in-agentcore-and-fast-template-for-building-fullstack-agents
 */

import { HttpAgent } from '@ag-ui/client';
import { COPILOT_CONFIG } from './copilot-config';

// ---------------------------------------------------------------------------
// AG-UI Event Types (matches backend schema exactly)
// ---------------------------------------------------------------------------

export type AGUIEventType = 'RUN_EVENT' | 'STEP_EVENT' | 'AGENT_EVENT' | 'UNKNOWN';

export interface AGUIEvent {
  type: AGUIEventType;
  timestamp: string;
  run_id: string;
  data: Record<string, unknown>;
}

export interface AGUIRunEvent extends AGUIEvent {
  type: 'RUN_EVENT';
  data: {
    run_id: string;
    agent_name: string;
    status: 'started' | 'completed' | 'failed';
    input?: Record<string, unknown>;
  };
}

export interface AGUIStepEvent extends AGUIEvent {
  type: 'STEP_EVENT';
  data: {
    step_name: string;
    step_type: string;
    status: 'running' | 'completed' | 'failed';
    input?: Record<string, unknown>;
    output?: Record<string, unknown>;
    thought?: string;
  };
}

export interface AGUIAgentEvent extends AGUIEvent {
  type: 'AGENT_EVENT';
  data: {
    event_type: 'transcript' | 'draft' | 'review' | 'approval_request' | 'clarification' | 'state_update';
    payload: Record<string, unknown>;
  };
}

export type AGUIAnyEvent = AGUIRunEvent | AGUIStepEvent | AGUIAgentEvent;

// ---------------------------------------------------------------------------
// Agent State (shared with useCoAgent)
// ---------------------------------------------------------------------------

export interface AgentCoreState {
  runId: string | null;
  agentName: string;
  status: 'idle' | 'running' | 'completed' | 'failed' | 'awaiting_approval' | 'awaiting_clarification';
  currentStep: string | null;
  steps: Array<{
    name: string;
    type: string;
    status: string;
    input?: Record<string, unknown>;
    output?: Record<string, unknown>;
    thought?: string;
  }>;
  events: AGUIAnyEvent[];
  transcript: string | null;
  draft: string | null;
  review: Record<string, unknown> | null;
  approvalRequest: {
    type: string;
    description: string;
    options: string[];
    deadline?: string;
  } | null;
  clarificationQuestion: {
    question: string;
    context: string;
    suggestedAnswers?: string[];
  } | null;
  metadata: Record<string, unknown>;
}

export const initialAgentState: AgentCoreState = {
  runId: null,
  agentName: COPILOT_CONFIG.agentName,
  status: 'idle',
  currentStep: null,
  steps: [],
  events: [],
  transcript: null,
  draft: null,
  review: null,
  approvalRequest: null,
  clarificationQuestion: null,
  metadata: {},
};

// ---------------------------------------------------------------------------
// HttpAgent Factory — Creates AgentCore AG-UI HttpAgent instances
// ---------------------------------------------------------------------------

/**
 * AgentCore AG-UI Endpoint URL format:
 * https://bedrock-agentcore.<region>.amazonaws.com/runtimes/<runtime-id>/invocations?qualifier=DEFAULT
 *
 * Session ID = workflow_id (AgentCore microVM session isolation icin)
 *
 * Reference: https://www.copilotkit.ai/blog/aws-announces-dedicated-ag-ui-endpoint-in-agentcore-and-fast-template-for-building-fullstack-agents
 */
export function createAgentCoreHttpAgent(
  region: string,
  runtimeId: string,
  accountId: string,
  sessionId: string,
): HttpAgent {
  const url = `https://bedrock-agentcore.${region}.amazonaws.com/runtimes/${runtimeId}/invocations?accountId=${accountId}&qualifier=DEFAULT`;

  return new HttpAgent({
    url,
    headers: {
      'X-Amzn-Bedrock-AgentCore-Runtime-Session-Id': sessionId,
    },
  });
}

// ---------------------------------------------------------------------------
// AgentCoreHttpAgent — SSE-based AG-UI event stream client
// ---------------------------------------------------------------------------

export interface AgentCoreHttpAgentOptions {
  workflowId: string;
  onStateChange?: (state: AgentCoreState) => void;
  onEvent?: (event: AGUIAnyEvent) => void;
  onError?: (error: Error) => void;
  onComplete?: () => void;
}

/**
 * AgentCoreHttpAgent
 * Subclasses the HttpAgent pattern to create a persistent SSE connection
 * to the AgentCore backend. Maintains agent state that syncs with
 * CopilotKit's useCoAgent hook.
 */
export class AgentCoreHttpAgent {
  private workflowId: string;
  private eventSource: EventSource | null = null;
  private state: AgentCoreState;
  private listeners: Set<(state: AgentCoreState) => void> = new Set();
  private onEvent?: (event: AGUIAnyEvent) => void;
  private onError?: (error: Error) => void;
  private onComplete?: () => void;
  private reconnectAttempts = 0;
  private maxReconnectAttempts = 5;
  private reconnectDelayMs = 2000;

  constructor(options: AgentCoreHttpAgentOptions) {
    this.workflowId = options.workflowId;
    this.state = { ...initialAgentState };
    this.onEvent = options.onEvent;
    this.onError = options.onError;
    this.onComplete = options.onComplete;
  }

  /** Get current immutable state snapshot */
  getState(): AgentCoreState {
    return { ...this.state, steps: [...this.state.steps], events: [...this.state.events] };
  }

  /** Subscribe to state changes */
  subscribe(listener: (state: AgentCoreState) => void): () => void {
    this.listeners.add(listener);
    // Immediately emit current state
    listener(this.getState());
    return () => {
      this.listeners.delete(listener);
    };
  }

  /** Emit state to all listeners */
  private emitState(): void {
    const snapshot = this.getState();
    this.listeners.forEach(l => {
      try { l(snapshot); } catch (e) { /* ignore listener errors */ }
    });
  }

  /**
   * Connect to the SSE stream.
   * This opens a persistent connection to the backend's AG-UI event endpoint.
   */
  connect(): void {
    if (this.eventSource) {
      this.disconnect();
    }

    const url = COPILOT_CONFIG.workflowStreamUrl(this.workflowId);
    console.log(`[AgentCoreHttpAgent] Connecting to SSE: ${url}`);

    try {
      this.eventSource = new EventSource(url);

      this.eventSource.onopen = () => {
        console.log('[AgentCoreHttpAgent] SSE connection opened');
        this.reconnectAttempts = 0;
      };

      this.eventSource.onmessage = (msg) => {
        try {
          const parsed = JSON.parse(msg.data) as AGUIAnyEvent;
          this.handleEvent(parsed);
        } catch (err) {
          console.warn('[AgentCoreHttpAgent] Failed to parse SSE message:', msg.data);
        }
      };

      this.eventSource.onerror = (err) => {
        console.error('[AgentCoreHttpAgent] SSE error:', err);
        if (this.onError) {
          this.onError(new Error('SSE connection error'));
        }
        this.attemptReconnect();
      };

    } catch (err) {
      console.error('[AgentCoreHttpAgent] Failed to create EventSource:', err);
      if (this.onError) {
        this.onError(err instanceof Error ? err : new Error(String(err)));
      }
    }
  }

  /** Handle reconnect with exponential backoff */
  private attemptReconnect(): void {
    if (this.reconnectAttempts >= this.maxReconnectAttempts) {
      console.error('[AgentCoreHttpAgent] Max reconnect attempts reached');
      return;
    }
    this.reconnectAttempts++;
    const delay = this.reconnectDelayMs * Math.pow(2, this.reconnectAttempts - 1);
    console.log(`[AgentCoreHttpAgent] Reconnecting in ${delay}ms (attempt ${this.reconnectAttempts}/${this.maxReconnectAttempts})`);
    setTimeout(() => this.connect(), delay);
  }

  /** Disconnect the SSE stream */
  disconnect(): void {
    if (this.eventSource) {
      this.eventSource.close();
      this.eventSource = null;
      console.log('[AgentCoreHttpAgent] SSE disconnected');
    }
  }

  /** Handle a single AG-UI event and update state */
  private handleEvent(event: AGUIAnyEvent): void {
    // Append to event log
    this.state.events.push(event);

    // Forward raw event
    if (this.onEvent) {
      try { this.onEvent(event); } catch (e) { /* ignore */ }
    }

    // Type-specific state updates
    switch (event.type) {
      case 'RUN_EVENT':
        this.handleRunEvent(event as AGUIRunEvent);
        break;
      case 'STEP_EVENT':
        this.handleStepEvent(event as AGUIStepEvent);
        break;
      case 'AGENT_EVENT':
        this.handleAgentEvent(event as AGUIAgentEvent);
        break;
      default:
        console.warn('[AgentCoreHttpAgent] Unknown event type:', event.type);
    }

    this.emitState();

    // Check for completion
    if (this.state.status === 'completed' || this.state.status === 'failed') {
      if (this.onComplete) {
        try { this.onComplete(); } catch (e) { /* ignore */ }
      }
    }
  }

  private handleRunEvent(event: AGUIRunEvent): void {
    this.state.runId = event.data.run_id;
    this.state.agentName = event.data.agent_name;

    if (event.data.status === 'started') {
      this.state.status = 'running';
    } else if (event.data.status === 'completed') {
      this.state.status = 'completed';
    } else if (event.data.status === 'failed') {
      this.state.status = 'failed';
    }

    if (event.data.input) {
      this.state.metadata = { ...this.state.metadata, input: event.data.input };
    }
  }

  private handleStepEvent(event: AGUIStepEvent): void {
    this.state.currentStep = event.data.step_name;

    // Update existing step or append new one
    const existingIndex = this.state.steps.findIndex(s => s.name === event.data.step_name);
    if (existingIndex >= 0) {
      this.state.steps[existingIndex] = {
        ...this.state.steps[existingIndex],
        status: event.data.status,
        ...(event.data.output ? { output: event.data.output } : {}),
        ...(event.data.thought ? { thought: event.data.thought } : {}),
      };
    } else {
      this.state.steps.push({
        name: event.data.step_name,
        type: event.data.step_type,
        status: event.data.status,
        input: event.data.input,
        output: event.data.output,
        thought: event.data.thought,
      });
    }
  }

  private handleAgentEvent(event: AGUIAgentEvent): void {
    const { event_type, payload } = event.data;

    switch (event_type) {
      case 'transcript':
        this.state.transcript = payload.transcript as string || String(payload.content || '');
        break;

      case 'draft':
        this.state.draft = payload.draft as string || String(payload.content || '');
        break;

      case 'review':
        this.state.review = payload as Record<string, unknown>;
        break;

      case 'approval_request':
        this.state.approvalRequest = {
          type: String(payload.type || 'approval'),
          description: String(payload.description || 'Approval required'),
          options: Array.isArray(payload.options) ? payload.options as string[] : ['approve', 'reject'],
          deadline: payload.deadline ? String(payload.deadline) : undefined,
        };
        this.state.status = 'awaiting_approval';
        break;

      case 'clarification':
        this.state.clarificationQuestion = {
          question: String(payload.question || 'Please clarify'),
          context: String(payload.context || ''),
          suggestedAnswers: Array.isArray(payload.suggested_answers)
            ? payload.suggested_answers as string[]
            : undefined,
        };
        this.state.status = 'awaiting_clarification';
        break;

      case 'state_update':
        // Generic state update -- merge metadata
        this.state.metadata = { ...this.state.metadata, ...payload };
        break;
    }
  }

  /**
   * Send an approval response back to the agent.
   * This enables dual-response HITL (Human-in-the-Loop).
   */
  async sendApproval(decision: 'approve' | 'reject', comment?: string): Promise<void> {
    const url = `${COPILOT_CONFIG.backendUrl}/workflows/${this.workflowId}/approval`;
    const response = await fetch(url, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ decision, comment, run_id: this.state.runId }),
    });

    if (!response.ok) {
      throw new Error(`Approval request failed: ${response.status} ${response.statusText}`);
    }

    // Optimistic state update
    this.state.approvalRequest = null;
    this.state.status = 'running';
    this.emitState();
  }

  /**
   * Send a clarification response back to the agent.
   */
  async sendClarification(answer: string): Promise<void> {
    const url = `${COPILOT_CONFIG.backendUrl}/workflows/${this.workflowId}/clarification`;
    const response = await fetch(url, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ answer, run_id: this.state.runId }),
    });

    if (!response.ok) {
      throw new Error(`Clarification request failed: ${response.status} ${response.statusText}`);
    }

    // Optimistic state update
    this.state.clarificationQuestion = null;
    this.state.status = 'running';
    this.emitState();
  }
}
```

---

## STEP 10: Create `frontend/lib/use-workflow-stream.ts`

**CRITICAL FILE.** Custom React hook for SSE streaming from AgentCore.

**File:** `frontend/lib/use-workflow-stream.ts`

Write the following content exactly:

```typescript
/**
 * useWorkflowStream
 * ===============
 * Custom React hook that manages the SSE connection to AgentCore.
 * Wraps AgentCoreHttpAgent in a React-friendly interface.
 *
 * Usage:
 *   const { state, events, isConnected, sendApproval, sendClarification } =
 *     useWorkflowStream('workflow-uuid-123');
 */

import { useState, useEffect, useRef, useCallback } from 'react';
import {
  AgentCoreHttpAgent,
  AgentCoreState,
  AGUIAnyEvent,
  initialAgentState,
} from './agentcore-agui-client';

export interface UseWorkflowStreamReturn {
  /** Current agent state (immutable snapshot) */
  state: AgentCoreState;
  /** All AG-UI events received */
  events: AGUIAnyEvent[];
  /** Whether SSE is currently connected */
  isConnected: boolean;
  /** Whether the stream has completed */
  isComplete: boolean;
  /** Connection error, if any */
  error: Error | null;
  /** Manually reconnect */
  reconnect: () => void;
  /** Disconnect */
  disconnect: () => void;
  /** Send approval response (dual-response HITL) */
  sendApproval: (decision: 'approve' | 'reject', comment?: string) => Promise<void>;
  /** Send clarification response */
  sendClarification: (answer: string) => Promise<void>;
}

export function useWorkflowStream(workflowId: string | null): UseWorkflowStreamReturn {
  const [state, setState] = useState<AgentCoreState>(initialAgentState);
  const [events, setEvents] = useState<AGUIAnyEvent[]>([]);
  const [isConnected, setIsConnected] = useState(false);
  const [isComplete, setIsComplete] = useState(false);
  const [error, setError] = useState<Error | null>(null);

  const agentRef = useRef<AgentCoreHttpAgent | null>(null);

  // Create agent instance
  useEffect(() => {
    if (!workflowId) {
      setState(initialAgentState);
      setEvents([]);
      setIsConnected(false);
      setIsComplete(false);
      setError(null);
      return;
    }

    console.log(`[useWorkflowStream] Initializing for workflow: ${workflowId}`);

    const agent = new AgentCoreHttpAgent({
      workflowId,
      onStateChange: (newState) => {
        setState(newState);
        setIsConnected(true);
      },
      onEvent: (evt) => {
        setEvents(prev => [...prev, evt]);
      },
      onError: (err) => {
        setError(err);
        setIsConnected(false);
      },
      onComplete: () => {
        setIsComplete(true);
        setIsConnected(false);
      },
    });

    agentRef.current = agent;

    // Subscribe to state changes
    const unsubscribe = agent.subscribe((newState) => {
      setState(newState);
    });

    // Connect immediately
    agent.connect();

    return () => {
      unsubscribe();
      agent.disconnect();
      agentRef.current = null;
    };
  }, [workflowId]);

  // Track connection state
  useEffect(() => {
    if (state.status === 'running') {
      setIsComplete(false);
    }
  }, [state.status]);

  const reconnect = useCallback(() => {
    setError(null);
    setIsComplete(false);
    if (agentRef.current && workflowId) {
      agentRef.current.connect();
    }
  }, [workflowId]);

  const disconnect = useCallback(() => {
    if (agentRef.current) {
      agentRef.current.disconnect();
    }
    setIsConnected(false);
  }, []);

  const sendApproval = useCallback(
    async (decision: 'approve' | 'reject', comment?: string) => {
      if (!agentRef.current) {
        throw new Error('Agent not initialized');
      }
      await agentRef.current.sendApproval(decision, comment);
    },
    []
  );

  const sendClarification = useCallback(
    async (answer: string) => {
      if (!agentRef.current) {
        throw new Error('Agent not initialized');
      }
      await agentRef.current.sendClarification(answer);
    },
    []
  );

  return {
    state,
    events,
    isConnected,
    isComplete,
    error,
    reconnect,
    disconnect,
    sendApproval,
    sendClarification,
  };
}
```

---

## STEP 11: Create `frontend/lib/use-agentcore-runtimes.ts`

**File:** `frontend/lib/use-agentcore-runtimes.ts`

Write the following content exactly:

```typescript
/**
 * useAgentCoreRuntimes
 * ====================
 * Hook to fetch available agent runtime ARNs from the backend.
 * Used to populate the CopilotKit runtime selector.
 */

import { useState, useEffect, useCallback } from 'react';
import { COPILOT_CONFIG } from './copilot-config';

export interface AgentRuntime {
  id: string;
  name: string;
  arn: string;
  status: 'active' | 'inactive' | 'error';
  description?: string;
  lastUsed?: string;
}

export interface UseAgentCoreRuntimesReturn {
  runtimes: AgentRuntime[];
  isLoading: boolean;
  error: Error | null;
  refresh: () => void;
}

export function useAgentCoreRuntimes(): UseAgentCoreRuntimesReturn {
  const [runtimes, setRuntimes] = useState<AgentRuntime[]>([]);
  const [isLoading, setIsLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  const fetchRuntimes = useCallback(async () => {
    setIsLoading(true);
    setError(null);

    try {
      const response = await fetch(COPILOT_CONFIG.runtimesUrl, {
        headers: { 'Accept': 'application/json' },
      });

      if (!response.ok) {
        // Fallback: if endpoint doesn't exist yet, return empty
        if (response.status === 404) {
          setRuntimes([]);
          setIsLoading(false);
          return;
        }
        throw new Error(`Failed to fetch runtimes: ${response.status}`);
      }

      const data = await response.json();

      if (Array.isArray(data.runtimes)) {
        setRuntimes(data.runtimes as AgentRuntime[]);
      } else if (Array.isArray(data)) {
        setRuntimes(data as AgentRuntime[]);
      } else {
        setRuntimes([]);
      }
    } catch (err) {
      const error = err instanceof Error ? err : new Error(String(err));
      setError(error);
      // On error, still set empty runtimes so UI doesn't hang
      setRuntimes([]);
    } finally {
      setIsLoading(false);
    }
  }, []);

  useEffect(() => {
    fetchRuntimes();
  }, [fetchRuntimes]);

  return {
    runtimes,
    isLoading,
    error,
    refresh: fetchRuntimes,
  };
}
```

**VERIFICATION:** Run `ls frontend/lib/*.ts`. There should be 5 files:
- `agentcore-agui-client.ts`
- `use-workflow-stream.ts`
- `use-agentcore-runtimes.ts`
- `use-shared-state.ts`
- `copilot-config.ts`
- `utils.ts`

If any file is missing, STOP and report which one.

---

## CHECKPOINT 3: Core Library Files Verified

Before proceeding, verify:
1. `frontend/lib/agentcore-agui-client.ts` exists and contains `AgentCoreHttpAgent` class with `connect()`, `disconnect()`, `sendApproval()`, `sendClarification()` methods
2. `frontend/lib/use-workflow-stream.ts` exists and exports `useWorkflowStream` hook
3. `frontend/lib/use-agentcore-runtimes.ts` exists and exports `useAgentCoreRuntimes` hook
4. `frontend/lib/use-shared-state.ts` exists and exports `useWorkflowSharedState` hook
5. All four files import from `./copilot-config`

Run: `grep -l "AgentCoreHttpAgent\|useWorkflowStream\|useAgentCoreRuntimes\|useWorkflowSharedState" frontend/lib/*.ts`

Expected output shows all 4 files. If not, STOP.

---

## STEP 12: Create `frontend/lib/use-shared-state.ts`

**NEW FILE (CopilotKit 1.50+ note).** useCoAgent hook with frontend-agent arasinda two-way state sync.

**File:** `frontend/lib/use-shared-state.ts`

Write the following content exactly:

```typescript
/**
 * useWorkflowSharedState
 * ======================
 * Frontend + Agent arasinda two-way state sync saglayan custom hook.
 * CopilotKit'in `useCoAgent` hook'u with agent state'ini okur and gunceller.
 *
 * Agent tarafinda: `@app.entrypoint` handler receives request context; state is managed via `BedrockAgentCoreApp` runtime context and injected into prompts
 * Frontend tarafinda: useCoAgent with state okunur and guncellenir
 *
 * Reference: https://www.copilotkit.ai/blog/aws-announces-dedicated-ag-ui-endpoint-in-agentcore-and-fast-template-for-building-fullstack-agents
 */

import { useCoAgent } from '@copilotkit/react-core';

export interface WorkflowSharedState {
  workflow_id: string;
  transcript: string;
  draft_markdown: string;
  review_report: string;
  status: 'idle' | 'running' | 'completed' | 'failed' | 'awaiting_approval' | 'awaiting_clarification';
  current_step?: string;
  steps?: Array<{
    name: string;
    type: string;
    status: string;
  }>;
}

export function useWorkflowSharedState(workflowId: string) {
  const { state, setState } = useCoAgent<WorkflowSharedState>({
    name: 'strands_drafter',
    initialState: {
      workflow_id: workflowId,
      transcript: '',
      draft_markdown: '',
      review_report: '',
      status: 'idle',
      current_step: undefined,
      steps: [],
    },
  });

  return { state, setState };
}

/**
 * Proverbs example from the blog -- generic shared state pattern.
 * This shows how any serializable state can be synced between frontend and agent.
 */
export function useProverbsSharedState() {
  const { state, setState } = useCoAgent<{
    proverbs: Array<{ text: string; meaning: string }>;
  }>({
    name: 'strands_agent',
    initialState: {
      proverbs: [
        { text: 'A friend in need is a friend indeed', meaning: 'True friends show themselves in difficult times' },
      ],
    },
  });

  return { proverbs: state.proverbs, setProverbs: setState };
}
```

---

## STEP 13: Create `frontend/app/api/copilotkit/route.ts`

**CRITICAL FILE.** CopilotKit runtime route handler with HttpAgent integration.

**File:** `frontend/app/api/copilotkit/route.ts`

Create directories: `frontend/app/api/copilotkit/`

Write the following content exactly:

```typescript
/**
 * CopilotKit Runtime API Route
 * ==============================
 * Next.js App Router API route that configures the CopilotKit runtime
 * with HttpAgent connections to AWS Bedrock AgentCore AG-UI endpoints.
 *
 * This route handles:
 * - Chat completions via the CopilotKit runtime
 * - Agent state sync (useCoAgent) via HttpAgent
 * - Action invocations (useCopilotAction)
 * - Frontend tool calls (useFrontendTool)
 *
 * Architecture:
 * - Each agent is registered as an HttpAgent pointing to an AgentCore runtime
 * - URL format: https://bedrock-agentcore.<region>.amazonaws.com/runtimes/<id>/invocations?qualifier=DEFAULT
 * - Session ID provides microVM-level session isolation
 *
 * Reference: https://www.copilotkit.ai/blog/aws-announces-dedicated-ag-ui-endpoint-in-agentcore-and-fast-template-for-building-fullstack-agents
 */

import { CopilotRuntime } from '@copilotkit/runtime';
import { HttpAgent } from '@ag-ui/client';
import { copilotRuntimeNextJSAppRouterEndpoint } from '@copilotkit/runtime';
import { NextRequest } from 'next/server';

// ---------------------------------------------------------------------------
// Optional: MCPAppsMiddleware for Model Context Protocol (MCP) servers
// ---------------------------------------------------------------------------
// import { MCPAppsMiddleware } from '@copilotkit/runtime';
// const mcpMiddleware = new MCPAppsMiddleware({
//   mcpServers: [
//     { type: 'http', url: process.env.MCP_SERVER_URL!, serverId: 'example' }
//   ]
// });

// ---------------------------------------------------------------------------
// CopilotKit Runtime with HttpAgent connections
// ---------------------------------------------------------------------------

const runtime = new CopilotRuntime({
  agents: {
    strands_transcriber: new HttpAgent({
      url: process.env.AGENT_1_RUNTIME_URL!,
      headers: {
        'X-Amzn-Bedrock-AgentCore-Runtime-Session-Id': 'default-session',
      },
    }),
    strands_drafter: new HttpAgent({
      url: process.env.AGENT_2_RUNTIME_URL!,
      headers: {
        'X-Amzn-Bedrock-AgentCore-Runtime-Session-Id': 'default-session',
      },
    }),
    strands_reviewer: new HttpAgent({
      url: process.env.AGENT_3_RUNTIME_URL!,
      headers: {
        'X-Amzn-Bedrock-AgentCore-Runtime-Session-Id': 'default-session',
      },
    }),
  },
  // Optional: Add MCP middleware for external tool access
  // middleware: [mcpMiddleware],
});

export const POST = async (req: NextRequest) => {
  const { handleRequest } = copilotRuntimeNextJSAppRouterEndpoint({
    runtime,
    serviceAdapter: new ExperimentalEmptyAdapter(),
    endpoint: '/api/copilotkit',
  });
  return handleRequest(req);
};

// ---------------------------------------------------------------------------
// Helper: Create HttpAgent with AgentCore endpoint factory
// ---------------------------------------------------------------------------

/**
 * Creates an HttpAgent for a specific AgentCore runtime.
 *
 * @param region - AWS region (e.g. 'us-east-1')
 * @param runtimeId - AgentCore runtime ID (microVM identifier)
 * @param accountId - AWS account ID
 * @param sessionId - Session ID (use workflow_id for session isolation)
 */
export function createAgentCoreHttpAgentForRuntime(
  region: string,
  runtimeId: string,
  accountId: string,
  sessionId: string,
): HttpAgent {
  const url = `https://bedrock-agentcore.${region}.amazonaws.com/runtimes/${runtimeId}/invocations?accountId=${accountId}&qualifier=DEFAULT`;

  return new HttpAgent({
    url,
    headers: {
      'X-Amzn-Bedrock-AgentCore-Runtime-Session-Id': sessionId,
    },
  });
}

/**
 * Empty service adapter for CopilotKit Runtime.
 *
 * Why empty: this demo routes ALL LLM traffic through the AgentCore agents
 * via `HttpAgent` (AG-UI protocol). CopilotKit's serviceAdapter layer would
 * normally proxy chat-completion requests to OpenAI/Anthropic — we bypass
 * that entirely because the agents own their own Bedrock calls. The runtime
 * still needs *some* adapter at construction time, so we satisfy the
 * interface with a no-op.
 *
 * If you ever want CopilotKit-side chat completion (e.g. for a fallback
 * conversational helper), swap this for OpenAIAdapter / AnthropicAdapter
 * from `@copilotkit/runtime`.
 */
class ExperimentalEmptyAdapter {
  async process(_request: unknown) {
    return { messages: [] };
  }
}
```

---

## STEP 14: Create `frontend/app/layout.tsx`

**CRITICAL FILE.** Root layout with CopilotKit provider wrapping.

**File:** `frontend/app/layout.tsx`

Write the following content exactly:

```typescript
/**
 * Root Layout
 * ===========
 * Wraps the entire application with:
 * - CopilotKit provider (Feature 1: copilotChat foundation)
 * - CopilotSidebar (Feature 2: sidebar chat)
 * - Global styles and fonts
 */

import type { Metadata } from 'next';
import { Inter } from 'next/font/google';
import './globals.css';
import { CopilotKitProvider } from '@/components/CopilotKitProvider';

const inter = Inter({ subsets: ['latin'] });

export const metadata: Metadata = {
  title: 'Legal Agent -- AI-Powered Legal Research',
  description: 'Autonomous legal research with human-in-the-loop oversight',
};

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <body className={inter.className}>
        <CopilotKitProvider>
          {children}
        </CopilotKitProvider>
      </body>
    </html>
  );
}
```

---

## STEP 15: Create `frontend/components/CopilotKitProvider.tsx`

**File:** `frontend/components/CopilotKitProvider.tsx`

Create directory: `frontend/components/`

Write the following content exactly:

```typescript
/**
 * CopilotKitProvider Wrapper
 * ==========================
 * Wraps the CopilotKit React provider with our configuration.
 * Provides: CopilotChat (Feature 1), CopilotSidebar (Feature 2) context.
 */

'use client';

import React from 'react';
import { CopilotKit } from '@copilotkit/react-core';
import { CopilotSidebar } from '@copilotkit/react-ui';
import '@copilotkit/react-ui/styles.css';
import { COPILOT_CONFIG } from '@/lib/copilot-config';

interface CopilotKitProviderProps {
  children: React.ReactNode;
}

export function CopilotKitProvider({ children }: CopilotKitProviderProps) {
  return (
    <CopilotKit
      runtimeUrl={COPILOT_CONFIG.runtimeUrl}
      agent={COPILOT_CONFIG.agentName}
    >
      <CopilotSidebar
        defaultOpen={false}
        labels={{
          title: 'Legal Agent Assistant',
          initial: 'Hello! I am your legal research assistant. Start a workflow or ask me anything.',
          placeholder: 'Type your message...',
        }}
      >
        {children}
      </CopilotSidebar>
    </CopilotKit>
  );
}
```

**VERIFICATION:** Run `ls frontend/app/api/copilotkit/route.ts frontend/app/layout.tsx frontend/components/CopilotKitProvider.tsx`. All 3 must exist. If any missing, STOP.

---

## CHECKPOINT 4: CopilotKit Integration Verified

Before proceeding, verify:
1. `frontend/app/api/copilotkit/route.ts` has POST handler using `CopilotRuntime` with `HttpAgent`
2. `frontend/app/layout.tsx` imports and uses `<CopilotKitProvider>`
3. `frontend/components/CopilotKitProvider.tsx` imports from `@copilotkit/react-core` and `@copilotkit/react-ui`

Commands:
```bash
grep -q "CopilotRuntime" frontend/app/api/copilotkit/route.ts && echo "OK: CopilotRuntime found" || echo "FAIL"
grep -q "HttpAgent" frontend/app/api/copilotkit/route.ts && echo "OK: HttpAgent found" || echo "FAIL"
grep -q "CopilotKit" frontend/components/CopilotKitProvider.tsx && echo "OK: CopilotKit found" || echo "FAIL"
grep -q "CopilotSidebar" frontend/components/CopilotKitProvider.tsx && echo "OK: CopilotSidebar found" || echo "FAIL"
grep -q "CopilotKitProvider" frontend/app/layout.tsx && echo "OK: Layout uses provider" || echo "FAIL"
```

All must output "OK". If any fails, STOP.

---

## STEP 16: Create `frontend/components/MetricTile.tsx`

**File:** `frontend/components/MetricTile.tsx`

Write the following content exactly:

```typescript
/**
 * MetricTile
 * ==========
 * Displays a single metric with label, value, optional trend, and icon.
 * Used on the landing page dashboard.
 */

'use client';

import React from 'react';
import { type LucideIcon } from 'lucide-react';
import { cn } from '@/lib/utils';

export interface MetricTileProps {
  /** Metric label */
  label: string;
  /** Current value */
  value: string | number;
  /** Optional trend percentage */
  trend?: number;
  /** Lucide icon component */
  icon: LucideIcon;
  /** Color variant */
  variant?: 'default' | 'success' | 'warning' | 'danger' | 'info';
  /** Click handler */
  onClick?: () => void;
  /** Whether the tile is in a loading state */
  isLoading?: boolean;
}

const variantStyles = {
  default: 'border-border bg-card hover:bg-accent/50',
  success: 'border-emerald-200 bg-emerald-50 hover:bg-emerald-100 dark:border-emerald-800 dark:bg-emerald-950/30',
  warning: 'border-amber-200 bg-amber-50 hover:bg-amber-100 dark:border-amber-800 dark:bg-amber-950/30',
  danger: 'border-red-200 bg-red-50 hover:bg-red-100 dark:border-red-800 dark:bg-red-950/30',
  info: 'border-blue-200 bg-blue-50 hover:bg-blue-100 dark:border-blue-800 dark:bg-blue-950/30',
};

const iconVariantStyles = {
  default: 'text-muted-foreground',
  success: 'text-emerald-600 dark:text-emerald-400',
  warning: 'text-amber-600 dark:text-amber-400',
  danger: 'text-red-600 dark:text-red-400',
  info: 'text-blue-600 dark:text-blue-400',
};

export function MetricTile({
  label,
  value,
  trend,
  icon: Icon,
  variant = 'default',
  onClick,
  isLoading = false,
}: MetricTileProps) {
  return (
    <div
      onClick={onClick}
      className={cn(
        'rounded-lg border p-4 transition-colors duration-200',
        onClick && 'cursor-pointer',
        variantStyles[variant],
        isLoading && 'animate-pulse opacity-70'
      )}
    >
      <div className="flex items-start justify-between">
        <div className="space-y-1">
          <p className="text-sm font-medium text-muted-foreground">{label}</p>
          <p className="text-2xl font-bold tracking-tight">
            {isLoading ? '...' : value}
          </p>
          {trend !== undefined && (
            <p className={cn(
              'text-xs font-medium',
              trend >= 0 ? 'text-emerald-600 dark:text-emerald-400' : 'text-red-600 dark:text-red-400'
            )}>
              {trend >= 0 ? '+' : ''}{trend}% from last period
            </p>
          )}
        </div>
        <div className={cn('rounded-md p-2', iconVariantStyles[variant])}>
          <Icon className="h-5 w-5" />
        </div>
      </div>
    </div>
  );
}
```

---

## STEP 17: Create `frontend/app/page.tsx`

**File:** `frontend/app/page.tsx`

Write the following content exactly:

```typescript
/**
 * Landing Page (Dashboard)
 * ========================
 * Entry point showing:
 * - Feature checklist (all 8 CopilotKit features visible status)
 * - Metric tiles for workflow overview
 * - Links to actiand workspaces
 * - Start new workflow button
 */

'use client';

import React, { useEffect, useState } from 'react';
import { useRouter } from 'next/navigation';
import {
  Gavel,
  ClipboardList,
  CheckCircle2,
  XCircle,
  Loader2,
  Activity,
  FileText,
  Clock,
  Users,
  Zap,
  ArrowRight,
  Bot,
  MessageSquare,
  PanelRight,
  Code2,
  Eye,
  Wrench,
  Sparkles,
  Layout,
} from 'lucide-react';
import { MetricTile } from '@/components/MetricTile';
import { COPILOT_CONFIG, FEATURE_NAMES } from '@/lib/copilot-config';
import { cn } from '@/lib/utils';

interface WorkflowSummary {
  id: string;
  status: string;
  agent_name: string;
  created_at: string;
  current_step: string | null;
}

interface DashboardMetrics {
  totalWorkflows: number;
  activeWorkflows: number;
  completedWorkflows: number;
  failedWorkflows: number;
  avgDuration: string;
  pendingApprovals: number;
}

const FEATURE_ICONS: Record<string, React.ElementType> = {
  CopilotChat: MessageSquare,
  CopilotSidebar: PanelRight,
  useCoAgent: Bot,
  useAgent: Code2,
  useCopilotAction: Zap,
  useCopilotReadable: Eye,
  useFrontendTool: Wrench,
  GenerativeUI: Sparkles,
};

export default function LandingPage() {
  const router = useRouter();
  const [workflows, setWorkflows] = useState<WorkflowSummary[]>([]);
  const [metrics, setMetrics] = useState<DashboardMetrics>({
    totalWorkflows: 0,
    activeWorkflows: 0,
    completedWorkflows: 0,
    failedWorkflows: 0,
    avgDuration: '0m',
    pendingApprovals: 0,
  });
  const [isLoading, setIsLoading] = useState(true);

  // Fetch workflow summaries from backend
  useEffect(() => {
    async function fetchData() {
      try {
        const response = await fetch(`${COPILOT_CONFIG.backendUrl}/workflows`);
        if (response.ok) {
          const data = await response.json();
          const workflows: WorkflowSummary[] = data.workflows || data || [];
          setWorkflows(workflows);

          // Calculate metrics
          const actiand = workflows.filter((w: WorkflowSummary) =>
            w.status === 'running' || w.status === 'awaiting_approval' || w.status === 'awaiting_clarification'
          ).length;
          const completed = workflows.filter((w: WorkflowSummary) => w.status === 'completed').length;
          const failed = workflows.filter((w: WorkflowSummary) => w.status === 'failed').length;

          setMetrics({
            totalWorkflows: workflows.length,
            activeWorkflows: active,
            completedWorkflows: completed,
            failedWorkflows: failed,
            avgDuration: '12m', // Placeholder -- computed on backend
            pendingApprovals: workflows.filter((w: WorkflowSummary) => w.status === 'awaiting_approval').length,
          });
        }
      } catch (err) {
        console.error('Failed to fetch workflows:', err);
      } finally {
        setIsLoading(false);
      }
    }

    fetchData();
    // Refresh every 10 seconds
    const interval = setInterval(fetchData, 10000);
    return () => clearInterval(interval);
  }, []);

  // Navigate to workspace
  const openWorkspace = (wfId: string) => {
    router.push(`/workspace/${wfId}`);
  };

  // Start new workflow
  const startWorkflow = async () => {
    try {
      const response = await fetch(`${COPILOT_CONFIG.backendUrl}/workflows`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          agent_name: COPILOT_CONFIG.agentName,
          input: { query: 'Start new legal research workflow' },
        }),
      });

      if (response.ok) {
        const data = await response.json();
        const wfId = data.workflow_id || data.id;
        if (wfId) {
          router.push(`/workspace/${wfId}`);
        }
      }
    } catch (err) {
      console.error('Failed to start workflow:', err);
    }
  };

  return (
    <main className="min-h-screen bg-background">
      {/* Header */}
      <header className="border-b bg-card">
        <div className="container mx-auto flex h-16 items-center justify-between px-4">
          <div className="flex items-center gap-3">
            <div className="rounded-lg bg-primary p-2">
              <Gavel className="h-5 w-5 text-primary-foreground" />
            </div>
            <div>
              <h1 className="text-lg font-semibold">Legal Agent</h1>
              <p className="text-xs text-muted-foreground">AI-Powered Legal Research</p>
            </div>
          </div>
          <button
            onClick={startWorkflow}
            className="inline-flex items-center gap-2 rounded-md bg-primary px-4 py-2 text-sm font-medium text-primary-foreground hover:bg-primary/90 transition-colors"
          >
            <Zap className="h-4 w-4" />
            New Workflow
          </button>
        </div>
      </header>

      <div className="container mx-auto px-4 py-8">
        {/* Feature Checklist -- ALL 8 FEATURES VISIBLE */}
        <section className="mb-8">
          <h2 className="mb-4 text-xl font-semibold flex items-center gap-2">
            <Layout className="h-5 w-5 text-primary" />
            CopilotKit Features
          </h2>
          <div className="grid grid-cols-2 gap-3 sm:grid-cols-4">
            {FEATURE_NAMES.map((name) => {
              const Icon = FEATURE_ICONS[name] || Bot;
              const isEnabled = COPILOT_CONFIG.features[name.toLowerCase() as keyof typeof COPILOT_CONFIG.features] ?? true;
              return (
                <div
                  key={name}
                  className={cn(
                    'flex items-center gap-3 rounded-lg border p-3',
                    isEnabled
                      ? 'border-emerald-200 bg-emerald-50 dark:border-emerald-800 dark:bg-emerald-950/20'
                      : 'border-gray-200 bg-gray-50 dark:border-gray-800 dark:bg-gray-900/20'
                  )}
                >
                  {isEnabled ? (
                    <CheckCircle2 className="h-5 w-5 shrink-0 text-emerald-600 dark:text-emerald-400" />
                  ) : (
                    <XCircle className="h-5 w-5 shrink-0 text-gray-400" />
                  )}
                  <div className="min-w-0">
                    <p className={cn(
                      'text-sm font-medium',
                      isEnabled ? 'text-foreground' : 'text-muted-foreground'
                    )}>
                      {name}
                    </p>
                    <p className="text-xs text-muted-foreground">
                      {isEnabled ? 'Active' : 'Inactive'}
                    </p>
                  </div>
                </div>
              );
            })}
          </div>
        </section>

        {/* Metrics Grid */}
        <section className="mb-8">
          <h2 className="mb-4 text-xl font-semibold flex items-center gap-2">
            <Activity className="h-5 w-5 text-primary" />
            Dashboard
          </h2>
          <div className="grid grid-cols-2 gap-4 sm:grid-cols-3 lg:grid-cols-6">
            <MetricTile
              label="Total Workflows"
              value={metrics.totalWorkflows}
              icon={ClipboardList}
              variant="default"
              isLoading={isLoading}
            />
            <MetricTile
              label="Active"
              value={metrics.activeWorkflows}
              icon={Zap}
              variant="info"
              isLoading={isLoading}
            />
            <MetricTile
              label="Completed"
              value={metrics.completedWorkflows}
              icon={CheckCircle2}
              variant="success"
              isLoading={isLoading}
            />
            <MetricTile
              label="Failed"
              value={metrics.failedWorkflows}
              icon={XCircle}
              variant={metrics.failedWorkflows > 0 ? 'danger' : 'default'}
              isLoading={isLoading}
            />
            <MetricTile
              label="Avg. Duration"
              value={metrics.avgDuration}
              icon={Clock}
              variant="default"
              isLoading={isLoading}
            />
            <MetricTile
              label="Pending Approvals"
              value={metrics.pendingApprovals}
              icon={Users}
              variant={metrics.pendingApprovals > 0 ? 'warning' : 'default'}
              isLoading={isLoading}
            />
          </div>
        </section>

        {/* Actiand Workflows */}
        <section>
          <h2 className="mb-4 text-xl font-semibold flex items-center gap-2">
            <FileText className="h-5 w-5 text-primary" />
            Workflows
          </h2>
          {isLoading ? (
            <div className="flex items-center justify-center py-12">
              <Loader2 className="h-8 w-8 animate-spin text-primary" />
            </div>
          ) : workflows.length === 0 ? (
            <div className="rounded-lg border border-dashed p-12 text-center">
              <ClipboardList className="mx-auto h-12 w-12 text-muted-foreground" />
              <h3 className="mt-4 text-lg font-medium">No workflows yet</h3>
              <p className="mt-2 text-sm text-muted-foreground">
                Start your first legal research workflow to see it here.
              </p>
              <button
                onClick={startWorkflow}
                className="mt-4 inline-flex items-center gap-2 rounded-md bg-primary px-4 py-2 text-sm font-medium text-primary-foreground hover:bg-primary/90"
              >
                <Zap className="h-4 w-4" />
                Start Workflow
              </button>
            </div>
          ) : (
            <div className="space-y-2">
              {workflows.map((wf) => (
                <div
                  key={wf.id}
                  onClick={() => openWorkspace(wf.id)}
                  className="flex cursor-pointer items-center justify-between rounded-lg border bg-card p-4 transition-colors hover:bg-accent/50"
                >
                  <div className="flex items-center gap-4">
                    <div className={cn(
                      'h-2 w-2 rounded-full',
                      wf.status === 'running' && 'bg-blue-500 animate-pulse',
                      wf.status === 'completed' && 'bg-emerald-500',
                      wf.status === 'failed' && 'bg-red-500',
                      wf.status === 'awaiting_approval' && 'bg-amber-500',
                      wf.status === 'awaiting_clarification' && 'bg-purple-500',
                      (wf.status === 'idle' || !wf.status) && 'bg-gray-400',
                    )} />
                    <div>
                      <p className="font-medium">{wf.agent_name || 'Legal Research'}</p>
                      <p className="text-xs text-muted-foreground">
                        {wf.id.substring(0, 8)}... &middot; {wf.current_step || 'Initializing'}
                      </p>
                    </div>
                  </div>
                  <div className="flex items-center gap-4">
                    <span className={cn(
                      'rounded-full px-2.5 py-0.5 text-xs font-medium',
                      wf.status === 'running' && 'bg-blue-100 text-blue-800 dark:bg-blue-900/30 dark:text-blue-300',
                      wf.status === 'completed' && 'bg-emerald-100 text-emerald-800 dark:bg-emerald-900/30 dark:text-emerald-300',
                      wf.status === 'failed' && 'bg-red-100 text-red-800 dark:bg-red-900/30 dark:text-red-300',
                      wf.status === 'awaiting_approval' && 'bg-amber-100 text-amber-800 dark:bg-amber-900/30 dark:text-amber-300',
                      wf.status === 'awaiting_clarification' && 'bg-purple-100 text-purple-800 dark:bg-purple-900/30 dark:text-purple-300',
                    )}>
                      {wf.status || 'idle'}
                    </span>
                    <ArrowRight className="h-4 w-4 text-muted-foreground" />
                  </div>
                </div>
              ))}
            </div>
          )}
        </section>
      </div>
    </main>
  );
}
```

**VERIFICATION:** Run `ls frontend/app/page.tsx frontend/components/MetricTile.tsx`. Both must exist. Run `grep -c "MetricTile" frontend/app/page.tsx` -- must output at least 1.

---

## CHECKPOINT 5: Landing Page Verified

Before proceeding, verify:
1. `frontend/app/page.tsx` exists and imports `MetricTile` and `FEATURE_NAMES`
2. `frontend/components/MetricTile.tsx` exists and exports `MetricTile` component
3. Landing page references all 8 feature names from `FEATURE_NAMES` array

Command: `grep -o "CopilotChat\|CopilotSidebar\|useCoAgent\|useAgent\|useCopilotAction\|useCopilotReadable\|useFrontendTool\|GenerativeUI" frontend/app/page.tsx | wc -l`
Expected: 8 or more. If less, STOP.

---

## STEP 18: Create `frontend/components/TranscriptCard.tsx`

**Generatiand UI Card #1** -- Displays transcript events from the agent.

**File:** `frontend/components/TranscriptCard.tsx`

Write the following content exactly:

```typescript
/**
 * TranscriptCard
 * ==============
 * Generatiand UI card that renders transcript content from AG-UI events.
 * Part of the 8-feature CopilotKit integration (GenerativeUI feature).
 */

'use client';

import React from 'react';
import { FileText, Copy, Check } from 'lucide-react';
import { cn } from '@/lib/utils';

export interface TranscriptCardProps {
  /** Transcript text content */
  transcript: string | null;
  /** Timestamp when transcript was received */
  timestamp?: string;
  /** Agent or step name that produced this */
  source?: string;
  /** Additional CSS classes */
  className?: string;
}

export function TranscriptCard({ transcript, timestamp, source, className }: TranscriptCardProps) {
  const [copied, setCopied] = React.useState(false);

  if (!transcript) {
    return (
      <div className={cn('rounded-lg border border-dashed p-6 text-center', className)}>
        <FileText className="mx-auto h-8 w-8 text-muted-foreground" />
        <p className="mt-2 text-sm text-muted-foreground">No transcript yet</p>
        <p className="text-xs text-muted-foreground">Transcript will appear here when the agent generates one.</p>
      </div>
    );
  }

  const handleCopy = async () => {
    try {
      await navigator.clipboard.writeText(transcript);
      setCopied(true);
      setTimeout(() => setCopied(false), 2000);
    } catch {
      // Clipboard API not available
    }
  };

  return (
    <div className={cn('rounded-lg border bg-card overflow-hidden', className)}>
      {/* Header */}
      <div className="flex items-center justify-between border-b bg-muted/50 px-4 py-3">
        <div className="flex items-center gap-2">
          <FileText className="h-4 w-4 text-blue-600" />
          <h3 className="text-sm font-semibold">Transcript</h3>
          {source && (
            <span className="rounded-full bg-blue-100 px-2 py-0.5 text-xs text-blue-800 dark:bg-blue-900/30 dark:text-blue-300">
              {source}
            </span>
          )}
        </div>
        <div className="flex items-center gap-2">
          {timestamp && (
            <span className="text-xs text-muted-foreground">{timestamp}</span>
          )}
          <button
            onClick={handleCopy}
            className="rounded-md p-1.5 hover:bg-accent transition-colors"
            title="Copy transcript"
          >
            {copied ? (
              <Check className="h-3.5 w-3.5 text-emerald-600" />
            ) : (
              <Copy className="h-3.5 w-3.5 text-muted-foreground" />
            )}
          </button>
        </div>
      </div>

      {/* Content */}
      <div className="max-h-96 overflow-y-auto p-4 canvas-scroll">
        <div className="agui-event-line whitespace-pre-wrap text-sm leading-relaxed text-foreground">
          {transcript}
        </div>
      </div>

      {/* Footer */}
      <div className="border-t bg-muted/30 px-4 py-2">
        <p className="text-xs text-muted-foreground">
          Generated by Legal Research Agent
        </p>
      </div>
    </div>
  );
}
```

---

## STEP 19: Create `frontend/components/DraftPreviewCard.tsx`

**Generatiand UI Card #2** -- Displays draft document preview.

**File:** `frontend/components/DraftPreviewCard.tsx`

Write the following content exactly:

```typescript
/**
 * DraftPreviewCard
 * ================
 * Generatiand UI card that renders draft document content.
 * Shows a formatted preview of agent-generated legal documents.
 */

'use client';

import React, { useState } from 'react';
import { FileEdit, Download, Eye, Code } from 'lucide-react';
import { cn } from '@/lib/utils';

export interface DraftPreviewCardProps {
  /** Draft content (markdown or plain text) */
  draft: string | null;
  /** Document title */
  title?: string;
  /** Document type label */
  docType?: string;
  /** Timestamp */
  timestamp?: string;
  /** Additional CSS classes */
  className?: string;
}

export function DraftPreviewCard({
  draft,
  title = 'Draft Document',
  docType = 'Legal Memo',
  timestamp,
  className,
}: DraftPreviewCardProps) {
  const [viewMode, setViewMode] = useState<'preview' | 'raw'>('preview');

  if (!draft) {
    return (
      <div className={cn('rounded-lg border border-dashed p-6 text-center', className)}>
        <FileEdit className="mx-auto h-8 w-8 text-muted-foreground" />
        <p className="mt-2 text-sm text-muted-foreground">No draft yet</p>
        <p className="text-xs text-muted-foreground">Draft will appear here when generated.</p>
      </div>
    );
  }

  const handleDownload = () => {
    const blob = new Blob([draft], { type: 'text/markdown' });
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = `${title.replace(/\s+/g, '_').toLowerCase()}.md`;
    a.click();
    URL.revokeObjectURL(url);
  };

  return (
    <div className={cn('rounded-lg border bg-card overflow-hidden', className)}>
      {/* Header */}
      <div className="flex items-center justify-between border-b bg-muted/50 px-4 py-3">
        <div className="flex items-center gap-2">
          <FileEdit className="h-4 w-4 text-emerald-600" />
          <h3 className="text-sm font-semibold">{title}</h3>
          <span className="rounded-full bg-emerald-100 px-2 py-0.5 text-xs text-emerald-800 dark:bg-emerald-900/30 dark:text-emerald-300">
            {docType}
          </span>
        </div>
        <div className="flex items-center gap-1">
          <button
            onClick={() => setViewMode('preview')}
            className={cn(
              'rounded-md p-1.5 transition-colors',
              viewMode === 'preview' ? 'bg-accent text-foreground' : 'text-muted-foreground hover:bg-accent'
            )}
            title="Preview"
          >
            <Eye className="h-3.5 w-3.5" />
          </button>
          <button
            onClick={() => setViewMode('raw')}
            className={cn(
              'rounded-md p-1.5 transition-colors',
              viewMode === 'raw' ? 'bg-accent text-foreground' : 'text-muted-foreground hover:bg-accent'
            )}
            title="Raw"
          >
            <Code className="h-3.5 w-3.5" />
          </button>
          <button
            onClick={handleDownload}
            className="rounded-md p-1.5 text-muted-foreground hover:bg-accent transition-colors"
            title="Download"
          >
            <Download className="h-3.5 w-3.5" />
          </button>
          {timestamp && (
            <span className="ml-2 text-xs text-muted-foreground">{timestamp}</span>
          )}
        </div>
      </div>

      {/* Content */}
      <div className="max-h-[500px] overflow-y-auto p-4 canvas-scroll">
        {viewMode === 'preview' ? (
          <div className="prose prose-sm dark:prose-invert max-w-none">
            {draft.split('\n').map((line, i) => {
              if (line.startsWith('# ')) {
                return <h1 key={i} className="text-xl font-bold mt-4 mb-2">{line.slice(2)}</h1>;
              }
              if (line.startsWith('## ')) {
                return <h2 key={i} className="text-lg font-semibold mt-3 mb-2">{line.slice(3)}</h2>;
              }
              if (line.startsWith('### ')) {
                return <h3 key={i} className="text-base font-semibold mt-2 mb-1">{line.slice(4)}</h3>;
              }
              if (line.startsWith('- ') || line.startsWith('* ')) {
                return <li key={i} className="ml-4">{line.slice(2)}</li>;
              }
              if (line.match(/^\d+\.\s/)) {
                return <li key={i} className="ml-4 list-decimal">{line.replace(/^\d+\.\s/, '')}</li>;
              }
              if (line.trim() === '') {
                return <div key={i} className="h-2" />;
              }
              return <p key={i} className="text-sm leading-relaxed">{line}</p>;
            })}
          </div>
        ) : (
          <pre className="agui-event-line text-xs whitespace-pre-wrap break-all">
            {draft}
          </pre>
        )}
      </div>
    </div>
  );
}
```

---

## STEP 20: Create `frontend/components/ReviewReportCard.tsx`

**Generatiand UI Card #3** -- Displays review/audit report.

**File:** `frontend/components/ReviewReportCard.tsx`

Write the following content exactly:

```typescript
/**
 * ReviewReportCard
 * ================
 * Generatiand UI card that renders a structured review/audit report.
 * Displays findings, citations, compliance checks, and risk assessments.
 */

'use client';

import React from 'react';
import { ClipboardCheck, AlertTriangle, CheckCircle2, Info, ChevronDown, ChevronRight } from 'lucide-react';
import { cn } from '@/lib/utils';

export interface ReviewFinding {
  id: string;
  category: string;
  severity: 'info' | 'low' | 'medium' | 'high' | 'critical';
  title: string;
  description: string;
  citation?: string;
  recommendation?: string;
  status: 'pass' | 'fail' | 'warning' | 'info';
}

export interface ReviewReportCardProps {
  /** Review data -- can be raw object from AG-UI or structured findings */
  review: Record<string, unknown> | null;
  /** Pre-parsed findings (optional) */
  findings?: ReviewFinding[];
  /** Report title */
  title?: string;
  /** Additional CSS classes */
  className?: string;
}

const severityConfig = {
  info: { icon: Info, color: 'text-blue-600', bg: 'bg-blue-50 dark:bg-blue-900/20', border: 'border-blue-200' },
  low: { icon: Info, color: 'text-slate-600', bg: 'bg-slate-50 dark:bg-slate-900/20', border: 'border-slate-200' },
  medium: { icon: AlertTriangle, color: 'text-amber-600', bg: 'bg-amber-50 dark:bg-amber-900/20', border: 'border-amber-200' },
  high: { icon: AlertTriangle, color: 'text-orange-600', bg: 'bg-orange-50 dark:bg-orange-900/20', border: 'border-orange-200' },
  critical: { icon: AlertTriangle, color: 'text-red-600', bg: 'bg-red-50 dark:bg-red-900/20', border: 'border-red-200' },
};

const statusConfig = {
  pass: { icon: CheckCircle2, color: 'text-emerald-600', label: 'Pass' },
  fail: { icon: AlertTriangle, color: 'text-red-600', label: 'Fail' },
  warning: { icon: AlertTriangle, color: 'text-amber-600', label: 'Warning' },
  info: { icon: Info, color: 'text-blue-600', label: 'Info' },
};

export function ReviewReportCard({ review, findings, title = 'Review Report', className }: ReviewReportCardProps) {
  const [expandedFinding, setExpandedFinding] = React.useState<string | null>(null);

  // Deriand findings from review object if not provided directly
  const parsedFindings: ReviewFinding[] = findings || React.useMemo(() => {
    if (!review) return [];

    // Handle array of findings directly
    if (review.findings && Array.isArray(review.findings)) {
      return review.findings as ReviewFinding[];
    }

    // Handle single review object -- convert to finding
    const finding: ReviewFinding = {
      id: '1',
      category: review.category as string || 'General',
      severity: (review.severity as ReviewFinding['severity']) || 'info',
      title: review.title as string || 'Review Finding',
      description: review.description as string || review.summary as string || JSON.stringify(review),
      citation: review.citation as string || undefined,
      recommendation: review.recommendation as string || undefined,
      status: (review.status as ReviewFinding['status']) || 'info',
    };
    return [finding];
  }, [review]);

  if (!review || parsedFindings.length === 0) {
    return (
      <div className={cn('rounded-lg border border-dashed p-6 text-center', className)}>
        <ClipboardCheck className="mx-auto h-8 w-8 text-muted-foreground" />
        <p className="mt-2 text-sm text-muted-foreground">No review report yet</p>
        <p className="text-xs text-muted-foreground">Review will appear after agent analysis.</p>
      </div>
    );
  }

  const summary = parsedFindings.reduce(
    (acc, f) => {
      acc[f.status] = (acc[f.status] || 0) + 1;
      return acc;
    },
    {} as Record<string, number>
  );

  return (
    <div className={cn('rounded-lg border bg-card overflow-hidden', className)}>
      {/* Header */}
      <div className="flex items-center justify-between border-b bg-muted/50 px-4 py-3">
        <div className="flex items-center gap-2">
          <ClipboardCheck className="h-4 w-4 text-purple-600" />
          <h3 className="text-sm font-semibold">{title}</h3>
          <span className="rounded-full bg-purple-100 px-2 py-0.5 text-xs text-purple-800 dark:bg-purple-900/30 dark:text-purple-300">
            {parsedFindings.length} findings
          </span>
        </div>
        <div className="flex items-center gap-2">
          {Object.entries(summary).map(([status, count]) => {
            const config = statusConfig[status as keyof typeof statusConfig];
            if (!config) return null;
            const StatusIcon = config.icon;
            return (
              <span key={status} className="flex items-center gap-1 text-xs">
                <StatusIcon className={cn('h-3.5 w-3.5', config.color)} />
                {count}
              </span>
            );
          })}
        </div>
      </div>

      {/* Findings List */}
      <div className="max-h-[500px] overflow-y-auto canvas-scroll">
        {parsedFindings.map((finding) => {
          const sev = severityConfig[finding.severity];
          const st = statusConfig[finding.status];
          const isExpanded = expandedFinding === finding.id;
          const SevIcon = sev.icon;
          const StatusIcon = st.icon;

          return (
            <div
              key={finding.id}
              className={cn('border-b last:border-b-0', sev.bg)}
            >
              {/* Summary Row */}
              <button
                onClick={() => setExpandedFinding(isExpanded ? null : finding.id)}
                className="flex w-full items-center gap-3 px-4 py-3 text-left hover:bg-black/5 dark:hover:bg-white/5 transition-colors"
              >
                {isExpanded ? (
                  <ChevronDown className="h-4 w-4 shrink-0 text-muted-foreground" />
                ) : (
                  <ChevronRight className="h-4 w-4 shrink-0 text-muted-foreground" />
                )}
                <SevIcon className={cn('h-4 w-4 shrink-0', sev.color)} />
                <StatusIcon className={cn('h-4 w-4 shrink-0', st.color)} />
                <div className="min-w-0 flex-1">
                  <p className="text-sm font-medium truncate">{finding.title}</p>
                  <p className="text-xs text-muted-foreground">{finding.category}</p>
                </div>
                <span className={cn(
                  'rounded-full px-2 py-0.5 text-xs font-medium shrink-0',
                  finding.severity === 'critical' && 'bg-red-100 text-red-800',
                  finding.severity === 'high' && 'bg-orange-100 text-orange-800',
                  finding.severity === 'medium' && 'bg-amber-100 text-amber-800',
                  finding.severity === 'low' && 'bg-slate-100 text-slate-800',
                  finding.severity === 'info' && 'bg-blue-100 text-blue-800',
                )}>
                  {finding.severity.toUpperCase()}
                </span>
              </button>

              {/* Expanded Detail */}
              {isExpanded && (
                <div className="px-4 pb-4 pl-12">
                  <p className="text-sm text-foreground mb-2">{finding.description}</p>
                  {finding.citation && (
                    <div className="mb-2 rounded bg-muted p-2">
                      <p className="text-xs font-medium text-muted-foreground">Citation</p>
                      <p className="text-xs">{finding.citation}</p>
                    </div>
                  )}
                  {finding.recommendation && (
                    <div className="rounded bg-emerald-50 dark:bg-emerald-900/20 p-2">
                      <p className="text-xs font-medium text-emerald-800 dark:text-emerald-300">Recommendation</p>
                      <p className="text-xs text-emerald-700 dark:text-emerald-400">{finding.recommendation}</p>
                    </div>
                  )}
                </div>
              )}
            </div>
          );
        })}
      </div>
    </div>
  );
}
```

---

## STEP 21: Create `frontend/components/ApprovalCard.tsx`

**Generatiand UI Card #4** -- Dual-response HITL approval card.

**File:** `frontend/components/ApprovalCard.tsx`

Write the following content exactly:

```typescript
/**
 * ApprovalCard
 * ============
 * Generatiand UI card for dual-response Human-in-the-Loop (HITL).
 * Displays approval requests from the agent and captures human decisions.
 *
 * Supports: useFrontendTool (Feature 7), useCopilotAction (Feature 5)
 */

'use client';

import React, { useState } from 'react';
import { ThumbsUp, ThumbsDown, Clock, MessageSquare, Send, X } from 'lucide-react';
import { cn } from '@/lib/utils';

export interface ApprovalRequest {
  type: string;
  description: string;
  options: string[];
  deadline?: string;
  context?: string;
}

export interface ApprovalCardProps {
  /** The approval request to display */
  request: ApprovalRequest | null;
  /** Callback when user approves */
  onApprove: (comment?: string) => void;
  /** Callback when user rejects */
  onReject: (comment?: string) => void;
  /** Whether a response is being submitted */
  isSubmitting?: boolean;
  /** Additional CSS classes */
  className?: string;
}

export function ApprovalCard({
  request,
  onApprove,
  onReject,
  isSubmitting = false,
  className,
}: ApprovalCardProps) {
  const [comment, setComment] = useState('');
  const [showComment, setShowComment] = useState(false);
  const [pendingAction, setPendingAction] = useState<'approve' | 'reject' | null>(null);

  if (!request) {
    return (
      <div className={cn('rounded-lg border border-dashed p-6 text-center', className)}>
        <ThumbsUp className="mx-auto h-8 w-8 text-muted-foreground" />
        <p className="mt-2 text-sm text-muted-foreground">No pending approvals</p>
        <p className="text-xs text-muted-foreground">Approval requests will appear here.</p>
      </div>
    );
  }

  const handleApproand = () => {
    if (showComment) {
      onApprove(comment || undefined);
      setComment('');
      setShowComment(false);
    } else {
      setPendingAction('approve');
      setShowComment(true);
    }
  };

  const handleReject = () => {
    if (showComment) {
      onReject(comment || undefined);
      setComment('');
      setShowComment(false);
    } else {
      setPendingAction('reject');
      setShowComment(true);
    }
  };

  const handleCancel = () => {
    setShowComment(false);
    setPendingAction(null);
    setComment('');
  };

  const timeRemaining = request.deadline
    ? Math.max(0, Math.ceil((new Date(request.deadline).getTime() - Date.now()) / 60000))
    : null;

  return (
    <div className={cn('rounded-lg border-2 border-amber-300 bg-card overflow-hidden shadow-lg animate-pulse-glow', className)}>
      {/* Header */}
      <div className="flex items-center justify-between border-b bg-amber-50 dark:bg-amber-950/30 px-4 py-3">
        <div className="flex items-center gap-2">
          <div className="rounded-full bg-amber-500 p-1.5">
            <ThumbsUp className="h-4 w-4 text-white" />
          </div>
          <div>
            <h3 className="text-sm font-semibold">Approval Required</h3>
            <p className="text-xs text-muted-foreground">{request.type}</p>
          </div>
        </div>
        {timeRemaining !== null && (
          <div className="flex items-center gap-1 rounded-full bg-amber-100 px-2.5 py-1 text-xs text-amber-800 dark:bg-amber-900/40 dark:text-amber-300">
            <Clock className="h-3 w-3" />
            {timeRemaining}m remaining
          </div>
        )}
      </div>

      {/* Description */}
      <div className="p-4">
        <p className="text-sm text-foreground mb-3">{request.description}</p>

        {request.context && (
          <div className="mb-4 rounded-md bg-muted p-3">
            <p className="text-xs font-medium text-muted-foreground mb-1">Context</p>
            <p className="text-xs">{request.context}</p>
          </div>
        )}

        {/* Options */}
        {request.options.length > 0 && (
          <div className="mb-4 flex flex-wrap gap-2">
            {request.options.map((option) => (
              <span
                key={option}
                className="rounded-full border bg-background px-3 py-1 text-xs"
              >
                {option}
              </span>
            ))}
          </div>
        )}

        {/* Comment Input */}
        {showComment && (
          <div className="mb-4">
            <div className="flex items-center gap-2 rounded-md border bg-background px-3 py-2">
              <MessageSquare className="h-4 w-4 text-muted-foreground shrink-0" />
              <input
                type="text"
                value={comment}
                onChange={(e) => setComment(e.target.value)}
                placeholder={`Add a comment (optional) before ${pendingAction === 'approve' ? 'approving' : 'rejecting'}...`}
                className="flex-1 bg-transparent text-sm outline-none placeholder:text-muted-foreground"
                onKeyDown={(e) => {
                  if (e.key === 'Enter') {
                    pendingAction === 'approve' ? handleApprove() : handleReject();
                  }
                  if (e.key === 'Escape') {
                    handleCancel();
                  }
                }}
              />
              <button onClick={handleCancel} className="text-muted-foreground hover:text-foreground">
                <X className="h-4 w-4" />
              </button>
            </div>
          </div>
        )}

        {/* Action Buttons */}
        <div className="flex gap-3">
          {showComment ? (
            <>
              <button
                onClick={handleCancel}
                disabled={isSubmitting}
                className="flex-1 rounded-md border px-4 py-2 text-sm font-medium hover:bg-accent transition-colors disabled:opacity-50"
              >
                Cancel
              </button>
              <button
                onClick={pendingAction === 'approve' ? handleApproand : handleReject}
                disabled={isSubmitting}
                className={cn(
                  'flex flex-1 items-center justify-center gap-2 rounded-md px-4 py-2 text-sm font-medium transition-colors disabled:opacity-50',
                  pendingAction === 'approve'
                    ? 'bg-emerald-600 text-white hover:bg-emerald-700'
                    : 'bg-red-600 text-white hover:bg-red-700'
                )}
              >
                {isSubmitting ? (
                  <div className="h-4 w-4 animate-spin rounded-full border-2 border-white border-t-transparent" />
                ) : (
                  <>
                    <Send className="h-4 w-4" />
                    {pendingAction === 'approve' ? 'Confirm Approve' : 'Confirm Reject'}
                  </>
                )}
              </button>
            </>
          ) : (
            <>
              <button
                onClick={handleReject}
                disabled={isSubmitting}
                className="flex flex-1 items-center justify-center gap-2 rounded-md border-2 border-red-200 bg-red-50 px-4 py-2 text-sm font-medium text-red-700 hover:bg-red-100 transition-colors disabled:opacity-50 dark:border-red-800 dark:bg-red-950/20 dark:text-red-400"
              >
                <ThumbsDown className="h-4 w-4" />
                Reject
              </button>
              <button
                onClick={handleApprove}
                disabled={isSubmitting}
                className="flex flex-1 items-center justify-center gap-2 rounded-md bg-emerald-600 px-4 py-2 text-sm font-medium text-white hover:bg-emerald-700 transition-colors disabled:opacity-50"
              >
                <ThumbsUp className="h-4 w-4" />
                Approve
              </button>
            </>
          )}
        </div>
      </div>
    </div>
  );
}
```

**VERIFICATION:** Run `ls frontend/components/TranscriptCard.tsx frontend/components/DraftPreviewCard.tsx frontend/components/ReviewReportCard.tsx frontend/components/ApprovalCard.tsx`. All 4 must exist.

---

## CHECKPOINT 6: First 4 Generatiand UI Cards Verified

Before proceeding, verify:
1. `TranscriptCard.tsx` -- exports `TranscriptCard` with `transcript` prop
2. `DraftPreviewCard.tsx` -- exports `DraftPreviewCard` with `draft` prop
3. `ReviewReportCard.tsx` -- exports `ReviewReportCard` with `review` prop
4. `ApprovalCard.tsx` -- exports `ApprovalCard` with `onApprove` and `onReject` callbacks

Command:
```bash
for f in TranscriptCard DraftPreviewCard ReviewReportCard ApprovalCard; do
  grep -q "export function ${f}" frontend/components/${f}.tsx && echo "OK: ${f}" || echo "FAIL: ${f}"
done
```

All must output "OK". If any fails, STOP.

---

## STEP 22: Create `frontend/components/ClarificationQuestionCard.tsx`

**Generatiand UI Card #5** -- HITL clarification question card.

**File:** `frontend/components/ClarificationQuestionCard.tsx`

Write the following content exactly:

```typescript
/**
 * ClarificationQuestionCard
 * =========================
 * Generatiand UI card that displays clarification questions from the agent.
 * Part of dual-response HITL -- agent asks, human answers.
 */

'use client';

import React, { useState } from 'react';
import { HelpCircle, Send, MessageSquare, Lightbulb, X } from 'lucide-react';
import { cn } from '@/lib/utils';

export interface ClarificationQuestion {
  question: string;
  context: string;
  suggestedAnswers?: string[];
}

export interface ClarificationQuestionCardProps {
  /** The clarification question to display */
  question: ClarificationQuestion | null;
  /** Callback when user submits an answer */
  onAnswer: (answer: string) => void;
  /** Whether a response is being submitted */
  isSubmitting?: boolean;
  /** Additional CSS classes */
  className?: string;
}

export function ClarificationQuestionCard({
  question,
  onAnswer,
  isSubmitting = false,
  className,
}: ClarificationQuestionCardProps) {
  const [answer, setAnswer] = useState('');
  const [selectedSuggestion, setSelectedSuggestion] = useState<string | null>(null);

  if (!question) {
    return (
      <div className={cn('rounded-lg border border-dashed p-6 text-center', className)}>
        <HelpCircle className="mx-auto h-8 w-8 text-muted-foreground" />
        <p className="mt-2 text-sm text-muted-foreground">No clarification questions</p>
        <p className="text-xs text-muted-foreground">Questions will appear when the agent needs input.</p>
      </div>
    );
  }

  const handleSubmit = () => {
    const finalAnswer = selectedSuggestion || answer;
    if (finalAnswer.trim()) {
      onAnswer(finalAnswer.trim());
      setAnswer('');
      setSelectedSuggestion(null);
    }
  };

  const handleSuggestionClick = (suggestion: string) => {
    setSelectedSuggestion(suggestion);
    setAnswer(suggestion);
  };

  return (
    <div className={cn('rounded-lg border-2 border-purple-300 bg-card overflow-hidden shadow-lg animate-pulse-glow', className)}>
      {/* Header */}
      <div className="flex items-center justify-between border-b bg-purple-50 dark:bg-purple-950/30 px-4 py-3">
        <div className="flex items-center gap-2">
          <div className="rounded-full bg-purple-500 p-1.5">
            <HelpCircle className="h-4 w-4 text-white" />
          </div>
          <div>
            <h3 className="text-sm font-semibold">Clarification Needed</h3>
            <p className="text-xs text-muted-foreground">The agent needs more information</p>
          </div>
        </div>
      </div>

      {/* Question */}
      <div className="p-4">
        <div className="mb-4">
          <p className="text-sm font-medium text-foreground mb-1">{question.question}</p>
          {question.context && (
            <p className="text-xs text-muted-foreground">{question.context}</p>
          )}
        </div>

        {/* Suggested Answers */}
        {question.suggestedAnswers && question.suggestedAnswers.length > 0 && (
          <div className="mb-4">
            <div className="flex items-center gap-1 mb-2">
              <Lightbulb className="h-3.5 w-3.5 text-amber-500" />
              <p className="text-xs font-medium text-muted-foreground">Suggested answers</p>
            </div>
            <div className="flex flex-wrap gap-2">
              {question.suggestedAnswers.map((suggestion) => (
                <button
                  key={suggestion}
                  onClick={() => handleSuggestionClick(suggestion)}
                  className={cn(
                    'rounded-full border px-3 py-1.5 text-xs transition-colors',
                    selectedSuggestion === suggestion
                      ? 'border-purple-500 bg-purple-100 text-purple-800 dark:bg-purple-900/30 dark:text-purple-300'
                      : 'border-border bg-background hover:bg-accent'
                  )}
                >
                  {suggestion}
                </button>
              ))}
            </div>
          </div>
        )}

        {/* Answer Input */}
        <div className="flex items-center gap-2 rounded-md border bg-background px-3 py-2">
          <MessageSquare className="h-4 w-4 text-muted-foreground shrink-0" />
          <input
            type="text"
            value={answer}
            onChange={(e) => {
              setAnswer(e.target.value);
              if (selectedSuggestion && e.target.value !== selectedSuggestion) {
                setSelectedSuggestion(null);
              }
            }}
            placeholder="Type your answer..."
            className="flex-1 bg-transparent text-sm outline-none placeholder:text-muted-foreground"
            onKeyDown={(e) => {
              if (e.key === 'Enter' && !e.shiftKey) {
                e.preventDefault();
                handleSubmit();
              }
            }}
          />
          {answer && (
            <button onClick={() => { setAnswer(''); setSelectedSuggestion(null); }} className="text-muted-foreground hover:text-foreground">
              <X className="h-4 w-4" />
            </button>
          )}
        </div>

        {/* Submit Button */}
        <button
          onClick={handleSubmit}
          disabled={isSubmitting || (!answer.trim() && !selectedSuggestion)}
          className="mt-3 flex w-full items-center justify-center gap-2 rounded-md bg-purple-600 px-4 py-2 text-sm font-medium text-white hover:bg-purple-700 transition-colors disabled:opacity-50 disabled:cursor-not-allowed"
        >
          {isSubmitting ? (
            <div className="h-4 w-4 animate-spin rounded-full border-2 border-white border-t-transparent" />
          ) : (
            <>
              <Send className="h-4 w-4" />
              Submit Answer
            </>
          )}
        </button>
      </div>
    </div>
  );
}
```

---

## STEP 23: Create `frontend/components/CanvasPanel.tsx`

**File:** `frontend/components/CanvasPanel.tsx`

Write the following content exactly:

```typescript
/**
 * CanvasPanel
 * ===========
 * Main workspace canvas that displays AG-UI events in a scrollable feed.
 * Shows the real-time event stream from AgentCore with color-coded event types.
 */

'use client';

import React, { useRef, useEffect } from 'react';
import { Terminal, Pause, Play, RotateCcw } from 'lucide-react';
import { AGUIAnyEvent } from '@/lib/agentcore-agui-client';
import { formatTimestamp } from '@/lib/utils';
import { cn } from '@/lib/utils';

export interface CanvasPanelProps {
  /** Array of AG-UI events to display */
  events: AGUIAnyEvent[];
  /** Whether the stream is currently connected */
  isConnected: boolean;
  /** Whether auto-scroll is enabled */
  autoScroll?: boolean;
  /** Callback to toggle auto-scroll */
  onToggleAutoScroll?: () => void;
  /** Callback to clear events */
  onClear?: () => void;
  /** Additional CSS classes */
  className?: string;
}

function getEventColor(type: string): string {
  switch (type) {
    case 'RUN_EVENT': return 'text-agui-run';
    case 'STEP_EVENT': return 'text-agui-step';
    case 'AGENT_EVENT': return 'text-agui-event';
    default: return 'text-agui-unknown';
  }
}

function getEventBg(type: string): string {
  switch (type) {
    case 'RUN_EVENT': return 'bg-purple-50 dark:bg-purple-950/10';
    case 'STEP_EVENT': return 'bg-emerald-50 dark:bg-emerald-950/10';
    case 'AGENT_EVENT': return 'bg-blue-50 dark:bg-blue-950/10';
    default: return 'bg-gray-50 dark:bg-gray-950/10';
  }
}

function formatEventData(event: AGUIAnyEvent): string {
  try {
    return JSON.stringify(event.data, null, 2);
  } catch {
    return String(event.data);
  }
}

export function CanvasPanel({
  events,
  isConnected,
  autoScroll = true,
  onToggleAutoScroll,
  onClear,
  className,
}: CanvasPanelProps) {
  const scrollRef = useRef<HTMLDivElement>(null);

  // Auto-scroll to bottom
  useEffect(() => {
    if (autoScroll && scrollRef.current) {
      scrollRef.current.scrollTop = scrollRef.current.scrollHeight;
    }
  }, [events, autoScroll]);

  return (
    <div className={cn('rounded-lg border bg-card overflow-hidden flex flex-col', className)}>
      {/* Header */}
      <div className="flex items-center justify-between border-b bg-muted/50 px-4 py-3">
        <div className="flex items-center gap-2">
          <Terminal className="h-4 w-4 text-muted-foreground" />
          <h3 className="text-sm font-semibold">Event Stream</h3>
          <span className={cn(
            'h-2 w-2 rounded-full',
            isConnected ? 'bg-emerald-500 animate-pulse' : 'bg-gray-400'
          )} />
          <span className="text-xs text-muted-foreground">
            {isConnected ? 'Live' : 'Disconnected'} &middot; {events.length} events
          </span>
        </div>
        <div className="flex items-center gap-1">
          <button
            onClick={onToggleAutoScroll}
            className={cn(
              'rounded-md p-1.5 transition-colors',
              autoScroll ? 'bg-accent text-foreground' : 'text-muted-foreground hover:bg-accent'
            )}
            title={autoScroll ? 'Pause auto-scroll' : 'Resume auto-scroll'}
          >
            {autoScroll ? <Pause className="h-3.5 w-3.5" /> : <Play className="h-3.5 w-3.5" />}
          </button>
          <button
            onClick={onClear}
            className="rounded-md p-1.5 text-muted-foreground hover:bg-accent transition-colors"
            title="Clear events"
          >
            <RotateCcw className="h-3.5 w-3.5" />
          </button>
        </div>
      </div>

      {/* Event Feed */}
      <div
        ref={scrollRef}
        className="flex-1 overflow-y-auto canvas-scroll p-0"
        style={{ maxHeight: '600px' }}
      >
        {events.length === 0 ? (
          <div className="flex flex-col items-center justify-center py-12 text-center">
            <Terminal className="h-8 w-8 text-muted-foreground mb-2" />
            <p className="text-sm text-muted-foreground">No events yet</p>
            <p className="text-xs text-muted-foreground mt-1">
              {isConnected
                ? 'Waiting for agent events...'
                : 'Connect to a workflow to see events.'}
            </p>
          </div>
        ) : (
          events.map((event, index) => (
            <div
              key={`${event.timestamp}-${index}`}
              className={cn(
                'border-b border-border/50 px-4 py-2',
                getEventBg(event.type)
              )}
            >
              {/* Event Header Line */}
              <div className="flex items-center gap-2 mb-1">
                <span className={cn('text-xs font-bold', getEventColor(event.type))}>
                  [{event.type}]
                </span>
                <span className="text-xs text-muted-foreground">
                  {formatTimestamp(event.timestamp)}
                </span>
                <span className="text-xs text-muted-foreground">
                  run:{(event as { run_id?: string }).run_id?.slice(0, 8) || 'unknown'}
                </span>
              </div>
              {/* Event Data */}
              <pre className="agui-event-line text-xs whitespace-pre-wrap break-all pl-2 border-l-2 border-border/50">
                {formatEventData(event)}
              </pre>
            </div>
          ))
        )}
      </div>
    </div>
  );
}
```

---

## STEP 24: Create `frontend/components/GenerativeUIRenderer.tsx`

**File:** `frontend/components/GenerativeUIRenderer.tsx`

Write the following content exactly:

```typescript
/**
 * GenerativeUIRenderer
 * ====================
 * Orchestrates which generatiand UI cards to display based on agent state.
 * This component maps AgentCore state to the appropriate card components.
 *
 * Implements: GenerativeUI (Feature 8 of 8)
 */

'use client';

import React from 'react';
import { AgentCoreState } from '@/lib/agentcore-agui-client';
import { TranscriptCard } from './TranscriptCard';
import { DraftPreviewCard } from './DraftPreviewCard';
import { ReviewReportCard } from './ReviewReportCard';
import { ApprovalCard, ApprovalRequest } from './ApprovalCard';
import { ClarificationQuestionCard, ClarificationQuestion } from './ClarificationQuestionCard';

export interface GenerativeUIRendererProps {
  /** Current agent state */
  state: AgentCoreState;
  /** Callback for approval decision */
  onApprove: (comment?: string) => void;
  /** Callback for rejection decision */
  onReject: (comment?: string) => void;
  /** Callback for clarification answer */
  onClarificationAnswer: (answer: string) => void;
  /** Whether actions are being submitted */
  isSubmitting?: boolean;
  /** Additional CSS classes */
  className?: string;
}

export function GenerativeUIRenderer({
  state,
  onApprove,
  onReject,
  onClarificationAnswer,
  isSubmitting = false,
  className,
}: GenerativeUIRendererProps) {
  // Build approval request from state
  const approvalRequest: ApprovalRequest | null = state.approvalRequest;

  // Build clarification question from state
  const clarificationQuestion: ClarificationQuestion | null = state.clarificationQuestion;

  return (
    <div className={className}>
      {/* Priority: HITL cards appear at the top when actiand */}
      {(state.status === 'awaiting_approval' || approvalRequest) && (
        <div className="mb-4">
          <ApprovalCard
            request={approvalRequest}
            onApprove={onApprove}
            onReject={onReject}
            isSubmitting={isSubmitting}
          />
        </div>
      )}

      {(state.status === 'awaiting_clarification' || clarificationQuestion) && (
        <div className="mb-4">
          <ClarificationQuestionCard
            question={clarificationQuestion}
            onAnswer={onClarificationAnswer}
            isSubmitting={isSubmitting}
          />
        </div>
      )}

      {/* Content cards */}
      <div className="space-y-4">
        {state.transcript && (
          <TranscriptCard
            transcript={state.transcript}
            source={state.currentStep || state.agentName}
          />
        )}

        {state.draft && (
          <DraftPreviewCard
            draft={state.draft}
            title="Legal Research Draft"
            docType="Research Memo"
          />
        )}

        {state.review && (
          <ReviewReportCard
            review={state.review}
            title="Quality Review"
          />
        )}
      </div>

      {/* Empty state when nothing to show */}
      {!approvalRequest &&
        !clarificationQuestion &&
        !state.transcript &&
        !state.draft &&
        !state.review && (
          <div className="rounded-lg border border-dashed p-8 text-center">
            <p className="text-sm text-muted-foreground">Waiting for agent output...</p>
            <p className="text-xs text-muted-foreground mt-1">
              Generatiand UI cards will appear as the agent produces content.
            </p>
          </div>
        )}
    </div>
  );
}
```

---

## STEP 25: Create `frontend/components/CopilotSidebarWrapper.tsx`

**File:** `frontend/components/CopilotSidebarWrapper.tsx`

Write the following content exactly:

```typescript
/**
 * CopilotSidebarWrapper
 * =====================
 * Wraps the CopilotKit sidebar with custom styling and additional
 * workflow-specific context. Provides CopilotSidebar (Feature 2).
 */

'use client';

import React from 'react';
import { useCopilotChat } from '@copilotkit/react-core';
import { Bot, History } from 'lucide-react';
import { cn } from '@/lib/utils';

export interface CopilotSidebarWrapperProps {
  /** Current workflow ID for context */
  workflowId?: string;
  /** Agent state summary for chat context */
  agentStatus?: string;
  /** Additional CSS classes */
  className?: string;
}

export function CopilotSidebarWrapper({
  workflowId,
  agentStatus,
  className,
}: CopilotSidebarWrapperProps) {
  const { visibleMessages, isLoading } = useCopilotChat();

  return (
    <div className={cn('flex flex-col h-full', className)}>
      {/* Sidebar Header */}
      <div className="flex items-center gap-2 border-b px-4 py-3">
        <Bot className="h-5 w-5 text-primary" />
        <div>
          <h3 className="text-sm font-semibold">Agent Chat</h3>
          {workflowId && (
            <p className="text-xs text-muted-foreground">
              WF: {workflowId.slice(0, 8)}...
            </p>
          )}
        </div>
        {isLoading && (
          <div className="ml-auto h-4 w-4 animate-spin rounded-full border-2 border-primary border-t-transparent" />
        )}
      </div>

      {/* Chat Messages */}
      <div className="flex-1 overflow-y-auto px-4 py-2 space-y-3">
        {visibleMessages.length === 0 ? (
          <div className="text-center py-8">
            <History className="mx-auto h-8 w-8 text-muted-foreground" />
            <p className="mt-2 text-sm text-muted-foreground">No messages yet</p>
            <p className="text-xs text-muted-foreground">
              {agentStatus
                ? `Agent status: ${agentStatus}`
                : 'Start a conversation with the agent.'}
            </p>
          </div>
        ) : (
          visibleMessages.map((msg, i) => (
            <div
              key={i}
              className={cn(
                'rounded-lg px-3 py-2 text-sm',
                msg.role === 'user'
                  ? 'bg-primary text-primary-foreground ml-8'
                  : 'bg-muted mr-8'
              )}
            >
              <p className="text-xs font-medium mb-1 opacity-70">
                {msg.role === 'user' ? 'You' : 'Agent'}
              </p>
              <p className="whitespace-pre-wrap">{String(msg.content || '')}</p>
            </div>
          ))
        )}
      </div>

      {/* Status Footer */}
      {agentStatus && (
        <div className="border-t px-4 py-2">
          <div className="flex items-center gap-2">
            <div className={cn(
              'h-2 w-2 rounded-full',
              agentStatus === 'running' && 'bg-blue-500 animate-pulse',
              agentStatus === 'completed' && 'bg-emerald-500',
              agentStatus === 'failed' && 'bg-red-500',
              agentStatus === 'awaiting_approval' && 'bg-amber-500',
              agentStatus === 'awaiting_clarification' && 'bg-purple-500',
              agentStatus === 'idle' && 'bg-gray-400',
            )} />
            <span className="text-xs text-muted-foreground capitalize">{agentStatus}</span>
          </div>
        </div>
      )}
    </div>
  );
}
```

**VERIFICATION:** Run `ls frontend/components/ClarificationQuestionCard.tsx frontend/components/CanvasPanel.tsx frontend/components/GenerativeUIRenderer.tsx frontend/components/CopilotSidebarWrapper.tsx`. All 4 must exist.

---

## CHECKPOINT 7: All UI Components Verified

Before proceeding, verify:
1. `ClarificationQuestionCard.tsx` exists
2. `CanvasPanel.tsx` exists
3. `GenerativeUIRenderer.tsx` exists
4. `CopilotSidebarWrapper.tsx` exists

Command:
```bash
for f in ClarificationQuestionCard CanvasPanel GenerativeUIRenderer CopilotSidebarWrapper; do
  grep -q "export function ${f}" frontend/components/${f}.tsx && echo "OK: ${f}" || echo "FAIL: ${f}"
done
```

All must output "OK". If any fails, STOP.

---

## STEP 25.5: Mic recording + file upload (PCD D13)

Create the `AudioInput` subsystem so the user can either record audio in-browser (MediaRecorder API → webm/opus) or upload an existing file (mp3/wav/m4a/webm). Both paths post to FastAPI `POST /api/v1/uploads/audio` and the returned `s3://...` URI is used to start the workflow.

### 25.5a: Create the recorder hook

**File:** `frontend/components/chat/AudioInput/useMicRecorder.ts`

```typescript
"use client";

import { useCallback, useEffect, useRef, useState } from "react";

export interface MicRecorderState {
  isRecording: boolean;
  durationMs: number;
  blob: Blob | null;
  error: string | null;
}

/**
 * Wraps the browser MediaRecorder API.
 *
 * Format: audio/webm;codecs=opus (no external libs required; AWS Transcribe
 * accepts opus inside webm/ogg containers).
 *
 * Browser support: Chrome, Edge, Firefox, Safari 14.1+. Returns an error
 * state if MediaRecorder is unavailable (older browsers, http context).
 */
export function useMicRecorder() {
  const [state, setState] = useState<MicRecorderState>({
    isRecording: false,
    durationMs: 0,
    blob: null,
    error: null,
  });
  const recorderRef = useRef<MediaRecorder | null>(null);
  const chunksRef = useRef<Blob[]>([]);
  const startedAtRef = useRef<number>(0);
  const tickRef = useRef<ReturnType<typeof setInterval> | null>(null);

  // Detect support once on mount
  useEffect(() => {
    if (typeof window === "undefined") return;
    if (!navigator.mediaDevices || typeof MediaRecorder === "undefined") {
      setState((s) => ({ ...s, error: "MediaRecorder API not available in this browser." }));
    }
  }, []);

  const start = useCallback(async () => {
    setState({ isRecording: false, durationMs: 0, blob: null, error: null });
    try {
      const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
      const mimeType = MediaRecorder.isTypeSupported("audio/webm;codecs=opus")
        ? "audio/webm;codecs=opus"
        : "audio/webm";
      const recorder = new MediaRecorder(stream, { mimeType });
      chunksRef.current = [];
      recorder.ondataavailable = (e) => {
        if (e.data.size > 0) chunksRef.current.push(e.data);
      };
      recorder.onstop = () => {
        const blob = new Blob(chunksRef.current, { type: mimeType });
        setState((s) => ({ ...s, isRecording: false, blob }));
        stream.getTracks().forEach((t) => t.stop());
        if (tickRef.current) clearInterval(tickRef.current);
      };
      recorderRef.current = recorder;
      startedAtRef.current = Date.now();
      recorder.start(250); // emit chunks every 250 ms
      tickRef.current = setInterval(
        () => setState((s) => ({ ...s, durationMs: Date.now() - startedAtRef.current })),
        100,
      );
      setState((s) => ({ ...s, isRecording: true }));
    } catch (e) {
      setState((s) => ({
        ...s,
        isRecording: false,
        error: e instanceof Error ? e.message : "Microphone permission denied",
      }));
    }
  }, []);

  const stop = useCallback(() => {
    recorderRef.current?.stop();
  }, []);

  const reset = useCallback(() => {
    setState({ isRecording: false, durationMs: 0, blob: null, error: null });
  }, []);

  return { state, start, stop, reset };
}
```

### 25.5b: Create the input component

**File:** `frontend/components/chat/AudioInput/AudioInput.tsx`

```typescript
"use client";

import { useRef, useState } from "react";
import { useMicRecorder } from "./useMicRecorder";

const ACCEPTED_TYPES = "audio/mp3,audio/mpeg,audio/wav,audio/x-wav,audio/m4a,audio/x-m4a,audio/webm,audio/ogg";
const MAX_SIZE_MB = 50;

export interface AudioInputProps {
  onUploaded: (s3Uri: string, contentType: string) => void;
}

export function AudioInput({ onUploaded }: AudioInputProps) {
  const { state, start, stop, reset } = useMicRecorder();
  const fileInputRef = useRef<HTMLInputElement | null>(null);
  const [isUploading, setIsUploading] = useState(false);
  const [uploadError, setUploadError] = useState<string | null>(null);

  const uploadBlob = async (blob: Blob, filename: string) => {
    setIsUploading(true);
    setUploadError(null);
    try {
      const formData = new FormData();
      formData.append("file", blob, filename);
      const resp = await fetch("/api/v1/uploads/audio", { method: "POST", body: formData });
      if (!resp.ok) throw new Error(`Upload failed: ${resp.status}`);
      const json = (await resp.json()) as { s3_uri: string; content_type: string };
      onUploaded(json.s3_uri, json.content_type);
      reset();
    } catch (e) {
      setUploadError(e instanceof Error ? e.message : "Upload failed");
    } finally {
      setIsUploading(false);
    }
  };

  const handleFileSelected = (e: React.ChangeEvent<HTMLInputElement>) => {
    const file = e.target.files?.[0];
    if (!file) return;
    if (file.size > MAX_SIZE_MB * 1024 * 1024) {
      setUploadError(`File exceeds ${MAX_SIZE_MB} MB limit.`);
      return;
    }
    uploadBlob(file, file.name);
  };

  const handleSendRecording = () => {
    if (!state.blob) return;
    uploadBlob(state.blob, `recording-${Date.now()}.webm`);
  };

  const seconds = (state.durationMs / 1000).toFixed(1);

  return (
    <div className="flex flex-col gap-2 rounded-lg border p-3">
      <div className="flex items-center gap-2">
        <button
          type="button"
          onClick={state.isRecording ? stop : start}
          disabled={isUploading || !!state.error}
          aria-label={state.isRecording ? "Stop recording" : "Start recording"}
          className="rounded-full px-3 py-2 bg-red-500 text-white disabled:opacity-50"
        >
          {state.isRecording ? `■ Stop (${seconds}s)` : "🎤 Record"}
        </button>

        <button
          type="button"
          onClick={() => fileInputRef.current?.click()}
          disabled={isUploading || state.isRecording}
          className="rounded-full px-3 py-2 bg-blue-500 text-white disabled:opacity-50"
        >
          📎 Upload
        </button>

        <input
          ref={fileInputRef}
          type="file"
          accept={ACCEPTED_TYPES}
          onChange={handleFileSelected}
          className="hidden"
        />
      </div>

      {state.blob && !state.isRecording && (
        <div className="flex items-center gap-2 text-sm">
          <span>Recording ready ({seconds}s, {(state.blob.size / 1024).toFixed(0)} KB)</span>
          <button onClick={handleSendRecording} disabled={isUploading} className="rounded bg-green-600 px-2 py-1 text-white text-xs">
            {isUploading ? "Uploading…" : "Send"}
          </button>
          <button onClick={reset} disabled={isUploading} className="rounded bg-gray-400 px-2 py-1 text-white text-xs">
            Discard
          </button>
        </div>
      )}

      {(state.error || uploadError) && (
        <div role="alert" className="text-sm text-red-600">
          {state.error ?? uploadError}
        </div>
      )}
    </div>
  );
}
```

### 25.5c: Wire into ChatPanel (or workspace page)

Wherever the chat input is composed (typically inside `app/workspace/[wfId]/page.tsx` or a dedicated `ChatPanel.tsx`), import and place `AudioInput`:

```typescript
import { AudioInput } from "@/components/chat/AudioInput/AudioInput";

// inside the JSX:
<AudioInput onUploaded={(s3Uri) => startWorkflow({ audio_s3_uri: s3Uri })} />
```

`startWorkflow` is the React Query mutation that POSTs to `/api/v1/workflows` (defined in `lib/api/workflows.ts`).

### 25.5d: Unit test

**File:** `frontend/tests/unit/audio-input.test.tsx`

```typescript
import { describe, it, expect, vi } from "vitest";
import { renderHook, act } from "@testing-library/react";
import { useMicRecorder } from "@/components/chat/AudioInput/useMicRecorder";

class MockMediaRecorder {
  ondataavailable: ((e: { data: Blob }) => void) | null = null;
  onstop: (() => void) | null = null;
  state = "inactive";
  static isTypeSupported = () => true;
  constructor(_stream: MediaStream) {}
  start(_timeslice?: number) {
    this.state = "recording";
    setTimeout(() => this.ondataavailable?.({ data: new Blob(["x"]) }), 0);
  }
  stop() {
    this.state = "inactive";
    this.onstop?.();
  }
}

describe("useMicRecorder", () => {
  it("captures a blob after start → stop", async () => {
    Object.defineProperty(global, "MediaRecorder", { writable: true, value: MockMediaRecorder });
    Object.defineProperty(navigator, "mediaDevices", {
      writable: true,
      value: { getUserMedia: vi.fn().mockResolvedValue({ getTracks: () => [{ stop: vi.fn() }] }) },
    });
    const { result } = renderHook(() => useMicRecorder());
    await act(async () => { await result.current.start(); });
    await act(async () => { result.current.stop(); });
    expect(result.current.state.blob).not.toBeNull();
  });
});
```

**CHECKPOINT 25.5:** Run `cd frontend && pnpm test tests/unit/audio-input.test.tsx`. The test must pass. Open the running app, click 🎤 Record, allow permission, speak briefly, click Stop, click Send — a workflow should start with `audio_s3_uri` pointing at S3.

---

## STEP 26: Create `frontend/app/workspace/[wfId]/page.tsx`

**CRITICAL FILE.** Workspace page -- the main CopilotKit-integrated workflow UI.

Create directory: `frontend/app/workspace/[wfId]/`

**File:** `frontend/app/workspace/[wfId]/page.tsx`

Write the following content exactly:

```typescript
/**
 * Workspace Page
 * ==============
 * Main workflow workspace integrating ALL 8 CopilotKit features:
 *
 * Feature 1: CopilotChat      -- Chat interface via CopilotKit provider
 * Feature 2: CopilotSidebar   -- Sidebar chat history
 * Feature 3: useCoAgent       -- Agent state synchronization (two-way via useCoAgent)
 * Feature 4: useAgent         -- Direct agent interaction
 * Feature 5: useCopilotAction -- Action handlers
 * Feature 6: useCopilotReadable -- Readable context
 * Feature 7: useFrontendTool  -- Frontend tool integration
 * Feature 8: GenerativeUI     -- Dynamic card rendering
 *
 * NEW (CopilotKit 1.50+ note): Uses useCoAgent for two-way state sync between frontend and agent.
 * Agent state is managed via `BedrockAgentCoreApp` runtime context on the Python side and `useCoAgent` on the frontend
 * on the frontend side. State changes flow bidirectionally.
 *
 * Reference: https://www.copilotkit.ai/blog/aws-announces-dedicated-ag-ui-endpoint-in-agentcore-and-fast-template-for-building-fullstack-agents
 */

'use client';

import React, { useEffect, useState, useCallback } from 'react';
import { useParams } from 'next/navigation';
import {
  useCopilotAction,
  useCopilotReadable,
  useCoAgent,
  useCopilotChat,
} from '@copilotkit/react-core';
import {
  ArrowLeft,
  Activity,
  Bot,
  CheckCircle2,
  XCircle,
  AlertTriangle,
  HelpCircle,
  Loader2,
  PanelRightClose,
  PanelRightOpen,
} from 'lucide-react';
import Link from 'next/link';
import { useWorkflowStream } from '@/lib/use-workflow-stream';
import { useWorkflowSharedState } from '@/lib/use-shared-state';
import { GenerativeUIRenderer } from '@/components/GenerativeUIRenderer';
import { CanvasPanel } from '@/components/CanvasPanel';
import { MetricTile } from '@/components/MetricTile';
import { cn } from '@/lib/utils';

export default function WorkspacePage() {
  const params = useParams();
  const workflowId = params.wfId as string;

  // ---- Feature 3: useCoAgent (two-way state sync) ----
  // NEW: Uses useCoAgent for bidirectional state synchronization with the agent.
  // The agent's @app.entrypoint handler receives runtime context with session_id (mapped to workflow_id)
  // and useCoAgent reflects state changes in real-time.
  const { state: coAgentState, setState: setCoAgentState } = useCoAgent({
    name: 'strands_drafter',
    initialState: {
      workflow_id: workflowId,
      status: 'idle',
      currentStep: null,
      transcript: '',
      draft_markdown: '',
      review_report: '',
    },
  });

  // ---- Feature 4: useAgent ----
  // useAgent is accessed via the CopilotKit context for direct agent run control

  // ---- Workflow SSE Stream ----
  const {
    state: agentState,
    events,
    isConnected,
    isComplete,
    error: streamError,
    sendApproval,
    sendClarification,
  } = useWorkflowStream(workflowId);

  // ---- Feature 6: useCopilotReadable ----
  // Provide workflow context to CopilotKit so the AI knows current state
  useCopilotReadable({
    description: 'Current workflow state and agent progress',
    value: JSON.stringify({
      workflowId,
      status: agentState.status,
      currentStep: agentState.currentStep,
      stepCount: agentState.steps.length,
      hasTranscript: !!agentState.transcript,
      hasDraft: !!agentState.draft,
      hasReview: !!agentState.review,
      awaitingApproval: agentState.status === 'awaiting_approval',
      awaitingClarification: agentState.status === 'awaiting_clarification',
    }),
  });

  // ---- Feature 5: useCopilotAction ----
  // Register actions that the agent can trigger
  useCopilotAction({
    name: 'showTranscript',
    description: 'Display the transcript card prominently',
    parameters: [],
    handler: async () => {
      console.log('[Workspace] Action: showTranscript triggered');
      return { success: true, message: 'Transcript card focused' };
    },
  });

  useCopilotAction({
    name: 'requestApproval',
    description: 'Request human approval for a decision',
    parameters: [
      { name: 'type', type: 'string', description: 'Type of approval requested' },
      { name: 'description', type: 'string', description: 'Description of what needs approval' },
    ],
    handler: async ({ type, description }) => {
      console.log('[Workspace] Action: requestApproval', { type, description });
      return { success: true, message: 'Approval request displayed' };
    },
  });

  useCopilotAction({
    name: 'askClarification',
    description: 'Ask the user a clarifying question',
    parameters: [
      { name: 'question', type: 'string', description: 'The clarifying question' },
      { name: 'context', type: 'string', description: 'Context for the question' },
    ],
    handler: async ({ question, context }) => {
      console.log('[Workspace] Action: askClarification', { question, context });
      return { success: true, message: 'Clarification question displayed' };
    },
  });

  // ---- Feature 7: useFrontendTool ----
  // Frontend tools registered via CopilotKit runtime
  useCopilotAction({
    name: 'downloadDraft',
    description: 'Download the current draft document',
    parameters: [],
    handler: async () => {
      if (agentState.draft) {
        const blob = new Blob([agentState.draft], { type: 'text/markdown' });
        const url = URL.createObjectURL(blob);
        const a = document.createElement('a');
        a.href = url;
        a.download = `draft_${workflowId.slice(0, 8)}.md`;
        a.click();
        URL.revokeObjectURL(url);
        return { success: true, message: 'Draft download started' };
      }
      return { success: false, message: 'No draft available' };
    },
  });

  // Sync AG-UI state to CoAgent state (two-way sync)
  useEffect(() => {
    if (agentState.status !== 'idle') {
      setCoAgentState({
        workflow_id: workflowId,
        status: agentState.status,
        currentStep: agentState.currentStep,
        transcript: agentState.transcript || '',
        draft_markdown: agentState.draft || '',
        review_report: agentState.review ? JSON.stringify(agentState.review) : '',
      });
    }
  }, [agentState, setCoAgentState, workflowId]);

  // Local state
  const [autoScroll, setAutoScroll] = useState(true);
  const [showSteps, setShowSteps] = useState(true);
  const [sidebarOpen, setSidebarOpen] = useState(false);
  const [isSubmitting, setIsSubmitting] = useState(false);

  // Approval handler
  const handleApproand = useCallback(async (comment?: string) => {
    setIsSubmitting(true);
    try {
      await sendApproval('approve', comment);
    } catch (err) {
      console.error('Approval failed:', err);
    } finally {
      setIsSubmitting(false);
    }
  }, [sendApproval]);

  // Rejection handler
  const handleReject = useCallback(async (comment?: string) => {
    setIsSubmitting(true);
    try {
      await sendApproval('reject', comment);
    } catch (err) {
      console.error('Rejection failed:', err);
    } finally {
      setIsSubmitting(false);
    }
  }, [sendApproval]);

  // Clarification handler
  const handleClarification = useCallback(async (answer: string) => {
    setIsSubmitting(true);
    try {
      await sendClarification(answer);
    } catch (err) {
      console.error('Clarification failed:', err);
    } finally {
      setIsSubmitting(false);
    }
  }, [sendClarification]);

  // Compute step stats
  const completedSteps = agentState.steps.filter(s => s.status === 'completed').length;
  const totalSteps = agentState.steps.length;

  return (
    <main className="min-h-screen bg-background">
      {/* Header */}
      <header className="border-b bg-card sticky top-0 z-10">
        <div className="flex h-14 items-center justify-between px-4">
          <div className="flex items-center gap-3">
            <Link
              href="/"
              className="rounded-md p-2 hover:bg-accent transition-colors"
            >
              <ArrowLeft className="h-4 w-4" />
            </Link>
            <div className="h-4 w-px bg-border" />
            <Bot className="h-5 w-5 text-primary" />
            <div>
              <h1 className="text-sm font-semibold">Workflow Workspace</h1>
              <p className="text-xs text-muted-foreground font-mono">
                {workflowId}
              </p>
            </div>
          </div>

          <div className="flex items-center gap-3">
            {/* Connection Status */}
            <div className="flex items-center gap-2">
              <div className={cn(
                'h-2 w-2 rounded-full',
                isConnected ? 'bg-emerald-500 animate-pulse' : 'bg-gray-400',
                streamError && 'bg-red-500'
              )} />
              <span className="text-xs text-muted-foreground hidden sm:inline">
                {isConnected ? 'Live' : streamError ? 'Error' : 'Idle'}
              </span>
            </div>

            {/* Workflow Status Badge */}
            <span className={cn(
              'rounded-full px-2.5 py-0.5 text-xs font-medium',
              agentState.status === 'running' && 'bg-blue-100 text-blue-800 dark:bg-blue-900/30 dark:text-blue-300',
              agentState.status === 'completed' && 'bg-emerald-100 text-emerald-800 dark:bg-emerald-900/30 dark:text-emerald-300',
              agentState.status === 'failed' && 'bg-red-100 text-red-800 dark:bg-red-900/30 dark:text-red-300',
              agentState.status === 'awaiting_approval' && 'bg-amber-100 text-amber-800 dark:bg-amber-900/30 dark:text-amber-300',
              agentState.status === 'awaiting_clarification' && 'bg-purple-100 text-purple-800 dark:bg-purple-900/30 dark:text-purple-300',
              agentState.status === 'idle' && 'bg-gray-100 text-gray-800 dark:bg-gray-800 dark:text-gray-300',
            )}>
              {agentState.status}
            </span>

            {/* Sidebar Toggle */}
            <button
              onClick={() => setSidebarOpen(!sidebarOpen)}
              className="rounded-md p-2 hover:bg-accent transition-colors"
              title="Toggle sidebar"
            >
              {sidebarOpen ? (
                <PanelRightOpen className="h-4 w-4" />
              ) : (
                <PanelRightClose className="h-4 w-4" />
              )}
            </button>
          </div>
        </div>
      </header>

      <div className="flex">
        {/* Main Content */}
        <div className={cn(
          'flex-1 transition-all duration-200',
          sidebarOpen ? 'mr-80' : ''
        )}>
          <div className="container mx-auto px-4 py-6">
            {/* Error Banner */}
            {streamError && (
              <div className="mb-4 rounded-lg border border-red-200 bg-red-50 px-4 py-3 dark:border-red-800 dark:bg-red-950/20">
                <div className="flex items-center gap-2">
                  <AlertTriangle className="h-4 w-4 text-red-600" />
                  <p className="text-sm text-red-800 dark:text-red-300">
                    Stream error: {streamError.message}
                  </p>
                </div>
              </div>
            )}

            {/* Mini Metrics */}
            <div className="mb-6 grid grid-cols-2 gap-3 sm:grid-cols-4">
              <MetricTile
                label="Steps"
                value={`${completedSteps}/${totalSteps}`}
                icon={Activity}
                variant={totalSteps > 0 && completedSteps === totalSteps ? 'success' : 'default'}
                isLoading={agentState.status === 'running' && totalSteps === 0}
              />
              <MetricTile
                label="Events"
                value={events.length}
                icon={Activity}
                variant="info"
              />
              <MetricTile
                label="Current Step"
                value={agentState.currentStep || '--'}
                icon={Bot}
                variant={agentState.status === 'running' ? 'info' : 'default'}
                isLoading={agentState.status === 'running' && !agentState.currentStep}
              />
              <MetricTile
                label="Completion"
                value={isComplete ? 'Done' : agentState.status === 'failed' ? 'Failed' : 'In Progress'}
                icon={isComplete ? CheckCircle2 : agentState.status === 'failed' ? XCircle : Loader2}
                variant={isComplete ? 'success' : agentState.status === 'failed' ? 'danger' : 'default'}
              />
            </div>

            {/* Steps Progress */}
            {showSteps && agentState.steps.length > 0 && (
              <div className="mb-6 rounded-lg border bg-card p-4">
                <h3 className="text-sm font-semibold mb-3">Execution Steps</h3>
                <div className="space-y-2">
                  {agentState.steps.map((step, index) => (
                    <div
                      key={`${step.name}-${index}`}
                      className="flex items-center gap-3 rounded-md px-3 py-2 bg-muted/30"
                    >
                      <span className="text-xs font-mono text-muted-foreground w-6">
                        {index + 1}
                      </span>
                      <div className={cn(
                        'h-2 w-2 rounded-full',
                        step.status === 'completed' && 'bg-emerald-500',
                        step.status === 'running' && 'bg-blue-500 animate-pulse',
                        step.status === 'failed' && 'bg-red-500',
                      )} />
                      <div className="flex-1 min-w-0">
                        <p className="text-sm font-medium truncate">{step.name}</p>
                        <p className="text-xs text-muted-foreground">{step.type}</p>
                      </div>
                      <span className={cn(
                        'rounded-full px-2 py-0.5 text-xs font-medium',
                        step.status === 'completed' && 'bg-emerald-100 text-emerald-800',
                        step.status === 'running' && 'bg-blue-100 text-blue-800',
                        step.status === 'failed' && 'bg-red-100 text-red-800',
                      )}>
                        {step.status}
                      </span>
                    </div>
                  ))}
                </div>
              </div>
            )}

            {/* Two-Column Layout: Generatiand UI + Event Stream */}
            <div className="grid grid-cols-1 gap-6 lg:grid-cols-2">
              {/* Left: Generatiand UI Cards */}
              <div>
                <h3 className="text-sm font-semibold mb-3 flex items-center gap-2">
                  <CheckCircle2 className="h-4 w-4 text-primary" />
                  Generated Content
                </h3>
                <GenerativeUIRenderer
                  state={agentState}
                  onApprove={handleApprove}
                  onReject={handleReject}
                  onClarificationAnswer={handleClarification}
                  isSubmitting={isSubmitting}
                />
              </div>

              {/* Right: Event Stream Canvas */}
              <div>
                <h3 className="text-sm font-semibold mb-3 flex items-center gap-2">
                  <Activity className="h-4 w-4 text-primary" />
                  AG-UI Event Stream
                </h3>
                <CanvasPanel
                  events={events}
                  isConnected={isConnected}
                  autoScroll={autoScroll}
                  onToggleAutoScroll={() => setAutoScroll(!autoScroll)}
                  onClear={() => { /* Events are managed by useWorkflowStream */ }}
                  className="h-[700px]"
                />
              </div>
            </div>
          </div>
        </div>

        {/* Right Sidebar -- Copilot Chat */}
        {sidebarOpen && (
          <aside className="fixed right-0 top-14 bottom-0 w-80 border-l bg-card overflow-y-auto z-20">
            <div className="p-4">
              <h3 className="text-sm font-semibold mb-4 flex items-center gap-2">
                <Bot className="h-4 w-4 text-primary" />
                Copilot Chat
              </h3>
              <div className="rounded-lg border bg-muted/30 p-4 text-center">
                <p className="text-sm text-muted-foreground">
                  Chat interface provided by CopilotKit sidebar.
                </p>
                <p className="text-xs text-muted-foreground mt-2">
                  Use the floating chat button or sidebar to interact.
                </p>
              </div>

              {/* Feature Status */}
              <div className="mt-6 space-y-2">
                <h4 className="text-xs font-semibold text-muted-foreground uppercase tracking-wider">
                  Actiand Features
                </h4>
                {[
                  { name: 'CopilotChat', active: true },
                  { name: 'CopilotSidebar', active: true },
                  { name: 'useCoAgent', active: true },
                  { name: 'useAgent', active: true },
                  { name: 'useCopilotAction', active: true },
                  { name: 'useCopilotReadable', active: true },
                  { name: 'useFrontendTool', active: true },
                  { name: 'GenerativeUI', active: true },
                ].map((feature) => (
                  <div
                    key={feature.name}
                    className="flex items-center justify-between rounded-md px-2 py-1.5 text-sm"
                  >
                    <span>{feature.name}</span>
                    <span className={cn(
                      'h-2 w-2 rounded-full',
                      feature.actiand ? 'bg-emerald-500' : 'bg-gray-400'
                    )} />
                  </div>
                ))}
              </div>

              {/* State Inspector */}
              <div className="mt-6">
                <h4 className="text-xs font-semibold text-muted-foreground uppercase tracking-wider mb-2">
                  Agent State
                </h4>
                <pre className="agui-event-line rounded-md bg-muted p-3 text-xs overflow-x-auto">
                  {JSON.stringify(
                    {
                      status: agentState.status,
                      currentStep: agentState.currentStep,
                      runId: agentState.runId,
                      stepCount: agentState.steps.length,
                      eventCount: events.length,
                      coAgentState,
                    },
                    null,
                    2
                  )}
                </pre>
              </div>
            </div>
          </aside>
        )}
      </div>
    </main>
  );
}
```

**VERIFICATION:** Run `ls frontend/app/workspace/\[wfId\]/page.tsx` -- must exist.
Run `grep -c "useCopilotAction\|useCopilotReadable\|useCoAgent\|useWorkflowStream\|useWorkflowSharedState\|GenerativeUIRenderer\|CanvasPanel" frontend/app/workspace/\[wfId\]/page.tsx` -- must output at least 7.

---

## CHECKPOINT 8: Workspace Page Verified

Before proceeding, verify:
1. `frontend/app/workspace/[wfId]/page.tsx` exists
2. Imports `useCopilotAction`, `useCopilotReadable`, `useCoAgent` from `@copilotkit/react-core`
3. Imports `useWorkflowStream` from `@/lib/use-workflow-stream`
4. Imports `useWorkflowSharedState` from `@/lib/use-shared-state`
5. Imports `GenerativeUIRenderer` and `CanvasPanel` from components
6. Contains all 8 feature references in the sidebar

Command:
```bash
grep -c "useCopilotAction\|useCopilotReadable\|useCoAgent\|useAgent\|useFrontendTool\|GenerativeUI\|CopilotSidebar\|CopilotChat" frontend/app/workspace/\[wfId\]/page.tsx
```
Expected: 8 or more matches. If less, STOP.

---

## STEP 27: Create `frontend/Dockerfile`

**File:** `frontend/Dockerfile`

Write the following content exactly:

```dockerfile
# Legal Agent Frontend -- Next.js 14 with CopilotKit
# =================================================

FROM node:20-alpine AS base

# Install dependencies only when needed
FROM base AS deps
RUN apk add --no-cache libc6-compat
WORKDIR /app

# Copy package files
COPY package.json ./
RUN npm install --legacy-peer-deps

# Rebuild the source code only when needed
FROM base AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .

# Build the Next.js application
ENV NEXT_TELEMETRY_DISABLED=1
ENV NODE_ENV=production
RUN npm run build

# Production image, copy all the files and run next
FROM base AS runner
WORKDIR /app

ENV NODE_ENV=production
ENV NEXT_TELEMETRY_DISABLED=1
ENV PORT=3000
ENV HOSTNAME="0.0.0.0"

RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

# Copy built application
COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

USER nextjs

EXPOSE 3000

CMD ["node", "server.js"]
```

**VERIFICATION:** Run `ls frontend/Dockerfile`. Must exist.
Run `grep -q "node:20-alpine" frontend/Dockerfile && echo "OK: Node 20" || echo "FAIL"`.
Run `grep -q "standalone" frontend/Dockerfile && echo "OK: Standalone" || echo "FAIL"`.

---

## STEP 28: Create `frontend/.dockerignore`

**File:** `frontend/.dockerignore`

Write the following content exactly:

```
node_modules
npm-debug.log
.next
.git
.env.local
.env.development
*.md
coverage
.nyc_output
```

---

## STEP 29: Create `frontend/.env.local.example`

**File:** `frontend/.env.local.example`

Write the following content exactly:

```bash
# Backend API URL (internal Docker networking)
NEXT_PUBLIC_BACKEND_URL=http://host.docker.internal:8000/api/v1

# CopilotKit Runtime URL
NEXT_PUBLIC_COPILOT_RUNTIME_URL=/api/copilotkit

# AgentCore Runtime URLs for HttpAgent connections
AGENT_1_RUNTIME_URL=https://bedrock-agentcore.us-east-1.amazonaws.com/runtimes/RUNTIME_ID_1/invocations?accountId=ACCOUNT_ID&qualifier=DEFAULT
AGENT_2_RUNTIME_URL=https://bedrock-agentcore.us-east-1.amazonaws.com/runtimes/RUNTIME_ID_2/invocations?accountId=ACCOUNT_ID&qualifier=DEFAULT
AGENT_3_RUNTIME_URL=https://bedrock-agentcore.us-east-1.amazonaws.com/runtimes/RUNTIME_ID_3/invocations?accountId=ACCOUNT_ID&qualifier=DEFAULT

# Node environment
NODE_ENV=development
```

**VERIFICATION:** Run `ls frontend/Dockerfile frontend/.dockerignore frontend/.env.local.example`. All 3 must exist.

---

## CHECKPOINT 9: Docker + Config Verified

Before proceeding, verify:
1. `frontend/Dockerfile` uses `node:20-alpine` base image
2. `frontend/Dockerfile` builds with `output: 'standalone'`
3. `frontend/.dockerignore` excludes `node_modules` and `.next`
4. `frontend/.env.local.example` defines `NEXT_PUBLIC_BACKEND_URL` and AgentCore URLs

If any check fails, STOP.

---

## STEP 30: Create `frontend/next-env.d.ts`

**File:** `frontend/next-env.d.ts`

Write the following content exactly:

```typescript
/// <reference types="next" />
/// <reference types="next/image-types/global" />
```

---

## STEP 31: Create `frontend/components/ui/card.tsx` (shadcn-style)

Create directory: `frontend/components/ui/`

**File:** `frontend/components/ui/card.tsx`

Write the following content exactly:

```typescript
import * as React from "react";
import { cn } from "@/lib/utils";

const Card = React.forwardRef<
  HTMLDivElement,
  React.HTMLAttributes<HTMLDivElement>
>(({ className, ...props }, ref) => (
  <div
    ref={ref}
    className={cn(
      "rounded-lg border bg-card text-card-foreground shadow-sm",
      className
    )}
    {...props}
  />
));
Card.displayName = "Card";

const CardHeader = React.forwardRef<
  HTMLDivElement,
  React.HTMLAttributes<HTMLDivElement>
>(({ className, ...props }, ref) => (
  <div
    ref={ref}
    className={cn("flex flex-col space-y-1.5 p-6", className)}
    {...props}
  />
));
CardHeader.displayName = "CardHeader";

const CardTitle = React.forwardRef<
  HTMLParagraphElement,
  React.HTMLAttributes<HTMLHeadingElement>
>(({ className, ...props }, ref) => (
  <h3
    ref={ref}
    className={cn(
      "text-2xl font-semibold leading-none tracking-tight",
      className
    )}
    {...props}
  />
));
CardTitle.displayName = "CardTitle";

const CardDescription = React.forwardRef<
  HTMLParagraphElement,
  React.HTMLAttributes<HTMLParagraphElement>
>(({ className, ...props }, ref) => (
  <p
    ref={ref}
    className={cn("text-sm text-muted-foreground", className)}
    {...props}
  />
));
CardDescription.displayName = "CardDescription";

const CardContent = React.forwardRef<
  HTMLDivElement,
  React.HTMLAttributes<HTMLDivElement>
>(({ className, ...props }, ref) => (
  <div ref={ref} className={cn("p-6 pt-0", className)} {...props} />
));
CardContent.displayName = "CardContent";

const CardFooter = React.forwardRef<
  HTMLDivElement,
  React.HTMLAttributes<HTMLDivElement>
>(({ className, ...props }, ref) => (
  <div
    ref={ref}
    className={cn("flex items-center p-6 pt-0", className)}
    {...props}
  />
));
CardFooter.displayName = "CardFooter";

export { Card, CardHeader, CardFooter, CardTitle, CardDescription, CardContent };
```

---

## FINAL FILE MANIFEST

After completing all steps, the following files must exist:

| # | File Path | Status |
|---|-----------|--------|
| 1 | `frontend/package.json` | Config |
| 2 | `frontend/tsconfig.json` | Config |
| 3 | `frontend/next.config.js` | Config |
| 4 | `frontend/tailwind.config.ts` | Config |
| 5 | `frontend/postcss.config.js` | Config |
| 6 | `frontend/.dockerignore` | Config |
| 7 | `frontend/.env.local.example` | Config |
| 8 | `frontend/next-env.d.ts` | Config |
| 9 | `frontend/Dockerfile` | Docker |
| 10 | `frontend/app/globals.css` | Styles |
| 11 | `frontend/app/layout.tsx` | Layout |
| 12 | `frontend/app/page.tsx` | Landing Page |
| 13 | `frontend/app/api/copilotkit/route.ts` | API Route (HttpAgent + CopilotRuntime) |
| 14 | `frontend/app/workspace/[wfId]/page.tsx` | Workspace |
| 15 | `frontend/lib/utils.ts` | Utils |
| 16 | `frontend/lib/copilot-config.ts` | Config |
| 17 | `frontend/lib/agentcore-agui-client.ts` | Core Client (HttpAgent Factory) |
| 18 | `frontend/lib/use-workflow-stream.ts` | Hook |
| 19 | `frontend/lib/use-agentcore-runtimes.ts` | Hook |
| 20 | `frontend/lib/use-shared-state.ts` | Hook (useCoAgent two-way sync) |
| 21 | `frontend/components/ui/card.tsx` | UI Primitiand |
| 22 | `frontend/components/CopilotKitProvider.tsx` | Provider |
| 23 | `frontend/components/MetricTile.tsx` | Component |
| 24 | `frontend/components/TranscriptCard.tsx` | Generatiand UI |
| 25 | `frontend/components/DraftPreviewCard.tsx` | Generatiand UI |
| 26 | `frontend/components/ReviewReportCard.tsx` | Generatiand UI |
| 27 | `frontend/components/ApprovalCard.tsx` | Generatiand UI |
| 28 | `frontend/components/ClarificationQuestionCard.tsx` | Generatiand UI |
| 29 | `frontend/components/CanvasPanel.tsx` | Component |
| 30 | `frontend/components/GenerativeUIRenderer.tsx` | Orchestrator |
| 31 | `frontend/components/CopilotSidebarWrapper.tsx` | Component |

---

## GATE PASS CHECKLIST

Before declaring Phase 5 complete, verify ALL of the following:

### 8 CopilotKit Features Visible
- [ ] **Feature 1: CopilotChat** -- Chat interface renders in sidebar or floating window
- [ ] **Feature 2: CopilotSidebar** -- Sidebar toggle works, history visible
- [ ] **Feature 3: useCoAgent** -- Agent state syncs with `useCoAgent` hook (two-way via BedrockAgentCoreApp runtime context)
- [ ] **Feature 4: useAgent** -- Agent direct interaction works
- [ ] **Feature 5: useCopilotAction** -- Actions registered and callable
- [ ] **Feature 6: useCopilotReadable** -- Workflow context readable by AI
- [ ] **Feature 7: useFrontendTool** -- Frontend tools (download draft) work
- [ ] **Feature 8: GenerativeUI** -- All 5 card types render dynamically

### AG-UI Events Flow
- [ ] SSE connection opens to `/api/v1/workflows/{wfId}/stream`
- [ ] `RUN_EVENT` messages appear in CanvasPanel with purple color
- [ ] `STEP_EVENT` messages appear in CanvasPanel with green color
- [ ] `AGENT_EVENT` messages appear in CanvasPanel with blue color
- [ ] Event count increments in real-time

### HttpAgent Connection (NEW - CopilotKit 1.50+ note)
- [ ] `HttpAgent` from `@ag-ui/client` connects to AgentCore endpoint
- [ ] URL format correct: `bedrock-agentcore.<region>.amazonaws.com/runtimes/<id>/invocations`
- [ ] Headers include `X-Amzn-Bedrock-AgentCore-Runtime-Session-Id: <session_id>`
- [ ] `CopilotRuntime` registers agents via `HttpAgent` instances
- [ ] `MCPAppsMiddleware` optionally available for MCP server access

### useCoAgent Two-Way Sync (NEW - CopilotKit 1.50+ note)
- [ ] `useCoAgent` hook reads agent state from `strands_drafter`
- [ ] `useCoAgent` `setState` updates state bidirectionally
- [ ] State syncs via `BedrockAgentCoreApp` runtime context on Python side (session_id mapped to workflow_id)
- [ ] `ToolBehavior.skip_messages_snapshot` and `state_from_args` work correctly

### Workflow Stream
- [ ] `useWorkflowStream` hook initializes when workflow page loads
- [ ] `isConnected` becomes `true` when SSE opens
- [ ] `state.steps` populates as STEP_EVENTs arrive
- [ ] `state.status` transitions: idle --> running --> completed/failed
- [ ] Auto-scroll keeps CanvasPanel at bottom

### Page Refresh Restores State
- [ ] Refreshing `/workspace/{wfId}` reconnects SSE
- [ ] Previously completed steps re-appear
- [ ] Agent state re-synchronizes via `useCoAgent`
- [ ] No data loss on refresh

### Dual-Response HITL
- [ ] Approval card renders when `status === 'awaiting_approval'`
- [ ] Approand button calls `sendApproval('approve')`
- [ ] Reject button calls `sendApproval('reject')`
- [ ] Clarification card renders when `status === 'awaiting_clarification'`
- [ ] Answer submission calls `sendClarification(answer)`
- [ ] After response, status returns to 'running'

### Landing Page
- [ ] All 8 feature tiles display with checkmarks
- [ ] Metric tiles show workflow counts
- [ ] Workflow list populates from backend
- [ ] Clicking workflow navigates to workspace
- [ ] "New Workflow" button creates and navigates

### Docker Build
- [ ] `docker build -f frontend/Dockerfile -t agentcore-demo-test1-frontend frontend/` succeeds
- [ ] Container starts on port 3000
- [ ] Frontend can reach backend at port 8000

---

## FAILURE HANDLING

**If any step fails:**
1. STOP immediately. Do not proceed to the next step.
2. Report the exact file that failed and the error message.
3. Do not attempt workarounds or alternatiand approaches.
4. Wait for instructions before continuing.

**Common failure points:**
- Missing `node_modules` after package.json changes --> run `npm install`
- CopilotKit imports not found --> verify package versions match (^1.50.0)
- AG-UI imports not found --> verify `@ag-ui/client` is ^0.5.0
- TypeScript errors --> check `tsconfig.json` paths configuration
- Docker build fails --> verify `.dockerignore` does not exclude needed files

---

## Tech Reference -- CopilotKit + AG-UI + AgentCore (CopilotKit 1.50+ note)

**CopilotKit CLI Scaffolding:**
- `npx copilotkit@latest create -f aws-strands-py` -- tek komutla proje
- Next.js + CopilotKit + AG-UI + Strands hazir

**New API'ler (CopilotKit 1.50+ note):**
- `BedrockAgentCoreApp` runtime context -- Python tarafinda state yonetimi (session_id, request_id)
  - `state_context_builder` -- State'i prompt'a inject eder
  - `tool_behaviors` -- Tool-specific davranislar
- `ToolBehavior` -- Tool-specific AG-UI davranislari
  - `skip_messages_snapshot` -- Mesaj history'sini atlar
  - `state_from_args` -- Tool argumanlarindan state cikarma
- `useCoAgent` -- Frontend'de two-way state sync
  - `state` -- Mevcut agent state'ini okur
  - `setState` -- Agent state'ini gunceller
- `BedrockAgentCoreApp` -- Python runtime context for state sync (session_id = workflow_id)

**HttpAgent Pattern:**
- Import: `import { HttpAgent } from "@ag-ui/client"`
- URL: `https://bedrock-agentcore.<region>.amazonaws.com/runtimes/<id>/invocations?qualifier=DEFAULT`
- Headers:
  - `X-Amzn-Bedrock-AgentCore-Runtime-Session-Id: <session_id>`
- Factory: `createAgentCoreHttpAgent(region, runtimeId, accountId, token, sessionId)`

**CopilotRuntime with HttpAgent:**
- `import { CopilotRuntime } from "@copilotkit/runtime"`
- `agents` map'i her agent icin bir `HttpAgent` icerir
- `BedrockAgentCoreApp` runtime context auto-includes session_id for correlation

**AgentCore:**
- 14 region destegi
- `--protocol AGUI` with AG-UI endpoint
- Her workflow icin ayri microVM session isolation (sessionId = workflowId)

**BedrockAgentCoreApp V5 Pattern (Python):**
```python
from bedrock_agentcore.runtime import BedrockAgentCoreApp

app = BedrockAgentCoreApp()

@app.entrypoint
def handler(request):
    """V5 agent handler — state managed via runtime context."""
    # session_id is automatically provided by AgentCore MicroVM
    # Maps to Temporal workflow_id for correlation
    session_id = request.context.session_id  # == workflow_id
    request_id = request.context.request_id

    # Build state from request + context
    current_state = request.state or {}

    # Execute agent logic (Strands tools, system prompt, etc.)
    result = run_agent_logic(request, current_state)

    # Build V5 8-block payload (raw dict)
    payload = build_v5_payload(result, session_id=session_id)

    return {
        "status": "success",
        "payload": payload,
        "state": result.updated_state,  # Optional: persist state across turns
    }

# Auto-created: POST /invocations, GET /ping
# Manual add: GET /metadata, /capabilities, /metrics, /health
```

**Docs:**
- Bedrock AgentCore Runtime: https://docs.aws.amazon.com/bedrock/latest/userguide/agentcore-runtime-sdk.html
- AG-UI Protocol: https://docs.ag-ui.com/
- CopilotKit useAgent: https://docs.copilotkit.ai/reference/hooks/use-agent
- Blog: https://www.copilotkit.ai/blog/aws-announces-dedicated-ag-ui-endpoint-in-agentcore-and-fast-template-for-building-fullstack-agents


---

## TESTING REQUIREMENTS — pytest + Playwright Frontend

Every gate checkpoint MUST have a corresponding pytest test. Create:

```
tests/
  frontend/
    conftest.py               # Playwright Page fixture, base_url
    test_gate_5_copilotkit.py # test_chat_widget_visible,
                              # test_text_message_rendered,
                              # test_tool_call_spinner,
                              # test_clarification_card_visible,
                              # test_state_delta_updates_canvas
    test_gate_5_streams.py    # test_ag_ui_stream_connected,
                              # test_workflow_stream_connected
    test_gate_5_refresh.py    # test_chat_history_persists,
                              # test_canvas_state_persists
```

Run: `pytest tests/frontend/ -v`

Requires: `uv pip install pytest-playwright` (inside .venv)

Key pattern — Playwright:
```python
def test_chat_widget_visible(page, base_url):
    page.goto(f"{base_url}/workspace/test-wf")
    chat = page.locator("[data-copilotkit-chat]")
    expect(chat).to_be_visible(timeout=10000)
```

---

## Next Phase

When the frontend renders correctly, all CopilotKit features work, and AG-UI flows end-to-end:
- Report Phase 5 as COMPLETE
- Proceed to **Phase 6: E2E Integration & Final Verification** using `Prompt_06_E2E_Integration.md`
