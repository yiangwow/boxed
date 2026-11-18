# Architecture Decision Records

## ADR-001: Web platform, no engine, grid-based deterministic sim

**Date:** 2026-07-27
**Status:** Accepted

**Context:** Boxed is a spec-writing puzzle game. The UI is clause composition
(HTML's strength), the world is small discrete scenarios, and the central mechanic
requires generating arbitrary structures that provably satisfy player-written
clauses. Candidates: Unity, Claude Artifact single-file, own-repo web app.

**Decision:** TypeScript + Vite + Canvas2D in a dedicated repo. 2D grid world.
Fully deterministic seeded simulation with the entity sim as pure functions. The
clause-satisfaction checker is the core unit, enforced by property tests (fairness
invariant: instantiated entities always satisfy their spec).

**Consequences:**
- Spec-composition UI in plain HTML/CSS — much cheaper than Unity UI.
- Discrete grid keeps conforming-model generation tractable and failures legible.
- Determinism makes failures replayable and the core testable without a browser.
- No engine features (physics, animation tooling) — acceptable: stark, minimal
  presentation is the intended aesthetic.
- Artifact rejected: content volume and save systems outgrow single-file constraints.
- Unity rejected: engine weight buys nothing this design needs.
