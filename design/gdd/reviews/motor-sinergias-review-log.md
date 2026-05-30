# Review Log — GDD #11 Motor de Sinergias

---

## Review — 2026-05-28 — Verdict: NEEDS REVISION → REVISED IN SESSION

Scope signal: L
Specialists: game-designer, systems-designer, qa-lead, creative-director
Blocking items: 8 resolved | Recommended: 7 (open)
Summary: El Motor de Sinergias es un sistema técnicamente sólido con una arquitectura pull-API limpia. Los bloqueantes convergían en un único fallo de raíz: el contrato API entre el Motor y Combate no estaba dibujado, produciendo una contradicción en GDD #6 (synergy_multiplier ×1.15 vs 1.00) y 6 ACs que testaban al sistema equivocado. Adicionalmente: Arcano+Visión carecía de contenido de fórmula, el schema de synergies.json nunca fue definido, y la asimetría de F-SE-07 fue corregida (Arcano ahora amplifica AMBAS penalizaciones de Impulsividad). La revisión fue completada in-session con las 4 decisiones de diseño resueltas por el autor.
Prior verdict resolved: No — primera revisión; bloqueantes resueltos en la misma sesión.

### Decisiones de diseño tomadas en esta revisión

| Pregunta | Decisión |
|----------|----------|
| Arcano+Visión efecto mecánico | `hp_penalty_bonus = -1` (int aditivo — HP total Visión+Arcano = −6) |
| UI sinergias negativas | HUD notifica la primera vez ("Anulación Térmica activa") al confirmar loadout |
| F-SE-07 asimetría | Corregido — Arcano amplifica AMBAS penalizaciones (×1.25). Duración Mente+Fuego+Arcano = 3.75s |
| Gato deactivation timing | Instantáneo — sin periodo de gracia |

### Bloqueantes resueltos (8/8)

1. Motor↔Combate seam — GDD #6 Fórmula 4.1 corregida; F-SE-06 añade nota de alineación
2. synergies.json schema — Nueva §3.4 con formato completo y ejemplo funcional
3. Arcano+Visión sin contenido — F-SE-02 tabla añade fila con `hp_penalty_bonus = -1`
4. Regla de decisión Arcano / "magnitud" indefinida — Tabla de categorías en F-SE-02 + definición formal en F-SE-03
5. F-SE-07 asimetría sin rationale — Corregido a amplificar ambas penalizaciones
6. Negative synergy discoverability — §3.3 HUD requirement + CA-MSE-035 playtest AC
7. PQ-MSE-04 Gato timing — §5 documenta desactivación instantánea; PQ-MSE-04 resuelto parcialmente
8. Test gaps AC — CA-MSE-013b, CA-MSE-020 (3.75s), CA-MSE-030b, CA-MSE-032, CA-MSE-033, CA-MSE-034 añadidos

### Recomendaciones abiertas (7)

- REC-01: §2 Player Fantasy debería reconocer la degradación MVP (sin costo moral)
- REC-02: ACs de Combate dentro de §8 Motor deben migrarse a GDD #6 o tests de integración una vez el seam esté implementado
- REC-03: F-SE-07 precondition gap (IMPULSIVIDAD_COOLDOWN_ADD cuando mente_fuego inactivo) — menor
- REC-04: resistencia_base per-demon vs combined ambiguity near clamp — menor
- REC-05: velocity_multiplier_combined range documentation error — menor
- REC-06: CA-MSE-002/014/021/029/007 under-specified test setups — menor
- REC-07: PQ-MSE-03 (Arcano near-dominant) necesita deadline de balance, no solo "post-MVP"

---

## Review — 2026-05-28 — Verdict: MAJOR REVISION NEEDED

Scope signal: L
Specialists: game-designer, systems-designer, qa-lead, economy-designer, creative-director
Blocking items: 12 | Recommended: 12
Summary: Tres clusters BLOCKING independientes emergieron que la revisión previa no había resuelto: (1) Arcano near-dominance viola Pilar 4 — con 2 slots MVP y GDD #22 ausente, existe exactamente una clase de estrategia dominante, colapsando el espacio de descubrimiento de builds antes de que se abra; (2) dos bugs de correctitud en F-SE-07 — un gap de precondición causa una penalización de cooldown incorrecta cuando Arcano+Mente está equipado sin Fuego, y predecir_duration_final carece de guardia max(0,...) permitiendo valores de timer negativos; (3) 7 ACs testean el comportamiento de Combate (multiplicación de factores) en lugar de la responsabilidad propia del Motor (retornar valores de factor). Un cuarto bloqueante — las sinergias positivas no tienen bucle de feedback perceptible — golpea directamente la Player Fantasy ("jugador-arqueólogo"). El creative-director confirmó los cuatro clusters bloqueantes y dictaminó que se requiere acción de diseño sobre Arcano antes de la aprobación, no deuda diferida.
Prior verdict resolved: Sí — 8/8 bloqueantes previos confirmados resueltos en el documento; 7 recomendaciones previas permanecen abiertas (integradas en los 12 ítems recomendados actuales).

---

## Review — 2026-05-29 — Verdict: APPROVED
Scope signal: L
Specialists: game-designer, systems-designer, economy-designer, qa-lead, creative-director
Blocking items: 8 resolved | Recommended: 7 (open — no bloquean implementación)
Summary: Tercera revisión — todos los bloqueantes del MAJOR REVISION NEEDED previo resueltos en sesión. Cuatro decisiones de diseño tomadas: (1) Arcano recibe `hp_bonus: −3` como coste visible MVP sin depender de GDD #22 — tradeoff amplificación vs HP resuelto; (2) HUD notifica sinergias positivas primera vez por sesión (mismo mecanismo que negativas); (3) arcano_mente schema añade `impulsividad_cooldown_multiplier` e `impulsividad_duration_multiplier`; (4) §5 mente_gato ejemplo corregido. Bugs de fórmula F-SE-07 corregidos (precondición + max guard), sign error de corrupcion_dodge_penalty resuelto, 7 ACs reasignados correctamente al Motor.
Prior verdict resolved: Sí — los 4 clusters BLOCKING del MAJOR REVISION NEEDED (2026-05-28) resueltos.
