# Scope & Architecture Decisions (ADR-lite)

> Companion: [`project_analysis.md`](./project_analysis.md) (gaps G1–G12), [`implementation_plan.md`](./implementation_plan.md), [`repo_layout.md`](./repo_layout.md).
> Purpose: turn every documented inconsistency into an explicit decision so that `docs/` stays authoritative and implementation can start without ambiguity. Decisions marked **[CONFIRM]** need a quick human sign-off before Phase 2 begins; recommended answers are given.

---

## How to use this file

- Each decision: **Decision → Rationale → Impact → Revisit when**.
- Status values: `settled by docs`, `recommended`, `recommended — [CONFIRM]`.
- If a decision is rejected, edit **only this file** and propagate the change to `packages/contracts` when it exists — do not silently diverge in code.

---

## Decision summary

| ID | Topic | Decision (short) | Status |
|---|---|---|---|
| D1 | MVP boundary | Tabs + sessions + streaming + HITL (session-level); workflow engine/UI last | recommended — [CONFIRM] HITL |
| D2 | Session transport | SSE (not WebSocket) | settled by docs |
| D3 | Backend topology | Gateway + Agent Harness + Redis; monolith/WS docs superseded | settled by docs |
| D4 | Entities | `sessionId` first-class on every event; `taskId` optional correlation | recommended |
| D5 | Session state set | `idle, running, streaming, waiting, completed, error` everywhere; separate `task.status` enum | recommended |
| D6 | Canonical events | One event model (`type, sessionId, taskId?, timestamp, payload`); Redis & SSE are encodings of it | recommended |
| D7 | HTTP/SSE API | Sessions-centric endpoint set on Gateway; no `/tasks` routes in MVP | recommended |
| D8 | Agent schema format | JSON Schema (draft-07 subset) + UI metadata; single validator in `packages/agent-schemas` | recommended |
| D9 | Persistence | MVP: in-memory session/task store in Gateway; Mongo optional; required post-MVP | recommended — [CONFIRM] |
| D10 | Workflow | MVP ships interfaces + models + canvas placeholder only | recommended |
| D11 | Human-in-the-loop | Session-level approvals in MVP (waiting state, approve endpoint, prompt UI) | recommended — [CONFIRM] |
| D12 | Run semantics | Run / Rerun vs Restart(clone) pinned | recommended |
| D13 | Replay & hydration | In-tab `ReplaySubject` suffices; page reload = fresh dashboard | recommended |
| D14 | Ops & env | `/health`; `.env` + defaults; minimal logs; no auth in MVP | recommended |
| D15 | Tooling | npm workspaces monorepo; Vite+React+RxJS+Vitest; Fastify+ioredis; compose for Redis (+ Mongo optional) | recommended |

<br/>

**Confirmations requested** (deciding these unlocks Phase 2; everything else proceeds in parallel):
- **D1 / D11** — Keep session-level human-in-the-loop in the MVP, or drop it to start? (Recommended: keep — it is small and the product's differentiator; drop first if time-boxing is urgent.)
- **D9** — Backend persistence: in-memory (simplest) or wire Mongo from day one? (Recommended: in-memory first; Mongo models ready.)

All other "recommended" decisions are reversible without architecture change — that's the point of keeping them explicit.

---

## Decisions

### D1 — MVP boundary
- **Decision**: MVP = single-agent tab sessions with real-time streaming, dynamic schema-driven forms, run/stop/rerun/restart, session cloning, and (recommended) session-level HITL approvals. Workflow builder/engine, orchestration, persistence, Python harness = post-MVP.
- **Rationale**: matches `architecture/MVP_Overview.md` scope (workflow builder = future); everything post-MVP already has interfaces/models/stubs defined in specs, so no rework risk.
- **Impact**: none of the post-MVP pieces block the MVP; contracts keep them attachable later.
- **Revisit when**: a workflow user story is pulled into MVP (then D10 unwinds).

---

### D2 — Transport: SSE
- **Decision**: All session events stream over **SSE** (`GET /sessions/:sessionId/events`), not WebSocket. `Frontend_SSE_Client_Scaffold.md` replaces the WS client; EventSource auto-reconnects (ideal for long-running / paused workflows) and needs no client lib.
- **Rationale**: resilience, polyglot backend via Redis, zero client dependency.
- **Impact**: Gateway owns SSE lifecycle (headers, heartbeat, close-on-client-gone); no WS server in MVP.
- **Revisit when**: bidirectional server→client pushing for controls is needed (not an MVP need; run/stop are plain POSTs).

---

### D3 — Backend topology: Gateway + Harness + Redis
- **Decision**: Implement the **Gateway** (Fastify, sessions REST + SSE, Redis bridge) and **Agent Harness** (worker executing agents, emitting events to Redis) as separate processes. The older sessions-monolith backend (`spec/backend_api_contract.md`, `scaffolds/Backend_Scaffold.md`) is **superseded** (annotate those files "superseded — see Gateway_Scaffold").
- **Rationale**: the product description explicitly adopts gateway+harness+Redis; the SSE scaffold says it replaces WS; polyglot Python/Node support requires harness decoupled from HTTP.
- **Impact**: two Node processes + Redis; contracts package mediates their shared types.
- **Revisit when**: never (architecture landing zone).

---

### D4 — Entities: session-first
- **Decision**: `sessionId` is **required on every runtime event** (logs, output, state, errors, approvals); `taskId` is an optional correlation ID published only when the Gateway creates an internal task row (1:1 with a session run).
- **Rationale**: fixes G5; harness scaffolds currently default `sessionId = taskId`, which breaks fan-out if workflows or retries arrive.
- **Impact**: harness worker must carry `sessionId` through every handler; no more deriving session from task.
- **Revisit when**: a true multi-run task (one session, multiple task attempts) is needed — then promote `runId`/`taskId` to first-class (cheap, additive).

---

### D5 — Canonical session state set
- **Decision**: Adopt the spec's set everywhere: `idle, running, streaming, waiting, completed, error`. Replace every `failed` with `error`; add `streaming` to MongoDB `Session.state`. Keep a **separate** `task.status` enum (PENDING, RUNNING, WAITING_FOR_HUMAN, QUEUED, COMPLETED, FAILED) for internal task rows.
- **Rationale**: fixes G1; spec JSON is authoritative (`spec/session_state_machine.json`); `failed` is a leftover from older drafts.
- **Impact**: state indicator icons (⚪🟢🔵🟡🟣🔴) map 1:1 to machine states; one enum in `packages/contracts` used by gateway, harness, frontend.
- **Revisit when**: nothing.

---

### D6 — Canonical event model (one trait, two encodings)
- **Decision**: One canonical event type: `{ type, sessionId, taskId?, timestamp, payload }`. `spec/event_payloads.json` defines session event payloads (extended with `approval.*` from `Redis_Event_Schema.md`); Redis Pub/Sub messages and SSE `data:` frames are both just JSON encodings of this same model. No "flat payload" dialect.
- **Rationale**: fixes G5; the frontend parser suffers one shape; fan-out/SSE/replay need no per-channel special-casing.
- **Impact**: `packages/contracts` exports the canonical type + per-type payload schemas + `normalizeEvent`; gateway, harness, frontend all import from there.
- **Revisit when**: new event types need adding (additive, non-breaking).

---

### D7 — Canonical HTTP/SSE API (sessions-centric)
- **Decision**: Gateway exposes: `POST /sessions/create` `{agentId, input?, config?}` → `{sessionId}`; `POST /sessions/:id/run` `{input?}` → `202 {sessionId, state}`; `POST /sessions/:id/stop` → `202`; `POST /sessions/:id/approve` `{choice: string}` → `202` (only if HITL kept); `GET /sessions/:id` → session view; `GET /sessions/:id/events` → SSE stream; `GET /agents/list` → `AgentDescriptor[]`; `GET /agents/:agentId/schema` → canonical agent schema; `GET /health`. No `/tasks` routes in MVP (tasks are internal).
- **Rationale**: fixes G3; sessions are the UI concept; task endpoints would leak internals.
- **Impact**: frontend client + gateway implemented against this one contract; Gateway scaffold's `/tasks` routes dropped.
- **Revisit when**: external programmatic task submission is needed (then add `POST /tasks` as an alternative entry).

---

### D8 — Agent schema format (JSON Schema + UI metadata)
- **Decision**: Canonical agent schema = **JSON Schema (draft-07 subset)** for `input` (+ optional `config`), with `x-ui-*` annotations (label, placeholder, widget, order). `spec/agent_schema_format.json` wins; the loader's field-list example is superseded. Validator: `ajv` in `packages/agent-schemas`; the Gateway validates input at `/run` before publishing `task.created`; the frontend generates forms from `inputSchema.properties` + `x-ui-*`.
- **Rationale**: fixes G4; JSON Schema is standard, toolable, and matches the API contract.
- **Impact**: seed agent schemas written in this format; field-list code never written.
- **Revisit when**: a richer form-builder DSL is needed (extend `x-ui-*` additively).

---

### D9 — Persistence: in-memory MVP, optional Mongo
- **Decision**: MVP runs with an **in-memory session/task store in the Gateway** (Map + TTL sweep when tab closes / session ends) and an in-memory pending-approval store in the Harness. MongoDB models remain specced but **unwired by default** (compose service commented). Sessions are not durable across gateway restarts in MVP.
- **Rationale**: removes an infrastructure dependency from the critical path; the store interface can match the Mongo models so swapping later is mechanical.
- **Impact**: `store/` module boundary exists from day 1; restart during a run loses the session (acceptable for MVP; document it).
- **Revisit when**: durability across restarts is required (wire Mongo; interface already shaped like the Mongo models).

---

### D10 — Workflow engine: interfaces now, implementation post-MVP
- **Decision**: MVP ships only: `workflow_engine_interface.md` contracts as TS interfaces in `packages/contracts`, the workflow **canvas placeholder** ("Coming Soon") and the `Workflow`/`WorkflowNode`/`WorkflowEdge` models. No engine, no persistence, no canvas editing.
- **Rationale**: the UI/event-bus reuse story doesn't need the engine; keeping it out de-risks the MVP.
- **Impact**: none on gateway/harness/frontend day-1 work; placeholder communicates direction.
- **Revisit when**: first multi-agent workflow user story is pulled in.

---

### D11 — Human-in-the-loop: session-level approvals in MVP (recommended)
- **Decision**: Keep **session-level** HITL: an agent can request approval → session state `waiting`, `approval.requested` event, `AgentApprovalPanel` renders options, user picks → `POST /sessions/:id/approve` → `task.resume` → harness restores and continues. **No workflow-node approvals, no multi-approver, no timeout policies** in MVP.
- **Rationale**: it is the product's differentiator, touches only one state + one endpoint + one component, and the event types already exist; also the spec's `waiting` state needs a consumer or it's dead weight.
- **Impact**: calculator seed agent gets `approvalRequired` config to demo it; G9 resolved.
- **Revisit when**: workflow-node approvals (post-MVP) — reuse the same event pair with `nodeId` added.

---

### D12 — Run semantics: Rerun vs Restart vs Clone pinned
- **Decision**: **Run Again (Rerun)** = re-execute the *same* session (new task; logs/output reset in UI; state idle→running). **Restart** = the clone flow (new session + new tab from `SessionSnapshot`, per `Cloning_Model.md`). **Clone** (tab menu) = same as Restart. Snapshot is built **client-side** from the session's input/config/uiState (MVP; no backend snapshot endpoint, G10). Input editing disabled while running/streaming/waiting.
- **Rationale**: matches `Tab_Model.md`/`Cloning_Model.md`/`Session_Lifecycle.md` and resolves G10 pragmatically.
- **Impact**: `AgentControlPanel` + tab menu behavior; no backend changes beyond what D7 already has.
- **Revisit when**: server-side snapshots/scheduling are needed.

---

### D13 — Replay & hydration: tab-scoped replay only
- **Decision**: The frontend `ReplaySubject(100)` per session covers **late tab subscribers within the same page load**. A full page reload shows fresh dashboards: it lists live sessions (via `GET /sessions/:id` state or the agent list) but does **not** backfill history in MVP. Gateway TTL-sweeps in-memory sessions after client disconnect (short grace, e.g. 60 s) so SSE reconnects (EventSource auto-retry) re-attach cleanly.
- **Rationale**: cheapest MVP-satisfying behavior; server-side event buffer/journal is a post-MVP add (fan-out already isolates per-session channels).
- **Impact**: document reload behavior in UI ("session history not persisted"); no extra backend work.
- **Revisit when**: mid-session reload/backfill is demanded (add bounded Redis stream per session).

---

### D14 — Ops, config & health
- **Decision**: `GET /health` on Gateway (`{ok, redis:"up|down", version}`); all ports/URLs via `.env` with defaults in `packages/contracts/src/env.ts` (gateway 3001, redis 6379, mongo 27017, frontend 5173); `.env.example` committed; pino logging (info level; request logs in dev); **no auth, no rate limiting, no TLS** in MVP (local tool assumption).
- **Rationale**: kills G12; keeps ops minimal but observable.
- **Impact**: single env surface; compose reflects the same defaults.
- **Revisit when**: anything is deployed beyond localhost.

---

### D15 — Tooling & repo shape
- **Decision**: **npm workspaces monorepo** (no Turborepo/Nx yet): `packages/contracts`, `packages/agent-schemas`, `apps/gateway`, `apps/frontend`, `services/agent-harness`; TypeScript strict everywhere; Vite + React + RxJS + Vitest (+ RTL, marble tests) for frontend; Fastify + ioredis for gateway/harness; Supertest for API tests; docker compose for Redis (+ optional Mongo); GitHub Actions for lint/typecheck/test/build; smoke script for e2e.
- **Rationale**: zero extra tooling risk; matches scaffold library choices; monorepo layout in `repo_layout.md` (fixes G6).
- **Impact**: two-package contracts split (contracts vs agent-schemas); can merge later without consumer changes if desired.
- **Revisit when**: build times hurt (add Turborepo), or a Python harness arrives (add uv/poetry workspace + its own CI job).