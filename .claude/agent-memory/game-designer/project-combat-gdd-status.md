---
name: project-combat-gdd-status
description: Status and known open issues with GDD #6 Combate en Tiempo Real as of first adversarial review
metadata:
  type: project
---

GDD #6 (combate-tiempo-real.md) passed first adversarial design review on 2026-05-26. Verdict: NEEDS REVISION.

**Key blocking issues identified:**
1. Resistance sign convention conflict: Salud/Daño formula uses `(1 + resistencia_defensiva)` and states "resist -40% = value -0.4". Combat GDD E1 example applies it as (1 + 0.5) = ×1.5 for a +0.5 resistance, which matches — but edge case E1 text says "+0.5 resistance → round(22 × 1 × 1.5) = 33", meaning a *higher* resistance value produces *more* damage. This is internally contradictory with the stated intent of resistance reducing damage.
2. No input queueing during HIT_STUN + no cancels = the player is fully passive for 0.30s+ per hit with zero recourse. This conflicts with the "calculated and deadly" fantasy.
3. Demon ability queuing (fires after recovery, not during) makes the "encadenar físico + demonio" fantasy feel disconnected rather than fluid.
4. No strategic decision layer in base combat — no resource management, no positioning incentive, no counter mechanic. Button-mash risk the GDD explicitly names but does not mitigate.
5. Early game (no demons equipped) delivers zero of the stated fantasy — Edrick feels like a generic melee fighter, not "a weapon that sharpens itself."

**Why:** First adversarial review pass, no prior review log entry.
**How to apply:** When the user asks to revise combat GDD, prioritize these five items. Items 1 and 2 are BLOCKING for implementation.
