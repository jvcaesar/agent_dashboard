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
- `waiting`: `AgentApprovalPanel` renders prompt + options (D11); indicator 🟡
- `completed`: final output, input re-enabled, “Run Again”, indicator 🟣
- `error`: error banner, logs show error, “Restart Session”, indicator 🔴
```