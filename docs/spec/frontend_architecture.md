# Frontend Architecture

## Framework
React + RxJS

---

## Event Bus
- Global dispatcher: Subject<Event>
- Session streams: ReplaySubject<Event>
- Operators:
  - filter
  - scan
  - distinctUntilChanged
  - shareReplay
  - switchMap
  - takeUntil
  - merge
  - combineLatest

---

## Component Architecture
Components are session-bound:
- AgentInputForm
- AgentOutputViewer
- AgentLogViewer
- AgentControlPanel
- AgentStateIndicator
- AgentMetadataPanel

Each receives:
```ts
{ sessionId: string }
```
---

## Tab Model
```ts
type Tab = {
  id: string;
  title: string;
  sessionId: string;
  uiState: {
    activePanel: string;
    scrollPositions: Record<string, number>;
    expandedSections: Record<string, boolean>;
  };
};
```

---

## Folder Structure

```text
/src
  /components
  /state/eventBus
  /workflow
  /api
  /routes
```
```