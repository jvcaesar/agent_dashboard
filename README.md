# Agent Dashboard

An event-driven, browser-style dashboard for running, observing, and controlling **AI agents**. Each browser tab represents exactly **one agent session**, streaming logs, partial/final output, and lifecycle state in real time — architected so the same session components can later be reused inside a multi-agent workflow builder.

> **Status: 📐 Product definition complete — implementation not started.**
> This repository currently contains the complete, reconciled specification in [`docs/`](docs/). Code arrives with **Phase 0** of the [implementation plan](docs/analysis/implementation_plan.md). All product decisions (**D1–D16**) are confirmed — see [`docs/analysis/scope_decisions.md`](docs/analysis/scope_decisions.md). Progress is tracked in the [implementation tracker](#implementation-tracker) below.

---

## Implementation tracker

> **Living checklist** — update the Status column as work progresses (flip a part to ✅ only when its *Done when* verification passes).
> Legend: ⬜ not started · 🔄 in progress · ✅ done (verification passed) · ⏸️ blocked/paused
> **Progress: 0 / 21 parts** · Last updated: 2026-09-05 · Full deliverables & commands: [`docs/analysis/implementation_plan.md`](docs/analysis/implementation_plan.md)

| Phase | Part | Deliverable | Done when | Status |
|---|---|---|---|---|
| **P0 — Foundation** (sequential) | 0.1 | Monorepo scaffold: npm workspaces, tsconfig.base, editorconfig/gitignore | `npm install` clean; `npm run typecheck` exits 0 | ⬜ |
| | 0.2 | Infra: docker-compose (Redis; Mongo commented), `.env.example`, env defaults module | `docker compose up -d redis` healthy; `redis-cli ping` → `PONG` | ⬜ |
| | 0.3 | Root scripts (dev/build/test/lint/typecheck/smoke) + GitHub Actions CI | CI green on skeleton PR; `npm run test` exits 0 | ⬜ |
| **P1 — Contracts** (after P0) | 1.1 | Canonical types + validators: events, session states + transition table, entities | `npm test -w @agent/contracts` green; `npm run typecheck` clean | ⬜ |
| | 1.2 | Agent-schema package: format, ajv validator, registry, seed schema files (echo, calculator) | `npm test -w @agent/agent-schemas` green; seed files compile | ⬜ |
| **P2 — Gateway** (∥ P3/P4) | 2.1 | Fastify skeleton: config, CORS, pino, `GET /health` | Supertest /health (Redis up/down); `curl :3001/health` | ⬜ |
| | 2.2 | Sessions REST core: in-memory store (TTL), create/run/stop/approve routes | Supertest contract shapes; 404s; `task.created` carries `sessionId` | ⬜ |
| | 2.3 | Redis bridge + fan-out + SSE stream | Real-Redis test: publish → SSE frame; no subscriber leaks on disconnect | ⬜ |
| | 2.4 | Agent list & schema routes + seeding | `GET /agents/list` ≥ 2 agents; schema round-trip; 404 unknown | ⬜ |
| **P3 — Harness** (∥ P2/P4) | 3.1 | Harness skeleton: Redis worker, sub/pub, canonical emit helpers | `task.created` → events on `agent.logs` + session channel with `sessionId` | ⬜ |
| | 3.2 | Contract-compliant runtime + seed agents (echo, calculator) | Ordered event sequences; abort → `CANCELLED`; `INVALID_INPUT`; run < 2 s | ⬜ |
| | 3.3 | HITL pause/resume (D11) | `approval.requested` + `waiting`; approve → resume → completed | ⬜ |
| **P4 — Frontend** (∥ P2/P3) | 4.1 | Vite shell + wireframe layout regions | RTL smoke; `npm run build -w @agent/frontend` | ⬜ |
| | 4.2 | State layer: RxJS event bus + per-session streams | Marble tests: routing, replay ≤ 100, unsubscribe cleanup | ⬜ |
| | 4.3 | API client + SSE glue + dev mock transport | fetch/EventSource mocks; parse → `routeEvent` | ⬜ |
| | 4.4 | Session workspace components (header, form, output, logs, controls, indicator, approvals) | RTL scripted session; state icons; schema-driven form; approve fires | ⬜ |
| | 4.5 | Tab bar & sidebar (tabs ∥ sessions) | Open/switch/close: SSE connect, resubscribe, unsubscribe | ⬜ |
| | 4.6 | Cloning + Rerun/Restart UX (D12) | Clone → fresh idle tab, input preserved; snapshot rejects runtime state | ⬜ |
| **P5 — Hardening** (after P2+P3+P4) | 5.1 | `scripts/smoke.mjs` + contract-alignment sweep | `npm run smoke` exit 0; run twice, no Redis-sub leaks | ⬜ |
| | 5.2 | HITL e2e + clone/rerun e2e | Smoke covers approval flow + restart-from-clone | ⬜ |
| | 5.3 | README quickstart finalized | Fresh-clone walkthrough follows README successfully | ⬜ |

*P6 post-MVP backlog (workflow engine, Mongo wiring, server-side replay, Python harness via uv, auth) is tracked in the [implementation plan](docs/analysis/implementation_plan.md), not in this tracker.*

---

## How it works

```text
Frontend (React + RxJS, tab per session)
   │   HTTP + SSE
   ▼
Gateway (Fastify: sessions, SSE, Redis bridge)
   │   Redis Pub/Sub
   ▼
Agent Harness (worker: executes agents, emits events)
   │   optional
   ▼
MongoDB (state store — optional in MVP)
   │   post-MVP
   ▼
Workflow Engine (RxJS graph: runs/stops/resumes workflows)
```

- The **Gateway** owns the sessions-centric REST API and streams every session event to the browser over **SSE**.
- The **Agent Harness** is a Redis worker that executes agents and publishes canonical events — decoupled from HTTP, so a Python harness can be added later without changes.
- Every event follows one canonical model: `{ type, sessionId, taskId?, timestamp, payload }`.

## MVP features

- **Browser-style UI** — tab bar, agent-list sidebar, session workspace (input, output, logs, controls)
- **Real-time streaming** — logs, partial/final output, and state changes over SSE → an RxJS event bus
- **Schema-driven forms** — input forms generated from agent JSON Schemas (draft-07 + `x-ui-*` hints)
- **Session controls** — Run / Stop / Rerun / Restart, plus session cloning (tab duplication)
- **Human-in-the-loop approvals** — an agent can pause in `waiting`, the user picks an option, the agent resumes
- **Workflow builder placeholder** — ships as "Coming Soon"; the engine is post-MVP behind already-specced interfaces

## Tech stack (planned)

| Layer | Choice |
|---|---|
| Frontend | React + Vite + RxJS + CSS modules, TypeScript strict |
| Gateway | Fastify + ioredis, pino logging |
| Agent Harness | Node worker over Redis pub/sub (Python-ready via the same runtime contract) |
| Validation | ajv — JSON Schema (draft-07) + `x-ui-*` UI metadata |
| Testing | Vitest (+ React Testing Library, RxJS marble tests), Supertest, e2e smoke script |
| Infra & CI | Docker Compose (Redis; Mongo optional/commented), GitHub Actions (lint · typecheck · test · build) |
| Monorepo | npm workspaces |

## Planned repository layout

```text
packages/contracts        → canonical TS types + validators (single source of truth)
packages/agent-schemas    → agent JSON schemas + ajv validation + registry
apps/gateway              → HTTP/SSE API, Redis bridge        → :3001
services/agent-harness    → Redis worker executing agents      → (no HTTP)
apps/frontend             → React + RxJS dashboard             → :5173
infra/                    → docker-compose (Redis; Mongo commented)
scripts/                  → smoke.mjs (end-to-end check)
```

Full target tree: [`docs/analysis/repo_layout.md`](docs/analysis/repo_layout.md) (confirmed).

## Seed agents

Two contract-compliant agents ship with the MVP — spec: [`docs/spec/seed_agents.md`](docs/spec/seed_agents.md).

| Agent | Purpose | Notes |
|---|---|---|
| **echo** | Proves the full loop (form → run → streaming → complete); default smoke-test target | Streams input back in chunks; abort + `INVALID_INPUT` handling |
| **calculator** | Proves step-logging, validation, and the approval flow | `approvalRequired: true` pauses in `waiting` for a human decision (D11) |

Zero external dependencies — everything runs locally out of the box.

## API at a glance

| Method & path | Purpose |
|---|---|
| `POST /sessions/create` | Create a session (`{agentId, input?, config?}`) |
| `POST /sessions/:id/run` · `/stop` · `/approve` | Run, stop, resolve an approval |
| `GET /sessions/:id/events` | **SSE** stream of canonical session events |
| `GET /agents/list` · `/agents/:agentId/schema` | Agent discovery + schema (drives the dynamic form) |
| `GET /health` | Gateway + Redis connectivity |

Session states: `idle → running → streaming → waiting → completed | error`

## Getting started (target workflow)

> ⚠️ These commands become real in **Phase 0** — they describe the confirmed plan, not the current repo state.

```bash
# Prerequisites: Node 20+, Docker
git clone https://github.com/jvcaesar/agent_dashboard.git
cd agent_dashboard

docker compose -f infra/docker-compose.yml up -d redis   # start Redis
cp .env.example .env                                      # ports/URLs (defaults work)
npm install
npm run dev        # gateway :3001 · harness (Redis worker) · frontend :5173
npm run smoke      # end-to-end: create → run → assert streamed event sequence
```

## Documentation map

| Path | Contents |
|---|---|
| [`docs/spec/`](docs/spec/) | **Authoritative contracts** — agent runtime contract, agent schema format, canonical event payloads, session state machine, seed agents, workflow engine interface |
| [`docs/architecture/`](docs/architecture/) | UI & system design — MVP overview, wireframes, component architecture, event bus, tab model, cloning model, session lifecycle |
| [`docs/scaffolds/`](docs/scaffolds/) | Implementation scaffolds (gateway, harness, frontend, schema loader, Redis events, workflow…). **Files with ⚠️ banners are superseded/historical — do not implement from them.** |
| [`docs/analysis/`](docs/analysis/) | The planning layer — project analysis (gaps G1–G12), **scope decisions (D1–D16, all confirmed)**, implementation plan (Phases 0–6), confirmed repo layout |

> Note: `spec/all_spec.md` and `architecture/agent_dashboard_architecture_all.md` are generated pre-decision exports kept for reference — the individual files are authoritative.

## Roadmap

| Phase | Scope | Mode |
|---|---|---|
| **P0** | Monorepo scaffold, compose + env, scripts, CI | sequential |
| **P1** | `contracts` + `agent-schemas` packages | sequential |
| **P2 / P3 / P4** | Gateway ∥ Agent Harness + seed agents ∥ Frontend | **parallel tracks** |
| **P5** | End-to-end smoke, HITL e2e, README quickstart finalized | sequential |
| **P6** | Post-MVP: workflow engine, Mongo wiring, server-side replay, Python harness (uv), auth | backlog |

Details, per-part deliverables and verification commands: [`docs/analysis/implementation_plan.md`](docs/analysis/implementation_plan.md).