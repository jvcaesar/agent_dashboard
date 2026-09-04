
---

### `Component_Architecture.md`

```markdown
# Component Architecture

## Principle

Components are **session-bound** and **reactive**:
- Each major component receives `sessionId`
- Subscribes to `sessionEvents(sessionId)`
- Filters relevant events
- Updates local UI state via React hooks

## Core Components

- `AgentHeader`
- `AgentInputForm`
- `AgentOutputViewer`
- `AgentLogViewer`
- `AgentControlPanel`
- `AgentStateIndicator`
- `AgentMetadataPanel`

## Example: Log Viewer

```ts
function AgentLogViewer({ sessionId }: { sessionId: string }) {
  const [logs, setLogs] = useState<LogEntry[]>([]);

  useEffect(() => {
    const sub = sessionEvents(sessionId)
      .pipe(
        filter(e => e.type === "log.append"),
        scan((acc, e) => [...acc, e.payload], [])
      )
      .subscribe(setLogs);

    return () => sub.unsubscribe();
  }, [sessionId]);

  // render logs...
}
```

## Reuse in Workflow Builder

- Same components used inside workflow nodes
- Nodes pass `sessionId` just like tabs
- No component changes needed when canvas is added
```
