# Specification Package (`/spec`)

This directory contains the **complete implementation‑ready specification** for the Agent Dashboard MVP.  
It defines every contract required for backend, frontend, agent runtime, and workflow execution.

These files are designed for:
- IDE agents
- LLM coding assistants
- Backend implementers
- Frontend developers
- Workflow engine developers

The `/spec` folder is the **single source of truth** for how the system behaves at runtime.

---

# 📦 Contents

## 1. `backend_api_contract.md`
> ⚠️ **Superseded (D2/D3/D7, bannered 2026-09-05)** — canonical API → `packages/contracts`.
Describes the superseded monolith backend's HTTP/WebSocket endpoints:
- `/sessions/create`
- `/sessions/{id}/run`
- `/sessions/{id}/stop`
- `/sessions/{id}/events`
- `/agents/list`
- `/agents/{id}/schema`

Includes request/response shapes and transport semantics.

Use this file when implementing the **backend server** or connecting the **frontend API client**.

---

## 2. `event_payloads.json`
Canonical JSON definitions for all event types emitted by the backend:
- `session.created`
- `session.started`
- `session.stateChanged`
- `output.partial`
- `output.final`
- `log.append`
- `session.error`
- `session.completed`

These payloads map directly to the **RxJS event bus** on the frontend.

Use this file when implementing:
- backend event streaming
- frontend event parsing
- workflow node event subscriptions

---

## 3. `agent_schema_format.json`
Defines the JSON Schema format for agent input, output, and configuration.

Used by:
- backend agent registry
- frontend dynamic input form generator
- workflow node inspector

This ensures all agents follow a consistent, machine‑readable schema.

---

## 4. `session_state_machine.json`
Formal definition of the session lifecycle:
- states: `idle`, `running`, `streaming`, `waiting`, `completed`, `error`
- allowed transitions
- lifecycle rules

This file is used by:
- backend session manager
- frontend state indicator
- workflow engine (node execution logic)

---

## 5. `frontend_architecture.md`
Defines the frontend runtime architecture:
- RxJS event bus
- session‑bound components
- tab model
- folder structure
- runtime glue between SSE and RxJS (D2 — WebSocket superseded)

Use this file when implementing the **React + RxJS frontend**.

---

## 6. `workflow_engine_interface.md`
Defines the workflow engine’s core interfaces:
- `WorkflowNode`
- `WorkflowEdge`
- `WorkflowEngine`
- RxJS operator mapping for edges

This file prepares the MVP for the future workflow builder without requiring refactors.

---

## 7. `agent_runtime_contract.md`
Defines how agents execute at runtime:
- `astreamEvents()` for streaming logs/output
- `ainvoke()` for single‑shot execution
- `abort()` for cancellation
- canonical `AgentEvent` types
- required lifecycle semantics

This file is used by backend developers implementing:
- agent runners
- model wrappers
- tool executors
- workflow node execution

---

# 🧭 How to Use This Folder

### Frontend developers
Start with:
- `frontend_architecture.md`
- `event_payloads.json`
- `session_state_machine.json`

Then implement:
- SSE client (D2 — WebSocket superseded)
- RxJS event bus
- session‑bound components
- tab model

### Backend developers
Start with:
- `backend_api_contract.md`
- `agent_runtime_contract.md`

Then implement:
- session manager
- agent runtime
- event streaming
- lifecycle transitions

### Workflow engine developers
Start with:
- `workflow_engine_interface.md`
- `session_state_machine.json`
- `event_payloads.json`

Then implement:
- reactive node execution
- edge operator mapping
- workflow orchestration

---

# 🚀 Start Here

If you are an IDE agent or LLM building the system:

1. Read `backend_api_contract.md`  
2. Read `agent_runtime_contract.md`  
3. Read `frontend_architecture.md`  
4. Use `event_payloads.json` + `session_state_machine.json` as strict runtime contracts  
5. Use `workflow_engine_interface.md` for future workflow support  

This folder contains everything required to begin implementation immediately.

