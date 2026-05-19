# Forui.view

`Forui.view` is the app-facing UI surface for building `ViewNode` trees. Most
Forui components import it with a glob so components, event helpers, style
modifiers, and test helpers are all in file scope for normal calls and UFCS
chains:

```fai
use * from Forui.view
```

## Components

Containers:

- `Window(title) do ... end` - app shell wrapper
- `View do ... end` - generic grouping node
- `VStack do ... end` - vertical layout
- `HStack do ... end` - horizontal layout
- `ZStack do ... end` - overlapping layout
- `ScrollView do ... end` - scrollable layout

Text:

- `Label(text)` - short UI/control text
- `Paragraph(text)` - body copy and prose
- `Heading(text, level: n)` - semantic headings
- `Span(text)` - inline text, often with `cssClass`
- `Pre do ... end` and `Code do ... end` - preformatted/code content

Controls:

- `Button(text, onClick: do ... end)`
- `TextInput(placeholder, signalValue: signal)`
- `Toggle(isOn: value, signalValue: signal)`
- `SegmentedControl(options, signalValue: signal)`
- `Link` lives in `Forui.router`, not `Forui.view`

## Modifiers

Modifiers return a new `ViewNode` and are designed for UFCS chains:

```fai
VStack do
    Label('Settings')
        .fontSize(24)
        .fontWeight('700')
    Paragraph('Manage your account preferences.')
        .foreground('#6d6d72')
end
    .padding(24)
    .background('#ffffff')
    .cornerRadius(8)
```

Common modifier groups:

- spacing: `padding`, `paddingTop`, `paddingRight`, `paddingBottom`,
  `paddingLeft`, `margin`, `marginTop`, `marginRight`, `marginBottom`,
  `marginLeft`, `gap`
- color and borders: `background`, `foreground`, `border`, `borderLeft`,
  `cornerRadius`, `opacity`
- typography: `fontSize`, `fontWeight`, `fontStyle`, `fontFamily`,
  `lineHeight`, `letterSpacing`, `textAlign`
- layout: `flex`, `alignItems`, `justifyContent`, `flexWrap`, `alignSelf`,
  `display`, `width`, `minWidth`, `maxWidth`, `height`, `minHeight`,
  `centered`, `position`, `top`, `zIndex`
- escape hatches: `cssClass` for reusable CSS and `withKey` for stable list
  reconciliation identity

## Events

Prefer component parameters when available:

```fai
Button('Save', onClick: do
    save()
end)

TextInput('Name', signalValue: name, onSubmit: do
    submit()
end)
```

Use `onClick` and `onChange` modifiers when attaching handlers to an existing
node expression is clearer.

## Testing

Use `Forui.testMount` or `Forui.testMountAt` to render a component without a
browser, then inspect the tree with:

- `childCount`
- `getChild`
- `getProp`
- `getIntProp`
- `hasProp`
- `findByKind`
- `findByProp`

```fai
let tree = testMount(SettingsPage)
let title = findByProp(tree, 'text', 'Settings')
assert.isNotNull(title)
```

## Internal Helpers

The module also exposes lower-level frame, registration, and event bridge
helpers such as `pushFrame`, `popFrame`, `registerNode`,
`installViewEventBridge`, `attachClickHandler`, and `registerHandler`.
Application code normally should not call these directly; they are used by
component constructors, modifiers, render adapters, and tests.
