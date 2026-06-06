---
name: project-gdd16-acs
description: 26 ACs for GDD #16 (Narrative Progression); Re-Review #3 completed 2026-06-05 — 5 BLOCKING findings, suite still not test-ready
metadata:
  type: project
---

GDD #16 Sistema de Progresión Narrativa — Acceptance Criteria authored on 2026-05-31. 26 total ACs (CA-PN-001 through CA-PN-025 plus CA-PN-015a/b split).

## Re-Review #3 — 2026-06-05 — BLOCKED

Adversarial review of Re-Review #2 rewrites (2026-06-04). 5 BLOCKING findings remain. Suite cannot be handed off to test implementation.

**BLOCKING findings (must fix before test writing begins):**

- FINDING-R3-01 / CA-PN-006: Re-entrance path unspecified. AC says "configure listener to call `_on_trigger`" but `_on_trigger` is private (underscore prefix). Must specify whether (a) method is exposed publicly, or (b) re-entrance is triggered by re-emitting the EventBus signal rather than direct call. The execution path matters for what is actually being tested.

- FINDING-R3-02 / CA-PN-008: "or EventBusMock" ambiguity NOT resolved — institutionalized instead. Two incompatible harnesses remain equally valid. Must pick one: recommend DI/EventBusMock per [[feedback-test-infrastructure]] pattern. `watch_signals(real EventBus)` should be deprecated as authoritative for blocking ACs.

- FINDING-R3-03 / CA-PN-012: MATH ERROR. True value = 68.974%, which falls OUTSIDE the ±0.01 tolerance band [68.99, 69.01]. GDD example rounds 0.6923×0.90 to 0.623 (truncated), then 0.6667×0.10 to 0.067 — producing apparent 0.690, but unrounded computation is 0.68974. Fix: change expected value to 68.97 OR expand tolerance to ±0.05. Also update F3 example in GDD body to use unrounded intermediates for consistency.

- FINDING-R3-04 / CA-PN-020: "global call N=2" language STILL PRESENT. "Global" contradicts "variable de instancia." Must replace with "la segunda llamada a `set_major_event` dentro de esta invocación del gate."

- FINDING-R3-05 / F1-ext / climax_tono: NEW BLOCKING GAP. Zero ACs for the only Pilar 5 gate in Act 1. Needs: (a) gate fires at corruption_level=0.5, (b) gate does not fire at corruption_level<0.5, (c) get_nested returns graceful default for absent path. Also: get_nested is listed as an Open Question in GDD #4 — ACs cannot be implemented until that Open Question is resolved.

**RECOMMENDED findings (should fix before sprint review):**

- FINDING-R3-06 / CA-PN-019: B6 order invariant documented as intentionally uncovered — gap acknowledged but remains with zero test coverage. Flag at sprint review.

- FINDING-R3-07 / CA-PN-025: Compound AC structural weakness remains. player_choice record_event call not verified in emboscada→clímax chain. CA-PN-024 covers it in isolation; full chain untested.

- FINDING-R3-08 / CA-PN-018: ResourceLoader DI now specified (good). Log capture mechanism still undefined — "log capturado" has no specified object. Must specify injectable logger interface or GUT error assertion method.

**Priority fix order for Re-Review #4:**
1. FINDING-R3-03 (math — 5 min word change)
2. FINDING-R3-04 (word substitution — 2 min)
3. FINDING-R3-02 (architectural decision — pick DI)
4. FINDING-R3-01 (method visibility decision)
5. FINDING-R3-05 (new ACs for climax_tono — most writing)

**Pre-existing open flags (still valid):**
- GDD #6 `combat_sequence_complete` signal must be added to EventBus (ADR-002) before CA-PN-001, CA-PN-008, CA-PN-019, CA-PN-025 can pass
- `world_state.get_nested(path)` Open Question in GDD #4 must be resolved before F1-ext ACs can be written or implemented
- Pre-VS gate: CA-PN-001 through CA-PN-012 must pass before Vertical Slice narrative content is authored

Related: [[project-gdd15-acs]], [[feedback-test-infrastructure]], [[feedback-pre-vs-gate]]
