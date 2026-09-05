
---

> ⚠️ **GENERATED EXPORT (pre-decision snapshot, 2026-09-05)** — This file concatenates the architecture docs and **includes pre-decision content** (e.g., WebSocket transport options). The **individual files in `docs/architecture/` are authoritative** — several now carry decision notes/banners (D2, D11–D13).

### `MVP_Overview.md`

```markdown
# MVP Overview

## Vision

A browser-style agent dashboard where each tab represents an agent session:
- Simple, friendly UI (like a browser, not an IDE)
- RxJS-powered event bus for streaming, logs, and state
- Future-ready for a workflow builder that reuses the same sessions and components

## Core Concepts

- **Agent**: A configurable unit of work with input schema, output schema, and capabilities.
- **Session**: A single run context for an agent, with lifecycle, logs, and outputs.
- **Tab**: A UI view bound to a session.
- **Event Bus**: RxJS-based pub/sub system distributing session events.
- **Workflow Node (future)**: Another view bound to a session, rendered on a canvas.

## MVP Scope

- Browser-style UI with:
  - Tab bar
  - Agent list sidebar
  - Session workspace (input, output, logs, controls)
- RxJS event bus for:
  - Session lifecycle events
  - Logs
  - Partial and final outputs
- Session cloning and tab duplication
- Workflow builder preparation (models, hooks, folder structure)
```

---

### `Session_Lifecycle.md`

```markdown
# Session Lifecycle

## States

- `idle`
- `running`
- `streaming`
- `waiting`
- `completed`
- `error`

## State Machine

| From →     | idle | running | streaming | waiting | completed | error |
|-----------|------|---------|-----------|---------|-----------|--------|
| **idle**      | —    | run     | —         | —       | —         | error  |
| **running**   | —    | —       | partial   | waiting | completed | error  |
| **streaming** | —    | —       | —         | —       | completed | error  |
| **waiting**   | —    | resume  | —         | —       | —         | error  |
| **completed** | restart | rerun | —         | —       | —         | —      |
| **error**     | restart | rerun | —         | —       | —         | —      |

## Event Payloads (Backend → Frontend)

```json
{
  "session.created": {
    "sessionId": "abc123",
    "type": "session.created",
    "state": "idle",
    "timestamp": 1234567890
  },
  "session.started": {
    "sessionId": "abc123",
    "type": "session.started",
    "state": "running",
    "input": {},
    "config": {},
    "timestamp": 1234567891
  },
  "session.stateChanged": {
    "sessionId": "abc123",
    "type": "session.stateChanged",
    "from": "running",
    "to": "waiting",
    "timestamp": 1234567892
  },
  "output.partial": {
    "sessionId": "abc123",
    "type": "output.partial",
    "chunk": "partial text or structured data",
    "timestamp": 1234567893
  },
  "output.final": {
    "sessionId": "abc123",
    "type": "output.final",
    "output": {},
    "timestamp": 1234567894
  },
  "log.append": {
    "sessionId": "abc123",
    "type": "log.append",
    "severity": "info",
    "message": "Agent did something",
    "timestamp": 1234567895
  },
  "session.error": {
    "sessionId": "abc123",
    "type": "session.error",
    "error": {
      "message": "...",
      "stack": "...",
      "code": "AGENT_FAILURE"
    },
    "timestamp": 1234567896
  },
  "session.completed": {
    "sessionId": "abc123",
    "type": "session.completed",
    "state": "completed",
    "durationMs": 1234,
    "tokensUsed": 567,
    "timestamp": 1234567897
  }
}
```

## UI Mapping

- `idle`: input enabled, logs/output empty, indicator ⚪
- `running`: input disabled, logs streaming, output visible, indicator 🟢
- `streaming`: partial output streaming, indicator 🔵
- `waiting`: waiting banner/spinner, indicator 🟡
- `completed`: final output, input re-enabled, “Run Again”, indicator 🟣
- `error`: error banner, logs show error, “Restart Session”, indicator 🔴
```

---

### `EventBus_RxJS.md`

```markdown
# Event Bus (RxJS)

## Goals

- Session-scoped event streams
- Multicasting to tabs and workflow nodes
- Replay for late subscribers
- Clean subscription management for tab switching and closing

## Core API

```ts
type Event = {
  sessionId: string;
  type: string;
  payload?: any;
  timestamp: number;
};

interface EventBus {
  publish(event: Event): void;
  sessionEvents(sessionId: string): Observable<Event>;
  globalEvents(): Observable<Event>;
}
```

## Implementation Sketch

- Global dispatcher: `Subject<Event>`
- Session streams: `ReplaySubject<Event>` per `sessionId`
- `sessionEvents(sessionId)` returns `sessionSubjects[sessionId].asObservable().pipe(shareReplay({ bufferSize: N }))`

## Key RxJS Patterns

- `filter()` for event type selection
- `scan()` for accumulating logs/output
- `distinctUntilChanged()` for state indicators
- `shareReplay()` for multicasting
- `switchMap()` for tab switching
- `takeUntil()` for cleanup on tab close
- `merge()` / `combineLatest()` for workflow edges

## Transport

- WebSocket or SSE from backend
- Each backend message mapped to `Event` and fed into `publish()`
```

---

### `Component_Architecture.md`

```markdown
# Component Architecture

## Principle

Components are **session-bound** and **reactive**:
- Each major component receives `sessionId`
- Subscribes to `sessionEvents(sessionId)`
- Filters relevant events
- Updates local UI state via React hooks

## Core Components

- `AgentHeader`
- `AgentInputForm`
- `AgentOutputViewer`
- `AgentLogViewer`
- `AgentControlPanel`
- `AgentStateIndicator`
- `AgentMetadataPanel`

## Example: Log Viewer

```ts
function AgentLogViewer({ sessionId }: { sessionId: string }) {
  const [logs, setLogs] = useState<LogEntry[]>([]);

  useEffect(() => {
    const sub = sessionEvents(sessionId)
      .pipe(
        filter(e => e.type === "log.append"),
        scan((acc, e) => [...acc, e.payload], [])
      )
      .subscribe(setLogs);

    return () => sub.unsubscribe();
  }, [sessionId]);

  // render logs...
}
```

## Reuse in Workflow Builder

- Same components used inside workflow nodes
- Nodes pass `sessionId` just like tabs
- No component changes needed when canvas is added
```

---

### `Tab_Model.md`

```markdown
# Tab Model and Interaction Patterns

## Tab Object

```ts
type Tab = {
  id: string;          // usually equals sessionId
  title: string;       // agent name or custom label
  sessionId: string;
  uiState: {
    activePanel: "input" | "output" | "logs";
    scrollPositions: Record<string, number>;
    expandedSections: Record<string, boolean>;
  };
};
```

## Core Behaviors

- **Creation**: new session + new tab, input focused, state `idle`
- **Switching**:
  - active tab changes
  - components subscribe to new `sessionId` via `switchMap`
- **State Indicators**:
  - driven by `session.stateChanged` and lifecycle
  - icons: ⚪ 🟢 🔵 🟡 🔴 🟣
- **Duplication**:
  - snapshot input/config/uiState
  - create new session in `idle`
  - bind new tab to new `sessionId`
- **Closing**:
  - unsubscribe via `takeUntil`
  - optionally destroy session

## UX Notes

- Tab bar: title + state icon + close button + “+” for new tab
- Keyboard shortcuts (optional): Ctrl+Tab, Ctrl+Shift+Tab
```

---

### `Workflow_Preparation.md`

```markdown
# Workflow Builder Preparation

## Node Model

```ts
type WorkflowNodeType = "agent" | "transform" | "trigger" | "branch";

type WorkflowNode = {
  id: string;
  type: WorkflowNodeType;
  sessionId: string | null;
  config: Record<string, any>;
  position: { x: number; y: number };
  inputs: Port[];
  outputs: Port[];
};
```

## Edge Model

```ts
type WorkflowEdge = {
  id: string;
  fromNodeId: string;
  fromPort: string;
  toNodeId: string;
  toPort: string;
  operator: "map" | "merge" | "switchMap" | "combineLatest" | "filter" | null;
};
```

## Event Bus Hooks

- Nodes subscribe to `sessionEvents(sessionId)`
- Edges implemented via RxJS operators:
  - sequential: `switchMap`
  - parallel: `merge`
  - conditional: `filter`
  - multi-input sync: `combineLatest`
  - transform: `map`

## Folder Structure (Stub)

```text
/workflow
  /canvas
    WorkflowCanvas.tsx
  /nodes
    AgentNode.tsx
    TransformNode.tsx
    BranchNode.tsx
  /edges
    WorkflowEdge.ts
  /models
    WorkflowNode.ts
    WorkflowEdge.ts
  /services
    workflowEngine.ts
```

## Node Inspector Reuse

- Right sidebar from tabs reused as node inspector
- Same components: `AgentControlPanel`, `AgentMetadataPanel`, etc.
- Same `sessionId` binding
```

---

### `Wireframes.md`

```markdown
# Wireframes (Text Spec)

## Main Layout

- **Top bar**: tab bar + global actions
- **Left sidebar**: agent list
- **Main pane**: active session workspace
- **Right sidebar**: agent control panel / inspector
- **Bottom pane**: logs

## Tab Bar

- Tab title: agent name
- State indicator icon
- Close button
- “+” for new session

## Session Workspace

- Header: agent name + session ID
- Input panel: generated from agent input schema
- Output panel: streaming + final result
- Metadata: tokens, duration, cost
- Controls: Run, Stop, Restart

## Logs Panel

- Bottom console-style area
- Real-time streaming
- Severity colors
- Search/filter
- Autoscroll toggle

## Workflow Placeholder

- Separate screen or mode:
  - “Workflow Builder Coming Soon”
  - Button: “Use tab interface”
  - Empty canvas area
  - Disabled node palette
```

---

### `Folder_Structure.md`

```markdown
# Suggested Folder Structure

```text
/src
  /architecture   // optional, for these docs
  /components
    /tabs
    /session
    /workflow
  /state
    eventBus/
  /workflow
    canvas/
    nodes/
    edges/
    models/
    services/
  /api
    backendClient.ts
  /pages or /routes
    Dashboard.tsx
```

---

### `Cloning_Model.md`

```markdown
# Session Cloning Model

## Snapshot

Clone captures **static** session data:

```ts
type SessionSnapshot = {
  agentId: string;
  inputDraft: Record<string, any>;
  config: Record<string, any>;
  uiState: {
    activePanel: string;
    scrollPositions: Record<string, number>;
    expandedSections: Record<string, boolean>;
  };
  metadata: Record<string, any>;
};
```

## New Session

```ts
const newSession: Session = {
  id: uuid(),
  agentId: snapshot.agentId,
  state: "idle",
  inputHistory: [],
  outputHistory: [],
  logs: [],
  metadata: snapshot.metadata
};
```

- No logs, output, errors, or runtime state are copied.
- New `ReplaySubject<Event>` for `newSession.id`.

## New Tab

```ts
const newTab: Tab = {
  id: newSession.id,
  title: originalTab.title + " (clone)",
  sessionId: newSession.id,
  uiState: snapshot.uiState
};
```

## UX Rules

- Clone starts in `idle`
- Input + config preserved
- Logs/output cleared
- Optional “Clone and Run” vs “Clone Only” (default: “Clone Only”)
```

---

If you want, next step could be a **`/spec` package** for backend API + schemas, but this architecture set is already enough for an IDE agent or LLM to start implementing the front end and wiring it to a backend.