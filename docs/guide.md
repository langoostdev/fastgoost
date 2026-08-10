# fastgoost guide

The README is the tour. This is the reference: every type, every method, every
option, and the reasoning where a choice is not obvious.

- [Application](#application) · [Routes](#routes) · [Request](#request) · [Responses](#responses)
- [Errors](#errors) · [Validation](#validation) · [Middleware](#middleware) · [Dependencies](#dependencies)
- [Routers](#routers) · [Static files](#static-files) · [Streaming](#streaming)
- [OpenAPI](#openapi) · [Testing](#testing) · [Running](#running) · [Deployment](#deployment)

---

## Application

```goost
import { newApp } from "fastgoost"

let app = newApp()
let app = newApp({title: "My API", version: "2.0.0"})
```

| Option | Default | Meaning |
|---|---|---|
| `title` | `"fastgoost"` | Shown in the banner and the spec |
| `version` | `"0.1.0"` | Your API's version, not the framework's |
| `description` | `""` | Prose for the spec |
| `prefix` | `""` | A path every route sits under |
| `docs` | `true` | `false` serves no documentation at all |
| `openapiUrl` | `"/openapi.json"` | Where the spec is served |
| `docsUrl` | `"/docs"` | Swagger UI; `""` to omit |
| `redocUrl` | `"/redoc"` | ReDoc; `""` to omit |
| `servers` | — | An OpenAPI server list |
| `tags` | `[]` | Tags applied to every route |

### Methods

| | |
|---|---|
| `get` `post` `put` `patch` `delete` `options` `head` `(path, spec)` | Register a route |
| `route(method, path, spec)` | The same with the method as data |
| `all(path, spec)` | One handler for every common method |
| `add(method, path, handler, options)` | The lowest form |
| `use(middleware)` | Add middleware |
| `include(router)` | Mount a router at its own prefix |
| `mount(prefix, router)` | Mount it under a further prefix |
| `sse(path, spec)` `ws(path, spec)` | Streaming endpoints |
| `static(urlPrefix, dirOrSpec)` | Serve a directory |
| `onStartup(fn)` `onShutdown(fn)` | Lifecycle hooks, each `fn(app)` |
| `onError(status, fn)` | Replace the response for one status; `fn(req, error)` |
| `onException(fn)` | Replace the response for an unexpected error |
| `openapi()` | The generated spec, as an object |
| `urlFor(name, params)` | Build a path for a named route |
| `handle(rawRequest)` | Run one request; what `testing` uses |
| `run(portOrOptions)` | Bind and serve; blocks |

`app.state` is an empty object that is yours. Handlers reach it through
`req.app.state`, and because it is a reference, every request thread sees the
same one.

---

## Routes

The second argument is a handler, or an object carrying one:

```goost
app.get("/items", listItems)

app.get("/items", {
    handler:     listItems,
    summary:     "List items",
    description: "Longer prose for the docs page",
    tags:        ["items"],
    status:      200,
    query:       ListQuery,
    body:        NewItem,
    headers:     {"X-Api-Key": Str()},
    responses:   {"404": "No such collection"},
    middleware:  [requireAuth],
    deps:        {db: openDatabase},
    name:        "item-list",
    operationId: "listItems",
    hidden:      false,
})
```

`status` is the code a successful return uses, so a handler that returns data
still answers `201` on a create. `hidden: true` keeps a route out of the docs
and the banner — health checks and internal endpoints.

### Path patterns

| Pattern | Matches | `req.params` |
|---|---|---|
| `/users` | `/users` | `{}` |
| `/users/{id}` | `/users/ada` | `{id: "ada"}` |
| `/users/{id:int}` | `/users/42` | `{id: 42}` |
| `/p/{v:float}` | `/p/1.5` | `{v: 1.5}` |
| `/p/{v:bool}` | `/p/yes` | `{v: true}` |
| `/files/{rest:path}` | `/files/a/b.txt` | `{rest: "a/b.txt"}` |
| `/assets/*` | `/assets/js/app.js` | `{"*": "js/app.js"}` |

`bool` accepts `true/false`, `1/0`, `yes/no`. A segment that does not convert
does not match, so `/users/{id:int}` answers `404` for `/users/abc` rather than
handing a handler a string.

A greedy segment must be last. A placeholder must fill a whole segment —
`/v{n}` is a route table bug and is reported at startup, not on the first
request that hits it.

### Which route wins

Patterns are scored by specificity, read left to right: a literal outranks a
placeholder, which outranks a wildcard. Registration order does not matter.

```goost
app.get("/files/{name}", byName)
app.get("/files/latest", latest)     // still wins for /files/latest
```

### Methods you do not register

- **`HEAD`** falls through to the `GET` route; the server sends the headers
  without the body.
- **`OPTIONS`** on a known path answers `204` with `Allow`.
- A known path with an unregistered method is `405` with `Allow`, not `404`.
- A trailing slash is the same path: `/items/` is `/items`.

---

## Request

Fields:

| | |
|---|---|
| `method` `path` `rawQuery` `version` | The request line |
| `headers` | Object, lowercased names |
| `rawHeaders` | The `"Key: Value"` lines as sent |
| `body` | The raw body |
| `params` | Path parameters, converted |
| `query` | Query parameters — validated and typed if the route declared them |
| `data` | The validated body |
| `state` | Yours; middleware writes here |
| `deps` | Resolved dependencies |
| `app` | The application |
| `route` | The matched route record |
| `remoteAddr` | The socket's peer |

Methods:

| | |
|---|---|
| `header(name)` | Case-insensitive; `""` when absent |
| `cookies()` `cookie(name)` | Parsed once per request |
| `bearer()` | The `Authorization: Bearer` token, or `""` |
| `contentType()` | Media type without parameters |
| `json()` | Parse the body; throws `400` if it is not JSON |
| `form()` | `{fields, files}` from url-encoded or multipart; throws `415` otherwise |
| `q(name)` `qOr(name, fallback)` | One query parameter |
| `param(name)` | One path parameter |
| `url()` | Path with query string |
| `accepts(mediaType)` | What the client said it would take |
| `isSecure()` | `X-Forwarded-Proto: https` |
| `clientIp()` | Left-most `X-Forwarded-For`, else the peer |

`isSecure` and `clientIp` read headers a client can set. Behind a proxy you
control they are the truth; exposed directly they are a suggestion.

---

## Responses

A handler returns a value:

| Returned | Sent |
|---|---|
| nothing | `204`, empty |
| a string | `200 text/plain; charset=utf-8` |
| an object or array | `200 application/json` |
| anything with `intoResponse()` | whatever it says |

### Functions

```goost
json(data)              json(data, 201)
text(body)              text(body, 202)
html(body)
redirect(location)      redirect(location, 301)
file(path)
noContent()
raw(status, headers, body)
```

### The builder

`res` is a shared, immutable starting point; every method returns a new one, so
concurrent requests cannot see each other's headers.

```goost
res.status(code)
   .header(name, value)
   .contentType(value)
   .cookie(name, value)                    // Path=/, HttpOnly, SameSite=Lax
   .cookieWith(name, value, options)       // maxAge, secure, sameSite, domain, path, httpOnly
   .clearCookie(name)
   .json(data) | .text(s) | .html(s) | .file(path) | .redirect(url) | .noContent() | .body(bytes)
```

A `Response` returned from a helper has the same `header`, `cookie`,
`cookieWith`, `clearCookie`, `contentType` and `withStatus` methods.

### Your own response type

```goost
class Csv {
    fn intoResponse() {
        return raw(200, {"Content-Type": "text/csv"}, self.rows)
    }
}

app.get("/export", fn(req) { return Csv{rows: buildCsv()} })
```

---

## Errors

```goost
throw err.notFound("no user " + id)
abort(404)
abort(404, "no user " + id)
throw httpError(404, "no user " + id)
```

`err` names the statuses worth naming: `badRequest` `unauthorized` `forbidden`
`notFound` `notAllowed` `conflict` `gone` `tooLarge` `unsupported` `invalid`
`tooMany` `internal` `unavailable`, plus `status(code, detail)` and
`withHeaders(error, headers)`. Each takes a detail — pass `""` for the status
code's own phrase.

The default body is `{"detail": …}`. Anything JSON-serialisable works as the
detail, which is how a `422` carries a list.

An error that is not an HTTP error is logged with its message and answered
with a bare `500`.

---

## Validation

A schema is a field map, or a single spec. A spec is an object:

| Key | Applies to | Meaning |
|---|---|---|
| `kind` | all | `"string"` `"int"` `"float"` `"number"` `"bool"` `"array"` `"object"` `"any"` |
| `required` | all | Default: true unless `default` is present |
| `default` | all | Used when the value is absent |
| `nullable` | all | Allow an explicit null |
| `min` `max` | numbers | Inclusive bounds |
| `minLength` `maxLength` | strings, arrays | Length bounds |
| `pattern` | strings | Regular expression |
| `enum` | all | Allowed values |
| `items` | arrays | Element spec |
| `fields` | objects | Nested field specs |
| `description` `example` | all | Documentation |

`kind` rather than `type` because `type` is a reserved word in Langoost and
`spec.type` does not parse. A quoted `"type":` key is accepted as a synonym.

The shorthands build the same objects: `Str(opts)`, `Int(opts)`, `Num(opts)`,
`Bool(opts)`, `Any(opts)`, `List(items, opts)`, `Obj(fields, opts)`.

### Coercion

A query string only carries text, so validation converts as well as checks:
`{kind: "int"}` against `"42"` yields the int `42`, and against `"abc"` yields
an error rather than a zero. A single value where a list was declared becomes a
one-element list, because that is what an HTML form sends.

`{kind: "int"}` accepts `36.0` and rejects `1.5` — a float that is not a whole
number is not an integer, and silently truncating it would be a lie.

### Errors

Everything wrong is reported at once, so a client fixing a form sees every
problem in one round trip:

```json
{"detail": [{"loc": ["body", "email"], "msg": "does not match …", "type": "pattern"}]}
```

`loc` is the path to the value: `["body", "items", 0, "name"]`.

### Unknown keys

Pass through untouched. Being strict would break the common case of a client
sending a field the server has not learned about yet.

---

## Middleware

```goost
fn myMiddleware(req, next) {
    // before
    let resp = next(req)
    // after
    return resp
}
```

The first registered is the outermost. `app.use` applies to everything,
including 404s. `router.use` applies to that router's routes, folded in when it
is mounted. A route's `middleware: [...]` applies to that route.

### Built in

```goost
import { logger, cors, requestId, securityHeaders, basicAuth, rateLimit, timing, enforceHttps } from "fastgoost/middleware"
```

| | Options |
|---|---|
| `logger()` | `colour`, `skip` (path prefixes) |
| `requestId()` | `header`, `trust` |
| `cors()` | `origins`, `methods`, `headers`, `expose`, `credentials`, `maxAge` |
| `securityHeaders()` | `hsts`, `csp`, `frameOptions`, `referrerPolicy` |
| `basicAuth()` | `users`, `check`, `realm` |
| `rateLimit()` | `limit`, `windowMs`, `key` |
| `timing()` | `label` |
| `enforceHttps()` | `host`, `status` |

`cors({origins: ["*"], credentials: true})` throws at startup: browsers reject
that combination, and a server that sends it looks fine until a real request
fails.

`basicAuth` compares passwords in constant time. `rateLimit` keeps counters in
this process, so behind several instances each gets its own budget.

---

## Dependencies

```goost
app.get("/me", {handler: showMe, deps: {user: currentUser, db: openDatabase}})

fn showMe(req) { return req.deps.user }
```

Each is `fn(req) → value`, run before the handler, in sorted name order. One
that throws stops the request — which is what makes an authentication
dependency also the authorisation check. The handler cannot run without one,
so it does not need an `if` to prove it did.

---

## Routers

```goost
let users = newRouter("/users", {tags: ["users"]})
users.use(requireAuth)
users.get("/{id:int}", showUser)

app.include(users)
app.mount("/api/v1", users)
```

A router has every method an application has: it is the same class, because
Langoost's `extends` does not survive a module boundary. Calling `run()` on one
works and serves just its routes.

---

## Static files

```goost
app.static("/assets", "./public")
app.static("/", {dir: "./dist", index: "index.html", cacheControl: "max-age=3600", fallback: "index.html"})
```

`fallback` is what a single-page app wants: anything that does not name a real
file serves the shell, and the router in the browser takes over.

Paths are normalised — `.` and `..` resolved, repeated slashes collapsed —
before they touch the filesystem, and `%2F` stays inside its segment, so a
request cannot climb out of the root.

---

## Streaming

### Server-Sent Events

```goost
app.sse("/events", fn(stream) { … })
```

| | |
|---|---|
| `stream.send(event, data)` | A named event |
| `stream.sendJson(event, data)` | The same, JSON-encoded |
| `stream.sendId(event, data, id)` | With an id, for `Last-Event-ID` resumption |
| `stream.data(payload)` | Unnamed — `EventSource.onmessage` |
| `stream.comment(note)` | A no-op line that keeps proxies from reaping the connection |
| `stream.retry(ms)` | How long the client waits before reconnecting |
| `stream.lastEventId()` | What the client sent when reconnecting |
| `stream.alive` | False once a write has failed |
| `stream.req` | The request |

The connection closes when the handler returns.

### WebSockets

```goost
app.ws("/ws", fn(socket) { … })
```

| | |
|---|---|
| `socket.recv()` | One message, or `void` when the peer closes |
| `socket.send(text)` `socket.sendJson(data)` `socket.sendBinary(data)` | |
| `socket.close()` | |
| `socket.req` | The request that was upgraded |

Handshake, masking, continuation frames, ping/pong and close are handled.
Extensions are not negotiated, so no frame arrives compressed. A request to a
`ws` route that is not an upgrade gets `426`.

---

## OpenAPI

`app.openapi()` returns the document; `/openapi.json` serves it. Path
parameters, declared query and header parameters, the request body schema,
tags, summaries and descriptions all come from the route declarations, and a
`422` is documented wherever validation is configured.

`/docs` (Swagger UI) and `/redoc` load their assets from a CDN, so the page
needs the internet even when the API is local. The spec itself is always served
by your own process.

---

## Testing

```goost
import { request } from "fastgoost/testing"

let resp = request(app, "POST", "/items", {json: {title: "milk"}})
```

Options: `query` (object), `json` (encoded for you), `body` (verbatim), `form`,
`headers`, `cookies`, `remoteAddr`.

The response has `status`, `headers`, `body`, and methods `json()`,
`header(name)`, `cookie(name)`, `ok()`.

Name test functions `test_*` in files named `*_test.goost` and run
`langoost test tests`.

---

## Running

```goost
app.run(8080)
app.run({port: 8080, host: "127.0.0.1"})
```

| Option | Default | |
|---|---|---|
| `host` | `"0.0.0.0"` | |
| `port` | `8000` | |
| `idleTimeoutMs` | `30000` | How long a kept-alive socket may sit silent |
| `headerTimeoutMs` | `10000` | Deadline for the request line and headers |
| `maxBodyBytes` | `8388608` | Larger bodies get `413` |
| `maxHeaders` | `100` | More get `431` |
| `keepAlive` | `true` | |
| `maxConnections` | `512` | In-flight connections; further ones wait rather than being refused. `0` disables the cap |
| `serverHeader` | `"fastgoost"` | `""` to omit |
| `banner` | `true` | |

The banner is colour and box drawing on a terminal, plain ASCII into a pipe.

### What the server speaks

HTTP/1.1 and HTTP/1.0, keep-alive without pipelining, `Content-Length` and
chunked request bodies, `Expect: 100-continue`, `HEAD`. One thread per
connection. `Content-Length` on responses is always computed rather than
trusted from a handler — a declared length that disagrees with the bytes
written is how a kept-alive connection becomes a request-smuggling bug.

No TLS: put a reverse proxy in front. No HTTP/2: the `http2` stdlib module has
`serve` and `serveCleartext`.

---

## Deployment

```sh
langoost run app.goost                  # develop
langoost compile app.goost -o app       # one binary
langoost build .                         # build/ with a binary, run.sh and .env
```

Behind nginx or Caddy, terminating TLS and setting `X-Forwarded-Proto` and
`X-Forwarded-For`. Add `securityHeaders({hsts: true})` once TLS is actually in
front — a wrong HSTS header is remembered by browsers for a year.

Read secrets from the environment with `os.getenv`, never from source.

### The one failure mode to know about

A **non-catchable** Langoost error inside a handler — an out-of-range index, a
division by zero — stops that connection's thread without unwinding. The socket
is never closed, so that one client waits until it gives up. Every other
connection, and the process, carry on. `try` cannot catch these by design, so
bounds-check before indexing and check a divisor before dividing:

```goost
if index >= 0 && index < len(items) { use(items[index]) }
if divisor != 0 { rate = total / divisor }
```
