# GDD: Estado del Mundo

> **Estado**: En Revisión — Post Decisiones de Diseño (2026-05-25)
> **Creado**: 2026-05-25
> **Sistema**: Estado del Mundo (World State)
> **Milestone**: MVP — Foundation Layer
> **Depende de**: — (sin dependencias)
> **Dependen de este sistema**: Guardado y Carga, Transformación Visual, Vinculación de Demonios, NPC y Diálogo, Progresión Narrativa, Seguimiento Moral, Restricción por Demonio
> **Decisiones de Diseño Aplicadas**:
>   - Corrupción reversible (arco moral gris, permite redención)
>   - El Gato deferido a GDD posterior (fuera de Foundation)
>   - NPCs pre-poblados en inicialización (seguro para queries narrativas)

---

## 1. Visión General

El Estado del Mundo es el **registro centralizado de toda la información que persiste** entre sesiones de juego. Incluye: qué áreas el jugador ha visitado y explorado, qué demonios ha vinculado (y cuál está equipado actualmente), qué NPCs ha encontrado y qué diálogos ha visto, qué decisiones morales ha tomado, qué eventos narrativos se han disparado, y cualquier dato que afecte la reactividad del mundo (restricciones de acceso, disponibilidad de NPCs, narrativa alternativa).

En MVP, el Estado del Mundo es **relativamente simple**: trackea los 3 reinos visitables, ~5-7 NPCs principales, los 7 demonios MVP, decisiones binarias de cada encuentro importante (¿ejecutaste al NPC X o lo dejaste ir?), e indicadores narrativos clave (¿ha ocurrido un evento crítico?). No hay economía compleja, no hay inventario de cientos de items — solo lo esencial para que el mundo sienta "vivo y reactivo".

El Estado del Mundo sirve como la **base de datos viva** que otros sistemas consultan: 
- **Loadout & Build Management** lee qué demonios están disponibles (desbloqueados pero no necesariamente equipados)
- **NPC y Diálogo** consulta qué eventos han ocurrido para decidir cuál rama de diálogo mostrar
- **Transformación Visual** lee el nivel de corrupción moral global para ajustar el aspecto de Edrick
- **Seguimiento Moral** acumula puntos de corrupción basados en acciones registradas aquí

El Estado del Mundo es **agnóstico de plataforma**: se serializa a un archivo de texto (JSON, YAML o formato binario) que se guarda en el disco. El sistema no implementa cifrado, control de versiones, o detección de manipulación en MVP — son post-launch concerns.

---

## 2. Fantasy del Jugador

Cuando Edrick juega y toma decisiones — cuál demonio equipar, si ejecuta o perdona a un enemigo, qué NPC ayuda — el jugador debería sentir: **Progreso real** — las decisiones se recuerdan y quedan; no es un loop sin significado. **Mundo vivo y reactivo** — los NPCs recuerdan lo que hiciste; las áreas se vuelven inaccesibles si tomaste la ruta equivocada; los diálogos cambian basado en tu reputación. **Libertad narrativa** — no hay una "ruta correcta"; dos playthroughs pueden ser completamente diferentes. **Peso moral** — tomar decisiones oscuras tiene un coste (corrupción visual, diálogos de personajes molestos, restricciones), pero es tu elección. **Continuidad cinematográfica** — cuando guardas y regresas, la cinematografía del mundo refleja TUS decisiones, no reinicia a un estado por defecto.

---

## 3. Reglas Detalladas

### 3.1 Estructura del Estado del Mundo

El Estado del Mundo se almacena en una estructura de datos única (usualmente JSON o YAML) con estos campos principales:

```
world_state:
  metadata:
    version: "1.0"
    last_save_timestamp: float (segundos desde epoch)
    playtime_seconds: float
  
  progression:
    current_kingdom: string (id: "reino_1", "reino_2", "reino_3")
    current_checkpoint: string (id del checkpoint más reciente)
    acts_completed: list[string] (ej: ["acto_1_completo"])
    major_events: dict[string, int] (fase actual del evento: 0 = no iniciado, 1..N = fase N completada; ej: {"evento_la_voz_del_gato": 0, "evento_encuentro_draeven": 0, "cat_reveal": 0})
  
  demons:
    equipped_demons: list[string] (ids de demonios ACTUALMENTE en el loadout, 3-5 demonios, ej: ["fuego", "hielo", "arcano"]. Estos son libremente equipables/desequipables.)
    available_demons: list[string] (ids de TODOS los demonios que el jugador ha obtenido en algún momento narrativo, ej: ["fuego", "hielo", "arcano", "visión", "mente", "dash"]. Este es histórico — nunca decrece.)
    demon_saturation: dict[string, float] (saturación visual acumulada por demonio, rango 0.0-1.0 — se congela cuando se desactiva del loadout)
  
  companion_state: dict (estado del compañero permanente — será definido completamente por Vinculación GDD #13. Incluirá: bonding_status, relationship_history, abilities, etc. Inicialmente vacío {}; poblado cuando el gato es encontrado narrativamente.)
  
  narrative:
    corruption_level: float (rango 0.0-1.0; ver §4.1 para 3 fuentes de cambio y §4.1.E para mapeo visual de 5 estados)
    corruption_floor: float (rango 0.0-1.0, monotónicamente creciente — mínimo permanente al que corruption_level puede decaer. Sube con actos narrativos oscuros. Ver §4.1.D)
    npc_encounters: dict[string, dict] (por NPC: {"nombre_npc": {"met": bool, "alive": bool, "dialogue_branches_seen": list[string], "reputation": float}})
    player_choices: dict[string, dict] (decisiones con contexto narrativo; ej: {"ejecutar_bandido_x": {"value": "ejecutado", "act": 1, "timestamp": 12345.0, "conscious": true}})
  
  world:
    areas_visited: dict[string, dict] (por área: {"reino_1_foresta": {"visited": bool, "explored_percent": 0-100, "enemies_defeated": int}})
    doors_unlocked: list[string] (ids de puertas/accesos desbloqueados)
    collectibles_found: list[string] (items/secretos encontrados)
    checkpoint_reached: string (id del checkpoint más reciente para respawn)
  
  session:
    current_hp: int (HP actual de Edrick en esta sesión — se pierde al cargar)
    current_position: dict {"x": float, "y": float} (posición actual — se resetea al cargar)
```

**Nota**: Los campos en la sección `session` se resetean al cargar. Los campos fuera de `session` persisten permanentemente.

### 3.2 Inicialización

Cuando comienza un nuevo juego:
1. Se crea una estructura de Estado del Mundo con todos los valores iniciales
2. `equipped_demons` = [] (VACÍO — Edrick no tiene demonios en el loadout aún)
3. `available_demons` = [] (vacío — Edrick no ha obtenido demonios aún)
3b. `demon_saturation` = {} (diccionario vacío; se poblará con entradas cuando demonios se equipes)
4. `current_kingdom` = "reino_1" (comienza en el reino 1)
5. `corruption_level` = 0.0 (sin corrupción)
5b. `corruption_floor` = 0.0 (sin actos oscuros previos — floor irrecuperable inicial en cero)
6. **NPCs pre-poblados**: Se crea una entrada en `npc_encounters` para CADA NPC del juego (todos los ~5-7 NPCs MVP) con valores: `met: false`, `alive: true`, `reputation: 0.0`, `dialogue_branches_seen: []`
7. Todos los eventos narrativos tienen fase `0` (no iniciado)
8. Todos los checkpoint y áreas se inicializan con estado por defecto

**Nota crítica**: Los NPCs están PRE-poblados en el diccionario `npc_encounters` desde el inicio. Esto garantiza que sistemas narrativos pueden consultar `world_state.npc_encounters["npc_id"]` en cualquier momento sin riesgo de null-reference, aunque el NPC no haya sido encontrado aún (`met: false`).

**Primer encuentro y obtención de demonios**: El jugador comienza desarmado y sin demonios en el loadout (aunque el Gato está siempre presente narrativamente). A través del juego, cuando obtiene un demonio:

1. El demonio se añade a `available_demons` (SIEMPRE)
2. El jugador recibe una notificación: "Has obtenido [demonio]. Puedes equiparlo en el menú de Loadout"
3. El jugador puede elegir CUÁNDO y SI equiparlo:
   - Si `equipped_demons` tiene < 5 slots: equipar es un click
   - Si `equipped_demons` tiene 5 slots: sistema requiere desequipar algo primero, LUEGO equipar el nuevo
   - El jugador puede optar por NO equiparlo y dejar espacio para un demonio futuro

**Ejemplos**:
- Demonio Fuego obtenido → `available_demons = ["fuego"]`, `equipped_demons` sigue igual (jugador decide después)
- Jugador equipa Fuego → `equipped_demons = ["fuego"]`
- Demonio Hielo obtenido → `available_demons = ["fuego", "hielo"]`, puede equipar sin desequipar porque hay espacio
- Demonio Arcano obtenido → `available_demons = ["fuego", "hielo", "arcano"]`, pero loadout está lleno (5 demonios) → jugador debe desequipar algo para equipar Arcano, O dejarlo sin equipar
- *Nota: El Gato vive en `companion_state` separado, no en `available_demons`*

### 3.3 Actualización de Estado

El Estado del Mundo se actualiza de forma **continua pero selectiva**:

**Eventos que actualizan estado (ejemplos)**:
- Jugador entra en una nueva área → `areas_visited[area_id].visited = true`
- Jugador **obtiene/vincula** un demonio (momento narrativo cinematográfico) → agregar SOLO a `available_demons` (decisión de equipo es después)
- Jugador **equipa** un demonio del disponible (acción UI en Loadout) → agregar a `equipped_demons` (si hay espacio: 3-5 slots máximo, excepto Gato)
- Jugador **desequipa** un demonio (acción UI en Loadout) → remover de `equipped_demons`, mantener en `available_demons`
- Jugador completa un diálogo con NPC → registrar en `npc_encounters[npc_id].dialogue_branches_seen`
- Jugador ejecuta decisión moral (ejecutar vs perdonar) → registrar en `player_choices`
- Evento narrativo avanza de fase → `major_events["evento_la_voz_del_gato"] += 1` (o asigna fase específica)
  - Ej: Acto 2 completa fase 1 → `major_events["cat_reveal"] = 1`; Acto 3 completa fase 2 → `major_events["cat_reveal"] = 2`
- Jugador toma acción moralmente oscura → `corruption_level` aumenta (cantidad específica por acción)

**Restricción**: Las actualizaciones NO son automáticas. Cada sistema (Combate, Vinculación, Diálogo, Loadout) es responsable de llamar explícitamente al sistema de Estado para registrar cambios:

```
// Obtención narrativa: el demonio entra a available_demons, listo para equipar
world_state.obtain_demon("fuego")  // Añade SOLO a available_demons

// Decisión de equipo: jugador decide si lo activa (operación separada)
if len(world_state.equipped_demons) < 5:
  world_state.equip_demon("fuego")  // Añade a equipped_demons
else:
  // Loadout lleno: pedir al jugador desequipar algo primero
  show_ui_message("Loadout lleno (5 demonios). Desequipa uno para equipar Fuego.")

// Cambio de loadout: remover un demonio activo
world_state.unequip_demon("arcano")  // Remueve "arcano" de equipped_demons, no de available_demons

// Registrar acciones morales (con contexto enriquecido)
world_state.record_player_choice("ejecutar_bandido_x", "ejecutado", act=1, conscious=true)
// Internamente almacena: {"value": "ejecutado", "act": 1, "timestamp": Time.get_unix_time_from_system(), "conscious": true}
world_state.increment_corruption(0.05)
```

**Límite de loadout**: `equipped_demons` NUNCA puede tener más de 5 demonios (excepto Gato, que es aparte y no cuenta hacia el límite). Si el jugador intenta equipar un demonio cuando el loadout está lleno, el sistema rechaza la acción y requiere que desequipe otro primero.

### 3.3b Timing de Escritura al Disco (Contrato de Persistencia)

El Estado del Mundo usa un esquema **híbrido** para minimizar pérdida de datos sin impactar rendimiento:

| Trigger de escritura | Qué se persiste | Cuándo ocurre |
|----------------------|----------------|---------------|
| **Salida de sala/área** | Estado de sesión (`current_hp`, `current_position`) | Al cruzar cualquier transición de área |
| **Save point explícito** | Estado completo del `world_state` | Al activar save point o en menú de pausa |
| **Salir del juego** | Estado completo del `world_state` + sesión | Antes de cerrar la aplicación |
| **Evento narrativo crítico** | `major_events`, `player_choices` afectados | Inmediatamente tras el evento (escritura parcial) |

**NUNCA**: escritura on-every-mutation. Las actualizaciones de `demon_saturation`, `explored_percent`, `reputation` se acumulan en memoria y se persisten en el siguiente save point.

**Contrato para `Guardado y Carga` (GDD #12)**: El GDD de Guardado y Carga debe implementar esta tabla exacta de triggers. Estado del Mundo provee `serialize_full()` y `serialize_session_only()`. `Guardado y Carga` elige qué método llamar según el trigger.

### 3.4 Gating Narrativo (Reactividad)

El mundo usa el Estado del Mundo para controlar acceso:

**Ejemplo A — Gating de Demonio**:
```
Si world_state.major_events.get("cat_reveal", 0) == 0:
  // Acto 1: Telepatía aún no desbloqueada
  show_dialogue("gato_mindlink_act1_vague")
Sino si world_state.major_events.get("cat_reveal", 0) >= 1:
  // Acto 2+: Fase 1 completada, comunicación telepática real
  show_dialogue("gato_mindlink_act2_full")
```

**Ejemplo B — Restricción de Área**:
```
Si world_state.player_choices.has("ejecutar_lord_X") and world_state.player_choices["ejecutar_lord_X"]["value"] == "ejecutado":
  // Si ejecutaste al lord, los guardias de su castillo son hostiles
  enemies_in_area = "hostiles"
Sino:
  enemies_in_area = "neutrales"
```

**Ejemplo C — Transformación Visual**:
```
corruption_level = world_state.corruption_level
Si corruption_level < 0.3:
  edrick_appearance = "íntegro" (sprite limpio, aura blanca)
Sino si corruption_level < 0.7:
  edrick_appearance = "en_transición" (sprite con grietas, aura gris)
Sino:
  edrick_appearance = "corrompido" (sprite oscuro, aura roja)
```

### 3.5 Contrato de Eventos Significativos (Authoring Contract)

El Estado del Mundo solo acepta eventos de un conjunto **cerrado y versionado** de tipos. Esta lista se define en `/assets/data/world/event_types.json` y es la única fuente de verdad para qué constituye un "evento significativo".

```json
{
  "event_types": {
    "kill":      "Edrick eliminó a un personaje (NPC o enemigo nombrado)",
    "choice":    "Jugador tomó una decisión narrativa explícita (diálogo, acción binaria)",
    "discovery": "Jugador descubrió un área, objeto o secreto por primera vez",
    "binding":   "Edrick vinculó un nuevo demonio (momento narrativo único)",
    "dialogue":  "NPC completó una rama de diálogo con contenido narrativo persistente",
    "whisper":   "Edrick fue expuesto a un susurro demoniaco y lo interrumpió (corrupción proporcional al tiempo escuchado)"
  }
}
```

**Reglas de autoría**:
- Todo llamado a `world_state.record_player_choice()` o `world_state.set_major_event()` desde sistemas downstream **debe** incluir un `event_type` del conjunto anterior
- Eventos que no encajan en ningún tipo → no son "significativos" para el Estado del Mundo (se pueden loggear localmente pero no persisten como `player_choices` o `major_events`)
- Nuevos tipos de evento requieren modificar `event_types.json` **y** documentarlo aquí antes de implementar

**Contrato para sistemas downstream**: Al registrar un evento, los campos obligatorios son:
```
world_state.record_event(
  event_type: string,    // Uno de los 6 tipos definidos arriba
  event_key: string,     // ID único del evento específico (ej: "ejecutar_bandido_x" o "susurro_001")
  value: string,         // Resultado o dato del evento
  act: int,              // Acto narrativo actual (1, 2 o 3)
  conscious: bool        // True si fue acción explícita del jugador; false si fue consecuencia
)
```

**Nota especial para event_type="whisper"** (GDD #8 — Exploración del Mundo):
- `event_key`: ID único del susurro (ej: "susurro_001_reino_1")
- `value`: Porcentaje de audio escuchado como string (ej: "0.5" para 50%)
- `pct_listened`: float ∈ [0.0, 1.0] — parámetro adicional que indica qué porcentaje del susurro fue escuchado antes de interrumpir
- `conscious`: true (siempre — el jugador consciente elige interrumpir o dejar terminar el susurro)
- El delta de corrupción_floor se calcula internamente: `corruption_delta = whisper_base_delta × pct_listened`
- El susurro trigger se marca como "completado" y no se repetirá en futuras visitas

### 3.6 Ejemplos de Reactividad Completa

**Escenario A — Obtención vs Equipo (decisiones separadas)**:
Jugador en Reino 1, Acto 1, obtiene Fuego en encuentro narrativo. Después obtiene Hielo. Más tarde obtiene Arcano pero decide NO equiparlo todavía.

**Qué sucede**:
1. Al obtener Fuego: `available_demons = ["fuego"]`. Jugador equipa Fuego inmediatamente → `equipped_demons = ["fuego"]`. El Gato vive en `companion_state` (definido por GDD #13), no en `available_demons`.
2. Al obtener Hielo: `available_demons = ["fuego", "hielo"]`. Jugador equipa Hielo → `equipped_demons = ["fuego", "hielo"]`
3. Al obtener Arcano: `available_demons = ["fuego", "hielo", "arcano"]`. Jugador decide NO equiparlo aún porque quiere mantener solo Fuego + Hielo → `equipped_demons = ["fuego", "hielo"]` (Arcano está disponible pero no equipado)
4. Más tarde, jugador abre Loadout UI y decide: "Equipo Arcano, desequipo Hielo" → `equipped_demons = ["fuego", "arcano"]`
5. El Gato siempre está activo independientemente del loadout (no toma slot)

---

**Escenario B — Persistencia y reactividad post-guardado**:
Jugador obtiene Fuego, ejecuta a un NPC bandido. Guarda, cierra el juego, y carga.

**Qué sucede**:
1. Al cargar, el Estado carga todas las persistencias: `available_demons = ["fuego", "hielo"]`, `equipped_demons = ["fuego"]`, `companion_state = {...}` (Gato), `corruption_level = 0.15`, `player_choices["ejecutar_bandido_x"] = {"value": "ejecutado", "act": 1, "timestamp": 12345.0, "conscious": true}`
2. Edrick aparece visualmente con saturación de Fuego (aura moderada) + corrupción visual leve (aura gris suave combinada)
3. El bandido (si estuviera en el mismo área) estaría muerto (su enemigo no reaparece)
4. Dialogues de otros NPCs que conocen al bandido tienen variaciones: "Oí que ejecutaste a Grax. Eres más oscuro de lo que pensé."
5. Los efectos de Fuego (resistencias, modificadores de movimiento) están activos porque está en `equipped_demons`
6. Si Edrick baja HP durante esta sesión, al recargar el próximo checkpoint, su HP vuelve a su máximo, pero el Estado del Mundo (corruption_level, available_demons, etc.) persiste

---

## 4. Fórmulas

---

### 📌 DEFINICIÓN CANÓNICA DE CORRUPCIÓN

**⚠️ CRÍTICO**: La corrupción es un eje que atraviesa múltiples GDDs. Estas son las definiciones autoritativas:

| Aspecto | Definición Canónica | GDD Responsable |
|---------|-------------------|-----------------|
| **Tier values** de demonios (cómo generan corrupción pasiva) | Tabla en GDD #3 §4.3 (Base de Datos de Demonios) | GDD #3 |
| **Señal de corrupción pasiva** emitida en combate | Fórmula en GDD #6 §3 (Combate en Tiempo Real) — cada 60s emite `corruption_passive_tick(amount)` | GDD #6 |
| **Acumulación y decay** de `corruption_level` | Fórmulas §4.1 AQUÍ (Estado del Mundo) — clamping, floor, decay | GDD #4 |
| **Efectos visuales** (sprite, aura) de corrupción | Tabla de umbrales en GDD #14 §3 (Transformación Visual de Edrick) | GDD #14 |
| **Acciones narrativas** que modifican corrupción | Deltas definidos en GDD #22 (Seguimiento Moral) | GDD #22 |
| **Interacción** entre corrupción y reputación de NPC | Patrones de autoria en GDD #15 (NPC y Diálogo) | GDD #15 |

**Regla de consistencia**: Si encuentras inconsistencia entre estas definiciones, la prioridad es:
1. GDD #3 (definición de demonios) es fuente de verdad para tiers
2. GDD #6 (combate) es fuente de verdad para la señal/trigger
3. GDD #4 (este documento) es fuente de verdad para la acumulación
4. Los demás GDDs la *consultan* pero no la redefinan

---

### 4.1 Cambio de Corrupción Moral

La corrupción tiene **tres fuentes** que el sistema combina:

1. **Acciones narrativas oscuras/redentoras** (deltas únicos por evento)
2. **Corrupción pasiva por demonios equipados** (acumulativa por minuto en combate — ver 4.1.B)
3. **Decay natural** (reduce corruption_level lentamente cuando no estás usando demonios fuertes — ver 4.1.C)

Sobre estas tres fuentes se aplica el **floor permanente** (4.1.D) que limita cuánto puede decaer.

#### 4.1.A — Deltas Narrativos (acciones únicas)

Cada acción narrativa tiene un "delta de corrupción" predefinido (definido en GDD Seguimiento Moral #22):

```
corruption_level_nuevo = clamp(corruption_level_anterior + delta_corrupcion, corruption_floor, 1.0)
```

**Costes/Beneficios de ejemplo**:
- Ejecutar a un NPC rendido: +0.10 (también sube floor +0.02)
- Abandonar a un aliado en combate: +0.15 (también sube floor +0.02)
- Mentir a un NPC para obtener información: +0.05 (sube floor +0.005)
- Pacto demoníaco "oscuro" (binding agresivo): +0.10 (sube floor +0.03)
- Perdonar a un enemigo rendido: −0.08 (NO afecta floor)
- Sacrificarse por un aliado: −0.15 (NO afecta floor)
- Salvar a un inocente en peligro: −0.10 (NO afecta floor)

#### 4.1.B — Corrupción Pasiva por Demonios Equipados

**Decisión cross-review B2 (2026-05-26)**: Tabla definida en GDD #3 Sección 4.3. Combate emite señal `corruption_passive_tick(amount_per_minute)` cada 60 segundos en estado de combate activo. Estado del Mundo aplica:

```
func _on_corruption_passive_tick(amount_per_minute: float):
    corruption_level = clamp(corruption_level + amount_per_minute, corruption_floor, 1.0)
```

**Ejemplo cumulativo**: 30 min de combate con Arcano+Fuego+Dash equipados:
- Suma de tiers (de tabla GDD #3 §4.3): 0.005 + 0.003 + 0.003 = 0.011/min
- Ganancia total: 0.011 × 30 = +0.33 corrupción

#### 4.1.C — Decay Natural

Mientras Edrick no esté ganando corrupción pasiva (no en combate, o solo Gato/demonios débiles equipados), `corruption_level` decae **−0.0005 por minuto de tiempo de juego** siempre que `corruption_level > corruption_floor`.

```
func _process(delta: float):
    if corruption_level > corruption_floor and not in_active_combat:
        corruption_level = max(corruption_level - (CORRUPTION_DECAY_RATE / 60.0) * delta, corruption_floor)
```

Donde `CORRUPTION_DECAY_RATE = 0.0005` por minuto.

**Equilibrio resultante**:
- Equipar Arcano + Fuego + Dash y entrar en combate: +0.011/min (ganancia rápida)
- Desequipar todo (solo Gato) y explorar tranquilo: −0.0005/min (purga lenta — 22 veces más lenta que la ganancia con Tier S)
- Resultado: la corrupción se acumula rápido y se purga lento (apropiado temáticamente)

#### 4.1.D — Floor de Corrupción Permanente (Memoria Mecánica)

**Decisión cross-review D-W6 + B2 (2026-05-26)**: El sistema mantiene `corruption_floor` (float 0.0-1.0) que representa el **mínimo permanente irrecuperable**. Actos narrativos oscuros incrementan el floor permanentemente — nunca decrece.

```
narrative.corruption_floor: float (rango 0.0-1.0, monotónicamente creciente)
```

**Reglas**:
- `corruption_floor` se inicializa a 0.0 al comenzar nueva partida
- Solo crece cuando ocurren actos narrativos oscuros (tabla 4.1.A "sube floor")
- Nunca decrece — ni siquiera con actos redentores
- **Nunca excede 1.0**: `corruption_floor = min(corruption_floor + delta, 1.0)`
- Define el mínimo al que `corruption_level` puede decaer

**Ejemplo**: Edrick ejecuta 3 NPCs (+0.10 cada uno = +0.30 corrupción, +0.02 cada uno = +0.06 floor). Floor final: 0.06.
- Si después salva 3 inocentes (−0.10 cada uno = −0.30 corrupción): `corruption_level = clamp(0.30 − 0.30, 0.06, 1.0) = 0.06` (no baja del floor).
- Visualmente: las marcas más profundas (cicatrices del filtro de saturación) persisten incluso tras la redención. La memoria del cuerpo importa.

#### 4.1.E — Mapeo a Estados Visuales

| Rango | Estado | Apariencia |
|-------|--------|-----------|
| 0.0 ≤ corruption < 0.2 | Íntegro | Sprite limpio, aura blanca |
| 0.2 ≤ corruption < 0.4 | Manchado | Marcas sutiles, ojos ligeramente oscuros |
| 0.4 ≤ corruption < 0.6 | Comprometido | Saturación demoníaca visible, pose agresiva |
| 0.6 ≤ corruption < 0.8 | Corrompido | Aspecto claramente demoníaco, voz alterada |
| 0.8 ≤ corruption ≤ 1.0 | Caído | Casi monstruoso, narrativa no-retorno |

### 4.2 Saturación Demoníaca

La saturación de un demonio aumenta mientras está en `equipped_demons`, se congela cuando se desequipa:

```
Si demonio está en equipped_demons:
  saturation_demonio = clamp(saturation_demonio + delta_time * (SATURATION_RATE / 60.0), 0.0, 1.0)
Sino:
  saturation_demonio = se mantiene congelado en valor actual
```

Donde:
- `delta_time` = segundos del frame actual (float, ej: 0.016 a 60fps)
- `SATURATION_RATE` = **0.05 por minuto** → dividir entre 60 convierte a unidades/segundo
- Resultado: alcanza saturación total 1.0 en 20 minutos de uso continuo

**Ejemplo**: A 60fps, `delta_time ≈ 0.016s`. Incremento por frame = `0.016 × (0.05/60) ≈ 0.0000133`. En 20 minutos (72000 frames) → `72000 × 0.0000133 ≈ 1.0`. ✓

**Mapeo a intensidad visual**:
- 0.0 ≤ saturation < 0.3: aura leve/sutil
- 0.3 ≤ saturation < 0.6: aura moderada/visible
- 0.6 ≤ saturation ≤ 1.0: aura intensa/dominante

La saturación afecta el sprite de Edrick: más saturación = efectos visuales más intensos (partículas más abundantes, colores más vívidos, etc.).

### 4.3 Progreso de Exploración de Área

Cuando el jugador se mueve dentro de un área, el sistema registra cobertura:

```
tiles_visitados = cantidad de tiles únicos pisados en el área
tiles_totales_area = cantidad de tiles accesibles en el área

// Guard obligatorio — evita division por zero en áreas stub o vacías
Si tiles_totales_area == 0:
  explored_percent = 0.0
Sino:
  explored_percent = (tiles_visitados / tiles_totales_area) * 100.0
```

**Umbral narrativo**: Cuando `explored_percent >= 90%`, el área se considera "completamente explorada" y puede desbloquear:
- Contenido narrativo oculto (diálogos de NPCs sobre lo que descubriste)
- Secretos/coleccionables gateados por "completitud"
- Logros o mejora de reputación

### 4.4 Reputación de NPC

La reputación de un NPC comienza en 0 y cambia basada en decisiones del jugador:

```
reputation_npc = clamp(reputation_base + sum(reputacion_cambios), -1.0, 1.0)
```

Donde `reputacion_cambios` son eventos registrados como `player_choices`:
- Si perdonaste al bandido que atacó a este NPC: +0.20
- Si ejecutaste a alguien que el NPC amaba: −0.30
- Si mentiste al NPC (y él se entera): −0.10
- Si ayudaste a este NPC contra enemigos: +0.25

**Mapeo a reactividad narrativa**:
- -1.0 ≤ reputation < -0.5: NPC es completamente hostil, diálogos adversarios
- -0.5 ≤ reputation < 0.0: NPC desconfía de ti, diálogos fríos
- 0.0 ≤ reputation < 0.5: NPC neutral, diálogos normales
- 0.5 ≤ reputation ≤ 1.0: NPC es aliado, diálogos amables, descuentos si hay comercio

### 4.5 Determinación de Rama de Diálogo

Cuando un NPC habla, el sistema selecciona qué rama mostrar basado en el estado del mundo:

```
branches_disponibles = []
Para cada branch en npc.dialogue_branches:
  Si branch.requisitos_narrativos_se_cumplen(world_state):
    branches_disponibles.add(branch)

Si branches_disponibles está vacío:
  mostrar_default_dialogue()
Sino:
  // Ordenar ascendente por campo `priority: int` — número menor = mayor prioridad
  branches_disponibles.sort_by(lambda b: b.priority)
  seleccionar_y_mostrar(branches_disponibles[0])

// Cada branch en el archivo de datos de NPC DEBE tener campo priority explícito:
// { "id": "branch_cat_reveal", "priority": 1, "conditions": {"cat_reveal": {"gte": 1}}, "text": "..." }
// Sin este campo, el orden de evaluación es UNDEFINED. Validar en importación de datos.
```

**Requisitos típicos de rama**:
- `major_events.get("evento_la_voz_del_gato", 0) >= 1` — muestra solo si ocurrió el evento (fase 1+)
- `npc_reputation > 0.5` — muestra solo si el NPC te aprecia
- `equipped_demons.contains("fuego")` — muestra solo si tienes Fuego equipado (reacción temática)
- `corruption_level >= 0.7` — muestra solo si Edrick está corrompido

### 4.6 Disponibilidad de Demonio Narrativo (Base Genérica)

Algunos demonios pueden estar gateados por progreso narrativo. La restricción se define en la Base de Datos de Demonios (GDD #3) y se evalúa contra `major_events`:

```
Si demonio en available_demons:
  Si demonio.restriction_event_key == "":
    mostrar_en_loadout = true  // Sin restricción narrativa
  Sino si major_events.get(demonio.restriction_event_key, 0) >= demonio.restriction_required_phase:
    mostrar_en_loadout = true  // Fase requerida alcanzada
  Sino:
    mostrar_en_loadout = false  // Disponible pero temporalmente inaccesible
Sino:
  mostrar_en_loadout = false  // No obtenido aún
```

Ejemplo: Si un demonio "X" tiene `restriction_event_key = "acto_2_inicio"` y `restriction_required_phase = 1`, aparecerá en loadout solo cuando `major_events["acto_2_inicio"] >= 1`.

---

## 5. Casos Extremos

### 5.1 Obtener Demonio Cuando Loadout Está Lleno (5 demonios equipados)

**Escenario**: Jugador tiene 5 demonios equipados y obtiene un nuevo demonio en encuentro narrativo.

**Qué sucede**:
1. El demonio se añade a `available_demons` (siempre ocurre)
2. El sistema muestra mensaje: "Has obtenido [demonio], pero tu loadout está lleno. Ve al menú de Loadout para equiparlo (deberás desequipar otro primero)."
3. En UI de Loadout, el nuevo demonio está disponible para equipar
4. Si jugador intenta equiparlo sin desequipar: UI rechaza con mensaje "Loadout lleno. Máximo 5 demonios."
5. Jugador desequipa uno, LUEGO equipa el nuevo

**Restricción**: El sistema NUNCA auto-desequipa un demonio cuando el jugador obtiene uno nuevo (cuando el loadout está lleno). La decisión de qué desequipar es del jugador. EXCEPCIÓN: Si un demonio equipado se vuelve no disponible por restricción narrativa (major_events no cumple la fase), el sistema lo auto-desequipa automáticamente para mantener el loadout válido (ver CA-018 en §8.3).

---

### 5.3 Intentar Desequipar el Último Demonio No-Gato

**Escenario**: Jugador solo tiene 1 demonio equipado (ej: Fuego). Intenta desequiparlo en UI de Loadout.

**Qué sucede**:
1. Sistema permite la acción — no hay restricción mínima en MVP
2. `equipped_demons = []` (vacío)
3. Edrick combate sin demonios equipados: sin resistencias, sin bonificadores, sin habilidades activas especiales
4. Solo el Gato sigue actuando (siempre presente)
5. Jugador puede combatir de esta forma (es una estrategia válida), pero sin ventajas demoníacas

**Restricción**: No hay un requisito mínimo de demonios equipados en MVP. El jugador elige libremente.

---

### 5.4 Cargar Save Cuando El Mundo Ha Cambiado Radicalmente

**Escenario**: Jugador guarda en Acto 1 con Fuego equipado. Vuelve a cargar en Acto 2, donde Fuego narrativamente "ha cambiado" o tiene restricciones nuevas.

**Qué sucede**:
1. El Estado del Mundo carga exactamente como fue guardado: `equipped_demons = ["fuego"]`, `corruption_level = 0.15`, etc.
2. Sistemas posteriores (Vinculación, Transformación Visual, NPC Diálogo) consultan el estado Y los eventos actuales
3. Si Fuego tiene nuevas restricciones en Acto 2, el sistema las aplica: Fuego sigue equipado pero con modificadores alterados (ej: daño reducido porque fue "debilitado" narrativamente)
4. Contenido narrativo que asume cierta progresión (ej: "Mataste al Bandido X") se muestra — el save es válido

**Restricción**: El Estado del Mundo es literal — guarda lo que sucedió, no lo que "debería" haber sucedido. Si el mundo cambió entre guardado y carga, es responsabilidad de otros GDDs (Progresión Narrativa, NPC y Diálogo) manejar la reactividad.

---

### 5.5 Corrupción Alcanza 1.0 o Baja a 0.0

**Escenario A — Máxima corrupción**: Jugador ejecuta NPCs, miente, traiciona aliados hasta que `corruption_level = 1.0`.
- Edrick alcanza estado "Corrompido": aura roja oscura, sprite deformado, voz oscura en diálogos
- Ciertos NPCs pueden volverse completamente hostiles (si su reputación era ya negativa)
- Nuevas ramas de diálogo "solo para corrompido" se desbloquean
- Restricción de acceso: algunas áreas sagradas pueden rechazar a Edrick si está "demasiado corrompido"

**Escenario B — Mínima corrupción**: Jugador toma decisiones redentoras hasta que `corruption_level = 0.0`.
- Edrick regresa al estado "Íntegro": aura blanca, sprite limpio, diálogos honestos
- NPCs previamente hostiles pueden reacercarse si su reputación mejora
- Nuevas ramas de diálogo "solo para íntegro" se desbloquean

**Nota**: No hay "punto de no retorno". El arco moral es continuo — el jugador puede escalar o descender en corrupción en cualquier momento según sus elecciones.

---

### 5.6 NPC Muere Pero El Jugador No Lo Sabía

**Escenario**: Enemigo mata a un NPC aliado en combate. El NPC se marca como "muerto" en `npc_encounters[npc_id].alive = false`. Más tarde, el jugador intenta hablar al NPC.

**Qué sucede**:
1. El sistema verifica `npc_encounters[npc_id].alive == false`
2. El NPC no aparece en el mundo (está invisible/inaccesible)
3. Si el jugador intenta interactuar donde estaba el NPC, nada sucede
4. Otros NPCs comentan: "Oí que [NPC] fue asesinado en combate"
5. El jugador no puede recuperar diálogos no vistos del NPC (perdió esa rama narrativa)

**Restricción**: Las muertes de NPCs son permanentes en MVP — no hay resurrección.

---

### 5.7 Exploración Sobre 100% (Imposible Matemáticamente)

**Escenario**: Sistema detecta que `explored_percent > 100%` (error de cálculo o tiles contados dos veces).

**Qué sucede**:
1. El sistema clampea: `explored_percent = clamp(explored_percent, 0.0, 100.0)`
2. Se registra como 100% exploración
3. Desbloquea cualquier contenido gateado por "100% exploración"
4. Sistema registra un warning en logs: "Area exploration > 100% (error de cálculo detectado)"

**Restricción**: Esto no debería ocurrir en producción, pero el clamping previene corrupción de datos.

---

### 5.8 Save Corrupto o Inválido

**Escenario**: Archivo de save se corrompe (error de escritura), o jugador lo edita manualmente.

**Qué sucede**:
1. Al cargar, el sistema valida el schema del Estado del Mundo
2. Si faltan campos críticos (ej: `corruption_level` no existe), el sistema carga con valor por defecto (0.0)
3. Si un demonio en `equipped_demons` no existe en Base de Datos de Demonios, se remueve silenciosamente
4. El save se "autocorrige" a valores válidos
5. Jugador puede continuar jugando, pero puede haber perdido datos (ej: demonios desaparecen)

**Restricción**: No hay sistema de "save backup" en MVP — si el save se corrompe irreparablemente, el jugador debe empezar de nuevo.

---

### 5.9 Demonio Se Vuelve No Disponible Mientras UI Está Abierta

**Escenario**: Jugador tiene UI de Loadout abierta. En paralelo, evento narrativo ocurre que hace un demonio no disponible (ej: `major_events["acto_2_comienza"] = 1` y el demonio requiere `major_events["acto_2_comienza"] >= 2`).

**Qué sucede**:
1. El evento registra la nueva fase: `major_events["acto_2_comienza"] = 1`
2. Sistema verifica disponibilidad del demonio usando fórmula 4.6
3. Demonio ya NO cumple el requisito y pasa `mostrar_en_loadout = false`
4. UI de Loadout se refresca automáticamente (si está abierta)
5. Si el demonio estaba equipado, se auto-desequipa (no puede permanecer equipado si es no disponible)

**Restricción**: El sistema asume que la UI de Loadout usa observadores/listeners para actualizar cuando el Estado cambia. Los cambios narrativos deben ser deferidos hasta cerrar la UI, o la UI debe ser refreshed automáticamente.

---

## 6. Dependencias

### 6.1 Dependencias Entrantes (qué depende de este sistema)

Este es un **Foundation-layer system** — muchos sistemas consultan el Estado del Mundo:

1. **Guardado y Carga** (GDD #12)
   - Depende de: El Estado del Mundo es TODO lo que se guarda/carga
   - Punto de integración: `save_file.json` = serialización completa de `world_state`
   - Validación: Guardar y cargar debe ser bidireccional (serialize/deserialize)

2. **Loadout & Build Management** (GDD #10)
   - Depende de: `available_demons`, `equipped_demons`
   - Punto de integración: UI de Loadout lee estas listas y permite cambios
   - Validación: Cambios en UI se registran de vuelta en Estado

3. **Transformación Visual de Edrick** (GDD #14)
   - Depende de: `corruption_level`, `demon_saturation[demonio]`, `equipped_demons`
   - Punto de integración: Sprite y aura de Edrick se actualizan basado en estos valores
   - Validación: Cambios visuales reflejan exactamente el estado registrado

4. **NPC y Diálogo** (GDD #15)
   - Depende de: `major_events`, `player_choices`, `npc_encounters`, `corruption_level`, `equipped_demons`
   - Punto de integración: Rama de diálogo se selecciona basada en requisitos gateados por Estado
   - Validación: Diálogos correctos se muestran para cada combinación de estado

5. **Progresión Narrativa** (GDD #16)
   - Depende de: `major_events`, `acts_completed`, `player_choices`
   - Punto de integración: Eventos narrativos disparan nuevos eventos, que se registran en Estado
   - Validación: Árbol narrativo se ramifica correctamente basado en elecciones

6. **Seguimiento Moral** (GDD #22, Vertical Slice)
   - Depende de: `corruption_level`, `player_choices`
   - Punto de integración: Sistema lee decisiones del jugador, incrementa corrupción
   - Validación: Corrupción se calcula y persiste correctamente

7. **Restricción por Demonio** (GDD #23, Vertical Slice)
   - Depende de: `available_demons`, `equipped_demons`, `major_events`
   - Punto de integración: Ciertos demonios pueden restringir acceso a áreas/acciones
   - Validación: Restricciones se aplican/remuevan según loadout

8. **Vinculación de Demonios** (GDD #13)
   - Depende de: `available_demons`, `companion_state`, momento narrativo de obtención
   - Punto de integración: Sistema registra cada vinculación en `available_demons`. Define la estructura completa de `companion_state` (ver contrato de retorno abajo)
   - Validación: Cada demonio obtenido queda registrado permanentemente. El gato (compañero) es definido completamente por Vinculación GDD #13.
   - **CONTRATO DE RETORNO (Vinculación → Estado del Mundo)**: Vinculación GDD #13 DEBE extender `companion_state` con: `bonding_status: string` (estados: "not_met", "encountered", "bonding", "revealed", etc.), `revelation_act: int` (acto en que se revela identidad), `relationship_history: array` (momentos clave). Estado del Mundo NO define estos detalles internos; Vinculación es responsable.

9. **Combate en Tiempo Real** (GDD #6) — añadido cross-review 2026-05-26 (resuelve W-01)
   - Depende de: `equipped_demons` para calcular el tier total de corrupción pasiva
   - Punto de integración: Estado del Mundo escucha señal `corruption_passive_tick(amount_per_minute)` emitida por Combate cada 60s en combate activo. Aplica delta a `corruption_level` (ver Sección 4.1.B). Lee tabla de tiers desde GDD #3 §4.3.
   - Validación: Tiempo en combate activo se traduce correctamente en ganancia de corrupción según loadout

10. **Exploración del Mundo** (GDD #8) — añadido auditoría 2026-05-27 (resuelve BD-01)
    - Depende de: `visited_zones`, `discovered_pois`, `discovered_secrets` para persistir progreso de exploración
    - Punto de integración: Estado del Mundo escucha señales emitidas por Exploración: `zona_visitada(zona_id)`, `poi_activado(poi_id, tipo)`, `secreto_descubierto(secreto_id)`. Cada señal actualiza el campo correspondiente en `world_state`.
    - Validación: Al recargar save, las zonas previamente visitadas permanecen marcadas; los POIs activados no se vuelven a disparar; los secretos descubiertos aparecen en Bestiario/Mapa.

### 6.2 Dependencias Salientes (qué este sistema necesita)

El Estado del Mundo es **independiente** — no lee de otros GDDs, solo ESCRIBE. Pero necesita:

1. **Base de Datos de Demonios** (GDD #3)
   - Requerido para: Validar que IDs de demonios en `available_demons` y `equipped_demons` existen
   - Integración: Al cargar save, validar que cada demonio está en la BD

2. **Salud y Daño** (GDD #2)
   - Requerido para: Contexto (sabe que existen "5 tipos de daño", pero no interactúa directamente)
   - Integración: Mínima — Estado es agnóstico de mecánicas de combate

### 6.3 Bidireccionalidad

- **Loadout ↔ Estado**: Loadout lee `available_demons`, `equipped_demons`. Estado escribe estos campos cuando Loadout los cambia.
- **Vinculación → Estado**: Vinculación GDD #13 escribe a `companion_state` cuando el gato es encontrado narrativamente.
- **NPC Diálogo → Estado**: NPC Diálogo **lee** del Estado (no escribe — solo la narrativa escribe).
- **Narrativa → Estado**: Progresión Narrativa **escribe** a Estado (`major_events`, `player_choices`, `corruption_level`). Estado no escribe a Narrativa.
- **Transformación Visual → Estado**: Transformación Visual **lee** del Estado (no escribe).

### 6.4 Restricciones de Dependencia

- Estado del Mundo **NO debe importar sistemas de gameplay específicos** (ej: Combate, IA). Es agnóstico de cómo funciona el juego, solo registra qué pasó.
- Estado del Mundo **NO debe tomar decisiones narrativas**. Solo registra hechos. Las decisiones (qué rama mostrar, si un NPC es hostil) las toma el sistema que consulta el Estado.
- Cambios al Estado deben ser **explícitos y trazables** — otros sistemas no pueden hacer cambios silenciosos.

---

## 7. Parámetros de Ajuste

Los siguientes parámetros se pueden ajustar para tuning sin afectar la estructura del Estado del Mundo:

### 7.1 Velocidad de Saturación Demoníaca

```
SATURATION_RATE = 0.05 por minuto (alcanza 1.0 en 20 minutos de uso continuo)
Rango seguro: 0.01-0.10 por minuto (entre 100 minutos y 10 minutos para saturación total)
```

**Parámetro de ajuste**: Si los demonios se sienten demasiado "potentes" visualmente rápido, reducir a 0.03. Si tardan demasiado, aumentar a 0.07.

### 7.2 Deltas de Corrupción por Acción

**Acciones que AUMENTAN corrupción:**

| Acción | Delta | Rango Seguro |
|--------|-------|--------------|
| Ejecutar NPC rendido | +0.10 | +0.05 a +0.15 |
| Abandonar aliado en combate | +0.15 | +0.10 a +0.20 |
| Mentir a NPC para ganancia personal | +0.05 | +0.02 a +0.08 |
| Equipo demonio "corrupto" (por 5 min) | +0.01 | +0.005 a +0.02 |
| Robar item importante | +0.08 | +0.05 a +0.12 |

**Acciones que DISMINUYEN corrupción:**

| Acción | Delta | Rango Seguro |
|--------|-------|--------------|
| Perdonar enemigo rendido | −0.08 | −0.05 a −0.12 |
| Sacrificar ventaja propia por aliado | −0.15 | −0.10 a −0.20 |
| Confesar verdad costosa | −0.05 | −0.02 a −0.08 |
| Salvar inocente en peligro | −0.10 | −0.05 a −0.15 |
| Restaurar lo robado + pedir perdón | −0.08 | −0.05 a −0.12 |

**Parámetro de ajuste**: Si corrupción es difícil de controlar (sube rápido, baja lento), ajustar los multiplicadores. Usar para balancear el arco moral del juego.

### 7.3 Umbrales de Corrupción y Mapeo Visual

| Rango | Estado | Comportamiento |
|-------|--------|----------------|
| 0.0-0.3 | Íntegro | Sprite limpio, aura blanca suave, diálogos normales |
| 0.3-0.7 | En Transición | Sprite con grietas/oscurecimiento, aura gris, diálogos más sarcásticos |
| 0.7-1.0 | Corrompido | Sprite deformado/oscuro, aura roja/negra, diálogos malvados |

**Parámetro de ajuste**: Mover los umbrales si quieres que los estados duran más/menos. Ej: cambiar 0.3 a 0.5 hace "En Transición" más largo.

### 7.4 Umbrales de Reputación de NPC

| Rango | Estado | Comportamiento |
|-------|--------|----------------|
| -1.0 a -0.5 | Enemigo | NPC hostil, diálogos adversarios, puede atacar si provocado |
| -0.5 a 0.0 | Desconfiado | NPC se aleja, diálogos fríos, no ayuda |
| 0.0 a 0.5 | Neutral | NPC neutral, diálogos normales, puede comerciar |
| 0.5 a 1.0 | Aliado | NPC amigable, diálogos cálidos, ofrece ayuda/descuentos |

**Parámetro de ajuste**: Mover los umbrales si quieres reputación más "extrema" (cambiar a -1.0 a -0.3, 0.7 a 1.0) o más "gradual" (cambiar a -0.8 a 0.8).

### 7.5 Umbral de Exploración

```
EXPLORATION_THRESHOLD = 90% (cuando área se considera "completamente explorada")
Rango seguro: 80% a 100%
```

**Parámetro de ajuste**: Si 90% es demasiado alto (jugadores raramente lo alcanzan), reducir a 75%. Si muy bajo, aumentar a 95%.

### 7.6 Configuración de Archivo de Datos

Todos estos parámetros se almacenan en `/assets/data/world/state_config.json`:

```json
{
  "saturation": {
    "rate_per_minute": 0.05,
    "visual_thresholds": [0.3, 0.6]
  },
  "corruption": {
    "costs": {
      "execute_npc": 0.10,
      "abandon_ally": 0.15,
      "lie_to_npc": 0.05
    },
    "state_thresholds": [0.3, 0.7]
  },
  "reputation": {
    "thresholds": [-0.5, 0.0, 0.5]
  },
  "exploration": {
    "completion_threshold": 0.90
  }
}
```

**Restricción**: NINGÚN parámetro debe estar hardcoded. TODO es data-driven desde este archivo.

### 7.7 Balance Pass Plan

Después de MVP (cuando todo el mundo esté jugable):

1. **Fase 1 (Post-MVP)**: Playtesting interno. ¿Corrupción sube demasiado rápido? ¿Saturación es visible?
2. **Fase 2 (Pre-Vertical Slice)**: Ajustar umbrales basado en feedback. Los valores pueden variar ±20%.
3. **Fase 3 (Vertical Slice)**: Balance final con playtesting externo.

**NO BLOQUEA MVP**: Los valores presentes son suficientemente razonables para validar la persistencia y reactividad.

---

## 8. Criterios de Aceptación

### 8.1 Carga y Serialización del Estado

- [ ] **CA-001**: Al iniciar nuevo juego, `WorldState` tiene exactamente: `corruption_level == 0.0`, `equipped_demons == []`, `available_demons == []`, todos los `major_events` tienen fase `0`, todos los NPCs tienen `met: false`, `alive: true`, `reputation: 0.0`. El gato vivirá en `companion_state` (definido por GDD #13). Verificable con test unitario que llama `WorldState.new_game()` y comprueba cada campo.
- [ ] **CA-002**: El Estado del Mundo se serializa a JSON/YAML de forma bidireccional. PASS: serializar un WorldState con valores conocidos, deserializar, y verificar que cada campo tiene el mismo valor. Si `corruption_level` intenta exceder 1.0, el valor resultante es exactamente 1.0 (clamp silencioso). Si intenta bajar de 0.0, el resultado es 0.0. En ambos casos no hay error ni excepción.
- [ ] **CA-003**: Al guardar, todos los campos críticos están presentes: `progression`, `demons`, `narrative`, `world`, `session`. PASS: test que guarda, lee el JSON resultante, verifica que todas las keys están presentes.
- [ ] **CA-004**: Al cargar, si faltan campos opcionales, se rellenan con valores por defecto. PASS: test que carga un JSON sin el campo opcional `flavor_text`, verifica que se inicializa a "" sin error.
- [ ] **CA-005**: Save corrupto detectable — si campos críticos faltan, sistema muestra error en logs. PASS: test que intenta cargar un JSON sin `corruption_level` (crítico), verifica que genera error de validación y aparece en logs.

### 8.2 Estructura de Demonios

- [ ] **CA-006**: `equipped_demons` nunca excede 5 demonios. PASS: test que intenta equipar 6 demonios, verifica que el 6to falla y `len(equipped_demons) == 5`.
- [ ] **CA-007**: `available_demons` es histórico — una vez un demonio se obtiene, nunca se remueve. PASS: test que obtiene demonio, desequipa, intenta remover manualmente, verifica que sigue en `available_demons`.
- [ ] **CA-008**: Cambio de `equipped_demons` se persiste correctamente al guardar. PASS: test que equipa demonio, guarda, recarga, verifica que sigue equipado.
- [ ] **CA-009**: Obtener demonio → se añade a `available_demons` solamente (NO a `equipped_demons` automáticamente). PASS: test que obtiene demonio, verifica que aparece en `available_demons` pero NO en `equipped_demons`.
- [ ] **CA-010**: Equipo demonio → se añade a `equipped_demons` si hay espacio (≤5), falla si lleno con mensaje claro. PASS: test que equipa demonio cuando hay espacio (éxito), luego llena todos y trata de equipar uno más (falla con mensaje).
- [ ] **CA-011**: Desequipo demonio → se remueve de `equipped_demons`, permanece en `available_demons`. PASS: test que equipa demonio, desequipa, verifica que `has(equipped_demons) == false` pero `has(available_demons) == true`.
- [ ] **CA-012**: Demonio con `restriction_required_phase` no cumplida no aparece en Loadout UI (pero sigue en `available_demons`). [MOVE TO GDD #10 Loadout & Build Management] PASS: test que restringe demonio a fase 5, verifica que no aparece en lista de equipables cuando fase < 5.

### 8.3 Gating Narrativo

- [ ] **CA-013**: Después de guardar y cargar, `areas_visited['area_id'].visited` y `areas_visited['area_id'].explored_percent` tienen el mismo valor que antes de guardar. PASS: guardar con `explored_percent == 47.5`, recargar, verificar que el campo == 47.5 (±0.001).
- [ ] **CA-014**: Después de guardar y cargar, `npc_encounters['npc_id'].reputation`, `dialogue_branches_seen`, y `alive` tienen los mismos valores que antes de guardar. PASS: guardar con `alive: false, reputation: -0.3`, recargar, verificar ambos campos.
- [ ] **CA-015**: `player_choices` se registra cuando jugador ejecuta decisión binaria. Campo: `{value: string, act: int, timestamp: float, conscious: bool}`. PASS: test que registra elección, verifica que los 4 campos están presentes con tipos correctos.
- [ ] **CA-016**: Rama de diálogo que requiere `player_choices["ejecutar_bandido_x"]["value"] == "ejecutado"` solo aparece si esa elección fue hecha. PASS: test de branch selection con valor presente vs ausente.
- [ ] **CA-017**: Demonio se vuelve no disponible cuando `major_events["demonio_x_event"] >= required_phase` no se cumple. Fórmula 4.6 evaluada correctamente. PASS: test que evalúa fórmula con valores conocidos, verifica que demonio es no disponible cuando condición falla.
- [ ] **CA-018**: Si demonio estaba equipado cuando se vuelve no disponible, sistema lo auto-desequipa. PASS: equipo demonio, incrementa fase restrictiva, verificar que `equipped_demons.has(demonio) == false`.

### 8.4 Corrupción Moral

- [ ] **CA-019**: `corruption_level` comienza en 0.0. PASS: test que llama `WorldState.new_game()` y verifica `corruption_level == 0.0`.
- [ ] **CA-020**: Acción oscura incrementa `corruption_level` correctamente (ej: ejecutar +0.10). PASS: test que llama `increment_corruption(0.10)` y verifica resultado.
- [ ] **CA-021**: Acción redentora disminuye `corruption_level` correctamente (ej: perdonar −0.08). PASS: test que llama `increment_corruption(-0.08)` y verifica resultado.
- [ ] **CA-022**: `corruption_level` está clampado entre 0.0 y 1.0 (nunca va fuera de rango). PASS: test que suma +0.5 a nivel 0.8, verifica que resultado es 1.0 (no 1.3).
- [ ] **CA-023**: Transformación Visual se actualiza cuando `corruption_level` cruza umbral (0.3 o 0.7). PASS: test que incrementa a 0.31 y verifica que `edrick_appearance != "integro"`. [MOVE TO GDD #14 Transformación Visual de Edrick]
- [ ] **CA-024**: `corruption_level` persiste correctamente al guardar/cargar. PASS: guardar en 0.45, recargar, verificar == 0.45 (±0.001).

### 8.5 Saturación Demoníaca

- [ ] **CA-025**: Cuando demonio está en `equipped_demons`, su saturación aumenta (0.05 por minuto). [blocked: clock-injection]
- [ ] **CA-026**: Cuando demonio se desequipa, su saturación se congela (no sube más). [blocked: clock-injection]
- [ ] **CA-027**: Cuando demonio se re-equipa, saturación retoma desde donde quedó (no se reset). [blocked: clock-injection]
- [ ] **CA-028**: Saturación está clampada entre 0.0 y 1.0. PASS: test que intenta establecer saturación a -0.5 (resultado: 0.0) y 1.5 (resultado: 1.0). [blocked: clock-injection]
- [ ] **CA-029**: Cuando demonio se desequipa, su saturación en `demon_saturation` se mantiene en el valor en el momento de desequipar. PASS: equipar Fuego por 10 minutos de juego (saturación ≈ 0.5), desequipar, esperar 5 minutos más, verificar que `demon_saturation["fuego"] ≈ 0.5` (±0.005). [blocked: clock-injection] Transformación Visual refleja saturación (aura más intensa según saturación) — movido a GDD #14.

### 8.6 Reputación de NPC

- [ ] **CA-030**: `npc_encounters[npc_id].reputation` comienza en 0.0. PASS: test que llama `WorldState.new_game()` y verifica `npc_encounters["npc_1"].reputation == 0.0` para al menos 3 NPCs.
- [ ] **CA-031**: Decisión jugador que afecta NPC actualiza su reputación correctamente (ej: perdonar +0.20). PASS: test que llama `update_npc_reputation("npc_1", 0.20)` y verifica el valor cambió correctamente.
- [ ] **CA-032**: Reputación está clampada entre -1.0 y 1.0. PASS: test que intenta establecer reputación a -2.0 (resultado: -1.0) y 2.0 (resultado: 1.0).
- [ ] **CA-033**: Después de registrar `player_choices["ejecutar_bandido_x"]` via `record_player_choice`, el valor almacenado tiene exactamente: `value == "ejecutado"`, `act == [el acto pasado como parámetro]`, `conscious == [el bool pasado]`. PASS: test unitario que registra la acción y verifica los 4 campos del dict. NPC con reputación > 0.5 muestra rama de diálogo con `required_reputation_min: 0.5` cuando hay otras ramas disponibles — verificar que branch selection retorna la rama correcta. [MOVE TO GDD #15 NPC y Diálogo]
- [ ] **CA-034**: Una rama de diálogo con condición `major_events["cat_reveal"] >= 1` NO aparece cuando `major_events["cat_reveal"] == 0`. La misma rama SÍ aparece cuando `major_events["cat_reveal"] == 1`. PASS: test unitario de branch selection con ambos estados del WorldState. [MOVE TO GDD #15 NPC y Diálogo]

### 8.7 Exploración de Área

- [ ] **CA-035**: Al entrar en área, `areas_visited[area_id].visited = true`. PASS: test que llama `enter_area("area_1")` y verifica `areas_visited["area_1"].visited == true`.
- [ ] **CA-036**: Si el archivo JSON de save tiene `corruption_level: null`, el sistema reemplaza el valor con `0.0` y registra WARNING en logs. Si el campo está completamente ausente, igual se inicializa a `0.0`. El sistema continúa cargando sin crash. PASS: cargar un JSON con `corruption_level: null` y verificar que el WorldState resultante tiene `corruption_level == 0.0` y que los logs contienen "corruption_level missing or null, defaulting to 0.0". Al moverse en área, `explored_percent` se actualiza basado en tiles únicos pisados.
- [ ] **CA-037**: Cuando `explored_percent >= 90%`, área se marca como completamente explorada. PASS: test que establece `explored_percent = 0.91` y verifica que la bandera de "explorada completamente" se activa.
- [ ] **CA-038**: Un save con `metadata.version: "1.0"` cargado en sistema `version: "1.1"` aplica el migrador de versión. Campos nuevos en 1.1 se inicializan a sus defaults. Campos removidos se ignoran silenciosamente. PASS: el WorldState migrado puede guardarse y recargarse sin pérdida adicional. (Solo aplicable cuando exista una segunda versión del schema — este AC está latente hasta entonces.) Contenido gateado por exploración se desbloquea cuando se alcanza 90%.

### 8.8 Guardado y Carga

- [ ] **CA-039**: Al guardar, timestamp se registra en `metadata.last_save_timestamp`. PASS: test que guarda, verifica que `metadata.last_save_timestamp` es un float > 0.
- [ ] **CA-040**: Al cargar, todos los campos del Estado se restauran exactamente como fueron guardados. PASS: save/load roundtrip con valores conocidos, verificar que cada campo es idéntico (±0.001 para floats).
- [ ] **CA-041**: Campos de `session` (HP actual, posición) NO persisten entre cargas — se resetean. PASS: test que guarda con HP=25, recarga, verifica que HP reseteó a un valor por defecto (ej: max_hp).
- [ ] **CA-042**: Campos fuera de `session` (corruption_level, available_demons, etc.) SÍ persisten. PASS: test que guarda con `corruption_level=0.5`, recarga, verifica que sigue siendo 0.5.
- [ ] **CA-043**: Cargar en checkpoint diferente resetea posición pero mantiene Estado del Mundo. PASS: cargar save, cambiar checkpoint, verificar que `corruption_level` y `available_demons` persisten pero posición se reseteó.

### 8.9 Integración con Otros Sistemas

- [ ] **CA-044**: Múltiples llamadas a métodos mutadores de WorldState dentro del mismo frame se procesan secuencialmente y todas tienen efecto. PASS: test que llama `increment_corruption(0.05)` tres veces en el mismo frame y verifica que `corruption_level == initial + 0.15` (±0.001). Godot es single-threaded en el game loop — no se usa threading.
- [ ] **CA-045**: Loadout UI rechaza equipar demonio si `equipped_demons` está lleno (muestra mensaje: "Loadout lleno. Desequipa un demonio primero."). [MOVE TO GDD #10 Loadout & Build Management]
- [ ] **CA-046**: Loadout UI rechaza desequipar último demonio pero no falla (error/restricción es opcional — jugador puede combatir sin demonios). [MOVE TO GDD #10 Loadout & Build Management]
- [ ] **CA-047**: Con demonio Fuego a `demon_saturation["fuego"] == 0.0` y a `== 1.0`, el sistema de Combate calcula los mismos modificadores de daño en ambos casos. PASS: verificar que el cálculo de `damage_modifier` no lee `demon_saturation` — solo lee `equipped_demons`. [MOVE TO GDD #6 Combate en Tiempo Real]
- [ ] **CA-048**: Transformación Visual lee `corruption_level` y actualiza sprite/aura correctamente. Cuando `corruption_level` cruza umbral (0.3 o 0.7), la apariencia cambia. PASS: test que incrementa `corruption_level` a 0.31 y verifica que `edrick_appearance != "integro"`. [MOVE TO GDD #14 Transformación Visual de Edrick]
- [ ] **CA-049**: NPC Diálogo consulta `major_events` antes de mostrar rama. Si rama requiere `major_events["evento_x"] >= 2` y el estado actual es `== 1`, la rama no aparece. [MOVE TO GDD #15 Sistema de NPC y Diálogo]

### 8.10 Edge Cases

- [ ] **CA-050**: Demonio obtiene ID que no existe en BD — cargar lo detecta y lo remueve de `available_demons`. PASS: test que carga un JSON con demonio ID inválido, verifica que se removió y que no hay error de runtime.
- [ ] **CA-051**: Después de mover a Edrick por exactamente 50 tiles únicos en un área con 100 tiles totales, `areas_visited['area_id'].explored_percent == 50.0`. Si se pisan tiles ya visitados, el porcentaje no cambia. PASS: test con área sintética de 100 tiles.
- [ ] **CA-052**: Jugador está en menú Loadout cuando evento narrativo ocurre que hace demonio no disponible — UI se recarga automáticamente y demonio se desequipa si estaba equipado. [MOVE TO GDD #10 Loadout & Build Management]
- [ ] **CA-053**: WorldState no debe tener dependencias en Autoloads de Godot en su constructora. Debe poder instanciarse con `WorldState.new()` en un test GUT sin inicializar la escena completa. PASS: test que llama `WorldState.new()` sin `get_tree()` y sin `autoload.*` disponibles.
- [ ] **CA-054**: Save se carga cuando Acto ha cambiado radicalmente (ej: guardado en Act 1, cargado en Act 2) — Estado antiguo persiste, sistemas posteriores lo manejan correctamente. PASS: test que carga un save de Act 1, simula cambio a Act 2, verifica que WorldState persiste sin corrupción.

### 8.11 Testing Checklist

**Unit Tests**:
- [ ] Incremento de corrupción calcula correctamente
- [ ] Saturación aumenta/se congela según equipo
- [ ] Reputación está clampada
- [ ] Exploración no excede 100%

**Integration Tests**:
- [ ] Loadout → Estado: cambios se registran
- [ ] Estado → Transformación Visual: sprite se actualiza
- [ ] Estado → NPC Diálogo: rama correcta aparece
- [ ] Guardado → Carga: restaura todo excepto `session`

**Manual Testing**:
- [ ] Equipo/desequipo demonios — cambios aparecen en UI
- [ ] Ejecuto NPC — corruption sube visible
- [ ] Guardo, cierro, cargo — Estado idéntico
- [ ] Demonio desaparece narrativamente — desequipa automáticamente

---

## 9. Consideraciones de Diseño Conocidas (Design Theory Warnings)

### 9.1 D-W2: Meso-Loop de Progresión Fragmentado

**Estado**: Conocido, requiere futura GDD de Progresión Narrativa.

Sesiones de 15-30 minutos sin demonio binding narrativo importante no tienen recompensa de "crecí". El binding demoníaco es raro (narrativo, no common); no hay paralelo a "gold accumulated" o "bestiary entries" que señalicen progresión entre eventos grandes.

**Recomendación**: GDD futura de Progresión Narrativa debe definir meso-loop rewards:
- Reputación de NPCs visible (ej: "Aldea C te reconoce (+10 rep)")
- Entradas de Bestiary desbloqueadas por kill específicos
- Diálogos de NPCs que responden a `corruption_level` (ej: NPC rehúye a Edrick si está muy corrupto)
- Saturación: visible en world (ej: plantas mueren cerca de Edrick si saturación > 0.8)

**Impact en diseño actual**: Este GDD es responsable de mantener el `WorldState`. La estructura está lista. El "qué recompensa al jugador entre bindings" es responsabilidad de sistemas narrativos posteriores (GDD Progresión, GDD Diálogos, GDD Bestiario).

### 9.2 D-W6: Corrupción Reversible Permite Min-Maxing — RESOLVED

**Estado**: Resuelto mediante `corruption_floor` mechanism (implementado 2026-05-26).

Antes: 3 ejecuciones (+0.30) + 3 salvaciones (−0.30) = mismo estado visual → jugador podía "redeem grind".

Ahora: Cada acto oscuro sube `corruption_floor` permanentemente (+0.02 por acto); decay NO puede bajar de floor → "memoria" de crímenes pasados persiste mecánicamente.

### 9.3 Trade-off: Reputación y Corrupción Son Independientes

**Estado**: Conocido, MVP constraint deliberado.

**El problema**: NPCs no reaccionan a cambios observados en corrupción visual. Un NPC que conoces desde Acto 1 mantiene su reputación fija incluso si Edrick se vuelve visualmente monstruoso en Acto 3. La validación es binaria (did-you-take-this-action: yes/no), no observacional (I-see-what-you-are-becoming).

**Razón MVP**: Acoplar reputation a corruption requeriría que Visual Transformation (GDD #14) publique eventos cuando el estado visual cambia, y que NPC Dialogue (GDD #15) lea esos eventos. Eso es integración de 3 sistemas. MVP simplifica: reputation = lo que hiciste, corrupción = cómo luces.

**Plan post-MVP**: GDD #15 (NPC y Diálogo) y GDD #14 (Transformación Visual) trabajarán juntas para añadir observación. Los NPCs notarán cuando Edrick se vuelve visiblemente corrupto y ajustarán sus diálogos/comportamiento independientemente de acciones.

**Impact en Pilar 3 ("Mundo vivo y reactivo")**: Parcialmente diferido. MVP entrega reactividad a ACCIONES. Post-MVP entrega reactividad a TRANSFORMACIÓN.

### 9.4 Trade-off: Demonio Cero-Equipado Permite Juego Desarmado

**Estado**: Conocido, MVP constraint deliberado.

**El problema**: Pilar 2 dice "demonios son cambios transformadores con costos Y beneficios". Sistema permite `equipped_demons = []` (sin demonios). Un jugador que desquipa todo puede combatir sin ninguna transformación demoníaca.

**Razón MVP**: Permite momentos de pausa narrativa. Edrick como "solo humano" es temáticamente válido en ciertos momentos. La restricción mecánica (sin bonificadores, sin habilidades especiales) actúa como costo.

**Nota importante**: El Gato es siempre presente (vive en `companion_state`, no en `equipped_demons`), así que no puedes estar completamente desarmado narrativamente.

**Plan post-MVP**: Considerar si corrupción pasiva se aplica incluso con loadout vacío, o si demonios equipados REQUIEREN un mínimo de poder de combate para evitar trivializar ciertas áreas.

### 9.5 Trade-off: "Edrick Puro" (Sin Corrupción) Sin Path Narrativo Definido

**Estado**: Conocido, requiere decisión creativa.

**El problema**: Sistema permite que corruption_level decaiga a corruption_floor. Pilar 5 dice transformación es "inevitable, ganada". Estos son contradictorios si un jugador puede mantener corrupción baja (<0.2 "Íntegro") durante toda la partida.

**Pregunta para Creative Director**: ¿Existe un ending coherente para un Edrick que NUNCA se corrompió? ¿O la narrativa de Act 3 presupone cierto nivel de transformación?

**Plan pending**: Antes de escribir GDD #16 (Progresión Narrativa), necesita resolverse: ¿hay path bajo-corrupción, o hay un corruption_floor progresivo por act que sube automáticamente con la narrativa?

**Current state**: Sin decisión. Documentado como conocido; diferido a creative-director + narrative-director en fase de narrative GDD.
