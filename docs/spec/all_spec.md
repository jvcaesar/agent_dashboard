> ⚠️ **GENERATED EXPORT (pre-decision snapshot, 2026-09-05)** — This file concatenates the spec docs and **includes superseded content without banners** (notably the sessions-monolith `backend_api_contract`). The **individual files in `docs/spec/` are authoritative** — several now carry decision banners/notes (D2–D8, D11). Prefer them, or `packages/contracts` once it exists.

This is meant for **direct implementation** by an IDE agent or LLM — meaning:

- JSON schemas  
- API contracts  
- event payload definitions  
- state machine definitions  
- workflow engine interface  
- frontend architecture spec  

You can drop these into a `/spec` folder in your repo or feed them directly to an implementation agent.

---

# 📦 ** Code‑Ready Spec Package**

This package contains **6 files**, each designed for implementation.

---

# 1. **`backend_api_contract.md`**

```markdown
# Backend API Contract

## Base URL
`/api`

---

## POST /sessions/create
Create a new session.

### Request
```json
{
  "agentId": "string",
  "config": {},
  "inputDraft": {}
}
```

### Response
```json
{
  "sessionId": "string",
  "state": "idle"
}
```

---

## POST /sessions/{sessionId}/run
Start execution.

### Request
```json
{
  "input": {},
  "config": {}
}
```

### Response
```json
{
  "status": "started"
}
```

---

## POST /sessions/{sessionId}/stop
Stop execution.

### Response
```json
{
  "status": "stopped"
}
```

---

## GET /sessions/{sessionId}/events (WebSocket or SSE)
Stream lifecycle events, logs, partial outputs, final outputs.

### Event Format
See `event_payloads.json`.

---

## GET /agents/list
List available agents.

### Response
```json
[
  {
    "id": "string",
    "name": "string",
    "description": "string"
  }
]
```

---

## GET /agents/{agentId}/schema
Retrieve agent input/output schema.

### Response
```json
{
  "inputSchema": {},
  "outputSchema": {},
  "configSchema": {}
}
```
```

---

# 2. **`event_payloads.json`**

```json
{
  "session.created": {
    "sessionId": "string",
    "type": "session.created",
    "state": "idle",
    "timestamp": "number"
  },
  "session.started": {
    "sessionId": "string",
    "type": "session.started",
    "state": "running",
    "input": {},
    "config": {},
    "timestamp": "number"
  },
  "session.stateChanged": {
    "sessionId": "string",
    "type": "session.stateChanged",
    "from": "string",
    "to": "string",
    "timestamp": "number"
  },
  "output.partial": {
    "sessionId": "string",
    "type": "output.partial",
    "chunk": {},
    "timestamp": "number"
  },
  "output.final": {
    "sessionId": "string",
    "type": "output.final",
    "output": {},
    "timestamp": "number"
  },
  "log.append": {
    "sessionId": "string",
    "type": "log.append",
    "severity": "info | warn | error",
    "message": "string",
    "timestamp": "number"
  },
  "session.error": {
    "sessionId": "string",
    "type": "session.error",
    "error": {
      "message": "string",
      "stack": "string",
      "code": "string"
    },
    "timestamp": "number"
  },
  "session.completed": {
    "sessionId": "string",
    "type": "session.completed",
    "state": "completed",
    "durationMs": "number",
    "tokensUsed": "number",
    "timestamp": "number"
  }
}
```

---

# 3. **`agent_schema_format.json`**

```json
{
  "agentId": "string",
  "name": "string",
  "description": "string",
  "inputSchema": {
    "type": "object",
    "properties": {
      "fieldName": {
        "type": "string | number | boolean | object | array",
        "title": "string",
        "description": "string",
        "default": "any",
        "enum": ["optional"]
      }
    },
    "required": ["fieldName"]
  },
  "outputSchema": {
    "type": "object",
    "properties": {}
  },
  "configSchema": {
    "type": "object",
    "properties": {
      "temperature": {
        "type": "number",
        "default": 0.7
      }
    }
  }
}
```

---

# 4. **`session_state_machine.json`**

```json
{
  "states": [
    "idle",
    "running",
    "streaming",
    "waiting",
    "completed",
    "error"
  ],
  "transitions": {
    "idle": ["running", "error"],
    "running": ["streaming", "waiting", "completed", "error"],
    "streaming": ["completed", "error"],
    "waiting": ["running", "error"],
    "completed": ["idle", "running"],
    "error": ["idle", "running"]
  }
}
```

---

# 5. **`frontend_architecture.md`**

```markdown
# Frontend Architecture

## Framework
React + RxJS

---

## Event Bus
- Global dispatcher: Subject<Event>
- Session streams: ReplaySubject<Event>
- Operators:
  - filter
  - scan
  - distinctUntilChanged
  - shareReplay
  - switchMap
  - takeUntil
  - merge
  - combineLatest

---

## Component Architecture
Components are session-bound:
- AgentInputForm
- AgentOutputViewer
- AgentLogViewer
- AgentControlPanel
- AgentStateIndicator
- AgentMetadataPanel

Each receives:
```ts
{ sessionId: string }
```

---

## Tab Model
```ts
type Tab = {
  id: string;
  title: string;
  sessionId: string;
  uiState: {
    activePanel: string;
    scrollPositions: Record<string, number>;
    expandedSections: Record<string, boolean>;
  };
};
```

---

## Folder Structure

```text
/src
  /components
  /state/eventBus
  /workflow
  /api
  /routes
```
```

---

# 6. **`workflow_engine_interface.md`**

```markdown
# Workflow Engine Interface

## Overview
Workflow engine is a reactive graph built on RxJS.

---

## Node Model

```ts
type WorkflowNode = {
  id: string;
  type: "agent" | "transform" | "trigger" | "branch";
  sessionId: string | null;
  config: Record<string, any>;
  position: { x: number; y: number };
  inputs: Port[];
  outputs: Port[];
};
```

---

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

---

## Engine API

```ts
interface WorkflowEngine {
  loadWorkflow(definition: WorkflowDefinition): void;
  start(): void;
  stop(): void;
  subscribe(nodeId: string, callback: (event: Event) => void): void;
}
```

---

## Execution Model

- Nodes subscribe to `sessionEvents(sessionId)`
- Edges transform streams using RxJS operators
- Workflow is a visual RxJS pipeline
```
