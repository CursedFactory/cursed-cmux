# Terminal And Browser Performance Remediation Plan

## Goal

Reduce the visible stutter, input delay, redraw glitches, and focus lag in the macOS app, especially in the following user-observed cases:

- terminal typing with fancy prompts/renderers such as zsh themes
- multi-pane terminal layouts where stutter gets worse as more panes are visible
- browser/devtools/inspector panes during resize, split churn, and host reparenting
- workspace switching or focus restoration that feels one beat late

This plan is intentionally verbose and includes direct code references, copied behavior notes, and likely causal chains so implementation can proceed without having to rediscover the same context.

## User-Observed Symptoms

- Stutters and delays are present even in simpler layouts, but are much worse with multiple visible panes.
- Browser redraw problems are especially noticeable around inspector and developer tools.
- Terminal issues are especially noticeable with visually busy shells such as zsh setups with prompt redraws.

Those symptoms are consistent with main-thread layout and forced redraw work occurring on hot paths that should otherwise be wakeup-driven by Ghostty or lazily refreshed by WebKit/AppKit.

## Scope

Primary scope:

- `Sources/GhosttyTerminalView.swift`
- `Sources/TerminalWindowPortal.swift`
- `Sources/AppDelegate.swift`
- `Sources/TabManager.swift`
- `Sources/BrowserWindowPortal.swift`
- `Sources/Panels/BrowserPanelView.swift`
- `Sources/Panels/BrowserPanel.swift`
- `Sources/ContentView.swift`
- `Sources/Workspace.swift`

Out of scope for the first pass:

- broad Ghostty submodule rendering changes
- speculative Metal or CA tuning without a measured local cause
- full redesign of sidebar architecture
- removal of correctness guards that were clearly added for past regressions

## Constraints

- Do not regress terminal correctness for IME, focus restore, or split-reparent recovery.
- Do not remove the documented pointer-event-only `hitTest` protections in `TerminalWindowPortal`.
- Do not remove the existing `TabItemView` `Equatable` path in the sidebar.
- Prefer narrowing or coalescing refreshes over deleting them entirely.
- Preserve behavior that exists to work around SwiftUI/WebKit responder bugs unless there is a safer replacement.

## Performance Review Summary

The strongest pattern in the current code is not one giant algorithmic problem. It is repeated synchronous "insurance" work on hot paths:

- per-keystroke terminal forced refreshes
- geometry fanout across all visible portal-hosted terminals
- browser refresh paths that do immediate plus async plus delayed work, then sometimes invoke that whole stack twice
- broad state observation that can invalidate large SwiftUI regions during prompt-time metadata updates
- layered keyboard dispatch interception that adds baseline overhead before the terminal even receives input

The app already contains a number of good defensive optimizations, but they are partially offset by recovery/reparent logic that now runs too often in steady-state scenarios.

## Current Architecture Notes

### Terminal rendering model

The intended steady-state model is already documented in code:

- `Sources/GhosttyTerminalView.swift:4002-4003`
  - `// Let Ghostty continue rendering on its own wakeups for steady-state frames.`

That is the right architecture. The main performance issue is that some typing and geometry paths still bypass that model and force explicit refreshes.

### Terminal hot path finding

At `Sources/GhosttyTerminalView.swift:6111-6115`:

```swift
if shouldRefreshAfterTextInput {
    terminalSurface?.forceRefresh(reason: "keyDown.textInput")
}
```

At `Sources/GhosttyTerminalView.swift:4007-4045`, `forceRefresh` does all of the following synchronously:

- validates attached view/surface/window state
- reasserts display ID via `ghostty_surface_set_display_id`
- calls `view.forceRefreshSurface()`
- calls `ghostty_surface_refresh(surface)`

This is a strong candidate for typing latency, especially when fancy prompts already trigger lots of redraw work in the shell.

### Multi-pane terminal fanout

At `Sources/TerminalWindowPortal.swift:749-762`:

```swift
synchronizeLayoutHierarchy()
synchronizeAllHostedViews(excluding: nil)

for entry in entriesByHostedId.values {
    guard let hostedView = entry.hostedView, !hostedView.isHidden else { continue }
    if hostedView.reconcileGeometryNow() {
        hostedView.refreshSurfaceNow(reason: "portal.externalGeometrySync")
    }
}
```

At `Sources/TerminalWindowPortal.swift:712-718`, `synchronizeLayoutHierarchy()` calls `layoutSubtreeIfNeeded()` on multiple nodes before host-frame sync.

This means one external geometry change can cascade into:

- full hierarchy layout flushes
- hosted-view synchronization
- a loop over every visible terminal entry
- forced refreshes for each entry whose geometry reconciliation changed anything

This directly matches the user report that problems are much worse with multiple panes.

### Browser refresh duplication

At `Sources/BrowserWindowPortal.swift:2549-2611`, `refreshHostedWebViewPresentation(...)` runs:

- one immediate refresh pass
- one async main-queue refresh pass
- one delayed refresh pass after 30 ms

Each pass marks layout/display invalidation and eventually calls:

- `layoutSubtreeIfNeeded()`
- `displayIfNeeded()`
- optional `browserPortalReattachRenderingState(...)`

Then at `Sources/BrowserWindowPortal.swift:2795-2815`, `forceRefreshWebView(...)` does this:

```swift
synchronizeWebView(... forcePresentationRefresh: true)
...
refreshHostedWebViewPresentation(webView, in: containerView, reason: refreshSource)
```

And `synchronizeWebView(... forcePresentationRefresh: true)` can itself already route into `refreshHostedWebViewPresentation(...)` at `Sources/BrowserWindowPortal.swift:3491-3534`.

That is likely duplicate refresh scheduling for one logical event.

### Browser inspector/devtools reparenting heuristic

At `Sources/BrowserWindowPortal.swift:2647-2690` and `Sources/Panels/BrowserPanelView.swift:5852-5901`, related WebKit subviews are identified via class name matching:

```swift
let className = String(describing: type(of: view))
guard className.contains("WK") else { return false }
```

This is broad and ownership-agnostic. It can pull along any sibling whose class name happens to contain `WK`, then manually convert frames during host changes.

That is a plausible cause of:

- inspector jumps
- transient blank/overlap states
- hit-test drift
- frame mismatch during inline/detached devtools transitions

### Keyboard dispatch layering

The app has multiple keyboard interception layers:

- `AppDelegate.installShortcutMonitor()` local monitor in `Sources/AppDelegate.swift:9094-9162`
- `NSApplication.cmux_applicationSendEvent` in `Sources/AppDelegate.swift:12649-12675`
- `NSWindow.cmux_sendEvent` in `Sources/AppDelegate.swift:12783-12865`
- `NSWindow.cmux_performKeyEquivalent` in `Sources/AppDelegate.swift:12903-13062`
- terminal-local `performKeyEquivalent` in `Sources/GhosttyTerminalView.swift:5579-5645`

Most of these exist for real correctness reasons, but the total path is still expensive and makes terminal input more sensitive to any extra work done in focus or responder discovery.

### Sidebar churn as a secondary amplifier

The sidebar is probably not the root cause of hard redraw glitches, but it can steal main-thread time while the user is typing.

Relevant sites:

- `Sources/ContentView.swift:1553-1560`
- `Sources/ContentView.swift:8660-8794`
- `Sources/ContentView.swift:9014-9020`
- `Sources/ContentView.swift:12021-12030`
- `Sources/Workspace.swift:5627-5659`

The root view and sidebar both subscribe broadly to shared state, and each workspace row effectively has two `sidebarObservationPublisher` listeners today.

## Existing Optimizations To Preserve

These are important and should not be casually removed during remediation:

### Terminal pointer-event guard

`Sources/TerminalWindowPortal.swift:132-145, 193-195`

The portal `hitTest` path is explicitly guarded so keyboard events do not pay pointer-routing costs. This should remain intact.

### Ghostty wakeup coalescing

The terminal code already coalesces wakeup/tick behavior and avoids some redundant drawable updates. Any fix should move more work onto that model, not away from it.

### Sidebar row isolation

`TabItemView` is `Equatable` and wrapped in `.equatable()` in `Sources/ContentView.swift:8997`. That optimization is valuable and should be preserved.

### Debounced workspace sidebar observation

The 40 ms debounce on sidebar row observation is helpful. The problem is the number and breadth of upstream invalidations, not the existence of the debounce itself.

## Stage 1: Remove Per-Keystroke Terminal Forced Refresh In Steady State

### Objective

Make normal terminal typing rely on Ghostty's normal wakeup-driven rendering instead of explicit `forceRefresh(...)` on every text input event.

### Evidence

`Sources/GhosttyTerminalView.swift:6111-6115`

```swift
if shouldRefreshAfterTextInput {
    terminalSurface?.forceRefresh(reason: "keyDown.textInput")
}
```

`Sources/GhosttyTerminalView.swift:4007-4045`

```swift
func forceRefresh(reason: String = "unspecified") {
    ...
    ghostty_surface_set_display_id(currentSurface, displayID)
    view.forceRefreshSurface()
    ghostty_surface_refresh(surface)
}
```

### Hypothesis

The app likely accumulated this refresh as a safety net for a smaller class of input/display-sync problems, but it is now running on the hottest possible path: normal typing.

Fancy zsh prompts make this worse because each keystroke can already trigger:

- shell-side prompt state changes
- cursor redraws
- OSC/status updates
- potential sidebar metadata updates

Adding a full forced surface refresh on top of that is a plausible direct source of lag.

### Changes

- Trace and document the conditions that set `shouldRefreshAfterTextInput`.
- Split those conditions into two buckets:
  - steady-state text input
  - recovery-only input cases
- Remove forced refresh from the steady-state bucket.
- Keep forced refresh only for explicitly justified cases such as:
  - IME composition edge cases if proven necessary
  - post-reparent stale-surface recovery
  - display-topology / sleep-wake edge cases
  - known first-frame-after-focus failures

### Suggested implementation direction

- Rename the boolean to something more explicit than `shouldRefreshAfterTextInput` if needed.
- Prefer a narrow enum or reason set instead of one broad boolean.
- Add comments documenting why each remaining force-refresh case still exists.

### Success criteria

- Normal typing no longer calls `forceRefresh(...)` on each keystroke.
- No regression in IME behavior or first-frame correctness after focus/reparent.
- Fancy prompts no longer visibly hitch as often during normal input.

## Stage 2: Coalesce Terminal Portal Geometry Sync Across Visible Panes

### Objective

Stop one layout/resize event from synchronously refreshing every visible terminal pane.

### Evidence

`Sources/TerminalWindowPortal.swift:749-762`

```swift
synchronizeLayoutHierarchy()
synchronizeAllHostedViews(excluding: nil)

for entry in entriesByHostedId.values {
    guard let hostedView = entry.hostedView, !hostedView.isHidden else { continue }
    if hostedView.reconcileGeometryNow() {
        hostedView.refreshSurfaceNow(reason: "portal.externalGeometrySync")
    }
}
```

`Sources/TerminalWindowPortal.swift:712-718`

```swift
installedContainerView?.layoutSubtreeIfNeeded()
installedReferenceView?.layoutSubtreeIfNeeded()
hostView.superview?.layoutSubtreeIfNeeded()
hostView.layoutSubtreeIfNeeded()
_ = synchronizeHostFrameToReference()
```

### Hypothesis

This logic is reasonable during true host-change or reparent recovery, but too expensive for high-frequency geometry churn like:

- split drags
- live resize
- sidebar resize
- workspace transitions with multiple visible panes

### Changes

- Distinguish structural host changes from ordinary frame churn.
- Skip hierarchy-wide `layoutSubtreeIfNeeded()` when host/reference frames are already current.
- Track changed hosted entries and only reconcile/refresh those.
- Coalesce repeated external geometry signals to one pass per runloop turn or display tick.
- Consider a lightweight geometry-only pass first, with refresh deferred only if a terminal remains visually stale.

### Suggested implementation direction

- Introduce a small coalescer or generation token around `synchronizeAllEntriesFromExternalGeometryChange()`.
- Cache the last synchronized host/reference frame pair.
- Avoid iterating `entriesByHostedId.values` when only one hosted view was affected.
- Push any required recovery refresh to the next turn rather than inline when the view is still in churn.

### Success criteria

- Split dragging cost scales closer to the number of changed panes, not all visible panes.
- Multi-pane layouts stop degrading as sharply during resize or split churn.
- No regression in terminal geometry correctness after portal host replacement.

## Stage 3: Collapse Browser Refresh To One Presentation Pipeline

### Objective

Prevent duplicate browser refresh scheduling and reduce expensive layout/display/reattach cycles.

### Evidence

`Sources/BrowserWindowPortal.swift:2549-2611` already defines a three-pass presentation refresh pipeline.

`Sources/BrowserWindowPortal.swift:2795-2815`

```swift
synchronizeWebView(... forcePresentationRefresh: true)
...
refreshHostedWebViewPresentation(webView, in: containerView, reason: refreshSource)
```

`Sources/BrowserWindowPortal.swift:3491-3534`

```swift
if forcePresentationRefresh {
    refreshReasons.append("anchor")
}
...
case .refresh:
    refreshHostedWebViewPresentation(...)
```

### Hypothesis

Some browser glitches are not just from WebKit being hard to host. They are from the app running multiple overlapping refresh strategies for one logical refresh request.

### Changes

- Make `forceRefreshWebView(...)` choose one path:
  - either `synchronizeWebView(... forcePresentationRefresh: true)` is authoritative
  - or `forceRefreshWebView(...)` bypasses sync and directly does the refresh
- Do not allow both for the same request.
- Re-evaluate whether every refresh needs immediate + async + delayed passes.
- Consider downgrading some refreshes to geometry-only invalidation if rendering state reattach is not required.

### Suggested implementation direction

- Return richer sync results from `synchronizeWebView(...)`, such as whether presentation refresh was already performed.
- Gate delayed passes behind evidence that immediate/async did not settle the view.
- Make the reason taxonomy stricter so geometry-only changes stay geometry-only.

### Success criteria

- Browser pane resize/rebind operations schedule one coherent refresh pipeline.
- Visible flicker and blank-frame recoveries are reduced.
- Developer tools transitions do not feel like they are refreshing the same view tree twice.

## Stage 4: Replace Broad `WK*` Reparenting With Ownership-Aware Hosting

### Objective

Stabilize devtools/inspector hosting so browser redraw errors are not caused by overly broad companion-view movement.

### Evidence

`Sources/BrowserWindowPortal.swift:2647-2690`

```swift
let relatedSubviews = sourceSuperview.subviews.filter { view in
    if view === primaryWebView { return true }
    let className = String(describing: type(of: view))
    guard className.contains("WK") else { return false }
    ...
}
```

`Sources/Panels/BrowserPanelView.swift:5852-5901` repeats the same pattern in local-inline hosting.

### Hypothesis

This was probably added as a pragmatic fix for docked inspector companion views, but it is too broad to be stable long-term.

### Changes

- Track the hosted primary `WKWebView` explicitly.
- If inspector-side companion containers must move, identify them using stronger criteria than `className.contains("WK")`.
- Prefer moving a known host container subtree rather than picking arbitrary siblings.
- Avoid manual frame conversion where constraints or a stable slot container can own sizing.

### Suggested implementation direction

- Introduce a tiny ownership record in the portal entry for known companion views.
- Only adopt/reparent views discovered from a controlled attachment point.
- Log when unknown `WK*` siblings are found rather than automatically moving them.

### Success criteria

- Docked inspector/devtools no longer jump or overlap during reparent/resize.
- Browser panes do not transiently lose hit testing or appear offset.
- The code no longer depends on a generic `contains("WK")` sibling sweep for correctness.

## Stage 5: Shorten The Terminal Keyboard Dispatch Path

### Objective

Reduce baseline per-key overhead without regressing the responder-chain correctness workarounds.

### Evidence

Relevant layers:

- `Sources/AppDelegate.swift:9094-9162`
- `Sources/AppDelegate.swift:12649-12675`
- `Sources/AppDelegate.swift:12783-12865`
- `Sources/AppDelegate.swift:12903-13062`
- `Sources/GhosttyTerminalView.swift:5579-5645`

### Hypothesis

No single layer is catastrophic, but the stack is deep enough that any extra focus or view discovery work becomes noticeable. The worst case is terminal-focused plain text input paying for browser and menu related checks before Ghostty receives the event.

### Changes

- Fast-path terminal-focused plain text key events earlier.
- Reuse responder classification computed in one layer instead of rediscovering it in later layers.
- Avoid browser-specific routing work when terminal focus is already known and no command-equivalent path is relevant.
- Re-check whether both app/window swizzles are still needed for all event types.

### Suggested implementation direction

- Introduce a small event-context classification struct populated once per key event.
- Carry that through the swizzled path rather than recomputing owning web view / ghostty view repeatedly.
- Keep command-key menu routing logic intact, but make the plain-text path very short.

### Success criteria

- Reduced keyboard-path overhead for terminal-focused keyDown events.
- No regression in browser form Enter handling, menu shortcuts, or terminal font zoom routing.

## Stage 6: Reduce Sidebar And Workspace Observation Churn

### Objective

Stop prompt-time metadata updates from invalidating broad SwiftUI regions while the user is typing.

### Evidence

`Sources/ContentView.swift:1556-1560`

```swift
@EnvironmentObject var tabManager: TabManager
@EnvironmentObject var notificationStore: TerminalNotificationStore
...
```

`Sources/ContentView.swift:8708-8794` computes sidebar-wide derived state on each body pass.

`Sources/ContentView.swift:9014-9020` and `12021-12030` both subscribe to `tab.sidebarObservationPublisher` for the same workspace row stack.

`Sources/Workspace.swift:5627-5659` merges many `@Published` properties into one observation stream.

`Sources/Workspace.swift:6556-6610` is representative of the focused-panel mirror pattern where one logical update can publish panel-local and workspace-local copies.

### Hypothesis

This is a secondary amplifier. It is unlikely to be the main root cause of hard redraw glitches, but it can still steal enough main-thread time to worsen terminal typing when prompt metadata updates burst.

### Changes

- Remove duplicate `sidebarObservationPublisher` subscriptions per workspace row stack if possible.
- Move more derived sidebar state closer to row-local computation or precomputed snapshots.
- Narrow root `ContentView` reads of `TabManager` so unrelated publishes do not invalidate the whole tree.
- Consider one consolidated sidebar snapshot model for hot metadata instead of many separate `@Published` properties.

### Success criteria

- Prompt metadata churn redraws only the affected row subtree.
- Typing does not visibly hitch when cwd/branch/PR/status changes arrive.

## Implementation Order

1. Stage 1 terminal per-keystroke forced refresh removal
2. Stage 2 terminal portal geometry coalescing
3. Stage 3 browser refresh pipeline deduplication
4. Stage 4 browser devtools/inspector ownership-aware hosting
5. Stage 5 keyboard dispatch fast-path cleanup
6. Stage 6 sidebar observation narrowing

## Why This Order

- Stage 1 is the highest-probability direct input-latency win.
- Stage 2 addresses why the problem gets much worse with multiple panes.
- Stages 3 and 4 target the inspector/devtools redraw issues the user specifically called out.
- Stage 5 is a path-shortening cleanup that should help once the major redraw sources are under control.
- Stage 6 is valuable but should not block the more direct render/input fixes.

## Risks

- Some forced refreshes likely exist because of prior bugs around sleep/wake, reparent, display changes, or first-focus frames.
- Browser hosting code is full of practical WebKit workarounds; over-simplifying it can regress detached or docked inspector behavior.
- Keyboard-path changes can easily regress menu shortcuts or browser form handling if fast paths are too aggressive.
- Sidebar changes can accidentally defeat the existing `Equatable` row optimization if reactive state leaks back into `TabItemView`.

## Validation Notes

Per repo guidance, do not run full local test suites.

Useful validation after implementation:

- build with `./scripts/reload.sh --tag <perf-tag>`
- manual typing check in:
  - single terminal pane
  - 2-4 split terminal panes
  - zsh prompt with frequent redraws
- manual browser checks in:
  - normal browser pane resize
  - devtools/inspector attached and detached
  - pane reparent / split close / fullscreen transitions
- debug timing logs already present around `performKeyEquivalent`, `sendEvent`, and terminal typing can be used to compare before/after behavior

## Nice-To-Have Follow-Ups

- Add lightweight counters or debug-only aggregation for:
  - terminal `forceRefresh` reasons and counts
  - portal geometry sync counts per second
  - browser refresh pipeline phase counts per refresh reason
- If needed, add a debug window or menu item to show these counters live during resize/typing experiments.

## Concrete First Change

If implementation starts immediately, the first change should be:

- identify every condition behind `shouldRefreshAfterTextInput`
- narrow or remove the steady-state path
- keep only the minimum proven recovery cases

That is the shortest path to a user-visible reduction in typing delay.
