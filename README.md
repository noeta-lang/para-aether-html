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
| `onion(layers)` | fn | compose a stack into the interceptor `para.html.handle`'s `intercept:` takes |
| `serve(req, title, page, layers, …)` | fn | the drop-in replacement for `para.html.handle`, with the onion around every wake |
| `LiveMount` | class, `impl Middleware` | mount a page into an aether `App` at a prefix, beside its other routes |
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
    { version = "^0.4", package = "para/aether" },
    { version = "^0.5", package = "para/html" },
    { version = "^0.3", package = "para/aether_html" },   # <- add this
]

# A tier is bound, never imported: this table is what makes `@html { … }` an expression in the page
# you hand to `serve` or `LiveMount`. Add `css = "para/html"` too if you write `@css { … }` sheets.
# The binding names the PACKAGE, not the `para` key — that key is a scope array covering three
# members and cannot say which one you meant.
[directives]
html = "para/html"
```

Pure Noeta on both sides — no `[trust]` entry needed.

The tier comes from that binding and from nothing else. `use para.html.{render, Html, on_click}` brings the *functions* into scope, as it always did, but since noeta 0.5 it no longer enables `@html` — a page whose manifest lacks the binding fails on the block, not on the import. Both lines are wanted: the import for the surface you call, the binding for the tier you write.

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

Writing your own layer is one `impl`, and its `handle` must be **`pub`** — a trait is an outward contract, and since noeta 0.5 a method that implements one has to say so (`E0015` names it if you forget):

```noeta ignore
class Audit {
    pub fn new(): Audit {
        return Audit {}
    }

    impl FrameMiddleware {
        pub fn handle(f: Frame, next: FrameNext): void {
            if !f.tick() {
                log.info("frame ${f.name()} from ${f.get("identity", "?")}")
            }
            next.run(f)          // not calling this is how a layer refuses
        }
    }
}
```

## Mounting into an aether app

`serve` is the standalone door — the page owns the origin. To put a page beside an app's other routes, mount it:

```noeta ignore
app = App.new()
app.register("Api", Api.new())
app.use_middleware(LiveMount.new("/todos", "Todos", page, [
    Authorize.new(sessions, fn(s, f) => s.get("user") != none),
]))
app.serve(8080)
```

The page is live at `/todos`, its socket at `/todos/ws`, its shim at `/todos/live.js`; everything else falls through to the rest of the app untouched.

**A middleware rather than a route**, for two reasons. aether's routes dispatch to reflected controller methods and a page is a closure. And the onion already has exactly the right shape — a layer that answers without calling `next` *is* a mount.

Which paths belong to the mount is `para.html.serves`, not a predicate of this package's own: one rule, read by para/html's routing and by this gate, so a mount and the page it mounts cannot disagree about where the page lives. Matching is segment-exact, so a `/to` mount does not claim `/todos`.

### Two nested onions

Mounting makes the actual model visible, and it is worth being explicit about which layer does what:

| | runs | for | belongs there |
| --- | --- | --- | --- |
| aether's HTTP onion | once, on the upgrade | the request that opens the socket | CORS, request logging, connect-time auth |
| this frame onion | every wake, for the socket's life | each client frame and idle tick | per-frame authorization, action budgets, frame tracing |

Neither can do the other's job. An HTTP layer cannot see a click, because by the time one arrives its `Response` has long since been returned. A frame layer cannot reject the connection, because it only exists once there is one.

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
