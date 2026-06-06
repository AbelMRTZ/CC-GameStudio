# GDD: Loadout & Build Management

> **Estado**: Aprobado
> **Autor**: Abel + Claude Code Agents
> **Última Actualización**: 2026-05-28 (re-revisión 2: firma `loadout_changed` extendida a `(equipped_demons, gato_available: bool)` alineada con GDD #11, nueva §3.1.F monitoreo Gato via método directo `notify_gato_available`, §3.3 interfaz pública actualizada, §5 Gato re-emisión, §6.1/6.2 contratos, Rule 12 referencia `recalculate_hp_on_max_change()`, CA-024/029/038/039/034 corregidas)
> **Implements Pillar**: Pilar 4 — Sinergia y Libertad de Construcción
> **Milestone**: MVP — Core Layer
> **Depende de**: Base de Datos de Demonios (#3), Estado del Mundo (#4)
> **Dependen de este sistema**: Motor de Sinergias (#11), Vinculación (#13), Transformación Visual (#14), HUD de Combate (#18), Build Management UI (#20), Restricción por Demonio (#23)

---

## 1. Visión General

El Loadout & Build Management gestiona qué demonios de la Base de Datos están equipados activamente en Edrick en cada momento del juego. Internamente, mantiene la lista `equipped_demons` — la fuente de verdad que Combate, Motor de Sinergias, Transformación Visual y HUD consultan para determinar las habilidades disponibles, los modificadores de stats y el aspecto visual de Edrick. Desde la perspectiva del jugador, es el sistema que habilita la libertad de construcción de builds: el jugador selecciona qué demonios vinculados quiere llevar activos, y el sistema aplica los cambios de forma inmediata sobre movimiento, resistencias, daño y apariencia. Cada decisión de loadout es una declaración de identidad — no hay una build "correcta", solo combinaciones que se adaptan mejor a distintas situaciones, y la experimentación con sinergias (tanto positivas como negativas) es el corazón de la progresión del jugador.

---

## 2. Fantasy del Jugador

La fantasía del Loadout es el momento en que la experimentación revela su fruto: descubrir que la combinación exacta de demonios que el jugador estaba probando *funciona* — que la sinergia existe, que el sistema recompensó la curiosidad. No es la satisfacción de haber elegido bien desde el principio, sino la de haber *encontrado* algo. Un instante "a-ha" que solo aparece porque el jugador se atrevió a experimentar.

Pero en Demons of Dravaryn, el descubrimiento de un combo poderoso no es solo satisfacción táctica. Cada demonio activo drena algo de Edrick — cada combinación más poderosa es también una combinación más corrupta. El jugador que construye el loadout perfecto no solo encontró el combo ganador: ha decidido, implícitamente, cuánto de Edrick está dispuesto a sacrificar por esa ventaja. La maestría estratégica en este juego tiene sabor moral.

El momento ideal: el jugador dice "¡lo encontré!" — y a la vez siente un destello de "¿pero a qué costo?"

---

## 3. Diseño Detallado

### 3.1 Reglas Centrales

**A. Estructura de Slots**

1. El loadout activo tiene **N slots de demonio** (N = 2 en MVP) más un **slot especial de Gato** que es visible pero no editable.
2. Cada slot puede estar vacío u ocupado por un demonio de la lista `demonios_disponibles` del jugador.
3. Cada demonio puede aparecer como máximo UNA VEZ en el loadout activo.
4. El **Slot de Gato** refleja al Gato como aliado — visible en la Build Screen, bloqueado, no intercambiable. No forma parte de los N slots de demonio de Edrick. Excepción: cuando Estado del Mundo marca al Gato como narrativamente no disponible, el slot muestra un indicador de ausencia (ver §5).
5. Los slots adicionales (más allá de los 2 del MVP) se desbloquean permanentemente al encontrar **Sellos de Vínculo** escondidos en los reinos. Cada Sello otorga +1 slot de forma permanente.

**B. Operaciones de Loadout**

6. **Equipar** (slot vacío → demonio): el jugador selecciona un demonio de la lista disponible y lo asigna a un slot vacío.
7. **Intercambiar** (slot ocupado → otro demonio): reemplaza el demonio activo. El demonio anterior se desactiva; el nuevo se activa.
8. **Limpiar** (slot ocupado → vacío): el jugador quita un demonio. El slot queda vacío y sus modificadores cesan.

**C. Timing del Cambio**

9. Al completar cualquier operación de Loadout (equipar, intercambiar o limpiar), Edrick siempre entra en una **animación de intercambio de 0.8 segundos** — sin importar si está en exploración o en combate. Durante la animación, Edrick no puede moverse, atacar ni dashearse.
10. **Solo en combate**: durante la animación Edrick puede recibir daño (es vulnerable). Al finalizar la animación, **si el estado en ese momento sigue siendo COMBAT**, se aplica un **cooldown de 5 segundos** antes de poder cambiar el loadout de nuevo. Si el combate terminó durante los 0.8s de animación (el estado ya es EXPLORACIÓN al finalizar), el cooldown no se activa.
11. Los cambios en stats, sinergias y transformación visual se aplican al **finalizar** la animación, no al iniciarla.

**D. Aplicación de Cambios**

Al finalizar la animación de intercambio:

12. El HP máximo se recalcula sumando los `hp_bonus` de todos los demonios equipados. Loadout llama a `HealthSystem.recalculate_hp_on_max_change(nuevo_HP_MAX: int)` (GDD #2 §3.5.1), que aplica el clamp, reduce el HP actual si supera el nuevo máximo, y emite `health_changed(nuevo_hp, 0, "stat_reduction")` si hay reducción.
13. Los modificadores de stats del demonio anterior se eliminan; los del nuevo se aplican inmediatamente.
14. El Motor de Sinergias recibe notificación vía la señal `loadout_changed(equipped_demons, gato_available)` del EventBus (ver Regla 16) y recalcula todas las sinergias activas.
15. La Transformación Visual actualiza sprite y aura de Edrick según el nuevo demonio activo.
16. El EventBus emite `loadout_changed(equipped_demons: Array[String], gato_available: bool, slot_cooldowns: Array[float], hp_max: int)` para que HUD, Motor de Sinergias y otros sistemas actualicen su estado. `gato_available` refleja el valor cacheado en `_gato_available`. `slot_cooldowns[i]` = `cooldown_segundos` del demonio equipado en slot i (0.0 si el slot está vacío). `hp_max` = HP máximo calculado tras aplicar todos los hp_bonus. **Esta es la misma señal del paso 14 — no son dos notificaciones separadas. Motor de Sinergias se suscribe al EventBus.**

**E. Acceso a la Build Screen**

17. La Build Screen puede abrirse en cualquier estado (EXPLORACIÓN o COMBAT).
18. Al abrirse en COMBAT, la Build Screen **pausa el tiempo del juego**: los enemigos se detienen y no pueden atacar ni avanzar durante la navegación del menú.
19. El tiempo reanuda únicamente cuando el jugador confirma una operación (iniciando SWAP_ANIM) o cierra la Build Screen sin hacer cambios.
20. Durante SWAP_ANIM la Build Screen se cierra automáticamente y el mundo reanuda en tiempo real — Edrick es vulnerable al daño enemigo durante los 0.8s de la animación.
21. En EXPLORACIÓN, abrir la Build Screen no pausa el tiempo (no hay enemigos activos). El comportamiento de lockout de movimiento/ataque aplica igualmente en exploración y combate (ver Reglas 9 y 10).

> **Nota de implementación Godot 4.6.3 — Pausa de árbol de escena:** Para que la pausa de Regla 18 funcione correctamente, los nodos que deben continuar procesando durante `get_tree().paused = true` requieren configuración explícita: Loadout state machine (`process_mode = PROCESS_MODE_ALWAYS`), Edrick input handler (`PROCESS_MODE_ALWAYS`), Build Screen UI (`PROCESS_MODE_ALWAYS`). Los `SceneTreeTimer` que controlan SWAP_ANIM también deben iniciarse **después** de que el árbol se despausa (en el momento del confirm), ya que los timers existentes se congelan durante pausa. Si el timer de animación se crea mientras el árbol está pausado, no avanzará hasta el resume.

**F. Monitoreo de Estado del Gato**

22. Loadout expone el método público `notify_gato_available(is_available: bool)`. Estado del Mundo (#4) lo invoca directamente cuando el estado narrativo del Gato cambia (Gato desaparece o regresa). Este patrón es consistente con el método directo que usa Vinculación (#13) para `add_available_demon()`.
23. Al recibir `notify_gato_available(is_available)`, Loadout actualiza su propiedad cacheada `gato_available` y re-emite `loadout_changed(equipped_demons, gato_available, slot_cooldowns, hp_max)` inmediatamente — aunque no haya habido swap de demonio. El Motor de Sinergias recibe la notificación y desactiva o reactiva las sinergias que requieren al Gato.

24. Durante el estado COMBAT, cuando `is_swap_on_cooldown = true`, el Loadout emite `swap_cooldown_updated(remaining: float)` en cada tick de `_process()` mientras `swap_cooldown_remaining > 0`. Al expirar el cooldown, emite `swap_cooldown_updated(0.0)` una vez. El HUD de Combate usa esta señal para renderizar F-HUD-04 sin acceso directo al nodo Loadout (ADR-002).

### 3.2 Estados y Transiciones

| Estado | Descripción | Transición desde | Transición hacia |
|--------|-------------|-----------------|-----------------|
| **EXPLORACIÓN** | Fuera de combate. Build Screen accesible sin restricción. | COMBAT al terminar combate | COMBAT al iniciar encuentro; SWAP_ANIM si cambia demonio |
| **COMBAT** | En combate activo. Cambios de loadout tienen penalización. | EXPLORACIÓN al iniciar encuentro | EXPLORACIÓN al terminar combate; SWAP_ANIM si cambia demonio |
| **SWAP_ANIM** | 0.8s de animación de intercambio. Edrick no puede actuar. En combate: vulnerable + cooldown posterior. | Desde cualquier estado al ejecutar operación de loadout | Vuelve al **estado actual** (no al estado de entrada) al finalizar animación. El cooldown se evalúa contra el estado al FINALIZAR: si ya es EXPLORACIÓN, no aplica cooldown. |

*Cooldown de intercambio en combate*: timer interno de 5 segundos que empieza al finalizar SWAP_ANIM **solo si el estado al finalizar la animación es COMBAT** (no el estado al iniciarla). Si el combate terminó durante SWAP_ANIM, el cooldown no se activa. La Build Screen muestra el tiempo restante y bloquea operaciones adicionales hasta que expire.

*Reset intencional al salir de combate* (CA-LBM-035): Si hay cooldown activo cuando el combate termina, se cancela inmediatamente. El cooldown penaliza swaps **dentro** de un encuentro, no entre encuentros. Si el jugador sale de combate con cooldown restante y re-entra inmediatamente, puede swappear sin restricción — este comportamiento es por diseño. La resistencia al swap es una fricción intra-combate, no inter-combate.

### 3.3 Interacciones con Otros Sistemas

| Sistema | Dirección | Qué fluye |
|---------|-----------|-----------|
| **Base de Datos de Demonios (#3)** | BD → Loadout | Datos de cada demonio: stats, habilidades, transformaciones visuales |
| **Estado del Mundo (#4)** | Bidireccional | Estado del Mundo provee `demonios_disponibles`; Loadout escribe `equipped_demons` activo para persistencia. Estado del Mundo notifica cambios de estado narrativo del Gato vía `notify_gato_available(bool)`. |
| **Motor de Sinergias (#11)** | Loadout → Motor | Señal `loadout_changed(equipped_demons: Array[String], gato_available: bool)` para recalcular sinergias activas. `gato_available` determina si las sinergias narrativas que requieren al Gato están habilitadas. |
| **Combate en Tiempo Real (#6)** | Loadout → Combate | Combate lee `equipped_demons` para mapear habilidades activas a inputs |
| **Transformación Visual (#14)** | Loadout → TV | Notificación con nuevo `demon_id` activo al equipar/intercambiar |
| **HUD de Combate (#18)** | Loadout → HUD | Estado del loadout: slots, demonios activos, cooldown de swap en combate |
| **Build Management UI (#20)** | UI → Loadout | La UI emite las operaciones (equipar/intercambiar/limpiar) al sistema |

**Interfaz pública del Loadout** (datos expuestos a otros sistemas):
- `equipped_demons: Array[String]` — IDs de demonios activos en slots (sin incluir Gato)
- `available_demons: Array[String]` — demonios vinculados pero no equipados actualmente
- `slot_count: int` — número de slots actuales (2 en MVP)
- `is_swap_on_cooldown: bool` — estado del cooldown de intercambio en combate
- `swap_cooldown_remaining: float` — segundos restantes del cooldown
- `gato_available: bool` — estado actual de disponibilidad narrativa del Gato (cacheado desde Estado del Mundo; actualizado vía `notify_gato_available`)
- `notify_gato_available(is_available: bool)` — método público para que Estado del Mundo notifique cambios en el estado del Gato

---

## 4. Fórmulas

### Fórmula 1: HP Máximo del Loadout Activo

```
HP_MAX = clamp(HP_BASE + ∑ hp_bonus_i, HP_FLOOR, HP_CEILING)
```

**Variables:**
| Variable | Símbolo | Tipo | Rango | Descripción |
|----------|---------|------|-------|-------------|
| HP base de Edrick | `HP_BASE` | int | 75 (constante) | HP sin ningún demonio equipado |
| Bonus de HP del demonio i | `hp_bonus_i` | int | -5 a +10 por demonio | Suma sobre todos los demonios en `equipped_demons`. Gato = 0 implícito. |
| Piso mínimo | `HP_FLOOR` | int | 1 (constante) | HP_MAX nunca puede ser < 1 |
| Techo máximo | `HP_CEILING` | int | 125 (tunable) | Protege contra stacking descontrolado en expansión post-MVP |
| HP máximo final | `HP_MAX` | int | 1 a 125 | HP máximo resultante con el loadout activo |

**Rango de salida:** 70–90 HP en MVP (dos slots llenos). Mejor caso: Hielo + Fuego = 90 HP. Peor caso: Visión + demonio con bonus 0 = 70 HP. (Para llegar a 65 HP se necesitarían dos demonios con `hp_bonus` negativo — en MVP solo Visión tiene −5. La regla de unicidad de slots hace imposible equipar Visión dos veces.)

**Ejemplos:**
- Fuego + Arcano: `HP_MAX = 75 + 5 + 0 = 80 HP`
- Hielo + Fuego: `HP_MAX = 75 + 10 + 5 = 90 HP`
- Visión + Dash: `HP_MAX = 75 + (-5) + 0 = 70 HP` (si HP actual era 73, se reduce a 70 al finalizar animación)
- Sin demonios (slots vacíos): `HP_MAX = 75 HP`

**Nota**: HP_CEILING = 125 permite progresión amplia sin trivializar el combate. Con un Agresor Melee haciendo 12 daño base y 125 HP, el jugador aguanta ~10 golpes — manejable con el escalado por reino.

### Fórmula 2: Capacidad de Slots del Loadout

```
slot_count = clamp(SLOTS_BASE + sellos_vinculo_encontrados, SLOTS_BASE, SLOTS_MAX)
```

**Variables:**
| Variable | Símbolo | Tipo | Rango | Descripción |
|----------|---------|------|-------|-------------|
| Slots base | `SLOTS_BASE` | int | 2 (constante MVP) | Slots disponibles al inicio del juego |
| Sellos encontrados | `sellos_vinculo_encontrados` | int | 0 a (SLOTS_MAX − SLOTS_BASE) | Sellos de Vínculo recogidos en la exploración del mundo |
| Cap máximo | `SLOTS_MAX` | int | 5 (tunable) | Techo de slots — no superable aunque se encuentren más Sellos |
| Slots actuales | `slot_count` | int | 2 a 5 | Número de slots disponibles en el loadout |

**Rango de salida:** 2–5 slots. MVP fijo en 2. Exactamente 3 Sellos de Vínculo en el mundo (encontrados en reinos post-MVP) pueden llevar el total a 5.

**Justificación del cap en 5:** Con 5 slots hay C(5,2) = 10 pares posibles de sinergias — manejable para el Motor de Sinergias. Con 6+ slots la carga cognitiva supera el límite razonable (D-W1 documentado en GDD #3). Con 18 demonios en visión completa y cap de 5, el jugador siempre debe elegir y sacrificar algo.

### Fórmula 3: Timing de Swap — Constantes Fijas

El cooldown de intercambio es **fijo** y no modificable por demonios ni sinergias. Un cooldown variable convertiría el swap en una habilidad de rotación en combate (en lugar de una decisión de preparación), crearía dependencia circular Loadout → Motor → Loadout, y haría a Arcano aún más dominante si lo redujese.

| Constante | Valor | Descripción |
|-----------|-------|-------------|
| `SWAP_ANIM_DURATION` | 0.8 s | Duración de la animación de intercambio. Siempre activa (exploración y combate). |
| `SWAP_COMBAT_COOLDOWN` | 5.0 s | Cooldown entre swaps en combate. Solo activo en estado COMBAT. Empieza al finalizar la animación. |

**Rango de salida:** Tiempo efectivo entre inicio de un swap y disponibilidad del siguiente en combate:
```
tiempo_hasta_proximo_swap = SWAP_ANIM_DURATION + SWAP_COMBAT_COOLDOWN = 0.8 + 5.0 = 5.8 segundos
```

**Ejemplo:** A t=0s Edrick swappea Hielo → Fuego durante combate. A t=0.8s la animación termina, modificadores de Fuego aplican, cooldown empieza. A partir de t=5.8s el siguiente swap es posible (no antes). Durante esos 5.8s Edrick combate con el loadout elegido.

**Nota de implementación (Godot 4.6.3):** Los `SceneTreeTimer` disparan en el siguiente frame tras expirar su duración, añadiendo hasta ~16ms por timer a 60 fps. Con dos timers en cadena (`SWAP_ANIM_DURATION` → `SWAP_COMBAT_COOLDOWN`), el tiempo efectivo real es ≥ 5.8s, no exactamente 5.8s. Los tests de integración deben especificar tolerancia de ±50ms o usar `Time.get_ticks_msec()` para medición precisa. Las ACs que usan "exactamente 0.8s" deben leerse como "0.8s ± 50ms".

---

## 5. Casos Extremos

- **Si todos los slots están vacíos durante el combate**: Edrick opera con `HP_MAX = HP_BASE = 75`, sin habilidades especiales ni modificadores de stats. Todas las sinergias cesan. Es un estado válido — el jugador puede limpiar todos los slots voluntariamente.

- **Si el jugador intenta equipar el mismo demonio en dos slots**: El sistema ignora la operación silenciosamente y mantiene el estado actual. Cada demonio solo puede aparecer una vez en el loadout activo.

- **Si HP actual > nuevo HP_MAX al equipar o limpiar un demonio**: Al finalizar la animación de intercambio, el HP actual se reduce automáticamente al nuevo HP_MAX. La reducción se procesa como `tipo="stat_reduction"` en `health_changed` (alineado con GDD #2 §3.5.1) — no como daño de combate (sin screen shake, sin flash de golpe, sin I-frames). La barra de HP en el HUD se actualiza directamente al nuevo valor. No hay texto flotante ni animación adicional. **Caso extremo de magnitud**: el peor caso en MVP es perder 20 HP en un swap (Hielo+Fuego → Visión+neutro, de HP_MAX=90 a HP_MAX=70). Este coste es intencional. Ejemplo concreto: HP actual = 80, des-equipa Hielo → HP_MAX baja de 85 a 75 → HP actual se reduce a 75.

- **Si el jugador intenta equipar un demonio no vinculado** (no en `demonios_disponibles`): El sistema rechaza la operación. La Build Screen no muestra demonios no desbloqueados — no es posible intentarlo desde la UI normal.

- **Si el jugador intenta iniciar un swap durante la animación SWAP_ANIM**: La operación se ignora. No hay encolamiento de swaps. El jugador espera a que la animación finalice.

- **Si el jugador intenta swap durante el cooldown de combate**: La Build Screen muestra el tiempo restante del cooldown. El slot no responde a la interacción hasta que el timer expire.

- **Si el Gato desaparece narrativamente** (Estado del Mundo lo marca como no disponible): Estado del Mundo llama a `notify_gato_available(false)`. Loadout actualiza `gato_available = false` y re-emite `loadout_changed(equipped_demons, false)` inmediatamente — sin swap de demonio. Motor de Sinergias recibe la señal y desactiva las sinergias que requieren al Gato (Mente + Gato, Visión + Gato). El slot especial de Gato muestra un indicador "ausente" en la Build Screen. Al regresar el Gato, `notify_gato_available(true)` activa la ruta inversa.

- **Si el jugador encuentra un Sello de Vínculo con `slot_count` ya en `SLOTS_MAX` (5)**: El Sello no otorga slots adicionales. La UI informa que se ha alcanzado el máximo. El Sello se marca como recogido en Estado del Mundo pero sin efecto mecánico.

- **Si demonios post-MVP suman más de 50 de `hp_bonus` total** (hipotético con 5 slots llenos): `HP_MAX` se clampea a `HP_CEILING = 125`. El jugador opera con 125 HP aunque los demonios "deberían" sumar más.

---

## 6. Dependencias

### 6.1 Dependencias Salientes (este sistema necesita)

| Sistema | Tipo | Qué necesita |
|---------|------|-------------|
| **Base de Datos de Demonios (#3)** | Dura | Lee datos de cada demonio: stats, hp_bonus, transformaciones visuales, habilidades. Sin este sistema, Loadout no puede saber qué efectos aplicar al equipar. |
| **Estado del Mundo (#4)** | Dura | Lee cuáles demonios han sido vinculados (disponibles para equipar). Escribe el estado `equipped_demons` activo para persistencia cross-sesión. **Ownership de `available_demons`**: Loadout mantiene una copia local inicializada desde Estado del Mundo en `_ready()`. Cuando Vinculación (#13) vincula un nuevo demonio, notifica a Loadout vía método directo `add_available_demon(id: String)` para actualizar la copia local. Estado del Mundo es la fuente de verdad; la copia de Loadout es un cache de lectura. En la carga de sesión, Loadout debe inicializarse *después* de Estado del Mundo (Autoload order en project settings). **Limitación MVP**: El sistema solo soporta adición (`add_available_demon`), no remoción de demonios de `available_demons`. En MVP ningún demonio regular desaparece narrativamente. Si eventos narrativos futuros requieren remoción, se definirá `remove_available_demon(id: String)` en la GDD del sistema correspondiente. El Gato usa un slot separado y nunca entra en `available_demons` — la desaparición narrativa del Gato se notifica vía método directo `notify_gato_available(is_available: bool)`: Estado del Mundo lo llama cuando `companion_state` del Gato cambia, Loadout actualiza `gato_available` y re-emite `loadout_changed`. |

### 6.2 Dependencias Entrantes (dependen de este sistema)

| Sistema | Qué espera del Loadout |
|---------|------------------------|
| **Motor de Sinergias (#11)** | Recibe señal `loadout_changed(equipped_demons: Array[String], gato_available: bool)` para recalcular sinergias activas. `gato_available` determina si las sinergias narrativas del Gato están habilitadas. La señal se emite en dos casos: (a) al finalizar SWAP_ANIM, y (b) al recibir `notify_gato_available` sin swap. |
| **Combate en Tiempo Real (#6)** | Lee `equipped_demons` para mapear las habilidades activas de cada demonio a los inputs del jugador. |
| **Transformación Visual (#14)** | Recibe notificación con el `demon_id` activo al completar un swap para actualizar sprite y aura de Edrick. |
| **HUD de Combate (#18)** | Recibe `loadout_changed(equipped_demons, gato_available, slot_cooldowns, hp_max)` para íconos, slot_count, cooldown_max por slot y HP_MAX. Recibe `swap_cooldown_updated(remaining: float)` para el indicador de swap (Regla 3 GDD #18). Todo via EventBus — sin acceso directo a propiedades del nodo Loadout. |
| **Build Management UI (#20)** | Es la capa de interacción del jugador. Emite operaciones (equipar/intercambiar/limpiar) al sistema y lee el estado del loadout para renderizarlo. **Requisito duro hacia GDD #20**: La UI DEBE especificar un affordance de pre-confirmación para swaps en combate — un estado visual (tooltip, cambio de label, highlight) que informe al jugador que confirmar iniciará 0.8s de vulnerabilidad antes de que puedan cancelar. Sin este affordance, la transición irrevocable al SWAP_ANIM es invisible para el jugador. |
| **Vinculación de Demonios (#13)** | Al vincular un nuevo demonio, notifica a este sistema para que actualice `available_demons` (el demonio pasa de no disponible a disponible). |
| **Restricción por Demonio (#23)** | Lee `equipped_demons` para verificar si el jugador tiene los demonios requeridos para acceder a ciertas áreas o momentos narrativos. |

### 6.3 Bidireccionalidad

- **Loadout ↔ Estado del Mundo**: bidireccional. Estado del Mundo es propietario del inventario de demonios vinculados; Loadout consume esos datos y escribe qué está actualmente equipado.
- **Loadout → Base de Datos (#3)**: unidireccional. Loadout lee de BD; BD no sabe qué está equipado.
- **Loadout → Motor de Sinergias (#11)**: unidireccional. Loadout notifica cambios; Motor recalcula y no escribe en Loadout.
- **Loadout → Combate (#6)**: unidireccional. Combate lee `equipped_demons`; no escribe en Loadout.
- **Build Management UI (#20) ↔ Loadout**: bidireccional. UI emite operaciones a Loadout; Loadout expone estado a UI para rendering.

---

## 7. Parámetros de Ajuste

Todos los valores deben estar en configuración externa (archivo de datos), no hardcodeados en código.

| Parámetro | Valor Base | Rango Seguro | Qué rompe si... |
|-----------|-----------|--------------|-----------------|
| `SWAP_ANIM_DURATION` | 0.8 s | 0.3–1.5 s | **Demasiado corto (<0.3s)**: el swap se siente instantáneo, pierde peso visual. **Demasiado largo (>1.5s)**: frustrante en exploración, parece un loading. |
| `SWAP_COMBAT_COOLDOWN` | 5.0 s | 3.0–10.0 s | **Demasiado corto (<3s)**: el jugador puede rotar builds en combate, destruye la decisión de preparación. **Demasiado largo (>10s)**: el swap en combate se vuelve inútil — el encuentro habrá terminado. |
| `SLOTS_BASE` | 2 | 1–3 | MVP fijo en 2. Si baja a 1, la restricción inicial dificulta el tutorial de sinergias. Si sube a 3, la tensión de "¿qué sacrifico?" en MVP se pierde. |
| `SLOTS_MAX` | 5 | 3–6 | **<3**: poca profundidad de build en visión completa. **>6**: carga cognitiva excesiva (>15 pares de sinergias posibles) y el HUD de slots necesitaría rediseño. Este valor tiene coste alto de cambio post-implementación — debe decidirse antes de implementar la UI. |
| `HP_FLOOR` | 1 HP | 1 (no reducir) | Si se reduce a 0, un loadout con demonios de hp_bonus negativo podría llevar el HP_MAX a 0, matando a Edrick al equipar. |
| `HP_CEILING` | 125 HP | 100–150 | **<100**: limita demasiado la progresión de builds defensivos. **>150**: la presión de daño de los enemigos se vuelve insignificante en versiones finales.

> **Nota de balance MVP — Arcano near-dominant**: Arcano es near-dominant en MVP porque su único costo designado (corrupción Tier S, +0.005/min en combate) pertenece al Sistema #22 (Seguimiento Moral), alcance Vertical Slice. En MVP, Arcano+cualquier demonio es mecánicamente superior a cualquier build sin Arcano, sin tradeoff activo. **Deuda de diseño aceptada**: el primer balance pass post-MVP y la implementación del Sistema #22 son los puntos de corrección designados. No bloquea implementación.

---

## 8. Criterios de Aceptación

### Estructura de Slots

**CA-LBM-001** — *Slots iniciales en MVP*
**GIVEN** un jugador inicia una nueva partida sin haber encontrado ningún Sello de Vínculo,
**WHEN** abre la Build Screen,
**THEN** ve exactamente 2 slots de demonio editables y 1 slot de Gato visible pero no interactuable.

**CA-LBM-002** — *Slot de Gato no es editable*
**GIVEN** el jugador tiene la Build Screen abierta,
**WHEN** intenta hacer clic o asignar un demonio al slot de Gato,
**THEN** el sistema no ejecuta ninguna operación: el slot de Gato permanece en su estado actual.

**CA-LBM-003** — *Sello de Vínculo añade exactamente 1 slot*
**GIVEN** el jugador tiene `slot_count = 2` y recoge un Sello de Vínculo,
**WHEN** abre la Build Screen después de recogerlo,
**THEN** `slot_count = 3` y la Build Screen muestra 3 slots de demonio editables.

**CA-LBM-004** — *Cap máximo de slots en 5*
**GIVEN** el jugador tiene `slot_count = 5` y recoge un Sello de Vínculo adicional,
**WHEN** el sistema procesa el Sello,
**THEN** `slot_count` permanece en 5 y el Sello queda marcado como recogido sin efecto mecánico adicional.

**CA-LBM-005** — *Fórmula de slots: cálculo exacto*
**GIVEN** el jugador ha encontrado 2 Sellos de Vínculo (`sellos_vinculo_encontrados = 2`),
**WHEN** el sistema calcula `slot_count = clamp(2 + 2, 2, 5)`,
**THEN** `slot_count = 4` — la Build Screen muestra exactamente 4 slots de demonio editables.

### Operaciones de Loadout

**CA-LBM-006** — *Equipar: slot vacío recibe demonio*
**GIVEN** el jugador tiene al menos 1 slot vacío y tiene "Fuego" en `demonios_disponibles`,
**WHEN** asigna Fuego a ese slot vacío,
**THEN** al finalizar la animación de 0.8s, el slot muestra "Fuego" y `equipped_demons` contiene el ID de Fuego.

**CA-LBM-007** — *Intercambiar: slot ocupado recibe otro demonio*
**GIVEN** el slot 1 tiene "Fuego" equipado y el jugador tiene "Hielo" en `demonios_disponibles`,
**WHEN** el jugador asigna Hielo al slot 1,
**THEN** al finalizar la animación de 0.8s: el slot 1 muestra "Hielo", Fuego ya no está en `equipped_demons`, e Hielo sí está.

**CA-LBM-008** — *Limpiar: slot ocupado queda vacío*
**GIVEN** el slot 1 tiene "Arcano" equipado,
**WHEN** el jugador ejecuta la operación Limpiar sobre ese slot,
**THEN** al finalizar la animación de 0.8s, el slot 1 está vacío y los modificadores de Arcano ya no están activos.

**CA-LBM-009** — *Unicidad: duplicado rechazado*
**GIVEN** el jugador tiene "Fuego" equipado en el slot 1 y el slot 2 está vacío,
**WHEN** intenta equipar "Fuego" en el slot 2,
**THEN** el sistema no ejecuta ninguna operación: el slot 2 permanece vacío, no se inicia ninguna animación.

### Timing y Animación

**CA-LBM-010** — *Animación dura 0.8s en exploración*
**GIVEN** Edrick está en estado EXPLORACIÓN,
**WHEN** el jugador ejecuta cualquier operación de loadout,
**THEN** la animación dura 0.8s ± 50ms (medir con `Time.get_ticks_msec()` en start y en callback de fin de animación — ver nota de tolerancia en §4 Fórmula 3) y durante ese intervalo Edrick no puede moverse, atacar ni dashearse.

**CA-LBM-011** — *Animación dura 0.8s en combate, Edrick vulnerable*
**GIVEN** Edrick está en estado COMBAT,
**WHEN** el jugador ejecuta cualquier operación de loadout,
**THEN** la animación dura 0.8s ± 50ms (medir con `Time.get_ticks_msec()`) y durante ese intervalo Edrick puede recibir daño de enemigos pero no puede moverse, atacar ni dashearse.

**CA-LBM-012** — *Cooldown de combate: 5.0s empieza al finalizar animación*
**GIVEN** Edrick está en COMBAT y ejecuta un swap a t=0s,
**WHEN** la animación finaliza a t=0.8s,
**THEN** `is_swap_on_cooldown = true`, `swap_cooldown_remaining` comienza en 5.0s; a t=5.8s `is_swap_on_cooldown = false`.

**CA-LBM-013** — *Sin cooldown en exploración*
**GIVEN** Edrick está en estado EXPLORACIÓN,
**WHEN** ejecuta un swap y la animación de 0.8s finaliza,
**THEN** `is_swap_on_cooldown = false` inmediatamente — el jugador puede ejecutar otro swap de forma inmediata.

### Fórmula HP_MAX

**CA-LBM-014** — *Fuego + Arcano = 80 HP*
**GIVEN** Edrick tiene equipados Fuego (`hp_bonus=+5`) y Arcano (`hp_bonus=0`),
**WHEN** el sistema calcula `HP_MAX = clamp(75+5+0, 1, 125)`,
**THEN** `HP_MAX = 80`.

**CA-LBM-015** — *Hielo + Fuego = 90 HP*
**GIVEN** Edrick tiene equipados Hielo (`hp_bonus=+10`) y Fuego (`hp_bonus=+5`),
**WHEN** el sistema calcula `HP_MAX = clamp(75+10+5, 1, 125)`,
**THEN** `HP_MAX = 90`.

**CA-LBM-016** — *Visión + Dash = 70 HP*
**GIVEN** Edrick tiene equipados Visión (`hp_bonus=-5`) y Dash (`hp_bonus=0`),
**WHEN** el sistema calcula `HP_MAX = clamp(75+(-5)+0, 1, 125)`,
**THEN** `HP_MAX = 70`.

**CA-LBM-017** — *Sin demonios = 75 HP*
**GIVEN** Edrick no tiene ningún demonio equipado,
**WHEN** el sistema calcula `HP_MAX = clamp(75+0, 1, 125)`,
**THEN** `HP_MAX = 75`.

**CA-LBM-018** — *Clamp piso en HP_FLOOR = 1*
**GIVEN** una configuración donde `hp_bonus` total suma -80 (test de borde),
**WHEN** el sistema calcula `HP_MAX = clamp(75+(-80), 1, 125) = clamp(-5, 1, 125)`,
**THEN** `HP_MAX = 1` — nunca retorna 0 ni negativo.

**CA-LBM-019** — *Clamp techo en HP_CEILING = 125*
**GIVEN** una configuración donde `hp_bonus` total suma +100 (test de borde),
**WHEN** el sistema calcula `HP_MAX = clamp(75+100, 1, 125) = clamp(175, 1, 125)`,
**THEN** `HP_MAX = 125` — nunca supera HP_CEILING.

**CA-LBM-020** — *HP actual se reduce si supera nuevo HP_MAX*
**GIVEN** Edrick tiene HP actual = 80 y HP_MAX = 85 (Hielo equipado),
**WHEN** el jugador limpia el slot de Hielo y la animación de 0.8s finaliza,
**THEN** `HP_MAX = 75` y el HP actual de Edrick se reduce automáticamente de 80 a 75.

**CA-LBM-021** — *HP actual no cambia si no supera nuevo HP_MAX*
**GIVEN** Edrick tiene HP actual = 60 y HP_MAX = 80 (Fuego equipado),
**WHEN** el jugador intercambia Fuego por Arcano (nuevo `HP_MAX = 75`) y la animación finaliza,
**THEN** HP actual permanece en 60 — 60 < 75, no hay reducción.

### Aplicación de Cambios

**CA-LBM-022** — *Cambios aplican al FINALIZAR la animación, no al iniciarla*
**GIVEN** Edrick tiene Fuego equipado (HP_MAX = 80, HP actual = 78) y inicia swap a Visión a t=0s (HP_MAX futuro = 70),
**WHEN** se mide el estado a t=0.4s (mitad de la animación),
**THEN** HP_MAX sigue siendo 80 y HP actual sigue siendo 78 — el cambio solo ocurre a t=0.8s.

**CA-LBM-023** — *Modificadores del demonio anterior cesan al finalizar animación*
**GIVEN** Edrick tiene Hielo equipado (con su modificador de velocidad activo),
**WHEN** el jugador intercambia Hielo por Fuego y la animación de 0.8s finaliza,
**THEN** el modificador de velocidad de Hielo ya no está activo — Edrick opera con los modificadores de Fuego únicamente.

**CA-LBM-024** — *Motor de Sinergias recibe notificación al finalizar animación*
**GIVEN** el Motor de Sinergias está activo y Edrick tiene Fuego + Hielo equipados,
**WHEN** el jugador intercambia Hielo por Arcano y la animación finaliza,
**THEN** el Motor de Sinergias recibe `loadout_changed(equipped_demons: [Fuego, Arcano], gato_available: <valor_actual_de_gato_available>)` — la sinergia Fuego+Hielo cesa y la de Fuego+Arcano se activa.

**CA-LBM-029a** — *EventBus emite `loadout_changed` exactamente una vez al finalizar animación de swap*
**GIVEN** cualquier sistema suscrito al EventBus escucha `loadout_changed`,
**WHEN** una operación de loadout (equipar, intercambiar o limpiar) completa su animación de 0.8s,
**THEN** el EventBus emite `loadout_changed(equipped_demons: Array[String], gato_available: bool)` exactamente una vez — no al iniciar la animación ni durante ella. El payload contiene el estado post-swap de `equipped_demons` y el valor actual de `gato_available`.

**CA-LBM-029b** — *Loadout re-emite `loadout_changed` al cambiar el estado narrativo del Gato (sin swap)*
**GIVEN** el Motor de Sinergias está activo y no hay operación de swap en curso,
**WHEN** Estado del Mundo llama a `notify_gato_available(false)` (Gato desaparece narrativamente),
**THEN** el EventBus emite `loadout_changed(equipped_demons: <sin_cambios>, gato_available: false)` exactamente una vez — `equipped_demons` es idéntico al valor anterior, solo `gato_available` cambia.

### Casos Extremos

**CA-LBM-025** — *Swap durante animación SWAP_ANIM es ignorado*
**GIVEN** Edrick está en medio de una animación de swap (t=0.4s),
**WHEN** el jugador intenta iniciar otro swap,
**THEN** la segunda operación es ignorada: no se encola, no interrumpe la animación en curso.

**CA-LBM-026** — *Swap durante cooldown de combate es rechazado*
**GIVEN** Edrick está en COMBAT con `is_swap_on_cooldown = true`,
**WHEN** el jugador intenta ejecutar un swap desde la Build Screen,
**THEN** el sistema no inicia ninguna animación, no modifica `equipped_demons`, y la Build Screen muestra el cooldown restante.

**CA-LBM-027** — *Slots vacíos: Edrick opera solo con HP_BASE sin habilidades*
**GIVEN** Edrick no tiene ningún demonio equipado,
**WHEN** Edrick está en combate,
**THEN** `HP_MAX = 75`, ninguna habilidad especial está disponible, y el Motor de Sinergias no tiene sinergias activas.

**CA-LBM-028** — *Demonio no disponible no puede equiparse*
**GIVEN** un demonio no está en `demonios_disponibles` del jugador,
**WHEN** el sistema recibe una operación de equipar ese demonio,
**THEN** el sistema rechaza la operación: `equipped_demons` no se modifica y no se inicia ninguna animación.

### Build Screen y Acceso

**CA-LBM-035** — *Cooldown se cancela al salir de combate*
**GIVEN** Edrick está en COMBAT con `is_swap_on_cooldown = true` y `swap_cooldown_remaining = 3.0s`,
**WHEN** el combate termina (estado transiciona a EXPLORACIÓN),
**THEN** `is_swap_on_cooldown = false` inmediatamente — el cooldown no continúa corriendo fuera del combate.

**CA-LBM-036** — *Build Screen pausa el juego en combate (confirmar)*
**GIVEN** Edrick está en estado COMBAT con al menos un enemigo activo en escena,
**WHEN** el jugador abre la Build Screen y confirma una operación de swap,
**THEN** el tiempo del juego estaba pausado durante la navegación y reanuda al confirmar (SWAP_ANIM comienza, la Build Screen se cierra automáticamente).

**CA-LBM-038** — *Cerrar Build Screen sin cambios reanuda tiempo en combate*
**GIVEN** Edrick está en COMBAT, la Build Screen está abierta (tiempo pausado),
**WHEN** el jugador cierra la Build Screen sin haber ejecutado ninguna operación,
**THEN** el tiempo reanuda inmediatamente: `get_tree().paused == false` y los enemigos retoman su movimiento. No se inicia ninguna animación ni cooldown.

**CA-LBM-039** — *Build Screen en EXPLORACIÓN no pausa el tiempo*
**GIVEN** Edrick está en estado EXPLORACIÓN,
**WHEN** el jugador abre la Build Screen,
**THEN** el tiempo no se pausa: `get_tree().paused` permanece `false` durante toda la navegación del menú.

**CA-LBM-040** — *Combate termina durante SWAP_ANIM: sin cooldown*
**GIVEN** Edrick inicia un swap en COMBAT a t=0s,
**WHEN** el último enemigo muere a t=0.4s (durante la animación) y el estado transiciona a EXPLORACIÓN antes de que la animación finalice,
**THEN** al finalizar la animación a t=0.8s, `is_swap_on_cooldown = false` — el estado al finalizar es EXPLORACIÓN, no COMBAT, por lo que el cooldown no aplica.

**CA-LBM-037** — *Swap en slot 1 no afecta el estado del slot 2*
**GIVEN** el slot 1 tiene "Fuego" equipado y el slot 2 tiene "Hielo" equipado,
**WHEN** el jugador intercambia el slot 1 (Fuego → Arcano) y la animación de 0.8s finaliza,
**THEN** el slot 2 sigue mostrando "Hielo", los modificadores de Hielo siguen activos, y `equipped_demons = [Arcano, Hielo]`.

### UI y Feedback Visual (ADVISORY)

**CA-LBM-030** — *Gato ausente narrativamente: slot se muestra vacío o "ausente"*
**GIVEN** el Estado del Mundo marca al Gato como no disponible,
**WHEN** el jugador abre la Build Screen,
**THEN** el slot de Gato muestra un indicador visual diferenciado de ausencia — no se muestra al Gato como presente.

**CA-LBM-031** — *Sello de Vínculo con slots al máximo: UI informa el límite*
**GIVEN** el jugador tiene `slot_count = 5` y recoge un Sello de Vínculo,
**WHEN** el sistema procesa el Sello,
**THEN** la UI muestra un mensaje informando que el máximo de slots ha sido alcanzado.

**CA-LBM-032** — *HUD muestra cooldown restante de swap en combate*
**GIVEN** Edrick está en COMBAT con `is_swap_on_cooldown = true`,
**WHEN** el jugador abre la Build Screen,
**THEN** la Build Screen muestra `swap_cooldown_remaining` actualizado en tiempo real mientras los slots no responden a interacción.

### Visual/Feel (ADVISORY)

**CA-LBM-033** — *Transformación visual actualiza al finalizar animación*
**GIVEN** Edrick tiene Fuego equipado y el jugador intercambia por Hielo,
**WHEN** la animación de 0.8s finaliza,
**THEN** el sprite y aura de Edrick actualizan al aspecto visual de Hielo — la transformación NO ocurre antes de que termine la animación.

### Timing Compuesto

**CA-LBM-034** — *Tiempo efectivo entre swaps en combate = 5.8s*
**GIVEN** Edrick está en COMBAT y ejecuta un swap a t=0s,
**WHEN** el jugador intenta el siguiente swap tan pronto como es posible,
**THEN** el primer momento en que el segundo swap puede ejecutarse es t ≥ 5.8s (0.8s animación + 5.0s cooldown, ± 100ms de tolerancia acumulada por 2 timers en cadena — ver §4 Fórmula 3; derivación: 2 × ±50ms = ±100ms). Cualquier intento antes de t=5.70s es rechazado (5.8s − 0.1s = 5.70s).

---

**Resumen de evidencia requerida:**

| Story Type | CAs | Gate | Ubicación de tests |
|---|---|---|---|
| Lógica / fórmulas | CA-001 a 009, 014 a 021, 025 a 029, 037 | BLOCKING | `tests/unit/loadout/` |
| Integración | CA-010 a 013, 022 a 024, 034 a 040 | BLOCKING | `tests/integration/loadout/` |
| UI / Visual | CA-030 a 033 | ADVISORY | `production/qa/evidence/` |

**Flag de implementación (QA):** CA-LBM-022 es el criterio más propenso a implementarse incorrectamente. Se recomienda un test de integración que mida explícitamente el estado a t=0.4s para confirmar que los cambios NO han ocurrido aún. CA-LBM-040 requiere mock de transición de estado durante animación — puede necesitar inyección de estado en el state machine para pruebas unitarias.

---

## 9. Preguntas Abiertas

1. **Animación de apertura/cierre de la Build Screen**: ¿La Build Screen tiene una animación de entrada/salida (fade, slide), o aparece instantáneamente? Afecta la percepción del tiempo de pausa en combate. Resolver antes del GDD #20 (Build Management UI).

2. **Indicador de estado COMBAT activo en el HUD**: Sin un indicador visual persistente de "estoy en combate", la restricción de swap de 5.8s puede parecer un bug la primera vez. ¿El HUD de Combate (#18) muestra un ícono o color distinto cuando `is_swap_on_cooldown` podría aplicar? Resolver en GDD #18.

3. **Sellos de Vínculo — sistema propietario**: Los Sellos de Vínculo se encuentran en exploración de reinos, pero no hay GDD que defina cómo se recogen y persisten items de este tipo. ¿Viven en el GDD de Exploración del Mundo (#8), en Estado del Mundo (#4), o necesitan un GDD de ítems propio? Resolver antes de diseñar los reinos post-MVP.
