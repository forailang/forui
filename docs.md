# Forui — forai UI Framework

Forui is a reactive UI framework for forai. It compiles to WebAssembly and renders
via a platform adapter (browser DOM, native, SSR). State is managed with signals;
the diff engine handles efficient updates.

## Core concepts

**`mount(app, render)`** — start the app. Call once with a render function and a
platform adapter. The adapter receives a diff list after every signal change.

**Signals** — reactive values. Create with `useSignal(initialValue)`. Mutate with
`setValue(sig, newValue)` (requires `mutable` param). Read with `sig.value`.
Status helpers: `isLoading(sig)`, `isLoaded(sig)`, `isError(sig)`. Use
`usePoll(intervalMs, active, handler)` for page-local refresh loops; polling
starts after the interval, does not overlap handlers, and is skipped during SSR.

**View tree** — build UI by returning `ViewNode` from your render function.
Primitives: `Label`, `Paragraph`, `Heading`, `Button`, `TextInput`, `TextArea`,
`Toggle`, `VStack`, `HStack`, `ZStack`, `ScrollView`, `Spacer`, `Divider`,
`ImageView`, `SegmentedControl`.
Modifiers are ordinary functions and work well through UFCS chaining:
`padding`, `background`, `foreground`,
`cornerRadius`, `fontSize`, `fontWeight`, `lineHeight`, `fontStyle`,
`letterSpacing`, `textAlign`, `fontFamily`, `flex`, `alignItems`,
`justifyContent`, `flexWrap`, `gap`, `alignSelf`, `display`, `width`,
`minWidth`, `maxWidth`, `height`, `minHeight`, `position`, `top`, `zIndex`,
`centered`, `border`, `borderLeft`, `opacity`, `cssClass`, `withKey`.
They must be in file scope; app UI files usually use `use * from Forui.view`,
while smaller utility files can import only the symbols they need.

Use `Label` for short UI/control text. Use `Paragraph` for body copy and
multiline prose. Use `Heading(text, level: n)` for page and section headings;
the default level is 3.

Use `cssClass` as an escape hatch for reusable CSS, pseudo-selectors, media
queries, syntax highlighting, or missing Forui helpers. Use `withKey` only for
stable reconciliation identity in dynamic lists, not for styling hooks.

**Router** — client-side routing. `navigate(path)` changes routes.
`routeParam(name)` extracts URL params. `Router` / `Route` components.
`Link(path, children)` for navigation links.

## Common imports

Imports are per file; there is no global namespace. For app UI files, prefer a
glob import from `Forui.view` so components and modifiers are both available
for direct calls and UFCS chains.

```fai
use * from Forui.view
use { useSignal, usePoll, isLoading, isError, setValue, reload, value } from Forui.signal
use { navigate, routeParam, Router, Route, Link } from Forui.router
use { mount } from Forui
```

```fai
VStack do
    Label('Hello')
        .fontSize(24)
        .fontWeight('700')
end
    .padding(16)
    .background('#ffffff')
```

Run `fai doc Forui.view`, `fai doc Forui.signal`, or `fai doc Forui.router` for
full function listings.
