# Forui.signal

`Forui.signal` provides component-scoped reactive state for Forui apps. Use it
for form fields, local UI state, and loaded data that should trigger a rerender
when it changes.

Most app components import the signal helpers they use directly:

```fai
use { useSignal, isLoading, isLoaded, isError, setValue, reload } from Forui.signal
```

## Component State

Create state with `useSignal(initialValue)`. The signal has a stable slot across
rerenders, so every `useSignal` call must stay in the same order every render.
Call it near the top of the component, never inside an `if`, `for`, or another
conditional branch.

Always bind a signal with `var` when you plan to mutate it. Helpers such as
`setValue`, `setLoaded`, `setError`, and `reload` take the signal as mutable.

```fai
def Counter
    @return ViewNode
do
    var count = useSignal(0)

    Button('Increment', onClick: do
        count.setValue(count.value + 1)
    end)
end
```

The signal's current data is in `.value`. Its lifecycle is stored in `.status`
and can be checked with `isInitial`, `isLoading`, `isLoaded`, and `isError`.
`setValue` and `setLoaded` move the signal to `loaded` and clear any error.
`setError` moves it to `error` and stores the message in `.error`.

## Loaded Data

Pass a trailing loader closure to `useSignal` for data that should load after
the first render. The initial value must have the same shape as the loaded data;
for empty arrays or nullable values, prefer a typed default so inference has
enough information.

```fai
var tasks = useSignal([]) do
    getTasks()
end

if tasks.isLoading()
    Label('Loading...')
else if tasks.isError()
    Label('Error: ' + tasks.error!)
else
    for task in tasks.value
        Label(task.text)
    end
end
```

A loader signal starts at `initial`, moves to `loading`, then becomes `loaded`
when the loader returns or `error` when it throws. On the browser client the
loader runs with `nowait`, so the first render should handle the loading state.
During SSR and `testMount`, loader signals are left in `loading`; avoid tests
that expect remote or async loader results to be available from SSR.

Use `reload()` to run a signal's stored loader again. This is the usual pattern
after a mutation that changes server-backed data.

```fai
Button('Refresh', onClick: do
    tasks.reload()
end)
```

## Lower-Level Helpers

`createSignal` constructs a standalone signal without using the hook slot
system. It is useful in tests and lower-level framework code. `useLoad` attaches
a loader to an existing signal, but most app code should prefer the trailing
closure form of `useSignal`.

Dirty tracking and hook lifecycle helpers such as `markDirty`, `clearDirty`,
`resetHooks`, `saveHookState`, `restoreHookState`, `enterSsrMode`, and
`exitSsrMode` are framework integration APIs. App components usually should not
call them directly, except for `clearHookSignals` in tests that need a fresh
component state set.
