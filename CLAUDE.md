# CLAUDE.md

## Project

**Boxed** (working title) — puzzle-sandbox web game. You specify helpers into
existence; the Instantiator always produces something satisfying every clause you
wrote but never what you meant. Design spec:
`docs/superpowers/specs/2026-07-27-boxed-game-design.md`.

## Architecture (planned — update as built)

- TypeScript + Vite, Canvas2D, zero engine dependencies
- Deterministic seeded simulation; entity sim as pure functions
- Core invariant: every instantiated entity satisfies every clause of its spec
  (property-tested; never violate — it is the game's contract with the player)

## Status

Pre-implementation. Spec approved 2026-07-27. No code yet.

## Commands

None yet — scaffold pending (Vite + Vitest expected).

## Workflow

- Feature branch + PR, never push main directly
- Authorship: `yiangwow` / `<removed>`, no Co-Authored-By lines
- Architectural decisions go in `ADR.md`
- Update this file in the same commit as any change affecting it
