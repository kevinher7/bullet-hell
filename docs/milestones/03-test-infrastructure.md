# Milestone 3 — Test Infrastructure

**Status:** Not started
**Depends on:** M2
**Unblocks:** M5 (the ECS refactor should not happen without this)

---

## Goal

Turn the harness into a test suite. Two layers now, the third (goldens) deliberately
built-but-unused.

## Done when

- [ ] `ctest --preset dev` runs and passes.
- [ ] Unit tests exist for pure logic: `Vec2` math, the accumulator, `Rng`
      reproducibility, script parsing (including malformed input).
- [ ] `expect` blocks work: a JSON script asserts on summary fields with a float
      tolerance.
- [ ] `tests/replays/*.json` is **globbed** — adding a test is adding a file, with
      no C++ written and no CMake edited.
- [ ] A failing assertion prints expected vs actual vs tick, legibly.
- [ ] Golden machinery exists and is exercised by exactly one smoke test:
      `--dump-hash`, a compare path, and `--update-goldens`.
- [ ] At least one test that *should* fail does fail (verify the suite can go red).

## In scope

- A test framework — doctest or Catch2, via FetchContent.
- `tests/unit/`, `tests/replays/`.
- The `expect` block extension to the script format (`"format": 2`).
- `tools/` scripts for running replays and updating goldens.
- CTest registration.

## Explicitly out of scope

- **Actual golden files.** Machinery only. Decision 11 — the sim's shape is still
  churning daily and goldens recorded now would be noise.
- Performance tests (M4 handles the profiling story).

## The `expect` contract

```json
{
  "format": 2,
  "seed": 42,
  "commands": [ { "tick": 0, "hold": ["right"], "for": 120 } ],
  "expect": [
    { "tick": 120, "player": { "x": 480.0 } }
  ]
}
```

**Hard scope cap:** exact match on summary fields, plus a float tolerance. Nothing
else. When a test wants ranges, `any_of`, or conditionals, that test gets written
in C++. Assertion mini-languages grow teeth and then you're maintaining a language
instead of a game.

## Concepts this milestone teaches

- **Tests as data vs tests as code**, and where the boundary sits. Data tests are
  authorable by tools and agents; code tests are expressive. You want both, for
  different things.
- Why a readable, hand-written expectation survives refactors that a recorded
  snapshot doesn't — the core argument in Decision 11.
- CTest wiring, test discovery, and how a suite goes red.
- Testing *nondeterminism-adjacent* things: asserting a seeded RNG is reproducible
  is a different exercise from asserting a function is correct.

## Known gotchas

- **CMake glob needs `CONFIGURE_DEPENDS`**, or a newly added test file won't be
  picked up until you reconfigure — and you'll conclude your test "passes" when it
  never ran.
- **A test suite that can't go red is worthless.** Verify it fails before trusting
  it green. Do this deliberately, once, at the start.
- Float tolerance: pick a value and put it in this doc. Too tight and it's flaky
  across presets; too loose and it stops catching regressions.
- Resist the urge to record goldens "while you're here". That's the exact failure
  Decision 11 exists to prevent.
- Don't let the `expect` schema and the dump schema drift — they name the same
  fields and should share one definition.

## Open questions

- doctest or Catch2? (doctest is faster to compile and enough for this; Catch2 is
  more common in the wild.)
- Float tolerance value?
- Should replay tests run under `dev` (ASan, slow) or `debug`? Running the suite
  under sanitizers catches far more, at the cost of wall-clock. Lean: `dev`, and
  reconsider if the suite gets slow enough to skip.

## Log

| Date | Entry |
|------|-------|
|      |       |
