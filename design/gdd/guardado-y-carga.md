# GDD: Guardado y Carga

> **Status**: En Revisión — post /design-review (2026-05-31)
> **Author**: Abel + Agentes
> **Last Updated**: 2026-05-31
> **Implements Pillar**: Infraestructura — habilita Pilar 1 (Narrativa Cinematográfica) y Pilar 5 (Transformación Moral de Edrick)

---

## Overview

El sistema de Guardado y Carga es la capa de persistencia del juego: serializa el `WorldState` completo a un archivo JSON en disco y lo deserializa al volver a jugar. Su responsabilidad está acotada a **tres grupos de triggers**: (1) guardado completo iniciado por el jugador — save point activado o "Guardar" desde el menú de pausa — escribe el estado completo; (2) guardado automático al salir del juego — escribe el estado completo; (3) evento narrativo crítico — escribe parcialmente solo `progression.major_events` y `narrative.player_choices`. El sistema llama a `WorldState.serialize_full()` para los dos primeros grupos y ejecuta un patch quirúrgico directo sobre el JSON para el tercero; no reimplementa la lógica de serialización, que pertenece a GDD #4. Los campos de `session` (HP actual, posición) solo están presentes en el JSON tras un guardado completo, pero se resetean a sus valores por defecto al cargar — su presencia en el archivo no tiene efecto funcional en MVP. En MVP el sistema no implementa cifrado, control de versiones del schema ni detección de manipulación: si el archivo falla la validación de schema, los campos inválidos se autocorrigen a valores por defecto y el juego continúa.

---

## Player Fantasy

Este sistema es infraestructura pura. No tiene fantasía propia — el jugador nunca interactúa con él directamente ni lo percibe como sistema. La fantasía que habilita está declarada en GDD #4 (Estado del Mundo §2): *"Progreso real — las decisiones se recuerdan y quedan."* Este GDD no duplica esa fantasía; solo garantiza que el mecanismo técnico la soporte.

---

## Detailed Design

### Core Rules

1. **Un único slot de guardado (MVP):** El juego mantiene un único archivo `user://saves/save_data.json`. No hay múltiples slots ni backup automático. Si el archivo existe al iniciar, el menú principal ofrece "Continuar". Si no existe, solo "Nueva Partida".

2. **Directorio de saves:** Al guardar por primera vez, el sistema verifica que `user://saves/` existe. Si no, lo crea con `DirAccess.make_dir_recursive("user://saves/")` (método estático — **no** `make_dir_recursive_absolute`, que interpreta el argumento como ruta del SO). El directorio nunca se elimina automáticamente.

3. **Triggers de guardado:**

   | Trigger | Método | Contenido guardado |
   |---------|--------|--------------------|
   | Save point activado | `WorldState.serialize_full()` | Estado completo |
   | "Guardar" en menú de pausa | `WorldState.serialize_full()` | Estado completo |
   | Salida del juego | `WorldState.serialize_full()` | Estado completo |
   | Evento narrativo crítico | Patch quirúrgico directo | Solo `progression.major_events` + `narrative.player_choices` |

   El cruce de zona/área **no dispara ninguna escritura a disco**. El `WorldState` se actualiza en memoria al cruzar zonas, pero sin I/O de archivo. El siguiente guardado completo (save point, pausa, o salida) capturará ese estado actualizado.

4. **Patrón de escritura atómica — obligatorio para todos los saves:** Toda escritura a disco usa write-to-temp-then-rename:
   ```
   1. Serializar el Dictionary a JSON string
   2. Abrir user://saves/save_data.json.tmp en modo WRITE
      Si open() retorna null → emitir save_failed, retornar
   3. store_string(json_string)
      Si retorna false → emitir save_failed, borrar .tmp, retornar
   4. DirAccess.rename("user://saves/save_data.json.tmp",
                        "user://saves/save_data.json")
      Si falla → emitir save_failed, intentar borrar .tmp, retornar
   ```
   `FileAccess.WRITE` trunca el archivo al abrirlo, antes de escribir ningún dato. Sin write-to-temp, un proceso terminado entre `open()` y `store_string()` destruye el único save del jugador. El rename es atómico en Windows, Linux y macOS cuando source y destino están en el mismo filesystem.

5. **Guardado parcial (evento crítico):** El sistema lee `save_data.json`, parsea el JSON, actualiza en memoria solo `progression.major_events` y `narrative.player_choices` relevantes, y aplica el patrón write-to-temp-then-rename para reescribir el archivo completo. No llama a `serialize_full()` — es una actualización quirúrgica para proteger datos en momentos de alta tensión narrativa. Si el parseo del archivo existente falla, el sistema emite `save_failed` y no sobreescribe el archivo.

6. **Feedback visual:** Los guardados por save point, menú de pausa y salida del juego muestran un ícono de guardado en pantalla durante `SAVE_INDICATOR_DURATION` (1.5 s). El ícono no bloquea el gameplay. Los guardados de **evento narrativo crítico son silenciosos** (no muestran el ícono) para preservar la inmersión narrativa en momentos de alta carga emocional. Si hay una cinemática activa en el momento del guardado, el ícono también se suprime.

7. **Carga:** Solo ocurre al iniciar desde el menú principal ("Continuar"). No hay carga en mitad de partida (sin "load save" desde pausa en MVP — solo "Volver al Menú Principal" que hace reset de la sesión).

8. **Validación al cargar:** Antes de deserializar, el sistema verifica que el JSON tiene los campos críticos (`progression`, `demons`, `narrative`, `world`, `session`). Campos faltantes → valor por defecto. Demonio en `equipped_demons` o `available_demons` que no existe en la BD → removido silenciosamente. `narrative.corruption_level` fuera de [0.0, 1.0] → clampeado. `narrative.corruption_floor > narrative.corruption_level` → `corruption_floor` ajustado al valor de `corruption_level`. El juego continúa sin crash.

9. **Error de escritura:** Si el rename final falla (o `store_string()` retorna false, o `FileAccess.open()` retorna null), el sistema registra el error en logs y emite `save_failed` vía EventBus. Sin reintentos automáticos en MVP.

---

### States and Transitions

| Estado | Descripción |
|--------|-------------|
| IDLE | Sistema en reposo. Listo para guardar o cargar. |
| SAVING | Escritura a disco en progreso. Nuevos triggers se descartan. |
| LOADING | Lectura y deserialización en progreso. |
| ERROR | Operación fallida. Transición inmediata a IDLE; notifica vía EventBus. |

Transiciones:
- `IDLE → SAVING`: trigger recibido (save point / pausa / cierre de app / evento narrativo crítico)
- `SAVING → IDLE`: rename atómico completado con éxito — **el estado se cambia a IDLE ANTES de emitir `game_saved`** para evitar re-entradas síncronas por listeners del signal
- `SAVING → ERROR → IDLE`: `store_string()` retorna false, o rename falla, o `FileAccess.open()` retorna null — ERROR es una transición inmediata (inline), no un estado persistente; se emite `save_failed` y se retorna a IDLE
- `IDLE → LOADING`: solicitud de carga desde menú principal
- `LOADING → IDLE`: archivo válido leído y deserializado; `game_loaded` emitido
- `LOADING → ERROR → IDLE`: archivo no existe, JSON inválido o sin permiso de lectura

En SAVING o LOADING, triggers adicionales de guardado se loggean como *"save skipped — operation in progress"* y se descartan. **Nota sobre eventos narrativos descartados:** Si un evento crítico es descartado durante SAVING, el evento ya fue registrado en `WorldState` en memoria — el próximo guardado completo (save point, pausa, salida) lo capturará correctamente.

---

### Interactions with Other Systems

| Sistema | Dirección | Interfaz |
|---------|-----------|----------|
| **WorldState** (GDD #4) | CALL | `WorldState.serialize_full() → Dictionary`, `WorldState.deserialize(data: Dictionary)` |
| **EventBus** (ADR-002) | EMIT | `game_saved`, `save_failed`, `game_loaded`, `load_failed`, `show_save_indicator` |
| **Menú Principal y Pausa** (GDD #21) | RECEIVE CALLS | `SaveLoadSystem.save_game()`, `SaveLoadSystem.load_game()`, `SaveLoadSystem.has_save_file() → bool` |
| **Application / SceneTree** | LISTEN + CONFIG | `NOTIFICATION_WM_CLOSE_REQUEST` → dispara save-on-exit; **requiere** `get_tree().set_auto_accept_quit(false)` en `_ready()` + `get_tree().quit()` manual tras completar el save |
| **Cinemáticas** (GDD #17) | LISTEN | `cinematic_started(camera_data: Dictionary)` vía EventBus → activa flag de supresión del ícono |

> **⚠️ Prerequisito de implementación:** Las señales `game_saved`, `save_failed`, `game_loaded`, `load_failed`, `show_save_indicator` deben declararse en `res://autoload/event_bus.gd` (ADR-002) antes de implementar este sistema. En Godot 4, emitir una señal no declarada lanza un error en runtime. Actualizar ADR-002 es tarea bloqueante del primer sprint.

> **Nota sobre `cinematic_started`:** El listener de este sistema debe aceptar el parámetro aunque no lo use: `func _on_cinematic_started(_camera_data: Dictionary) -> void:`. Omitir el parámetro causa fallo de conexión en Godot 4.

---

## Formulas

Este sistema no tiene fórmulas de gameplay. Las especificaciones formales son una tabla de constantes, el algoritmo de partial save, y el algoritmo de validación de schema al cargar.

---

**Tabla de constantes:**

| Símbolo | Tipo | Valor | Descripción |
|---------|------|-------|-------------|
| `SAVE_INDICATOR_DURATION` | float | 1.5 s | Tiempo que el ícono permanece visible. Rango seguro: 0.8–3.0 s. |
| `SAVE_FILE_PATH` | String | `"user://saves/save_data.json"` | Ruta final del archivo de save. |
| `SAVE_TEMP_PATH` | String | `"user://saves/save_data.json.tmp"` | Ruta temporal para write-to-temp-then-rename. |
| `CRITICAL_FIELDS` | Set | `{progression, demons, narrative, world, session}` | Campos que deben existir en el JSON para que el save sea válido. |

---

**Algoritmo de partial save (evento narrativo crítico):**

```
Sea partial_data = leer y parsear JSON desde save_data.json
Si parseo falla (JSON.parse_string() retorna null):
    emitir save_failed; registrar ERROR; retornar (no reescribir el archivo)

Para cada evento_id en major_events_changed:
    partial_data["progression"]["major_events"][evento_id] =
        WorldState.get_major_event(evento_id)

Para cada choice_id en player_choices_changed:
    partial_data["narrative"]["player_choices"][choice_id] =
        WorldState.get_player_choice(choice_id)

Aplicar write-to-temp-then-rename con JSON.stringify(partial_data)
```

**Rutas canónicas:** El partial save actualiza `progression.major_events` y `narrative.player_choices`. Estas son las dos rutas exactas definidas en el schema de GDD #4. No existe un campo `player_choices` bajo `progression`.

---

**Algoritmo de validación de schema al cargar:**

```
Sea CRITICAL_FIELDS = {progression, demons, narrative, world, session}
Sea DEFAULT_VALUES = {progression: {...}, demons: {...}, narrative: {...},
                      world: {...}, session: {...}}

Para cada campo f en CRITICAL_FIELDS:
    Si f NO está en el JSON cargado:
        json[f] = DEFAULT_VALUES[f]
        registrar WARNING: "Campo '{f}' faltante en save — inicializado a default"

# Validación de IDs de demonios
Para cada demonio_id en json["demons"]["equipped_demons"]:
    Si demonio_id NO existe en DemonDatabase:
        json["demons"]["equipped_demons"].erase(demonio_id)
        registrar WARNING: "Demonio '{demonio_id}' no existe en BD — removido de equipped_demons"

Para cada demonio_id en json["demons"]["available_demons"]:
    Si demonio_id NO existe en DemonDatabase:
        json["demons"]["available_demons"].erase(demonio_id)
        registrar WARNING: "Demonio '{demonio_id}' no existe en BD — removido de available_demons"

# Validación de invariantes de corrupción
Si json["narrative"]["corruption_level"] < 0.0 o > 1.0:
    json["narrative"]["corruption_level"] =
        clamp(json["narrative"]["corruption_level"], 0.0, 1.0)
    registrar WARNING: "corruption_level fuera de rango — corregido"

Si json["narrative"]["corruption_floor"] > json["narrative"]["corruption_level"]:
    json["narrative"]["corruption_floor"] = json["narrative"]["corruption_level"]
    registrar WARNING: "corruption_floor > corruption_level — ajustado"
```

**Variables:**

| Variable | Tipo | Descripción |
|----------|------|-------------|
| `CRITICAL_FIELDS` | Set[String] | Campos cuya ausencia activa autocorrección |
| `DEFAULT_VALUES` | Dictionary | Valores iniciales por campo (idénticos a `WorldState.new_game()`) |
| `DemonDatabase` | Resource | Base de Datos de Demonios (GDD #3) — fuente de IDs válidos |

---

**Notas de implementación — requeridas en ADR antes del primer sprint:**

1. **API correcta de JSON (Godot 4.x):** No existe `JSON.parse(text)` como método estático en Godot 4 — es API de Godot 3 y lanzará un error en runtime. Usar `JSON.parse_string(text)` (retorna `null` en fallo) o `var j = JSON.new(); j.parse(text); j.get_data()`. Para serializar: `JSON.stringify(dict)` (estático — correcto en Godot 4).

2. **API correcta de directorio (Godot 4.x):** Usar `DirAccess.make_dir_recursive("user://saves/")` — estático, resuelve el path virtual de Godot. El sufijo `_absolute` (`make_dir_recursive_absolute`) interpreta el argumento como ruta del SO, comportamiento indefinido con paths `user://`.

3. **Null-check en FileAccess.open():** En Godot 4, `FileAccess.open()` retorna `null` en fallo (no lanza excepción). Sin null-check, llamar `store_string()` sobre null produce un crash, no una emisión limpia de `save_failed`. Patrón correcto:
   ```gdscript
   var file = FileAccess.open(path, FileAccess.WRITE)
   if file == null:
       EventBus.save_failed.emit()
       return
   ```

4. **set_auto_accept_quit(false) para save-on-exit:** El nodo principal debe llamar `get_tree().set_auto_accept_quit(false)` en `_ready()`. Sin esto, Godot cierra el proceso al recibir `NOTIFICATION_WM_CLOSE_REQUEST` antes de que el save síncrono complete. El handler debe llamar `get_tree().quit()` manualmente tras el save.

5. **Orden de transición de estado:** `_state = IDLE` debe ejecutarse ANTES de `EventBus.game_saved.emit()`. Godot despacha señales síncronamente; si un listener de `game_saved` dispara otro save, el sistema debe estar en IDLE para procesarlo.

6. **Inyección de FileAccess (requerido para CA-SL-015):** El sistema debe aceptar un `FileAccessWrapper` inyectable. Ver Open Questions §1 para el patrón concreto.

---

## Edge Cases

- **Si el directorio `user://saves/` no existe al intentar guardar:** El sistema lo crea con `DirAccess.make_dir_recursive("user://saves/")` antes de abrir el archivo temporal. Si la creación falla (permiso denegado), el sistema emite `save_failed`, registra el error, y continúa sin crash.

- **Si `FileAccess.open()` retorna null al abrir el archivo temporal (bloqueado o disco lleno):** El sistema emite `save_failed` vía EventBus, registra WARNING con el path y el error. No hay retry. El archivo `save_data.json` original no es modificado.

- **Si el write-to-temp completa pero el rename falla:** El sistema emite `save_failed`, registra ERROR, e intenta borrar `save_data.json.tmp` para evitar archivos huérfanos. `save_data.json` permanece intacto con el estado del último save exitoso.

- **Si el archivo de save no existe al intentar cargar ("Continuar" sin partida previa):** `has_save_file()` retorna `false`. El menú principal no muestra "Continuar". Si se llama `load_game()` de todas formas (bug de UI), el sistema registra ERROR y retorna `null` — el llamador debe manejar `null` iniciando nueva partida.

- **Si el JSON cargado es sintácticamente inválido:** `JSON.parse_string()` retorna `null`. El sistema emite `load_failed` con payload `{reason: "corrupt_json"}`, registra ERROR, y NO carga el save. El menú principal (GDD #21) escucha `load_failed` y muestra el error al jugador: *"El archivo de guardado está dañado. No se puede continuar."*

- **Si un trigger de guardado llega mientras el sistema está en SAVING o LOADING:** El trigger se descarta. Se registra: `"save skipped — operation in progress"`. Si el trigger descartado era un evento narrativo crítico, el evento persiste en `WorldState` en memoria — el próximo guardado completo lo capturará.

- **Si el guardado ocurre durante una cinemática activa:** El guardado se ejecuta normalmente. Solo se suprime el feedback visual (`show_save_indicator` no se emite). El sistema escucha `cinematic_started(camera_data: Dictionary)` — el parámetro `camera_data` es ignorado pero la firma del handler debe incluirlo.

- **Si el jugador cierra la app abruptamente (kill de proceso, crash):** `NOTIFICATION_WM_CLOSE_REQUEST` no llega. El save-on-exit no ocurre. El jugador pierde el progreso desde el último guardado exitoso. El patrón write-to-temp garantiza que el archivo anterior queda íntegro — la pérdida es de progreso, no de corrupción del save.

- **Si un demonio en `equipped_demons` o `available_demons` no existe en DemonDatabase (save editado o post-patch):** El algoritmo de validación lo remueve silenciosamente de ambas listas. Visible solo en la pantalla de Loadout.

- **Si `narrative.corruption_floor` supera `narrative.corruption_level` al cargar (save editado):** El algoritmo ajusta `corruption_floor` al valor de `corruption_level`. Sin esta corrección, la fórmula de decaimiento de GDD #4 snapea la corrupción al floor en el primer tick sin acción del jugador — estado inválido silencioso.

---

## Dependencies

### Upstream (este sistema depende de)

| Sistema | Tipo | Interfaz |
|---------|------|----------|
| **Estado del Mundo** (GDD #4) | Hard | `WorldState.serialize_full() → Dictionary`, `WorldState.deserialize(data: Dictionary)`, `WorldState.new_game()` → DEFAULT_VALUES |
| **Base de Datos de Demonios** (GDD #3) | Hard | Validación de IDs en `equipped_demons` y `available_demons` al cargar |
| **EventBus** (ADR-002) | Hard | Señales inter-sistema — **pendiente declarar en event_bus.gd**: `game_saved`, `save_failed`, `game_loaded`, `load_failed`, `show_save_indicator` |

### Downstream (dependen de este sistema)

| Sistema | Tipo | Interfaz |
|---------|------|----------|
| **Menú Principal y Pausa** (GDD #21) | Hard | `SaveLoadSystem.save_game()`, `SaveLoadSystem.load_game()`, `SaveLoadSystem.has_save_file() → bool`; escucha `load_failed` para mostrar mensaje de save corrupto |
| **HUD** (GDD #18) | Soft | Escucha `show_save_indicator` de EventBus para mostrar ícono de guardado |

### Lateral / Trigger

| Sistema | Rol | Señal |
|---------|-----|-------|
| **SceneTree (Godot)** | Trigger de save-on-exit | `NOTIFICATION_WM_CLOSE_REQUEST` |
| **Cinemáticas** (GDD #17) | Supresión del ícono de guardado | `cinematic_started(camera_data: Dictionary)` (EventBus / ADR-002) |

### Notas de bidireccionalidad

- GDD #4 (Estado del Mundo §6.1 item 1) ya documenta que Guardado y Carga depende de él. ✅ Consistente.
- GDD #8 (Exploración del Mundo) ya no es un trigger de I/O de guardado — no hay dependencia lateral. ✅ Sin gap de bidireccionalidad.

---

## Tuning Knobs

| Parámetro | Valor MVP | Rango Seguro | Efecto si demasiado alto | Efecto si demasiado bajo |
|-----------|-----------|-------------|--------------------------|--------------------------|
| `SAVE_INDICATOR_DURATION` | 1.5 s | 0.8 – 3.0 s | El ícono molesta al jugador | El jugador no lo ve y no sabe que guardó |
| Triggers de guardado completo | 3 (save point, pausa, salida) | — | Más triggers → más I/O, sensación de "lag" | Menos triggers → mayor riesgo de pérdida de datos |

**Interacciones entre knobs:**
- `SAVE_INDICATOR_DURATION` es independiente de los demás knobs.
- El partial save (evento crítico) no tiene tuning knobs — su frecuencia es narrativa, no configurable en este sistema.

**Knobs que NO existen en MVP:**
- Tamaño máximo de archivo (no aplica — el JSON no supera ~50 KB; la serialización via `JSON.stringify()` es el mayor coste de CPU, no el tamaño en disco)
- Frecuencia de auto-save por tiempo (no implementada — post-MVP)
- Número de slots de guardado (fijo en 1 para MVP)

---

## Visual/Audio Requirements

No aplica para este sistema. El único feedback visible (ícono de guardado `SAVE_INDICATOR_DURATION`) es propiedad del HUD (GDD #18) y se dispara por la señal `show_save_indicator`.

---

## UI Requirements

No aplica en MVP. Este sistema no posee pantallas ni widgets propios. La UI que interactúa con él (botón "Continuar", botón "Nueva Partida", mensaje de save corrupto) pertenece a GDD #21. El mensaje de error es responsabilidad de GDD #21, que lo muestra al recibir `load_failed`.

---

## Acceptance Criteria

### Bloque A — Core Rules

**CA-SL-001** — GIVEN que el jugador activa un save point, WHEN el sistema escribe a disco, THEN el archivo existe en `user://saves/save_data.json` y ningún otro archivo de save persiste en `user://saves/` (el archivo `.tmp` fue eliminado o renombrado).

**CA-SL-002** — GIVEN que el jugador cruza una transición de zona, WHEN el sistema recibe `zona_transition_started` del EventBus, THEN el sistema NO escribe nada a disco; el estado permanece en IDLE; ningún archivo en `user://saves/` es creado o modificado; el log no registra ninguna operación de I/O relacionada con guardado.

**CA-SL-003** — GIVEN que el jugador activa un save point [verificar por separado para "Guardar" en pausa — ambas variantes deben probarse individualmente], WHEN la operación completa, THEN el JSON contiene los 5 CRITICAL_FIELDS con los valores actuales del WorldState en ese momento.

**CA-SL-004** `[requires: harness de integración externo que lanza y cierra el proceso Godot]` — GIVEN que el jugador cierra la app normalmente vía botón de cierre de ventana, WHEN el proceso termina, THEN `user://saves/save_data.json` existe, es JSON sintácticamente válido (`JSON.parse_string()` no retorna null), y contiene los 5 CRITICAL_FIELDS. Requiere `set_auto_accept_quit(false)` en el nodo principal — no verificable con GUT headless.

**CA-SL-005** `[requires: WorldState mock/spy injectable; FileAccessWrapper injectable]` — GIVEN que ocurre un evento narrativo crítico, WHEN el sistema ejecuta el partial save, THEN: (a) el JSON resultante contiene los valores actualizados en `progression.major_events` y `narrative.player_choices`; (b) todos los demás campos en el JSON son idénticos al JSON previo al trigger; (c) el spy en WorldState registra que `serialize_full()` NO fue invocado.

**CA-SL-006** — GIVEN que el jugador está en mitad de partida, WHEN inspecciona el menú de pausa, THEN no existe opción "Cargar Partida"; solo "Volver al Menú Principal".

**CA-SL-007** — GIVEN que no existe `save_data.json`, WHEN el jugador llega al menú principal, THEN la opción "Continuar" no aparece o está deshabilitada. GIVEN que sí existe un `save_data.json` válido, THEN "Continuar" aparece habilitada e interaccionable.

---

### Bloque B — Máquina de Estados

**CA-SL-008** — GIVEN que el sistema está en IDLE, WHEN se recibe un trigger de guardado completo y el rename atómico completa con éxito, THEN el sistema transita IDLE → SAVING → IDLE; `game_saved` es emitido exactamente una vez; el estado es IDLE en el momento en que `game_saved` se emite (la transición a IDLE precede la emisión).

**CA-SL-009** `[requires: SaveLoadSystem con _state configurable por test helper]` — GIVEN que el estado es forzado a SAVING mediante `_state = SAVING` vía test helper, WHEN llega un trigger de guardado, THEN el trigger se descarta; el log registra `"save skipped — operation in progress"`; ninguna señal de guardado es emitida.

**CA-SL-010** `[requires: SaveLoadSystem con _state configurable por test helper]` — GIVEN que el estado es forzado a LOADING mediante `_state = LOADING` vía test helper, WHEN llega un trigger de guardado, THEN el trigger se descarta; el log registra `"save skipped — operation in progress"`; ninguna señal es emitida; la "carga" en curso no es interrumpida.

---

### Bloque C — Validación al Cargar

**CA-SL-011a** — GIVEN que el JSON no contiene el campo `progression`, WHEN se ejecuta `load_game()`, THEN `progression` se rellena con `DEFAULT_VALUES["progression"]`; el log registra `WARNING: "Campo 'progression' faltante en save — inicializado a default"`; el juego carga sin crash.

**CA-SL-011b** — GIVEN que el JSON no contiene el campo `demons`, WHEN se ejecuta `load_game()`, THEN `demons` se rellena con `DEFAULT_VALUES["demons"]`; el log registra `WARNING: "Campo 'demons' faltante en save — inicializado a default"`; el juego carga sin crash.

**CA-SL-011c** — GIVEN que el JSON no contiene el campo `narrative`, WHEN se ejecuta `load_game()`, THEN `narrative` se rellena con `DEFAULT_VALUES["narrative"]`; el log registra `WARNING: "Campo 'narrative' faltante en save — inicializado a default"`; el juego carga sin crash.

**CA-SL-011d** — GIVEN que el JSON no contiene el campo `world`, WHEN se ejecuta `load_game()`, THEN `world` se rellena con `DEFAULT_VALUES["world"]`; el log registra `WARNING: "Campo 'world' faltante en save — inicializado a default"`; el juego carga sin crash.

**CA-SL-011e** — GIVEN que el JSON no contiene el campo `session`, WHEN se ejecuta `load_game()`, THEN `session` se rellena con `DEFAULT_VALUES["session"]`; el log registra `WARNING: "Campo 'session' faltante en save — inicializado a default"`; el juego carga sin crash.

**CA-SL-012** — GIVEN que `equipped_demons` contiene un ID inexistente en DemonDatabase, WHEN el sistema valida el schema, THEN ese ID es removido; el log registra `WARNING: "Demonio '{id}' no existe en BD — removido de equipped_demons"`; el jugador no recibe notificación; el juego carga sin crash. Repetir para `available_demons` con mensaje `"removido de available_demons"`.

**CA-SL-013** — GIVEN que `save_data.json` existe pero su contenido no es JSON válido, WHEN el jugador selecciona "Continuar", THEN el sistema emite `load_failed` con payload `{reason: "corrupt_json"}`; registra ERROR; transita a IDLE; no intenta autocorregir el JSON; **postcondición**: el sistema acepta un nuevo intento de carga sin descartarlo (demuestra ERROR → IDLE completó). La pantalla de error visible al jugador es responsabilidad de GDD #21 que escucha `load_failed` — no se verifica en tests de SaveLoadSystem.

---

### Bloque D — Edge Cases de I/O

**CA-SL-014a** `[requires: directorio user://saves/ eliminado previamente]` — GIVEN que `user://saves/` no existe, WHEN el sistema ejecuta un trigger de guardado, THEN el directorio se crea automáticamente; `save_data.json` se crea correctamente; `game_saved` es emitido; sin crash ni error visible al jugador.

**CA-SL-014b** `[requires: mock de DirAccess inyectable]` — GIVEN que `user://saves/` no existe y `DirAccess.make_dir_recursive()` retorna error (permiso denegado, simulado), WHEN el sistema ejecuta un trigger de guardado, THEN emite `save_failed`; registra ERROR con el path; transita SAVING → ERROR → IDLE; sin crash; ningún archivo `.tmp` es creado; **postcondición**: el sistema acepta un nuevo trigger de guardado sin descartarlo.

**CA-SL-015** `[requires: FileAccessWrapper injectable con store_string() simulado]` — GIVEN que `store_string()` retorna `false` (simulado via FileAccessWrapper mock), WHEN el sistema intenta completar el guardado, THEN emite `save_failed`; registra WARNING con el path y el error; transita SAVING → ERROR → IDLE; el juego continúa sin crash; sin retry automático; **postcondición**: el sistema acepta un nuevo trigger de guardado sin descartarlo (demuestra ERROR → IDLE completó).

**CA-SL-016** — GIVEN que `has_save_file()` retorna false, WHEN se llama `load_game()` directamente (bug de UI), THEN el método retorna `null`; registra ERROR; emite `load_failed` con `{reason: "no_save_file"}`; sin crash.

**CA-SL-017** `[requires: test manual documentado en production/qa/evidence/CA-SL-017-kill-test.md; sign-off: QA Lead]` — GIVEN que existe un `save_data.json` válido de una sesión anterior con un save point activado, WHEN el proceso es terminado abruptamente (via Task Manager o `taskkill /F /IM godot.exe` en Windows), THEN al reiniciar la app, "Continuar" carga el save anterior sin corrupción; el progreso realizado después del último guardado exitoso no está presente. Protocolo mínimo: (1) iniciar partida y activar un save point; (2) realizar cambios visibles sin activar otro save; (3) forzar kill del proceso; (4) relanzar; (5) verificar "Continuar" disponible; (6) verificar que el estado corresponde al save point previo al kill; (7) documentar evidencia en el archivo especificado.

---

### Bloque E — Integración con EventBus

**CA-SL-018** — GIVEN que se completa un guardado por save point, pausa o salida del juego, WHEN el sistema termina la operación, THEN `show_save_indicator` es emitido vía EventBus. `show_save_indicator` NO es emitido en ninguno de estos casos: (a) el cruce de zona no dispara guardado; (b) guardado parcial de evento narrativo crítico (silencioso por diseño); (c) cualquier guardado durante `cinematic_started` activo.

**CA-SL-021** — GIVEN que `has_save_file()` retorna true y el JSON es válido, WHEN el sistema ejecuta `load_game()` exitosamente, THEN `game_loaded` es emitido exactamente una vez; el sistema está en estado IDLE al finalizar; `load_failed` NO es emitido.

**CA-SL-022** `[requires: WorldState mock/spy injectable; FileAccessWrapper injectable]` — GIVEN que el sistema está en IDLE, WHEN llega un trigger de evento narrativo crítico y el partial save completa con éxito, THEN el sistema transita IDLE → SAVING → IDLE; `game_saved` es emitido exactamente una vez; `show_save_indicator` NO es emitido; el estado es IDLE al finalizar; el sistema acepta un nuevo trigger inmediatamente.

---

### Bloque F — Feedback Visual `[ADVISORY]`

**CA-SL-019** `[requires: clock injection o evidencia manual]` — GIVEN que el HUD recibe `show_save_indicator`, WHEN se mide el tiempo, THEN el ícono permanece visible durante exactamente `SAVE_INDICATOR_DURATION` segundos (verificar contra el valor de la constante, no contra el literal 1.5); no bloquea gameplay durante ese tiempo.

**CA-SL-020** `[ADVISORY]` — GIVEN que `cinematic_started` está activo en EventBus (flag de supresión habilitado), WHEN ocurre un trigger de guardado completo, THEN el guardado se ejecuta y `game_saved` es emitido, pero `show_save_indicator` NO es emitido y el ícono no aparece en pantalla.

---

### Resumen de Gates

| Criterios | Tipo | Gate | Tests |
|-----------|------|------|-------|
| CA-SL-001–003, CA-SL-005, CA-SL-008–016, CA-SL-021–022 | Logic | **BLOCKING** | Tests unitarios en `tests/unit/save_load/` |
| CA-SL-004 | Integration | **BLOCKING** | Harness de integración externo (lanza/cierra proceso Godot) |
| CA-SL-006–007 | UI | ADVISORY | Walkthrough manual |
| CA-SL-017–018 | Integration | **BLOCKING** | CA-SL-017: manual documentado con sign-off; CA-SL-018: integración con EventBus mock |
| CA-SL-019–020 | Visual | ADVISORY | Evidencia manual en `production/qa/evidence/` |

> ⚠️ **Prerequisitos de implementación antes de abrir cualquier story:**
> - CA-SL-005/015/022 requieren `FileAccessWrapper` inyectable — ADR antes del sprint.
> - CA-SL-005/022 requieren `WorldState` mock/spy — confirmar patrón de inyección.
> - CA-SL-009/010 requieren test helper que exponga `_state` — diseñar en ADR.
> - Las cinco señales de save/load deben declararse en ADR-002 (`event_bus.gd`).

---

## Open Questions

- **¿FileAccess debe ser inyectable? — Patrón requerido:** CA-SL-015 requiere mockear `FileAccess`. GDScript no tiene interfaces. El patrón requerido es un `FileAccessWrapper` class:
  ```gdscript
  class_name FileAccessWrapper
  func open(path: String, flags: FileAccess.ModeFlags) -> FileAccess:
      return FileAccess.open(path, flags)
  ```
  `SaveLoadSystem` acepta un `@export var file_access: FileAccessWrapper`. En tests, se sustituye por un `MockFileAccessWrapper` que devuelve null o stubs. **ADR requerido antes del primer sprint de implementación.**
  - **Owner**: Lead Programmer / godot-gdscript-specialist
  - **Target**: Antes de sprint de implementación

- **Escritura atómica — RESUELTA:** Write-to-temp-then-rename es obligatorio para todos los saves en MVP. Añadido a Core Rule 4 y al algoritmo de partial save. No requiere decisión adicional.

- **¿Save cloud (Steam Cloud) en scope?** La ruta `user://` resuelve a `%APPDATA%\Godot\app_userdata\[project_name]\` en Windows. Si Steam Cloud es un requisito, la ruta de guardado y la integración con Steamworks Remote Storage necesitan diseño antes de cualquier SDK integration — **no se puede añadir post-implementación sin refactorizar `SaveLoadSystem`**. Evaluar con Producer y Technical Director antes de integración Steam.
  - **Owner**: Producer / Technical Director
  - **Target**: Evaluación post-MVP; confirmar antes de integración Steam
