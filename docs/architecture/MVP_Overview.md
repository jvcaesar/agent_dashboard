# MVP Overview

## Vision

A browser-style agent dashboard where each tab represents an agent session:
- Simple, friendly UI (like a browser, not an IDE)
- RxJS-powered event bus for streaming, logs, and state
- Future-ready for a workflow builder that reuses the same sessions and components

## Core Concepts

- **Agent**: A configurable unit of work with input schema, output schema, and capabilities.
- **Session**: A single run context for an agent, with lifecycle, logs, and outputs.
- **Tab**: A UI view bound to a session.
- **Event Bus**: RxJS-based pub/sub system distributing session events.
- **Workflow Node (future)**: Another view bound to a session, rendered on a canvas.

## MVP Scope

- Browser-style UI with:
  - Tab bar
  - Agent list sidebar
  - Session workspace (input, output, logs, controls)
- RxJS event bus for:
  - Session lifecycle events
  - Logs
  - Partial and final outputs
- Session cloning and tab duplication
- Workflow builder preparation (models, hooks, folder structure)
