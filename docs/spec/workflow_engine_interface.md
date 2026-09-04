
---

# 6. **`workflow_engine_interface.md`**

```markdown
# Workflow Engine Interface

## Overview
Workflow engine is a reactive graph built on RxJS.

---

## Node Model

```ts
type WorkflowNode = {
  id: string;
  type: "agent" | "transform" | "trigger" | "branch";
  sessionId: string | null;
  config: Record<string, any>;
  position: { x: number; y: number };
  inputs: Port[];
  outputs: Port[];
};
```

---

## Edge Model

```ts
type WorkflowEdge = {
  id: string;
  fromNodeId: string;
  fromPort: string;
  toNodeId: string;
  toPort: string;
  operator: "map" | "merge" | "switchMap" | "combineLatest" | "filter" | null;
};
```

---

## Engine API

```ts
interface WorkflowEngine {
  loadWorkflow(definition: WorkflowDefinition): void;
  start(): void;
  stop(): void;
  subscribe(nodeId: string, callback: (event: Event) => void): void;
}
```

---

## Execution Model

- Nodes subscribe to `sessionEvents(sessionId)`
- Edges transform streams using RxJS operators
- Workflow is a visual RxJS pipeline
```
