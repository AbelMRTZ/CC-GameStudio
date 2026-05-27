---
name: project-gdd15-acs
description: 15 Acceptance Criteria authored for GDD #15 (NPC & Dialogue); test types, gate levels, and open flags documented
metadata:
  type: project
---

GDD #15 Sistema de NPC y Diálogo — Acceptance Criteria authored on 2026-05-27.

15 ACs produced (CA-NPC-001 through CA-NPC-015):
- 8 Unit tests (blocking) — branch selection logic, F1/F2 formulas, anti-farming rule
- 5 Integration tests (mix blocking/advisory) — activation, input restore, moral consequences, forced interruption, simultaneous interactions
- 2 Integration tests (advisory) — NPC dead scene, simultaneous Areas2D

**Why:** Branch selection (R3) and reputation formulas (F1, F2) are Logic story type — automated tests are BLOCKING gate per QA standards.

**How to apply:** When implementation of GDD #15 begins, verify that unit test files exist in `tests/unit/npc_dialogue/` before marking any Logic stories Done. Integration tests live in `tests/integration/npc_dialogue/`.

Open flags carried forward:
- E4 (pause during dialogue): untestable until GDD #21 (Pause System) is designed — move AC to GDD #21
- E6/E7 (empty branch, NPC with no branches): data validation smoke check, not logic AC
- R6 delta integration: add smoke AC once first real NPC content is implemented
- Pre-VS gate: CA-NPC-004 to CA-NPC-010 must pass before Kingdom 2 narrative content is authored

Related: [[feedback-pre-vs-gate]], [[feedback-test-infrastructure]]
