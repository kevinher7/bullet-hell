# Milestone 0 — Toolchain Skeleton

**Status:** Not started
**Depends on:** nothing
**Unblocks:** everything

---

## Goal

Prove the toolchain end to end before writing anything interesting. A window opens
and closes cleanly. That's the entire feature set.

The value here isn't the window — it's that every later milestone builds on a
configure/build/run loop that is already known to work, so a failure at Milestone 4
is a Milestone 4 problem and not a linker mystery.

## Done when

- [ ] `cmake --preset dev && cmake --build --preset dev` succeeds from a clean clone.
- [ ] The binary opens an SDL window, waits for a close event, exits with status 0.
- [ ] `compile_commands.json` exists and clangd resolves SDL headers in the editor.
- [ ] All four presets (`debug`, `dev`, `profile`, `release`) configure and build.
- [ ] `dev` demonstrably has ASan on — deliberately leak something, confirm it reports, remove it.
- [ ] Empty target dirs exist with placeholder `CMakeLists.txt`: `ecs`, `sim`, `render`, `platform`, `harness`, `app`.

## In scope

- Root `CMakeLists.txt`, `CMakePresets.json`, `cmake/dependencies.cmake`.
- SDL2 via FetchContent with `FIND_PACKAGE_ARGS`.
- The `bh_warnings` INTERFACE target.
- `src/platform/` creating and destroying a window.
- `src/app/main.cpp` doing nothing but wiring.

## Explicitly out of scope

- Any drawing beyond a clear-to-a-color.
- Any game logic whatsoever.
- `nlohmann/json` — not needed until Milestone 2.
- Tests — Milestone 3.

## Concepts this milestone teaches

- **Targets as the unit of a CMake project**, not files. Everything downstream
  hangs off this.
- **`PRIVATE` vs `PUBLIC` vs `INTERFACE` linkage** and usage-requirement
  propagation. This is where the Decision 5 boundary is actually implemented — get
  it wrong here and the enforcement is decorative for the rest of the project.
- **FetchContent vs `find_package`** and why the repo shouldn't depend on host
  state.
- The build/link pipeline: what a translation unit is, what the linker is actually
  complaining about when it complains.

## Known gotchas

- **`PUBLIC SDL2::SDL2` is the trap.** It propagates SDL's include dirs to every
  consumer and quietly destroys the seam. `PRIVATE`.
- SDL2 built from source on macOS: decide `SDL_SHARED` vs `SDL_STATIC` now. Static
  gives a self-contained binary and one less runtime-path problem.
- First FetchContent configure is slow (builds SDL). That's a one-time cost, not a
  broken build — don't go chasing it.
- `SYSTEM` on the FetchContent declaration keeps SDL's own warnings out of your
  output. Without it, `-Wconversion` will bury you.
- Apple Clang's C++20 support is good but not complete. If something exotic doesn't
  compile, check the standard-library support table before assuming a code bug.

## Open questions

- SDL2 static or shared on macOS?
- Do we want `-Werror` from day one? (Lean: no — it turns exploratory code into a
  fight. Add it once the project has habits.)

## Log

_Running history. Date, what happened, what surprised me._

| Date | Entry |
|------|-------|
|      |       |
