# forui issues

## RPC (or any side-effecting async call) in a component render body loops and OOMs

**Severity:** high — crashes the client with an unrecoverable wasm trap, hard to diagnose.

### Symptom

The client wasm aborts during a render pass with:

```
FAI __fai_poll failed RuntimeError: memory access out of bounds
    at rt_alloc
    at Forui.snapshotTree
    at Forui.snapshotTree
    at Forui.snapshotTree
    at Forui.renderPass#resume
    at sched_poll
```

The stack shows the same remote def resuming over and over
(`$data.firstRun.firstRunStatus#resume` repeated), i.e. the RPC is being
re-fired continuously until the heap is exhausted and `rt_alloc` walks off the
end of linear memory. The UI freezes ("conversation just dies"); no graceful
error surfaces.

### Reproduction

A page component calls a remote def directly in its render body and does **not**
store the result in a signal:

```fai
def FirstRunPage
    @return ViewNode
do
    let status = firstRunStatus()   # <-- remote def called every render
    if status.complete
        ...
    else
        FirstRunSetupShell(status)
    end
end
```

Sequence that triggers it: some child signal update causes a render → the render
calls `firstRunStatus()` (async) → when that RPC resolves the framework schedules
another render → which calls `firstRunStatus()` again → … The loop never
settles. Each `renderPass` re-runs `snapshotTree` over the (growing) tree and
allocates, so memory climbs until `rt_alloc` traps with "memory access out of
bounds". Under heavier apps it can take a few interactions to tip over, which
makes it look intermittent.

### Workaround (app side)

Memoize the call so it runs once per mount:

```fai
var initialStatus = useSignal(firstRunStatus())
let status = initialStatus.value()
```

### Root cause / what forui should do

1. **Reads in render must be pure.** Calling a remote def (a side-effecting
   async op) from a render body is a footgun: its completion invalidates the
   render, which re-runs the call, which re-invalidates — an unbounded loop.
   forui should either:
   - detect/guard against a render that dispatches the *same* remote def it
     dispatched on the previous pass (dev-mode warning at minimum), and/or
   - provide a first-class `useRemote`/`useResource`-style hook that runs an
     async call once, caches the result in a signal, and exposes
     loading/error/data — so the common "load data for this page" case has an
     obvious non-looping API.

2. **`snapshotTree` / `rt_alloc` should fail gracefully, not trap.** An
   allocation past the end of linear memory currently produces an unrecoverable
   `RuntimeError: memory access out of bounds` mid-render with no application
   error. The allocator should either grow memory or surface a catchable error
   with context (which component/tree, approximate size) instead of a bare wasm
   trap, so runaway renders are diagnosable.

3. **Consider a render-loop backstop.** A cap on consecutive re-renders with no
   intervening user/event input (with a dev-mode log of the offending
   component) would turn this class of bug from "silent freeze + OOM" into an
   actionable message.

### Where this was hit

`brain` first-run page (`src/pages/firstRun/FirstRunPage.fai`). Fixed app-side by
memoizing `firstRunStatus()` in a signal.

### Related sharp edges seen alongside this

- **Static assets served without cache headers.** `/web.wasm` and
  `/fai-runtime.js` are served with no `Cache-Control`/`ETag`. The runtime loads
  the wasm via `fetch('/web.wasm')`, and a browser hard-refresh does **not**
  force JS `fetch()` to bypass the HTTP cache — so clients keep running a stale
  compiled bundle after a rebuild ("hard refresh, no change"). Consider
  `Cache-Control: no-store` (or a build-versioned URL like `/web.wasm?v=<hash>`)
  for the runtime + wasm assets.
- **Programmatic value-setting doesn't update bound signals.** Setting an
  `<input>`/`<textarea>` `.value` and dispatching a native `input` event does
  not update the signal bound via `signalValue:` (so tools/automation that set
  values can't drive forui forms). A real keypress works. Worth confirming forui
  listens for the standard `input` event so synthetic input is honored.
