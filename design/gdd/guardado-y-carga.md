# GDD: Guardado y Carga

> **Status**: Aprobado — 2026-06-03
> **Author**: Abel + Agentes
> **Last Updated**: 2026-06-03
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

1. **Un único slot de guardado (MVP):** El juego mantiene un archivo de save activo (`user://saves/save_data.json`) y un backup rotado de la sesión anterior (`user://saves/save_data.bak`). No hay múltiples slots de progreso independientes. Si `save_data.json` existe al iniciar, el menú principal ofrece "Continuar". Si no existe, solo "Nueva Partida". El `.bak` es un fallback interno de recuperación, no una funcionalidad expuesta al jugador.

2. **Directorio de saves:** Al guardar por primera vez, el sistema verifica que `user://saves/` existe. Si no, lo crea:
   ```gdscript
   var dir := DirAccess.open("user://")
   dir.make_dir_recursive("saves")
   ```
   **No** usar `make_dir_recursive("user://saves/")` como método estático (no lo es en Godot 4.x — requiere instancia), ni `make_dir_recursive_absolute()` (interpreta el argumento como ruta del SO). El directorio nunca se elimina automáticamente.

3. **Triggers de guardado:**

   | Trigger | Método | Contenido guardado |
   |---------|--------|--------------------|
   | Save point activado | `WorldState.serialize_full()` | Estado completo |
   | "Guardar" en menú de pausa | `WorldState.serialize_full()` | Estado completo |
   | Salida del juego | `WorldState.serialize_full()` | Estado completo |
   | Evento narrativo crítico | Patch quirúrgico directo | Solo `progression.major_events` + `narrative.player_choices` |

   El cruce de zona/área **no dispara ninguna escritura a disco**. El `WorldState` se actualiza en memoria al cruzar zonas, pero sin I/O de archivo. El siguiente guardado completo (save point, pausa, o salida) capturará ese estado actualizado.

4. **Patrón de escritura atómica — obligatorio para todos los saves completos:** Toda escritura a disco usa write-to-temp-then-rename. Para saves completos se rota el backup antes de escribir:
   ```
   0. Abrir DirAccess sobre el directorio de saves:
      var dir := DirAccess.open("user://saves/")
      Si dir == null → emitir save_failed; retornar
   1. [Backup] Si save_data.json existe:
      var backup_err := dir.rename("save_data.json", "save_data.bak")
      Si backup_err != OK → registrar WARNING "backup fallido — continuando sin backup"
   2. Serializar el Dictionary a JSON string
   3. Abrir user://saves/save_data.json.tmp en modo WRITE con FileAccess.open()
      Si open() retorna null → emitir save_failed; retornar
   4. file_access.store_string_in(file, json_string)
      Si retorna false → emitir save_failed; dir.remove("save_data.json.tmp"); retornar
   5. var rename_err := dir.rename("save_data.json.tmp", "save_data.json")
      Si rename_err != OK → emitir save_failed; dir.remove("save_data.json.tmp"); retornar
   ```
   `FileAccess.WRITE` trunca el archivo al abrirlo, antes de escribir ningún dato. El rename en el paso 5 devuelve `Error` (no `bool`) — la comprobación de fallo es `!= OK`. Llamar `DirAccess.rename()` como método estático o comprobar falsy produce comportamiento incorrecto (OK == 0 == falsy en GDScript). El rename es atómico cuando source y destino están en el mismo filesystem (ambos bajo `user://saves/` siempre lo están). El backup en paso 1 da al jugador una sesión de fallback ante corrupción lógica post-patch.

5. **Guardado parcial (evento crítico):** El sistema lee `save_data.json`, parsea el JSON, actualiza en memoria solo `progression.major_events` y `narrative.player_choices` relevantes, y aplica el patrón write-to-temp-then-rename para reescribir el archivo completo. No llama a `serialize_full()` — es una actualización quirúrgica para proteger datos en momentos de alta tensión narrativa. Si el parseo del archivo existente falla, el sistema emite `save_failed` y no sobreescribe el archivo.

6. **Feedback visual:** Los guardados por save point, menú de pausa y salida del juego muestran un ícono de guardado en pantalla durante `SAVE_INDICATOR_DURATION` (1.5 s). El ícono no bloquea el gameplay. Los guardados de **evento narrativo crítico son silenciosos** (no muestran el ícono) para preservar la inmersión narrativa en momentos de alta carga emocional. Si hay una cinemática activa en el momento del guardado, el ícono también se suprime.

7. **Carga:** Solo ocurre al iniciar desde el menú principal ("Continuar"). No hay carga en mitad de partida (sin "load save" desde pausa en MVP — solo "Volver al Menú Principal" que hace reset de la sesión).

8. **Validación al cargar:** Antes de deserializar, el sistema verifica que el JSON tiene los campos críticos (`progression`, `demons`, `narrative`, `world`, `session`, `companion_state`). Campos faltantes → valor por defecto (`companion_state` inicializa a `{}`). Demonio en `equipped_demons` o `available_demons` que no existe en la BD → removido silenciosamente (previa verificación de que el campo existe y es un Array). Sub-campos `narrative.corruption_level` y `narrative.corruption_floor` faltantes dentro de `narrative` → inicializados a `0.0`. `narrative.corruption_level` fuera de [0.0, 1.0] → clampeado. `narrative.corruption_floor` fuera de [0.0, 1.0] → clampeado independientemente. `narrative.corruption_floor > narrative.corruption_level` → `corruption_floor` ajustado al valor de `corruption_level`. El juego continúa sin crash.

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

**Excepción — salida del juego:** `NOTIFICATION_WM_CLOSE_REQUEST` nunca se descarta ni se loggea como "skipped". Si llega durante SAVING, el handler **espera a que el save actual complete** (via `await EventBus.game_saved` o `await EventBus.save_failed`) antes de ejecutar el save-on-exit completo y llamar `get_tree().quit()`. Esto garantiza que ningún evento narrativo crítico en vuelo se pierda al cerrar la app.

---

### Interactions with Other Systems

| Sistema | Dirección | Interfaz |
|---------|-----------|----------|
| **WorldState** (GDD #4) | CALL | `WorldState.serialize_full() → Dictionary`, `WorldState.deserialize(data: Dictionary)` |
| **EventBus** (ADR-002) | EMIT | `game_saved`, `save_failed`, `game_loaded`, `load_failed`, `show_save_indicator` |
| **Menú Principal y Pausa** (GDD #21) | RECEIVE CALLS | `SaveLoadSystem.save_game()`, `SaveLoadSystem.load_game()`, `SaveLoadSystem.has_save_file() → bool` |
| **Application / SceneTree** | LISTEN + CONFIG | `NOTIFICATION_WM_CLOSE_REQUEST` → dispara save-on-exit; **requiere** `get_tree().set_auto_accept_quit(false)` en `_ready()` + `get_tree().quit()` manual tras completar el save. Si el sistema está en SAVING cuando llega la notificación, el handler espera (`await EventBus.game_saved` / `await EventBus.save_failed`) antes de ejecutar el save-on-exit — el cierre nunca descarta un save en vuelo. |
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
| `SAVE_BACKUP_PATH` | String | `"user://saves/save_data.bak"` | Backup del save anterior; rotado en cada guardado completo antes del overwrite. |
| `CRITICAL_FIELDS` | Set | `{progression, demons, narrative, world, session, companion_state}` | Campos que deben existir en el JSON para que el save sea válido. |

---

**Algoritmo de partial save (evento narrativo crítico):**

```
# Guard de early-return: sin cambios pendientes no hay nada que escribir
Si major_events_changed.is_empty() y player_choices_changed.is_empty():
    retornar

# Abrir el archivo existente — null significa que no existe aún
var file_handle = FileAccess.open(SAVE_FILE_PATH, FileAccess.READ)
Si file_handle == null:
    emitir save_failed (baja severidad)
    registrar WARNING: "partial save: save_data.json no existe — evento queda en memoria"
    retornar  # el evento persiste en WorldState; el próximo guardado completo lo capturará

var json_text := file_handle.get_as_text()
file_handle = null  # cerrar antes de reescribir

Sea partial_data = JSON.parse_string(json_text)
Si partial_data == null:
    emitir save_failed; registrar ERROR "partial save: JSON inválido"; retornar

# Guards de sub-campo — el schema puede no tener la clave si el save es de versión anterior
Si not partial_data.has("progression"):
    partial_data["progression"] = {}
Si not partial_data["progression"].has("major_events"):
    partial_data["progression"]["major_events"] = {}

Para cada evento_id en major_events_changed:
    partial_data["progression"]["major_events"][evento_id] =
        WorldState.get_major_event(evento_id)

Si not partial_data.has("narrative"):
    partial_data["narrative"] = {}
Si not partial_data["narrative"].has("player_choices"):
    partial_data["narrative"]["player_choices"] = {}

Para cada choice_id en player_choices_changed:
    partial_data["narrative"]["player_choices"][choice_id] =
        WorldState.get_player_choice(choice_id)

Aplicar write-to-temp-then-rename con JSON.stringify(partial_data)
(el partial save NO rota save_data.bak — el backup solo se rota en guardados completos)
```

**Rutas canónicas:** El partial save actualiza `progression.major_events` y `narrative.player_choices`. Estas son las dos rutas exactas definidas en el schema de GDD #4. No existe un campo `player_choices` bajo `progression`.

---

**Algoritmo de validación de schema al cargar:**

```
Sea CRITICAL_FIELDS = {progression, demons, narrative, world, session, companion_state}
Sea DEFAULT_VALUES = {progression: {...}, demons: {...}, narrative: {...},
                      world: {...}, session: {...}, companion_state: {}}

Para cada campo f en CRITICAL_FIELDS:
    Si f NO está en el JSON cargado:
        json[f] = DEFAULT_VALUES[f]
        registrar WARNING: "Campo '{f}' faltante en save — inicializado a default"

# Validación de IDs de demonios — verificar existencia y tipo antes de iterar
Si not json["demons"].has("equipped_demons") o not json["demons"]["equipped_demons"] is Array:
    json["demons"]["equipped_demons"] = []
    registrar WARNING: "equipped_demons faltante o tipo incorrecto — inicializado a []"

Para cada demonio_id en json["demons"]["equipped_demons"].duplicate():
    Si demonio_id NO existe en DemonDatabase:
        json["demons"]["equipped_demons"].erase(demonio_id)
        registrar WARNING: "Demonio '{demonio_id}' no existe en BD — removido de equipped_demons"

Si not json["demons"].has("available_demons") o not json["demons"]["available_demons"] is Array:
    json["demons"]["available_demons"] = []
    registrar WARNING: "available_demons faltante o tipo incorrecto — inicializado a []"

Para cada demonio_id en json["demons"]["available_demons"].duplicate():
    Si demonio_id NO existe en DemonDatabase:
        json["demons"]["available_demons"].erase(demonio_id)
        registrar WARNING: "Demonio '{demonio_id}' no existe en BD — removido de available_demons"

# Validación de sub-campos de corrupción — verificar existencia antes de acceder
Si not json["narrative"].has("corruption_level"):
    json["narrative"]["corruption_level"] = 0.0
    registrar WARNING: "corruption_level faltante en narrative — inicializado a 0.0"
Si not json["narrative"].has("corruption_floor"):
    json["narrative"]["corruption_floor"] = 0.0
    registrar WARNING: "corruption_floor faltante en narrative — inicializado a 0.0"

# Validación de invariantes de corrupción
Si json["narrative"]["corruption_level"] < 0.0 o > 1.0:
    json["narrative"]["corruption_level"] =
        clamp(json["narrative"]["corruption_level"], 0.0, 1.0)
    registrar WARNING: "corruption_level fuera de rango — corregido"

Si json["narrative"]["corruption_floor"] < 0.0 o > 1.0:
    json["narrative"]["corruption_floor"] =
        clamp(json["narrative"]["corruption_floor"], 0.0, 1.0)
    registrar WARNING: "corruption_floor fuera de rango — corregido"

Si json["narrative"]["corruption_floor"] > json["narrative"]["corruption_level"]:
    json["narrative"]["corruption_floor"] = json["narrative"]["corruption_level"]
    registrar WARNING: "corruption_floor > corruption_level — ajustado"
```

**Variables:**

| Variable | Tipo | Descripción |
|----------|------|-------------|
| `SAVE_BACKUP_PATH` | String | Ruta del backup (`user://saves/save_data.bak`) |
| `CRITICAL_FIELDS` | Set[String] | Campos cuya ausencia activa autocorrección |
| `DEFAULT_VALUES` | Dictionary | Valores iniciales por campo; `companion_state` inicializa a `{}`; el resto es idéntico a `WorldState.new_game()` |
| `DemonDatabase` | Resource | Base de Datos de Demonios (GDD #3) — fuente de IDs válidos |

---

**Notas de implementación — requeridas en ADR antes del primer sprint:**

1. **API correcta de JSON (Godot 4.x):** No existe `JSON.parse(text)` como método estático en Godot 4 — es API de Godot 3 y lanzará un error en runtime. Usar `JSON.parse_string(text)` (retorna `null` en fallo) o `var j = JSON.new(); j.parse(text); j.get_data()`. Para serializar: `JSON.stringify(dict)` (estático — correcto en Godot 4).

2. **API correcta de directorio (Godot 4.x) — métodos de instancia:** `DirAccess.make_dir_recursive()` y `DirAccess.rename()` son **métodos de instancia**, no estáticos. Llamarlos como métodos estáticos produce un error en runtime. `DirAccess.rename()` devuelve `Error` (no `bool`) — la comprobación de fallo debe ser `!= OK`. Usar `if not dir.rename(...)` trataría `OK` (valor 0, falsy en GDScript) como fallo, marcando cada save exitoso como error. Patrones correctos:
   ```gdscript
   # Crear directorio
   var dir := DirAccess.open("user://")
   dir.make_dir_recursive("saves")

   # Rename atómico
   var dir2 := DirAccess.open("user://saves/")
   var err := dir2.rename("save_data.json.tmp", "save_data.json")
   if err != OK:
       EventBus.save_failed.emit()
       dir2.remove("save_data.json.tmp")
       return
   ```

3. **Null-check en FileAccess.open():** En Godot 4, `FileAccess.open()` retorna `null` en fallo (no lanza excepción). Sin null-check, llamar `store_string()` sobre null produce un crash, no una emisión limpia de `save_failed`. Patrón correcto:
   ```gdscript
   var file = FileAccess.open(path, FileAccess.WRITE)
   if file == null:
       EventBus.save_failed.emit()
       return
   ```

4. **set_auto_accept_quit(false) para save-on-exit:** El nodo principal debe llamar `get_tree().set_auto_accept_quit(false)` en `_ready()`. Sin esto, Godot cierra el proceso al recibir `NOTIFICATION_WM_CLOSE_REQUEST` antes de que el save síncrono complete. El handler debe llamar `get_tree().quit()` manualmente tras el save.

5. **Orden de transición de estado:** `_state = IDLE` debe ejecutarse ANTES de `EventBus.game_saved.emit()`. Godot despacha señales síncronamente; si un listener de `game_saved` dispara otro save, el sistema debe estar en IDLE para procesarlo.

6. **Inyección de FileAccess (requerido para CA-SL-015 y CA-SL-023):** `FileAccess` es una clase nativa de Godot y no puede ser subclaseada en GDScript. El wrapper debe envolver tanto `open()` como `store_string_in()` para que los tests puedan simular tanto fallo de apertura como fallo de escritura. Ver Open Questions §1 para el patrón concreto actualizado.

7. **Espera bloqueante en save-on-exit durante SAVING (requerido para CA-SL-004):** Si `NOTIFICATION_WM_CLOSE_REQUEST` llega mientras `_state == SAVING`, el handler debe esperar antes de ejecutar el save-on-exit. En GDScript, `await` sobre una señal del EventBus es la opción más limpia — pero requiere await sobre cualquiera de las dos señales terminadoras. Patrón de referencia:
   ```gdscript
   func _notification(what: int) -> void:
       if what == NOTIFICATION_WM_CLOSE_REQUEST:
           if _state == State.SAVING:
               await EventBus.game_saved  # espera a que el save en vuelo complete
           save_game()
           await EventBus.game_saved
           get_tree().quit()
   ```
   Nota: el await sobre `game_saved` solo es correcto si el save previo emitió esa señal. Si emitió `save_failed`, el await anterior no se habría liberado — en la implementación real se debe esperar cualquiera de las dos señales (game_saved o save_failed). Diseñar el patrón exacto en ADR.

8. **`SaveLoadSystem` como Autoload vs nodo de escena:** Debe especificarse en ADR antes del primer sprint. Si es un Autoload, `get_tree().set_auto_accept_quit(false)` puede llamarse en `_ready()`. Si es un nodo hijo de escena, solo puede llamarse después de que el nodo entre en el árbol. Esta decisión afecta la arquitectura de todos los tests de integración.

---

## Edge Cases

- **Si el directorio `user://saves/` no existe al intentar guardar:** El sistema lo crea con `DirAccess.open("user://").make_dir_recursive("saves")` antes de abrir el archivo temporal. Si la creación falla (permiso denegado), el sistema emite `save_failed`, registra el error, y continúa sin crash.

- **Si `FileAccess.open()` retorna null al abrir el archivo temporal (bloqueado o disco lleno):** El sistema emite `save_failed` vía EventBus, registra WARNING con el path y el error. No hay retry. El archivo `save_data.json` original no es modificado.

- **Si el write-to-temp completa pero el rename falla:** `dir.rename("save_data.json.tmp", "save_data.json")` retorna `!= OK`. El sistema emite `save_failed`, registra ERROR, e intenta borrar `save_data.json.tmp` con `dir.remove("save_data.json.tmp")`. `save_data.json` permanece intacto con el estado del último save exitoso. Si se rotó `save_data.bak` antes del write, el backup también está disponible como fallback adicional.

- **Si el archivo de save no existe al intentar cargar ("Continuar" sin partida previa):** `has_save_file()` retorna `false`. El menú principal no muestra "Continuar". Si se llama `load_game()` de todas formas (bug de UI), el sistema registra ERROR y retorna `null` — el llamador debe manejar `null` iniciando nueva partida.

- **Si el JSON cargado es sintácticamente inválido:** `JSON.parse_string()` retorna `null`. El sistema emite `load_failed` con payload `{reason: "corrupt_json"}`, registra ERROR, y NO carga el save. El menú principal (GDD #21) escucha `load_failed` y muestra el error al jugador: *"El archivo de guardado está dañado. No se puede continuar."*

- **Si un trigger de guardado llega mientras el sistema está en SAVING o LOADING:** El trigger se descarta. Se registra: `"save skipped — operation in progress"`. Si el trigger descartado era un evento narrativo crítico, el evento persiste en `WorldState` en memoria — el próximo guardado completo lo capturará. **Excepción: `NOTIFICATION_WM_CLOSE_REQUEST` nunca se descarta** — ver §States and Transitions y Nota de implementación 7.

- **Si ocurre un evento narrativo crítico antes de cualquier guardado completo (primera sesión):** El partial save intenta abrir `save_data.json`, que no existe aún — `FileAccess.open()` retorna `null`. El sistema emite `save_failed` de baja severidad, registra WARNING, y retorna sin escribir nada. El evento narrativo ya está en `WorldState` en memoria; el siguiente guardado completo (save point, pausa o salida) lo persistirá. GDD #21 **no debe mostrar** este `save_failed` al jugador como error visible — es una condición esperada en la primera sesión.

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

**CA-SL-003** `[requires: WorldState spy injectable]` — GIVEN que el jugador activa un save point [verificar por separado para "Guardar" en pausa — ambas variantes deben probarse individualmente], WHEN la operación completa, THEN: (a) el JSON en disco contiene los 6 CRITICAL_FIELDS; (b) para cada campo `f` en CRITICAL_FIELDS, `json[f]` es estructuralmente igual (deep equality de Dictionary, no comparación de strings JSON) al valor capturado por el spy desde `WorldState.serialize_full()` inmediatamente antes del trigger de guardado. El spy debe capturar el snapshot antes del trigger, no después.

**CA-SL-004** `[requires: harness de integración externo que lanza y cierra el proceso Godot]` — GIVEN que el jugador cierra la app normalmente vía botón de cierre de ventana, WHEN el proceso termina, THEN `user://saves/save_data.json` existe, es JSON sintácticamente válido (`JSON.parse_string()` no retorna null), y contiene los 6 CRITICAL_FIELDS. Requiere `set_auto_accept_quit(false)` en el nodo principal — no verificable con GUT headless.

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

**CA-SL-015** `[requires: FileAccessWrapper injectable con store_string_in() simulado]` — GIVEN que `file_access.store_string_in()` retorna `false` (simulado via FileAccessWrapper mock), WHEN el sistema intenta completar el guardado, THEN emite `save_failed`; registra WARNING con el path y el error; transita SAVING → ERROR → IDLE; el juego continúa sin crash; sin retry automático; **postcondición**: el sistema acepta un nuevo trigger de guardado sin descartarlo (demuestra ERROR → IDLE completó).

**CA-SL-016** — GIVEN que `has_save_file()` retorna false, WHEN se llama `load_game()` directamente (bug de UI), THEN el método retorna `null`; registra ERROR; emite `load_failed` con `{reason: "no_save_file"}`; sin crash.

**CA-SL-017** `[requires: test manual documentado en production/qa/evidence/CA-SL-017-kill-test.md; sign-off: QA Lead]` — GIVEN que existe un `save_data.json` válido de una sesión anterior con un save point activado, WHEN el proceso es terminado abruptamente (via Task Manager o `taskkill /F /IM godot.exe` en Windows), THEN al reiniciar la app, "Continuar" carga el save anterior sin corrupción; el progreso realizado después del último guardado exitoso no está presente; si existía `save_data.bak`, también carga sin corrupción. Protocolo mínimo: (1) iniciar partida y activar un save point; (2) realizar cambios visibles sin activar otro save; (3) forzar kill del proceso; (4) relanzar; (5) verificar "Continuar" disponible; (6) verificar que el estado corresponde al save point previo al kill; (7) documentar evidencia en el archivo especificado.

---

### Bloque E — Integración con EventBus

**CA-SL-018a** — GIVEN que se completa un guardado por save point, pausa o salida del juego, WHEN el sistema termina la operación exitosamente, THEN `show_save_indicator` es emitido vía EventBus exactamente una vez.

**CA-SL-018b** `[unit]` — GIVEN que el sistema recibe `zona_transition_started` del EventBus, WHEN procesa el evento, THEN `show_save_indicator` NO es emitido; el sistema permanece en IDLE.

**CA-SL-018c** `[requires: WorldState mock/spy injectable]` — GIVEN que ocurre un evento narrativo crítico y el partial save completa, WHEN el sistema termina la operación, THEN `game_saved` es emitido y `show_save_indicator` NO es emitido. (El guardado parcial es silencioso por diseño.)

**CA-SL-018d** `[requires: cinematic_started flag activo vía EventBus]` — GIVEN que el flag de supresión está activo (sistema ha recibido `cinematic_started`) y ocurre un trigger de guardado completo, WHEN el sistema completa el guardado, THEN `game_saved` es emitido pero `show_save_indicator` NO es emitido.

**CA-SL-021** — GIVEN que `has_save_file()` retorna true y el JSON es válido, WHEN el sistema ejecuta `load_game()` exitosamente, THEN `game_loaded` es emitido exactamente una vez; el sistema está en estado IDLE al finalizar; `load_failed` NO es emitido.

**CA-SL-022** `[requires: WorldState mock/spy injectable; FileAccessWrapper injectable]` — GIVEN que el sistema está en IDLE, WHEN llega un trigger de evento narrativo crítico y el partial save completa con éxito, THEN el sistema transita IDLE → SAVING → IDLE; `game_saved` es emitido exactamente una vez; `show_save_indicator` NO es emitido; el estado es IDLE al finalizar; el sistema acepta un nuevo trigger inmediatamente.

---

### Bloque E — Nuevos ACs (correcciones de revisión 2026-06-03)

**CA-SL-023** `[requires: FileAccessWrapper injectable con rename() simulado via DirAccess mock]` — GIVEN que `file_access.store_string_in()` completa con éxito pero `dir.rename("save_data.json.tmp", "save_data.json")` retorna `!= OK` (simulado), WHEN el sistema intenta completar el guardado, THEN emite `save_failed`; registra ERROR; intenta borrar `save_data.json.tmp`; `save_data.json` permanece intacto; transita SAVING → ERROR → IDLE; **postcondición**: el sistema acepta un nuevo trigger de guardado sin descartarlo.

**CA-SL-024** `[requires: FileAccessWrapper injectable — primera sesión, sin save_data.json previo]` — GIVEN que `save_data.json` NO existe y ocurre un evento narrativo crítico, WHEN el sistema intenta ejecutar el partial save, THEN emite `save_failed` (baja severidad); registra WARNING; NO escribe ningún archivo a disco; transita SAVING → ERROR → IDLE; el evento sigue disponible en `WorldState` en memoria; el sistema acepta un nuevo trigger de guardado completo sin descartarlo.

**CA-SL-025** `[requires: save con narrative.corruption_floor ausente dentro de narrative presente]` — GIVEN que el JSON cargado contiene `narrative` como Dictionary pero sin la sub-clave `corruption_floor`, WHEN se ejecuta `load_game()`, THEN `corruption_floor` se inicializa a `0.0`; el log registra `WARNING: "corruption_floor faltante en narrative — inicializado a 0.0"`; la carga completa sin crash; `corruption_level` no es modificado por la ausencia de `corruption_floor`.

---

### Bloque F — Feedback Visual `[ADVISORY]`

**CA-SL-019** `[requires: clock injection o evidencia manual]` — GIVEN que el HUD recibe `show_save_indicator`, WHEN se mide el tiempo, THEN el ícono permanece visible durante exactamente `SAVE_INDICATOR_DURATION` segundos (verificar contra el valor de la constante, no contra el literal 1.5); no bloquea gameplay durante ese tiempo.

**CA-SL-020** `[ADVISORY]` — GIVEN que `cinematic_started` está activo en EventBus (flag de supresión habilitado), WHEN ocurre un trigger de guardado completo, THEN el guardado se ejecuta y `game_saved` es emitido, pero `show_save_indicator` NO es emitido y el ícono no aparece en pantalla.

---

### Resumen de Gates

| Criterios | Tipo | Gate | Tests |
|-----------|------|------|-------|
| CA-SL-001–003, CA-SL-005, CA-SL-008–016, CA-SL-021–025 | Logic | **BLOCKING** | Tests unitarios en `tests/unit/save_load/` |
| CA-SL-004 | Integration | **BLOCKING** | Harness de integración externo (lanza/cierra proceso Godot) |
| CA-SL-006–007 | UI | ADVISORY | Walkthrough manual |
| CA-SL-017, CA-SL-018a | Integration | **BLOCKING** | CA-SL-017: manual documentado con sign-off; CA-SL-018a: integración con EventBus mock |
| CA-SL-018b–018d | Logic | **BLOCKING** | Tests unitarios en `tests/unit/save_load/` |
| CA-SL-019–020 | Visual | ADVISORY | Evidencia manual en `production/qa/evidence/` |

> ⚠️ **Prerequisitos de implementación antes de abrir cualquier story:**
> - CA-SL-003/005/015/022/023/024 requieren `FileAccessWrapper` extendido (proxy de `open()` y `store_string_in()`) — ADR antes del sprint.
> - CA-SL-003/005/022/024 requieren `WorldState` mock/spy — confirmar patrón de inyección en ADR.
> - CA-SL-009/010 requieren test helper que exponga `_state` — diseñar en ADR.
> - Las **seis señales** de save/load deben declararse en ADR-002: `game_saved`, `save_failed`, `game_loaded`, `load_failed`, `show_save_indicator`, y confirmar `cinematic_started` ya declarada.
> - `SaveLoadSystem` debe especificarse como Autoload o nodo de escena en ADR antes del primer sprint.

---

## Open Questions

- **¿FileAccess debe ser inyectable? — Patrón requerido (ACTUALIZADO post-revisión 2026-06-03):** CA-SL-015 y CA-SL-023 requieren mockear tanto `open()` como `store_string_in()`. `FileAccess` es una clase nativa — no subclaseable en GDScript. El wrapper debe envolver ambas operaciones:
  ```gdscript
  class_name FileAccessWrapper
  func open(path: String, flags: FileAccess.ModeFlags) -> FileAccess:
      return FileAccess.open(path, flags)
  func store_string_in(file: FileAccess, text: String) -> bool:
      return file.store_string(text)
  ```
  `SaveLoadSystem` acepta un `@export var file_access: FileAccessWrapper`. En tests GUT, se asigna manualmente antes de llamar al método: `save_system.file_access = MockFileAccessWrapper.new()`. El mock puede devolver `null` de `open()` (fallo de apertura) o `false` de `store_string_in()` (fallo de escritura). **ADR requerido antes del primer sprint de implementación.**
  - **Owner**: Lead Programmer / godot-gdscript-specialist
  - **Target**: Antes de sprint de implementación

- **Escritura atómica + backup — RESUELTA:** Write-to-temp-then-rename es obligatorio. Los saves completos incluyen rotación de `save_data.bak` antes de cada overwrite (paso 1 del algoritmo). El partial save no rota backup — opera quirúrgicamente sobre el JSON existente. No requiere decisión adicional.

- **¿Save cloud (Steam Cloud) en scope?** La ruta `user://` resuelve a `%APPDATA%\Godot\app_userdata\[project_name]\` en Windows. Si Steam Cloud es un requisito, la ruta de guardado y la integración con Steamworks Remote Storage necesitan diseño antes de cualquier SDK integration — **no se puede añadir post-implementación sin refactorizar `SaveLoadSystem`**. Evaluar con Producer y Technical Director antes de integración Steam.
  - **Owner**: Producer / Technical Director
  - **Target**: Evaluación post-MVP; confirmar antes de integración Steam
