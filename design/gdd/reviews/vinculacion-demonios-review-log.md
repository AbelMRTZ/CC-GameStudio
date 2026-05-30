# Review Log: Vinculación de Demonios (GDD #13)

---

## Review — 2026-05-29 — Verdict: MAJOR REVISION NEEDED

Scope signal: L
Specialists: game-designer, systems-designer, gameplay-programmer, audio-director, qa-lead, godot-specialist, creative-director
Blocking items: 11 | Recommended: 10
Summary: El GDD es estructuralmente completo (8/8 secciones, 3 fórmulas, 33 CAs) pero falla en entregar la fantasía de jugador que tiene asignada en Pilares 2 y 5. La secuencia estándar de 2.3s (partículas→aura→silencio) tiene gramática visual de adquisición, no de carga — contradicción interna entre la Section B (fantasía) y la Section C (mecánica). La convergencia de partículas no es alcanzable con GPUParticles2D/CPUParticles2D estándar en Godot 2D. El bug del Gato (P-VD-04) es una re-emisión de señal garantizada que dispara todos los listeners downstream dos veces. El timeout de secuencias custom está clasificado como advisory cuando es un freeze permanente de juego. GDD #5 (Audio) no tiene contrato de Binding en absoluto. El binding de E4 (portador simultáneo) puede destruir permanentemente ~17% del espacio de build en MVP sin mitgación estructural.
Prior verdict resolved: No — primera revisión

---

## Revisión — 2026-05-29 — Veredicto: APROBADO (v3)

Scope signal: L
Specialists: game-designer, systems-designer, gameplay-programmer, audio-director, godot-gdscript-specialist, qa-lead, creative-director
Blocking items resueltos: 4 (B1–B4) | Recommended resueltos: 8 (R1–R8)
Summary: Tercera revisión en la misma sesión. Cuatro bloqueantes resueltos: B1 — `tween_all_completed` (API inexistente) reemplazado por Tween paralelo único con `set_parallel(true)` e `impact_tween.finished`; B2 — BindingSystem declarado como Autoload, `_edrick_alive` reset mediante `edrick_respawned()` de GDD #2 (GDD #2 debe añadir esta señal); B3 — referencia a Edrick via `get_tree().get_first_node_in_group("player")` con null guard y abort limpio; B4 (creativo) — partículas rediseñadas como secuencia bidireccional (explosión radial OUTBURST_TIME=1.0s + reversión abrupta IMPACT_TIME=0.5s). BINDING_DURATION permanece 2.6s. Añadidos OUTBURST_RADIUS=150px, TENSION_ANIM_TIME=0.3s. Registry entities.yaml sincronizado: PARTICLE_TRAVEL_TIME y PARTICLE_MIN_SPEED deprecados, OUTBURST_TIME/IMPACT_TIME/OUTBURST_RADIUS/TENSION_ANIM_TIME añadidos, BINDING_DURATION corregido de 2.3s a 2.6s. Añadidos AC-VD-036 (reset de _edrick_alive), AC-VD-037 (deduplicación en Registro), AC-VD-038 (validación de fantasy en playtest). Total ACs: 37 (33 bloqueantes, 4 advisory). GDD listo para aprobación sin re-review completo.
Prior verdict resolved: Sí — segunda revisión (v2) resolvió 13 bloqueantes; esta sesión resuelve 4 bloqueantes adicionales encontrados en tercer re-review y 8 revisiones recomendadas

---

## Revisión — 2026-05-29 — Veredicto: REVISIÓN MAYOR APLICADA (pendiente re-review)

Scope signal: L
Specialists: game-designer, systems-designer, gameplay-programmer, audio-director, qa-lead, godot-specialist, creative-director
Blocking items resueltos: 13 | Recommended: 0
Summary: Segunda revisión completa en la misma sesión. Se resolvieron 13 bloqueantes: renombrado `bound_demons` → `available_demons` (campo canónico de GDD #4); añadido `_edrick_alive` como estado local en BindingSystem (GDD #2 emite `edrick_died()`); corregido timing de aura/tensión de timer fijo a callback `tween_all_completed`; añadido `TENSION_ANIM_TIME` (0.3s) a F-VD-02 — duración total 2.3s → 2.6s; E4 rediseñado como PROHIBICIÓN DE DISEÑO; E2 reemplaza "imposibilidad estructural" con mecanismo real (`_sequence_active` flag); P-VD-02 resuelto con contrato `{ demon_id, edrick_position, portador_position }`; `queue_free()` añadido en ambas rutas de escenas custom; AudioManager añadido a `process_mode = ALWAYS`; silencio del Gato documentado como diseño deliberado; §2 Fantasy reformulada como inmovilización involuntaria. GDD tiene 34 CAs (31 bloqueantes, 3 advisory). Requiere re-review completo antes de Aprobado.
Prior verdict resolved: Parcial — primera revisión resolvió 11 bloqueantes previos; esta sesión resuelve 13 adicionales encontrados en segunda pasada
