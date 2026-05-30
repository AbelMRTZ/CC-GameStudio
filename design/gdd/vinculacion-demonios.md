# GDD: Vinculación de Demonios

> **Estado**: Aprobado
> **Autor**: Abel + Claude Code Agents
> **Última Actualización**: 2026-05-29
> **Implements Pillar**: Pilar 2 — Demonios como Poder Transformador; Pilar 1 — Narrativa Cinematográfica; Pilar 5 — Transformación Moral de Edrick
> **Milestone**: MVP — Feature Layer
> **Depende de**: Base de Datos de Demonios (#3), Motor de Sinergias (#11), Combate en Tiempo Real (#6)
> **Dependen de este sistema**: Progresión Narrativa (#16), Tutorial Integrado (#25)

---

## 1. Visión General

El Sistema de Vinculación de Demonios orquesta el momento más importante del loop de progresión: el instante en que Edrick absorbe un demonio después de matar a su portador. La vía estándar de obtención es inmutable — los demonios no se negocian, no se descubren, no se compran. Se obtienen siempre a través de una muerte. El motivo de esa muerte puede ser muy variado: un portador que ataca a Edrick, un monstruo que guarda el paso de un reino, una criatura inocente que Edrick decide sacrificar de todas formas, o una figura trágica que el jugador lamenta haber eliminado. Pero el acto en sí nunca cambia. El sistema tiene exactamente una excepción diseñada: el Gato, que se une voluntariamente en un encuentro narrativo gestionado por GDD #16. Esa excepción no es un error — es la primera señal de que el Gato no es como los demás.

Internamente, el sistema detecta cuando un portador muere en circunstancias de binding válidas, reproduce la **secuencia de vinculación** — una ceremonia audiovisual breve que suspende el gameplay — y registra el demonio en Estado del Mundo para que Loadout y Motor de Sinergias lo consuman.

El sistema también define el contrato del primer binding: no una pantalla de tutorial, sino el culmen dramático del primer reino. El jugador explora y combate sin la capacidad del primer demonio; el binding al final del primer acto es el culmen dramático del primer reino — y la primera vez que Edrick siente que portar un demonio le cuesta algo más que sangre.

---

## 2. Fantasy del Jugador

La fantasía de la Vinculación opera en dos capas que el jugador experimenta simultánea e irreconciliablemente.

**Capa 1 — La inmovilización involuntaria**: el portador cae, y el mundo se detiene — no porque Edrick lo quiera, sino porque algo sale del portador y lo atraviesa. Edrick no elige el momento de la vinculación. No aprieta ningún botón. El mundo se congela sin su permiso, la ceremonia ocurre, y él solo puede esperar que termine. Eso es la carga antes de que sea el poder. Cuando la secuencia termina y el gameplay reanuda, el mundo es el mismo. Pero Edrick ya no lo es. El jugador descubre la nueva capacidad al usarla, no al leerla en un menú.

**Capa 2 — La sombra sostenida**: el binding no viene con fanfarria. El mundo no para a felicitar a Edrick. La cámara no celebra. Lo que queda es la misma quietud que queda en The Witcher 3 después de un contrato difícil: el trabajo está hecho, el poder está ahí, y sin embargo el ambiente se siente ligeramente peor que antes. El jugador sabe que lo que ganó justificó la muerte, pero algo se torció en el proceso. Edrick también lo sabe — aunque en los primeros bindings lo descarta, y en los últimos ya no lo descarta ni siquiera lo intenta.

Lo que el jugador **no debe sentir nunca**: que "desbloqueó una habilidad". El verbo correcto es "cargó con algo". Cada demonio es una carga antes de ser un poder, y el jugador debe percibir ese orden.

**Alineación de pilares**: Pilar 2 — *"Obtener uno se siente ESPECIAL, ganado, y significativo"*. Pilar 5 — *"la corrupción se siente necesaria, incluso ganada"*.

---

## 3. Diseño Detallado

### 3.1 Reglas Centrales

**A. Estructura de Portador en la Base de Datos**

Cada demonio en `demons.json` (GDD #3) define tres campos que este sistema consume:
```
portador_id: String         # ID único del personaje/criatura que porta el demonio
vinculacion_tipo: String    # "standard" | "custom"
vinculacion_scene: String   # (solo si tipo="custom") ruta a la escena de binding
```

El Sistema de Vinculación carga estos campos en `_ready()` y construye un mapa inverso `portador_id → demon_id` en caché. El Gato no tiene `portador_id` — su binding se gestiona externamente.

---

**B. Detección de Muerte del Portador**

El Sistema de Vinculación escucha la señal unificada `portador_murio(portador_id: String, position: Vector2)` en el EventBus. Esta señal puede ser emitida por:
- **Combate (#6)**: cuando Edrick mata a un portador hostil en combate (Combate detecta si el enemigo es un portador y emite `portador_murio` además de `enemy_died`)
- **Sistema de NPC (#15)**: cuando Edrick mata a un NPC no-hostil que porta un demonio

Al recibir `portador_murio(portador_id, position)`:
1. Verificar si `portador_id` existe en el mapa `portador_id → demon_id`. Si no: ignorar.
2. Verificar si ese demonio ya está en `world_state.available_demons`. Si sí: ignorar (ya vinculado).
3. Verificar si `_edrick_alive == true`. Si no: ignorar (E1).
4. Verificar si el estado actual es IDLE. Si no (secuencia activa): emitir `binding_concurrent_discarded(demon_id)` y ignorar (E4).
5. Si las cuatro condiciones se cumplen: iniciar la secuencia de binding correspondiente.

---

**C. Secuencia Estándar (`vinculacion_tipo = "standard"`)**

La secuencia estándar es automática al detectar la muerte. Dura aproximadamente `BINDING_DURATION` segundos (tunable). **BindingSystem es un Autoload** — persiste entre escenas y gestiona la suscripción al EventBus una única vez en `_ready()`.

1. **Congelación del mundo**: `get_tree().paused = true`. `_sequence_active = true`. Nodos que deben tener `process_mode = ALWAYS`: BindingSystem (Autoload, permanece activo), ParticleContainer (Node2D padre de los sprites individuales), AnimationPlayer de tensión/aura de Edrick, AudioManager (#5). **Referencia a Edrick**: adquirida vía `get_tree().get_first_node_in_group("player")` al inicio de la secuencia — si devuelve null, registrar error y abortar limpiamente: descongela el árbol, `_sequence_active = false`, retorna a IDLE sin crash, demonio no registrado. Temporización: usar `get_tree().create_timer(duration, true)` (argumento `process_always = true`) o Timer nodes con `process_mode = ALWAYS` — los Timer estándar de Godot se pausan con el árbol y colgarían la secuencia indefinidamente.
2. **Estallido desde el portador — fase de explosión** (`OUTBURST_TIME` ≈ 1.0s): `PARTICLE_COUNT` (40) Node2D sprites se crean sobre `portador_position`. Se crea un único Tween paralelo desde BindingSystem: `outburst_tween = create_tween().set_parallel(true)`. Se añaden 40 llamadas `tween_property(sprite[i], "position", scatter_pos[i], OUTBURST_TIME)`, donde `scatter_pos[i] = portador_position + OUTBURST_RADIUS × Vector2(cos(θ[i]), sin(θ[i]))` con ángulos θ[i] distribuidos uniformemente en [0, 2π] (ver F-VD-03). Los sprites explotan radialmente hacia afuera — el portador se dispersa. Color: `transformacion_visual.aura` del demonio en tono **desaturado y oscuro**. Al completar `outburst_tween.finished`, comienza inmediatamente la fase de impacto.
3. **Reversión hacia Edrick — fase de impacto** (`IMPACT_TIME` ≈ 0.5s): Al recibir `outburst_tween.finished`, se crea un segundo Tween paralelo: `impact_tween = create_tween().set_parallel(true)`. Se añaden 40 llamadas `tween_property(sprite[i], "position", edrick_position, IMPACT_TIME)` — duración fija `IMPACT_TIME` para todos los sprites independientemente de la distancia individual (los sprites más lejanos viajan más rápido; todos llegan simultáneamente). La reversión es **abrupta** — las partículas no deceleran antes de girar. Cazan a Edrick. Al completar `impact_tween.finished`: AnimationPlayer reproduce tensión (`TENSION_ANIM_TIME` ≈ 0.3s: Edrick encorva → endereza), luego fade-in de distorsión de aura (`AURA_FADE_TIME`). La aura es una **distorsión oscura y sutil** sobre el sprite de Edrick — mismo `transformacion_visual.aura` pero con baja opacidad y shader de distorsión visual, no una luz. El fade-out del tema de binding comienza aquí — ver §8.2.
4. **Beat de silencio**: `SILENCE_TIME` sin texto ni sonido. Solo Edrick con la distorsión visible mientras el mundo permanece congelado. Edrick se percibe más pesado, no más poderoso.
5. **Descongelación**: `get_tree().paused = false`. `_sequence_active = false`. El gameplay reanuda exactamente donde se detuvo.
6. El sistema emite `binding_sequence_complete(demon_id)` internamente → pasa a fase de Registro (E).

---

**D. Secuencias Custom (`vinculacion_tipo = "custom"`)**

Ciertos demonios (especialmente el Dash — culmen del Acto 1) definen su propia escena de binding. El Sistema de Vinculación:
1. Carga la escena referenciada en `vinculacion_scene`
2. La instancia como `CanvasLayer` sobre el gameplay
3. Pasa datos de contexto a la escena instanciada: `{ demon_id: String, edrick_position: Vector2, portador_position: Vector2 }`. Contrato resuelto (P-VD-02): la escena custom accede a estos tres campos vía propiedades exportadas o mediante `set()` inmediatamente tras `instantiate()`
4. Conecta `binding_sequence_complete` con el flag `CONNECT_ONE_SHOT` para prevenir doble-conexión si la escena custom tiene un bug de re-emisión de señal
5. Escucha la señal `binding_sequence_complete()` que esa escena emite al terminar
6. Cuando la recibe: llama `custom_scene.queue_free()` para liberar la escena de la memoria, luego pasa a Registro (E)

La escena custom es completamente libre en duración, animación, y audio. El Sistema de Vinculación no asume nada de su contenido — solo espera la señal de finalización.

**Timeout de seguridad**: si `binding_sequence_complete()` no llega en `SEQUENCE_CUSTOM_TIMEOUT` segundos (tunable, MVP = 60s), el sistema ejecuta salida de emergencia: descongela el árbol si estaba pausado, llama `custom_scene.queue_free()` para liberar la escena y evitar memory leak, emite `binding_custom_timeout(demon_id)` en el EventBus, el demonio **no** se registra, y el sistema vuelve a `IDLE`.

---

**E. Registro del Demonio (post-secuencia)**

Al recibir `binding_sequence_complete(demon_id)` (estándar o custom):
1. Guard de deduplicación: si `world_state.available_demons.has(demon_id)` ya es verdad, ignorar. En caso contrario, `world_state.available_demons.append(demon_id)`.
2. El Sistema emite `demon_bound(demon_id)` al EventBus global.

Loadout (#10) escucha `demon_bound` y añade el nuevo demonio a la interfaz. Motor de Sinergias (#11) puede consultarlo vía pull API si la nueva disponibilidad activa sinergias visibles.

---

**F. Portadores No-Hostiles**

Algunos portadores no atacan a Edrick. El sistema no distingue entre hostil y no-hostil — si `portador_murio` llega con un `portador_id` válido, el binding ocurre. La responsabilidad de *cómo* murió ese NPC (y el peso narrativo de esa decisión) pertenece al Sistema de NPC (#15) y a Progresión Narrativa (#16). El Sistema de Vinculación no juzga, solo registra.

---

**G. Excepción: El Gato**

El Gato no tiene `portador_id`. Su binding no pasa por este sistema. GDD #16 (Progresión Narrativa) emite `cat_encounter_complete()` en el EventBus cuando el evento narrativo del encuentro con el Gato se completa. El Sistema de Vinculación escucha `cat_encounter_complete()` y ejecuta únicamente el paso de Registro (E) para `"cat"` — sin secuencia de binding propia. El Sistema entonces emite `demon_bound("cat")` exactamente una vez, como cualquier otro demonio. GDD #16 **no** emite `demon_bound` directamente.

**Audio del encuentro con el Gato**: `binding_started` **no se emite** para el Gato — el encuentro ocurre en silencio intencional. El silencio es diseño deliberado: el Gato se une voluntariamente, sin la ceremonia oscura de los demás. GDD #5 no debe esperar ningún evento de audio de Vinculación para `"cat"`.

---

### 3.2 Estados y Transiciones

| Estado | Descripción | Cuándo ocurre |
|--------|-------------|---------------|
| **IDLE** | Escuchando EventBus. Sin binding activo. | Entre bindings |
| **DETECTING** | `portador_murio` recibido — verificando validez, disponibilidad y `_edrick_alive` | < 1 frame |
| **SEQUENCE_STANDARD** | Mundo congelado. Sprites tweenados oscuros + distorsión de aura ejecutándose | ~2.0–3.0s |
| **SEQUENCE_CUSTOM** | Escena custom cargada y ejecutándose. Timeout: `SEQUENCE_CUSTOM_TIMEOUT` (MVP = 60s). Sin señal en ese tiempo → salida de emergencia, demonio no registrado, vuelta a IDLE. | Varía por demonio |
| **REGISTERING** | Escribiendo en Estado del Mundo, emitiendo `demon_bound` | < 1 frame |

Reglas adicionales:
- SEQUENCE_STANDARD y SEQUENCE_CUSTOM son **mutuamente excluyentes** — el sistema solo puede estar en un estado de secuencia a la vez.
- Si un segundo `portador_murio` llega durante un binding activo, el evento se descarta. **Ver E4 — esto es una PROHIBICIÓN DE DISEÑO, no un caso de uso normal.**
- Una vez iniciada la secuencia, **no puede ser interrumpida** por input del jugador.

---

### 3.3 Interacciones con Otros Sistemas

| Sistema | Dirección | Qué fluye | Cuándo |
|---------|-----------|-----------|--------|
| **IA de Enemigos (#7) / Combate (#6)** | → Vinculación | `portador_murio(portador_id, position)` | Cuando Edrick mata a un portador hostil |
| **Sistema de NPC (#15)** | → Vinculación | `portador_murio(portador_id, position)` | Cuando Edrick mata a un portador no-hostil |
| **Base de Datos (#3)** | → Vinculación | `portador_id`, `vinculacion_tipo`, `vinculacion_scene`, `transformacion_visual.aura` | Una vez en `_ready()` |
| **Estado del Mundo (#4)** | ← Vinculación | Escribe `available_demons`, hace demonio disponible | En Registro post-secuencia |
| **Motor de Sinergias (#11)** | ← Vinculación | Vinculación puede consultar `is_synergy_active()` post-binding (dependencia suave) | Opcional, por diseño de secuencia |
| **Progresión Narrativa (#16)** | → Vinculación | Emite `cat_encounter_complete()` cuando el encuentro narrativo del Gato completa — Vinculación escucha y ejecuta solo Registro para `"cat"` | Evento narrativo único |
| **Sistema de Audio (#5)** | ← Vinculación | `binding_started(demon_id)` → reproduce tema de binding | Al iniciar `SEQUENCE_STANDARD` únicamente |
| **Loadout (#10)** | ← Estado del Mundo | Lee `available_demons` actualizado post-`demon_bound` | Inmediato tras Registro |

---

## 4. Fórmulas

### F-VD-01: Condición de Activación de Binding

La fórmula que determina si un `portador_murio` dispara una secuencia de binding:

```
binding_valido(portador_id) =
    (portador_id ∈ portador_to_demon_map)
  ∧ (portador_to_demon_map[portador_id] ∉ world_state.available_demons)
  ∧ (_edrick_alive == true)
```

**Precondición**: `world_state != null` — verificado en `_ready()`. Si `world_state` es null al recibir `portador_murio`, el sistema registra error y descarta el evento.

**Variables:**

| Variable | Símbolo | Tipo | Rango | Descripción |
|----------|---------|------|-------|-------------|
| ID del portador muerto | `portador_id` | String | cualquier String | ID recibido en la señal `portador_murio` |
| Mapa portador → demonio | `portador_to_demon_map` | Dict[String, String] | construido desde `demons.json` en `_ready()` | Solo contiene demonios con `portador_id` definido; Gato excluido |
| Demonios ya disponibles | `world_state.available_demons` | Array[String] | 0–7 elementos (MVP: 6 demonios + Gato) | Campo canónico en Estado del Mundo (#4) |
| Vitalidad local de Edrick | `_edrick_alive` | bool | `true` / `false` | Estado local en BindingSystem (Autoload). Inicializado `true` en `_ready()` (una única vez al arrancar la app). Puesto a `false` al recibir `edrick_died()` de GDD #2. Reseteado a `true` al recibir `edrick_respawned()` de GDD #2 — sin este reset, ningún binding sería posible tras la primera muerte de Edrick. |

**Rango de salida:** bool. O(1) amortizado para el lookup del mapa + O(n) para `Array.has()` sobre `available_demons`, donde n ≤ 7 (MVP). Prácticamente constante dado el límite de demonios.

**Ejemplo:**
- `portador_id = "portador_dash"`, `available_demons = []`, `_edrick_alive = true` → `true` (binding válido)
- `portador_id = "portador_dash"`, `available_demons = ["dash"]`, `_edrick_alive = true` → `false` (ya vinculado, ignorar)
- `portador_id = "portador_dash"`, `_edrick_alive = false` → `false` (Edrick muerto, ignorar)
- `portador_id = "enemigo_generico_3"` (no portador) → `false` (no en mapa, ignorar)

---

### F-VD-02: Desglose de Duración de Secuencia Estándar

```
BINDING_DURATION = OUTBURST_TIME + IMPACT_TIME + TENSION_ANIM_TIME + AURA_FADE_TIME + SILENCE_TIME
                 = 1.0 + 0.5 + 0.3 + 0.5 + 0.3 = 2.6s
```

**Nota importante**: La fase de partículas se divide en dos sub-fases: `OUTBURST_TIME` (explosión radial desde el portador) e `IMPACT_TIME` (reversión hacia Edrick). La tensión y el aura se inician al completar el Tween de impacto (`impact_tween.finished`), no en un timer independiente — garantizando que la animación no se adelante a las partículas independientemente de variaciones de frame.

**Variables:**

| Variable | Símbolo | Tipo | Rango | Descripción |
|----------|---------|------|-------|-------------|
| Tiempo de explosión | `OUTBURST_TIME` | float | [0.6, 1.5]s; MVP = 1.0s | Duración de la fase de explosión radial — sprites salen del portador hacia afuera |
| Tiempo de impacto | `IMPACT_TIME` | float | [0.3, 0.8]s; MVP = 0.5s | Duración de la fase de reversión — sprites viajan desde posiciones de dispersión hasta Edrick simultáneamente |
| Tiempo de animación de tensión | `TENSION_ANIM_TIME` | float | [0.2, 0.5]s; MVP = 0.3s | Duración de la animación de tensión de Edrick (encorva → endereza) al recibir el impacto |
| Tiempo de fade-in del aura | `AURA_FADE_TIME` | float | [0.3, 0.8]s; MVP = 0.5s | Fade-in de la distorsión de aura de Edrick al recibir las partículas |
| Beat de silencio | `SILENCE_TIME` | float | [0.1, 0.5]s; MVP = 0.3s | Pausa tras el aura, antes de descongelar el mundo |
| Duración total | `BINDING_DURATION` | float | [1.4, 3.8]s; MVP = **2.6s** | Duración total del mundo congelado |

**Rango de salida:** 2.6s (MVP). Ajustable dentro del rango sin consecuencias en otros sistemas — el binding es una caja negra desde fuera.

---

### F-VD-03: Posición y Temporización de la Secuencia Bidireccional de Partículas

La secuencia opera en dos fases de duración fija. Los sprites tienen tiempos de llegada sincronizados (todos llegan simultáneamente en cada fase), no velocidades sincronizadas.

**Fase 1 — Explosión (`OUTBURST_TIME`):**
```
scatter_pos[i] = portador_position + OUTBURST_RADIUS × Vector2(cos(θ[i]), sin(θ[i]))
```
donde `θ[i]` son `PARTICLE_COUNT` ángulos distribuidos uniformemente en [0, 2π]. Cada sprite anima desde `portador_position` hasta `scatter_pos[i]` en exactamente `OUTBURST_TIME` segundos — todos los sprites llegan simultáneamente al radio de dispersión.

**Fase 2 — Impacto (`IMPACT_TIME`):**
```
impact_dist[i] = |edrick_position - scatter_pos[i]|
```
Cada sprite anima desde `scatter_pos[i]` hasta `edrick_position` en exactamente `IMPACT_TIME` segundos (duración fija del Tween `tween_property`). Los sprites más lejanos se mueven más rápido; todos llegan a Edrick simultáneamente.

La tensión y el aura se inician al completar el Tween de impacto (`impact_tween.finished`).

**Variables:**

| Variable | Símbolo | Tipo | Rango | Descripción |
|----------|---------|------|-------|-------------|
| Posición del portador | `portador_position` | Vector2 | posición en pantalla al morir | Recibida en `portador_murio(portador_id, position)` |
| Posición de Edrick | `edrick_position` | Vector2 | posición de CharacterBody2D en grupo `"player"` al frame de inicio | Capturada vía `get_tree().get_first_node_in_group("player").global_position`; no recomputada (mundo congelado) |
| Radio de dispersión | `OUTBURST_RADIUS` | float | [80, 250]px; MVP = 150px | Radio desde el portador al que los sprites se dispersan antes de revertir |
| Posición de dispersión i | `scatter_pos[i]` | Vector2 | círculo de radio `OUTBURST_RADIUS` alrededor de `portador_position` | Destino de cada sprite en la fase de explosión |
| Distancia de impacto i | `impact_dist[i]` | float | [0, ~1000]px | Distancia desde `scatter_pos[i]` hasta `edrick_position`; varía por sprite |

**Condiciones de frontera:**
- `distance(portador_position, edrick_position) = 0` (portador muere encima de Edrick): `scatter_pos[i]` están en un círculo de radio `OUTBURST_RADIUS` alrededor de Edrick. `impact_dist[i] = OUTBURST_RADIUS` para todos. Sin crash.
- `impact_dist[i] = 0` para algún sprite (Edrick cae exactamente en un punto de dispersión): probabilidad prácticamente nula dado ángulos aleatorios. Si ocurre, el Tween de impacto para ese sprite completa con distancia 0 en `IMPACT_TIME` — sin crash ni hang.

---

## 5. Casos Extremos

- **E1 — Edrick muere simultáneamente con el portador**: Si `portador_murio` y `edrick_died` llegan en el mismo frame, el binding NO ocurre. BindingSystem mantiene `_edrick_alive: bool` como estado local del Autoload. Se inicializa `true` en `_ready()` (que se ejecuta una única vez al arrancar la app, dado que BindingSystem es Autoload). Se pone a `false` al recibir `edrick_died()` de GDD #2. **Reset al respawn**: BindingSystem escucha también `edrick_respawned()` de GDD #2 y resetea `_edrick_alive = true` al recibirla — sin este reset, ningún binding sería posible después de la primera muerte de Edrick en la sesión. **Nota de dependencia**: GDD #2 (Salud y Daño) debe emitir `edrick_respawned()` en el EventBus cuando Edrick vuelve a estar vivo — ver §6.1. Si `_edrick_alive == false`, el evento se descarta. El portador permanece muerto; el demonio queda inasequible hasta que Edrick reviva. Implicación de diseño: los portadores deben colocarse en situaciones donde la muerte simultánea sea improbable — responsabilidad del Level Design, no del sistema.

- **E2 — El jugador intenta pausar durante la secuencia**: El sistema mantiene un flag `_sequence_active: bool` que se pone a `true` al inicio de cualquier secuencia (paso 1 de §3.1.C) y a `false` al descongelar (paso 5). **Mecanismo mandatado**: BindingSystem implementa `_input(event: InputEvent)` con `process_mode = ALWAYS` y llama `get_viewport().set_input_as_handled()` cuando detecta el evento de pausa y `_sequence_active == true`. Esto encapsula el bloqueo en BindingSystem sin requerir que la UI de pausa conozca el estado del sistema de binding. En Godot 4.6, `get_tree().paused = true` no bloquea input en nodos con `process_mode = ALWAYS` — este mecanismo es obligatorio.

- **E3 — Crash durante la secuencia antes del registro**: Si el juego cierra antes de que `demon_bound` se escriba en Estado del Mundo, el portador está muerto pero el demonio no está en `available_demons`. El demonio queda inasequible al recargar. El guardado automático debe ocurrir **después** del paso de Registro (§3.1.E), no durante la secuencia — responsabilidad de GDD #12 (Guardado y Carga).

- **E4 — PROHIBICIÓN DE DISEÑO: Segundo portador durante binding activo**: Este caso nunca ocurre en el juego por diseño. Level Design garantiza que ningún nivel MVP contiene dos portadores cuya área de combate se solape de manera que puedan morir dentro de la misma ventana de binding. La señal `binding_concurrent_discarded(demon_id)` existe únicamente como instrumento de QA — si se emite durante cualquier test de nivel, ese nivel viola el contrato de diseño y no puede mergearse. El sistema no define comportamiento "normal" para este caso porque ese comportamiento nunca debe ejecutarse.

- **E5 — `portador_murio` duplicado para un demonio ya vinculado**: La condición F-VD-01 evalúa `world_state.available_demons` y devuelve `false`. El evento se descarta sin consecuencias.

- **E6 — `demon_bound("cat")` llega antes del encuentro narrativo esperado**: El sistema ejecuta únicamente el Registro (§3.1.E); el Gato queda disponible en Loadout. Este GDD no valida la narrativa previa — GDD #16 es responsable del orden correcto de sus eventos.

---

## 6. Dependencias

### 6.1 Upstream — Este sistema depende de

| Sistema | GDD | Qué provee | Tipo |
|---------|-----|-----------|------|
| Base de Datos de Demonios | [#3](base-datos-demonios.md) | `portador_id`, `vinculacion_tipo`, `vinculacion_scene`, color de aura por demonio | Datos — lectura en `_ready()` |
| Estado del Mundo | [#4](estado-del-mundo.md) | Campo `available_demons` — lectura en F-VD-01; escritura en Registro §3.1.E | Datos — lectura/escritura |
| Salud y Daño | [#2](salud-daño.md) | Señales `edrick_died()` → pone `_edrick_alive = false` (E1); `edrick_respawned()` → resetea `_edrick_alive = true` (E1). **GDD #2 debe añadir la señal `edrick_respawned()`** — actualmente no existe en su GDD. Sin este reset, ningún binding es posible tras la primera muerte de Edrick en una sesión (BindingSystem es Autoload, `_ready()` se ejecuta una sola vez). | Evento — señal entrante |
| Sistema de Audio | [#5](sistema-audio.md) | Framework de audio que consume `binding_started(demon_id)`. **Nota pendiente**: GDD #5 debe ser actualizado para añadir el estado BINDING, el handler de `binding_started`, el handler de `binding_custom_timeout`, y la distinción entre gameplay-pause (este sistema) y player-pause. Ver §8.2 y P-VD-04. | Servicio — listener externo |
| Combate en Tiempo Real | [#6](combate-tiempo-real.md) | Señal `portador_murio(portador_id, position)` al matar portador hostil | Evento — señal entrante |
| Sistema de NPC y Diálogo | [#15](sistema-npc-dialogo.md) | Señal `portador_murio(portador_id, position)` al matar portador no-hostil | Evento — señal entrante |
| Progresión Narrativa | [#16](—) | Señal `cat_encounter_complete()` cuando el encuentro con el Gato completa — dispara Registro para `"cat"` sin secuencia visual | Evento — señal entrante (solo Gato) |
| Motor de Sinergias | [#11](motor-sinergias.md) | Pull API `is_synergy_active()` / `get_synergy_effect()` post-binding | API — consulta opcional (dependencia suave) |

### 6.2 Downstream — Sistemas que dependen de este

| Sistema | GDD | Qué consumen | Cuándo |
|---------|-----|-------------|--------|
| Loadout & Build Management | [#10](loadout-build-management.md) | `demon_bound(demon_id)` → añade demonio disponible a la interfaz | Inmediato tras Registro |
| Motor de Sinergias | [#11](motor-sinergias.md) | `demon_bound(demon_id)` → re-evalúa sinergias activas | Inmediato tras Registro |
| Progresión Narrativa | [#16](—) | `demon_bound(demon_id)` → dispara gates narrativos post-binding | Tras Registro (por cada demonio) |
| Tutorial Integrado | [#25](—) | `demon_bound(demon_id)` → trigger de guía post-primer-binding | Milestone Alpha |

### 6.3 Señales Emitidas por Este Sistema

| Señal | Destinatarios | Momento |
|-------|--------------|---------|
| `binding_started(demon_id: String)` | Sistema de Audio (#5) | Al iniciar `SEQUENCE_STANDARD` únicamente. Las escenas custom gestionan su propio audio — esta señal no se emite para `SEQUENCE_CUSTOM`. No se emite para el Gato (silencio intencional — ver §3.1.G). |
| `demon_bound(demon_id: String)` | Loadout (#10), Motor de Sinergias (#11), Progresión Narrativa (#16), Tutorial (#25) | Al completar Registro (standard, custom, o Gato) |
| `binding_concurrent_discarded(demon_id: String)` | QA / Level Design tools | Cuando un `portador_murio` se descarta por binding activo (E4). Solo diagnóstico — no consumir en gameplay. Si se emite en cualquier test de nivel, ese nivel viola el contrato de diseño. |
| `binding_custom_timeout(demon_id: String)` | QA / EventBus; Sistema de Audio (#5) para limpiar estado de audio si estaba activo | Cuando `SEQUENCE_CUSTOM` no recibe `binding_sequence_complete()` en `SEQUENCE_CUSTOM_TIMEOUT` segundos. Salida de emergencia — demonio no registrado. |

### 6.4 Nota de Bidireccionalidad

La verificación formal de que los GDDs upstream mencionan a Vinculación como dependiente suyo se ejecuta durante `/design-review` (sesión separada). Se sabe que GDD #3 §9.4 D-W4 delega explícitamente la secuencia de primer binding a este GDD — verificado en esta sesión. Los demás GDDs upstream (#4, #5, #6, #11, #15) deben ser auditados en la revisión.

---

## 7. Parámetros de Ajuste

### 7.1 Duración de Secuencia Estándar

| Parámetro | Valor MVP | Rango Seguro | Efecto en Gameplay |
|-----------|-----------|-------------|-------------------|
| `OUTBURST_TIME` | 1.0s | [0.6, 1.5]s | Duración de la explosión radial desde el portador. Por debajo de 0.6s la explosión es casi imperceptible antes de la reversión; por encima de 1.5s el jugador pierde el hilo de qué está pasando. |
| `IMPACT_TIME` | 0.5s | [0.3, 0.8]s | Duración de la reversión hacia Edrick. Por debajo de 0.3s parece teleportación; por encima de 0.8s diluye el efecto de "caza". La reversión debe sentirse abrupta. |
| `TENSION_ANIM_TIME` | 0.3s | [0.2, 0.5]s | Duración de la animación de tensión de Edrick (encorva → endereza) al recibir el impacto. Incluida en `BINDING_DURATION`. |
| `AURA_FADE_TIME` | 0.5s | [0.3, 0.8]s | Velocidad a la que la distorsión de aura aparece en Edrick. Por debajo de 0.3s pasa desapercibida; por encima de 0.8s interrumpe el ritmo. |
| `SILENCE_TIME` | 0.3s | [0.1, 0.5]s | El beat de silencio que da peso al momento. Por debajo de 0.1s desaparece el impacto emocional; por encima de 0.5s el jugador lee la secuencia como "colgada". |
| `BINDING_DURATION` (compuesto) | **2.6s** | [1.4, 3.8]s | Duración total del mundo congelado. No ajustar directamente — derivado de los cinco parámetros anteriores (OUTBURST_TIME + IMPACT_TIME + TENSION_ANIM_TIME + AURA_FADE_TIME + SILENCE_TIME). |

**Fuera del rango seguro:** Por encima de 3.8s totales, referencias de playtesting (Hollow Knight, Celeste) indican que pausas del mundo de más de 3.5s rompen el ritmo de flujo percibido. Por debajo de 1.4s el binding pasa sin registrarse emocionalmente.

### 7.2 Comportamiento de Partículas

| Parámetro | Valor MVP | Rango Seguro | Efecto en Gameplay |
|-----------|-----------|-------------|-------------------|
| `OUTBURST_RADIUS` | 150 px | [80, 250] px | Radio de dispersión desde el portador en la fase de explosión. Por debajo de 80px la explosión se confunde visualmente con el impacto; por encima de 250px las partículas pueden salir de pantalla antes de revertir. |
| `PARTICLE_COUNT` | 40 | [20, 60] | Número de Node2D sprites tweenados. Por debajo de 20 la nube parece escasa; por encima de 60 hay impacto en rendimiento (instanciado + Tween activo de 60+ nodos). |

### 7.3 Secuencias Custom

Las secuencias custom (por demonio, incluyendo el Dash) no tienen parámetros de contenido en este sistema — sus tiempos y efectos son internos a su propia escena (`vinculacion_scene`). Este sistema solo espera la señal `binding_sequence_complete()`.

| Parámetro | Valor MVP | Rango Seguro | Efecto en Gameplay |
|-----------|-----------|-------------|-------------------|
| `SEQUENCE_CUSTOM_TIMEOUT` | 60s | [30, 120]s | Tiempo máximo de espera para `binding_sequence_complete()`. Por debajo de 30s puede cortar escenas custom largas legítimas; por encima de 120s el jugador ya habrá reportado el juego como colgado antes del timeout. |

### 7.4 Audio de Binding

| Parámetro | Valor MVP | Rango Seguro | Efecto en Gameplay |
|-----------|-----------|-------------|-------------------|
| `BINDING_THEME_FADEOUT_TIME` | 0.5s | [0.3, 1.0]s | Duración del fade-out del tema de binding. El fade comienza al inicio de `AURA_FADE_TIME` (al completar los Tweens de sprites), no al inicio de `SILENCE_TIME` — da tiempo al silencio para ser completamente audible antes de que el gameplay reanude. |

---

## 8. Requisitos Visuales y de Audio

### 8.1 Secuencia Estándar — Visual

**Sprites oscuros — secuencia bidireccional (F-VD-03):**
- Tipo: `PARTICLE_COUNT` (40) Node2D sprites gestionados por dos Tweens paralelos secuenciales (`outburst_tween` + `impact_tween`) — ver §3.1.C
- Fase 1 — Explosión (`OUTBURST_TIME` ≈ 1.0s): sprites parten de `portador_position` y explotan radialmente hacia afuera, alcanzando `scatter_pos[i]` (radio `OUTBURST_RADIUS` ≈ 150px). El portador se dispersa.
- Fase 2 — Impacto (`IMPACT_TIME` ≈ 0.5s): sprites invierten sin deceleración y convergen en `edrick_position`. Todos llegan simultáneamente. Cazan a Edrick.
- Referencia a Edrick: `get_tree().get_first_node_in_group("player")` (`CharacterBody2D`, no nodo visual). Capturada una vez al inicio de la secuencia; no recomputada (mundo congelado). Null guard: si no existe, abortar la secuencia limpiamente (descongela árbol antes de abortar).
- Color: `transformacion_visual.aura` del demonio en tono **desaturado/oscuro** — no brillante. Residuo del portador, no don.
- Optimización: un único `ShaderMaterial` compartido entre los 40 sprites — no crear un Material por sprite.
- **Test de Gramática Visual**: "¿Esta secuencia se lee como: (A) recibir un regalo / (B) ser cazado y asaltado / (C) otra cosa?" Target: **(B)**. Si en playtest los testers usan vocabulario de adquisición ("absorbí", "conseguí"), la dirección o el lenguaje corporal de Edrick deben revisarse antes de Alpha.

**Distorsión de aura de Edrick:**
- Al completar `impact_tween.finished`: AnimationPlayer reproduce tensión de Edrick (`TENSION_ANIM_TIME` ≈ 0.3s: encorva → endereza), luego fade-in de la distorsión de aura (`AURA_FADE_TIME` = 0.5s)
- AnimationPlayer: referenciado vía `get_tree().get_first_node_in_group("player").get_node("AnimationPlayer")` — mismo null guard que CharacterBody2D; si no existe, omitir la animación de tensión sin crash.
- Efecto: distorsión oscura y sutil sobre el sprite de Edrick — mismo `transformacion_visual.aura` pero con baja opacidad y shader de distorsión visual, **no una luz ni brillo**
- La tensión de Edrick comunica peso recibido, no poder ganado
- La distorsión de aura persiste después de la secuencia — primera manifestación visual permanente del demonio en Edrick (GDD #14 toma ownership al recibir `demon_bound`)

### 8.2 Secuencia Estándar — Audio

**Requisito de nodo**: AudioManager (GDD #5) debe tener `process_mode = ALWAYS` para reproducir durante la congelación del mundo. Sin esta configuración, el AudioStreamPlayer se pausa con el árbol y el silencio durante la secuencia sería involuntario — contradiciendo el diseño.

| Momento | Evento de Audio | Nota |
|---------|----------------|------|
| Inicio de `SEQUENCE_STANDARD` | `binding_started(demon_id)` emitido al EventBus | Audio #5 reproduce el tema de binding del demonio. Tema sombrío y de peso — no un stinger de victoria ni fanfarrón heroico. Convención de nombre de asset: `mus_binding_[demon_id]_stinger.ogg`. Fallback obligatorio: si el asset específico no existe, Audio #5 reproduce `mus_binding_fallback.ogg` (asset obligatorio de pipeline — nunca omitir). `binding_started` solo se emite después de que F-VD-01 pasa completamente y la secuencia inicia — nunca antes de que el evento sea validado. |
| Al completar Tweens (inicio de `AURA_FADE_TIME`) | Fade-out del tema con duración `BINDING_THEME_FADEOUT_TIME` | El fade comienza aquí, no al inicio de `SILENCE_TIME`. Permite que el silencio sea completamente audible. |
| Beat de silencio (`SILENCE_TIME`) | Silencio completo | El tema ya está en fade-out o silenciado. No compite con el momento emocional. |
| Descongelación | Sin evento de audio específico | La música de gameplay puede reanudar en el siguiente frame |
| Encuentro del Gato | **Sin evento de audio** | `binding_started` no se emite para `"cat"`. Silencio intencional — GDD #5 no debe esperar ningún evento de Vinculación para el Gato. |

**Actualización requerida de GDD #5 (sistema-audio.md)** — este GDD asume los siguientes contratos en Audio que actualmente no existen:
1. Estado `BINDING` con handler de `binding_started(demon_id)` → inicia `mus_binding_[demon_id]_stinger.ogg`
2. Handler de `binding_custom_timeout(demon_id)` → fuerza parada de audio de binding si está activo
3. Distinción entre gameplay-pause (`get_tree().paused = true` durante binding) y player-pause (menú de pausa) — AudioManager debe continuar reproduciendo durante gameplay-pause gracias a `process_mode = ALWAYS`
4. `force_stop_audio(demon_id)` como contrato opcional para escenas custom (ver §8.3)

GDD #5 debe ser actualizado antes de implementar la secuencia de audio. Ver P-VD-04.

### 8.3 Secuencias Custom — Contrato Visual/Audio

Las escenas custom gestionan su propio audio y visuales internamente. El Sistema de Vinculación no impone restricciones de contenido, solo de contrato:
- La escena se instancia como `CanvasLayer` y renderiza por encima del gameplay
- La escena recibe `{ demon_id: String, edrick_position: Vector2, portador_position: Vector2 }` al instanciarla (ver §3.1.D)
- La escena debe emitir la señal `binding_sequence_complete()` al terminar — sin esta señal el sistema queda en estado `SEQUENCE_CUSTOM` hasta el timeout
- El Sistema de Vinculación no emite `binding_started` para secuencias custom — la escena custom es responsable de su propio audio
- **Lifecycle**: BindingSystem llama `custom_scene.queue_free()` en ambas rutas de salida — señal recibida Y timeout. La escena custom no es responsable de liberarse a sí misma.
- **Contrato opcional de audio**: Si la escena custom gestiona su propio AudioStreamPlayer y quiere que BindingSystem fuerce su parada en caso de timeout, puede exponer un método `force_stop_audio()`. BindingSystem lo llamará antes de `queue_free()` si existe. Este contrato es opcional — sin él, el `queue_free()` libera todos los nodos hijo incluyendo audio.

---

## 9. Criterios de Aceptación

### Detección (AC-VD-001 a AC-VD-006)

**AC-VD-001** — F-VD-01 Ejecutado Correctamente con las Tres Condiciones
**GIVEN** `portador_to_demon_map` contiene `"portador_fuego" → "fuego"`, `world_state.available_demons = []`, `_edrick_alive = true`, **WHEN** el EventBus emite `portador_murio("portador_fuego", Vector2(300, 200))`, **THEN** el sistema evalúa las tres condiciones de F-VD-01 y transiciona a `DETECTING`.
**Tipo**: Unit | **Bloquea**: Sí

**AC-VD-002** — Portador No Registrado se Descarta Silenciosamente
**GIVEN** `portador_to_demon_map` no contiene `"enemigo_generico_7"`, `_edrick_alive = true`, **WHEN** se emite `portador_murio("enemigo_generico_7", ...)`, **THEN** el sistema permanece en `IDLE`, no inicia secuencia, no emite señales.
**Tipo**: Unit | **Bloquea**: Sí

**AC-VD-003** — Demonio Ya Disponible se Descarta Silenciosamente
**GIVEN** `"fuego"` está en `world_state.available_demons`, **WHEN** se emite `portador_murio("portador_fuego", ...)`, **THEN** F-VD-01 evalúa `false` (segunda condición falla), sistema permanece en `IDLE`.
**Tipo**: Unit | **Bloquea**: Sí

**AC-VD-004** — Binding Válido Activa la Secuencia
**GIVEN** `"dash"` no está en `available_demons`, su portador está en el mapa, y `_edrick_alive = true`, **WHEN** se emite `portador_murio("portador_dash", ...)`, **THEN** sistema transiciona `IDLE → DETECTING → SEQUENCE` en el mismo frame.
**Tipo**: Unit | **Bloquea**: Sí

**AC-VD-005** — Mapa Construido Correctamente en `_ready()`
**GIVEN** `demons.json` tiene 4 demonios con `portador_id` y 1 sin (el Gato), **WHEN** `_ready()` ejecuta, **THEN** `portador_to_demon_map` tiene exactamente 4 entradas, sin entrada para el Gato.
**Tipo**: Unit | **Bloquea**: Sí

**AC-VD-006** — `portador_murio` Admite Cualquier Emisor
**GIVEN** el sistema escucha `portador_murio` en EventBus global, **WHEN** la señal es emitida por Combate (#6) y luego por NPC (#15) en momentos distintos, **THEN** ambas disparan F-VD-01 de forma idéntica, sin distinción por emisor.
**Tipo**: Integration | **Bloquea**: Sí

---

### Secuencia Estándar (AC-VD-007 a AC-VD-015)

**AC-VD-007** — Mundo Se Congela al Inicio
**WHEN** estado transiciona a `SEQUENCE_STANDARD`, **THEN** `get_tree().paused == true` y `_sequence_active == true` en el mismo frame, antes de procesar input del jugador.
**Tipo**: Unit | **Bloquea**: Sí

**AC-VD-008** — Secuencia Bidireccional: Posiciones Correctas en Ambas Fases
**GIVEN** portador en `Vector2(500, 400)`, Edrick en `Vector2(200, 300)`, `OUTBURST_RADIUS = 150px`, **WHEN** la secuencia inicia, **THEN**: (a) en la fase de explosión, todos los sprites parten de `Vector2(500, 400)` y sus destinos `scatter_pos[i]` están a exactamente 150px de `Vector2(500, 400)` (tolerancia ±0.1px por float); (b) en la fase de impacto, todos los sprites tienen destino `Vector2(200, 300)` y el Tween de impacto tiene duración fija `IMPACT_TIME` para todos los sprites.
**Tipo**: Unit | **Bloquea**: Sí

**AC-VD-009** — F-VD-03: Posiciones de Dispersión Correctas
**GIVEN** `portador_position = Vector2(0, 0)`, `OUTBURST_RADIUS = 150px`, `PARTICLE_COUNT = 40`, **WHEN** se calculan `scatter_pos[i]`, **THEN** los 40 puntos están distribuidos en ángulos separados 9° (360° / 40) y todos a exactamente 150px de `Vector2(0, 0)` (tolerancia: `|distance(scatter_pos[i], portador_position) - 150| ≤ 0.01px` por precisión float).
**Tipo**: Unit | **Bloquea**: Sí

**AC-VD-010** — F-VD-03: Condición de Frontera — Portador y Edrick en la Misma Posición
**GIVEN** `portador_position = edrick_position = Vector2(100, 100)`, `OUTBURST_RADIUS = 150px`, **WHEN** la secuencia inicia, **THEN** los sprites se dispersan en un círculo de radio 150px alrededor de `Vector2(100, 100)` en la fase de explosión, y en la fase de impacto todos convergen de vuelta en `Vector2(100, 100)` en `IMPACT_TIME` segundos. Sin crash ni hang — el Tween de duración `IMPACT_TIME` con distancia ≈ 150px es completamente válido.
**Tipo**: Unit | **Bloquea**: Sí

**AC-VD-011a** — Tensión y Aura se Inician al Completar el Tween de Impacto
**WHEN** `impact_tween.finished` dispara (todos los sprites han llegado a `edrick_position`), **THEN** AnimationPlayer inicia la animación de tensión de Edrick en el mismo frame. La tensión y el aura **no** se inician en un timer de `OUTBURST_TIME + IMPACT_TIME` segundos independiente. Test en GUT: verificar que `impact_tween` contiene exactamente `PARTICLE_COUNT` propiedades tweenadas con duración `IMPACT_TIME` y destino `edrick_position` — aserción sobre los parámetros del Tween, no sobre tiempo real de ejecución.
**Tipo**: Unit | **Bloquea**: Sí

**AC-VD-011b** — Aura Oscura: Distorsión, No Brillo
**WHEN** la distorsión de aura es visible, **THEN** el efecto visual usa tono desaturado/oscuro del color `transformacion_visual.aura` del demonio, sin luz emitida ni partículas brillantes.
**Tipo**: Visual / Playtest | **Bloquea**: No (Advisory)

**AC-VD-012** — F-VD-02: Suma de Parámetros de Duración = 2.6s
**GIVEN** valores MVP de las constantes exportadas de BindingSystem, **WHEN** se evalúa F-VD-02, **THEN** `OUTBURST_TIME + IMPACT_TIME + TENSION_ANIM_TIME + AURA_FADE_TIME + SILENCE_TIME = 1.0 + 0.5 + 0.3 + 0.5 + 0.3 = 2.6s`. Test en GUT: aserción directa sobre los valores de las constantes — no requiere medición de tiempo real de ejecución (que varía en runners headless).
**Tipo**: Unit | **Bloquea**: Sí

**AC-VD-013** — Mundo Se Descongela al Terminar y Flag Se Limpia
**WHEN** `SILENCE_TIME` termina, **THEN** `get_tree().paused = false`, `_sequence_active = false`, input del jugador vuelve a procesarse, IA de enemigos reanuda.
**Tipo**: Integration | **Bloquea**: Sí

**AC-VD-014** — `binding_started` Emitido al Inicio
**WHEN** estado entra en `SEQUENCE_STANDARD`, **THEN** EventBus emite `binding_started("fuego")` antes del primer frame de partículas. Exactamente una vez por binding.
**Tipo**: Unit | **Bloquea**: Sí

**AC-VD-015** — Referencia a Edrick Adquirida por Group Lookup con Null Guard
**GIVEN** Edrick existe en el grupo `"player"`, **WHEN** la secuencia inicia, **THEN** BindingSystem adquiere la referencia vía `get_tree().get_first_node_in_group("player")` y el destino de las partículas (fase 2) es esa `global_position` — no recomputada durante la secuencia (mundo congelado). **GIVEN** no hay nodo en el grupo `"player"` al iniciar, **THEN** BindingSystem registra error, descongela el árbol si estaba pausado, `_sequence_active = false`, retorna a IDLE sin crash — demonio no registrado.
**Tipo**: Unit | **Bloquea**: Sí

---

### Secuencia Custom (AC-VD-016 a AC-VD-019)

**AC-VD-016** — Escena Custom Cargada como CanvasLayer
**GIVEN** `"dash"` tiene `vinculacion_tipo = "custom"` y ruta de escena definida, **WHEN** se detecta binding válido, **THEN** la escena se instancia como `CanvasLayer` por encima del gameplay.
**Tipo**: Integration | **Bloquea**: Sí

**AC-VD-017** — Sistema Espera Señal sin Registrar
**WHEN** escena custom activa y no ha emitido `binding_sequence_complete()`, **THEN** sistema en `SEQUENCE_CUSTOM`, `available_demons` sin cambios, `demon_bound` no emitido.
**Tipo**: Unit | **Bloquea**: Sí

**AC-VD-018** — Registro Ocurre Solo Tras la Señal
**WHEN** escena custom emite `binding_sequence_complete()`, **THEN** sistema transiciona a `REGISTERING` en el mismo frame que recibe la señal.
**Tipo**: Integration | **Bloquea**: Sí

**AC-VD-019** — Sin Señal: Timeout Actúa y Libera la Escena
**GIVEN** escena custom activa y transcurren `SEQUENCE_CUSTOM_TIMEOUT` segundos (MVP = 60s) sin señal `binding_sequence_complete()`, **THEN** sistema emite `binding_custom_timeout(demon_id)`, llama `custom_scene.queue_free()`, el árbol se descongela si estaba pausado, el sistema vuelve a `IDLE`, y `available_demons` no contiene el demonio. Sin crash ni loop infinito.
**Tipo**: Unit | **Bloquea**: Sí

---

### Registro y Señales (AC-VD-020 a AC-VD-025)

**AC-VD-020** — Demonio Añadido a `world_state.available_demons`
**WHEN** Registro completa para `"fuego"`, **THEN** `world_state.available_demons.has("fuego") == true`, legible por todos los sistemas downstream.
**Tipo**: Integration | **Bloquea**: Sí

**AC-VD-021** — `demon_bound` Emitido Después de la Escritura
**WHEN** Registro completa, **THEN** EventBus emite `demon_bound("fuego")` exactamente una vez, síncronamente después de la escritura en `available_demons` (no antes).
**Tipo**: Unit | **Bloquea**: Sí

**AC-VD-022** — Loadout Disponibiliza el Demonio
**WHEN** EventBus emite `demon_bound("fuego")`, **THEN** el demonio aparece disponible en la interfaz de Loadout dentro del mismo ciclo de procesamiento de señales.
**Tipo**: Integration | **Bloquea**: Sí

**AC-VD-023** — Escritura Antes de Señal (Orden de Operaciones)
**GIVEN** un listener de prueba suscrito a `demon_bound`, **WHEN** recibe la señal, **THEN** dentro del handler `world_state.available_demons.has("fuego") == true`.
**Tipo**: Unit | **Bloquea**: Sí

**AC-VD-024** — Gato: Solo Registro, Sin Secuencia Visual
**GIVEN** GDD #16 emite `cat_encounter_complete()`, **WHEN** el sistema lo escucha, **THEN** ejecuta únicamente Registro para `"cat"`. Sin sprites tweenados, sin distorsión de aura, sin mundo congelado, sin `binding_started`. El sistema emite `demon_bound("cat")` exactamente una vez.
**Tipo**: Integration | **Bloquea**: Sí

**AC-VD-025** — Gato: Sin Error en Cualquier Momento de la Partida
**WHEN** GDD #16 emite `cat_encounter_complete()` con `"cat"` no en `available_demons`, **THEN** Registro completa sin error, sin secuencia visual, y sin requerir validación de estado narrativo previo. `demon_bound("cat")` emitido exactamente una vez.
**Tipo**: Unit | **Bloquea**: Sí

---

### Casos Extremos (AC-VD-026 a AC-VD-031)

**AC-VD-026** — E1: Edrick Muerto — Binding No Ocurre
**GIVEN** `_edrick_alive = false` (BindingSystem recibió `edrick_died()` de GDD #2), **WHEN** `portador_murio` llega con binding que sería válido según las dos primeras condiciones, **THEN** tercera condición F-VD-01 falla, evento descartado, sistema en `IDLE`, demonio no vinculado.
**Tipo**: Unit | **Bloquea**: Sí

**AC-VD-027** — E2: Input de Pausa Bloqueado por `set_input_as_handled()` en BindingSystem
**GIVEN** `SEQUENCE_STANDARD` activa (`_sequence_active = true`) y BindingSystem tiene `process_mode = ALWAYS`, **WHEN** jugador presiona la acción de pausa, **THEN** BindingSystem detecta el evento en `_input()`, llama `get_viewport().set_input_as_handled()`, y el menú de pausa no aparece. Test: simular `InputEventAction` de pausa cuando `_sequence_active == true` y verificar que ningún nodo de UI de pausa recibe el evento (`is_action_pressed("pause")` queda `false` en ese frame).
**Tipo**: Unit | **Bloquea**: Sí

**AC-VD-028** — E3: Crash Mid-Secuencia — Demonio Inasequible al Recargar *(Verificado en GDD #12)*
**GIVEN** juego cerrado durante `SEQUENCE_STANDARD` antes del Registro, **WHEN** jugador recarga sesión guardada, **THEN** `available_demons` no contiene el demonio. *(Este AC se verifica en la suite de tests de GDD #12, no en esta suite — requiere que GDD #12 documente que el autosave no ocurre durante la secuencia)*
**Tipo**: Cross-system | **Bloquea**: No (Advisory)

**AC-VD-029** — E4: `binding_concurrent_discarded` Emitido Como Señal de QA
**GIVEN** `SEQUENCE_STANDARD` activa para `"fuego"` (caso de test — E4 es una prohibición de diseño, nunca debe ocurrir en niveles reales), **WHEN** llega `portador_murio("portador_hielo", ...)`, **THEN** el segundo evento se descarta, EventBus emite `binding_concurrent_discarded("hielo")`, `"hielo"` no se vincula, binding de `"fuego"` continúa sin interrupción.
**Tipo**: Unit | **Bloquea**: Sí

**AC-VD-030** — E5: `portador_murio` Duplicado para Demonio Ya Disponible
**GIVEN** `"fuego"` en `available_demons`, **WHEN** llega `portador_murio("portador_fuego", ...)` de nuevo, **THEN** F-VD-01 evalúa `false`, sistema en `IDLE`, sin secuencia ni señal.
**Tipo**: Unit | **Bloquea**: Sí

**AC-VD-031** — Exclusión Mutua — Un Solo Binding Activo a la Vez
**GIVEN** `portador_murio` para dos portadores distintos llega en frames consecutivos, **WHEN** el primero ya activó la secuencia, **THEN** el sistema está en exactamente un estado de secuencia — nunca en dos simultáneamente.
**Tipo**: Unit | **Bloquea**: Sí

---

### Rendimiento y Smoke (AC-VD-032 a AC-VD-033)

**AC-VD-032** — Sin Caída de FPS con 40 Sprites Tweenados en Escena Típica
**GIVEN** 40 Node2D sprites con Tween activos en escena MVP (≤ 200 draw calls totales), **WHEN** sprites en vuelo durante `PARTICLE_TRAVEL_TIME`, **THEN** framerate ≥ 60 fps en hardware de desarrollo. Si aún no hay especificación mínima de hardware, documenta como línea de base.
**Tipo**: Unit | **Bloquea**: No (Advisory)

**AC-VD-033** — Smoke Check: Binding de Punta a Punta
**GIVEN** escena de prueba con portador en mapa, Edrick vivo, `available_demons` vacío, **WHEN** portador muere (señal emitida), **THEN** flujo completo: detección → secuencia → registro → `demon_bound` en EventBus → demonio disponible en Loadout. Debe pasar antes de cualquier hand-off a QA manual.
**Tipo**: Integration | **Bloquea**: Sí

---

### Contrato de Datos y Concurrencia (AC-VD-034 a AC-VD-035)

**AC-VD-034** — Custom Scene Data Contract: Tres Campos Accesibles
**GIVEN** escena custom instanciada para el Dash, **WHEN** BindingSystem pasa `{ demon_id, edrick_position, portador_position }` vía `set()` tras `instantiate()`, **THEN** la escena puede leer los tres valores correctamente en su `_ready()`. Test: escena stub que lea los tres campos y confirme que no son null/default.
**Tipo**: Integration | **Bloquea**: Sí

**AC-VD-035** — `binding_concurrent_discarded` No se Emite en Niveles MVP
**GIVEN** todos los niveles MVP integrados en build de test, **WHEN** un run de integración completo simula las muertes de portadores en el orden de diseño (no simultáneo), **THEN** `binding_concurrent_discarded` nunca se emite. Si se emite, el test falla y el nivel que lo causó viola el contrato de diseño E4. *(Requiere harness de integración de nivel completo — no cubre GUT estándar.)*
**Tipo**: Integration | **Bloquea**: Sí

**AC-VD-036** — E1: `_edrick_alive` se Resetea al Respawn
**GIVEN** `_edrick_alive = false` (BindingSystem recibió `edrick_died()` de GDD #2), **WHEN** el EventBus emite `edrick_respawned()` de GDD #2, **THEN** `_edrick_alive = true` y el siguiente `portador_murio` válido dispara correctamente una secuencia de binding (F-VD-01 retorna `true` con las demás condiciones cumplidas).
**Tipo**: Unit | **Bloquea**: Sí

**AC-VD-037** — Guard de Deduplicación en Registro (§3.1.E Paso 1)
**GIVEN** `world_state.available_demons` ya contiene `"fuego"` al momento en que BindingSystem ejecuta el paso de Registro (§3.1.E), **WHEN** el guard evalúa `world_state.available_demons.has("fuego")`, **THEN** el guard retorna `true`, el append se omite, `available_demons` no contiene duplicados, y `demon_bound` **no** se emite.
**Tipo**: Unit | **Bloquea**: Sí

**AC-VD-038** — Validación de Fantasy en Playtest: Vocabulario de Carga, No de Adquisición
**GIVEN** sesión de playtest donde testers experimentan al menos un binding estándar completo, **WHEN** se pregunta abiertamente "¿qué acaba de pasarle a Edrick?", **THEN** ≥60% de las respuestas usan vocabulario de carga o imposición ("me lo impuso", "fue forzado", "no quería", "algo oscuro le entró") en lugar de vocabulario de adquisición ("absorbí", "conseguí", "desbloqueé", "me lo gané"). Si el ratio es inferior al 60%, la secuencia visual de §8.1 y/o el lenguaje corporal de Edrick deben revisarse antes de Alpha.
**Tipo**: Playtest | **Bloquea**: No (Advisory)

---

**Resumen de criterios**: 37 totales — 33 bloqueantes, 4 advisory (AC-VD-011b, AC-VD-028, AC-VD-032, AC-VD-038).
**Gate mínimo para hand-off a QA**: AC-VD-033 (smoke check punta a punta).

---

## 10. Preguntas Abiertas

**P-VD-02 — RESUELTO**: Contrato de datos hacia escenas custom: `{ demon_id: String, edrick_position: Vector2, portador_position: Vector2 }`. Pasado vía `set()` inmediatamente tras `instantiate()`. Ver §3.1.D y AC-VD-034.

**P-VD-03 — Distorsión de Aura Post-Binding y Handoff a GDD #14**: La distorsión de aura en §3.1.C/§8.1 persiste tras la secuencia como primera manifestación visual permanente. Frontera definida: GDD #13 owns la distorsión durante la secuencia de binding; GDD #14 (Transformación Visual de Edrick) toma ownership al recibir `demon_bound` y reemplaza/amplía la distorsión inicial con su representación de largo plazo. La interfaz exacta (señal, callback, o lectura directa de GDD #3) a definir en GDD #14.

**P-VD-04 — GDD #5 (Sistema de Audio) requiere actualización**: GDD #5 no tiene actualmente el estado BINDING, ni los handlers de `binding_started`, `binding_custom_timeout`, ni la distinción gameplay-pause/player-pause. Debe ser actualizado antes de implementar la secuencia de audio. Requisitos detallados en §6.1 (fila de Audio) y §8.2.

**P-VD-05 — Portadores Únicos en MVP**: El sistema asume portadores únicos (un demonio por portador). ¿Puede en releases futuros (post-Alpha) existir un portador no-único? Si la respuesta es "nunca", sellar esta restricción en el schema de `demons.json`. Si puede cambiar, el schema debe contemplarlo para evitar deuda técnica.
