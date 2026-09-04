# Session Cloning Model

## Snapshot

Clone captures **static** session data:

```ts
type SessionSnapshot = {
  agentId: string;
  inputDraft: Record<string, any>;
  config: Record<string, any>;
  uiState: {
    activePanel: string;
    scrollPositions: Record<string, number>;
    expandedSections: Record<string, boolean>;
  };
  metadata: Record<string, any>;
};
```

## New Session

```ts
const newSession: Session = {
  id: uuid(),
  agentId: snapshot.agentId,
  state: "idle",
  inputHistory: [],
  outputHistory: [],
  logs: [],
  metadata: snapshot.metadata
};
```

- No logs, output, errors, or runtime state are copied.
- New `ReplaySubject<Event>` for `newSession.id`.

## New Tab

```ts
const newTab: Tab = {
  id: newSession.id,
  title: originalTab.title + " (clone)",
  sessionId: newSession.id,
  uiState: snapshot.uiState
};
```

## UX Rules

- Clone starts in `idle`
- Input + config preserved
- Logs/output cleared
- Optional “Clone and Run” vs “Clone Only” (default: “Clone Only”)
```
