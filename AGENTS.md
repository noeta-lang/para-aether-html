# AGENTS.md

Guidance for coding agents working in this repo — the standalone repo of the **para/aether_html** Noeta package (the per-frame onion for para/html LiveView pages; pure Noeta, composing para/aether + para/html). Toolchain issues (the language, the `noeta` binary, `std.*`) belong in the monorepo at github.com/noeta-lang/noeta, not here.

## Repo layout

- `aether_html.noe` — the whole surface: the `Frame`/`FrameMiddleware`/`FrameNext` onion, `onion`/`serve`, the `LiveMount` aether middleware, and the three shipped layers (`Authorize`, `RateLimit`, `Trace`).
- `examples/frame-onion/`, `examples/mounted-page/` — standalone packages depending on this repo via `{ path = "../.." }` alongside the sibling deps.
- `.github/workflows/` — `ci.yml`, the tag-triggered publish `release.yml`, `toolchain-pin.yml`, `docs-backfill.yml`.

## The three floors, and why each is where it is

`noeta.toml` pins `toolchain = ">=0.6"`, `para/html ^0.6`, `para/aether ^0.4`. None of them is arbitrary and none may be lowered:

- **noeta 0.6** is where a label binds on a *method* call. Before it, `LiveMount.new("/", title, page, on_tick: refresh)` bound `refresh` positionally and failed to check. (The manifest comment still says 0.5.2 — that release never existed; the line went 0.5.1 straight to 0.6.0, which is why the floor is `>=0.6`.)
- **para/html 0.6** is where a mount at the root claims the page's three URLs instead of every path, and where the served document tells the shim which socket to open. Against 0.5, a root mount silently swallows the app's other routes and `base: "/"` builds `//ws`.
- **para/aether 0.4** is the noeta-0.5 release in which a method implementing a trait must be `pub` — a rule that crosses the package boundary, since this package reaches aether through `Middleware` and `SessionStore`.

## Build & test

Everything here needs the `noeta` binary **and** a Rust toolchain on `PATH`: this package is pure Noeta, but para/html ships `dev-native` tier-body formatters that the composed toolchain cargo-builds. The first `noeta check`/`fmt`/`test` in a checkout therefore spends minutes on "composing the toolchain with native dependencies"; later runs reuse the cached binary. (Those formatters are dev-only and trust-free — they contribute nothing at run time and need no `[trust]` entry from a consumer.)

- The suite is `noeta check <file>.noe` + `noeta test <file>.noe`, run in each `examples/*/` directory **and at the package root** — `aether_html.noe` carries the `LiveMount` routing tests. CI runs only the examples, so run the root pair yourself before merging.
- If root resolution fails with `keyless verification failed: identity mismatch`, the committed root `noeta.lock` predates a dependency floor. Delete it, let it regenerate, and commit the result.
- `frame-onion` is the browserless exercise of the onion — runnable with `noeta run`, and the place any new layer earns its test. It drives `onion(layers)`, the same interceptor the server path uses, over a hand-built `para.html.wake`. **A layer whose only exercise is a live websocket is a layer nobody tests**; do not add one without a case here.
- `mounted-page` is the routing exercise: a root mount and a prefixed one in a single app, beside a controller whose `/health` and `/api/count` have to survive both. It is also the file to read when asking what an app's entry point should look like — `fn fetch` hands everything to `app.route` and names no LiveView URL.

## Conventions

- The **package root** `noeta.lock` is committed; `examples/*/noeta.lock` are **not** — they are gitignored and regenerate on every run.
- **Do not run `noeta fmt` across the repo.** The formatter breaks `@test fn name()` onto two lines; every test here is written on one, CI does not gate formatting, and a blanket reformat buries your change in unrelated churn. Match the surrounding style instead.
- **American English** throughout — code, comments, and docs (`behavior`, not `behaviour`). Markdown never hard-wraps: one line per paragraph.
- **Conventional commits** for all commit titles. Commit each green slice as it completes, but **never `git push` without explicit authorization**. Never move a published `v*` tag — a release is a new tag.
- Work on a branch in a worktree under `.claude/worktrees/<name>` (gitignored), and merge from the main checkout — `git merge` from inside the branch's own worktree is a silent no-op.
- Implement in full — no stubs or TODOs; new functionality lands with tests. Keep `README.md` and this file current in the same commit as the change.

## Design invariants

These are the load-bearing decisions; changing one is an architecture change, not a refactor.

- **Neither dependency may depend on the other.** para/html stays std-only and standalone-servable; aether stays free of the LiveView engine. That is the entire reason this package exists as a third repo rather than as a feature of either — the same rule `para/aether_db` states for the session store.
- **The unconditional limits stay in para/html**, not here. Payload size and frame rate apply to every page including the standalone one-file one; re-homing them as layers would leave the most common way a LiveView app gets written with no protections. A layer only sees wakes that already cleared them.
- **`FrameMiddleware.handle` returns `void`, not `Response`.** A frame produces effects; the reply is whatever the diff pushes. A layer with something to say writes a signal and lets the reactive graph carry it — one path to the browser, not two.
- **The onion is for semantics-free concerns.** Handler ids are generated and carry no meaning, so per-action policy ("may this user delete *this* row?") belongs on the binding via para/html's `.only_if`, not here. Do not grow this package a row-aware authorization API; it cannot have the information.
- **`LiveMount.new` is the only door to a mount**, and its optional knobs (`layers`, `every_ms`, `on_tick`, `styles`) are named arguments. v0.5.0 deleted the free `mount(…)` wrapper, which existed only because a label was ignored on a method call before noeta 0.6. Do not reintroduce a second constructor to work around a call-site wart — raise the floor instead.
- **Which paths a mount owns is `para.html.serves`**, never a predicate of this package's own, so a mount and the page it mounts cannot disagree about where the page lives.

## CI

`ci.yml` checks and tests every example with a pinned released `noeta` (the org-level `NOETA_VERSION` Actions variable, plus the pinned Rust toolchain for para/html's dev-native half). `release.yml` reuses `ci.yml` as its gate, then publishes the tag to the hosted registry (`noeta publish`, keyless Sigstore provenance via GitHub OIDC) and cuts a GitHub release from the conventional-commit log. `toolchain-pin.yml` answers a `noeta` release by rebuilding against it and opening a PR; by design it never touches `toolchain =` or the package version — those are human calls.
