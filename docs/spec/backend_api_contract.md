
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
