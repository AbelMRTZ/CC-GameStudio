---
name: project-gdd18-acs
description: 30 ACs authored for GDD #18 (HUD de Combate); 20 BLOCKING, 10 ADVISORY; all 4 formulas and 6 rules covered
metadata:
  type: project
---

GDD #18 HUD de Combate — Acceptance Criteria authored on 2026-06-04.

30 ACs produced (CA-HUD-001 through CA-HUD-030):
- 20 BLOCKING (Logic/Integration) — formula verification, signal-driven state transitions, hidden state processing
- 10 ADVISORY (Visual/Feel) — animation style, glow effects, color states

**Formula coverage:**
- F-HUD-01 (HP Fill Ratio): CA-HUD-001, CA-HUD-002, CA-HUD-030
- F-HUD-02 (HP Threshold Classifier, integer arithmetic): CA-HUD-003 through CA-HUD-006
- F-HUD-03 (Cooldown Sweep Angle): CA-HUD-010, CA-HUD-011, CA-HUD-012
- F-HUD-04 (Swap Overlay Ratio): CA-HUD-017, CA-HUD-018, CA-HUD-019, CA-HUD-020

**Rule coverage:**
- Regla 1 (HP Bar): CA-HUD-001 through CA-HUD-008
- Regla 2 (Demon Slots): CA-HUD-009 through CA-HUD-015
- Regla 3 (Swap Combat Indicator): CA-HUD-016 through CA-HUD-020
- Regla 4 (Save Indicator): CA-HUD-021, CA-HUD-022
- Regla 5 (Sanctuary Indicator): CA-HUD-023, CA-HUD-024, CA-HUD-025
- Regla 6 (Cinematic Visibility): CA-HUD-026, CA-HUD-027, CA-HUD-028

**Why:** HUD is a pure-display system — all BLOCKING ACs test signal-in → internal state transitions. F-HUD-02 boundary values (HP=18/19 at HP_MAX=75) use integer arithmetic and are the highest-risk failure mode. E7 (signal processing while HIDDEN) is a commonly missed integration case — CA-HUD-028 is its explicit test.

**How to apply:** When HUD implementation begins, unit tests for F-HUD-01 and F-HUD-02 can be isolated (pure math, no scene). F-HUD-03 and F-HUD-04 require signal injection via EventBus mock (same WorldStateMock injection pattern from [[feedback-test-infrastructure]]). CA-HUD-028 (hidden state processing) must be verified with a full scene integration test.

**Open flags:**
- CA-HUD-028 inline: HP=20, HP_MAX=75 → Alerta (not Crítico); fill = 0.267 — correct per F-HUD-02
- Regla 3 sourcing: `swap_cooldown_remaining` read via polling vs signal — ADR pending. CA-HUD-017/018 assume polling is readable; update if signal-driven
- `sanctuary_in_range` signal not yet in EventBus registry (GDD notes "candidato para Phase 5") — CA-HUD-023/024/025 are ADVISORY; do not block until signal is registered
- `cinematic_started` / `cinematic_ended` provisional (GDD #17 not designed) — CA-HUD-026/027/028 are BLOCKING but cannot be implemented until GDD #17 signals are registered

Related: [[project-gdd15-acs]], [[project-gdd16-acs]], [[feedback-test-infrastructure]]
