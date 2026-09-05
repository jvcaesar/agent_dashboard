# Agent Dashboard

An event-driven, browser-style dashboard for running, observing, and controlling **AI agents**. Each browser tab represents exactly **one agent session**, streaming logs, partial/final output, and lifecycle state in real time — architected so the same session components can later be reused inside a multi-agent workflow builder.

> **Status: 📐 Product definition complete — implementation not started.**
> This repository currently contains the complete, reconciled specification in [`docs/`](docs/). Code arrives with **Phase 0** of the [implementation plan](docs/analysis/implementation_plan.md). All product decisions (**D1–D16**) are confirmed — see [`docs/analysis/scope_decisions.md`](docs/analysis/scope_decisions.md).

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
| **P5** | End-to-end smoke, HITL e2e, README | sequential |
| **P6** | Post-MVP: workflow engine, Mongo wiring, server-side replay, Python harness (uv), auth | backlog |

Details, per-part deliverables and verification commands: [`docs/analysis/implementation_plan.md`](docs/analysis/implementation_plan.md).