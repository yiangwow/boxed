# Boxed — Game Design Spec

**Date:** 2026-07-27
**Status:** Approved (brainstorm complete, pre-implementation)
**Working title:** Boxed (rename freely before public release)

## Origin

Derived from a philosophy conversation (2026-07-26, "Observation, Measurement, and
Abstract Systems"): any finite rule set underdetermines the system it describes —
impostor structures satisfy every rule while not being what you meant. The game does
not teach this. Its mechanics simply behave this way. **Weirdness as physics, not
curriculum.**

## Concept

A puzzle-sandbox game where you accomplish tasks by *specifying* helpers into
existence. The Instantiator always produces something that satisfies every clause you
wrote and is never what you meant. The enemy is the gap between intent and
specification.

## Core loop

1. A scenario presents a need: cross the chasm, keep the fire lit, protect the
   sleeping thing, move the heavy thing.
2. The player composes a spec from clause chips and submits it. **Finite clause
   budget** — the spec can never be complete. The skill is choosing which constraints
   matter *in this scenario*.
3. The Instantiator produces a conforming entity. The player watches it act in the
   simulated world and discovers the clause they didn't write.
4. On failure: **the receipt** — every clause displayed, checkmarked, satisfied,
   green, over the wreckage. No commentary, ever. The receipt is the message.
5. Revise and resummon. Resummons per scenario are bounded, so failure costs
   something and observing before committing matters.

## Spec language

- Structured clause chips, not free text. Deterministic, machine-checkable, fair.
- A clause = predicate template + bindings:
  - `IT SPANS [the gap]`
  - `IT BEARS [my weight]`
  - `IF [fire dims] IT [feeds it]`
  - `IT DOES NOT [move]`
  - `IT IS [rigid]`
- **Vocabulary grows through betrayal.** The player starts with a tiny predicate
  set. When an impostor exploits a dimension the player never considered (the bridge
  *sways* → unlock `RIGID`; the guardian counts *you* as a threat → unlock
  `EXCEPT [me]`), that predicate enters the vocabulary. You can only rule out what
  has already burned you. Progression is accumulated scar tissue — structurally, a
  test suite.

## The Instantiator

- **Chapter 1:** picks a *random* conforming entity from the generative space.
  Failures feel like cosmic slapstick.
- **Later chapters:** picks the *minimally compliant* conforming entity — candidates
  scored against heuristics of likely player intent, then the pick is deliberately
  distant from intent. Failures start to feel selected, aware, declining. Whether it
  is malice or pattern-matched indifference is never answered, in-game or in docs.
- **Fairness invariant (absolute, machine-checked):** every entity ever instantiated
  satisfies every clause of the spec that summoned it. This is a property test in
  the codebase and the game's contract with the player. If the game ever cheats,
  the design collapses into "gotcha."

## World & presentation

- 2D grid scenarios, Canvas2D. Sparse and quiet: dark field, thin light, entities as
  stark readable shapes. Lain-tinged stillness, not fantasy-summoner flavor.
- The grid is a design choice, not a compromise: discrete space makes
  conforming-model generation tractable and failures legible.
- Chapters are places; scenarios within a chapter share terrain and mood.

## Progression

- Chapters gate: new scenario types, new predicate unlocks (via betrayal), slowly
  growing clause budget, Instantiator adversarialness ramp.
- No scores, no timers. Cost pressure comes from bounded resummons.

## Endgame (sketch only — detailed design deferred)

**The Categorical Voice:** a second spec language that pins the summon down exactly —
what you mean is what you get. Cost: no preview, no receipt, no verification. You can
only trust it. Certainty or verification, never both. Presented as an item with a
strange price and zero exposition. Not built until the core loop proves out.

## Tech

- TypeScript + Vite, Canvas2D, zero engine dependencies.
- Deterministic seeded simulation: entity sim as pure functions → replayable
  failures, testable core.
- The clause-satisfaction checker is the heart of the codebase and the most heavily
  tested unit (property tests enforcing the fairness invariant).
- Static bundle deploy (own server or GitHub Pages).

## Out of scope (v1)

- Free-text specs / LLM-driven instantiation
- Multiplayer, level editor, sound (v2 candidates)
- Categorical Voice implementation (sketch above only)

## Success criteria

- The core loop (spec → instantiate → watch → receipt → revise) is fun within one
  scenario, validated by a playable vertical slice before content buildout.
- The fairness invariant holds under property testing across the full generative
  space.
- A first-time player, without any tutorial text about the philosophy, describes an
  impostor failure as "my fault, I didn't say it couldn't do that."
