# Sesión Activa: GDD #12 Revisado — LISTO PARA RE-REVIEW

**Tarea completada**: `/design-review design/gdd/guardado-y-carga.md` — revisión full con 3 especialistas (systems-designer, qa-lead, godot-specialist) + creative-director. Veredicto inicial MAJOR REVISION NEEDED → 16 bloqueantes corregidos en sesión.
**Estado**: GDD #12 en "En Revisión". Listo para re-review tras `/clear`.
**Archivo**: design/gdd/guardado-y-carga.md
**Fecha**: 2026-05-31

---

## Cambios aplicados a GDD #12 (sesión 2026-05-31)

| Bloqueante | Fix aplicado |
|-----------|-------------|
| DirAccess._absolute() crash | → DirAccess.make_dir_recursive() — Regla 2 + notas impl. |
| JSON.parse() API de Godot 3 | → JSON.parse_string() en algoritmo + notas |
| 5 señales EventBus no declaradas | → Warning + bloque prerequisites en ACs |
| set_auto_accept_quit(false) faltante | → Interactions + nota de implementación |
| Escritura no atómica (destruye save en kill) | → Core Rule 4: write-to-temp-then-rename OBLIGATORIO |
| player_choices en dos namespaces | → Clarificado: progression.major_events + narrative.player_choices |
| Regla 5 vs CA-SL-018 contradicción | → Decisión: guardados parciales silenciosos; Regla 6 actualizada |
| Area-exit dead write sin lector | → Decisión: area-exit = solo in-memory, sin I/O |
| available_demons no validado | → Añadido al algoritmo de validación |
| corruption_floor > corruption_level | → Invariante añadida al algoritmo de validación |
| CA-SL-002 sin baseline | → Reescrita: verifica que area-exit NO escribe a disco |
| CA-SL-004 "parseable" indefinido | → Definido + reclasificado integration test |
| CA-SL-005 sin mecanismo spy | → Tag WorldState mock/spy añadido |
| CA-SL-009/010 I/O lento inalcanzable | → Cambiado a _state = SAVING via test helper |
| CA-SL-011 plantilla ambigua | → Dividida en CA-SL-011a a 011e |
| CA-SL-012 log string no especificado | → String exacto añadido |
| CA-SL-013 UI concern en sistema test | → Removida aserción "muestra mensaje" (GDD #21) |
| CA-SL-017 protocolo manual incompleto | → Protocolo + evidence path + sign-off |
| AC faltante: partial save FSM | → CA-SL-022 añadida |
| AC faltante: game_loaded signal | → CA-SL-021 añadida |
| AC faltante: DirAccess failure | → CA-SL-014b añadida |
| FileAccess injection sin patrón | → FileAccessWrapper pattern en notas + Open Questions |
| SAVING→IDLE antes de emit | → Nota en States + implementación |
| cinematic_started signature | → Nota en Interactions |

---

## Prerequisitos de implementación de GDD #12 (antes del sprint)

1. **Actualizar ADR-002**: añadir señales `game_saved`, `save_failed`, `game_loaded`, `load_failed`, `show_save_indicator` a `event_bus.gd`
2. **Crear ADR**: patrón `FileAccessWrapper` para inyección de FileAccess (CA-SL-015 lo requiere)
3. **Definir test helper**: exposición de `_state` en SaveLoadSystem para CA-SL-009/010

---

## Estado del proyecto (GDDs)

| GDD | Sistema | Estado |
|-----|---------|--------|
| GDD #1–11, #13, #15 | Varios | Aprobado |
| GDD #12 | Guardado y Carga | En Revisión — listo para re-review |
| GDD #14 | Transformación Visual de Edrick | En Revisión — pendiente decisiones Art Direction (P-TVE-02) |
| GDD #16–26 (excl. #15) | Varios | No Iniciados |

**GDDs en revisión**: 2 (GDD #12, #14)

---

## Pendientes que bloquean re-review de GDD #14

- **P-TVE-02**: arquitectura sprite para heterocromía — decisión de Art Direction
- **P-TVE-06**: GDD #2 (Salud y Daño) debe añadir señal `edrick_respawned()`
- **P-TVE-07**: GDD #10 (Loadout) debe añadir señal `loadout_swap_started(new_demons)`
- **GDD #13**: actualizar para listar GDD #14 como dependiente

---

## Próximos pasos recomendados

```bash
# Opción A — Re-revisar GDD #12 tras correcciones (recomendado):
/clear
/design-review design/gdd/guardado-y-carga.md    # Debería alcanzar APPROVED

# Opción B — Re-revisar GDD #14 tras resolver Art Direction:
# 1. Resolver P-TVE-02 con Art Director
# 2. /design-review design/gdd/transformacion-visual-edrick.md

# Opción C — Revisar todos los GDDs:
/review-all-gdds                                  # Holistic cross-system review

# Prerequisitos de implementación GDD #12 (antes de sprint):
# /architecture-decision   # FileAccessWrapper pattern
# Editar ADR-002 para añadir señales save/load
```
