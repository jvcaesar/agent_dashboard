# Suggested Folder Structure

> **Note (2026-09-05, G6/D15):** this document describes the **frontend tree only**. The authoritative **monorepo-wide layout** (`packages/contracts`, `apps/gateway`, `services/agent-harness`, `infra/`, `scripts/`) is [`docs/analysis/repo_layout.md`](../analysis/repo_layout.md) — confirmed as the target.

```text
/src
  /architecture   // optional, for these docs
  /components
    /tabs
    /session
    /workflow
  /state
    eventBus/
  /workflow
    canvas/
    nodes/
    edges/
    models/
    services/
  /api
    backendClient.ts
  /pages or /routes
    Dashboard.tsx
