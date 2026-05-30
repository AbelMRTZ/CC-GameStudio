# GDD: Guardado y Carga

> **Status**: Diseñado (pendiente /design-review en sesión fresca)
> **Author**: Abel + Agentes
> **Last Updated**: 2026-05-28
> **Implements Pillar**: Infraestructura — habilita Pilar 1 (Narrativa Cinematográfica) y Pilar 5 (Transformación Moral de Edrick)

---

## Overview

El sistema de Guardado y Carga es la capa de persistencia del juego: serializa el `WorldState` completo a un archivo JSON en disco y lo deserializa al volver a jugar. Su responsabilidad está acotada a cuatro triggers: salida de sala (guarda solo campos de sesión — HP y posición), activación de save point o apertura del menú de pausa (guarda estado completo), salida del juego (guarda estado completo + sesión), y ocurrencia de evento narrativo crítico (guarda parcialmente solo `major_events` y `player_choices`). El sistema llama a `WorldState.serialize_full()` o `WorldState.serialize_session_only()` según el trigger — no reimplementa la lógica de serialización, que pertenece a GDD #4. Los campos de `session` (HP actual, posición) no persisten entre cargas; se resetean a sus valores por defecto. En MVP el sistema no implementa cifrado, control de versiones del schema ni detección de manipulación: si el archivo falla la validación de schema, los campos inválidos se autocorrigen a valores por defecto y el juego continúa.

---

## Player Fantasy

Este sistema es infraestructura pura. No tiene fantasía propia — el jugador nunca interactúa con él directamente ni lo percibe como sistema. La fantasía que habilita está declarada en GDD #4 (Estado del Mundo §2): *"Progreso real — las decisiones se recuerdan y quedan."* Este GDD no duplica esa fantasía; solo garantiza que el mecanismo técnico la soporte.

---

## Detailed Design

### Core Rules

1. **Un único slot de guardado (MVP):** El juego mantiene un único archivo `user://saves/save_data.json`. No hay múltiples slots ni backup automático. Si el archivo existe al iniciar, el menú principal ofrece "Continuar". Si no existe, solo "Nueva Partida".

2. **Directorio de saves:** Al guardar por primera vez, el sistema verifica que `user://saves/` existe. Si no, lo crea con `DirAccess`. El directorio nunca se elimina automáticamente.

3. **Triggers de guardado** (implementa exactamente GDD #4 §3.3b):

   | Trigger | Método | Contenido guardado |
   |---------|--------|--------------------|
   | Salida de sala/área | `WorldState.serialize_session_only()` | Solo `session` (HP, posición) |
   | Save point activado | `WorldState.serialize_full()` | Estado completo |
   | "Guardar" en menú de pausa | `WorldState.serialize_full()` | Estado completo |
   | Salida del juego | `WorldState.serialize_full()` | Estado completo |
   | Evento narrativo crítico | Escritura parcial (patch) | Solo `major_events` + `player_choices` afectados |

4. **Guardado parcial (evento crítico):** El sistema lee el archivo actual, actualiza solo `major_events` y `player_choices` relevantes, y reescribe el archivo. No llama a `serialize_full()` — es una actualización quirúrgica para proteger datos en momentos de alta tensión narrativa.

5. **Feedback visual:** Cada operación de guardado (excepto area-exit, que es silencioso) muestra un ícono de guardado en pantalla durante **1.5 segundos**. El ícono no bloquea el gameplay. Si hay una cinemática activa, el ícono se suprime.

6. **Carga:** Solo ocurre al iniciar desde el menú principal ("Continuar"). No hay carga en mitad de partida (sin "load save" desde pausa en MVP — solo "Volver al Menú Principal" que hace reset de la sesión).

7. **Validación al cargar:** Antes de deserializar, el sistema verifica que el JSON tiene los campos críticos (`progression`, `demons`, `narrative`, `world`, `session`). Campos faltantes → valor por defecto. Demonio en `equipped_demons` que no existe en la BD → removido silenciosamente. El juego continúa sin crash.

8. **Error de escritura:** Si `FileAccess.store_string()` devuelve `false`, el sistema registra el error en logs y emite `save_failed` vía EventBus. Sin reintentos automáticos en MVP.

---

### States and Transitions

| Estado | Descripción |
|--------|-------------|
| IDLE | Sistema en reposo. Listo para guardar o cargar. |
| SAVING | Escritura a disco en progreso. Nuevos triggers se descartan. |
| LOADING | Lectura y deserialización en progreso. |
| ERROR | Operación fallida. Notifica vía EventBus, regresa a IDLE. |

Transiciones:
- `IDLE → SAVING`: trigger recibido (área exit / save point / pausa / cierre de app)
- `SAVING → IDLE`: `store_string()` devuelve `true` (éxito)
- `SAVING → ERROR`: `store_string()` devuelve `false` o FileAccess no puede abrir archivo
- `IDLE → LOADING`: solicitud de carga desde menú principal
- `LOADING → IDLE`: archivo válido leído y deserializado
- `LOADING → ERROR`: archivo no existe, inválido o sin permiso de lectura

En SAVING o LOADING, triggers adicionales de guardado se loggean como *"save skipped — already saving"* y se descartan.

---

### Interactions with Other Systems

| Sistema | Dirección | Interfaz |
|---------|-----------|----------|
| **WorldState** (GDD #4) | CALL | `WorldState.serialize_full() → Dictionary`, `WorldState.serialize_session_only() → Dictionary`, `WorldState.deserialize(data: Dictionary)` |
| **EventBus** (ADR-002) | EMIT | `game_saved`, `save_failed`, `game_loaded`, `load_failed`, `show_save_indicator` |
| **Exploración del Mundo** (GDD #8) | LISTEN | `zona_transition_started` (ADR-002 EventBus) → dispara autosave de sesión |
| **Menú Principal y Pausa** (GDD #21) | RECEIVE CALLS | `SaveLoadSystem.save_game()`, `SaveLoadSystem.load_game()`, `SaveLoadSystem.has_save_file() → bool` |
| **Application / SceneTree** | LISTEN | `NOTIFICATION_WM_CLOSE_REQUEST` → dispara save-on-exit |

---

## Formulas

Este sistema no tiene fórmulas de gameplay. Las especificaciones formales relevantes son una tabla de constantes y el algoritmo de validación de schema.

---

**Tabla de constantes:**

| Símbolo | Tipo | Valor | Descripción |
|---------|------|-------|-------------|
| `SAVE_INDICATOR_DURATION` | float | 1.5 s | Tiempo que el ícono de guardado permanece visible. Rango seguro: 0.8–3.0 s. |
| `SAVE_FILE_PATH` | String | `"user://saves/save_data.json"` | Ruta del archivo de save. Constante — no configurable en runtime. |
| `CRITICAL_FIELDS` | Set | `{progression, demons, narrative, world, session}` | Campos que deben existir en el JSON para que el save sea válido. |

---

**Algoritmo de validación de schema al cargar:**

```
Sea CRITICAL_FIELDS = {progression, demons, narrative, world, session}
Sea DEFAULT_VALUES = {progression: {...}, demons: {...}, narrative: {...}, world: {...}, session: {...}}

Para cada campo f en CRITICAL_FIELDS:
  Si f NO está en el JSON cargado:
    json[f] = DEFAULT_VALUES[f]
    registrar WARNING: "Campo '{f}' faltante en save — inicializado a default"

Para cada demonio_id en json.demons.equipped_demons:
  Si demonio_id NO existe en DemonDatabase:
    json.demons.equipped_demons.remove(demonio_id)
    registrar WARNING: "Demonio '{demonio_id}' no existe en BD — removido del save"
```

**Variables:**

| Variable | Tipo | Descripción |
|----------|------|-------------|
| `CRITICAL_FIELDS` | Set[String] | Campos cuya ausencia activa autocorrección |
| `DEFAULT_VALUES` | Dictionary | Valores iniciales por campo (idénticos a `WorldState.new_game()`) |
| `DemonDatabase` | Resource | Base de Datos de Demonios (GDD #3) — fuente de IDs válidos |

**Notas:** La escritura de archivo (`FileAccess.store_string()`) es síncrona en Godot — no hay async I/O ni retry policy posible sin threading. Fallo → emit `save_failed`, log, continuar.

---

## Edge Cases

- **Si el directorio `user://saves/` no existe al intentar guardar:** El sistema lo crea con `DirAccess.make_dir_recursive_absolute()` antes de abrir el archivo. Si la creación falla (permiso denegado), el sistema emite `save_failed`, registra el error, y continúa sin crash.

- **Si `FileAccess.open()` falla al escribir (archivo bloqueado o disco lleno):** `store_string()` devuelve `false`. El sistema emite `save_failed` vía EventBus, registra WARNING en logs con el path y el error. No hay retry. El jugador puede continuar jugando; su progreso no se perdió (solo no se guardó este trigger).

- **Si el archivo de save no existe al intentar cargar ("Continuar" sin partida previa):** El sistema devuelve `false` en `has_save_file()`. El menú principal no muestra la opción "Continuar". Si se llama `load_game()` de todas formas (bug de UI), el sistema registra ERROR y devuelve `null` — el llamador debe manejar `null` con nueva partida.

- **Si el JSON cargado es sintácticamente inválido (no parseable):** `JSON.parse()` devuelve error. El sistema emite `load_failed`, registra ERROR, y NO carga el save (el juego no intenta autocorregir JSON inválido). El menú principal muestra error visible al jugador: *"El archivo de guardado está dañado. No se puede continuar."*

- **Si un trigger de guardado llega mientras el sistema está en SAVING o LOADING:** El trigger se descarta silenciosamente. Se registra en logs: `"save skipped — operation in progress"`. No se encolan triggers.

- **Si el guardado ocurre al mismo tiempo que una cinemática activa:** El ícono de guardado se suprime (no aparece en pantalla). El guardado ocurre de todas formas — solo se suprime el feedback visual. El sistema escucha `cinematic_started` de EventBus para saber cuándo suprimir el ícono.

- **Si el jugador cierra la app abruptamente (kill de proceso, crash):** El hook `NOTIFICATION_WM_CLOSE_REQUEST` no llega. El save-on-exit no ocurre. El jugador pierde el progreso desde el último trigger automático (área exit). Este es el comportamiento esperado — no hay workaround sin auto-save periódico (post-MVP).

- **Si el archivo de save tiene un demonio en `equipped_demons` que ya no existe en la Base de Datos de Demonios (save editado manualmente o post-patch):** El algoritmo de validación (Sección D) lo remueve silenciosamente de `equipped_demons`. El demonio permanece en `available_demons` si estaba ahí. El jugador no recibe notificación — la diferencia es visible solo en la pantalla de Loadout.

- **Si se guarda un save durante la animación de swap de demonio (SWAP_ANIM_DURATION = 0.8s):** El save se dispara con el estado del `WorldState` en ese momento. Si el swap no ha finalizado, `equipped_demons` puede estar en estado transitorio (demonio antiguo aún presente). La animación de swap siempre completa su ciclo — este edge case es teórico en MVP porque los triggers de save no ocurren durante combate activo.

---

## Dependencies

### Upstream (este sistema depende de)

| Sistema | Tipo | Interfaz |
|---------|------|----------|
| **Estado del Mundo** (GDD #4) | Hard | `WorldState.serialize_full()`, `WorldState.serialize_session_only()`, `WorldState.deserialize(data)`, `WorldState.new_game()` → DEFAULT_VALUES |
| **Base de Datos de Demonios** (GDD #3) | Hard | Validación de IDs en `equipped_demons` al cargar — fuente de IDs válidos |
| **EventBus** (ADR-002) | Hard | Todos los eventos inter-sistema: `game_saved`, `save_failed`, `game_loaded`, `load_failed`, `show_save_indicator` |

### Downstream (dependen de este sistema)

| Sistema | Tipo | Interfaz |
|---------|------|----------|
| **Menú Principal y Pausa** (GDD #21) | Hard | `SaveLoadSystem.save_game()`, `SaveLoadSystem.load_game()`, `SaveLoadSystem.has_save_file() → bool` |
| **HUD** (GDD #18) | Soft | Escucha `show_save_indicator` de EventBus para mostrar ícono de guardado |

### Lateral / Trigger

| Sistema | Rol | Señal |
|---------|-----|-------|
| **Exploración del Mundo** (GDD #8) | Trigger de autosave de sesión | `zona_transition_started` (EventBus / ADR-002) |
| **SceneTree (Godot)** | Trigger de save-on-exit | `NOTIFICATION_WM_CLOSE_REQUEST` |
| **Cinemáticas** (GDD #17) | Supresión del ícono de guardado | `cinematic_started` (EventBus / ADR-002) |

### Notas de bidireccionalidad

- GDD #4 (Estado del Mundo §6.1 item 1) ya documenta que Guardado y Carga depende de él. ✅ Consistente.
- GDD #8 (Exploración del Mundo) no menciona explícitamente que su señal `zona_transition_started` es usada por Guardado y Carga. Pendiente de actualizar cuando se revise GDD #8.

---

## Tuning Knobs

| Parámetro | Valor MVP | Rango Seguro | Efecto si demasiado alto | Efecto si demasiado bajo |
|-----------|-----------|-------------|--------------------------|--------------------------|
| `SAVE_INDICATOR_DURATION` | 1.5 s | 0.8 – 3.0 s | El ícono molesta al jugador (obstruye UI) | El jugador no lo ve y no sabe que guardó |
| Triggers de guardado automático | 4 triggers definidos | — | Más triggers → más I/O, sensación de "lag" | Menos triggers → mayor riesgo de pérdida de datos |

**Interacciones entre knobs:**
- Reducir el número de triggers de guardado automático aumenta el riesgo de pérdida de progreso en crash. Si se elimina el trigger de area-exit, el jugador puede perder hasta 30–120 minutos de juego. No recomendado.
- `SAVE_INDICATOR_DURATION` es independiente de los demás knobs.

**Knobs que NO existen en MVP:**
- Tamaño máximo de archivo (no aplica — el JSON nunca supera ~50 KB)
- Frecuencia de auto-save por tiempo (no implementada — post-MVP si fuera necesario)
- Número de slots de guardado (fijo en 1 para MVP)

---

## Visual/Audio Requirements

No aplica para este sistema. Guardado y Carga es infraestructura pura sin representación visual o auditiva propia. El único feedback visible (ícono de guardado 1.5s) es propiedad del HUD (GDD #18) y se dispara por la señal `show_save_indicator`.

---

## UI Requirements

No aplica en MVP. Este sistema no posee pantallas ni widgets propios. La UI que interactúa con él (botón "Continuar", botón "Nueva Partida", indicador de save corrupto) pertenece al Menú Principal y Pausa (GDD #21). El ícono de guardado pertenece al HUD (GDD #18).

---

## Acceptance Criteria

### Bloque A — Core Rules

**CA-SL-001** — GIVEN que el jugador activa un save point, WHEN el sistema escribe a disco, THEN el archivo existe en `user://saves/save_data.json` y ningún otro archivo de save es creado en `user://saves/`.

**CA-SL-002** — GIVEN que el jugador cruza una transición de zona, WHEN el sistema recibe `zona_transition_started`, THEN el JSON contiene solo los campos de sesión actualizados; `progression`, `demons`, `narrative`, `world` no son modificados.

**CA-SL-003** — GIVEN que el jugador activa un save point o "Guardar" en pausa, WHEN completa la operación, THEN el JSON contiene los 5 campos críticos con los valores actuales del WorldState en ese momento.

**CA-SL-004** — GIVEN que el jugador cierra la app normalmente, WHEN el sistema recibe `NOTIFICATION_WM_CLOSE_REQUEST`, THEN el archivo JSON se escribe completo antes de que el proceso termine y es parseable tras el cierre.

**CA-SL-005** — GIVEN que ocurre un evento narrativo crítico, WHEN el sistema ejecuta el guardado parcial, THEN solo `major_events` y `player_choices` son actualizados; todos los demás campos permanecen sin cambio; `serialize_full()` NO es llamado.

**CA-SL-006** — GIVEN que el jugador está en mitad de partida, WHEN inspecciona el menú de pausa, THEN no existe opción "Cargar Partida"; solo "Volver al Menú Principal".

**CA-SL-007** — GIVEN que no existe `save_data.json`, WHEN el jugador llega al menú principal, THEN "Continuar" no aparece. GIVEN que sí existe, THEN "Continuar" aparece disponible.

---

### Bloque B — Máquina de Estados

**CA-SL-008** — GIVEN que el sistema está en IDLE, WHEN se recibe un trigger de guardado y `store_string()` devuelve true, THEN el sistema transita IDLE → SAVING → IDLE y la señal `game_saved` es emitida exactamente una vez.

**CA-SL-009** `[requires: mock de I/O lento]` — GIVEN que el sistema está en SAVING, WHEN llega un segundo trigger de guardado, THEN el segundo se descarta; el log registra `"save skipped — operation in progress"`; el primer guardado completa sin error; ninguna señal duplicada es emitida.

**CA-SL-010** `[requires: mock de I/O lento]` — GIVEN que el sistema está en LOADING, WHEN llega un trigger de guardado, THEN el trigger se descarta; el log registra `"save skipped — operation in progress"`; la carga no es interrumpida.

---

### Bloque C — Validación al Cargar

**CA-SL-011** — GIVEN que el JSON cargado no contiene el campo `demons` (repetir para cada uno de los 5 CRITICAL_FIELDS), WHEN se ejecuta `load_game()`, THEN el campo faltante se rellena con `DEFAULT_VALUES[f]`; el log registra `WARNING: "Campo '[f]' faltante en save — inicializado a default"`; el juego carga sin crash.

**CA-SL-012** — GIVEN que `equipped_demons` contiene un ID inexistente en DemonDatabase, WHEN el sistema valida el schema, THEN ese ID es removido silenciosamente; el log registra WARNING; el jugador no recibe notificación en pantalla; el juego carga sin crash.

**CA-SL-013** — GIVEN que `save_data.json` existe pero su contenido no es JSON válido, WHEN el jugador selecciona "Continuar", THEN el sistema emite `load_failed`; registra ERROR; muestra el mensaje exacto *"El archivo de guardado está dañado. No se puede continuar."*; no intenta autocorregir el JSON; regresa a IDLE.

---

### Bloque D — Edge Cases de I/O

**CA-SL-014** `[requires: directorio user://saves/ eliminado previamente]` — GIVEN que `user://saves/` no existe, WHEN el sistema ejecuta un trigger de guardado, THEN el directorio se crea automáticamente; `save_data.json` se crea correctamente; `game_saved` es emitido; sin crash ni error visible al jugador.

**CA-SL-015** `[requires: mock de FileAccess inyectable]` — GIVEN que `store_string()` devuelve `false` (simulado), WHEN el sistema intenta completar el guardado, THEN emite `save_failed`; registra WARNING con el path y el error; transita SAVING → ERROR → IDLE; el juego continúa sin crash; sin retry automático.

**CA-SL-016** — GIVEN que `has_save_file()` devuelve false, WHEN se llama `load_game()` directamente (bug de UI), THEN el método devuelve `null`; registra ERROR; emite `load_failed`; sin crash; el llamador puede manejar `null` iniciando nueva partida.

**CA-SL-017** `[requires: test manual documentado]` — GIVEN que existe un `save_data.json` válido de sesión anterior, WHEN el proceso es terminado abruptamente (kill), THEN al reiniciar, "Continuar" carga el save anterior sin corrupción; el progreso de la sesión terminada desde el último trigger automático no está presente.

---

### Bloque E — Integración con EventBus

**CA-SL-018** — GIVEN que se completa un guardado por save point, pausa o salida del juego, WHEN el sistema termina, THEN `show_save_indicator` es emitido vía EventBus; NOT emitido en area-exit ni en guardado parcial de evento crítico.

---

### Bloque F — Feedback Visual `[ADVISORY]`

**CA-SL-019** `[requires: clock injection o evidencia manual]` — GIVEN que el HUD recibe `show_save_indicator`, WHEN se mide el tiempo, THEN el ícono permanece visible exactamente `SAVE_INDICATOR_DURATION` (1.5 s) y luego desaparece; no bloquea gameplay durante ese tiempo.

**CA-SL-020** `[ADVISORY]` — GIVEN que `cinematic_started` está activo en EventBus, WHEN ocurre un trigger de guardado con feedback visual, THEN el guardado se ejecuta y `game_saved` es emitido, pero `show_save_indicator` NO es emitido y el ícono no aparece en pantalla.

---

### Resumen de Gates

| Criterios | Tipo | Gate | Tests |
|-----------|------|------|-------|
| CA-SL-001–005, CA-SL-008–016 | Logic | **BLOCKING** | Tests unitarios en `tests/unit/save_load/` |
| CA-SL-006–007 | UI | ADVISORY | Walkthrough manual |
| CA-SL-017–018 | Integration | **BLOCKING** | CA-SL-017: manual documentado; CA-SL-018: integración con EventBus mock |
| CA-SL-019–020 | Visual | ADVISORY | Evidencia manual en `production/qa/evidence/` |

> ⚠️ **Nota de implementación:** CA-SL-015 requiere que `FileAccess` sea inyectable (no llamado como singleton directo). Si se usa `FileAccess.open()` sin abstracción, este criterio no puede ser cubierto por tests automatizados. Definir patrón de inyección en ADR antes de implementar.

---

## Open Questions

- **¿FileAccess debe ser inyectable?** CA-SL-015 requiere mockear `FileAccess` para testear errores de escritura. Si el implementador usa `FileAccess.open()` como singleton directo, ese criterio no puede ser cubierto. Decisión pendiente de ADR antes de implementar.
  - **Owner**: Lead Programmer / godot-gdscript-specialist
  - **Target**: Antes de comenzar implementación de este sistema

- **¿El guardado parcial (evento crítico) debe ser atómico?** Actualmente se lee el archivo completo, se parcha en memoria, y se reescribe. Si el proceso muere entre la lectura y la escritura, el archivo puede quedar en estado corrupto. ¿Vale la pena escribir a un archivo temporal y hacer rename atómico en MVP?
  - **Owner**: Technical Director
  - **Target**: Pre-producción

- **¿Save cloud (Steam Cloud) en scope?** La ruta `user://` de Godot es local. Si Steam Cloud Saves es un requisito (preservar partida entre PCs), la ruta de guardado y el sistema de sync necesitan diseño adicional. No está en scope MVP pero afecta la arquitectura del path.
  - **Owner**: Producer / Technical Director
  - **Target**: Evaluación post-MVP
