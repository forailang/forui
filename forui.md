# Forui — Framework Reference

Forui is a declarative, signals-driven UI framework for forai.
Components are plain functions that return `ViewNode`.
State lives in signals — change a signal, the framework re-renders.

---

## Package Names and Import Paths

Two packages are required for web apps:

| Package | `fai.toml` name | What it provides |
|---------|-----------------|------------------|
| `forui` | `Forui` | Components, signals, routing |
| `html-forui` | `HtmlForui` | Browser rendering, SSR, router init |

**Import paths — use these exactly:**

```fai
use { VStack, HStack, ZStack, ScrollView, View, Window } from Forui.view
use { Label, Button, TextInput, Toggle, SegmentedControl } from Forui.view
use { Spacer, Divider, ImageView } from Forui.view
use { ViewNode, Children } from Forui.view

use { useSignal, setValue } from Forui.signal

use { Router, Route, Link } from Forui.router
use { navigate, routeParam, currentPath } from Forui.router

use { mount, renderSSR } from Forui

use { htmlRender, initWebRouter } from HtmlForui
```

Getting `Unknown name 'HStack'` or `Unknown type 'ViewNode'`? The import
path is wrong. Every symbol must be imported explicitly — there are no
wildcard re-exports from a top-level `Forui` module.

---

## fai.toml — Fullstack Project Setup

```toml
[project]
name = "MyApp"
version = "0.1.0"
source_root = "src"

# Client: compiles to WASM, runs in the browser
[project.client]
target = "wasm-html"
source = "src"
main = "src/client/main.fai"
build_dir = "build/client"

# Server: compiles to native binary
[project.server]
target = "native"
source = "src"
main = "src/server/main.fai"
build_dir = "build/server"

[dependencies]
Forui     = "file:///path/to/forui"
HtmlForui = "file:///path/to/html-forui"
# Or fetch from public git repos:
# Forui = "https://github.com/forailang/forui"

# Client calls server functions over RPC
[project.client.dependencies.server.remote.dev]
url = "http://localhost:3040"
```

**CRITICAL: Targets are compile-time isolated.**

The `client` and `server` targets compile **completely independently**. The
client cannot import from the server's source tree at compile time, and vice
versa. The only connection is the RPC bridge declared in `fai.toml`.

```
✓  use { getTasks } from Server     # RPC call — works on client
✗  use { getTasks } from server.main # direct import — compile error
```

---

## Project Structure

```
src/
  client/
    main.fai          — mount(App, htmlRender) + initWebRouter()
    app.fai           — Router with all Route definitions
    state/
      state.fai       — module-level shared state (session, etc.)
    components/
      topbar.fai      — reusable layout components
      card.fai
    pages/
      home.fai        — one file per route
      tasks.fai
      auth/
        login.fai
        signup.fai
  server/
    main.fai          — HTTP server, addRpcRoutes(), serveFiles(), listen()
    handlers/
      tasks.fai       — business logic per domain
      auth.fai
  shared/             — types used by both client and server
    types.fai         — Task, User, Session — shared DTOs
```

### Shared types across client and server

Both targets set `source = "src"` in `fai.toml`, so both compile the entire
`src/` tree. Types defined in `src/shared/types.fai` are visible to both
client and server code without any special imports — just reference them
directly.

```fai
# src/shared/types.fai
type Task
  id Int
  title String
  done Bool
end

type Session
  token String
  userId Int
end
```

```fai
# src/server/handlers/tasks.fai — uses Task without importing it
def getTasks
    @param token String
    @return Task[]
do
  # ...
end
```

```fai
# src/client/pages/tasks.fai — also uses Task directly
var tasks = useSignal([]) do
  getTasks(sessionToken())    # RPC call → returns Task[]
end
```

### All files in a directory = one module namespace

Every `.fai` file in a directory is loaded as part of the **same module**.
There is no file-level scoping. If you define `type Task` in both `types.fai`
and `main.fai` within the same directory, you get a duplicate export error.

```
✗  src/shared/types.fai   defines  type Task
✗  src/shared/main.fai    defines  type Task   ← duplicate export error

✓  src/shared/types.fai   defines  type Task   ← only once, anywhere in src/shared/
```

The `_` prefix (e.g. `_types.fai`) only controls **load order** — it does
not create a separate namespace.

---

## Layout Primitives

Containers take a `do...end` block of children:

```fai
VStack do
  HStack do
    Label('left')
    Spacer()
    Label('right')
  end
  Divider()
  Label('below')
end
```

| Container | Description |
|-----------|-------------|
| `VStack do ... end` | Vertical column |
| `HStack do ... end` | Horizontal row |
| `ZStack do ... end` | Layered (z-order) |
| `ScrollView do ... end` | Scrollable container |
| `View do ... end` | Generic wrapper |
| `Window('title') do ... end` | Top-level window |

---

## Leaf Elements

```fai
Label('Hello, world')
Paragraph('Body copy for a prose block')
Heading('Section title')
Heading('Page title', level: 1)
Button('Save', onClick: do ... end)
TextInput('placeholder', signalValue: mySignal)
TextInput('Email', signalValue: email, inputType: 'email')
TextInput('Password', signalValue: pw, inputType: 'password')
TextArea('Notes', signalValue: notes, rows: 6)
Toggle(isOn: flag.value, onChange: do with v Bool flag.setValue(v) end)
SegmentedControl(options: ['Day', 'Week'], signalValue: tabSignal)
Spacer()
Divider()
ImageView('/logo.png')
```

`Label` is for short UI/control text such as nav labels, badges, captions,
and form labels. Use `Paragraph` for body prose and `Heading` for semantic page
or section headings. `Heading` defaults to level 3.

---

## Modifiers

Every `ViewNode` has modifier functions that return a new modified node.
Chain them freely:

```fai
Label('Title')
  .fontSize(24)
  .fontWeight('700')
  .foreground('#1c1c1e')
  .padding(16)
  .background('#ffffff')
  .cornerRadius(12)
  .opacity(0.8)
  .flex('1')
  .onClick(do print('tapped') end)
```

| Modifier | Argument | Effect |
|----------|----------|--------|
| `.padding(n)` | `Int` | Uniform padding |
| `.background(color)` | `String` | Background colour |
| `.foreground(color)` | `String` | Text / icon colour |
| `.cornerRadius(n)` | `Int` | Border radius |
| `.fontSize(n)` | `Int` | Font size |
| `.fontWeight(w)` | `String` | `'400'`, `'600'`, `'700'`, etc. |
| `.lineHeight(v)` | `String` | CSS line-height (`'1.6'`, `'24px'`) |
| `.fontStyle(v)` | `String` | CSS font-style (`'italic'`) |
| `.letterSpacing(v)` | `String` | CSS letter-spacing (`'0.02em'`) |
| `.textAlign(v)` | `String` | CSS text-align |
| `.fontFamily(v)` | `String` | CSS font-family |
| `.flex(v)` | `String` | CSS flex value (`'1'`, `'0 0 auto'`) |
| `.alignItems(v)` | `String` | Flex cross-axis alignment |
| `.justifyContent(v)` | `String` | Flex main-axis alignment |
| `.flexWrap(v)` | `String` | Flex wrapping |
| `.gap(n)` | `Int` | Flex/grid gap in px |
| `.alignSelf(v)` | `String` | Child alignment in flex parent |
| `.display(v)` | `String` | CSS display |
| `.width(v)` / `.minWidth(v)` / `.maxWidth(v)` | `String` | CSS width constraints |
| `.height(v)` / `.minHeight(v)` | `String` | CSS height constraints |
| `.position(v)` / `.top(n)` / `.zIndex(n)` | `String` / `Int` | Positioning helpers |
| `.centered()` | none | Horizontal auto margins |
| `.border(n, color)` / `.borderLeft(n, color)` | `Int`, `String` | Solid borders |
| `.opacity(v)` | `Float` | `0.0`–`1.0` |
| `.cssClass(name)` | `String` | Stylesheet escape hatch |
| `.withKey(k)` | `String` | Stable identity for diffing, not styling |
| `.onClick(do...end)` | `ClickAction` | Click handler |
| `.onChange(do with v String...end)` | `ChangeAction` | Change handler |

Use `cssClass` for reusable stylesheet rules, pseudo-selectors, media queries,
syntax highlighting, or missing Forui helpers. Use `withKey` only for stable
identity in dynamic lists.

---

## Signals

Signals are reactive state containers. Changing a signal's value schedules
a re-render.

### Create

```fai
# Simple signal with an initial value
var count = useSignal(0)

# Signal with a loader — fetches data asynchronously on first use
var tasks = useSignal([]) do
  getTasks(sessionToken())   # runs via nowait; component re-renders on completion
end
```

`useSignal` must be called at the **top level of a component function**, not
inside a conditional or loop. The framework tracks signals by call order.

### Read

```fai
count.value          # current value
count.isInitial()    # never loaded
count.isLoading()    # fetch in progress
count.isLoaded()     # has a value
count.isError()      # failed
count.error          # error message (String?)
```

### Write — triggers re-render

```fai
count.setValue(count.value + 1)   # set value + mark loaded
tasks.setLoading()                # mark loading
tasks.setError('Network error')   # mark error
tasks.reload()                    # re-run the loader
```

### Loading states in UI

```fai
var items = useSignal([]) do
  fetchItems()
end

VStack do
  if items.isLoading()
    Label('Loading…')
  else
    if items.isError()
      Label('Error: ' + items.error!).foreground('#ff3b30')
    else
      for item in items.value
        Label(item.name)
      end
    end
  end
end
```

---

## Reusable Components — `Children` vs `ViewNode`

**Use `Children` when a helper wraps other nodes. Never pass `ViewNode` as a
parameter to a layout wrapper.**

This is the most common mistake. Here is why it matters:

A `ViewNode` constructor (`Label`, `Button`, `VStack`, etc.) registers
itself into whichever **frame is active at the moment it is called**. If
you call `Label('hi')` inside a function that passes the result as a
`ViewNode` argument, it registers in the *caller's* frame — not in your
helper's frame. The helper's container then wraps an empty child tree.

The `Children` closure defers construction. When the helper calls
`builder()`, the children are built *inside* the helper's own frame.

```fai
# ✓ Correct — accepts Children closure
def Card
    @param title String
    @param builder Children
    @return ViewNode
do
  VStack do
    Label(title).fontSize(13).foreground('#6d6d72')
    builder()            # children build inside VStack's frame
  end
  .padding(16)
  .background('#ffffff')
  .cornerRadius(12)
end

# Usage
Card('SETTINGS') do
  Label('Name')
  Button('Edit', onClick: do ... end)
end
```

```fai
# ✗ Wrong — accepts pre-built ViewNode
def Card
    @param title String
    @param content ViewNode    # already registered in caller's frame!
    @return ViewNode
do
  VStack do
    Label(title)
    # content is NOT here — it was registered before Card() ran
  end
end
```

---

## Routing

Define all routes once in `app.fai`. The `Router` renders the matching
`Route` for the current browser path:

```fai
use { Router, Route } from Forui.router
use { HomePage, TasksPage, LoginPage } from client.pages

def App
    @return ViewNode
do
  Router do
    Route('/') do
      HomePage()
    end
    Route('/tasks') do
      TasksPage()
    end
    Route('/task/:id') do
      TaskDetailPage()
    end
    Route('/login') do
      LoginPage()
    end
    Route('*') do
      Label('Page not found')
    end
  end
end
```

Navigate and link:

```fai
use { navigate, routeParam, Link } from Forui.router

# Programmatic navigation
Button('Go', onClick: do navigate('/tasks') end)

# Declarative link
Link('/tasks') do
  Label('My Tasks')
end

# Read a path parameter (/task/:id → id)
let id = routeParam('id')    # String?
```

---

## Server RPC

Functions defined in the server are callable from the client as if local.
The RPC bridge is configured in `fai.toml` via `[project.client.dependencies.server.remote.dev]`.

**Server — write plain forai functions:**

```fai
# src/server/handlers/tasks.fai
def getTasks
    @param token String
    @return Task[]
do
  # validate token, query DB
  tasks
end

def addTask
    @param token String
    @param title String
    @return Task
do
  # insert, return new task
  task
end
```

**Server `main.fai` — wire up routes and start listening:**

```fai
use std.http.server

def main
    @return Void
do
  var r = server.router()
  addRpcRoutes(r)                               # auto-generated from server functions
  server.serveFiles(r, 'build/client')          # serve WASM + assets
  server.get(r, '*') do with req HttpRequest    # SSR fallback
    server.html(200, renderSSR(App, req.path))
  end
  server.listen(r, 3040)
end
```

**Client — call server functions directly:**

```fai
use { getTasks, addTask } from Server    # 'Server' = the RPC target name

var tasks = useSignal([]) do
  getTasks(sessionToken())
end

Button('Add', onClick: do
  try
    addTask(sessionToken(), input.value)
    tasks.reload()
    input.setValue('')
  catch e
    err.setValue(e.message)
  end
end)
```

---

## Global State

Module-level `var` variables persist for the lifetime of the WASM session.
Use them in `state/state.fai` for auth sessions, user preferences, etc.

```fai
# src/client/state/state.fai

type Session
  token String
  userId Int
  userName String
end

var currentSession Session? = null

def isLoggedIn
    @return Bool
do
  currentSession != null
end

def sessionToken
    @return String
do
  currentSession!.token
end

def setSession
    @param s Session
    @return Void
do
  currentSession = s
end

def clearSession
    @return Void
do
  currentSession = null
end
```

Components read state directly — no import needed within the same target's
source tree:

```fai
if isLoggedIn()
  Label('Hello, ' + sessionToken())
else
  Link('/login') do Label('Log in') end
end
```

---

## Client Entry Point

```fai
# src/client/main.fai
use { mount } from Forui
use { htmlRender, initWebRouter } from HtmlForui
use { App } from client.app

def main
    @return Void
do
  mount(App, htmlRender)
  initWebRouter()
end
```

---

## Complete Page Example

```fai
# src/client/pages/auth/login.fai
use { VStack, Label, Button, TextInput } from Forui.view
use { useSignal } from Forui.signal
use { Link, navigate } from Forui.router
use { login } from Server
use { setSession } from client.state
use { TopBar, Card } from client.components

def LoginPage
    @return ViewNode
do
  var email    = useSignal('')
  var password = useSignal('')
  var err      = useSignal('')

  VStack do
    TopBar()

    Card('SIGN IN') do
      TextInput('Email', signalValue: email, inputType: 'email')
      TextInput('Password', signalValue: password, inputType: 'password')

      if err.value != ''
        Label(err.value).foreground('#ff3b30').fontSize(13)
      end

      Button('Log in', onClick: do
        if email.value == '' or password.value == ''
          err.setValue('Please fill in all fields')
        else
          try
            let s = login(email.value, password.value)
            setSession(s)
            navigate('/tasks')
          catch e
            err.setValue(e.message)
          end
        end
      end)

      Link('/signup') do
        Label('No account? Sign up →').foreground('#007aff')
      end
    end
    .padding(20)
  end
end

test LoginPage
it 'renders without crashing'
  let node = LoginPage()
  assert.equals(node.kind, 'vstack')
end
end
```

---

## Tests

Every function must have at least one test — `fai test` fails otherwise.
Write tests in the same file as the function:

```fai
def greet
    @param name String
    @return String
do
  'Hello, ' + name + '!'
end

test greet
it 'produces greeting'
  assert.equals(greet('Alice'), 'Hello, Alice!')
end
end
```

For UI components, at minimum assert the root node kind:

```fai
test TaskRow
it 'renders as hstack'
  let node = TaskRow(Task(id: 1, title: 'Test', done: false))
  assert.equals(node.kind, 'hstack')
end
end
```

---

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| `Unknown name 'HStack'` | Add `use { HStack } from Forui.view` |
| `Unknown type 'ViewNode'` | Add `use { ViewNode } from Forui.view` |
| `Unknown type 'Children'` | Add `use { Children } from Forui.view` |
| Importing client code from server | Not possible — targets are isolated |
| `duplicate export 'Task'` | Type defined in two files in same directory — remove one |
| `children render as siblings` | Pass `Children` closure, not `ViewNode` parameter |
| `missing tests for N functions` | Write `test` blocks before running |
| `Optional check requires optional` | Remove `!` — value is not optional |
| Using `return x` | forai has no `return` — last expression is returned |
| `[1 2 3]` arrays | Arrays use commas: `[1, 2, 3]` |
| `'hello {{x}}'` interpolation | Only double-quoted strings interpolate: `"hello {{x}}"` |
