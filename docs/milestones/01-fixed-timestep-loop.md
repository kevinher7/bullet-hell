# Milestone 1 — Fixed-Timestep Loop + InputFrame

**Status:** Not started
**Depends on:** M0
**Unblocks:** M2 (the harness needs `InputFrame` and `sim::update` to exist)

---

## Goal

A rectangle you can move with the keyboard, driven by a fixed-timestep loop, where
`sim/` already contains **zero** SDL from its very first commit.

This is the milestone that establishes the shape of the whole program. The
rectangle is a pretext.

## Done when

- [ ] `sim::update(SimState&, const InputFrame&)` advances exactly one tick (1/60 s).
- [ ] The main loop uses an accumulator: consume real elapsed time, run 0..N sim
      ticks, then render once.
- [ ] A rectangle moves in response to arrow keys and stops at the arena bounds.
- [ ] Movement speed is identical at 60 Hz and at an uncapped frame rate.
- [ ] `bh_sim` does **not** link SDL — verified by adding `#include <SDL.h>` to a
      sim file, watching it fail to compile, and removing it.
- [ ] `SimState` holds every piece of mutable sim state. No statics anywhere in `sim/`.
- [ ] `Rng` lives in `SimState` and is seeded explicitly, even though nothing uses
      it yet.

## In scope

- `sim/`: `SimState`, `InputFrame`, `sim::update`, `sim/math.hpp` (`Vec2`), `Rng`.
- `platform/`: SDL event pump → `InputFrame`; a monotonic clock source.
- `render/`: clear + draw one rectangle from `const SimState&`.
- `app/`: the accumulator loop.

## Explicitly out of scope

- Bullets, enemies, collision, shooting.
- The ECS. Player state is a plain struct member of `SimState` for now — and that
  is the *point*: M5 has to earn its existence by deleting something.
- Render interpolation (Decision 2 defers it).
- JSON, headless mode, tests.

## Concepts this milestone teaches

- **The accumulator pattern** and why "spiral of death" happens: if a sim tick
  takes longer than the tick duration, the accumulator grows without bound. Clamp
  the max ticks per frame.
- **Why variable timestep poisons determinism** — feel it by writing the naive
  `pos += vel * dt` version first, then noticing you can never reproduce a run.
- **Input as a value type.** `InputFrame` is the hinge: `sim::update` cannot tell a
  real keyboard from a JSON script. Everything in M2 depends on this being a plain
  struct with no SDL types in it.
- Passing state explicitly rather than reaching for globals — the ceremony that
  buys the entire replay story.

## Known gotchas

- **`InputFrame` must not contain SDL types.** No `SDL_Scancode`, no
  `SDL_Keysym`. Define your own `enum class Button`. If SDL's enum leaks into the
  hinge type, `sim/` transitively depends on SDL and the seam is gone.
- **Held vs pressed vs released** are three different things and the harness will
  need all three. Decide the representation now (lean: a bitmask of currently-held
  buttons plus a bitmask of newly-pressed-this-tick).
- Don't read wall-clock time inside `sim/`. `SimState.tick` is the only notion of
  time the sim has.
- Rendering must not mutate. Take `const SimState&` even when it's inconvenient —
  especially when it's inconvenient.

## Open questions

- Bitmask or `bool[N]` for `InputFrame`? (Bitmask serializes and diffs more
  cleanly for the harness — probably worth it.)
- Does `InputFrame` carry analog values (for a future gamepad), or stay digital?
  Deferring is fine; note that the JSON schema in M2 inherits this choice.
- Clamp on max sim ticks per frame: what value? (Lean: 5. Any more and you're
  visibly time-dilating anyway.)

## Log

| Date | Entry |
|------|-------|
|      |       |
