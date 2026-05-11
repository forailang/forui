# Forui — forai UI Framework

Forui is a reactive UI framework for forai. It compiles to WebAssembly and renders
via a platform adapter (browser DOM, native, SSR). State is managed with signals;
the diff engine handles efficient updates.

## Core concepts

**`mount(app, render)`** — start the app. Call once with a render function and a
platform adapter. The adapter receives a diff list after every signal change.

**Signals** — reactive values. Create with `useSignal(initialValue)`. Mutate with
`setValue(sig, newValue)` (requires `mutable` param). Read with `sig.value`.
Status helpers: `isLoading(sig)`, `isLoaded(sig)`, `isError(sig)`.

**View tree** — build UI by returning `ViewNode` from your render function.
Primitives: `Label`, `Paragraph`, `Heading`, `Button`, `TextInput`, `Toggle`,
`VStack`, `HStack`, `ZStack`, `ScrollView`, `Spacer`, `Divider`, `ImageView`,
`SegmentedControl`.
Modifiers (all must be imported): `padding`, `background`, `foreground`,
`cornerRadius`, `fontSize`, `fontWeight`, `lineHeight`, `fontStyle`,
`letterSpacing`, `textAlign`, `fontFamily`, `flex`, `alignItems`,
`justifyContent`, `flexWrap`, `gap`, `alignSelf`, `display`, `width`,
`minWidth`, `maxWidth`, `height`, `minHeight`, `position`, `top`, `zIndex`,
`centered`, `border`, `borderLeft`, `opacity`, `cssClass`, `withKey`.

Use `Label` for short UI/control text. Use `Paragraph` for body copy and
multiline prose. Use `Heading(text, level: n)` for page and section headings;
the default level is 3.

Use `cssClass` as an escape hatch for reusable CSS, pseudo-selectors, media
queries, syntax highlighting, or missing Forui helpers. Use `withKey` only for
stable reconciliation identity in dynamic lists, not for styling hooks.

**Router** — client-side routing. `navigate(path)` changes routes.
`routeParam(name)` extracts URL params. `Router` / `Route` components.
`Link(path, children)` for navigation links.

## Required imports (per file — no global namespace)

```fai
use { Label, Button, VStack, fontSize, foreground, padding } from Forui.view
use { useSignal, isLoading, isError, setValue, reload } from Forui.signal
use { navigate, routeParam, Router, Route, Link } from Forui.router
use { mount } from Forui
```

Run `fai doc Forui.view`, `fai doc Forui.signal`, or `fai doc Forui.router` for
full function listings.
