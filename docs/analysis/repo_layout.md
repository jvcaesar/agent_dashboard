# Proposed Repository Layout (Actor-based, npm workspaces)

> Companion: [`project_analysis.md`](./project_analysis.md), [`scope_decisions.md`](./scope_decisions.md) (D15), [`implementation_plan.md`](./implementation_plan.md) (Phase 0 creates this).
> Status: recommended snapshot for Day‑1 scaffolding. Derived from `docs/scaffolds/*` + `docs/spec/*`, reconciling the conflicts (G1–G12) per scope decisions.

---

## 1. Target tree

```text
agent-dashboard/
├─ package.json                       # npm workspaces root; scripts: dev/build/test/lint/typecheck/smoke
├─ tsconfig.base.json                 # strict TS defaults shared by all workspaces
├─ .gitignore / .editorconfig / .env.example  # env template (see §3); .env (git-ignored)
├─ README.md                         # quickstart (created in P5.3; outline in P0.3)
├─ docs/                             # existing spec/architecture/scaffolds/analysis docs; annotate the 2 monolith files "superseded" per D3
│
├─ packages/
│  ├─ contracts/
│  │   ├─ src/
│  │   │   ├─ index.ts              # re-export everything (single public surface)
│  │   │   ├─ events.ts             # canonical Event trait + SessionEvent variants (D6)
│  │   │   ├─ sessionState.ts       # SessionState set + transition table (D5; spec/session_state_machine.json)
│  │   │   ├─ api.ts                # request/response/SSE types (D7)
│  │   │   ├─ entities.ts           # Session, Tab, AgentDescriptor (D4)
│  │   │   └─ env.ts                # default ports/env constants (D14)
│  │   ├─ test/                     # transition-table + type-guard tests
│  │   └─ package.json              # name @agent/contracts
│  └─ agent-schemas/
│      ├─ src/                       # JSON Schema types, ajv validators (D8)
│      │   ├─ format.ts              # meta-schema for agent schema files
│      │   ├─ validator.ts           # validateInput/validateConfig
│      │   └─ registry.ts            # cached getAgentSchema(agentId)
│      ├─ schemas/
│      │   ├─ echo.json              # seed agent 1 (also smoke-test default)
│      │   └─ calculator.json        # seed agent 2 (HITL demo w/ approvalRequired config)
│      └─ package.json               # name @agent/agent-schemas
│
├─ apps/
│  ├─ gateway/
│  │   ├─ src/
│  │   │   ├─ server.ts            # Fastify boot (port from env)
│  │   │   ├─ app.ts               # build app (routes, CORS, pino, error handler)
│  │   │   ├─ routes/
│  │   │   │   ├─ sessions.ts     # create/run/stop/approve (D7)
│  │   │   │   ├─ agents.ts       # /agents/list, /:agentId/schema
│  │   │   │   └─ health.ts       # GET /health
│  │   │   ├─ redis/
│  │   │   │   ├─ pub.ts         # ioredis publisher
│  │   │   │   └─ sub.ts         # subscriber → fan-out (Session_Event_Fan‑Out)
│  │   │   ├─ sse/
│  │   │   │   └─ eventStream.ts # SSE helpers: headers, heartbeat, cleanup
│  │   │   ├─ events/
│  │   │   │   └─ fanout.ts      # agent.logs/agent.events → session:{sid}:events
│  │   │   └─ store/
│  │   │       └─ sessionStore.ts # in-memory Map + TTL sweep (D9/D13)
│  │   └─ package.json           # name @agent/gateway
│  └─ frontend/
│      ├─ src/
│      │   ├─ main.tsx / App.tsx  # dashboard shell (Wireframes.md)
│      │   ├─ api/
│      │   │   ├─ backendClient.ts # createSession/run/stop/approve/listAgents/getAgentSchema
│      │   │   ├─ eventStreamClient.ts # SSE → routeEvent (Frontend_SSE_Client_Scaffold)
│      │   │   └─ mockTransport.ts # dev-only injectable mock events
│      │   ├─ state/
│      │   │   ├─ eventBus.ts      # Subject + ReplaySubject(100), routeEvent, sessionEvents
│      │   │   └─ sessionStreams.ts # logs/partial/final/state selectors
│      │   ├─ models/
│      │   │   └─ Tab.ts / SessionViewModel.ts
│      │   ├─ components/
│      │   │   ├─ tabs/TabBar.tsx, Tab.tsx
│      │   │   ├─ session/AgentHeader, AgentInputForm, AgentOutputViewer,
│      │   │   │          AgentLogViewer, AgentControlPanel, AgentStateIndicator,
│      │   │   │          AgentMetadataPanel, AgentApprovalPanel
│      │   │   ├─ layout/Sidebar.tsx, Workspace.tsx
│      │   │   └─ workflow/WorkflowCanvasPlaceholder.tsx  # "Coming Soon" (D10)
│      │   └─ styles/              # CSS modules (wireframe class names)
│      └─ package.json             # name @agent/frontend; Vite settings (port 5173)
│
├─ services/
│  └─ agent-harness/
│      ├─ src/
│      │   ├─ worker.ts           # entry: redis sub (tasks channel)
│      │   ├─ redis/
│      │   │   └─ sub.ts / pub.ts
│      │   ├─ events/emit.ts      # canonical-event publishers (emitLog, output, state, approval)
│      │   ├─ runtime/
│      │   │   ├─ AgentRuntime.ts # interface per spec/agent_runtime_contract.md
│      │   │   ├─ echoAgent.ts    # streaming test agent (default smoke)
│      │   │   ├─ calculatorAgent.ts # + HITL approval flow (P3.3)
│      │   │   └─ stateMachine.ts # task.created → run; task.resume → resume (sessionId threaded)
│      │   └─ store/pendingApprovals.ts # in-memory (D9)
│      └─ package.json            # name @agent/agent-harness
│
├─ infra/
│  └─ docker-compose.yml        # redis (mongo commented; volumes, healthcheck)
│
├─ scripts/
│  └─ smoke.mjs                 # e2e smoke: gateway+harness+SSE assertions (P5.1)
│
└─ .github/
   └─ workflows/ci.yml          # lint, typecheck, test, build
```

---

## 2. Workspace map

| Package | Path | Role | Depends on | Entry / Port |
|---|---|---|---|---|
| @agent/contracts | `packages/contracts` | canonical TS types + validators; single source of truth (D6) | none | library |
| @agent/agent-schemas | `packages/agent-schemas` | agent JSON-schema files + ajv validation + registry (D8) | contracts (types) | library |
| @agent/gateway | `apps/gateway` | HTTP/SSE API, Redis bridge, fan-out (D7) | contracts, agent-schemas | `:3001`, `GET /health` |
| @agent/agent-harness | `services/agent-harness` | Redis worker executing agents, emitting events (D3) | contracts, agent-schemas | no HTTP; Redis sub/pub |
| @agent/frontend | `apps/frontend` | React+RxJS dashboard (tabs, forms, SSE) | contracts, agent-schemas (types) | Vite `:5173` |

Redis `:6379`, Mongo (optional) `:27017` — provisioned by `infra/docker-compose.yml`. All ports/URLs come from `.env`; `packages/contracts/src/env.ts` holds defaults (D14).

---

## 3. Environment variables (`.env.example` template)

| Var | Default | Used by | Notes |
|---|---|---|---|
| `PORT` | `3001` | gateway | HTTP + SSE listen port (D14) |
| `REDIS_URL` | `redis://localhost:6379` | gateway, harness | ioredis connection (both pub+sub) |
| `MONGO_URL` | `mongodb://localhost:27017/agent_system` | (optional) gateway/harness | unused in MVP default (D9) |
| `FRONTEND_PORT` | `5173` | frontend | Vite dev server (`.env` optional; Vite reads `VITE_*` via `import.meta.env`) |
| `VITE_API_URL` | `http://localhost:3001` | frontend | backendClient + EventSource base URL |
| `LOG_LEVEL` | `info` | gateway, harness | pino level |
| `SESSION_TTL_MS` | `60000` | gateway | in-memory sweep grace after client disconnect (D13) |

---

## 4. Scripts (root `package.json`)

| Script | What it does |
|---|---|
| `npm run dev` | concurrent gateway+harness+frontend (e.g. `concurrently` or manual per-workspace) |
| `npm run dev -w @agent/gateway` (etc.) | single process |
| `npm run build` | build all workspaces (tsc; Vite build for frontend) |
| `npm run test` | Vitest across workspaces |
| `npm run lint` / `npm run typecheck` | eslint / `tsc -b` |
| `npm run smoke` | `scripts/smoke.mjs` e2e (P5.1) |

---

## 5. Conventions

- **Names** come only from `packages/contracts` — no literal state/event/endpoint strings sprinkled in apps (G1–G5 prevention).
- **Comments** reference decision IDs (`// D6: canonical event`) so doc↔code stays traceable.
- **Post-MVP placeholders** live as stubs (`WorkflowCanvasPlaceholder.tsx`, empty snapshot hooks) — visible but inert (D10).
- **Superseded docs** (`spec/backend_api_contract.md`, `scaffolds/Backend_Scaffold.md`, field-list loader sections) get a banner: "Superseded — see Gateway_Scaffold / scope_decisions D3/D7/D8" (P0.3 doc pass).