---
name: narrative-arch-issues
description: Adversarial review findings for GDD #4 (Estado del Mundo) — 8 structural narrative problems identified 2026-05-26
metadata:
  type: project
---

Adversarial review of GDD #4 (Estado del Mundo) completed 2026-05-26.

Key problems found:
1. Cat/brother twist has NO dedicated state field — cat_reveal event key exists in examples only, not in schema definition
2. "gato" in available_demons implies player can unequip Gato — contradicts narrative permanence
3. NPC permadeath with no agency creates narrative content loss through bad luck (violates Pilar 3)
4. player_choices schema only captures binary explicit decisions — atmospheric/behavioral narrative texture is untracked
5. reputation and corruption_level are fully independent — an NPC can stay "Aliado" while Edrick becomes monstrous
6. corruption_floor at 0.06 (3 executions) still maps to "Íntegro" visual state — "memory matters" claim is mechanically hollow at low floors
7. major_events tracks visible state only — no distinction between visible narrative state and invisible moral tracking
8. The system permits a "pure Edrick" run (corruption stays ~0.1) but the story has no coherent path for this

**Why:** These were found during adversarial specialist review commissioned by the user.
**How to apply:** These issues inform design of GDD #15 (NPC y Diálogo), GDD #16 (Progresión Narrativa), GDD #22 (Seguimiento Moral), and the cat's eventual GDD.
