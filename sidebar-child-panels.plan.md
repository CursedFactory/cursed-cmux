# Sidebar Child Panels Plan

## Goal

Add one collapsable child section under each workspace row in the sidebar. Child rows are generic "panels" so future item types can share the same UI and activation model.

Initial supported child type:
- panes

Planned future child types:
- agent sessions
- worktrees

## Constraints

- Keep workspace row performance intact.
- Do not overload existing workspace multi-select semantics with child-row selection.
- Keep expand/collapse state window-local only.
- Use one disclosure per workspace, not separate per-type disclosures.
- Worktree support is optional if discovery becomes too expensive or invasive.

## Current Architecture Notes

- Sidebar is flat today: `VerticalTabsSidebar` renders `ForEach(tabs)` in `Sources/ContentView.swift`.
- Workspace rows are performance-sensitive: `TabItemView` is `Equatable` and should remain lean.
- Workspace activation flows through `TabManager.selectWorkspace` and async focus restoration.
- Pane ordering already exists via `Workspace.sidebarOrderedPanelIds()` and split-tree helpers.
- Pane and surface APIs already exist in v2 socket methods, but sidebar pane rows should read in-memory workspace state directly.
- Agent state is not first-class in app runtime yet; it is mostly exposed as workspace status pills and external CLI session stores.
- Worktree discovery does not exist yet.

## Stage 1: Generic Child Panel Infrastructure

### Objective

Introduce a generic sidebar child-panel model and UI wrapper under each workspace row without destabilizing current sidebar behavior.

### Changes

- Add a new workspace wrapper view around `TabItemView`.
- Keep `TabItemView` focused on rendering only the parent workspace row.
- Render child rows beneath it inside a single disclosed section.

### New internal model layer

Add generic internal types along these lines:
- `SidebarWorkspacePanelSection`
- `SidebarWorkspacePanelItem`
- `SidebarWorkspacePanelKind`
- `SidebarWorkspacePanelActivationTarget`

Suggested item fields:
- `id`
- `kind`
- `title`
- `subtitle`
- `icon`
- `tint`
- `isActive`
- `showsUnreadIndicator`
- `activationTarget`

Suggested kinds:
- `pane`
- `agent`
- `worktree`

### State

Add window-local state in `ContentView`:
- expanded workspace ids
- optionally focused child item id if we want child highlight separate from workspace selection

Do not persist this in `SessionPersistence` for the first pass.

### Activation contract

Add a dedicated sidebar child activation path in `TabManager`, separate from external focus/open-notification behavior.

Suggested behavior:
- child click selects workspace if needed
- child click records preferred target surface/pane
- existing workspace focus restoration converges onto that target

Avoid using `focusTab(...)` directly for normal sidebar child clicks because that path is window-raising and notification-oriented.

### Observation

Extend the child refresh path so pane/agent/worktree child rows redraw when:
- focused pane changes
- selected surface in pane changes
- panel title changes
- per-panel metadata changes

## Stage 2: Panes As First Supported Child Panel Type

### Objective

Expose current workspace panes as child rows so users can quickly activate a pane from the sidebar.

### Data source

Use in-memory workspace state:
- split tree ordering
- bonsplit pane ids
- selected tab in each pane
- focused pane
- mapped panel ids
- panel titles and types

Do not call socket APIs for sidebar rendering.

### Row content

Each pane child row should show:
- selected surface title
- surface type icon
- active/focused state
- optional compact subtitle such as:
  - branch
  - cwd tail
  - surface count in pane

Keep visual density low.

### Activation

Clicking a pane row should:
- select the workspace
- focus that pane's selected surface
- preserve existing focus restoration behavior

### Suggested implementation detail

Create a workspace helper that returns display-ready pane items in stable visual order, rather than rebuilding this mapping in SwiftUI.

## Stage 3: Agent Sessions As Child Panel Type

### Objective

Promote agent session state from ad hoc workspace status rows to typed child panel items.

### New runtime model

Add a workspace-scoped agent session store in app runtime.

Suggested fields:
- `id`
- `provider` (`claude`, `codex`, `opencode`, etc.)
- `label`
- `detail`
- `state`
- `surfaceId`
- `parentSessionId`
- `pid`
- `updatedAt`

### API direction

Add explicit APIs for:
- upsert agent session
- clear agent session
- list agent sessions for workspace

This should coexist with existing generic `set-status` support, not replace it immediately.

### Integrations

Bridge existing sources:
- built-in Claude hook session context in `CLI/cmux.swift`
- built-in Codex hook session context
- OpenCode plugin metadata from `cursed-opencode-cmux`

### Sidebar behavior

Agent child rows should:
- show agent label and live activity
- indicate waiting/input-needed state
- focus linked surface when clicked
- fall back to workspace select if no surface is linked

## Stage 4: Worktrees As Child Panel Type

### Objective

Optionally show git worktrees under a workspace and allow opening them.

### Discovery

If implemented, add background worktree discovery keyed by repo root.

Likely commands:
- `git rev-parse --show-toplevel`
- `git worktree list --porcelain`

### Row behavior

Worktree row click should prefer open behavior:
- if an existing pane/surface already matches that worktree directory, focus it
- otherwise create/open a new terminal surface in that worktree directory

### De-scope rule

If worktree discovery adds too much complexity or latency, skip this stage without blocking the rest of the feature.

## Implementation Order

1. Stage 1 generic child-panel structure
2. Stage 2 pane rows
3. Stage 3 typed agent sessions
4. Stage 4 optional worktrees

## Risks

- `TabItemView` is optimized and should not absorb new reactive child logic.
- Existing workspace selection, workspace multi-select, and sidebar page selection are separate concepts already.
- Child activation must cooperate with async workspace focus restoration.
- Drag/reorder hit testing should remain attached to the parent workspace row shell, not child rows.
- Agent support needs a typed runtime model; status-pill parsing is not enough long-term.

## Acceptance Criteria

### Stage 1

- each workspace can expand/collapse a child section
- expand/collapse is window-local
- no regression to workspace drag/reorder or multi-select
- parent workspace rows remain performant

### Stage 2

- pane child rows render in visual split order
- clicking a pane child activates the correct pane/surface
- active pane is visibly indicated

### Stage 3

- agent sessions appear as child rows with status
- clicking an agent row activates its linked surface when available
- Claude/Codex/OpenCode can all map into the same generic child model

### Stage 4

- worktree rows can be opened or focused
- if discovery is too expensive, stage may be omitted cleanly
