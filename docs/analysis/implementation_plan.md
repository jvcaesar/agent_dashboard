# Implementation Plan — Agent Dashboard MVP

> Companion: [`project_analysis.md`](./project_analysis.md), [`scope_decisions.md`](./scope_decisions.md) (decisions D1–D16), [`repo_layout.md`](./repo_layout.md), plus existing `docs/spec` and `docs/scaffolds` (contracts) — **scope decisions confirmed (2026-09-05): HITL in MVP, in-memory persistence, npm-workspaces tooling (D15), seed agents echo + calculator (D16); no open questions block any phase**.

---

## 1. Principles

1. **Every part is small, shippable, and independently verifiable.** Each part below lists a concrete deliverable + a runnable check (command/test/acceptance sign-off).
2. **Sequence only where there is a real dependency; everything else is parallel (marked `∥`).**
3. **A part is "done" when its verification passes and its output compiles** — not when it "looks right".
4. **Contracts come first.** Any two parts may only communicate through `packages/contracts` types/schemas, never through private shapes (D6/D7).
5. **Post-MVP items (Phase 6) must not be needed by any MVP part** — to do later without major changes (that's the point of D1/D10/D11/D13).

---

## 2. Dependency map

```text
P0 repository & infra foundation          (sequential, everything depends on it)
 │
 ├─► P1 shared contracts + agent schemas  (sequential after P0)
 │     │
 │     ├─► P2 Gateway                      (parallel with P3, 4)
 │     ├─► P3 Agent Harness + seed agents  (parallel with P2; needs P1 only)
 │     └─► P4 Frontend                      (can start against P1 contracts + mocks;
 │     │                                        full e2e after P2+P3 land)
 │     ▼
 └─► P5 end-to-end hardening              (sequential after P2+P3+P4 merge)
 └─► P6 post-MVP backlog                   (any time, no MVP dependencies)
```

**Read**: P0 → P1 → {P2 ∥ P3, and P4 starts as soon as P1 exists (it can develop against a tiny mock SSE/API harness until P2/P3 land)} → P5 → P6. If P4 must not wait, P2's SSE/API contract fixtures in spec, plus the P1 contract package, are enough to drive P4 with a mocked transport.

---

## 3. Verification toolbox

| Layer | Tool | Command (from repo root with npm workspaces) |
|---|---|---|
| Types & schemas | Vitest | `npm test -w @agent/contracts` / `-w @agent/agent-schemas` |
| Gateway API | Vitest + Supertest + real Redis | `npm test -w @agent/gateway` (requires `docker compose up -d redis`) |
| Harness | Vitest (unit) + integration against Redis | `npm test -w @agent/agent-harness` |
| Frontend unit | Vitest | `npm test -w @agent/frontend` |
| Frontend streams | RxJS marble tests (Vitest) | same as above |
| Whole stack | Node smoke script | `npm run smoke` (brings up gateway+harness, asserts SSE received ordered events) |
| Static | tsc + eslint | `npm run typecheck && npm run lint` |
| Build | — | `npm run build` (all workspaces) |
| CI | GitHub Actions (minimal) | lint + typecheck + test + build on PR |

`docker compose up -d redis` starts Redis (Mongo optional); everything else runs via `npm run dev -w …` in separate terminals (3 processes: gateway, harness, frontend).

---

## 4. Phases

### Phase 0 — Repository & infra foundation (sequential; prerequisite for everything)

**Part 0.1 — Monorepo scaffold**
- Depends: — · ∥: with 0.2.
- Deliverable: root `package.json` (npm workspaces), `tsconfig.base.json`, `.gitignore`, `.editorconfig`, `README.md` (quickstart pointer, workspace table); empty workspaces for `apps/*`, `services/*`, `packages/*`; `npm install` resolves clean.
- Verify: `npm install` succeeds; `npm run typecheck` (no code yet) exits 0.

**Part 0.2 — Infra compose + env**
- Depends: — · ∥: with 0.1.
- Deliverable: `infra/docker-compose.yml` (redis; mongo commented); `infra/redis.conf` minimal; `.env.example` (ports/URLs per D14); `.env` created locally from example; `packages/contracts/src/env.ts` constants (3001/6379/27017/5173).
- Verify: `docker compose -f infra/docker-compose.yml up -d redis`; `docker compose ps` shows healthy; `redis-cli ping` → `PONG`.

**Part 0.3 — Shared config & CI**
- Depends: 0.1, 0.2 · ∥: —
- Deliverable: root scripts dev/build/test/lint/typecheck/smoke; GitHub Actions workflow (lint, typecheck, test, build).
- Verify: CI runs green on a skeleton PR; `npm run test` exits 0 with zero tests (or "no tests found" accepted if configured).

---

### Phase 1 — Shared contracts package (sequential after P0; prerequisite for P2–P4)

**Part 1.1 — Canonical types (events, state machine, entities)**
- Depends: P0 · ∥: with 1.2.
- Deliverable: `packages/contracts` with pure TS types exported: `Event`, `SessionState` (the D5 set), `SessionEvent` variants (session.created, started, stateChanged, output.partial/final, log.append, session.error/completed, approval.requested/resumed) per `spec/event_payloads.json` + `Redis_Event_Schema.md` reconciled under D6; `Session`, `Tab`, `AgentDescriptor`, `AgentSchema`; validators (simple type guards + state-transition table per `spec/session_state_machine.json`).
- Verify: `npm test -w @agent/contracts`: unit tests for transition table (every allowed/disallowed edge), type-guard happy/sad paths; `npm run typecheck` clean.

**Part 1.2 — Canonical agent schema package**
- Depends: P0 · ∥: with 1.1.
- Deliverable: `packages/agent-schemas`: JSON Schema (draft-07) format types + `ajv` validator (validateInput, validateConfig) implementing D8; seed schema files (echo.json, calculator.json) validated against the format itself; schema registry `getAgentSchema(agentId)` cached.
- Verify: `npm test -w @agent/agent-schemas`: validator accepts/rejects fixtures; seed files each pass `ajv.compile`; format conformance test (no field-list leftovers).
- Note: it is a sibling of contracts (D15); P2–P4 import both. If you prefer one package, merge into `packages/contracts` — decision stored only in scope doc.

---

### Phase 2 — Gateway (sequential after P1; ∥ with P3/P4)

**Part 2.1 — Gateway skeleton + /health + config**
- Depends: 1.1, 1.2 · ∥: yes (with 2.x, 3.x, 4.x).
- Deliverable: `apps/gateway`: Fastify app, dotenv config (D14), CORS, JSON body parser; `GET /health` reporting redis connectivity: `{ ok:true, redis:"up|down" }`; error handler + pino request logging; `server.ts` boots on `PORT`.
- Verify: `npm test -w @agent/gateway`: Supertest /health with Redis up/down stubbed; manual `curl http://localhost:3001/health`.

**Part 2.2 — Sessions REST core**
- Depends: 2.1 · ∥: with 2.4 (2.3 needs 2.2).
- Deliverable: in-memory `SessionStore` (Map, TTL sweep per D13); routes per D7: `POST /sessions/create` (creates session + internal task row (1:1), publishes nothing yet), `POST /sessions/:id/run` (validates input via schema package; publishes `task.created` incl. `sessionId` (D4) to Redis channel `tasks`; starts no execution itself), `POST /sessions/:id/stop` (publishes stop command), `POST /sessions/:id/approve` (choice → `task.resume` (stub until P3)).
- Verify: Supertest: create→run→stop→approve return contract shapes; unknown id → 404; invalid input → 4xx; Redis mock asserts `tasks` channel payloads carry `sessionId`.

**Part 2.3 — Redis bridge + fan-out + SSE**
- Depends: 2.2 · ∥: with 2.4.
- Deliverable: Redis subscriber (channels `agent.logs`, `agent.events`) + per-scaffold fan-out to `session:{sessionId}:events` (`Session_Event_Fan-Out.md`); SSE route `GET /sessions/:sessionId/events` subscribing the per-session channel (headers, heartbeat every 15 s, close/cleanup on client disconnect); `normalizeEvent` from contracts; events carry sessionId first-class (D4/D6).
- Verify: integration test with real Redis (`docker compose up -d redis`): publish fake `log.append` on `agent.logs` ⇒ assert SSE client receives `data:` frame with canonical shape; fan-out publishes to `session:{sid}:events`; disconnect cleans subscriber (no leak, check via `redis-cli pubsub numsub`).

**Part 2.4 — Agent list & schema routes + seeding**
- Depends: 2.1 · ∥: with 2.3.
- Deliverable: `GET /agents/list` from `packages/agent-schemas` registry; `GET /agents/:agentId/schema` returns canonical schema (404 unknown); dev-only seeding script copies seed schemas into registry at boot (or static import).
- Verify: Supertest: list returns ≥2 descriptors; schema round-trips exactly; unknown id → 404; schema payload validates against contracts type.

---

### Phase 3 — Agent Harness + seed agents (sequential after P1; ∥ with P2/P4)

**Part 3.1 — Harness skeleton (worker + Redis sub/pub)**
- Depends: 1.1, 1.2 · ∥: with 2.x/4.x.
- Deliverable: `services/agent-harness`: Node worker per `Agent_Harness_Scaffold.md`: subscribes Redis `tasks`; dispatches `task.created` → run handler, `task.resume` → resume handler; redis pub helpers (channels `agent.logs`, `agent.events`, per-session channels) using canonical event model (D4/D6); `sessionId` threaded explicitly through every message.
- Verify: unit (handler mapping) + integration: inject `task.created` JSON on `tasks` ⇒ assert corresponding events published to `agent.logs` (+ `session:{sid}:events`) with `sessionId` present.

**Part 3.2 — Contract-compliant runtime + seed agents**
- Depends: 3.1 · ∥: with 3.3 prep.
- Deliverable: `AgentRuntime` interface implementation per `spec/agent_runtime_contract.md` (`streamEvents`, `invoke`, `abort`, `getState`): two concrete agents: **echo** (streams input back in chunks with delays, logs progress, emits usage+final) and **calculator** (parses `expression: string`, streams step logs, final numeric result); abort support; `echo` doubles as the default smoke-test agent; seed schemas already in `packages/agent-schemas` (1.2). Behavior and acceptance criteria per `spec/seed_agents.md` (D16).
- Verify: unit tests per agent: ordered event sequence (session.started → log/output.partial → output.final → completed) per contract guarantees; abort emits error with `CANCELLED`; invalid input → `INVALID_INPUT` error before execution; integration: complete run finishes under 2 s.

**Part 3.3 — HITL pause/resume (session-level, D11)**
- Depends: 3.2 · ∥: —.
- Deliverable: approval flow: runtime signals intent (e.g. calculator agent with `approvalRequired: true` config) → harness emits `approval.requested` (+ state `waiting`); persists minimal in-memory pending state; on `task.resume` → restores, emits resume log, continues, state `running` → completes. Snapshot/store hooks (`AgentSnapshot` model) stubbed empty (post-MVP wiring).
- Verify: integration: run agent with approvalRequired ⇒ assert `approval.requested` + state waiting on session channel; post `task.resume` ⇒ assert completion; unit: waiting→running→completed transitions legal per 1.1 table.

---

### Phase 4 — Frontend (uses P1 contracts; early parts work with mocked transport; ∥ with P2/P3)

**Part 4.1 — Vite shell + layout**
- Depends: 1.1 · ∥: with 4.2.
- Deliverable: `apps/frontend`: Vite + React + TS strict; empty shell matching `Wireframes.md` layout regions (tab bar, left sidebar, main workspace, right inspector slot, bottom logs); routing-free (single dashboard + workflow placeholder modal (D10)); styling via CSS modules with wireframe class names (`state-indicator`, `control-panel`, …).
- Verify: Vitest: RTL smoke renders regions by test id; `npm run build -w @agent/frontend` passes; manual: visual check against `docs/architecture/Wireframes.md`.

**Part 4.2 — State layer (event bus + session streams)**
- Depends: 1.1 · ∥: with 4.1, 4.3.
- Deliverable: `state/eventBus.ts` per `EventBus_RxJS.md`/`Frontend_Scaffold.md` (Subject + per-session ReplaySubject(100), `routeEvent`, `sessionEvents(sessionId)` w/ shareReplay); `state/sessionStreams.ts` (logsForSession, partialOutputForSession, finalOutputForSession, stateForSession), typed by contracts.
- Verify: RxJS **marble tests** (Vitest + `rxjs/testing`): routing fans out to session + global; late subscriber gets replay (≤100); log scan accumulates; state distinctUntilChanged; no leakage after unsubscribe (takeUntil/tab-close).

**Part 4.3 — API client + SSE glue**
- Depends: 1.1 (contracts suffice; wire mocks until P2 lands) · ∥: with 4.2, 4.4.
- Deliverable: `api/backendClient.ts` (createSession, runSession, stopSession, approveSession, listAgents, getAgentSchema) against D7; `api/eventStreamClient.ts` per `Frontend_SSE_Client_Scaffold.md` (connect/disconnect, parse, routeEvent, tolerate auto-reconnect (D13)); dev-mode injectable mock transport (same events, timed) so UI/state layers stay testable without a running backend.
- Verify: unit mocks of fetch/EventSource: correct URL+payloads per method; SSE parse→routeEvent; RTL: component set renders from mocked stream events.

**Part 4.4 — Session workspace components (session-bound)**
- Depends: 4.2, 4.3 · ∥: with 2.x/3.x.
- Deliverable: components per `Component_Architecture.md`: `AgentHeader` (name+sessionId), `AgentInputForm` (schema-driven per D8: fields, validation msgs, Run btn), `AgentOutputViewer` (partial stream + final), `AgentLogViewer` (accumulated logs, severity colors (wireframe)), `AgentControlPanel` (Run/Stop/Rerun/Restart per D12), `AgentStateIndicator` (⚪🟢🔵🟡🟣🔴 per machine (D5)), `AgentApprovalPanel` (HITL prompt+options, D11); all take `{ sessionId }` and subscribe via `sessionEvents` (each self-contained, unsubscribe on unmount).
- Verify: RTL per component: renders from a scripted virtual session (events injected via `routeEvent`); indicator icon matches each state; input form renders each schema property (incl. required/default); approval panel shows options and fires `approve`.

**Part 4.5 — Tab bar & sidebar (tabs ∥ sessions)**
- Depends: 4.4 · ∥: —.
- Deliverable: `TabBar` (title, state icon, close, "+" opening new-session modal (agent picker)); `Sidebar` (agent list from `/agents/list`, click opens new tab); `Tab` model incl. `uiState` per `Tab_Model.md`; tab close → disconnect SSE + drop session reference (gateway TTL cleans up (D13)).
- Verify: RTL: open from sidebar ⇒ tab appears + SSE connect called; switch tab ⇒ components resubscribe to new sessionId (verify via injected events reaching the right session only); close ⇒ unsubscribed (no further state updates).

**Part 4.6 — Cloning + Rerun/Restart UX**
- Depends: 4.5 · ∥: —.
- Deliverable: clone via `SessionSnapshot` (`Cloning_Model.md`, client-side per D12): new session+tab in idle, input/config preserved, logs cleared, "(clone)" title; Run Again = same-session rerun (D12); Restart = clone flow (D12); disabled controls per state (input disabled while running/streaming/waiting) per `Session_Lifecycle.md` UI mapping.
- Verify: RTL: clone ⇒ fresh idle tab with preserved input; rerun clears logs/output and state cycles idle→running→…; restart lands on clone tab; snapshot builder unit: rejects runtime state copies (no logs/output copied).

---

### Phase 5 — End-to-end hardening (sequential after P2+P3+P4 land)

**Part 5.1 — Smoke script + contract alignment pass**
- Depends: 2.x+3.x+4.x · ∥: —.
- Deliverable: `scripts/smoke.mjs` (the `smoke` npm script): boots gateway+harness (docker compose up redis); opens SSE client; POST create→run→expect complete event sequence (stateChanged running→streaming→…→completed, logs, partial, final) within timeout; asserts canonical shapes via contracts validators; reports OK/fail exit code. Also: sweep pass reconciling every literal state/endpoint/event name against `packages/contracts` (kills remaining G1–G5 stragglers in code/doc comments).
- Verify: `npm run smoke` (exit 0; run twice for idempotency (two sessions, no leaks in Redis subs)).

**Part 5.2 — HITL e2e + clone/rerun e2e**
- Depends: 5.1 · ∥: —.
- Deliverable: smoke extensions: approval flow (run calculator with approvalRequired → approval.requested → POST approve → completion) and restart-from-clone run.
- Verify: `npm run smoke` covers both scenarios; manual demo walkthrough against `MVP_Overview.md` acceptance.

**Part 5.3 — README + doc status pass**
- Depends: 5.1 · ∥: with 5.2.
- Deliverable: root README (quickstart: compose up, env, 3 processes, smoke); `docs/analysis/` linked from README. (Superseded-doc banners: already applied 2026-09-05.)
- Verify: fresh-clone walkthrough by a teammate follows README successfully (manual).

---

### Phase 6 — Post-MVP backlog (not started in MVP; contracts only)

Kept out of MVP per D1/D10/D13, **no MVP part may depend on these**:
- Workflow engine implementation + canvas (interfaces already in contracts; placeholder ships in 4.1).
- MongoDB persistence wiring (models specced; store interface shaped in 2.2).
- Server-side session replay buffer / history backfill after reload (D13).
- Python harness (runtime contract is transport-agnostic; Redis makes it drop-in; uv per D15).
- Backend snapshot endpoint for cloning (D12 uses client-side snapshots).
- Auth, rate limiting, TLS, multi-approver HITL, workflow-node approvals, agent marketplace.

---

## 5. Parallel vs sequence — summary

**Strictly sequential (each blocks the next):**
- P0.1/P0.2 → P0.3 (foundation before CI)
- P0 → P1 (contracts need the workspace)
- P1 → P2.x and P1 → P3.x (both processes import contracts)
- 2.1 → 2.2 → 2.3 (gateway layers stack)
- 3.1 → 3.2 → 3.3 (harness layers stack)
- 4.4 → 4.5 → 4.6 (components → tab integration → clone UX)
- All of P2+P3+P4 → P5 (hardening integrates everything)

**Parallel tracks (after P1, all three proceed simultaneously):**
- Track A (Gateway): 2.1 ∥ 2.4, then 2.2, then 2.3
- Track B (Harness): 3.1 → 3.2 → 3.3
- Track C (Frontend): 4.1 ∥ 4.2 ∥ 4.3 → 4.4 → 4.5 → 4.6 (4.3 uses mock transport until 2.2/2.4 land — no blocking)

**Within phases**: 1.1 ∥ 1.2; 0.1 ∥ 0.2; 5.2 ∥ 5.3.

**Single-developer order of attack** (if no parallelism available): 0.1 → 0.2 → 0.3 → 1.1 → 1.2 → 2.1 → 2.2 → 2.4 → 3.1 → 3.2 → 3.3 → 2.3 → 4.1 → 4.2 → 4.3 → 4.4 → 4.5 → 4.6 → 5.1 → 5.2/5.3. (Backend before SSE wiring so the frontend always has a live backend to point at; 2.3 last in gateway so fan-out is tested against a real harness.)