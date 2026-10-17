# Orientation for an LLM working on Aebiten

The non-obvious parts of this repo. `README.md` covers how to build and
write a game; `AE-MIGRATION.md` holds the per-subsystem status table.
This file is what those two don't say. Code wins where they disagree.

## What this is

**Aebiten** — a 2D game engine in **Aether**, not Go. The checkout is `aebiten/`, with `origin` at
`aether-lang-dev/aebiten`; nothing on `main` is Go. The API lineage is Ebitengine's; the
implementation is a from-scratch Aether port whose bottom half
(GLFW, GPU graphicsdriver, internal/ui, mobile/js — ~80k lines) was
**never ported**: `aether-ui`'s canvas replaces it.

Everything renders in software. An image is a CPU RGBA8888 buffer,
`DrawImage` is an inverse-affine per-pixel blit written in Aether, and the
screen is one such buffer pushed to an aether-ui canvas on a `ui.timer(16)`
tick. There is no GPU path and no shader story (Kage is deferred).

The Go original lives on branch **`legacy_golang`** (tracks upstream). It is
the oracle: settle behaviour questions by reading or running it. Do not
port line-for-line — the shapes differ deliberately (see README).

`origin` is `aether-lang-dev/aebiten`; `main` tracks `origin/main`.
Commit and push directly to `main`.

## Layout

    aebiten/core.ae     GeoM, ColorScale, ColorM, blend table, Image,
                        DrawImage/DrawTriangles + the pixel loops.
                        Pure — no `ui` import, headless-testable.
    aebiten/module.ae   the engine (`import aebiten`): game loop, input
                        table, window/canvas present. Imports `ui`.
    aebiten/{png,util,vector,audio,text}.ae
    examples/<name>/    one aeb node each (.build.ae + <name>.ae)
    tests/              headless; run from the REPO ROOT
    build_support/aebitenui   the backend-link builder
    lib/vg -> ../../aether-ui/vg   symlink so headless `ae run` finds vg.font

## Environment facts that break the build if wrong

- **Needs a sibling `../aether-ui`** (or `$AETHER_UI_ROOT`) at commit
  `ad960a8`+ — two changes this port drove upstream live there:
  `canvas_on_key_release` and `canvas_draw_image_scaled_ptr`. Both are
  load-bearing (input release semantics; `set_scale`).
- **Toolchain must be post-0.578** (aether#1733: `std.mem` scalar accessors
  codegen-inlined). Earlier toolchains make the pixel loops ~200x slower.
  Verified green on **0.823.0** (2026-10-17), with aeb built from `main`
  (db99a6f): v0.326's `bldr` predates 0.818's `@keep ptr` rule. The old stale-`~/.aether/bin`
  PATH shadow is gone; `/usr/local/bin` is the installed toolchain, and
  `../aether/build` holds whatever that sibling last compiled — the two
  drift, so check `ae --version` before trusting a perf number.
- **`aether.toml` pins `-O2` and that is not cosmetic.** The inlined
  accessors only pay off under the optimizer; the C substrate that used to
  carry these loops was deleted (commit f47b31e26) on the strength of the
  `-O2` numbers. If you benchmark without it you will "rediscover" a need
  for C that isn't there.
- Sibling checkouts (`../aether-ui`, `../aether`) may hold unrelated
  uncommitted work. Never run git write ops in them; hand off to the
  session that owns them.

## Commands

    ae run tests/test_core.ae     # tests, from the repo root, no args needed
    aeb all.ae                    # build every example (fan-out over nodes)
    aeb examples/snake            # one
    ./ci.sh                       # the gate: tests + builds + snake smoke

`ci.sh` phase 2 needs a display or `xvfb-run`. Driver testing runs a binary
with `AETHER_UI_TEST_PORT=<port>` and drives AetherUIDriver routes
(`/canvas/1/key`, `/keyup`, `/click`, `/move`, `/screenshot`, `/shutdown`).

**Caching hides regressions.** `ae run` caches by source hash in
`~/.aether/cache`; `aeb` caches content-addressed in
`~/.local/share/aeb/repo`, which `touch` does NOT invalidate (it hashes
content, not mtimes) — a `[hit]` line in the telemetry is a *replay*, warnings
and all. After a toolchain bump, a green run over cache hits proves nothing:
`rm -rf ~/.aether/cache` for the tests, and for aeb make a real content change
(or accept that only `[miss]` rows were actually compiled).

## Traps

- **A `ptr` parameter that outlives the call is `@keep ptr`** (Aether 0.818,
  E0902). This covers a parameter stored in a field (`opts_set_colorm`'s `cm`,
  `image_sub`'s root), captured by a callback (`run`'s engine), or handed on to
  another `@keep` parameter (`debug_print_*`'s font, via `image_sub`). The
  check is per function body and conservative: a view freed before the
  return still counts.
- **A driver key press must span an engine tick.** Sleep ~20ms between
  `/key` and `/keyup` or the per-tick just-pressed snapshot never sees it.
  This is the single most common "the driver test is flaky" cause.
- **Inside `aebiten.*` / `core.*` trailing blocks, the receiver's DSL
  setters OUTRANK your file's functions.** A game defining its own
  `update()` or `draw()` will silently bind the builder setter instead.
  The ports use `step` / `render`. Same applies to `translate`, `scale`,
  `rotate`, `opacity`, `blend`, `linear`, `nearest`, `tint`, `skew`,
  `color_matrix` inside a `core.draw {}` block.
- **`int` is 32-bit.** Packing 0xRRGGBBAA into one `int` overflows — the
  engine keeps channels separate throughout. Use `long` if you must pack.
- **Module-name shadowing.** In `aebiten.audio`, `import std.audio` also
  binds `audio.`, and the *current* module wins — a wrapper named the same
  as the std function self-recurses into a segfault. Call the extern
  directly for same-named wrappers.
- **Parser is line-sensitive.** A boolean condition continued onto the next
  line with a leading `||` lexes as a closure (`|x|`); a leading `+` breaks
  too. Keep expressions on one line or split into accumulator statements.
- **`ptr` ↔ `string` has no cast.** Bridge via `std.bytes`:
  string→ptr is `bytes.new` + `bytes.copy_from_string` + `bytes.data`;
  ptr→string is write through `bytes.data`, then `bytes.set_length(b, n)`
  (required — direct writes don't update the live length) + `bytes.finish`.
- **Don't re-declare runtime externs with your own param types** (`ptr`
  where the header says `const char*`) — conflicting-types at the C compile.
  Use the std wrapper plus a bytes bridge.
- **Struct boxes are `heap.new(T)` / `heap.free(p)`, never raw malloc.**
  `heap.new`/`heap.free` are *compiler builtins* (no import; there is no
  `std/heap` module) that lower to `calloc(1, sizeof(T))` / `free`, so the box
  is layout-exact and zero-initialised. The engine used hand-sized
  `malloc(48) as *GeoM` / `calloc(1, 256) as *Engine` numbers until 2026-09-08;
  all seven boxes (`GeoM`, `Opts`, `Image` x2, `Path`, `Face`, `Engine` x2) are
  converted. Don't reintroduce the old shape — a literal count silently
  under-allocates when the struct grows a field and surfaces as heap corruption
  far from the edit. (`Engine` was 256 hand-counted vs 216 actual: 40 bytes of
  luck, not correctness.) 0.651.0 warns on `malloc(<literal>) as *T`.
- **Raw `malloc` is still right for two things**, and the compiler does not
  flag either: variable-size buffers (`calloc(w*h, 4)` for pixels,
  `malloc(cap*16)` for path points) and the untyped scalar scratch arrays in
  `core.ae` (`malloc(160)` for a 4x5 ColorM, `malloc(32)` per-pixel temps)
  that are addressed by `mem.set_float64` offset, not as structs. Only
  *struct boxes* take `heap.new`.

## Both API faces are live

The explicit-handle API (`game_new` / `on_update` / `on_draw` / `run`,
`opts_new` + `geom_*`) and the builder DSL (`aebiten.game {}`,
`core.draw {}`) both ship, and six of nine examples still use the handle
form. New engine surface should get **both**: a plain function, plus a
setter exported for the block. See `builder game(...)` at
`aebiten/module.ae:530` and `builder draw(...)` at `aebiten/core.ae:1412`.

## Known gaps (before proposing a workaround)

Kage shaders (deferred, no software-shader design), gamepad/touch/mobile/JS
(out of scope — canvas backend), vector AA, PNG interlace + 16-bit, JPEG
(none — `rotate` uses `ebiten.png` instead of gophers.jpg), window resize
(canvas is fixed logical size; `set_scale` is an integer-ish upscale of the
whole frame). Upstream additions postdating the fork (e.g. `ScreenSize`) are
adopted case by case, not automatically — a diff against the Go *tip* may
just be a change this port hasn't chosen to chase.

## Conventions

- Ported files carry the Apache 2.0 header plus "Portions Copyright 2013
  Hajime Hoshi and The Ebiten Authors"; `NOTICE.md` states provenance.
- Upstream names stay upstream: "Ebitengine", `hajimehoshi/ebiten`, the
  `ebiten.png` asset, `ebitenutil` in comments. The *engine* is `aebiten`.
- New example ⇒ its own `examples/<name>/.build.ae` node (aeb Shape A,
  `bldr.build {}`); `all.ae` picks it up by glob. Add the test to `ci.sh`
  phase 0 if it's a module test.
