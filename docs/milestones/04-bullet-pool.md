# Milestone 4 — Bullet Pool, 10k on Screen, Instruments Baseline

**Status:** Not started
**Depends on:** M3
**Unblocks:** the SoA experiment; informs M6

---

## Goal

The headline goal of the project: **10,000+ bullets simulated and drawn at 60 FPS**,
with a profiler baseline recorded rather than guessed at.

This is where the project stops being scaffolding.

## Done when

- [ ] `BulletPool` is a fixed-capacity pool inside `SimState`. Swap-and-pop on
      death. **Zero allocation during a frame.**
- [ ] A debug spawner can put 10k+ bullets on screen on demand.
- [ ] Drawing goes through one texture and a batched submission
      (`SDL_RenderGeometry`), **not** one `SDL_RenderCopy` per bullet.
- [ ] Naïve collision: one player hitbox vs. all bullets (Decision 4).
- [ ] Sustained 60 FPS at 10k under the `release` preset, measured not vibed.
- [ ] **An Instruments Time Profiler trace captured under the `profile` preset**,
      with the top costs written into the log below as *numbers*.
- [ ] Replay tests still pass, now including one that spawns thousands of bullets
      and asserts the live count after N ticks.

## In scope

- `sim/bullets.hpp` — the pool, update, spawn, despawn.
- `sim/collision.*` — a free function over `(const BulletPool&, ...)`.
- `render/` — the batched draw path, sprite atlas (minimum viable: one quad).
- A debug spawn pattern good enough to stress the system.

## Explicitly out of scope

- **SoA.** It is the *next* thing, and only if the trace justifies it. Decision 3.
- Spatial partitioning for collision. Decision 4's trigger is profiler evidence.
- Real enemy bullet patterns — deferred (parking lot), and not needed to hit 10k.
- The ECS. Bullets are never entities (Decision 9).

## Concepts this milestone teaches

- **Why draw calls dominate.** The one-`RenderCopy`-per-bullet version is worth
  writing first, measuring, and then deleting. Feeling the cliff is the lesson;
  reading about it is not.
- **Cache behaviour**: contiguous iteration vs pointer chasing, and how to see the
  difference in a trace rather than assert it in a code review.
- **Pool/free-list allocation**: stable indices, generational handles, why
  swap-and-pop is fine when order doesn't matter.
- **Profiler literacy**: sampling vs instrumentation, why `-O0` traces lie, reading
  an inverted call tree, distinguishing "hot because slow" from "hot because
  called a lot".

## Known gotchas

- **Profile under `profile`, never `dev` or `debug`.** ASan distorts everything and
  `-O0` profiles code that doesn't ship. Decision 8.
- **The 60 FPS claim needs a stated measurement method.** Frame time percentiles,
  not an average — a 16 ms average with 40 ms spikes does not feel fluid, and
  "feels fluid" is goal #1.
- Vsync will mask real headroom. Measure with it off to find the actual ceiling,
  then turn it back on.
- Swap-and-pop invalidates indices. Anything holding a bullet index across a
  despawn is a bug — prefer not holding them at all.
- Don't let the debug spawner become the game. It's a stress tool.
- Beware optimizing what the profiler shows *without* checking the replay tests
  still pass — this is the first milestone where a "harmless" perf change can
  silently alter sim behaviour.

## Open questions

- Pool capacity: fixed at a hard max (e.g. 32k) or growable? (Lean: hard max, with
  a loud assert on overflow — growth mid-frame is an allocation, which is the thing
  being eliminated.)
- Do bullets need generational handles at all, if nothing holds a reference to one?
- Atlas format and how sprites get addressed — minimum viable now, real answer
  deferred (parking lot).

## Baseline measurements

_Fill in from the Instruments trace. Numbers, not adjectives — future comparisons
depend on these._

| Date | Build | Bullet count | Frame time p50 / p99 | Top 3 costs |
|------|-------|--------------|----------------------|-------------|
|      |       |              |                      |             |

## Log

| Date | Entry |
|------|-------|
|      |       |
