# AGENTS.md

Guidance for coding agents working in this repo — the standalone repo of the **para/aether_html** Noeta package (the per-frame onion for para/html LiveView pages; pure Noeta, composing para/aether + para/html). Toolchain issues (the language, the `noeta` binary, `std.*`) belong in the monorepo at github.com/noeta-lang/noeta, not here.

## Repo layout

- `noeta.toml` — the package manifest (`name = "para/aether_html"`). Pure Noeta (no `native` key), but it **depends on two sibling packages** — para/aether and para/html — bound under one `para` scope key as an array (multi-package-per-scope resolution). para/html is pinned to `^0.6`: the `Wake` seam landed in 0.3, 0.4 collapsed the entry points into one `handle` with named arguments and added the `base:` mounting `LiveMount` is built on, and 0.6 is where a mount at the root claims the page's three URLs instead of every path and the served document tells the shim which socket to open. `LiveMount` is only correct against 0.6 — against 0.5 a root mount swallows the app's other routes.
- `aether_html.noe` — the whole surface: the `Frame`/`FrameMiddleware`/`FrameNext` onion, `onion`/`serve`, the `LiveMount` aether middleware and its `mount(…)` door, and the three shipped layers (`Authorize`, `RateLimit`, `Trace`).
- `examples/*/` — each a standalone package depending on this repo via `{ path = "../.." }` alongside the sibling deps.
- `.github/workflows/` — CI (`ci.yml`) and the tag-triggered registry publish (`release.yml`).

## Build & test

No cargo in *this* repo, but the examples pull para/html, whose `dev-native` tier-body formatters the composed toolchain cargo-builds — so running them needs both the `noeta` binary and a Rust toolchain on `PATH`. (Those formatters are dev-only and trust-free; they contribute nothing at run time and need no `[trust]` entry from a consumer.)

- `noeta check <file>.noe` / `noeta test <file>.noe` in each `examples/*` directory is the test suite.
- `mounted-page` is the routing exercise: a root mount and a prefixed one in a single app, beside a controller whose `/health` and `/api/count` have to survive both. It is also the file to read when asking what an app's entry point should look like — `fn fetch` hands everything to `app.route` and names no LiveView URL.
- `frame-onion` is the browserless exercise of the onion — runnable with `noeta run`, and the place any new layer earns its test. It drives `onion(layers)`, the same interceptor the server path uses, over a hand-built `para.html.wake`. **A layer whose only exercise is a live websocket is a layer nobody tests**; do not add one without a case here.

## Conventions

- The **package root** `noeta.lock` is committed; `examples/*/noeta.lock` are **not** — they are gitignored and regenerate on every run. (The sibling repos' AGENTS.md files say example locks are committed; their `.gitignore` and git history disagree, and this is the rule.)
- Markdown never hard-wraps lines.
- **American English** throughout — code, comments, and docs (`behavior`, not `behaviour`).
- **Conventional commits** for all commit titles. Commit each green slice as it completes, but **never `git push` without explicit authorization**. Never move a published `v*` tag — a release is a new tag.
- Implement in full — no stubs or TODOs; new functionality lands with tests.
- Keep `README.md` and this file up to date when layout or behavior changes.

## Design invariants

These are the load-bearing decisions; changing one is an architecture change, not a refactor.

- **Neither dependency may depend on the other.** para/html stays std-only and standalone-servable; aether stays free of the LiveView engine. That is the entire reason this package exists as a third repo rather than as a feature of either — the same rule `para/aether_db` states for the session store.
- **The unconditional limits stay in para/html**, not here. Payload size and frame rate apply to every page including the standalone one-file one; re-homing them as layers would leave the most common way a LiveView app gets written with no protections. A layer only sees wakes that already cleared them.
- **`FrameMiddleware.handle` returns `void`, not `Response`.** A frame produces effects; the reply is whatever the diff pushes. A layer with something to say writes a signal and lets the reactive graph carry it — one path to the browser, not two.
- **The onion is for semantics-free concerns.** Handler ids are generated and carry no meaning, so per-action policy ("may this user delete *this* row?") belongs on the binding via para/html's `.only_if`, not here. Do not grow this package a row-aware authorization API; it cannot have the information.

## CI

`ci.yml` checks and tests every example with a pinned released `noeta` (plus the pinned Rust toolchain, for para/html's dev-native half); `release.yml` publishes the tag to the hosted registry (`noeta publish`, keyless Sigstore provenance via GitHub OIDC).
