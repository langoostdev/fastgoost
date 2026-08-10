# fastgoost

A declarative web framework for
[Langoost](https://github.com/langoostdev/langoost). You declare what a route
accepts, and that one declaration does three jobs: it validates the request, it
documents the endpoint, and it hands the handler typed values instead of
strings.

```goost
import { newApp } from "fastgoost"

let app = newApp({title: "Example", version: "1.0.0"})

app.get("/hello/{name}", fn(req) {
    return {greeting: "hello " + req.params.name}
})

app.run(8080)
```

```sh
langoost run app.goost
```

```
╭────────────────────────────────────────────────╮
│  fastgoost 1.0.0                               │
│  Example  1.0.0                                │
│                                                │
│  http://localhost:8080                         │
│  http://localhost:8080/docs  docs              │
├────────────────────────────────────────────────┤
│  GET    /hello/{name}                          │
╰────────────────────────────────────────────────╯
```

---

## What is here

| | |
|---|---|
| **Routing** | path parameters with converters, greedy segments, specificity ordering, 404 / 405 / OPTIONS handled for you |
| **Validation** | declarative schemas that coerce as well as check; every problem reported at once, as a `422` with a `detail` list |
| **Documentation** | OpenAPI 3.1 generated from the routes, served with Swagger UI and ReDoc |
| **Middleware** | onion model, global or per-router or per-route; logger, CORS, request-id, security headers, basic auth, rate limit, timing included |
| **Dependencies** | `fn(req) → value` resolved per route; throwing from one rejects the request before the handler runs |
| **Responses** | JSON, text, HTML, redirects, files, cookies — as functions or as a chainable builder |
| **Streaming** | Server-Sent Events and WebSockets, on the same port as everything else |
| **Testing** | drive the whole pipeline without binding a port |
| **Server** | HTTP/1.1 on raw sockets: keep-alive, chunked request bodies, `Expect: 100-continue`, HEAD, a thread per connection |

Not here: TLS (put a reverse proxy in front), HTTP/2 (the `http2` stdlib module
has it), an ORM, a template engine.

---

## Install

The package is the `fastgoost/` directory. Copy it into your project, or
declare it in `langoost.json`:

```json
{
  "dependencies": {
    "fastgoost": "github:langoostdev/fastgoost@v0.0.1"
  }
}
```

```sh
langoost install
```

Either way, `import { newApp } from "fastgoost"` resolves it.

> **Requires Langoost with the module-scoped loader.** A package that spans
> several files needs relative imports to resolve against the importing file's
> directory. Older builds resolve them against the search path, which makes
> `fastgoost/main.goost`'s `import { compile } from "./routing.goost"` fail
> from any other working directory. See [Notes on Langoost](#notes-on-langoost).

---

## The tour

### Routes

A handler takes one argument and returns a value. An object or array becomes
JSON, a string becomes `text/plain`, nothing becomes `204`.

```goost
app.get("/items", listItems)
app.post("/items", createItem)
app.put("/items/{id:int}", replaceItem)
app.patch("/items/{id:int}", updateItem)
app.delete("/items/{id:int}", removeItem)
app.all("/anything", handleAnything)
app.route("REPORT", "/things", handleReport)     // any method, as data
```

Path parameters convert as they match, and a segment of the wrong type does
not match at all — the handler never sees `"abc"` where it asked for a number:

```goost
app.get("/users/{id:int}", fn(req) { return {id: req.params.id} })   // int
app.get("/files/{rest:path}", fn(req) { return req.params.rest })    // swallows slashes
app.get("/assets/*", serveAsset)                                     // same, unnamed
```

Converters: `int`, `float`, `bool`, `str` (the default), `path`.

Order of registration does not matter. `/files/latest` outranks
`/files/{name}` because it is more specific, not because it was registered
first.

### Declaring what a route accepts

```goost
import { Str, Int, Bool, List } from "fastgoost"

let NewItem   = {title: Str({minLength: 1, maxLength: 200}), done: Bool({default: false})}
let ListQuery = {limit: Int({default: 20, min: 1, max: 100}), tag: List(Str(), {default: []})}

app.get("/items",  {handler: listItems,  query: ListQuery, summary: "List items"})
app.post("/items", {handler: createItem, body: NewItem, status: 201, summary: "Create an item"})
```

The second argument is either a bare handler or a spec object carrying one.
Inside a handler:

- `req.query` — validated and typed (`limit` is an `int`, not `"20"`)
- `req.data` — the validated body, with defaults filled in
- a bad request never arrives; it gets a `422` listing everything wrong

```json
{"detail": [
  {"loc": ["body", "title"], "msg": "must have at least 1 character", "type": "too_short"},
  {"loc": ["query", "limit"], "msg": "must be <= 100", "type": "less_than_equal"}
]}
```

Spec keys: `kind`, `required`, `default`, `nullable`, `min`, `max`,
`minLength`, `maxLength`, `pattern`, `enum`, `items`, `fields`, `description`,
`example`. Write them by hand or with `Str`/`Int`/`Num`/`Bool`/`List`/`Obj`/`Any`.

Route spec keys: `handler`, `summary`, `description`, `tags`, `status`,
`query`, `body`, `headers`, `responses`, `middleware`, `deps`, `name`,
`operationId`, `hidden`.

### Responses

Two spellings of the same thing:

```goost
return {ok: true}                        // JSON, 200
return json(item, 201)                   // JSON, 201
return res.status(201).json(item)        // the same

return res.status(201)
          .header("Location", "/items/" + id)
          .cookie("session", token)
          .json(item)

return res.html("<h1>hi</h1>")
return res.redirect("/items")
return res.file("./report.pdf")
return res.noContent()
```

Your own class can be a response too — give it `intoResponse()` and return it.

### Errors

```goost
throw err.notFound("no user " + id)
throw err.forbidden("not your item")
throw err.withHeaders(err.unauthorized("sign in"), {"WWW-Authenticate": "Bearer"})
abort(404)                                // the same, without writing `throw`
```

An unexpected error — one that is not an HTTP error — is logged in full and
answered with a bare `500`. The message never reaches the client.

Override any of it:

```goost
app.onError(404, fn(req, error) { return res.status(404).html(notFoundPage) })
app.onException(fn(req, error) { return res.status(500).json({id: reportToSentry(error)}) })
```

### Middleware

`fn(req, next)`. Call `next(req)` to continue, or return something else to stop.

```goost
import { logger, cors, requestId, securityHeaders, basicAuth, rateLimit, timing } from "fastgoost/middleware"

app.use(requestId())
app.use(logger())
app.use(cors({origins: ["https://example.com"], credentials: true}))
app.use(rateLimit({limit: 100, windowMs: 60000}))

app.use(fn(req, next) {                   // or your own
    req.state.startedAt = time.clock.unixMilli()
    return next(req)
})
```

The first registered is the outermost. Global middleware sees every request,
including the ones that 404.

### Routers

```goost
let users = newRouter("/users", {tags: ["users"]})
users.use(requireAuth)
users.get("/{id:int}", showUser)

app.include(users)                        // /users/{id}
app.mount("/api/v1", users)               // /api/v1/users/{id}
```

A router is an application you never call `run()` on.

### Dependencies

`fn(req) → value`, resolved per route, landing on `req.deps`. One that throws
rejects the request before the handler runs, which is what lets a dependency
*be* the authorisation check:

```goost
fn currentUser(req) {
    let payload = verify(req.cookie("session"), SECRET)
    if typeof(payload) != "string" { throw err.unauthorized("sign in first") }
    return lookupUser(payload)
}

app.get("/me", {handler: fn(req) { return req.deps.user }, deps: {user: currentUser}})
```

### Streaming

```goost
app.sse("/events", fn(stream) {
    while stream.alive {
        stream.sendJson("tick", {at: time.now()})
        time.sleep(1000)
    }
})

app.ws("/ws", fn(socket) {
    while true {
        let message = socket.recv()
        if typeof(message) == "void" { break }
        socket.send("echo: " + message)
    }
})
```

Both hold the connection for as long as the handler runs, on the same port as
the rest of the application. `stream.alive` goes false the moment a write
fails, which is how you notice a closed tab.

### Static files

```goost
app.static("/assets", "./public")
app.static("/", {dir: "./dist", fallback: "index.html"})     // single-page app
```

Paths are normalised before they touch the filesystem, so a request cannot
climb out of the root.

### Documentation

`/docs` (Swagger UI), `/redoc` and `/openapi.json` are served automatically
from the route declarations. Turn them off with `newApp({docs: false})`, or
move them with `{docsUrl: "/internal/docs", openapiUrl: "/internal/spec.json"}`.

### Testing

```goost
import { request } from "fastgoost/testing"
import { eq, ok } from "assert"

fn test_creates_an_item() {
    let resp = request(app, "POST", "/items", {json: {title: "milk"}})
    eq(resp.status, 201)
    eq(resp.json().title, "milk")
    eq(resp.header("location"), "/items/1")
}
```

The request goes through middleware, routing, validation, dependencies and
error handling, and stops just before the bytes would hit a socket.

```sh
langoost test tests
```

---

## Examples

| File | Shows |
|---|---|
| [examples/hello.goost](examples/hello.goost) | the smallest thing that is still a service |
| [examples/basics.goost](examples/basics.goost) | path params, query strings, JSON and form POSTs, headers, cookies, uploads — one route each, with the curl line |
| [examples/todo_api.goost](examples/todo_api.goost) | CRUD with validation, routers, documentation and errors |
| [examples/auth.goost](examples/auth.goost) | sessions, dependencies, an admin group |
| [examples/realtime.goost](examples/realtime.goost) | SSE and WebSockets, with a page that uses both |

```sh
langoost run examples/basics.goost
```

---

## Configuration

```goost
let app = newApp({
    title:       "My API",          // documentation metadata
    version:     "1.0.0",
    description: "…",
    prefix:      "/api",            // every route sits under this
    docs:        true,              // false serves no documentation at all
    openapiUrl:  "/openapi.json",
    docsUrl:     "/docs",
    redocUrl:    "/redoc",
})

app.run({
    port:            8080,
    host:            "0.0.0.0",
    idleTimeoutMs:   30000,         // how long a kept-alive socket may sit silent
    headerTimeoutMs: 10000,         // deadline for the request line and headers
    maxBodyBytes:    8388608,       // 8 MiB
    maxHeaders:      100,
    keepAlive:       true,
    maxConnections:  512,           // in-flight connections; 0 disables the cap
    serverHeader:    "fastgoost",
    banner:          true,
})
```

`app.onStartup(fn(app) { … })` runs before the first request — open a database
there. `app.onShutdown(fn(app) { … })` runs on SIGINT and SIGTERM where the
build supports signals.

---

## Layout

```
fastgoost/
├── main.goost         the public API — App, Request, Response, res, err, schemas
├── middleware.goost   logger, cors, requestId, securityHeaders, basicAuth, rateLimit, timing
├── testing.goost      request(app, method, path, options)
├── banner.goost       the startup panel
├── routing.goost      compiling, scoring and matching path patterns
├── validate.goost     schemas, coercion, constraints
├── httpx.goost        status phrases, MIME types, headers, cookies, queries
├── server.goost       the HTTP/1.1 connection loop
├── ws.goost           RFC 6455 framing, server side
├── openapi.goost      spec generation and the documentation UIs
└── support.goost      the few helpers more than one module needs
```

You import `fastgoost`, `fastgoost/middleware` and `fastgoost/testing`. The
rest is machinery, though nothing stops you importing it — `routing.goost` and
`validate.goost` are useful on their own.

---

## Notes on Langoost

Building this turned up five things worth knowing, four of which shaped the
API. They are listed here because the workarounds look arbitrary otherwise.

**Relative imports resolve against the search path.** A package file could not
import its own siblings: `fastgoost/main.goost` doing
`import { compile } from "./routing.goost"` looked in the running script's
directory and the working directory, never in `fastgoost/`. This is fixed in
`src/runtime/module/loader.go` — a module's VM now gets a loader scoped to that
module's own directory — and fastgoost needs that fix to load at all.

**A class method cannot be variadic.** `compileMethod` sets
`Arity: len(fn.Params) + 1` and ignores the variadic flag, so `...opts` in a
method binds as an ordinary parameter and the short call fails the arity check.
That is why registration takes a handler *or* a spec object rather than an
optional third argument, and why `res.status(201).json(x)` exists alongside
`json(x, 201)`. Top-level functions are unaffected.

**`extends` does not survive a module boundary.** `lookupMethod` resolves a
class's `ParentSlot` in the *calling* VM's globals, so a subclass exported from
a library loses its parent's methods the moment another module calls one. App
and Router are therefore one class rather than two.

**`typeof` reports `"closure"`.** The documented set is `"function"`; anonymous
functions and anything that captured a variable report `"closure"`, which is
almost every handler. Check both — `isCallable` in `support.goost`.

**The bundled `websocket` module does not compile.**
`goostlib/websocket.goost` declares `fn upgradeHeaders(…): string[]` and
returns an array literal, which the return-type checker rejects, so importing
it fails the whole program. `ws.goost` reimplements the parts needed.

One limitation has no workaround: a **non-catchable** runtime error inside a
handler — an out-of-range index, a division by zero — kills that connection's
thread without unwinding, so the socket is never closed and the client waits
until it gives up. The process and every other connection carry on. `try`
cannot catch these by design, so check bounds and divisors in handlers.

---

## Licence

MIT — see [LICENSE](LICENSE).
