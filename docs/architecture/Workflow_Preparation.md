
---

### `Workflow_Preparation.md`

```markdown
# Workflow Builder Preparation

## Node Model

```ts
type WorkflowNodeType = "agent" | "transform" | "trigger" | "branch";

type WorkflowNode = {
  id: string;
  type: WorkflowNodeType;
  sessionId: string | null;
  config: Record<string, any>;
  position: { x: number; y: number };
  inputs: Port[];
  outputs: Port[];
};
```

## Event Bus Hooks

- Nodes subscribe to `sessionEvents(sessionId)`
- Edges implemented via RxJS operators:
  - sequential: `switchMap`
  - parallel: `merge`
  - conditional: `filter`
  - multi-input sync: `combineLatest`
  - transform: `map`

## Folder Structure (Stub)

```text
/workflow
  /canvas
    WorkflowCanvas.tsx
  /nodes
    AgentNode.tsx
    TransformNode.tsx
    BranchNode.tsx
  /edges
    WorkflowEdge.ts
  /models
    WorkflowNode.ts
    WorkflowEdge.ts
  /services
    workflowEngine.ts
```

## Node Inspector Reuse

- Right sidebar from tabs reused as node inspector
- Same components: `AgentControlPanel`, `AgentMetadataPanel`, etc.
- Same `sessionId` binding
```
