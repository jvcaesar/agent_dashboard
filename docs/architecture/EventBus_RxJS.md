
---

### `EventBus_RxJS.md`

```markdown
# Event Bus (RxJS)

## Goals

- Session-scoped event streams
- Multicasting to tabs and workflow nodes
- Replay for late subscribers
- Clean subscription management for tab switching and closing

## Core API

```ts
type Event = {
  sessionId: string;
  type: string;
  payload?: any;
  timestamp: number;
};

interface EventBus {
  publish(event: Event): void;
  sessionEvents(sessionId: string): Observable<Event>;
  globalEvents(): Observable<Event>;
}
```

## Implementation Sketch

- Global dispatcher: `Subject<Event>`
- Session streams: `ReplaySubject<Event>` per `sessionId`
- `sessionEvents(sessionId)` returns `sessionSubjects[sessionId].asObservable().pipe(shareReplay({ bufferSize: N }))`

## Key RxJS Patterns

- `filter()` for event type selection
- `scan()` for accumulating logs/output
- `distinctUntilChanged()` for state indicators
- `shareReplay()` for multicasting
- `switchMap()` for tab switching
- `takeUntil()` for cleanup on tab close
- `merge()` / `combineLatest()` for workflow edges

## Transport

- SSE from backend (`GET /sessions/:sessionId/events`) — WebSocket superseded (D2)
- Each backend message mapped to `Event` and fed into `publish()`
- Replay is tab-scoped: `ReplaySubject(100)` covers late subscribers within the
  same page load; a full page reload starts fresh (D13; server-side backfill
  is post-MVP)
```
