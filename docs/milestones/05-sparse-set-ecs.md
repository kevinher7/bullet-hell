# Milestone 5 — Sparse-Set ECS for Actors

**Status:** Not started
**Depends on:** M3 (regression net), M4 (enough actor types to justify it)
**Unblocks:** nothing structurally — this is a refactor, not a feature

---

## Goal

Hand-roll a sparse-set ECS and move the **actors** — player, enemies, bosses,
pickups — into it. Bullets stay in their pool (Decision 9).

## The honest framing

**This is a learning goal, not a performance goal.** Written here so future-me
doesn't see an ECS next to a 10k-bullet target and infer a causal link. A generic
ECS is *slower* than the M4 pool for a homogeneous array of bullets; it pays for
indirection the pool doesn't. What it buys is composition over ~tens of entities,
and a genuinely instructive pile of low-level C++.

**It happens at M5 and not earlier** because an ECS built against imaginary
requirements gets built wrong. The entry condition below is a real gate, not a
formality.

## Entry condition

Do not start until **all** of these hold:

- [ ] There are **≥3 distinct actor types** with overlapping-but-not-identical state.
- [ ] There is **real duplication** to delete — the introducing diff should be
      net-negative or close to it.
- [ ] The replay suite from M3 is green and covers actor behaviour, so the refactor
      has a net underneath it.

If the entry condition isn't met, the correct action is to keep building the game
and come back. Note the date you checked in the log either way.

## Done when

- [ ] `ecs/` builds as a standalone target depending on **nothing** (Decision 10).
- [ ] `Entity` = packed index + generation. A stale handle is *detected*, and there
      is a test proving it — destroy an entity, recycle the slot, confirm the old
      handle is rejected rather than silently aliasing the new occupant.
- [ ] `SparseSet<T>` with `sparse[index] → dense_index`, plus packed
      `dense_components` and `dense_entities` kept in lockstep.
- [ ] Type-erased pool base with virtual `remove(Entity)`, so `Registry::destroy`
      sweeps every pool.
- [ ] Views iterate the **smallest** pool and probe the others.
- [ ] Actors migrated; `SimState` now holds `ecs::Registry actors` alongside
      `BulletPool bullets`.
- [ ] **Every replay test still passes, unchanged.** This is the milestone's real
      acceptance criterion.
- [ ] Dumps still sorted by entity id (Decision 12) — verify after migration, since
      this is exactly when pool ordering starts depending on history.

## In scope

- `src/ecs/` in full: entity handles, sparse sets, registry, views.
- Migrating actors in `sim/`.
- Unit tests for the ECS itself, independent of the game.

## Explicitly out of scope

- **Bullets.** They are never entities.
- Archetype storage. Sparse set was chosen (Decision 9); archetypes are a
  different, harder project.
- Systems scheduling, parallelism, event buses — all the ECS-framework features
  that have nothing to do with a single-threaded shmup.
- Replacing this with EnTT. If that ever happens, it's a new decision with its own
  entry in the log.

## Concepts this milestone teaches

- **Type erasure**: storing heterogeneous component pools behind one base, and why
  `virtual` shows up in a design people call "data-oriented".
- **Compile-time type indices**: giving each component type a stable runtime id
  without RTTI.
- **Generational handles** and the class of bug they eliminate — the single most
  transferable idea here.
- Placement new, alignment, manual object lifetime in a packed buffer.
- **Iterator and view design**: what it takes for `for (auto [e, pos, vel] : view)`
  to be both ergonomic and fast.
- Reading someone else's design (EnTT) before writing your own — and where you
  deliberately diverge.

## Known gotchas

- **Views must drive from the smallest pool.** Iterating the largest and probing is
  the classic "my ECS is slow" cause.
- **Iterator invalidation during iteration.** Creating or destroying entities inside
  a view loop is the second classic bug. Decide the rule now — deferred command
  buffer, or "not allowed, asserted" (lean: the latter; it's simpler and this game
  doesn't need more).
- **`sparse` must be sized/bounds-checked**, or a stale index writes into arbitrary
  memory. ASan under `dev` will catch this — another reason the suite runs there.
- **`ecs/` must never `#include "sim/..."`.** That edge is a design error, and it's
  what keeps this reusable for project 2.
- Don't fold collision into the ECS just because the ECS now exists. It reads both
  stores and belongs to neither.
- Watch for the refactor quietly changing update *order*. Same inputs must still
  produce same outputs — if a replay test breaks, the ECS is wrong, not the test.

## Open questions

- Entity bit split: 20/12? 24/8? What's the realistic max live actor count?
- Component registration: explicit, or lazily on first use?
- Does the player get a component set like everything else, or stay special-cased?
  (Lean: like everything else — special cases are what the ECS is supposed to
  delete.)

## Log

| Date | Entry |
|------|-------|
|      |       |
