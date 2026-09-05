# Seed Agents Specification

> **Status: ✅ confirmed (2026-09-05, decision D16 in [`scope_decisions.md`](../analysis/scope_decisions.md)).**
> Resolves gap **G7** (no concrete seed agents — only `mockAgentRuntime` existed).
> Companion contracts: [`agent_runtime_contract.md`](./agent_runtime_contract.md) (runtime interface + event-ordering guarantees), [`agent_schema_format.json`](./agent_schema_format.json) (D8 schema format), [`event_payloads.json`](./event_payloads.json) (D6 canonical event model). This document is the **acceptance reference for Phase 3** of [`implementation_plan.md`](../analysis/implementation_plan.md).

---

## Shared requirements (both agents)

- Implement the `AgentRuntime` interface: `streamEvents` / `invoke` / `abort` / `getState`.
- Emit only canonical events (D6): `{ type, sessionId, taskId?, timestamp, payload }`.
- Validate input against their JSON Schema (draft-07 + `x-ui-*`, D8) **before** execution; invalid input → `session.error` with code `INVALID_INPUT`, no execution.
- Support `abort` → emits `session.error` with code `CANCELLED` (partial output preserved).
- Schema files live in `packages/agent-schemas/schemas/` (`echo.json`, `calculator.json`), each validated against the format itself; both agents register in the harness registry by `agentId` — proving the registration path future agents (incl. Python) will use.
- Every run completes in < 2 s with default config (keeps the test suite fast).

---

## 1. echo — default smoke-test agent

| Field | Value |
|---|---|
| `agentId` | `echo` |
| Display name | Echo |
| Purpose | Proves the full loop (form → run → streaming → complete); default target of `scripts/smoke.mjs` |

**Input schema**: `message` — `string`, required, `x-ui-widget: textarea`, `x-ui-label: "Message to echo"`.

**Config schema**: `chunkDelayMs` — `integer`, default `100`; `chunkSize` — `integer`, default `16`.

**Behavior (event sequence)**:
1. `session.started` (state `running`)
2. N × (`log.append` progress + `output.partial` chunk) — input split into `chunkSize` chunks, `chunkDelayMs` apart
3. `output.final` — the full message
4. `session.completed` — `tokensUsed` (approximate), `durationMs`

**Acceptance criteria (Phase 3.2 tests)**: exact ordered sequence above; chunk count = `ceil(len/chunkSize)`; abort mid-stream → `CANCELLED`; missing `message` → `INVALID_INPUT` before execution; full run < 2 s.

---

## 2. calculator — computation + HITL demo

| Field | Value |
|---|---|
| `agentId` | `calculator` |
| Display name | Calculator |
| Purpose | Proves step-logging, input validation, and the D11 approval flow |

**Input schema**: `expression` — `string`, required, pattern `^[0-9+\-*/(). ]+$` (safe arithmetic only), `x-ui-label: "Expression"`.

**Config schema**: `approvalRequired` — `boolean`, default `false` (D11: when `true`, the run pauses for human approval before evaluating).

**Behavior (event sequence)**:
1. `session.started` (state `running`)
2. `log.append` steps: `tokenize` → `parse` → `evaluate`
3. If `approvalRequired: true`: state → `waiting`, emit `approval.requested` (prompt + options `["approve", "reject"]`); on approve → `task.resume` → `approval.resumed` (choice) → state `running`, continue; on reject → `session.error` code `REJECTED_BY_USER`
4. `output.partial` (steps so far) → `output.final` (numeric result)
5. `session.completed` — `durationMs`

**Acceptance criteria**: `"2 + 3 * 4"` → final `14`; `waiting` state occurs only when `approvalRequired` (waiting→running→completed legal per the state machine); division by zero → `session.error` code `EVALUATION_ERROR`; invalid characters → `INVALID_INPUT`; the approval path is exercised end-to-end by the Phase 5.2 smoke extension.

---

## Post-MVP note

An **LLM-backed agent** is intentionally excluded from the MVP (requires an API key; nondeterministic tests). It would implement the same `AgentRuntime` interface — purely additive, no rework (D16 revisit-trigger).