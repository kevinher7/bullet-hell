# Architecture Discussion — Bullet Hell

A working document for making (and recording) architecture decisions. This is
**not** a spec to implement — it's a set of open questions, the options, and the
trade-offs, so we can grill each one before you commit code to it.

Format per decision: **the question → options → trade-offs → my lean → what it
locks in vs. defers**. Fill in a **Decision:** line when you settle one. Nothing
here is binding until you write that line.

The two hard goals everything is measured against:
1. A playable game that *feels* fluid.
2. Render + simulate **10,000+ bullets at 60 FPS**.

And two constraints you've already committed to (from `CLAUDE.md`):
- A JSON-input testing suite (also lets agents play the game).
- Profiling performance with Instruments.

---

## 1. Library: SDL2 vs raylib vs SDL3

**Question:** What sits between your code and the OS/GPU?

**Options:**
- **SDL2** — windowing, input, audio, and a 2D GPU renderer (`SDL_Renderer`).
  Industry-standard, huge surface area, endless references.
- **raylib** — friendlier, batteries-included 2D/3D, less boilerplate to first
  pixel.
- **SDL3** — newer, cleaner GPU API, but fewer tutorials and moving target.

**Trade-offs:** SDL2 teaches more of the "real" idioms you'll see in jobs and has
the deepest reference pool; raylib gets you playing faster but hides more. The
choice barely touches `sim/` (that's the point of the boundary) — it only affects
`platform/` and `render/`, so it's *reversible* if you keep the seam clean.

**My lean:** SDL2. The career goal wants the standard toolchain, and the extra
boilerplate is itself the lesson. But see Decision 3 — for 10k bullets, *how* you
draw through SDL2 matters more than SDL2 vs raylib.

**Locks in:** the flavor of `platform/` + `render/`. **Defers:** nothing in `sim/`.

> **Decision:** _(unset)_

---

## 2. Fixed timestep — the shape of the loop

**Question:** How does simulation time relate to wall-clock and render time?

**Options:**
- **Fixed-timestep sim, decoupled render** (accumulator pattern): sim advances in
  discrete ticks (e.g. 1/60 s); render interpolates between the last two states.
- **Variable timestep**: sim advances by real frame delta-time.

**Trade-offs:** Variable timestep is simpler for about a week and then poisons
everything you care about — it's **non-deterministic**, so the JSON harness,
replays, and regression tests all become impossible or flaky. Fixed timestep is
the price of admission for the test suite you already decided to build.

**My lean:** Fixed timestep, non-negotiable given your constraints. The only real
sub-question is whether render interpolates (smooth at high refresh rates) or just
draws the latest tick (simpler, can look juddery above 60 Hz). Start without
interpolation; add it if it feels rough.

**Locks in:** determinism (which everything downstream depends on). **Defers:**
render interpolation.

> **Decision:** _(unset)_

---

## 3. Bullet storage & drawing — the 10k-at-60fps decision

**Question:** How are bullets stored in memory, and how do they reach the screen?
This is *the* performance decision of the project.

**Storage options:**
- **`std::vector<Bullet>` + swap-and-pop** on death — contiguous, cache-friendly,
  simple. Order isn't stable (fine for bullets).
- **Object pool / free-list** with a fixed-capacity array — no per-frame
  allocation at all, stable indices for handles.
- **Struct-of-Arrays (SoA)** — separate `xs[]`, `ys[]`, `vxs[]`… arrays instead of
  an array of `Bullet` structs. Maximizes cache utilization when you sweep one
  field across all bullets.

**Drawing options:**
- One `SDL_RenderCopy` per bullet — *this is the trap*; ~10k draw calls will not
  hit 60 FPS.
- **Batched draw**: one texture (a sprite atlas), geometry built once per frame,
  submitted in a handful of calls (`SDL_RenderGeometry`).

**Trade-offs:** The *whole point* of picking this game was to feel why
data-oriented design exists. `vector<Bullet>` is the honest starting point;
measure with Instruments; move toward SoA + batching **only when the profiler says
so**, not preemptively. Premature SoA is as much a mistake as premature
abstraction.

**My lean:** Start `std::vector<Bullet>` + swap-and-pop, single-texture batched
draw via `SDL_RenderGeometry`. Treat SoA as a *documented experiment* you run once
you have a profiler baseline — that measure-then-refactor cycle is the most
career-relevant thing in the repo.

**Locks in:** almost nothing early (that's deliberate). **Defers:** SoA, spatial
structures — driven by data, not guesses.

> **Decision:** _(unset)_

---

## 4. Collision — broad phase

**Question:** How do you avoid checking every bullet against every target?

**Options:**
- **Naïve O(n·m)** — bullets × player/enemies. With one player hitbox and 10k
  bullets this is only ~10k checks/tick and might just be *fine*.
- **Uniform grid** (spatial hash) — bucket entities by cell, check neighbors only.
- **Quadtree** — hierarchical; better for wildly varying densities.

**Trade-offs:** Bullet-hell collision is asymmetric — many bullets, *few* targets.
That often means the naïve version is genuinely adequate and a grid is
over-engineering. Don't build the grid until the profiler shows collision is hot.

**My lean:** Start naïve, one player hitbox vs. all bullets. Add a uniform grid
only if profiling demands it (and it's a clean, self-contained addition when it
does).

**Locks in:** nothing. **Defers:** broad-phase structure until measured.

> **Decision:** _(unset)_

---

## 5. The `sim` / `render` / `platform` boundary

**Question:** What is allowed to include what?

**Proposed rule:**
- `sim/` includes only `sim/` + the standard library. **No SDL, no rendering, no
  wall-clock, no `rand()`.** The compiler enforces determinism for you.
- `render/` reads `SimState`, never mutates it.
- `platform/` is the only place that knows SDL exists.
- `InputFrame` is the hinge type — produced by both real input (`platform/`) and
  the JSON harness, consumed by `sim::update`, which can't tell them apart.

**Trade-offs:** This costs a little ceremony now (passing state around instead of
reaching for globals) and pays for the entire test/replay/determinism story. It's
the same boundary as the "carry code forward" plan — the parts that survive here
are what become your informal engine by project 3.

**My lean:** Adopt as written. This is the one piece of up-front architecture I'd
*not* defer, because retrofitting it is miserable.

**Locks in:** testability, replays, headless mode. **Defers:** nothing you'd want
to defer.

> **Decision:** _(unset)_

---

## 6. RNG & determinism discipline

**Question:** Where does randomness live?

**Proposal:** A single seeded PRNG owned by `SimState` (e.g. a small xorshift or
`std::mt19937` seeded from the input script). The seed is part of the JSON input.
No other randomness anywhere in `sim/`.

**Known determinism traps to watch (write these on the wall):**
- Reading frame delta-time instead of the fixed tick.
- Iterating `std::unordered_map` / `unordered_set` (order isn't stable).
- Uninitialized memory feeding logic.
- Any `rand()` / clock read inside `sim/`.
- Floating point is *fine* on one machine+compiler; only cross-platform lockstep
  makes it painful, which you don't need.

**My lean:** Adopt. Cheap now, load-bearing forever.

> **Decision:** _(unset)_

---

## 7. JSON test harness — shape of the contract

**Question:** What exactly does a test script look like, and what comes back out?

**Sketch (to grill, not to freeze):**
```json
{
  "seed": 42,
  "commands": [
    { "tick": 0,   "hold": ["right"], "for": 120 },
    { "tick": 130, "press": ["jump"] }
  ]
}
```
Run: `./game --headless --script foo.json --run-ticks 400 --dump-state`
→ emits `SimState` as JSON (player pos, health, live bullet count, …).

**Open sub-questions:**
- **Scripted vs. interactive?** Pre-written script is enough for regression tests.
  A stdin request/response mode (step N ticks → emit state → read next command)
  lets an agent play *reactively*. Interactive is cheap to add later; don't build
  it until you want it.
- **What goes in the state dump?** Everything is slow and noisy; too little misses
  regressions. Lean: dump a small, stable "assertable" summary + a hash of the
  full state for golden-replay tests.
- **Library:** `nlohmann/json` — ergonomic, header-only, speed irrelevant for a
  test harness. (Pull via CMake FetchContent, not a system package.)

**My lean:** Scripted + state-dump in project 1; a hash for golden replays; leave
interactive stdin mode as a documented "later."

> **Decision:** _(unset)_

---

## 8. Build & tooling

**Question:** How is the project built and dependencies pulled?

**Proposal:**
- **CMake + Ninja**, `compile_commands.json` exported for clangd.
- **`CMakePresets.json`** with `debug`, `dev` (`-fsanitize=address,undefined`),
  `release`. Run under `dev` by default so sanitizers catch bugs at the source.
- **Dependencies via CMake `FetchContent`** (SDL, nlohmann/json) so the repo is
  self-contained and reproducible — *not* host-global packages.
- **Profiling:** Instruments (Time Profiler) from project 1 — sampling, zero code
  changes, compile with `-g`. Tracy deferred to a later project.

**My lean:** Adopt as written; it matches the toolchain plan.

> **Decision:** _(unset)_

---

## Decisions log

Record settled decisions here with a date and one-line rationale, so future-you
knows *why*, not just *what*.

| # | Decision | Date | Rationale |
|---|----------|------|-----------|
|   |          |      |           |

---

## Open questions parking lot

Things raised but not yet worth a full section:
- Sprite atlas layout / asset pipeline (defer until batching exists).
- Audio approach (SDL_mixer vs. raw SDL audio) — low priority for feel-testing.
- How enemy bullet *patterns* are authored (data-driven tables vs. code) — a real
  decision, but later than the engine seams.
