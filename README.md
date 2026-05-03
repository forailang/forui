# forui

forui is a declarative, signals-driven UI framework for [forai](https://github.com/forailang/forai).
Components are plain functions that return `ViewNode`. State lives in
signals — change a signal, the framework re-renders. The same component
code targets the browser DOM, server-side rendering, and test fixtures
through a platform adapter.

```fai
use { VStack, Label, Button } from Forui.view
use { fontSize, padding } from Forui.view
use { useSignal, setValue } from Forui.signal
use { mount } from Forui
use { htmlRender } from HtmlForui

def Counter
    @return ViewNode
do
  var count = useSignal(0)
  VStack do
      Label('Count: ' + toString(count.value)).fontSize(24)
      Button('Increment', onClick: do
          count.setValue(count.value + 1)
      end)
  end.padding(16)
end

def main
    @return Void
do
  mount(Counter, htmlRender)
end
```

## Why forui

- **Functions, not classes.** Components are forai functions. No lifecycle
  methods, no `this`, no implicit re-export magic.
- **Signals do the work.** Mutating a signal triggers a re-render of the
  components that read it. Loading and error states are first-class:
  `useSignal([]) do ... end` runs the loader and exposes `isLoading()` /
  `isError()` / `isLoaded()` for the UI to switch on.
- **UFCS modifiers.** Layout and style chain naturally:
  `Label('Hi').fontSize(20).padding(8).background('#fff')`. They're
  ordinary functions; new ones are easy to add.
- **Diff engine.** A virtual tree is diffed across renders and the
  adapter applies a minimal patch list.
- **Same code, multiple targets.** The browser uses `html-forui`; SSR
  uses `renderSSR`; tests use `testMount`.

## Adding to a project

forui ships as a forai package. Declare it (and `html-forui` for the
browser) as a dependency in your project's `fai.toml`:

```toml
[project]
name = "my-app"
version = "0.1.0"
source_root = "src"

[dependencies]
"file:../forui" = "0.1.0"
"file:../html-forui" = "0.1.0"
```

Then import explicitly per file (forui has no wildcard re-exports):

```fai
use { VStack, HStack, Label, Button, TextInput } from Forui.view
use { padding, background, fontSize, foreground } from Forui.view
use { useSignal, setValue, reload } from Forui.signal
use { Router, Route, Link, navigate, routeParam } from Forui.router
use { mount, renderSSR } from Forui
use { htmlRender, initWebRouter } from HtmlForui
```

## Signals

A signal is a reactive value. Reads are plain field access; writes go
through `setValue` (or one of its siblings) and schedule a re-render.

```fai
var count = useSignal(0)

count.value                   # 0
count.setValue(count.value + 1)
```

`useSignal` accepts an optional trailing-closure loader. The signal
goes through `loading -> loaded` (or `error`) automatically:

```fai
var tasks = useSignal([]) do
    fetchTasks()
end

VStack do
    if tasks.isLoading()
        Label('Loading…')
    else
        if tasks.isError()
            Label('Error: ' + tasks.error!).foreground('#b3261e')
        else
            for t in tasks.value
                Label(t.title)
            end
        end
    end
end
```

`tasks.reload()` re-runs the loader. `setLoading()`, `setLoaded(value)`,
and `setError(msg)` give explicit control when you need it.

> Signals must be declared at the top of a component function, not inside
> a conditional or loop — the framework tracks them by call order.

## Components: containers, leaves, modifiers

Containers (`Window`, `VStack`, `HStack`, `ZStack`, `ScrollView`, `View`)
take a trailing `do ... end` block. Children build inside the active
frame:

```fai
VStack do
    Label('Title').fontSize(24).fontWeight('700')
    HStack do
        Label('Left')
        Spacer()
        Label('Right')
    end
end.padding(16).background('#fafafa')
```

Leaves are component functions (`Label`, `Button`, `TextInput`,
`Toggle`, `SegmentedControl`, `Spacer`, `Divider`, `ImageView`).

Modifiers (`padding`, `background`, `foreground`, `cornerRadius`,
`fontSize`, `fontWeight`, `flex`, `width`, `maxWidth`, `opacity`,
`withKey`) attach via UFCS — they're ordinary functions defined in
`Forui.view`.

### Two-way binding with signals

`TextInput` and `Toggle` accept `signalValue: <Signal>` for two-way
binding — read the signal's value, write it on change:

```fai
var email = useSignal('')
TextInput('Email', signalValue: email, inputType: 'email')

var notifications = useSignal(true)
Toggle(signalValue: notifications)
```

## Reusable components: pass `Children`, not `ViewNode`

A common pitfall. Layout helpers (a Card, Section, Panel — anything
that wraps other nodes) must accept a `Children` closure parameter,
**not** a pre-built `ViewNode`. ViewNode constructors register
themselves into whichever frame is active when they run; if you build
the children at the call site and pass them in, they end up registered
in the *caller's* frame and your wrapper will see an empty subtree.

```fai
# Right — defers child construction
def Card
    @param title String
    @param builder Children
    @return ViewNode
do
  VStack do
      Label(title).fontSize(13).foreground('#6d6d72')
      builder()
  end.padding(16).background('#ffffff').cornerRadius(12)
end

# Use it
Card('SETTINGS') do
    Label('Name')
    Button('Edit', onClick: do navigate('/settings/name') end)
end
```

## Routing

```fai
use { Router, Route, Link, navigate, routeParam } from Forui.router

def App
    @return ViewNode
do
  Router do
      Route('/') do
          HomePage()
      end
      Route('/task/:id') do
          let id = routeParam('id')!
          TaskDetailPage(id)
      end
      Route('*') do
          Label('Page not found')
      end
  end
end
```

`navigate(path)` for programmatic navigation, `Link(path) do ... end`
for declarative links, `routeParam(name)` for `:id`-style params.

## RPC (fullstack apps)

Server-side `remote def` functions are callable from the client as if
local. The client wasm rewrites those bodies to a JSON RPC call so it
never executes server-only code (DB access, etc.):

```fai
# src/server/handlers/tasks.fai
remote def addTask
    @param token String
    @param title String
    @return Task
do
  # runs on the server only — DB writes etc.
  insertTask(token, title)
end

# src/client/pages/tasks.fai
use { addTask } from Server

Button('Add', onClick: do
    try
        addTask(sessionToken(), input.value)
        tasks.reload()
    catch e
        err.setValue(e.message)
    end
end)
```

The fullstack project layout (server target, client target, shared
types) is described in [forui.md](forui.md).

## Tests

forai requires every named function to have a test. Most components
can be covered with a one-line root-kind check; richer tests build a
tree with `testMount` and assert on its shape:

```fai
test Counter
it 'renders as a vstack'
    let node = testMount(Counter)
    assert.equals(node.kind, 'VStack')
    assert.equals(childCount(node), 2)
end
end
```

`testMount`, `findByKind`, `findByProp`, `getProp`, `getIntProp`, and
`hasProp` are exported from `Forui` / `Forui.view` for this.

## Layout

```text
src/forui.fai           public API: mount, rerender, renderSSR, test helpers
src/view/               components, modifiers, event registry/bridge
src/signal/             signals + hooks (useSignal, useLoad, reload)
src/router/             Router, Route, Link, navigate, routeParam
src/diff/               virtual-tree diffing
src/rpc/                client-side RPC plumbing
src/cookie/             cookie read/write
docs.md                 quick concepts overview (browseable via `fai doc`)
forui.md                full framework reference
```

## Status

forui is early. The API still moves, gaps are real, and you should
expect to read the source. The file structure is small enough that
adding a missing component, modifier, or signal helper is usually a
short patch in `src/view/` or `src/signal/`.

Run `fai check` and `fai test` from the project root to run the
checker and the test suite.

## License

Apache-2.0. See [LICENSE](LICENSE).
