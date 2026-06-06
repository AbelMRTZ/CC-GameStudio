# Review Log — HUD de Combate (GDD #18)

## Review — 2026-06-05 — Verdict: NEEDS REVISION
Scope signal: L
Specialists: ux-designer, ui-programmer, game-designer, systems-designer, gameplay-programmer, audio-director, qa-lead, creative-director
Blocking items: 10 | Recommended: 11
Summary: GDD sólido en arquitectura y fórmulas, pero con voids de contrato de datos críticos (HP_MAX, cooldown_max, combat_started/combat_ended sin registrar en GDD #6, swap_cooldown_updated sin registrar en GDD #10) y señales huérfanas (hit_stun_started/i_frames_active). Umbral Alert al 50% normaliza Pilar 5 — bajado a 40%. Creative Director endorsó no-ghost-bar y no-numbers como decisiones de pilar correctas.
Prior verdict resolved: No — primera revisión

### Acciones Cross-GDD pendientes antes de re-review
- X1: GDD #6 debe registrar `combat_started` / `combat_ended` en EventBus
- X2: GDD #10 debe extender `loadout_changed` con `slot_cooldowns` y `hp_max`
- X3: GDD #10 debe emitir `swap_cooldown_updated(remaining: float)`
- X4: GDD #5 debe añadir bus `SFX_HUD` y actualizar CA-001
- X5: GDD #3 debe registrar `cooldown_max` por demonio con floor ≥ 0.1s
