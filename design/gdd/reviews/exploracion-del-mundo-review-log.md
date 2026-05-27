# Review Log: GDD #8 — Exploración del Mundo

---

## Review — 2026-05-27 — Verdict: **APROBADO**

**Scope Signal:** M (moderate complexity, 1 formula, 6 dependencies, single new ADR likely)

**Specialists Consulted:** game-designer, systems-designer, ux-designer, qa-lead, narrative-director, creative-director

**Blocking Items Resolved:** 5
- WHISPER_TIMEOUT default to corruption (CRITICAL) → Convertido a interrupción gradual
- Corruption_floor delta unspecified (CRITICAL) → Fórmula de corrupción proporcional agregada
- Wayfinding contract missing (CRITICAL) → Minimap diegético + navegación implícita definida
- SECRET feedback (HIGH) → Audio+visual feedback agregado
- POI priority edge cases (HIGH) → Mecánica de single-use trigger resuelve ambigüedad

**Recommended Revisions Implemented:** 8
- Reescrito R5B (LORE_WHISPER) — Nueva mecánica de interrupción gradual
- Reescrito Formulas — Fórmula de corrupción proporcional, eliminados timeout/min_duration
- Actualizado Edge Cases — Reflejar nueva mecánica
- Actualizado Tuning Knobs — Removidos parámetros obsoletos
- Reescrito AC-EXPLO-11 a AC-EXPLO-16 — Nuevas ACs para interrupción gradual
- Actualizado AC-EXPLO-20 a AC-EXPLO-21 — Agregado SECRET feedback
- Cerrados 5 Open Questions — Resueltos por decisiones de diseño
- Agregada sección UI Requirements #6 — Minimap diegético

**Prior Verdict Resolved:** No prior review — First complete review

**Key Design Decisions Made:**
1. WHISPER INTERRUPTION GRADUAL: Corrupción proporcional a % de audio escuchado (0% = ~0 corrupción, 100% = delta máximo). Recompensa resistencia temprana, penaliza escucha prolongada. Narrativamente coherente con "elección moral consciente."
2. WHISPER SINGLE-USE: Trigger se desactiva tras primer uso (cualquier interrupción). No se repite en futuras visitas.
3. SECRET DISCOVERY FEEDBACK: Audio+visual simultáneo (sonido ambiental bajo + cambio visual) comunica que algo ocurrió sin "celebración" excesiva.
4. MINIMAP DIEGÉTICO: Pergamino accesible con tecla [M], no permanente en pantalla. MVP intencional poco-intuitivo. Concepto futuro de demonio "Visión Mejorada" (Vertical Slice+) permitirá ver rutas exactas.
5. NAVIGATION IMPLÍCITA PRIMARIA: Minimap es fallback. Navegación primaria sigue siendo iluminación + composición focal.

**Theme of Issues Resolved:**
La GDD original especificaba targets creativos ("atmospheric," "free but with purpose," "silent discovery") sin especificar las mecánicas que los producen. Revision enfocó en traducir targets en ACs verificables + decisiones narrativamente coherentes. Corrupción dejó de ser un castigo oculto para convertirse en una elección consciente del jugador.

**Next Steps:**
- [ ] Run `/ux-design` para minimap visual spec + button layout para susurro interruption
- [ ] Coordinate con GDD #4 (Estado del Mundo) para agregar handler `susurro_interrumpido(susurro_id, pct_listened)`
- [ ] Coordinate con GDD #22 (Seguimiento Moral) para definir `whisper_base_delta` en tabla de corruption formulas
- [ ] Level design puede comenzar con wayfinding contract + SECRET placement rules (ver GDD #8 Edge Cases)
- [ ] Audio director puede escalar susurro content (2-5 segundos, 2-4 palabras/segundo español)

---

## Summary

GDD #8 ahora representa un sistema de exploración donde:
- La corrupción es una *elección narrativa consciente*, no una trampa automática
- El jugador es recompensado por resistencia temprana (menos corrupción)
- El jugador es penalizado por escucha prolongada (más corrupción) pero de forma narrativamente significativa
- Los secretos comunican su descubrimiento sin romper atmósfera
- La navegación es implícita por defecto, con minimap diegético como fallback

La design vision del juego (cinematic narrative, moral descent, beautiful exploration) ahora está protegida por mecánicas claras que la producen, no solo por aspiraciones.

