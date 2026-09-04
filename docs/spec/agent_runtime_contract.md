### `agent_runtime_contract.md`

```markdown
# Agent Runtime Contract

## Purpose

Define how an agent implementation interacts with the session lifecycle, event bus, and backend:
- How agents are invoked
- How they stream output and logs
- How they signal state changes, completion, and errors
- How they support cancellation and retries

This contract is **backend-facing**: the backend calls the agent runtime; the agent runtime emits events that map directly to the session lifecycle.

---

## Core Concepts

- **AgentRuntime**: implementation that can run an agent once or as a stream.
- **AgentEvent**: canonical event type emitted by the runtime.
- **Session**: identified by `sessionId`, mapped 1:1 to a tab or workflow node.
- **Thread/Run IDs (optional)**: allow sub-streams or multi-step runs inside a session.

---

## AgentRuntime Interface (Backend → Agent)

```ts
export type AgentInput = Record<string, any>;
export type AgentConfig = Record<string, any>;

export type AgentRuntimeContext = {
  sessionId: string;
  runId?: string;
  config: AgentConfig;
};

export interface AgentRuntime {
  /**
   * Stream events for a given input.
   * Used for partial output, logs, and lifecycle events.
   */
  astreamEvents(
    input: AgentInput,
    context: AgentRuntimeContext
  ): AsyncIterable<AgentEvent>;

  /**
   * Single-shot invocation.
   * Returns a final event (usually OUTPUT_FINAL or SESSION_COMPLETED).
   */
  ainvoke(
    input: AgentInput,
    context: AgentRuntimeContext
  ): Promise<AgentEvent>;

  /**
   * Get current runtime state for a session/run.
   * Optional, used for debugging or workflow introspection.
   */
  getState(
    context: AgentRuntimeContext
  ): Promise<Record<string, any> | null>;

  /**
   * Request cancellation of a running job.
   * Must attempt to stop work and emit a SESSION_ERROR or SESSION_COMPLETED
   * depending on semantics.
   */
  abort(
    context: AgentRuntimeContext
  ): Promise<boolean>;
}
```

---

## Canonical AgentEvent Types (Agent → Backend)

All events emitted by the runtime must conform to these types.
They map directly to the session lifecycle and event payloads.

```ts
export type AgentEventType =
  | "SESSION_STARTED"
  | "SESSION_STATE_CHANGED"
  | "OUTPUT_PARTIAL"
  | "OUTPUT_FINAL"
  | "LOG_APPEND"
  | "SESSION_ERROR"
  | "SESSION_COMPLETED"
  | "USAGE"
  | "STATUS";

export interface BaseAgentEvent {
  sessionId: string;
  runId?: string;
  type: AgentEventType;
  timestamp: number;
}

export interface SessionStartedEvent extends BaseAgentEvent {
  type: "SESSION_STARTED";
  input: AgentInput;
  config: AgentConfig;
}

export interface SessionStateChangedEvent extends BaseAgentEvent {
  type: "SESSION_STATE_CHANGED";
  from: string; // previous state
  to: string;   // new state
}

export interface OutputPartialEvent extends BaseAgentEvent {
  type: "OUTPUT_PARTIAL";
  chunk: any; // text or structured data
}

export interface OutputFinalEvent extends BaseAgentEvent {
  type: "OUTPUT_FINAL";
  output: any; // final structured result
}

export interface LogAppendEvent extends BaseAgentEvent {
  type: "LOG_APPEND";
  severity: "info" | "warn" | "error";
  message: string;
}

export interface SessionErrorEvent extends BaseAgentEvent {
  type: "SESSION_ERROR";
  error: {
    message: string;
    code?: string;
    stack?: string;
  };
}

export interface SessionCompletedEvent extends BaseAgentEvent {
  type: "SESSION_COMPLETED";
  durationMs: number;
  tokensUsed?: number;
}

export interface UsageEvent extends BaseAgentEvent {
  type: "USAGE";
  inputTokens: number;
  outputTokens: number;
  estimatedCost?: number;
  provider?: string;
  model?: string;
}

export interface StatusEvent extends BaseAgentEvent {
  type: "STATUS";
  status: "thinking" | "idle" | "waiting" | "running";
}

export type AgentEvent =
  | SessionStartedEvent
  | SessionStateChangedEvent
  | OutputPartialEvent
  | OutputFinalEvent
  | LogAppendEvent
  | SessionErrorEvent
  | SessionCompletedEvent
  | UsageEvent
  | StatusEvent;
```

---

## Required Lifecycle Semantics

### On `astreamEvents(input, context)`

The runtime **must**:

1. Emit `SESSION_STARTED` as the first event.
2. Emit `SESSION_STATE_CHANGED` from `idle` → `running`.
3. While running:
   - Emit `LOG_APPEND` for internal progress.
   - Emit `OUTPUT_PARTIAL` for streaming output (if supported).
   - Optionally emit `STATUS` and `USAGE`.
4. On success:
   - Emit `OUTPUT_FINAL`.
   - Emit `SESSION_STATE_CHANGED` to `completed`.
   - Emit `SESSION_COMPLETED`.
5. On error:
   - Emit `SESSION_ERROR`.
   - Emit `SESSION_STATE_CHANGED` to `error`.

The backend maps these events to the session lifecycle state machine.

---

### On `ainvoke(input, context)`

- Must perform the same work as `astreamEvents`, but:
  - No partial events are required.
  - Returns a single `OUTPUT_FINAL` or `SESSION_ERROR` (and may include `SESSION_COMPLETED` semantics in payload).

---

## Cancellation Semantics (`abort`)

- When `abort(context)` is called:
  - Runtime must attempt to stop work.
  - If successful:
    - Emit `SESSION_STATE_CHANGED` to `error` or a dedicated `cancelled` state if you extend the machine.
    - Emit `SESSION_ERROR` with `code: "CANCELLED"` or similar.
  - Return `true` if cancellation was accepted, `false` otherwise.

---

## Logging Requirements

- All significant internal steps should emit `LOG_APPEND`.
- Errors must emit both:
  - `LOG_APPEND` with `severity: "error"`
  - `SESSION_ERROR` with structured error payload.

---

## Streaming Requirements

- `OUTPUT_PARTIAL` should be used for:
  - incremental text
  - incremental structured data
- `OUTPUT_FINAL` must represent the complete result.
- Partial events must not contradict the final output; they are a prefix or progressive view.

---

## Config Handling

- `context.config` must be honored by the runtime:
  - model selection
  - temperature / sampling parameters
  - tool usage flags
  - timeouts, etc.
- Invalid config should result in:
  - `SESSION_ERROR` with `code: "INVALID_CONFIG"`.

---

## Input Handling

- `input` is a structured object matching the agent’s `inputSchema`.
- Validation errors must:
  - not start execution
  - emit `SESSION_ERROR` with `code: "INVALID_INPUT"`.

---

## Contract Guarantees

An implementation of `AgentRuntime` **must guarantee**:

- Events for a given `sessionId` are ordered by `timestamp`.
- `SESSION_STARTED` is emitted before any output/log events.
- Either `SESSION_COMPLETED` or `SESSION_ERROR` is eventually emitted for each run.
- `OUTPUT_FINAL` is emitted at most once per run.
- `abort()` attempts to stop work and does not emit further non-error events after cancellation.

---
```