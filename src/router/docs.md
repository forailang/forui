# Forui.router

`Forui.router` provides client-side routing for Forui apps. A `Router` collects
its child `Route` declarations and renders the first matching route for the
current path.

Typical app route files import the core pieces directly:

```fai
use { Router, Route, Link, navigate, routeParam } from Forui.router
```

## Defining Routes

Place one `Router` near the top of the app. Each `Route` takes a path pattern
and a trailing `do ... end` body that renders the matching page.

```fai
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
        Route('*') do
            NotFoundPage()
        end
    end
end
```

Routes are tested in declaration order. Put specific routes before the wildcard
fallback because the first match wins. Route patterns support literal segments,
named params such as `:id`, and `*` as a catch-all route.

Use `basePath` when the app is mounted below a path prefix:

```fai
Router(basePath: '/app', children: do
    Route('/') do
        HomePage()
    end
    Route('/settings') do
        SettingsPage()
    end
end)
```

## Links And Navigation

Use `Link` for normal in-app navigation that the user clicks. It renders as a
web link in the HTML adapter, so browser link behavior and accessibility work as
expected, while regular clicks are routed through `navigate`.

```fai
Link('/tasks') do
    Label('Tasks')
end
```

Use `navigate(path)` for imperative navigation after an action, such as a login,
logout, save, or auth guard.

```fai
Button('Save', onClick: do
    saveTask()
    navigate('/tasks')
end)
```

`goBack()` delegates to the platform back handler when one is installed.
`currentPath()` returns the current router path.

## Route Params

Read a named `:param` from the route currently being rendered with
`routeParam(name)`. It returns `String?`, so handle the missing case or unwrap
with a fallback.

```fai
def TaskDetailPage
    @return ViewNode
do
    let idText = routeParam('id')
    if idText?
        TaskDetailView(idText!)
    else
        Label('Missing task id')
    end
end
```

## Browser Integration

`Forui.router` owns router state, matching, and route rendering. Browser history
integration lives in `HtmlForui.initWebRouter`; call that from the web client
entry after mounting the app.

```fai
use { mount } from Forui
use { htmlRender, initWebRouter } from HtmlForui
use { App } from platforms.web.routes

mount(App, htmlRender)
initWebRouter()
```

`setPathFromPlatform`, `onNavigate`, and `onBack` are adapter-facing hooks used
by `HtmlForui` and tests. App code usually uses `Link`, `navigate`, `goBack`,
and `routeParam` instead. Matching helpers such as `splitPath`, `pathMatches`,
`extractParams`, `stripBase`, and `findMatch` are exposed for tests and
framework integration, but most applications do not need them directly.
