# Milestone 6 — Golden Replays

**Status:** Not started
**Depends on:** M3 (machinery), M5 (sim shape stabilized)
**Unblocks:** confident large refactors from here on

---

## Goal

Turn on the golden-replay layer that M3 built and left dormant: long scripted runs
whose full-state hash is recorded and compared.

## Why it's last (Decision 11)

A golden test only has value if a break is **surprising**. Recorded while the sim's
shape changes daily, every legitimate feature breaks every golden,
`--update-goldens` becomes reflexive, and the suite degrades into a rubber stamp
that records whatever was just done — which is worse than no suite, because it
looks like coverage.

By M6 the entity model, the bullet pool, and the dump format have all held still
through a major refactor. *Now* a break means something.

## Entry condition

- [ ] Dump format hasn't changed in a while — no version bump for several sessions.
- [ ] M5 landed and the replay suite stayed green through it.
- [ ] Bullet patterns are authored in whatever form they'll keep (see parking lot —
      this is the last open question that would churn the state).

## Done when

- [ ] A handful (**not** dozens) of long replays — ~2000+ ticks each — with recorded
      hashes in `tests/replays/goldens/`.
- [ ] Goldens cover things assertion tests can't: emergent state after long runs,
      full bullet-field evolution, RNG-driven pattern sequences.
- [ ] `--update-goldens` requires an explicit flag and prints what changed. Never
      the default, never silent.
- [ ] A golden break shows a *useful* diff — which tick diverged first, not just
      "hash mismatch".
- [ ] Goldens verified stable across `debug` and `profile` presets (this is the real
      test of Decision 12's quantization).
- [ ] The repo documents the rule: **a golden break is investigated before it is
      re-recorded**, always.

## In scope

- Recording the goldens.
- First-divergence reporting (bisect over per-tick hashes).
- Documenting the update policy.

## Explicitly out of scope

- Turning every replay test into a golden. The assertion tests from M3 remain the
  primary layer; goldens are a small, targeted top layer.
- Cross-platform golden stability. Single machine + compiler is all we need
  (Decision 6) and chasing more is a different project entirely.

## Concepts this milestone teaches

- **Characterization testing** — pinning behaviour you haven't fully specified, and
  its trade-off: maximum sensitivity, minimum diagnostic value.
- Why the *diff quality* of a failing test determines whether the suite gets used
  or ignored.
- Bisecting a divergence over time — the technique behind every "our replay
  desynced at frame 8412" debugging story.
- The social half of testing: an update policy is what stops a golden suite
  decaying, and no amount of tooling substitutes for it.

## Known gotchas

- **Per-tick hashes, not just a final hash.** A single end-state hash tells you
  something broke and nothing about where. Store a cheap rolling hash per tick, or
  support `--dump-every N`, so first divergence is findable.
- **`--update-goldens` must be loud.** If it's convenient, it becomes reflexive,
  and Decision 11's entire argument reasserts itself at the tooling layer.
- If goldens differ between presets, the quantization from Decision 12 isn't
  aggressive enough. Fix the dump, don't loosen the comparison.
- Beware goldens that pass for the wrong reason — a replay where the player dies at
  tick 30 and nothing happens for the remaining 1970 ticks is a very stable, very
  useless golden. Check the summary looks alive.
- Keep goldens few. Their maintenance cost is superlinear in count, because every
  behaviour change touches all of them at once.

## Open questions

- How many goldens? (Lean: 3–5. One quiet run, one dense bullet field, one
  RNG-heavy pattern sequence.)
- Per-tick rolling hash, or store a sparse set of checkpoint hashes?
- Do goldens live in git as plain text (diffable, noisy) or as a single manifest?

## Log

| Date | Entry |
|------|-------|
|      |       |
