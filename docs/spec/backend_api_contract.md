
> ⚠️ **SUPERSEDED (2026-09-05)** — This contract describes the earlier sessions-monolith backend (Express + WebSocket). The MVP implements the **Gateway + Agent Harness + Redis (SSE)** architecture instead — see `docs/scaffolds/Gateway_Scaffold.md`, `docs/scaffolds/Frontend_SSE_Client_Scaffold.md`, and decisions **D2/D3/D7** in `docs/analysis/scope_decisions.md`. The sessions-centric endpoint set (create/run/stop/approve/events) and the `/agents/*` routes below remain valid in spirit; the canonical contract now lives in `packages/contracts` (`api.ts`).

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
