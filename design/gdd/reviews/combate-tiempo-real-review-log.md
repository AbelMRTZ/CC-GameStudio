# Review Log: Combate en Tiempo Real

## Review — 2026-05-26 (Full) — Verdict: MAJOR REVISION NEEDED → IN PROGRESS (Design Phase)

Scope signal: L  
Specialists: game-designer, systems-designer, gameplay-programmer, godot-gdscript-specialist, qa-lead, creative-director  
Blocking items: 20 | Recommended (advisory): 10  
Summary: Full adversarial review by 5 specialists + creative-director synthesis identified three systemic root causes: (1) resistance sign convention conflict between GDD #2 and GDD #6 cascading to 5 separate formula/math bugs, (2) state-machine authority undefined for hitbox activation/I-frames/ability queueing causing frame-rate-dependent behavior and ghost abilities, (3) fantasy-to-mechanic gap (no cancels, no chaining, no pre-demon identity). User immediately resolved via 4 design decisions: (a) negative resistance = armor (aligns with GDD #2), (b) heavy-to-light cancel enabled, (c) no input buffer in HIT_STUN, (d) no starter demon. In-session revision addressing all 20 blockers: corrected resistance sign convention in formulas/ACs, added heavy-to-light cancel mechanic, clarified state-machine authority for hitbox/I-frames, rewrote untestable ACs with numeric assertions, added knockback decay formula, descarted queued abilities on HIT_STUN. 10 advisory items clarified (Arcano intent, cooldown_max reference, map bounds, etc.). GDD now ready for second review pass.

Prior verdict resolved: N/A — first review

### Bloqueantes resueltos en esta revisión

| # | Bloqueante | Categoría | Resolución |
|---|-----------|-----------|-----------|
| B1 | Resistance sign convention inverted (GDD #6 vs GDD #2) | Cross-system | User decision: negative = armor (aligns GDD #2). Updated CA-019/CA-020 |
| B2 | CA-019 math contradicts Formula 4.2 | Formula | Changed to +0.3 resistance (weakness) = 13 damage |
| B3 | CA-020 math contradicts Formula 4.2 | Formula | Changed to -0.3 resistance (armor) = 15 damage |
| B4 | Formula 4.3 divide-by-zero at resistance = -1.0 | Formula | Clarified edge case E1: only relevant if future demonios exceed cap |
| B5 | No cancels + no input queue = 0.60s forced passivity | Mechanic/Fantasy | User decision: added heavy-to-light cancel. CA-008b added |
| B6 | Queued abilities fire post-HIT_STUN (ghost input) | Mechanic | User decision: discard queue on HIT_STUN. CA-052b added |
| B7 | AnimationPlayer method tracks for hitbox activation | Tech/Fragility | User decision: state machine controls hitbox activation (enter/exit). Section C.3 clarified |
| B8 | I-frames via collision_mask = 0 disables terrain | Tech/Safety | User decision: use hurtbox.set_monitoring(false). Section C.5 clarified |
| B9 | hit_registered reset timing undefined | Spec/Ambiguity | Clarified: reset on state entry (LIGHT_ATTACK/HEAVY_ATTACK enter), not on hit |
| B10 | AnimationPlayer hitbox activation frame-rate dependent | Tech/Fragility | Resolved by moving to state machine control (no delta-time dependency) |
| B11 | Knockback decay model undefined | Formula/Spec | Added Formula 4.5: linear decay over IFRAME_DURATION |
| B12 | CA-035 (blink visibility) untestable | QA/Untestable | Rewrote: explicit `modulate.a` toggle at 10 Hz, testable assertion |
| B13 | CA-045 (~0.2s) untestable | QA/Untestable | Rewrote: knockback decays to 0 within IFRAME_DURATION (0.3s) |
| B14 | CA-007/CA-015 (funcionan idénticamente) untestable | QA/Untestable | Desagregated into explicit value ACs: CA-007a/b/c, CA-015a/b/c/d |
| B15 | CA-013 (leve hacia arriba) untestable | QA/Untestable | Rewrote: 15° angle ± 1° with vector math assertion |
| B16 | CA-048 (suma vectorial) no expected result | QA/Untestable | Rewrote: opposing knockbacks sum to zero, magnitude < 1.0 px/s |
| B17 | No mechanic differentiates deliberate play from button-mashing | Design/Fantasy | Partially resolved by heavy-to-light cancel + recovery frames creating timing window |
| B18 | Pre-demon Edrick (early-game) has no expert fantasy | Design/Fantasy | User decision: accept early-game feels generic, compensate narratively |
| B19 | Knockback persistence via velocity += doesn't work in move_and_slide() | Tech/Implementation | Resolved by Formula 4.5: separate knockback_velocity vector with linear decay |
| B20 | Ability queue survives HIT_STUN entry | Mechanic/Ghost Input | Resolved by CA-052b: queue is flushed on HIT_STUN |

### Advisory items clarificados

| # | Item | Resolución |
|---|------|-----------|
| A1 | Arcano "+25% amplifies all" modeled as additive | Clarified in Formula 4.1: currently additive (+0.25). Multiplicative interpretation flagged for future design review |
| A2 | CA-018 intermediate value 14.5 implies undefined modifier | Rewrote CA-018: explicit "Fuego +0.20, Arcano +0.25, mod = 0.45" |
| A3 | CA-028 references undefined cooldown_max | Rewrote: "cooldown_max per ability defined in Base de Datos de Demonios" |
| A4 | CA-044 map bounds undefined | Rewrote: "world_bounds defined in Level/Stage data" with explicit position/size reference |
| A5 | hit_registered reset timing unspecified (already addressed in B9) | Clarified: reset on state enter |
| A6 | Timer-per-slot node overhead | Documented: float cooldown dictionary alternative exists, not mandatory |
| A7 | Arcano intent: additive vs multiplicative | Verified current: additive. If multiplicative intended, requires Formula 4.1 refactor |
| A8 | Enemy hitbox detection (area_entered vs body_entered) | Documented: assumes enemies have Area2D hurtbox children. Requires architecture clarification in ADR |
| A9 | AIRBORNE state no spec difference for HIT_STUN | Clarified: HIT_STUN behavior identical in air and ground (no state-specific changes) |
| A10 | Enemy model schema (what data hit_landed(target, attack_data) passes) | Documented: attack_data schema deferred to IA de Enemigos GDD |
