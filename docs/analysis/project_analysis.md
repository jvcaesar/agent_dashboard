# Project Analysis — Agent Dashboard MVP

> Planning review of `docs/`. Source of truth: Markdown/JSON outside `docs/pdf-files/` (per `docs/AGENTS.md`).
> Companion docs: [`implementation_plan.md`](./implementation_plan.md), [`scope_decisions.md`](./scope_decisions.md), [`repo_layout.md`](./repo_layout.md).

---

## 1. Purpose of the Project

The **Agent Dashboard** is an event-driven, browser-style dashboard for running, observing, and controlling **AI agents**. Each browser tab represents exactly **one agent session**, with real-time streaming of logs, partial/final output, and lifecycle state over a reactive event bus. It is deliberately architected so that the same session-bound components and event bus can later be reused inside a **multi-agent workflow builder** without refactoring.

```text
Frontend (React + RxJS, tab per session)
   │   HTTP + SSE
   ▼
Gateway (Fastify: sessions, SSE, Redis bridge)
   │   Redis Pub/Sub
   ▼
Agent Harness (Node/Python: executes agents, emits events)
   │   optional
   ▼
MongoDB (state store: tasks, sessions, workflow state — optional in MVP)
   │   post-MVP
   ▼
Workflow Engine (RxJS graph: runs/stops/resumes workflows)
```

The "product description" (`docs/scaffolds/Product_Description_Event‑Driven_Multi‑Agent_Platform_MVP.md`) adds the distributed framing: a platform for real-time agent execution, human-in-the-loop decision points, multi-agent workflows, dynamic agent schemas, session-based UI, and polyglot (Node + Python) agent support.

---

## 2. MVP Goal (What "done" looks like)

| # | Capability | In MVP? |
|---|---|---|
| 1 | Browser-style UI: tab bar, agent list sidebar, session workspace (input, output, logs, controls) | ✅ core |
| 2 | Session lifecycle: idle → running → streaming → completed/error, with `waiting` (approval) | ✅ core (waiting only if HITL kept; see D11) |
| 3 | Real-time events over SSE → RxJS bus (logs, partial/final output, state changes) | ✅ core |
| 4 | Dynamic input form generated from agent schema | ✅ core |
| 5 | Session cloning / tab duplication | ✅ core |
| 6 | Human-in-the-loop approval (agent pauses, user chooses, agent resumes) | ⚠️ recommended MVP (small; can defer) |
| 7 | Run / Stop / Restart / Rerun controls | ✅ core |
| 8 | Workflow builder | ❌ post-MVP (placeholder + interfaces only) |
| 9 | Multi-agent workflows / orchestration / persistence | ❌ post-MVP |
| 10 | Python harness | ❌ post-MVP (interface already defined) |
| 11 | Auth, metrics, distributed scaling, marketplace | ❌ post-MVP |

---

## 3. What Is Already Defined (and where)

| Concern | Defined in | Assessment |
|---|---|---|
| Product vision & high-level architecture | `scaffolds/Product_Description_*.md`, `architecture/MVP_Overview.md` | ✅ Consistent vision; two complementary framings (tab-dashboard vs distributed platform) |
| Session state machine + UI mapping | `spec/session_state_machine.json`, `architecture/Session_Lifecycle.md` | ✅ Defined — but **inconsistent state names** elsewhere (`failed` vs `error`; see G1) |
| Event payloads | `spec/event_payloads.json`, `scaffolds/Redis_Event_Schema.md`, `architecture/Session_Lifecycle.md` | ✅ Defined — two dialects (session events vs task events; see G5) |
| Backend API contract | `spec/backend_api_contract.md` | ⚠️ Partially **superseded** by Gateway+SSE direction (see G2/G3) |
| Agent runtime contract | `spec/agent_runtime_contract.md` | ✅ Strong: `streamEvents` / `invoke` / `abort`, event ordering guarantees |
| Agent schema format | `spec/agent_schema_format.json`, `scaffolds/Agent_Schema_Loader.md` | ⚠️ **Two different formats** (see G4) |
| Frontend architecture (event bus, tab model, cloning, components, wireframes) | `spec/frontend_architecture.md`, `scaffolds/Frontend_Scaffold.md`, `architecture/EventBus_RxJS.md`, `architecture/Tab_Model.md`, `architecture/Cloning_Model.md`, `architecture/Component_Architecture.md`, `architecture/Wireframes.md` | ✅ Comprehensive |
| SSE client | `scaffolds/Frontend_SSE_Client_Scaffold.md` | ✅ Defined (replaces WebSocket) |
| Gateway (Fastify, Redis, fan-out, approvals) | `scaffolds/Gateway_Scaffold.md`, `scaffolds/Session_Event_Fan‑Out.md`, `scaffolds/New_Session_Flow.md` | ✅ Scaffold exists — endpoints need reconciliation (G2/G3) |
| Agent Harness (Node + Python outline, mock runtime, snapshot, HITL pause/resume) | `scaffolds/Agent_Harness_Scaffold.md` | ✅ Structure complete — **only a mock runtime**; no concrete agents (G7) |
| Agent Schema Loader | `scaffolds/Agent_Schema_Loader.md` | ✅ Defined (format conflict above) |
| Workflow engine / orchestration / persistence | `spec/workflow_engine_interface.md`, `scaffolds/Workflow_Engine_Scaffold.md`, `scaffolds/Workflow_Orchestration.md`, `scaffolds/Workflow_Persistence_Layer.md` | ✅ Fully specced — recommended **post-MVP** (see D10), so kept as future contract |
| MongoDB state store | `scaffolds/Task_and_Session_State_Store_(MongoDB,.md` | ✅ Models defined — MVP can run in-memory (D9) |
| Redis event schema & channels | `scaffolds/Redis_Event_Schema.md` | ✅ Defined (needs alignment with canonical event model, G5) |
| Folder layout | `architecture/Folder_Structure.md` (frontend-only stub), `spec/frontend_architecture.md` | ⚠️ Missing **monorepo-level** layout (see G6 and `repo_layout.md`) |
| Repo scaffolding, env, infra, tests, CI, seed agents | — | ❌ **Not defined anywhere** — blocking |

---

## 4. Gaps & Inconsistencies (must resolve before/during implementation)

| ID | Issue | Where | Impact | Fix |
|---|---|---|---|---|
| G1 | Session state set inconsistent: spec machine has `error` + `streaming`; MongoDB `Session` enum uses `failed` and has **no** `streaming`; Redis `session.stateChanged` payload lists `failed` | `spec/session_state_machine.json` vs MongoDB state store scaffold vs `scaffolds/Redis_Event_Schema.md` | Backend/frontend state logic disagree; state indicators break | Adopt one set: `idle \| running \| streaming \| waiting \| completed \| error`; keep separate `task.status` enum (D5) |
| G2 | **Two backend architectures** documented: sessions-monolith (Express + WebSocket) vs gateway+harness (Fastify + SSE + Redis); the latter explicitly supersedes (`Frontend_SSE_Client_Scaffold`: "replaces the WebSocket client"; `Gateway_Scaffold`: old backend scaffold "now plays the role of an Agent Runtime Service") | `spec/backend_api_contract.md`, `scaffolds/Backend_Scaffold.md` vs `scaffolds/Gateway_Scaffold.md`, `scaffolds/Product_Description_*.md` | Conflicting endpoints, transports, ownership; double work | Implement **Gateway + Harness + SSE + Redis**; annotate monolith docs "superseded" (D2/D3) |
| G3 | Canonical endpoint set not agreed: `/sessions/create`+`/run`+`/stop` (backend spec) vs `/tasks` + `/approvals/:taskId` (Gateway scaffold); `/agents/list` & `/agents/:id/schema` only in backend spec | `spec/backend_api_contract.md` vs `scaffolds/Gateway_Scaffold.md` | Murky API contract; frontend client can't be written against a single contract | Single sessions-centric contract in `packages/contracts`; optional `/sessions/:id/approve` for HITL (D7) |
| G4 | **Two agent-schema formats**: field-list (loader example: `{ input: [{id,label,type,required,default}] }`) vs JSON-Schema-ish (`inputSchema.properties…`); API returns `{inputSchema,outputSchema,configSchema}` | `scaffolds/Agent_Schema_Loader.md` vs `spec/agent_schema_format.json`, `spec/backend_api_contract.md` | Dynamic forms & validation built on wrong format; rework | One canonical format (JSON Schema draft-07 subset + UI metadata), single validator in `packages/agent-schemas` (D8) |
| G5 | **Task vs session relationship fuzzy**: harness scaffold uses `sessionId = taskId`; `task.created` `sessionId` optional; Gateway task route doesn't propagate `sessionId`; two event dialects (flat payloads in spec vs `payload`-wrapped in Redis schema) | `scaffolds/Agent_Harness_Scaffold.md`, `scaffolds/Gateway_Scaffold.md`, `spec/event_payloads.json`, `scaffolds/Redis_Event_Schema.md` | Fan-out/SSE can misfire; frontend parser must guess shapes | `sessionId` first-class on **every** event; `taskId` optional correlation only; one canonical event model with `payload` wrapper; Redis & SSE are encodings of same model (D4/D6) |
| G6 | **No repo/runtime scaffolding**: no root `package.json`/workspaces, no `docker-compose` (Redis/Mongo), no `.env.example`, no dev/run/test scripts, no README | `docs/` only; `architecture/Folder_Structure.md` is frontend-only | Nothing can be run or verified end-to-end — **blocking** | Phase 0: monorepo + compose + env + scripts (see `repo_layout.md`, `implementation_plan.md` Phase 0) |
| G7 | **No concrete seed agents / real runtimes**: only `mockAgentRuntime`; `AgentRegistry` is empty stub | `scaffolds/Backend_Scaffold.md`, `scaffolds/Agent_Harness_Scaffold.md` | Sidebar, forms, streaming can't be demoed honestly; no test harness realism | Seed ≥2 agents (echo, calculator) with schema files registered in harness; contract-compliant runtime (Phase 3) |
| G8 | **No test/verification strategy** (and no CI): contracts exist but no unit/integration/e2e approach or commands | throughout | "Verifiable parts" impossible without defined checks | Per-layer strategy + verification commands in `implementation_plan.md` §"Verification toolbox" |
| G9 | **No HITL consumer wiring in MVP docs**: `approval.requested/resumed` events defined; but no UI panel or endpoint consistently included in MVP scope | `Redis_Event_Schema.md`, `architecture/Session_Lifecycle.md` (waiting state) vs no approval component in wireframes | Waiting sessions would strand users | Add `AgentApprovalPanel` + `POST /sessions/:id/approve` to MVP (D11) |
| G10 | Session cloning spec assumes snapshot obtainable from backend, but no endpoint exists (`GET /sessions/:id/snapshot` absent everywhere) | `architecture/Cloning_Model.md` vs backend/API docs | Frontend cannot clone sessions as specced | MVP: build `SessionSnapshot` client-side from live session state (input/config/uiState); backend endpoint post-MVP (D12) |
| G11 | Minor: SSE replay/hydration after page reload undefined (ReplaySubject is per-tab, in-memory only) | `EventBus_RxJS.md` vs `Session_Lifecycle.md` | Confusion about "open tab mid-session" UX | Document MVP behavior: reload = fresh dashboard; live tailing only (D13); server-side replay buffer post-MVP |
| G12 | Minor: ports/URLs scattered across docs (3000 vs 3001, WS vs SSE paths) | several scaffolds | Dev friction | Single `.env.example` + `packages/contracts/src/env.ts` defaults (D14) |

---

## 5. Missing Elements — the verdict

**The architecture and components needed for the MVP goal are defined — but the docs alone are not enough to start coding.** The design is ~80% complete, blocked by 2 hard-blocking gaps and a set of conflicts:

1. **Blocking — repo/runtime foundation (G6)**: no monorepo, no infra compose, no env, no scripts, no CI. *Fix: Phase 0 in `implementation_plan.md`; target layout in `repo_layout.md`.*
2. **Blocking — concrete seed agents (G7)**: only a mock runtime exists; nothing to demo the core loop (form → run → stream → complete). *Fix: echo + calculator agents in Phase 3.*
3. **Soft — conflict resolution (G1–G5)**: two generations of design coexist (monolith/WS/field-list vs gateway/SSE/JSON-Schema) disagreeing on states, endpoints, schemas, and the task/session relationship. *Fix: decisions D2–D8 in `scope_decisions.md`.*
4. **Soft — MVP-scope leftovers (G9–G11)**: HITL wiring, cloning snapshot source, SSE reload behavior are specced but not pinned to MVP. *Fix: D11–D13.*
5. **Missing but intentionally out of scope (not gaps)**: workflow engine implementation, orchestration, persistence, auth, Python harness — all have interfaces/contracts already, so they can be added later without major changes (D1/D10).

**Post-MVP hooks that make "add later without major changes" true**: canonical event model is additive (D6); `task`/`runId` promotion is additive (D4); workflow engine implements the already-specced interface behind the same events; Mongo store models exist as scaffolds (D9); SSE fan-out already routes per-session channels (D13).