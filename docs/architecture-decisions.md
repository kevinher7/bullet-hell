# Architecture Decisions — Bullet Hell

The **settled** record. `architecture-discussion.md` is the working document where
options get argued; this is where the outcome lands once a decision is made.

Each entry has a **Status**:

- **Settled** — committed. Changing it is a real refactor; write down why in the log.
- **Provisional** — a lean we're building on, but nothing depends on it deeply yet.
  Cheap to revisit.
- **Deferred** — deliberately not decided. The trigger for deciding is stated.

The two hard goals everything is measured against:

1. A playable game that *feels* fluid.
2. Render + simulate **10,000+ bullets at 60 FPS**.

---

## 1. Library: SDL2

**Status:** Provisional

SDL2 for windowing, input, audio, and the 2D renderer. Chosen for the depth of the
reference pool and because the extra boilerplate is itself the lesson.

Only `platform/` and `render/` are allowed to know it exists, so the choice stays
reversible as long as the seam in Decision 5 holds.

---

## 2. Fixed timestep, decoupled render

**Status:** Settled

The sim advances in discrete ticks (1/60 s) via an accumulator. Render draws the
latest tick.

Non-negotiable: variable timestep is non-deterministic, and determinism is the
foundation the harness, replays, and every regression test sit on. This decision
is load-bearing for Decisions 6, 7, 11 and 12.

**Deferred:** render interpolation between ticks. Add it only if motion feels
juddery above 60 Hz.

---

## 3. Bullet storage — pool now, SoA only if measured

**Status:** Settled

Bullets live in a dedicated fixed-capacity pool inside `SimState`, separate from
the ECS (see Decision 9). Swap-and-pop on death. No per-frame allocation.

Drawing goes through a single-texture batched path (`SDL_RenderGeometry`), not one
`SDL_RenderCopy` per bullet — ~10k draw calls will not hit 60 FPS.

**Struct-of-Arrays is a documented experiment, not a starting point.** It happens
after there is an Instruments baseline (Milestone 4) and the profiler says the
bullet update loop is actually hot. That measure-then-refactor cycle is the point.

---

## 4. Collision broad phase — naïve until proven otherwise

**Status:** Deferred

Start with naïve checks: one player hitbox against all bullets. ~10k checks/tick,
which is plausibly just fine. Bullet-hell collision is asymmetric — many bullets,
very few targets — so the usual justification for a spatial structure is weaker
than it looks.

**Trigger to revisit:** Instruments shows collision in the top few frame costs. A
uniform grid is a clean, self-contained addition when that happens.

---

## 5. The `sim` / `render` / `platform` boundary

**Status:** Settled

- `sim/` includes only `sim/`, `ecs/`, and the standard library. **No SDL, no
  rendering, no wall-clock, no `rand()`.**
- `render/` reads sim state through `const&` and never mutates it.
- `platform/` is the only place that knows SDL exists.
- `InputFrame` is the hinge type. It is produced by real input (`platform/`) *and*
  by the JSON harness, and `sim::update` cannot tell them apart.

**This is enforced mechanically, not by discipline.** Each directory is its own
CMake target, and `bh_sim` does not link SDL — so `#include <SDL.h>` inside `sim/`
is a build error, not a code review note. See "Target rules" below.

The one part CMake cannot enforce — render not mutating sim — is bought with the
API shape: `render::draw(const SimState&, ...)`.

---

## 6. RNG & determinism discipline

**Status:** Settled

A single seeded PRNG owned by `SimState`. The seed comes from the JSON input
script. No other source of randomness anywhere in `sim/`.

**Determinism traps to keep on the wall:**

- Reading frame delta-time instead of the fixed tick.
- Iterating `std::unordered_map` / `unordered_set` — order is not stable.
- Uninitialized memory feeding logic.
- Any `rand()` or clock read inside `sim/`.
- Floating point is fine on one machine + compiler. Only cross-platform lockstep
  makes it painful, and we don't need that. But see Decision 12 — it is *not*
  automatically stable across our own build presets.

**The invariant that makes all of this work:** `SimState` is the sole owner of all
mutable simulation state. No statics, no file-scope caches, no sim singleton. If
it can't be serialized out of that struct, it doesn't exist.

```cpp
struct SimState {
    uint64_t       tick;
    Rng            rng;
    ecs::Registry  actors;   // player, enemies, pickups — tens
    BulletPool     bullets;  // thousands
};
```

---

## 7. JSON harness — the contract

**Status:** Settled

Scripted input in, state dump out:

```json
{
  "format": 1,
  "seed": 42,
  "commands": [
    { "tick": 0,   "hold": ["right"], "for": 120 },
    { "tick": 130, "press": ["shoot"] }
  ],
  "expect": [
    { "tick": 120, "player": { "x": 480.0 } }
  ]
}
```

```
./game --headless --script foo.json --run-ticks 400 --dump-state
```

The dump is two-tier:

- **Summary** — a small, stable, human-readable set of fields (tick, player pos,
  hp, live bullet count). This is what `expect` blocks assert against.
- **Full-state hash** — one value over everything, for golden replays (Decision 11).

`--headless` skips window and renderer creation *entirely* — `app/` selects a
headless runner or a windowed runner, and the headless path never links a
renderer. Otherwise CI needs a display and "headless" quietly becomes
"headless-ish".

**Scope cap on `expect`:** exact match on summary fields, plus a float tolerance.
Nothing more. The moment a test wants `expect_any_of` or ranges, that test gets
written in C++ instead. Assertion mini-languages grow teeth.

**Deferred:** interactive stdin mode (step N ticks → emit state → read next
command), which lets an agent play *reactively*. Cheap to add later.

**Library:** `nlohmann/json`, via FetchContent. Speed is irrelevant here.

---

## 8. Build & tooling

**Status:** Settled

- **CMake + Ninja**, `CMAKE_EXPORT_COMPILE_COMMANDS ON` for clangd.
- **Dependencies via FetchContent** with `FIND_PACKAGE_ARGS` — use a system SDL if
  present, build from source otherwise. Repo stays self-contained.
- **Warnings via an INTERFACE target** (`bh_warnings`), linked by our targets only,
  so FetchContent'd dependencies don't drown us in warnings we can't fix. Pair with
  `SYSTEM` on the declarations.

**Four presets, not three:**

| Preset | Flags | Purpose |
|---|---|---|
| `debug` | `-O0 -g` | stepping through code |
| `dev` | `-O0 -g -fsanitize=address,undefined` | default for running; catches bugs at the source |
| `profile` | `-O2 -g` (RelWithDebInfo) | **Instruments** |
| `release` | `-O2`/`-O3` | shipping |

`profile` exists because Instruments on a `-O0` build profiles code you'll never
ship and lies about where time goes, while a stripped release build gives unnamed
frames. ASan and Instruments do not usefully coexist — `dev` and `profile` stay
strictly separate. Given the 10k-bullet goal, expect to live in `profile` a lot.

---

## 9. ECS — hybrid, sparse-set, built after the harness

**Status:** Settled

**Motivation is learning the pattern, not performance.** Recorded explicitly so
future-me doesn't see an ECS next to a 10k-bullet goal and infer a causal link
that isn't there. A generic ECS is *slower* than a purpose-built pool for a
homogeneous array of bullets — it pays for indirection the pool doesn't.

**Hybrid split:**

- **Actors** — player, enemies, bosses, pickups. Tens of entities. These live in
  the ECS, where composition actually helps and per-entity cost is irrelevant.
- **Bullets** — thousands. Dedicated pool (Decision 3). Never entities.

Collision is a free function reading both stores, `(const BulletPool&,
ecs::Registry&)`. It belongs to neither one — resist folding it into the ECS just
because the ECS exists.

**Sparse set**, hand-rolled, not EnTT. EnTT is the right professional answer and
the wrong learning answer. Read its design first, then write ~300 lines:

- `Entity` = packed index + generation (e.g. 20 / 12 bits). The generation is what
  makes stale handles *detectable* rather than silently aliasing a recycled slot.
- Per-type `SparseSet<T>`: `sparse[index] → dense_index`, plus packed
  `dense_components` and `dense_entities` kept in lockstep.
- Type-erased base with a virtual `remove(Entity)` so `Registry::destroy` can sweep
  every pool.
- Views iterate the **smallest** pool and probe the others. Getting this backwards
  is the classic "my ECS is slow" cause.

**Built at Milestone 5, not first.** Building an ECS against imaginary requirements
is the standard failure. It gets extracted once there are ≥3 actor types and real
duplication to delete — so the diff that introduces it *removes* code — and it
happens after the replay tests exist, because it's the riskiest refactor in the
project.

**Determinism note:** packed pools reorder on swap-and-pop, so iteration order
depends on entity history. Still deterministic (same inputs → same order), which
is all we need. But it means dumps must be sorted by entity id — see Decision 12.

---

## 10. No `core/` until something forces it

**Status:** Settled

`core/` has a habit of becoming a junk drawer. It gets created the day two modules
need the same type without one being able to depend on the other — and not before.

Until then: `Vec2` and friends live in `sim/math.hpp`. `render/` can see them
because `render → sim` is already a legal edge. `ecs/` owns its own `Entity` type
and depends on nothing.

**Resulting dependency graph:**

```
app     → platform, render, harness, sim
harness → sim                            (+ nlohmann/json)
render  → sim (const only), platform
platform→ SDL
sim     → ecs
ecs     → (nothing)
```

---

## 11. Test strategy — assertions first, goldens last

**Status:** Settled

Three layers, introduced in this order:

1. **Unit tests** for pure functions — sparse set, math, accumulator logic, script
   parsing.
2. **Assertion replay tests** — a JSON script with an `expect` block. The
   expectation is hand-written and readable, so it survives feature churn and the
   failure diff says what's actually wrong. Tests are *data*, so CTest globs the
   directory and no C++ is written per test — and agents can author them.
3. **Golden replays** — full-state hash comparison. **Machinery built early, golden
   count near zero until the sim stops churning** (Milestone 6).

**Why goldens come last:** a golden test only has value if a break is *surprising*.
While the sim's shape changes daily, every legitimate feature breaks every golden,
`--update-goldens` becomes reflexive, and the suite degrades into a rubber stamp
that records whatever was just done. That is worse than no suite, because it looks
like coverage.

The *machinery* (dump, canonical ordering, hash, compare tool, `--update-goldens`)
is cheap now and annoying to retrofit, so it gets built at Milestone 3 regardless.

---

## 12. State dump invariants

**Status:** Settled

Decided at the first dump ever written, while there is one entity to get it right
on. All three are painful to retrofit.

**1. Versioned.** A `"format": N` field. Old goldens must be recognizably old
rather than mysteriously wrong.

**2. Floats quantized before hashing.** Round positions (e.g. to `1e-4`) or dump
fixed-point integers. Our `dev` (`-O0`) and `profile` (`-O2`) presets can produce
different low bits — FMA contraction and vectorization are real — so a hash taken
under one preset will not match the other. Same reason `expect` blocks compare with
a tolerance, never `==`.

**3. Canonically ordered.** Sort by entity id before serializing. Once the ECS
lands, pool order depends on destruction history, so a harmless refactor of removal
logic would otherwise reorder the dump and invalidate every hash.

---

## Folder structure

```
bullet-hell/
├── CMakeLists.txt
├── CMakePresets.json
├── cmake/
│   └── dependencies.cmake      # FetchContent declarations
├── docs/
│   ├── architecture-discussion.md   # working doc — options and arguments
│   ├── architecture-decisions.md    # this file — settled record
│   └── milestones/                  # one doc per milestone, running log
├── assets/
├── src/
│   ├── ecs/                    # generic ECS. Knows nothing about the game.
│   ├── sim/                    # game rules, components, systems, SimState, InputFrame
│   ├── render/                 # reads SimState, owns the draw path
│   ├── platform/               # the ONLY place SDL exists
│   ├── harness/                # JSON script parse, headless runner, state dump
│   └── app/                    # main.cpp — wiring only
├── tests/
│   ├── unit/
│   └── replays/                # .json scripts + golden outputs
└── tools/                      # run-replays, compare-goldens
```

**Headers live next to sources, not in a top-level `include/`.** That split exists
to publish a library's API to external consumers; there are none. Instead `src/` is
on the include path, so every include is absolute-from-root:

```cpp
#include "sim/world.hpp"     // not "../../sim/world.hpp"
```

Dependency direction becomes visible in every file, and an illegal include is
greppable.

**`ecs/` is deliberately its own target**, not part of `sim/`. It's the most likely
thing to survive into project 2 and 3 — the beginnings of an informal engine — and
keeping it independently buildable is what makes that possible. If it ever needs to
`#include "sim/..."`, that's a design error.

## Target rules

One library target per `src/` directory. The `PRIVATE`/`PUBLIC` distinction is
load-bearing, not cosmetic:

```cmake
add_library(bh_platform window.cpp input.cpp)
target_link_libraries(bh_platform
    PRIVATE SDL2::SDL2      # PRIVATE: SDL headers do NOT leak to consumers
    PRIVATE bh_warnings)
```

`PUBLIC` propagates include directories transitively. Write `PUBLIC SDL2::SDL2`
once and SDL becomes visible everywhere downstream — the enforcement in Decision 5
silently evaporates while still *looking* correct. This is the single thing most
worth getting right in the build files.

---

## Milestones

One document each in `docs/milestones/`, kept as a running history.

| # | Milestone | Doc |
|---|-----------|-----|
| 0 | Toolchain skeleton — window opens | `00-toolchain-skeleton.md` |
| 1 | Fixed-timestep loop + InputFrame | `01-fixed-timestep-loop.md` |
| 2 | JSON harness + headless + state dump | `02-json-harness.md` |
| 3 | Test infrastructure | `03-test-infrastructure.md` |
| 4 | Bullet pool → 10k → Instruments baseline | `04-bullet-pool.md` |
| 5 | Sparse-set ECS for actors | `05-sparse-set-ecs.md` |
| 6 | Golden replays | `06-golden-replays.md` |

The ordering is itself a decision: the ECS (5) lands *after* replay tests (2, 3),
because it's the riskiest refactor in the project and should be made with a
regression net already underneath it.

---

## Decisions log

| # | Decision | Date | Rationale |
|---|----------|------|-----------|
| 2 | Fixed timestep, no interpolation yet | 2026-07-25 | Determinism is the foundation for the harness and every replay test |
| 3 | Bullet pool, SoA deferred to measurement | 2026-07-25 | Premature SoA is as much a mistake as premature abstraction |
| 5 | sim/render/platform boundary, CMake-enforced | 2026-07-25 | Retrofitting the seam is miserable; targets make it a build error |
| 6 | Single seeded PRNG owned by SimState | 2026-07-25 | Cheap now, load-bearing forever |
| 7 | Scripted harness + two-tier dump | 2026-07-25 | Tests as data; agents can author them |
| 8 | CMake + Ninja, four presets incl. `profile` | 2026-07-25 | Profiling a `-O0` build lies about where time goes |
| 9 | Hybrid ECS: actors in, bullets out; sparse-set; built at M5 | 2026-07-25 | ECS is a *learning* goal, and must not be allowed to threaten the 10k target |
| 10 | No `core/` until two modules force it | 2026-07-25 | Junk-drawer prevention; DAG is simpler without it |
| 11 | Assertion replay tests first, goldens last | 2026-07-25 | A golden only has value if a break is surprising |
| 12 | Dump versioned, quantized, canonically ordered | 2026-07-25 | Float bits differ across presets; pool order depends on history |

---

## Open questions parking lot

- Sprite atlas layout / asset pipeline — defer until batching exists.
- Audio approach (SDL_mixer vs. raw SDL audio) — low priority for feel-testing.
- How enemy bullet *patterns* are authored (data-driven tables vs. code) — a real
  decision, later than the engine seams. Blocks Milestone 6.
- Interactive stdin harness mode for reactive agent play.
