
---

### `Wireframes.md`

```markdown
# Wireframes (Text Spec)

## Main Layout

- **Top bar**: tab bar + global actions
- **Left sidebar**: agent list
- **Main pane**: active session workspace
- **Right sidebar**: agent control panel / inspector
- **Bottom pane**: logs

## Tab Bar

- Tab title: agent name
- State indicator icon
- Close button
- “+” for new session

## Session Workspace

- Header: agent name + session ID
- Input panel: generated from agent input schema
- Output panel: streaming + final result
- Metadata: tokens, duration, cost
- Controls: Run, Stop, Restart
- Approval panel: shown while state = `waiting` (D11) — renders the agent's
  prompt + choice options from `approval.requested`; picking an option fires
  `POST /sessions/:id/approve` and the session resumes

## Logs Panel

- Bottom console-style area
- Real-time streaming
- Severity colors
- Search/filter
- Autoscroll toggle

## Workflow Placeholder

- Separate screen or mode:
  - “Workflow Builder Coming Soon”
  - Button: “Use tab interface”
  - Empty canvas area
  - Disabled node palette
