# forui — Declarative UI Framework for forai

forui is a UI framework written in [forai](https://github.com/forailang/forai).
Components are forai functions that return `ViewNode`. Reactive state lives
in signals; the diff engine computes minimal patches between renders. The
same component code targets the browser DOM (via `html-forui`), SSR (via
`renderSSR`), and test fixtures (via `testMount`).

This is a forai library — there is no Rust here. The CLI from
[fai/](https://github.com/forailang/forai) drives everything: format,
check, test, doc.

## Framework Reference

[forui.md](forui.md) is the canonical reference: project layout,
import paths, fullstack project shape, RPC, routing, signals, common
mistakes. Read it before designing changes.

[docs.md](docs.md) is a short concepts overview, used as the
`fai doc` entry point (declared in `fai.toml`).

## Build and Test

```bash
fai fmt                    # format all .fai files in this project
fai check                  # run formatter + type checker
fai test                   # run all tests in src/
fai doc                    # browse the framework's own docs
fai doc Forui.signal       # docs for one module
```

`fai check` runs `fmt` then `check`. `fai test` runs the same prefix
then all `test` blocks. There is no separate build step for forui —
it compiles inside whichever app or example consumes it.

## Source Layout

A directory under `src/` is one forai module. Every `.fai` file in the
directory shares the same module namespace; consumers import from the
directory name (e.g. `use { Label } from Forui.view`).

- `src/forui.fai` — public API: `mount`, `rerender`, `renderSSR`,
  test helpers (`testMount`, `testMountAt`, `staticApp`, etc.).
- `src/view/`
  - `view.fai` — `ViewNode`, `Events`, `Children`, tree-building
    context, tree query helpers (`childCount`, `getChild`,
    `findByKind`, `findByProp`).
  - `events.fai` — handler registry, host-runtime event bridge,
    `onClick` / `onChange` UFCS attachers.
  - `components.fai` — containers (`Window`, `VStack`, etc.) and
    leaves (`Label`, `Button`, `TextInput`, `Toggle`,
    `SegmentedControl`, `Spacer`, `Divider`, `ImageView`).
  - `modifiers.fai` — UFCS modifiers (`padding`, `background`,
    `fontSize`, `cornerRadius`, `flex`, `withKey`, …).
- `src/signal/`
  - `signal.fai` — `Signal` type, `createSignal`, setters
    (`setValue`, `setLoading`, `setLoaded`, `setError`), dirty
    tracking, change listener, status helpers (`isLoading`, …).
  - `hooks.fai` — `useSignal`, `useLoad`, `reload`, `doLoad`,
    save/restore/reset hook state.
- `src/router/router.fai` — `Router`, `Route`, `Link`, `navigate`,
  `routeParam`, `currentPath`.
- `src/diff/diff.fai` — virtual-tree diff engine.
- `src/rpc/rpc.fai` — client-side RPC plumbing.
- `src/cookie/cookie.fai` — cookie read/write helpers.

When adding a new component, modifier, signal helper, or router
primitive, place it in the file matching its concept (above) and add
a `test` block in the same file.

## Key Concepts

### Signals

`useSignal(initial)` creates a component-scoped signal. Optional
trailing-closure loader runs through `loading -> loaded`/`error`:

```fai
var tasks = useSignal([]) do
    fetchTasks()
end
```

Read with `tasks.value`. Write with `tasks.setValue(...)`,
`setLoading()`, `setLoaded(value)`, `setError(message)`. `reload()`
re-runs the stored loader.

`useSignal` must be at the top of a component function — the
framework tracks signals by call order, so calling it inside a
conditional or loop will misalign on re-render.

### Components return ViewNode

A component is a forai function returning `ViewNode`. Containers take
a trailing `do ... end` block (a `Children` closure):

```fai
def Counter
    @return ViewNode
do
  var count = useSignal(0)
  VStack do
      Label('Count: ' + toString(count.value))
      Button('++', onClick: do
          count.setValue(count.value + 1)
      end)
  end
end
```

### Children, not ViewNode, for layout helpers

The single most important rule. Reusable wrappers (Card, Section,
Page, etc.) take a `Children` closure — never a `ViewNode`
parameter. ViewNode constructors register themselves into whichever
frame is active when they're called; if you pre-build children at the
call site and pass them in, they register in the caller's frame and
the wrapper sees an empty subtree.

```fai
# Right
def Card
    @param title String
    @param builder Children
    @return ViewNode
do
  VStack do
      Label(title)
      builder()           # builds inside Card's frame
  end.padding(16)
end
```

### Modifiers are UFCS

`padding`, `background`, `fontSize`, etc. are top-level functions
that take a `ViewNode` first and return a new `ViewNode`. UFCS makes
the chain syntax work:

```fai
Label('Hi').padding(8).background('#fff')
# = background(padding(Label('Hi'), 8), '#fff')
```

To add a new modifier, define a function in `src/view/modifiers.fai`
matching that signature and write a test.

### Imports are explicit per file

forui has no wildcard re-exports. Every file imports the symbols it
uses by name from the right module:

```fai
use { VStack, Label, Button } from Forui.view
use { padding, background, fontSize } from Forui.view
use { useSignal, setValue } from Forui.signal
```

A directory is one module — symbols defined in any file under
`src/view/` are imported from `Forui.view`, regardless of which file
they live in. Cross-file `var` references work too (the forai checker
collects module-level bindings before checking bodies).

## Test Conventions

forai requires every named function to have a `test` block — `fai
test` fails otherwise. Tests live in the **same file** as the
function, immediately after the def or grouped at the bottom of the
file.

For component functions, the minimum useful test is a root-kind
check:

```fai
test Counter
it 'renders as a vstack'
    let node = testMount(Counter)
    assert.equals(node.kind, 'VStack')
end
end
```

For framework primitives (signals, diff ops, router internals), test
the contract directly. `findByKind`, `findByProp`, `getProp`,
`getIntProp`, and `hasProp` (all in `Forui.view`) make assertions
about a built tree easy to write.

## Common Mistakes

| Mistake | Fix |
|---|---|
| Helper takes `ViewNode` parameter, children render in caller's frame | Take a `Children` closure parameter and call it inside the helper's frame |
| `useSignal` inside an `if` or `for` | Always at the top of a component function |
| Forgetting to import a modifier | Each modifier (`padding`, `fontSize`, …) imports from `Forui.view` per file |
| `count.setValue(x)` fails to type-check | The binding must be `var`, not `let` — setValue takes a mutable param |
| Custom container forgets to push/pop a frame | Use `buildContainer(kind, props, children)` from view.fai — it handles framing |

## Repository Notes

- `plans/` is intentionally gitignored — local planning notes only.
- `build/`, `*.wasm`, and `interface.json` are build artifacts.
- forui has no `target/` directory (no Rust); ignore the one in
  fai/ if you switch back and forth.
- When working on a forui-consuming app, point its `fai.toml`
  dependency at this directory: `"file:../forui" = "0.1.0"`.

## Reporting Language Gaps

forui is one of the larger pieces of forai source code that exists,
so it tends to surface forai bugs first. If you hit a missing
language feature or a checker/parser/codegen bug, **stop and report
it** rather than working around it. The pattern: minimal repro,
report to the user, then either fix in fai/ or pause forui work
until fixed.
