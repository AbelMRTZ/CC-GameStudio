# Session State — GDD #18 + Cross-GDD Actions Completadas

**Fecha**: 2026-06-05
**Última acción**: Acciones cross-GDD X1–X5 aplicadas para desbloquear re-review de GDD #18

## Tarea Completada Esta Sesión

- GDD #18 (HUD de Combate): `/design-review` completo — **In Revision** — 10 blockers resueltos
- Acciones X1–X5: todos los GDDs dependientes actualizados

## Acciones X1–X5 Completadas

- [x] **X1**: GDD #6 (combate-tiempo-real.md) — `combat_started` / `combat_ended` añadidas como señales salientes hacia HUD
- [x] **X2**: GDD #10 (loadout-build-management.md) — `loadout_changed` extendido: `(equipped_demons, gato_available, slot_cooldowns: Array[float], hp_max: int)`
- [x] **X3**: GDD #10 (loadout-build-management.md) — `swap_cooldown_updated(remaining: float)` añadida como señal emitida en tick de COMBAT con cooldown activo
- [x] **X4**: GDD #5 (sistema-audio.md) — bus `SFX_HUD` añadido (Capa 5, −14 dBFS vs SFX_Combate); CA-001 actualizado a `bus_count == 5`; "encounter" definido como ciclo `combat_started/combat_ended`
- [x] **X5**: GDD #3 (base-datos-demonios.md) — `cooldown_segundos` documentado con floor ≥ 0.1s para habilidades activas (null permitido para pasivas/instantáneas)

## Archivos Modificados Esta Sesión

- `design/gdd/hud-combate.md` — revisado post-review (10 blockers, 8 ACs actualizados)
- `design/gdd/reviews/hud-combate-review-log.md` — NUEVO
- `design/gdd/combate-tiempo-real.md` — añadidas `combat_started`/`combat_ended`
- `design/gdd/loadout-build-management.md` — payload extendido + `swap_cooldown_updated`
- `design/gdd/sistema-audio.md` — bus SFX_HUD + CA-001 actualizado
- `design/gdd/base-datos-demonios.md` — floor de cooldown_segundos

## Pendientes Globales

- [ ] `/design-review design/gdd/hud-combate.md` — **re-review listo** (X1–X5 resueltos; ejecutar en sesión fresca)
- [ ] Actualizar GDD #6 (Combate): añadir `combat_sequence_complete` como señal emitida (de GDD #16)
- [ ] Actualizar ADR-002 (EventBus): registrar `combat_started`, `combat_ended`, `swap_cooldown_updated`, señales del GDD #16
- [ ] Actualizar GDD #4 (Estado del Mundo): `world_state.get_nested(path)` para F1-ext
- [ ] Actualizar `acto-1-luxterra.md`: corregir línea obsoleta sobre la reunión Tristán
- [ ] GDD #12 (Guardado y Carga) — re-review pendiente en sesión fresca
- [ ] Diseñar GDD #17 (Cinemáticas) — GDD #16 aprobado, dependencia satisfecha

## Próximo Sistema Recomendado

| Opción | Sistema | Nota |
|--------|---------|------|
| A (recomendado) | `/design-review design/gdd/hud-combate.md` | Re-review en sesión FRESCA — X1–X5 resueltos |
| B | Actualizar ADR-002 | Registrar todas las señales nuevas de GDD #16 + #18 |
| C | `/design-system cinemáticas` | GDD #17, desbloqueado por GDD #16 aprobado |
| D | GDD #19 (Bestiario) | Independiente, scope M |

<!-- STATUS -->
Epic: Combat Systems
Feature: HUD de Combate
Task: Cross-GDD X1–X5 completados — listo para re-review
<!-- /STATUS -->
