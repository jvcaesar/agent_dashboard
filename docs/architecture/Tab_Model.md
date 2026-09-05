
---

### `Tab_Model.md`

```markdown
# Tab Model and Interaction Patterns

## Tab Object

```ts
type Tab = {
  id: string;          // usually equals sessionId
  title: string;       // agent name or custom label
  sessionId: string;
  uiState: {
    activePanel: "input" | "output" | "logs";
    scrollPositions: Record<string, number>;
    expandedSections: Record<string, boolean>;
  };
};
```

## Core Behaviors

- **Creation**: new session + new tab, input focused, state `idle`
- **Switching**:
  - active tab changes
  - components subscribe to new `sessionId` via `switchMap`
- **State Indicators**:
  - driven by `session.stateChanged` and lifecycle
  - icons: ⚪ 🟢 🔵 🟡 🔴 🟣
- **Duplication**:
  - snapshot input/config/uiState
  - create new session in `idle`
  - bind new tab to new `sessionId`
  - MVP note (D12): snapshot is built client-side from live session state
- **Closing**:
  - unsubscribe via `takeUntil`
  - optionally destroy session

## UX Notes

- Tab bar: title + state icon + close button + “+” for new tab
- Keyboard shortcuts (optional): Ctrl+Tab, Ctrl+Shift+Tab
```
