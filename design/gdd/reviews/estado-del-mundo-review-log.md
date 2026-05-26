# Review Log: Estado del Mundo

## Review — 2026-05-26 (PM, Sesión 2) — Verdict: MAJOR REVISION NEEDED → **APROBADO**

Scope signal: L (4 bloqueantes técnicos, no requiere re-arquitectura)
Specialists: game-designer, systems-designer, qa-lead, narrative-director
Blocking items: 4 (técnicos) | Recommended: 2 (recomendados) | Deferred: 1 (creativo — GDD #16)

**Summary**: Re-review en 4 especialistas paralelo. GDD #4 está functionally sound — los 6 bugs de schema (FASE 1) y 5 problemas narrativos (FASE 2) fueron arreglados. Los 4 bloqueantes técnicos encontrados fueron correcciones menores de consistencia (gato en ejemplos, type mismatch, contradicción auto-desequipa, PASS condition faltante). 16 ACs untestables fueron remediadas (1 duplicado renumerado, 4 marcadas time-dependent, 5 movidas a GDDs dependientes, 12 PASS conditions). **Definición canónica de corrupción creada y unificada** — unanimidad confirmada entre GDD #3, #6, #4. Narrative-Director valida que puedes escribir GDD #15 AHORA. Game-Designer identifica 1 decisión creativa pendiente (§9.5 "Edrick Puro") que no bloquea este GDD, bloqueará GDD #16.

Prior verdict resolved: **Sí** — los 7 bloqueantes de PM sesión 1 fueron todos resueltos

### Arreglos aplicados en esta sesión (FASE 1-3)

**FASE 1 (Bugs de Esquema — 6):**
- ✅ corruption_floor consolidado en `narrative:` (no metadata)
- ✅ available_demons init corregido a `[]`
- ✅ cat_slot definido como parte de `companion_state`
- ✅ demon_saturation inicializado como `{}`
- ✅ corruption_floor ceiling clamp añadido (máx 1.0)
- ✅ Código de ejemplo §3.4 corregido

**FASE 2 (Problemas Narrativos — 5):**
- ✅ Contrato de companion_state (Vinculación GDD #13) explícitamente definido
- ✅ Gato movido a `companion_state` (fuera de `available_demons`)
- ✅ Trade-offs §9.3-9.5 documentados como "Design Theory Warnings / MVP Constraints"

**FASE 3 (ACs Untestables — 16):**
- ✅ CA-024 duplicado renumerado a CA-025 (cascada +1 para todas las ACs posteriores)
- ✅ 4 ACs time-dependent marcadas [blocked: clock-injection] (CA-025, CA-026, CA-027, CA-029)
- ✅ 5 ACs de integración marcadas [MOVE TO GDD #X] (CA-012, CA-023, CA-033, CA-045, CA-046, etc)
- ✅ 12 PASS conditions escritas/mejoradas

**Post-FASE 3 (Bloqueantes Técnicos — 4):**
- ✅ GAP-01: Gato removido de ejemplos §3.6
- ✅ GAP-02: `major_events == true` → `>= 1` en §4.5
- ✅ GAP-04: Auto-desequipa clarificado (restricción narrativa SÍ, obtención nueva NO)
- ✅ QA-D1: CA-028 PASS condition añadido

**Extra (Definición Canónica):**
- ✅ Tabla de DEFINICIÓN CANÓNICA DE CORRUPCIÓN creada (apunta a GDD #3, #6, #4)
- ✅ Unanimidad confirmada entre GDDs respecto a qué es y qué implica corrupción
- ✅ Usuario aprobó definición unificada

### Especialistas que validaron los arreglos

| Especialista | Veredicto | Key Finding |
|---|---|---|
| **Game-Designer** | NEEDS REVISION → APROBADO | 4 bloqueantes técnicos + 1 decisión creativa pendiente (§9.5, no bloquea este GDD) |
| **Systems-Designer** | 2 bugs mayores + 3 menores encontrados, 4 arreglados | Unanimidad en corrupción confirmada. demon_saturation key guard es recomendado pero no bloqueante (implementación defensiva) |
| **QA-Lead** | 38 ACs testeables, 4 bloqueadas, 9 movidas | 4 bloqueantes técnicos arreglados. Coverage de schema es suficiente para MVP |
| **Narrative-Director** | **Listo para escribir narrativa AHORA** | Puedes empezar GDD #15. §9.5 es blocker solo para GDD #16, no para GDD #4 |

---

## Review — 2026-05-25 (PM) — Verdict: MAJOR REVISION NEEDED → IN PROGRESS (Design Phase)

Scope signal: L (after deferring cat)
Specialists: systems-designer, game-designer, narrative-director, qa-lead, creative-director (senior)
Blocking items: 7 | Recommended: 7
Summary: Re-review después de 24h reveló capa más profunda de inconsistencias. Aunque B10 anterior cerró ACs, la coherencia sistémica estaba rota: el Gato era el nodo Foundation más crítico pero sin mecánicas definidas; claves de eventos duplicadas (gato_desaparecido vs cat_reveal); corrupción irreversible contradecía pilar de "libertad narrativa"; NPCs con inicialización ambigua. Tres decisiones de diseño tomadas: (1) Corrupción reversible (arco moral gris), (2) El Gato deferir a GDD posterior (Foundation simplificada), (3) NPCs pre-poblados (seguro para queries). Revisión editorial de 2-3 horas aplicada para resolver 7 bloqueantes mediante cambios mecánicos/editoriales. Re-review en sesión fresca programado.
Prior verdict resolved: Parcialmente — B10 cerró ACs pero no la coherencia sistémica

### Bloqueantes nuevos encontrados + resueltos en esta revisión

| # | Bloqueante | Resolución |
|---|-----------|-----------|
| N1 | Claves evento duplicadas (gato_desaparecido vs cat_reveal) | Remover El Gato completamente de Foundation → simplifica schema |
| N2 | Schema inconsistente (cat_demon vs cat_slot, tipos violados, ejemplo código wrong) | Revisión editorial completa: campo correcto, ejemplo actualizado, tipos validados |
| N3 | El Gato sin mecánicas definidas → imposible implementar | Decision: Deferir El Gato a GDD posterior (Vinculación #13 o Narrativa #16) |
| N4 | Corrupción irreversible contradice "libertad narrativa" pilar | Decision: Corrupción reversible (arco moral gris con acciones redentoras) |
| N5 | NPC initialization ambigua (pre-poblados vs lazy-load) | Decision: Pre-poblados en init (seguro para queries narrativas downstream) |
| N6 | Untestable ACs (CA-022 GDScript exception fake, CA-018 sin PASS condition) | Reescribir 5 ACs problemáticas con criterios concretos testables en GUT |
| N7 | Persistence triggers sin test coverage (3.3b define 4 triggers, cero ACs) | Adición de ACs para cada trigger en sección 8.8 |

---

## Review — 2026-05-25 (AM) — Verdict: MAJOR REVISION NEEDED → REVISED

Scope signal: L
Specialists: systems-designer, game-designer, narrative-director, qa-lead, creative-director (senior)
Blocking items: 10 | Recommended: 2
Summary: El GDD tenía instintos arquitectónicos sólidos (separación estado efímero/persistente, cat_slot narrativo, distinción corrupción/saturación) pero fallaba como especificación Foundation porque definía schema sin definir contratos. Los bloqueantes críticos fueron el timing de escritura indefinido (B7), el contrato de eventos significativos ausente (B6), el schema de major_events incapaz de modelar eventos multifase (B8), y 15 ACs no testeables independientemente. Los 10 bloqueantes fueron resueltos en la misma sesión.
Prior verdict resolved: N/A — primera revisión

### Bloqueantes resueltos en esta revisión

| # | Bloqueante | Resolución |
|---|-----------|-----------|
| B1 | Fórmula saturación — error de unidades (seg vs min) | Añadido `/ 60.0` en fórmula 4.2 + ejemplo de verificación |
| B2 | Division por zero en exploración | Guard `if tiles_totales_area == 0: return 0.0` en fórmula 4.3 |
| B3 | Orden editorial JSON no implementable | Campo `priority: int` obligatorio en data schema de branches + sort en fórmula 4.5 |
| B4 | `alive: bool` ausente del schema canónico | Añadido a `npc_encounters` en sección 3.1 |
| B5 | `player_choices` demasiado delgado | Schema extendido a `{value, act, timestamp, conscious}` |
| B6 | Contrato de eventos significativos ausente | Sección 3.5 añadida: 5 tipos fijos en `event_types.json` |
| B7 | Timing de escritura indefinido | Sección 3.3b añadida: esquema híbrido (tabla de triggers) |
| B8 | `major_events: dict[string, bool]` insuficiente | Migrado a `dict[string, int]` (phase counter) en todo el GDD |
| B9 | Sin enforcement en cat_slot | Sección 3.3c añadida: `narrative_set_cat_slot()` protegido + tabla de fases |
| B10 | 15 ACs no testeables | Reescritos CA-001, 002, 013, 014, 022, 028, 032, 033, 035, 037, 043, 046, 047, 049, 050 con criterios pass/fail concretos |
