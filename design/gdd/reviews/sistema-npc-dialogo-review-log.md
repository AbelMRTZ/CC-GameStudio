# Review Log — Sistema de NPC y Diálogo (#15)

---

## Review — 2026-05-27 — Veredicto: MAJOR REVISION NEEDED → Revisiones Aplicadas

Scope signal: L
Especialistas: narrative-director, game-designer, systems-designer, ux-designer, qa-lead, creative-director
Bloqueantes: 7 (diseño) + 5 (ACs) = 12 | Recomendados: 4
Prior verdict: Primera revisión

**Resumen (creative-director):** El GDD tenía intenciones narrativas claras y estructura completa (8/8 secciones), pero los mecanismos concretos traicionaban el Pilar 5. F2 bloqueaba mecánicamente las ramas de redención para Edrick muy corrupto (ludonarrative dissonance). El anti-farming combinado con ramas finitas creaba un plateau de reputación que hacía la redención imposible post-plateau. La dominant strategy obvia (farmear karma preventivo) vaciaba la tensión moral central. Adicionalmente: input buffer ausente en [E], color como única señal moral (inaccesible para daltónicos), ambigüedad corruption_floor vs corruption_level, orden R3/F2 no especificado, y 5 ACs no testeables en GUT.

**Revisiones aplicadas en la misma sesión (2026-05-27):**

| Bloqueante | Fix |
|---|---|
| F2 bloquea redención | Eliminado `max_corruption` — F2 solo filtra por `min_reputation` |
| Anti-farming = plateau | Mantenido anti-farming para ramas regulares; añadidas ramas `reconciliation: true` que siempre aplican delta |
| Ambigüedad `corruption_floor` vs `corruption_level` | Nota aclaratoria en R5; eliminada referencia a `corruption_level` en lecturas |
| Orden R3 vs F2 | Paso previo explícito en R3: F2 filtra primero, luego R3 selecciona del pool |
| Input buffer [E] | Añadido en R2 + Tuning Knob `DIALOGUE_ACTIVATION_BUFFER_FRAMES=3` |
| Color única señal moral | Símbolo prefijo en UI3: `★` noble, `✖` oscura |
| CA-NPC-003 no determinístico | Reescrito como observable síncrono |
| CA-NPC-006/007 sin campo verificable | Referencia a `branch.type` field añadida |
| F1 clamp inferior ausente | Añadido CA-NPC-009b |
| E2 sin AC | Añadido CA-NPC-012 (DEFAULT cuando pool vacío) |
| CA-NPC-014 múltiple THEN | Dividido en CA-NPC-014a, 014b, 014c, 014d |
| E6/E7/DONE→INACTIVE sin AC | Añadidos CA-NPC-016, CA-NPC-017, CA-NPC-018 |

ACs: 15 originales → 22 tras revisión.
Estado: Pendiente re-review en nueva sesión (`/design-review design/gdd/sistema-npc-dialogo.md`).

---

## Review — 2026-05-27 — Veredicto: APROBADO

Scope signal: L
Especialistas: narrative-director, game-designer, systems-designer, ux-designer, qa-lead, creative-director
Bloqueantes: 13 nuevos encontrados, 13 resueltos en la misma sesión | Recomendados: 12 (advisory, documentados en GDD)
Prior verdict: MAJOR REVISION NEEDED (2026-05-27, primera sesión) — resuelto

**Resumen (creative-director):** La segunda revisión encontró que ~7/12 bloqueantes anteriores estaban completamente resueltos y ~5 habían creado nuevas brechas. Los 13 nuevos bloqueantes tocaban tres áreas: (1) ramas narrativas fijas que anulaban la reputación acumulada del jugador (ludonarrative dissonance con Pilar 1); (2) garantías insuficientes para el arco de redención del Pilar 5 (trigger diferido a GDD #16 no escrito, mínimo de 1 rama insuficiente, pump infinito posible); (3) schema de ramas subespecificado y sin restricciones de consistencia type/min_rep. Adicionalmente: buffer de input dependiente del framerate, símbolos de accesibilidad en cuerpo del texto en lugar del header, foco inicial de teclado no definido en decisión moral, y ausencia de ACs para histéresis, reputación inicial, y ventana de abort. Todas las resoluciones acordadas con el usuario en sesión: variantes por tier para ramas fijas, mínimo 2 ramas de reconciliación con tipos de trigger definidos, F3 para reputación inicial por corrupción visible, histéresis `noble_streak`, señal `reputation_tier_changed` para feedback del espejo.

**Revisiones aplicadas (2026-05-27, re-review):**

| Bloqueante | Fix |
|---|---|
| Ramas fijas override reputación | Sub-variantes por tier; `reputation_aware: false` como excepción |
| Reconciliación insuficiente | Mínimo 2 ramas/NPC; tipos de trigger en GDD #15; `reconciliation_applied_this_arc: bool` |
| Demonios → reputación inicial | F3 añadida; Tuning Knobs CORRUPTION_THRESHOLD y PENALTY |
| Sin histéresis | `noble_streak: int`; regla NOBLE_STREAK_REQUIRED=2 |
| Sin feedback mecánico espejo | Señal `reputation_tier_changed(npc_id, old_tier, new_tier)` |
| Pump infinito reconciliación | `reconciliation_applied_this_arc: bool` |
| Branch schema | F4 añadida: schema completo, contenedores, tabla de consistencia, delta discreto |
| type/min_rep sin restricción | Tabla de consistencia + warning en log |
| Buffer framerate-dep. | `DIALOGUE_ACTIVATION_BUFFER_MS=150` (time-based desde fade-in) |
| Símbolos en texto | Movidos al área del header |
| Foco inicial teclado | Primera opción recibe foco; opción única requiere confirmación explícita |
| AC contenido 006/007 | CA-NPC-022 smoke check CI |
| E8 ventana sin AC | CA-NPC-023 |

ACs: 22 → 26 (añadidos CA-NPC-019 histéresis, CA-NPC-020 F3, CA-NPC-022 smoke check, CA-NPC-023 E8).
