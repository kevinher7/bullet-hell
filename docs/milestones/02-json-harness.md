# Milestone 2 — JSON Harness, Headless Mode, State Dump

**Status:** Not started
**Depends on:** M1
**Unblocks:** M3, and safety-nets the M5 ECS refactor

---

## Goal

Run the sim with no window, driven by a JSON script instead of a keyboard, and
print the resulting state.

Built early *on purpose*: the seam is cheap to create now and miserable to
retrofit, and every risky refactor from here on gets to lean on it.

## Done when

- [ ] `./game --headless --script foo.json --run-ticks 400 --dump-state` works.
- [ ] `--headless` creates **no window and no renderer** — the headless path never
      touches `render/` or SDL video at all.
- [ ] A script produces byte-identical dumps across repeated runs on the same build.
- [ ] Running the same script under `debug` and under `profile` produces identical
      *summary* dumps (this is what forces float quantization to be real).
- [ ] The dump carries `"format": 1`, floats are quantized, entries are sorted by id.
- [ ] The same `InputFrame` type feeds both the real input path and the script path,
      with `sim::update` unmodified between them.

## In scope

- `harness/`: script parsing, a script→`InputFrame` driver, the headless runner,
  the state dump (summary + hash).
- `nlohmann/json` via FetchContent.
- `app/`: argument parsing; choose headless runner or windowed runner.

## Explicitly out of scope

- **`expect` blocks and test running** — that's M3. This milestone produces and
  prints state; it doesn't judge it.
- Golden files (M6).
- Interactive stdin mode (deferred, Decision 7).

## The contract (Decision 7)

```json
{
  "format": 1,
  "seed": 42,
  "commands": [
    { "tick": 0,   "hold": ["right"], "for": 120 },
    { "tick": 130, "press": ["shoot"] }
  ]
}
```

Dump is two-tier:

- **Summary** — tick, player pos, hp, live bullet count. Small, stable, readable.
- **Hash** — one value over full state. Emitted now, *compared* starting at M6.

## Concepts this milestone teaches

- **Dependency inversion in practice.** The sim doesn't know what's driving it.
  That's the entire payoff of the M1 ceremony, and this is where you feel it.
- **Determinism as a testable property**, not an aspiration — run twice, diff.
- Serialization and the difference between a *debug* dump (everything, unstable)
  and a *contract* dump (small, versioned, stable).
- Why float equality is a lie across optimization levels.

## Known gotchas

- **Quantize floats before they ever reach the output**, not at compare time.
  `-O0` and `-O2` produce different low bits (FMA contraction, vectorization), so
  an unquantized dump is not stable across your own presets. Decision 12.
- **Don't dump the whole world.** A dump that includes everything is noisy and
  breaks on every change; the summary is chosen deliberately to be the things you'd
  actually assert on.
- **`"format": 1` from the first line of output.** Free now; the alternative later
  is goldens that fail mysteriously instead of legibly.
- Command semantics need pinning down: does `hold` end when a later command
  contradicts it, or only after `for` ticks elapse? Ambiguity here becomes flaky
  tests. Write the rule in this doc once you pick it.
- Headless must not depend on SDL video init at all — if it does, CI needs a
  display and the whole point is lost.

## Open questions

- Does the script run out of commands mean "stop" or "keep simulating with no
  input"? (`--run-ticks` suggests the latter — state it explicitly.)
- Hash function: FNV-1a is plenty and trivially portable. Anything stronger is
  costume.
- Should the dump be emitted every N ticks or only at the end? (Lean: end only,
  plus optional `--dump-every N` for debugging a divergence.)
- Where does the quantization epsilon live — hardcoded, or part of the format?

## Log

| Date | Entry |
|------|-------|
|      |       |
