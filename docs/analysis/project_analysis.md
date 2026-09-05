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
| 2 | Session lifecycle: idle → running → streaming → waiting → completed/error | ✅ core (D11 confirmed) |
| 3 | Real-time events over SSE → RxJS bus (logs, partial/final output, state changes) | ✅ core |
| 4 | Dynamic input form generated from agent schema | ✅ core |
| 5 | Session cloning / tab duplication | ✅ core |
| 6 | Human-in-the-loop approval (agent pauses, user chooses, agent resumes) | ✅ core (confirmed in MVP) |
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
| Session state machine + UI mapping | `spec/session_state_machine.json`, `architecture/Session_Lifecycle.md` | ✅ Defined; state names reconciled 2026-09-05 — `error` + `streaming` canonical (G1 applied) |
| Event payloads | `spec/event_payloads.json` (canonical D6 model, restructured 2026-09-05 incl. approval events), `scaffolds/Redis_Event_Schema.md` (channel map; `task.*` internal) | ✅ Defined — single canonical dialect |
| Backend API contract | `spec/backend_api_contract.md` | ⚠️ **Superseded** (banner applied 2026-09-05, D2/D3/D7); canonical API lives in `packages/contracts` |
| Agent runtime contract | `spec/agent_runtime_contract.md` | ✅ Strong: `streamEvents` / `invoke` / `abort`, event ordering guarantees |
| Agent schema format | `spec/agent_schema_format.json` (authoritative), `scaffolds/Agent_Schema_Loader.md` (old format bannered 2026-09-05), `spec/seed_agents.md` (concrete seed schemas) | ✅ Single format (D8) |
| Frontend architecture (event bus, tab model, cloning, components, wireframes) | `spec/frontend_architecture.md`, `scaffolds/Frontend_Scaffold.md`, `architecture/EventBus_RxJS.md`, `architecture/Tab_Model.md`, `architecture/Cloning_Model.md`, `architecture/Component_Architecture.md`, `architecture/Wireframes.md` | ✅ Comprehensive |
| SSE client | `scaffolds/Frontend_SSE_Client_Scaffold.md` | ✅ Defined (replaces WebSocket) |
| Gateway (Fastify, Redis, fan-out, approvals) | `scaffolds/Gateway_Scaffold.md`, `scaffolds/Session_Event_Fan‑Out.md`, `scaffolds/New_Session_Flow.md` | ✅ Scaffold exists; routes reconciled to D7 on 2026-09-05 (scope note; `/tasks` removed from server.ts snippet) |
| Agent Harness (Node + Python outline, snapshot, HITL pause/resume) | `scaffolds/Agent_Harness_Scaffold.md`, `spec/seed_agents.md` (D16) | ✅ Structure complete; concrete agents specced (echo + calculator) — execution pending (Phase 3) |
| Agent Schema Loader | `scaffolds/Agent_Schema_Loader.md` | ✅ Defined; field-list format bannered superseded (D8) |
| Workflow engine / orchestration / persistence | `spec/workflow_engine_interface.md`, `scaffolds/Workflow_Engine_Scaffold.md`, `scaffolds/Workflow_Orchestration.md`, `scaffolds/Workflow_Persistence_Layer.md` | ✅ Fully specced — recommended **post-MVP** (see D10), so kept as future contract |
| MongoDB state store | `scaffolds/Task_and_Session_State_Store_(MongoDB).md` | ✅ Models defined; state enum reconciled 2026-09-05 (G1); MVP runs in-memory (D9) |
| Redis event schema & channels | `scaffolds/Redis_Event_Schema.md` | ✅ Defined; `session.stateChanged` enum aligned 2026-09-05 (G5) |
| Folder layout | `analysis/repo_layout.md` (**confirmed** monorepo layout, D15), `architecture/Folder_Structure.md` (frontend tree; pointer note added 2026-09-05) | ✅ Defined |
| Repo scaffolding, env, infra, scripts, CI | `analysis/repo_layout.md` (confirmed, D15), `implementation_plan.md` Phase 0 | ✅ Defined — execution pending (Phase 0) |
| Seed agents | `spec/seed_agents.md` (D16), `spec/agent_runtime_contract.md` | ✅ Defined — echo + calculator; execution pending (Phase 3) |

---

## 4. Gaps & Inconsistencies (must resolve before/during implementation)

| ID | Issue | Where | Impact | Fix |
|---|---|---|---|---|
| G1 | Session state set inconsistent: spec machine has `error` + `streaming`; MongoDB `Session` enum uses `failed` and has **no** `streaming`; Redis `session.stateChanged` payload lists `failed` | `spec/session_state_machine.json` vs MongoDB state store scaffold vs `scaffolds/Redis_Event_Schema.md` | Backend/frontend state logic disagree; state indicators break | ✅ Applied 2026-09-05: canonical set adopted; `failed`→`error` + `streaming` fixed in Redis_Event_Schema + Mongo store (D5) |
| G2 | **Two backend architectures** documented: sessions-monolith (Express + WebSocket) vs gateway+harness (Fastify + SSE + Redis); the latter explicitly supersedes (`Frontend_SSE_Client_Scaffold`: "replaces the WebSocket client"; `Gateway_Scaffold`: old backend scaffold "now plays the role of an Agent Runtime Service") | `spec/backend_api_contract.md`, `scaffolds/Backend_Scaffold.md` vs `scaffolds/Gateway_Scaffold.md`, `scaffolds/Product_Description_*.md` | Conflicting endpoints, transports, ownership; double work | ✅ Applied 2026-09-05: Gateway+Harness is the architecture; monolith docs bannered superseded (D2/D3) |
| G3 | Canonical endpoint set not agreed: `/sessions/create`+`/run`+`/stop` (backend spec) vs `/tasks` + `/approvals/:taskId` (Gateway scaffold); `/agents/list` & `/agents/:id/schema` only in backend spec | `spec/backend_api_contract.md` vs `scaffolds/Gateway_Scaffold.md` | Murky API contract; frontend client can't be written against a single contract | ✅ Confirmed (D7): sessions-centric contract; Gateway scaffold `/tasks` routes annotated superseded 2026-09-05 |
| G4 | **Two agent-schema formats**: field-list (loader example: `{ input: [{id,label,type,required,default}] }`) vs JSON-Schema-ish (`inputSchema.properties…`); API returns `{inputSchema,outputSchema,configSchema}` | `scaffolds/Agent_Schema_Loader.md` vs `spec/agent_schema_format.json`, `spec/backend_api_contract.md` | Dynamic forms & validation built on wrong format; rework | ✅ Confirmed (D8): JSON Schema + `x-ui-*`; loader format bannered superseded; concrete seed schemas in `spec/seed_agents.md` |
| G5 | **Task vs session relationship fuzzy**: harness scaffold uses `sessionId = taskId`; `task.created` `sessionId` optional; Gateway task route doesn't propagate `sessionId`; two event dialects (flat payloads in spec vs `payload`-wrapped in Redis schema) | `scaffolds/Agent_Harness_Scaffold.md`, `scaffolds/Gateway_Scaffold.md`, `spec/event_payloads.json`, `scaffolds/Redis_Event_Schema.md` | Fan-out/SSE can misfire; frontend parser must guess shapes | ✅ Applied 2026-09-05: `event_payloads.json` restructured to the canonical `payload`-wrapped model + approval events (D4/D6/D11) |
| G6 | **No repo/runtime scaffolding**: no root `package.json`/workspaces, no `docker-compose` (Redis/Mongo), no `.env.example`, no dev/run/test scripts, no README | `docs/` only; `architecture/Folder_Structure.md` is frontend-only | Nothing can be run or verified end-to-end — **blocking** | ✅ Resolved (definition): target layout confirmed in `repo_layout.md` (npm workspaces, D15); execution is Phase 0 work |
| G7 | **No concrete seed agents / real runtimes**: only `mockAgentRuntime`; `AgentRegistry` is empty stub | `scaffolds/Backend_Scaffold.md`, `scaffolds/Agent_Harness_Scaffold.md` | Sidebar, forms, streaming can't be demoed honestly; no test harness realism | ✅ Resolved (definition): echo + calculator confirmed (D16), specced in `spec/seed_agents.md`; execution is Phase 3 work |
| G8 | **No test/verification strategy** (and no CI): contracts exist but no unit/integration/e2e approach or commands | throughout | "Verifiable parts" impossible without defined checks | ✅ Resolved (definition): per-layer strategy + commands defined in `implementation_plan.md` §"Verification toolbox"; CI ships with Phase 0.3 |
| G9 | **No HITL consumer wiring in MVP docs**: `approval.requested/resumed` events defined; but no UI panel or endpoint consistently included in MVP scope | `Redis_Event_Schema.md`, `architecture/Session_Lifecycle.md` (waiting state) vs no approval component in wireframes | Waiting sessions would strand users | ✅ Resolved: `AgentApprovalPanel` + `POST /sessions/:id/approve` confirmed in MVP (D11) |
| G10 | Session cloning spec assumes snapshot obtainable from backend, but no endpoint exists (`GET /sessions/:id/snapshot` absent everywhere) | `architecture/Cloning_Model.md` vs backend/API docs | Frontend cannot clone sessions as specced | ✅ Resolved: client-side `SessionSnapshot` confirmed for MVP; backend endpoint post-MVP (D12) |
| G11 | Minor: SSE replay/hydration after page reload undefined (ReplaySubject is per-tab, in-memory only) | `EventBus_RxJS.md` vs `Session_Lifecycle.md` | Confusion about "open tab mid-session" UX | ✅ Resolved: reload = fresh dashboard + live tailing only, confirmed for MVP (D13); server-side replay buffer post-MVP |
| G12 | Minor: ports/URLs scattered across docs (3000 vs 3001, WS vs SSE paths) | several scaffolds | Dev friction | ✅ Resolved (definition): `.env.example` + env defaults specified in `repo_layout.md` §3 (D14); doc scan found no stray port refs |

---

## 5. Missing Elements — the verdict

**The product definition is now complete.** Architecture, components, all scope conflicts, and both former hard blockers (repo foundation, seed agents) are resolved by decision (see `scope_decisions.md`, confirmed 2026-09-05). What remains is execution, not definition:

1. **✅ Resolved (definition) — repo/runtime foundation (G6)**: target monorepo layout confirmed (`repo_layout.md`; npm workspaces per D15). Execution = Phase 0.
2. **✅ Resolved (definition) — concrete seed agents (G7)**: echo + calculator confirmed (D16), specced in `spec/seed_agents.md`. Execution = Phase 3.
3. **✅ Resolved — conflict resolution (G1–G5)**: two generations of design coexist (monolith/WS/field-list vs gateway/SSE/JSON-Schema); the newer generation was adopted via decisions D2–D8 in `scope_decisions.md` (confirmed 2026-09-05). Old docs were bannered superseded on 2026-09-05.
4. **✅ Resolved — MVP-scope leftovers (G9–G11)**: HITL in MVP (D11), client-side clone snapshots (D12), reload = fresh dashboard (D13) — all confirmed by product owner on 2026-09-05.
5. **Missing but intentionally out of scope (not gaps)**: workflow engine implementation, orchestration, persistence, auth, Python harness — all have interfaces/contracts already, so they can be added later without major changes (D1/D10).

**Post-MVP hooks that make "add later without major changes" true**: canonical event model is additive (D6); `task`/`runId` promotion is additive (D4); workflow engine implements the already-specced interface behind the same events; Mongo store models exist as scaffolds (D9); SSE fan-out already routes per-session channels (D13).