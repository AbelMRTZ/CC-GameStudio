# Review Log: Loadout & Build Management (GDD #10)

## Review — 2026-05-28 (re-revisión 2) — Veredicto: APROBADO

Scope signal: L
Especialistas: game-designer, systems-designer, qa-lead, ux-designer, gameplay-programmer, creative-director
Blocking items: 6 | Recommended: 8
Resumen: Re-revisión disparada por GDD #11 (Motor de Sinergias), que extendió `loadout_changed` a `(equipped_demons, gato_available: bool)` y asignó a Loadout la responsabilidad de re-emitir en cambios de estado narrativo del Gato sin swap. Los 6 blockers eran exclusivamente alineación de contrato cross-doc: firma de señal actualizada en Rule 14/16/§3.3/§6; nueva §3.1.F con Rules 22-23 (monitoreo Gato vía método directo `notify_gato_available`); CA-029 dividida en 029a+029b; `Engine.time_scale` → `get_tree().paused` en CA-038/039; Rule 12 referenciada a `HealthSystem.recalculate_hp_on_max_change()` (GDD #2 §3.5.1); CA-034 tolerancia 5.75s→5.70s. Creative-director confirmó que la pausa-en-combate es tradeoff válido para el género, no un defecto de diseño.
Prior verdict resolved: Sí — APROBADO (re-revisión anterior 2026-05-28), re-revisión disparada por extensión de señal en GDD #11.

## Review — 2026-05-28 (re-revisión) — Veredicto: APROBADO

Scope signal: L
Especialistas: game-designer, systems-designer, qa-lead, ux-designer, gameplay-programmer, creative-director
Blockers resueltos: 9 | Recomendados: 10 | ACs nuevas: 3 (CA-038/039/040)
Resumen: La re-revisión encontró 9 nuevos blockers de naturaleza diferente a la primera revisión: defectos de contrato de datos cross-doc (schema hp_bonus, tipo="stat_reduction" vs stat_change), ambigüedades de máquina de estados (SWAP_ANIM previous_state guard, notificación Rule 14 vs 16), gap de infraestructura Godot (pause-mode node exemptions), cache sin path de remoción (available_demons), y ACs auto-contradictorios con la sección de fórmulas. Todos resueltos en sesión. El creative-director evaluó los concerns de diseño del game-designer (espacio combinatorial pequeño en MVP, moral cost inerte) como tradeoffs válidos de alcance, no defectos de GDD. Veredicto: APROBADO.
Prior verdict resolved: Sí — MAJOR REVISION NEEDED (2026-05-28, 9 blockers, todos resueltos en misma sesión de diseño)

## Review — 2026-05-28 — Veredicto: MAJOR REVISION NEEDED → revisado en sesión

Scope signal: L
Especialistas: game-designer, systems-designer, qa-lead, ux-designer, gameplay-programmer, creative-director
Bloqueantes: 9 | Recomendados: 9
Resumen: El GDD tenía 8/8 secciones presentes y especificaciones de timing exhaustivas, pero el modelo de acceso a la Build Screen (pausa vs. tiempo real) estaba completamente indefinido — toda la arquitectura de timing dependía de esa respuesta. El rango de HP reclamado (65-95) era aritméticamente incorrecto (real: 70-90). El comportamiento de reducción de HP no tenía spec de señal ni feedback visual. Estado del Mundo (#4) faltaba en el header como dependencia dura. Tres ACs estaban ausentes (cooldown on exit, Build Screen durante SWAP_ANIM, swap parcial no perturba slot adyacente). Los 9 bloqueantes fueron resueltos en la misma sesión; el GDD está pendiente de re-revisión para confirmar aprobación.
Prior verdict resolved: N/A — primera revisión
