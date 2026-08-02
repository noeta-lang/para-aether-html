# para/aether_html

**The per-frame onion for [para/html](https://github.com/noeta-lang/para-html) LiveView pages, in [para/aether](https://github.com/noeta-lang/para-aether)'s vocabulary.** Authorization, rate limiting, and tracing on every websocket frame — not just on the connect.

## The problem it exists for

A LiveView page keeps a table of generated handler ids and invokes one whenever a frame arrives naming it. The table is built from a single `render_page()` when the socket opens and lives for the whole session, so **every id ever registered stays reachable** — on a hidden element, on a `disabled` one, or under a permission the user has since lost. An app that gates a button by not rendering it has gated nothing: the client picks the id.

The natural place to check is aether's middleware onion, whose documented purpose already includes answering without dispatching. It cannot reach:

```
GET /ws ──▶ [ onion: mw₁ → mw₂ → serve_request ] ──▶ Response(hijack)   ← runs ONCE
                                                          │
                                                          └─▶ session loop
                                                                frame … frame … frame …   ← hours
                                                                (onion has already unwound)
```

`server.websocket(handler)` is a connection-hijack response: the entire session runs *inside* one `Request → Response` cycle. `Middleware.handle(req, next): Response` is per request, so it fires at the upgrade and never again. Authorizing the upgrade and nothing after is not authorization.

This package layers aether's onion shape one level down — over para/html's `Wake`, a client frame or an idle tick — so a layer sees every unit of client-originated work.

## What it provides

One pure-Noeta module, `para.aether_html`:

| symbol | kind | purpose |
| --- | --- | --- |
| `FrameMiddleware` | trait | one layer of the frame onion — `handle(f, next)`, the same shape as aether's `Middleware` |
| `FrameNext` | struct | the rest of the pipeline as a value; `next.run(f)` continues inward, not calling it refuses |
| `Frame` | struct | one wake in flight: para/html's `Wake` plus the per-frame context layers share |
| `onion(layers)` | fn | compose a stack into the interceptor `para.html.handle_all` takes |
| `serve(req, title, page, layers)` | fn | the drop-in replacement for `para.html.handle`, with the onion around every wake |
| `serve_all(…)` | fn | `serve` with every knob para/html exposes — tick cadence, tick callback, stylesheet |
| `Authorize` | class, `impl FrameMiddleware` | resolve the session per frame and put it to the app's policy |
| `RateLimit` | class, `impl FrameMiddleware` | cap one named action to a budget per window |
| `Trace` | class, `impl FrameMiddleware` | report each frame and what it cost, to a sink of your choosing |

## Installation

The onion lives in its own companion package, not in para/html and not in para/aether, on purpose:

- **Not in para/html** — it is std-only and standalone-servable, and a dependency on the web framework would cost both.
- **Not in aether-core** — that would make the LiveView engine, its `@html` tier handler, and its keyed reconciler mandatory for *every* aether app, including the REST-only ones.

The same rule `para/aether_db` states for the session store, applied to the other pairing. So an app adds a third entry to the `para` scope it already lists the other two under:

```toml
[dependencies]
para = [
    { version = "^0.2", package = "para/aether" },
    { version = "^0.3", package = "para/html" },
    { version = "^0.1", package = "para/aether_html" },   # <- add this
]
```

Pure Noeta on both sides — no `[trust]` entry needed.

## Usage

Swap `handle` for `serve` and hand it a stack:

```noeta ignore
use para.aether_html.{serve, Authorize, RateLimit, Trace}
use para.html.{render, Html, on_click}
use para.aether.CookieSessions
use std.http.{Request, Response}
use std.log

sessions = CookieSessions.new(keys, 3600, true)

fn fetch(req: Request): Response {
    return serve(req, "Todos", page, [
        Trace.new(fn(m: string) => log.info(m)),
        Authorize.new(sessions, fn(s, f) => s.get("user") != none),
        RateLimit.new("e3", 5, 60000),          // the expensive action, 5/minute
    ])
}
```

The first layer registered is the outermost — it sees the frame first and finishes last, exactly as aether's first middleware sees the request first.

## What each layer is for, and what it is not

**`Authorize` is a coarse gate, by design.** It sees the handler id, but ids are generated (`e0`, `c1-e0`, `<key>-e0`) and carry no meaning — nothing in `c1-e0` says *which* todo. So it can answer "may this user act on this page at all?" and cannot answer "may this user delete *this* row?".

That second question belongs at the call site, on the binding, where the row is in scope:

```noeta ignore
<button ${on_click(fn() => todos.remove(t.id)).only_if(fn() => user.can_edit(t))}>delete</button>
```

para/html's `.only_if` is re-checked at event time for the same reason this package exists. The two are complementary: the onion carries cross-cutting, semantics-free concerns, the binding guard carries per-action policy.

**Per-frame identity resolution buys revocation — with the right store.** `Authorize` re-reads the session on every frame rather than capturing it at upgrade. With a *stored* backend ([`para/aether_db`](https://github.com/noeta-lang/para-aether-db)'s `DbSessions`, whose `open` hits the row) deleting the row lands mid-session: the next frame is refused and "log out everywhere" is real. With the stateless `CookieSessions` the whole session rides in the cookie the upgrade carried, and that cookie cannot change while the socket is open — so re-reading returns the same value forever and revocation still waits for the reconnect. The layer is correct either way; which behavior you get is a property of the store you wired.

**`Trace` wraps the work rather than preceding it.** That is why the seam hands over the rest of the wake as a thunk: what it measures covers the handler, the keyed reconcile, *and* the diff push — the real cost of a frame, not the cost of deciding to run it.

## What this package does *not* carry

The **unconditional limits** — payload size and frame rate — live in para/html's own session loop, apply to every page, and cannot be layered away. That placement is deliberate. Shipped as layers here, they would exist only for apps that adopted this package, leaving the plain one-file LiveView page — the simplest and most common way these get written — as the one with no protections at all. A limit no app should have to ask for belongs where every app already is.

A layer only ever sees wakes that already cleared them.

## Testing your layers

The onion is drivable with no browser and no socket. `para.html.wake` builds a wake by hand and `onion(layers)` returns exactly the interceptor the server path uses, so a test exercises the production path with a synthetic frame on top:

```noeta ignore
ran = signal(false)
intercept = onion([Authorize.new(sessions, deny_all)])
intercept(html.wake("e0", "", false), fn() => ran.set(true))
assert(!ran.get())     // the layer refused; the work never happened
```

[`examples/frame-onion/`](examples/frame-onion) does this across nesting order, refusal at both positions, per-frame context isolation, the rate budget, and tick exemption. A security layer whose only exercise is a live websocket is a security layer nobody tests.

## See also

- [para/html](https://github.com/noeta-lang/para-html) — the LiveView engine, the `Wake` seam, and the binding guard.
- [para/aether](https://github.com/noeta-lang/para-aether) — the web framework, its HTTP onion, and the `SessionStore` seam.
- [para/aether_db](https://github.com/noeta-lang/para-aether-db) — stored, revocable sessions; the store that makes per-frame identity resolution bite.
