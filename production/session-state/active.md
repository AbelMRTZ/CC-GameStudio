# Sesión Activa: GDD #13 Vinculación de Demonios — DISEÑADO ✅

**Tarea completada**: `design/gdd/vinculacion-demonios.md` — todas las secciones escritas y aprobadas.
**Estado**: GDD #13 completo. Pendiente `/design-review` en sesión fresca para confirmar aprobación.
**Archivo**: design/gdd/vinculacion-demonios.md
**Fecha**: 2026-05-29

---

## Resumen de GDD #13

Todas las 10 secciones aprobadas y escritas al archivo:

| Sección | Estado |
|---------|--------|
| §1 Visión General | ✅ Escrita |
| §2 Fantasy del Jugador | ✅ Escrita |
| §3 Diseño Detallado (Reglas + Estados + Interacciones) | ✅ Escrita |
| §4 Fórmulas (F-VD-01 a F-VD-03) | ✅ Escrita |
| §5 Casos Extremos (E1–E6) | ✅ Escrita |
| §6 Dependencias (upstream/downstream/señales) | ✅ Escrita |
| §7 Parámetros de Ajuste | ✅ Escrita |
| §8 Requisitos Visuales y de Audio | ✅ Escrita |
| §9 Criterios de Aceptación (AC-VD-001–033) | ✅ Escrita |
| §10 Preguntas Abiertas (P-VD-01–05) | ✅ Escrita |

---

## Decisiones clave tomadas (GDD #13)

| Decisión | Resolución |
|----------|------------|
| Cómo se obtienen los demonios | SOLO asesinando portadores — regla inmutable. El motivo puede variar. |
| Excepciones a la regla del asesinato | Solo el Gato — binding voluntario vía narrativa (GDD #16 emite `demon_bound("cat")` directamente) |
| El Dash demon en el timeline | Culmen dramático del Acto 1, NO tutorial introductorio |
| Detección de muerte | Señal unificada `portador_murio(portador_id, position)` — emitida por Combate (#6) o NPC (#15) |
| Tipos de secuencia | "standard" (world freeze + partículas + aura) vs "custom" (CanvasLayer scene separada) |
| Timing de partículas | Velocidad adaptativa F-VD-03: `distance / PARTICLE_TRAVEL_TIME`, mínimo 100 px/s |
| Duración total MVP | 2.3s (1.5 + 0.5 + 0.3) |
| Gato en el sistema | No tiene `portador_id`; este sistema solo corre Registro cuando recibe `demon_bound("cat")` |
| Muerte simultánea Edrick+portador | Chequeo `edrick_alive` antes de iniciar secuencia — descarta si false |

---

## Archivos modificados (GDD #13)

- `design/gdd/vinculacion-demonios.md` — GDD #13 completo, Estado: Diseñado
- `design/gdd/systems-index.md` — GDD #13 "No Iniciado" → "Diseñado", tracker: 14 iniciados, 4 diseñados pendientes revisión
- `design/registry/entities.yaml` — Añadidos: constantes PARTICLE_TRAVEL_TIME, AURA_FADE_TIME, SILENCE_TIME, BINDING_DURATION, PARTICLE_MIN_SPEED, PARTICLE_COUNT; señales portador_murio, binding_started, demon_bound, binding_sequence_complete

---

## Preguntas abiertas de GDD #13

| ID | Pregunta | Urgencia |
|----|---------|----------|
| P-VD-01 | Timeout para secuencias custom sin señal `binding_sequence_complete()` | Antes de Alpha |
| P-VD-02 | Contrato de datos pasados a escenas custom al instanciarlas | Antes de primera escena custom |
| P-VD-03 | Handoff aura post-binding a GDD #14 (Transformación Visual) | Antes de GDD #14 |
| P-VD-04 | Re-emisión de `demon_bound("cat")` — duplicar señal o solo registrar | Antes de implementación |
| P-VD-05 | Portadores únicos — ¿restricción permanente o abierta post-MVP? | Antes de Alpha |

---

## Deuda técnica generada (GDD #13)

- **AC-VD-028 depende de GDD #12**: GDD #12 debe documentar que el autosave no ocurre durante SEQUENCE_STANDARD — verificar en `/design-review design/gdd/guardado-y-carga.md`
- **GDD #14 boundary**: La frontera de ownership del aura entre GDD #13 y GDD #14 debe resolverse al diseñar GDD #14 (Transformación Visual de Edrick)

---

## Estado del proyecto

| Sistema | Estado |
|---------|--------|
| GDD #1-9, #15 | ✅ Aprobados (10 GDDs) |
| GDD #10 Loadout & Build Management | 🔄 Diseñado — pendiente re-`/design-review` (señal extendida) |
| GDD #11 Motor de Sinergias | 🔄 Diseñado — pendiente `/design-review` |
| GDD #12 Guardado y Carga | 🔄 Diseñado — pendiente `/design-review` |
| GDD #13 Vinculación de Demonios | 🔄 Diseñado — pendiente `/design-review` |
| GDD #14-26 (excl. #15) | ⬜ No iniciados |

**GDDs aprobados**: 10 / 26 sistemas totales
**GDDs diseñados (pendiente revisión)**: 4 (GDD #10, #11, #12, #13)

---

## Próximos pasos recomendados

```bash
# Opción A — Revisar los GDDs pendientes (orden recomendado):
/design-review design/gdd/guardado-y-carga.md            # GDD #12 — el más antiguo sin revisar
/design-review design/gdd/loadout-build-management.md   # GDD #10 — señal extendida pendiente
/design-review design/gdd/motor-sinergias.md             # GDD #11 — recién diseñado
/design-review design/gdd/vinculacion-demonios.md        # GDD #13 — recién diseñado

# Opción B — Continuar diseñando el siguiente sistema:
/design-system transformacion-visual-edrick              # GDD #14 — depende de Loadout + Base de Datos
```
