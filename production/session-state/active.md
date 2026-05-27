# Sesión Activa: design-review + REVISION GDD #9 Cámara — COMPLETO

**Tarea**: `/design-review design/gdd/camara.md` → Revisión adversarial + revision fixes aplicados
**Estado**: ✅ REVISIONS COMPLETE — Listo para re-review
**Archivo**: design/gdd/camara.md
**Modo de revisión**: full (4 especialistas + reescrituras iterativas)

---

## SESIÓN 1/4: Arquitectura + Player Fantasy (2026-05-27)

### Decisiones Arquitectónicas Cristalizadas

| Decisión | Opción | Rationale |
|----------|--------|-----------|
| Suavizado | (A) Lerp manual en _process | MVP necesita F3 dinámico. Predecible, debuggeable. Deshabilitar position_smoothing_enabled nativo |
| Player Fantasy | (B) Emergente, no dirigida | Follow basta para mayoría. Anchors en momentos CLAVE (5-8 por zona), no 80%+. Escalable para dev solitario |
| Señales globales | (A) EventBus Autoload | Desacoplado. Requiere nuevo ADR arquitectónico |
| MVP Scope | Reino 1 + narrativos | 4-6 semanas realista. Exploración + combate + algunos anchors cinematográficos |

### Secciones Reescritas (9 de 11)

✅ **Overview** — Especifica lerp manual, desactiva Camera2D.limit_*  
✅ **Player Fantasy** — "Composición emergente del movimiento natural" (no "cada plano dirigido")  
✅ **R2 — Follow Suave** — Especifica lerp manual con código Godot, elimina position_smoothing nativo  
✅ **R4 — Límites** — Límites locales, no asignados a Camera2D.limit_* (evita double-clamping)  
✅ **Interactions** — Añade EventBus, explicita GDD #6 como upstream  
✅ **Dependencies** — GDD #6 ahora declarado explícitamente  
✅ **Tuning Knobs** — Elimina `look_ahead_reset_time` fantasma, añade `lerp_delta_clamp_max`  
✅ **F5 — Posición Final** — Añade guardia zoom > 0.01, implementa centrado zona pequeña  
✅ **E1 — Pivote** — Actualiza para referirse a lerp manual  

❌ **Pendientes (Sesión 2)**:  
- F1, F2, F3 (refactor de semántica)
- E2–E7 (actualizar post-arquitectura)
- Rewrite completo de ACs

---

## SESIÓN 2/4 COMPLETADA — 2026-05-27 (Fórmulas + Acceptance Criteria)

### Secciones Reescritas en Sesión 2

✅ **F1 — Posición Objetivo** — Clarifica dir_input (discreto) vs velocity (continuo). Nota sobre intención.  
✅ **F2 — Look-Ahead Dinámico** — Especifica convergencia real (~0.57s al 95%, no ~0.3s). Tiempos documentados.  
✅ **F3 — Transición de Modo** — Clarifica continuidad al interrumpir (from = valor actual). Código GDScript.  
✅ **Acceptance Criteria** — Reescritos 10 ACs + 5 nuevos para edge cases. Total 15 ACs testables.

### Acceptance Criteria (Ahora 15 vs 10 originales)

**Testables (reescritos):**
- AC 1: Suavizado de Posición (delta ≤ 12 px/frame)
- AC 2: Look-Ahead en Exploración (0–50 px, 34 frames)
- AC 3: Look-Ahead en Combate (50→20 px en 0.4s) ✓
- AC 4: Pivote Suave (8–12 frames transición)
- AC 5: Límites de Zona ✓
- AC 6: Zona Pequeña se Centra ✓
- AC 7A: Entrada a Anchor (30 px, 1.1 zoom en 0.5s)
- AC 7B: Salida de Anchor (simetría confirmada)
- AC 8: Transición Zona sin Glitch (secuencia correcta)
- AC 9A: Cinemática Inicio/Fin (estados + interpolación)
- AC 9B: CINEMATIC bloqueado desde TRANSITION
- AC 9C: Estado post-cinemática = FOLLOW_EXPLORE
- AC 10: Input Responsivo (≤ 1 frame = 16.6ms)

**Nuevos para Edge Cases:**
- AC 11: Transición Zona Durante Combate (E3)
- AC 12: Dos Anchors Superpuestos (E4)
- AC 13: Dash 400 px/s (E5)
- AC 14: Zona Cargando Lentamente (E6)
- AC 15: Zoom Extremo + Offset Grande (E7)

### Problemas Bloqueantes (Sesión 1-2): Resueltos

| Problema | Status |
|----------|--------|
| 1. Suavizado nativo vs manual | ✅ DECIDIDO: lerp manual |
| 2. Double-clamping F5 + limit_* | ✅ DECIDIDO: desactivar limit_* |
| 3. zoom type float vs Vector2 | ✅ DOCUMENTADO en F4 |
| 4. Guardia división por cero F5 | ✅ IMPLEMENTADA |
| 5. Centrado zona pequeña F5 | ✅ IMPLEMENTADO |
| 6. Player Fantasy contradictoria | ✅ REDEFINIDA (emergente) |
| 7. `look_ahead_reset_time` fantasma | ✅ ELIMINADO |
| 8. F5 incompleta | ✅ COMPLETADA |
| 9. F1+F2 inconsistencia (against-wall) | ✅ REESCRITA F1 (dir_input ≠ velocity) |
| 10. F2 documentación falsa | ✅ REESCRITA F2 (0.57s real) |
| 11. GDD #6 ausente Dependencies | ✅ AÑADIDO explícitamente |
| 12. 7/10 ACs no testables | ✅ REESCRITOS 10, +5 nuevos |
| 13. 5 edge cases sin AC | ✅ CREADOS AC 11–15 |

### Problemas PENDIENTES para Sesión 3-4

- [ ] E2–E7 Edge Cases (actualizar descripciones post-arquitectura)
- [ ] Crear ADRs (Señales EventBus, Suavizado Manual)
- [ ] Open Questions (actualizar si aplica post-revisión)
- [ ] Verificar bidireccionalidad Dependencies (¿GDD #6, #8 mencionan cámara?)

---

## SESIÓN 3/4 COMPLETADA — 2026-05-27 (ADRs + Bidireccionalidad)

### A. Edge Cases (E2–E7) — VERIFICADAS ✅
E2–E7 ya están actualizadas en el archivo camara.md y reflejan arquitectura lerp manual + centrado zona pequeña en F5.
- E2 ✅: Zona pequeña → centrado automático (F5 con guardia `max_x < min_x`)
- E3 ✅: Transición zona durante combate → estado vuelve a FOLLOW_EXPLORE post-transición
- E4 ✅: Dos anchors superpuestos → último entrado es activo
- E5 ✅: Dash 400 px/s → look-ahead clampeado a máximo (speed_ratio)
- E6 ✅: Zona cargando lenta → cámara espera `zona_transition_ended`
- E7 ✅: Zoom extremo + offset grande → F5 centra si `max < min`

### B. ADRs Creados ✅
1. **ADR-001: Manual Camera Smoothing vs Native Position Smoothing** 
   - Archivo: docs/architecture/adr-001-manual-camera-smoothing.md
   - Decisión: lerp manual en _process
   - Rationale: permite F3 dinámico (smoothing_speed interpolable)
   - Implicaciones: DESACTIVAR `position_smoothing_enabled` en Camera2D node

2. **ADR-002: EventBus Autoload for Global Signal Distribution**
   - Archivo: docs/architecture/adr-002-eventbus-global-signals.md
   - Decisión: Autoload EventBus para signals globales
   - Señales iniciales: `combat_started`, `combat_ended`, `zona_transition_started`, `zona_transition_ended`
   - Implicaciones: requiere nuevo Autoload en proyecto + actualizar GDD #6

### C. Open Questions ✅
Las 3 open questions permanecen válidas (esperando clarificación de GDD #8, #17, #6):
1. ¿Parallax automático vs definido por zona?
2. ¿Cinemáticas pueden ignorar límites de zona?
3. ¿Feedback visual transición combate↔exploración?

### D. Bidireccionalidad Dependencies ✅
✅ **GDD #1 (Movimiento)** — menciona: "Cámara — Depende de la posición y velocidad de Edrick para seguimiento suave"
✅ **GDD #6 (Combate)** — ACTUALIZADO: añadido "Cámara (GDD #9) | Señales de transición de modo: combat_started y combat_ended"
✅ **GDD #8 (Exploración)** — menciona: "Cámara (#9) | Dura | zone_bounds: Rect2 para definir los límites dentro de los que la cámara puede moverse"

---

## Resumen de Progreso

| Sesión | Completado | Cambios |
|--------|-----------|---------|
| **1/4** | ✅ Arquitectura + Player Fantasy | 9 secciones reescritas |
| **2/4** | ✅ Fórmulas (F1, F2, F3) + ACs | 10 ACs reescritos + 5 nuevos |
| **3/4** | ✅ Edge Cases + ADRs + Dependencies | 2 ADRs + 1 GDD actualizado |
| **4/4** | ⏳ Final Review + Síntesis | `/design-review` (próximo paso) |

**Context Saved**: ~50% disponible para Sesión 4

---

## Secciones Completadas (2026-05-27 — SESIÓN ANTERIOR)
- [x] Overview — Cámara como ventana del mundo + infraestructura de modo
- [x] Player Fantasy — "El director que compone cada plano" — belleza dolorosa sin esfuerzo aparente
- [x] Detailed Design — 6 Core Rules + States/Transitions + Cross-system interactions
- [x] Formulas — 5 fórmulas completas (F1–F5) con tablas de variables y ejemplos
- [x] Edge Cases — 7 edge cases cubren cambios rápidos, zonas pequeñas, anchors, dash, cinemáticas
- [x] Dependencies — Depende de #1 (#1 Movimiento), #8 (#8 Exploración). Dependen: #17 Cinemáticas
- [x] Tuning Knobs — 10 parámetros ajustables con rangos seguros
- [x] Acceptance Criteria — 10 criterios testables (sigue suave, look-ahead, combate, límites, anchors, cinemáticas)
- [x] Visual/Audio Requirements — No VFX/audio específicos (infraestructura transparente)
- [x] UI Requirements — No UI específica
- [x] Open Questions — 3 preguntas pendientes de GDD #8, #17, #6

---

## Decisiones Clave Tomadas

### Framing de Overview
- Ambos: infraestructura técnica + impacto narrativo
- Referencia a que habilita Cinemáticas
- Control automático (usuario nunca toca la cámara)

### Player Fantasy
- **Opción elegida**: "El director que compone cada plano" (Candidato C)
- Belleza dolorosa sin esfuerzo aparente
- Usa regla de tercios, líneas de fuga, parallax
- Ancla en catedral con push-in imperceptible del 5%

### Detailed Design
- Follow con look-ahead horizontal semi-proporcional (50 px exploración, 20 px combate)
- 4 estados: FOLLOW_EXPLORE, FOLLOW_COMBAT, CINEMATIC, TRANSITION
- Camera Anchors para composición manual (offset + zoom)
- Límites de zona via Camera2D.limit_*

### Formulas
- F1: Posición objetivo = Edrick.pos + (dir × look-ahead)
- F2: Look-ahead semi-proporcional a velocidad (cresce suavemente de 0 a 50 px)
- F3: Transición de modo con lerp de 0.4s
- F4: Anchor blend interpola offset + zoom
- F5: Clampea final respetando límites de zona

---

## Verificación de Cross-Sistema

✅ **Registry conflicts**: Ninguno. Las constantes de cámara son nuevas:
- look_ahead_explore = 50 px
- look_ahead_combat = 20 px
- smoothing_speed_explore = 6.0
- smoothing_speed_combat = 9.0
- combat_transition_time = 0.4 s
- Candidatos para añadir al registry en siguiente sesión

✅ **Dependencias bidireccionales confirmadas**:
- #1 Movimiento menciona: "Cámara — Depende de la posición y velocidad de Edrick para seguimiento suave" ✓
- #8 Exploración menciona: "Dependen de este sistema: Cámara (#9)" ✓

---

## GDDs MVP Completados Hasta Ahora
1. ✅ Movimiento y Físicas 2D (#1) — Aprobado
2. ✅ Salud y Daño (#2) — Aprobado
3. ✅ Base de Datos de Demonios (#3) — Aprobado
4. ✅ Estado del Mundo (#4) — Aprobado
5. ✅ Sistema de Audio (#5) — Aprobado
6. ✅ Combate en Tiempo Real (#6) — Aprobado
7. ✅ IA de Enemigos (#7) — Aprobado
8. ✅ Exploración del Mundo (#8) — Aprobado (pending design-review)
9. ✅ **NUEVO: Cámara (#9) — Diseñado** (pending design-review)
10. ⏳ Loadout & Build Management (#10) — No Iniciado
11. ⏳ Motor de Sinergias (#11) — No Iniciado
... (21 MVP systems total)

---

## Próximos Pasos Recomendados

**Inmediato (esta sesión o próxima)**:
- [ ] Ejecutar `/design-review design/gdd/camara.md` en **NUEVA sesión** (no en la actual) para validación independiente
- [ ] Si design-review da PASS: cambiar systems-index status a "Aprobado"
- [ ] Ejecutar `/consistency-check` para verificar que los valores de cámara no conflictúan con otros GDDs

**Después**:
- [ ] Siguiente sistema en orden: GDD #10 (Loadout & Build Management) — depende de #3, sin bloqueantes
- [ ] O: GDD #11 (Motor de Sinergias) — depende de #3 y #10
- [ ] Cuando hayan ~15 GDDs MVP completados: ejecutar `/review-all-gdds` (holistic cross-check)

---

## Estadísticas de Esta Sesión

- **Tiempo total**: ~2 horas (sesión + corte por contexto)
- **Secciones diseñadas**: 11 (8 requeridas + 3 opcionales)
- **Fórmulas**: 5 completas con tablas + ejemplos
- **Edge cases**: 7
- **Acceptance criteria**: 10
- **Open questions**: 3
- **Archivos modificados**: camara.md (creado), systems-index.md (actualizado), session-state.md (actualizado)
- **Contexto utilizado**: ~98% (sesión cortada, reanudada)

---

## Decisiones Pendientes (No Bloqueantes)

1. ¿Parallax automático vs definido por zona? (esperar GDD #8 clarificación)
2. ¿Cinemáticas pueden ignorar límites de zona? (esperar GDD #17)
3. ¿Feedback visual de transición combate↔exploración? (esperar GDD #6)

---

---

## SESIÓN 4/4: PRÓXIMO PASO

**Estado actual**: ✅ LISTO PARA `/design-review`

El GDD #9 (Cámara) es:
- ✅ Completo: 8 secciones requeridas + 3 opcionales
- ✅ Técnicamente correcto: fórmulas verificadas, edge cases documentados, ACs testables
- ✅ Alineado con pilares: Pilar 1 (Narrativa Cinematográfica) + Pilar 3 (Mundo Hermoso)
- ✅ Dependencias bidireccionales actualizadas (GDD #1, #6, #8)
- ✅ Arquitectura documentada (ADR-001, ADR-002)
- ✅ Validable: 15 acceptance criteria con métricas claras

**Archivos creados/modificados en Sesiones 1-3:**
- design/gdd/camara.md (creado)
- design/gdd/sistemas-index.md (actualizado)
- design/gdd/combate-tiempo-real.md (actualizado: añadido Camera en dependientes)
- docs/architecture/adr-001-manual-camera-smoothing.md (creado)
- docs/architecture/adr-002-eventbus-global-signals.md (creado)
- production/session-state/active.md (actualizado)

**Próximo paso recomendado:**
```bash
/design-review design/gdd/camara.md
```

Este ejecutará una revisión adversarial completa del GDD por especialistas (resolución de conflictos, validación de fórmulas, consistency checks, game design theory). Si PASS: cambiar systems-index status a "Aprobado" y proceder a GDD #10 (Loadout & Build Management).
