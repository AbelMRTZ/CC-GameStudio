# GDD: Exploración del Mundo

> **Estado**: En Diseño
> **Autor**: Abel + agentes
> **Última Actualización**: 2026-05-26
> **Sistema**: Exploración del Mundo
> **Milestone**: MVP — Gameplay Layer
> **Implementa Pilar**: Pilar 3 — Mundo Hermoso y Vivo
> **Depende de**: Movimiento y Físicas 2D (#1), Estado del Mundo (#4)
> **Dependen de este sistema**: Cámara (#9), Restricción por Demonio (#23), Mapa (#24)

---

## Overview

El sistema de Exploración del Mundo define cómo Edrick Velmar navega los 9 Reinos de Dravaryn: la estructura de zonas que componen cada reino, las transiciones entre ellas, los puntos de interés que pueblan el mundo (NPCs, encuentros con demonios, secretos, pistas de lore), y la detección de eventos ambientales que activan storytelling. Técnicamente, gestiona la carga de escenas de zona mediante instanciación en Godot, detecta áreas relevantes mediante `Area2D` triggers, y reporta al Estado del Mundo (#4) qué zonas ha visitado el jugador y qué eventos ambientales se han activado. Para el jugador, la exploración **es** el argumento del juego: cada desvío del camino principal recompensa la curiosidad con storytelling ambiental, demonios ocultos, o lore narrativo. El mundo de Dravaryn nunca se siente vacío — cada zona tiene algo que descubrir, sentir, o comprender. La Exploración está acotada narrativamente (los 9 reinos siguen un orden), pero dentro de cada reino el movimiento es libre; el jugador no es empujado — es atraído.

> **Nota técnica provisional**: el sistema referencia `TileMapLayer` (API de Godot 4.4+, reemplazó `TileMap`). Confirmar antes de arquitectura.

---

## Player Fantasy

Cuando Edrick explora un reino, el jugador debería sentir dos emociones entrelazadas: **asombro ante lo desconocido** y **belleza melancólica en lo que encuentra**.

El asombro viene de la incertidumbre: el mundo no da marcadores, ni flechas, ni "sigue este camino." El jugador ve una estructura extraña al fondo del escenario, un pasillo sin iluminar hacia la derecha, una plataforma que lleva a ningún lugar obvio — y la única motivación es *¿qué hay ahí?* Cada reino tiene secretos que sólo existen para quien se desvía del camino crítico.

La belleza viene de lo que encuentra: Dravaryn es un mundo oscuro pero **habitable**. La luz filtra entre ruinas. Los NPCs siguen sus vidas aunque el mundo se esté cayendo. Los demonios que merodean tienen una extraña dignidad. El jugador debería querer *quedarse* en una zona un momento más — no por mecánica, sino por atmósfera. Como Hollow Knight, donde entrar a una nueva zona y escuchar el tema musical cambiar basta para detener al jugador en seco.

Lo que el jugador **no** debe sentir: urgencia artificial, sensación de estar perdido sin salida, ni mundo vacío. La exploración se siente **libre pero con propósito** — cada área tiene algo que vale la pena encontrar, aunque no sea obligatorio encontrarlo.

**Momento ancla**: El jugador dobla una esquina y ve, en capas de parallax detrás suyo, los restos de algo enorme. No sabe qué fue. No hay texto que lo explique. La música se vuelve más suave. El jugador para de moverse y solo mira. *Eso* es Dravaryn.

---

## Detailed Design

### Core Rules

**R1 — Estructura de Zonas**
El mundo se divide en **zonas** (habitaciones discretas). Cada zona es una escena Godot independiente (`zona_ID.tscn`). Las zonas se conectan mediante **puntos de transición** (bordes de pantalla o puertas). No existe un tilemap global — el mundo se compone de escenas cargadas dinámicamente.

**R2 — Transición entre Zonas**
Cuando Edrick alcanza un punto de transición, el sistema:
1. Bloquea el input del jugador
2. Ejecuta un fade-out (0.3s)
3. Descarga la escena actual
4. Carga la escena de la zona destino
5. Posiciona a Edrick en el punto de entrada correspondiente al origen
6. Ejecuta un fade-in (0.3s)
7. Restaura el input

**R3 — Registro de Zonas Visitadas**
Cada zona tiene un `zona_id: String` único. Al entrar por primera vez, Exploración emite `zona_visitada(zona_id)` para que Estado del Mundo (#4) lo registre. Las visitas repetidas no emiten el evento.

**R4 — Tipos de Puntos de Interés (POIs)**
Cada POI es un nodo `Area2D` con un campo `tipo: Enum`. Tipos disponibles:

| Tipo | Trigger | Resultado |
|------|---------|-----------|
| `LORE_INSCRIPTION` | Requiere [E] dentro del Area2D | Muestra texto diegético legible en UI. Jugador presiona [E] de nuevo para cerrar. No interrumpe movimiento fuera de la lectura. |
| `LORE_WHISPER` | Automático al entrar al Area2D | Bloquea input; activa susurro del demonio (voz inhumana + efecto visual de corrupción). Jugador elige "Ignorar voces" (resiste) o "Escuchar" (cede a la corrupción). |
| `NPC` | Requiere [E] dentro del Area2D | Bloquea input; activa Sistema de NPC y Diálogo (#15) |
| `DEMON` | Requiere [E] dentro del Area2D | Bloquea input; activa secuencia narrativa de encuentro de demonio |
| `SECRET` | Automático al entrar al Area2D (área oculta) | Desbloquea acceso a nueva zona; emite `secreto_descubierto(secreto_id)` |

**R5 — POIs de Tipo LORE_INSCRIPTION**
Al acercarse a un Area2D de tipo `LORE_INSCRIPTION`, un prompt visual aparece (ej: "Inspeccionar [E]"). El jugador puede presionar [E] para ver el texto de la inscripción, que aparece como prosa legible en la UI (sin HUD notification, sin overlay de esquina). El jugador permanece en `TRAVERSAL` y puede moverse mientras lee. Al presionar [E] de nuevo, el texto desaparece. Si el jugador se aleja del Area2D, el prompt desaparece inmediatamente y el texto (si estaba abierto) se cierra sin fade.

**R5B — POIs de Tipo LORE_WHISPER**
Al entrar a un Area2D de tipo `LORE_WHISPER`, la transición es inmediata:
1. Input se bloquea (estado `INTERACTING`).
2. Efecto visual: pantalla oscurece ligeramente, aparece efecto de nubladez/distorsión mental.
3. Voz inhumana y profundamente corrupta (audio diegético) comienza a hablar (duración variable: 2-5 segundos típicamente).
4. Subtítulo aparece en pantalla (esquina inferior/central, legible).
5. **El jugador puede interrumpir el susurro EN CUALQUIER MOMENTO presionando "Ignorar las voces" o presionando [E].** No hay timeout automático. El demonio sigue hablando hasta que el jugador activamente lo detiene.
6. **El delta de corrupción_floor aplicado es proporcional al tiempo de audio escuchado antes de interrumpir:**
   - Si interrumpe a 25% del audio: corrupción_floor aumenta ~25% del delta máximo
   - Si interrumpe a 50% del audio: corrupción_floor aumenta ~50% del delta máximo
   - Si deja terminar el audio completo (100%): corrupción_floor aumenta el delta máximo
   - La fórmula exacta del delta es definida en GDD #4 (Estado del Mundo) y GDD #22 (Seguimiento Moral)

7. Efecto visual se desvanece en 0.5s una vez se produce la interrupción, input se restaura.
8. **El susurro ocurre UNA SOLA VEZ por trigger.** Una vez que el jugador lo ha escuchado (parcial o completamente) y lo ha interrumpido, ese trigger de `LORE_WHISPER` se desactiva permanentemente. Futuras visitas a la zona no reproducirán el susurro nuevamente.

**R6 — POIs de Tipo NPC/DEMON**
Al activarse: el personaje queda en estado `INTERACTING` (sin input de movimiento), el sistema delega el control al sistema responsable (Diálogo #15 / secuencia de demonio), y Exploración espera la señal `interaccion_completada()` para restaurar el input.

**R7 — Bounds de Zona**
Cada zona define un `Rect2` de límites (`zone_bounds`). Este valor se expone como propiedad pública para que la Cámara (#9) lo use. La cámara no tiene acceso al resto de la escena de zona — solo a `zone_bounds` y a la posición de Edrick.

---

### States and Transitions

| Estado | Descripción | Transiciones válidas |
|--------|-------------|---------------------|
| `TRAVERSAL` | Estado base — el jugador se mueve libremente | → `TRANSITION` al tocar punto de transición |
| `TRANSITION` | Cargando nueva zona — input bloqueado | → `TRAVERSAL` cuando la zona nueva está activa |
| `INTERACTING` | POI activo — input bloqueado | → `TRAVERSAL` cuando `interaccion_completada()` se emite |
| `ZONE_LOCKED` | Zona requiere demonio que el jugador no tiene | → `TRAVERSAL` (no entra); GDD #23 gestiona la lógica de bloqueo |

*Nota: `ZONE_LOCKED` es una interfaz de reserva para GDD #23 (Restricción por Demonio). Exploración no implementa la lógica de qué demonio se requiere — solo define el estado y el punto de entrada de zona bloqueada.*

---

### Interactions with Other Systems

| Sistema | Dirección | Qué se intercambia |
|---------|-----------|-------------------|
| Movimiento (#1) | Recibe | Posición de Edrick (para detectar colisiones con puntos de transición y Area2Ds de POIs) |
| Estado del Mundo (#4) | Emite hacia | `zona_visitada(zona_id: String)`, `poi_activado(poi_id: String, tipo: String)`, `secreto_descubierto(secreto_id: String)`, `susurro_interrumpido(susurro_id: String, pct_listened: float)` — donde pct_listened ∈ [0.0, 1.0] indica qué porcentaje del audio fue escuchado antes de interrumpir |
| Sistema de Audio (#5) | Emite hacia | `zona_entered(zona_id: String)` — Audio usa el ID para cargar el tema musical del reino. Emite también `susurro_audio_started(susurro_id)` cuando inicia un susurro (para efectos de sonido). |
| NPC y Diálogo (#15) | Delega a | POIs de tipo NPC activan el sistema de diálogo; Exploración espera `interaccion_completada()` |
| Cámara (#9) | Expone para | `zone_bounds: Rect2` — límites de la zona actual para que la cámara no salga del área |
| Restricción por Demonio (#23) | Interfaz provisional | Exploración expone `bool is_zone_locked(zona_id: String)` para que #23 lo consulte y bloquee transiciones si es necesario |

---

## Formulas

### Constantes de Timing

| Constante | Símbolo | Valor | Unidad | Usado por |
|-----------|---------|-------|--------|-----------|
| Duración de fade | `FADE_DURATION` | 0.3 | segundos | Transición de zona (fade-out y fade-in usan el mismo valor) |
| Duración máxima de susurro | `WHISPER_MAX_DURATION` | 5.0 | segundos | Duración normal del susurro si el jugador no lo interrumpe |
| Duración del efecto visual de susurro | `WHISPER_EFFECT_DURATION` | 0.5 | segundos | Fade-in y fade-out del efecto de nubladez mental (se aplica al inicio y final del susurro) |

> **Nota de tuning**: Los susurros son momentos narrativos críticos donde la corrupción es una *elección del jugador*, no un castigo automático. La duración debe permitir que el texto se lea completamente (típicamente 2-4 palabras/segundo en español) y dar tiempo suficiente para que el jugador reflexione antes de interrumpir. El jugador que interrumpe temprano recibe menos corrupción (recompensa por resistencia); el que escucha más tiempo es más corrupto (riesgo narrativo asumido conscientemente).

### Fórmula de Corrupción por Whisper

El delta de `corrupcion_floor` cuando un jugador interrumpe un `LORE_WHISPER` es **proporcional al tiempo de audio escuchado:**

```
corruption_floor_delta = whisper_base_delta × (time_listened / WHISPER_MAX_DURATION)
```

Donde:
- `whisper_base_delta` = delta máximo si el jugador escucha hasta el final (definido en GDD #4 / GDD #22)
- `time_listened` = segundos reales de audio que el jugador escuchó antes de interrumpir
- `WHISPER_MAX_DURATION` = duración máxima del audio (5.0 segundos)

**Ejemplo:** Si `whisper_base_delta = 0.02` y el jugador interrumpe a los 2.5 segundos (50% de 5.0s):
```
corruption_floor_delta = 0.02 × (2.5 / 5.0) = 0.02 × 0.5 = 0.01
```

**Propiedad:** El jugador que interrumpe inmediatamente (t ≈ 0) recibe corrupción ≈ 0. El jugador que deja terminar (t = 5.0) recibe el delta máximo. Este diseño recompensa la resistencia y hace que la corrupción sea una *elección consciente* del jugador de escuchar más tiempo.

---

### Restricciones de Tamaño de Zona

Las zonas deben respetar los siguientes límites derivados de la resolución interna del juego (320×180 px):

```
zone_width_px  ∈ [W_min, W_max]
zone_height_px ∈ [H_min, H_max]
```

| Símbolo | Tipo | Rango | Descripción |
|---------|------|-------|-------------|
| `W_min` | int | 320 px | Zona no puede ser más angosta que una pantalla; un valor menor hace que `zone_bounds` sea inválido para la Cámara (#9) |
| `W_max` | int | 1920 px | Máximo recomendado para MVP (6 pantallas de ancho); zonas más anchas aumentan el costo de producción de assets |
| `H_min` | int | 180 px | Zona no puede ser más baja que una pantalla |
| `H_max` | int | 720 px | Máximo recomendado para MVP (4 pantallas de alto) |

**Output**: `zone_bounds: Rect2` con `size.x ∈ [320, 1920]` y `size.y ∈ [180, 720]`.

**Ejemplo**: Una zona de entrada al reino con 640×180 px — dos pantallas de ancho, una de alto. Edrick puede scrollear horizontalmente pero no hay scroll vertical.

---

### Lo que NO tiene fórmula

- **Densidad de POIs por zona**: decisión de level design, no fórmula del sistema. Ver Tuning Knobs.
- **Distribución de secretos**: los secretos se colocan a mano por decisión narrativa. Sin generación procedural.
- **Tiempo total de transición**: `FADE_DURATION × 2 + tiempo_de_carga`. El tiempo de carga es no determinista (hardware-dependiente); se valida en Acceptance Criteria, no en Formulas.

---

## Edge Cases

- **Si el jugador está en `INTERACTING` y toca un punto de transición**: ignorar la transición. La transición solo procede desde el estado `TRAVERSAL`. Al completarse la interacción y restaurarse el input, el jugador puede moverse al punto de transición normalmente.

- **Si dos POIs se solapan y el jugador entra en ambos simultáneamente**: el sistema prioriza en este orden: `DEMON` > `LORE_WHISPER` > `NPC` > `SECRET` > `LORE_INSCRIPTION`. Solo el POI de mayor prioridad se activa. El otro permanece disponible para su siguiente visita.

- **Si el jugador entra a un Area2D `LORE_INSCRIPTION` y se aleja antes de presionar [E]**: el prompt desaparece. El texto nunca se muestra. El POI permanece activo para la siguiente vez que se acerque.

- **Si el jugador abre el texto de una inscripción (`LORE_INSCRIPTION`), se aleja, y vuelve a entrar**: el texto se cierra inmediatamente cuando sale del Area2D. Al volver a entrar, el prompt aparece de nuevo. El jugador debe presionar [E] de nuevo para ver el texto.

- **Si un susurro (`LORE_WHISPER`) está activo y el jugador intenta transicionar a otra zona**: bloquear la transición. El susurro debe ser interrumpido antes de que se permita cualquier movimiento a puntos de transición. El estado permanece `INTERACTING`.

- **Si el jugador interrumpe un susurro** (presiona "Ignorar" o [E] en cualquier momento durante el audio): el audio se corta inmediatamente, el efecto visual se desvanece en 0.5s, el input se restaura. Se aplica un delta de corrupción_floor proporcional al tiempo escuchado (ver Fórmula de Corrupción por Whisper en Formulas). El susurro se marca como "completado" y no se repetirá en futuras visitas a la zona. El trigger se desactiva permanentemente.

- **Si el jugador deja que un susurro termine completo** (sin interrumpir antes de que el audio termine): el audio termina naturalmente (duración máxima ~5 segundos). El efecto visual se desvanece en 0.5s, el input se restaura. Se aplica el delta máximo de corrupción_floor (100% del valor base). El susurro se marca como "completado" y no se repetirá en futuras visitas. El trigger se desactiva permanentemente.

- **Si una zona `SECRET` ya fue descubierta y el jugador la visita de nuevo**: no se emite `secreto_descubierto` de nuevo. El Estado del Mundo (#4) ya tiene el registro; Exploración comprueba el estado antes de emitir.

- **Cuando se dispara un SECRET por primera vez**: se emite `secreto_descubierto(secreto_id)` y se activa feedback inmediato (audio + visual simultáneamente) para comunicar al jugador que algo ocurrió. El feedback debe ser sutil (no celebratorio) pero perceptible: un sonido ambiental bajo (rumble, click mecánico, eco distante) combinado con un cambio visual en el environment (luz que cambia, partícula, color shift) en el área donde el SECRET se activó. Esto asegura que el jugador sabe que su acción tuvo consecuencia, aunque no haya un marcador de "descubrimiento" explícito.

- **Si `zone_bounds` tiene `size.x < 320` o `size.y < 180`** (zona inválida): loguear error en consola de Godot con el `zona_id` infractor. La cámara (#9) usa `max(zone_bounds.size, Vector2(320, 180))` como fallback para no crashear — la zona se puede jugar pero el bounds incorrecto debe corregirse en producción.

- **Si el jugador alcanza un punto de transición hacia una zona bloqueada (`ZONE_LOCKED`)**: el sistema no inicia la transición. GDD #23 es quien detecta el bloqueo y muestra el feedback correspondiente al jugador (Exploración no muestra mensajes propios en este caso). Si GDD #23 no está implementado, la zona simplemente no reacciona al input de transición.

- **Si `zona_visitada` se emite pero Estado del Mundo (#4) no está disponible**: loguear advertencia. No crashear. La zona se carga normalmente; la pérdida del registro es un bug de inicialización, no un crash de gameplay.

- **Si el jugador interrumpe un LORE_WHISPER a los 0 segundos** (presiona "Ignorar" literalmente cuando el audio comienza): se aplica corrupción_floor ≈ 0. El trigger se desactiva. Comportamiento: resistencia casi pura, el jugador rechaza la tentación inmediatamente.

- **Si un jugador deja abierto un LORE_WHISPER sin interactuar** (el jugador está AFK, se levanta del teclado): el audio continúa hasta terminar (no hay timeout). Al terminar, el effect se desvanece, input se restaura, corrupción_floor recibe el delta máximo. El trigger se desactiva.Este comportamiento penaliza AFK de forma narrativamente coherente (inacción = aceptación), pero sin "invisibilidad" — el jugador verá que pasó algo cuando regrese y notará la corrupción aumentada.

---

## Dependencies

**Dependencias upstream (este sistema requiere):**

| Sistema | Tipo | Qué aporta |
|---------|------|-----------|
| Movimiento y Físicas 2D (#1) | Dura | Posición y estado de Edrick (`CharacterBody2D`). Sin movimiento, Exploración no puede detectar triggers, puntos de transición ni POIs. |
| Estado del Mundo (#4) | Dura | Receptor de los registros de visita y eventos de POI. Sin Estado del Mundo, los eventos de exploración se pierden — el mundo no se vuelve reactivo. |

**Dependencias downstream (sistemas que dependen de este):**

| Sistema | Tipo | Qué necesitan de Exploración |
|---------|------|------------------------------|
| Cámara (#9) | Dura | `zone_bounds: Rect2` para definir los límites dentro de los que la cámara puede moverse. Sin esto, la cámara no sabe dónde están los bordes del mundo. |
| Restricción por Demonio (#23) | Suave | El estado `ZONE_LOCKED` y la señal/interfaz de "zona requiere demonio". GDD #23 intercepta las transiciones hacia zonas restringidas. |
| Mapa (#24) | Suave | `zona_id` de cada zona visitada + `zone_bounds` para renderizar el mapa. Sin Exploración no hay datos de mapa. |

**Dependencias implícitas (no en systems-index, pero reales):**

| Sistema | Por qué |
|---------|---------|
| NPC y Diálogo (#15) | Exploración delega el control a #15 cuando se activa un POI de tipo `NPC`. Exploración espera `interaccion_completada()` de regreso. |
| Sistema de Audio (#5) | Exploración emite `zona_entered(zona_id)`. Audio necesita este evento para cambiar la música al entrar a cada zona. |

> **Nota de bidireccionalidad**: GDD #1 (Movimiento) lista "Exploración del Mundo" como sistema dependiente. GDD #4 (Estado del Mundo) debe ser actualizado para mencionar que Exploración es emisor de `zona_visitada`, `poi_activado`, y `secreto_descubierto`. GDD #5 (Audio) debe actualizar sus dependencias para incluir `zona_entered`.

---

## Tuning Knobs

| Knob | Valor Actual | Rango Seguro | Si demasiado alto | Si demasiado bajo |
|------|-------------|-------------|-------------------|-------------------|
| `FADE_DURATION` | 0.3 s | 0.1 – 0.5 s | Transiciones lentas, pacing muerto | Pop visual abrupto sin sensación de traversal |
| `WHISPER_MAX_DURATION` | 5.0 s | 4.0 – 6.0 s | El susurro se siente largo, puede abrumar al jugador | El mensaje incompleto, narrativamente débil |
| `WHISPER_EFFECT_DURATION` | 0.5 s | 0.2 – 1.0 s | Efecto demasiado prolongado puede desorientar | Efecto muy sutil, el jugador no nota que está siendo "invadido" |
| `W_max` (ancho máx. de zona) | 1920 px | 320 – 3200 px | Zonas enormes y vacías; costo alto de assets | Solo una pantalla — sin exploración horizontal |
| `H_max` (alto máx. de zona) | 720 px | 180 – 1080 px | Exploración vertical excesiva, difícil de producir para solitario | Solo una pantalla — sin exploración vertical |
| **Densidad de POIs por zona** | 3 (objetivo) | 1 – 6 | Mundo ruidoso; el jugador ignora POIs por fatiga | Mundo vacío; exploración no recompensa la curiosidad (viola Pilar 3) |
| **Densidad de LORE_INSCRIPTION por zona** | 1–2 (objetivo) | 0 – 3 | Demasiado texto; la zona se siente como museo | Sin contexto narrativo; el mundo se siente vacío |
| **Densidad de LORE_WHISPER por zona** | 0–1 (objetivo) | 0 – 2 | El demonio nunca calla; los susurros pierden impacto como decisiones morales | Sin presencia del demonio; la transformación es invisible |
| **Densidad de zonas SECRET** | 1 por reino (MVP) | 0 – 3 por reino | Los secretos dejan de sentirse especiales | Sin recompensa para exploración off-path |

> **Interacciones entre knobs**: Los susurros son momentos narrativos de alto impacto donde la corrupción es una *elección del jugador*. La duración (`WHISPER_MAX_DURATION`) debe coordinarse con la música de la zona (Sistema de Audio #5) y debe ser lo suficientemente larga para que el jugador reflexione y haga una elección consciente — o interrumpa temprano para resistir. Si un susurro ocurre mientras suena un tema épico, considerar un fade de la música durante el susurro para aumentar el impacto emocional. La densidad de whispers por zona debe ser baja (0-1) para evitar que la corrupción se sienta obligatoria o automática. Cada susurro debe sentirse como una tentación deliberada del demonio, no un tick del sistema.

---

## Visual/Audio Requirements

**Referencia principal:** Art Bible §2.1 Exploration — define lighting character, atmósfera, y emotional target para todas las zonas de exploración.

### Visual — Zonas de Exploración

- **Paleta**: Cada zona aplica el color acento de su reino (Art Bible §1.1). Reino 1 = ámbar/oro apagado. La paleta base es siempre grises/marrones; el acento al 40–60% de saturación máxima. Sin bloom.
- **Iluminación**: Low ambient, side-lit. Fuentes de luz diegéticas (antorchas, ventanas). Sombras largas y direccionales. Ángulo de sol de tarde — comunica "el tiempo se acaba" sin narración.
- **Detalle obligatorio**: El layer de fondo de cada zona debe contener exactamente una ventana o apertura con luz cálida visible al fondo, inalcanzable. Cuesta un tile; gana todo el registro emocional.
- **Movimiento ambiental**: Antorchas parpadean en ciclos de 3–5 segundos. Debris/hojas en parallax nunca llaman la atención — pulso lento y constante.

### Visual — Transiciones de Zona

- **Fade-out/Fade-in**: Negro puro durante la carga. Duración 0.3s cada uno (ya definido en Formulas). El negro debe ser uniforme — sin bordes, sin transparencias.
- **Sin loading screen UI**: La transición es solo el fade. Sin iconos de carga, sin textos — puramente cinematográfico.

### Visual — POIs

| Tipo POI | Feedback visual | Notas |
|----------|----------------|-------|
| `LORE` | Overlay de texto en esquina inferior; tipografía consistente con estilo UI del juego | Sin frame ni caja decorativa en MVP; solo texto sobre sombra ligera |
| `NPC` | Prompt [E] minimalista sobre el NPC cuando Edrick está dentro del área | Prompt desaparece al alejarse |
| `DEMON` | Prompt [E] con efecto de color antinatural (violación cromática — Art Bible §1.3) | El demonio debe romper una regla visual del entorno |
| `SECRET` | Sin feedback visual en el momento del descubrimiento en MVP — la zona que se abre ES el feedback | Evita feedback que "celebre" demasiado; el descubrimiento silencioso es más atmosférico |

### Audio — Eventos de Zona

- **`zona_entered(zona_id)`**: Sistema de Audio (#5) usa el ID para hacer crossfade al tema musical del reino. El crossfade debe ser suave — no un corte duro.
- **Silencio intencional**: Algunas zonas pueden tener solo ambient sound sin música. Esto es una decisión narrativa, no un bug.

> 📌 **Asset Spec** — Los requisitos visuales de exploración (paleta por reino, ventana de fondo, iluminación) están definidos. Cuando el Art Bible esté aprobado, ejecuta `/asset-spec system:exploracion-del-mundo` para producir specs de assets por zona.

---

## UI Requirements

El sistema de Exploración contribuye cuatro elementos de UI en pantalla:

**1. Prompt de Inscripción (LORE_INSCRIPTION)**
- Visible solo cuando Edrick está dentro del `Area2D` de un POI tipo `LORE_INSCRIPTION`
- Posición: sobre el nodo (objeto de inscripción), centrado horizontalmente, ~16 px por encima
- Estilo: minimalista — "Inspeccionar [E]" (o variante equivalente)
- El objeto en el mundo debe estar visualmente destacado (highlight, glow leve, o color diferente) para indicar interactividad
- Desaparece instantáneamente al salir del área (sin fade)

**2. Texto de Inscripción (LORE_INSCRIPTION — lectura)**
- Posición: centrada en pantalla (vertical y horizontal)
- Tipografía: fuente del juego, tamaño legible a 320 px de ancho nativo (especificación numérica en `/ux-design`)
- Fondo: sombra sutil (no caja ni frame)
- Duración: indefinida mientras el jugador no presione [E] de nuevo
- Sin fade — desaparece instantáneamente al presionar [E]
- El jugador NO está bloqueado; puede presionar direccionales para moverse mientras lee

**3. Efecto Visual de Susurro (LORE_WHISPER)**
- Pantalla oscurece ligeramente (opacidad reducida ~20%)
- Efecto de distorsión/nubladez mental (shader distortion u overlay con ruido)
- Fade-in: 0.5s al iniciar el susurro
- Fade-out: 0.5s al completar/ignorar el susurro

**4. Subtítulo y Opciones de Susurro (LORE_WHISPER — contenido)**
- Subtítulo: esquina inferior/central, legible
- Dos botones de opción visibles:
  - "Ignorar las voces" (o equivalente — rechazar)
  - "Escuchar" (o equivalente — aceptar) / O simplemente esperar a que el susurro termine
- Sin interrupción visual del mundo — el jugador sigue viendo la zona de fondo (oscurecida)

**5. Prompt de Interacción [E] (NPC/DEMON)**
- Visible solo cuando Edrick está dentro del `Area2D` de un POI tipo `NPC` o `DEMON`
- Posición: sobre el nodo del POI, centrado horizontalmente, clampeado al borde superior con 4 px de margen seguro (para evitar off-screen en plataformas altas)
- Estilo: minimalista — solo el icono/texto de tecla. Sin descripción de acción en MVP
- Para POIs `DEMON`: el prompt usa color antinatural (Art Bible §1.3 — violación cromática)
- Desaparece instantáneamente al salir del área (sin fade)

**6. Minimap Diegético (MVP Concept)**

El jugador puede acceder a un minimap presionando una tecla designada (propuesta: [M]). El minimap es diegético: representa un pergamino/mapa que Edrick lleva en su viaje. Comportamiento:

- **Trigger**: Presionar [M] abre una vista diegética del mapa en overlay (similar a abrir un inventario o menú)
- **Contenido MVP**: Muestra las zonas visitadas en el reino actual con su disposición relativa. No debe ser un mapa perfectamente legible — las zonas pueden estar dibujadas de forma aproximada, sin escala exacta
- **Intención narrativa**: El mapa no es "inteligencia de IA"; es un objeto físico que Edrick posee. No es intuitivo (confuso a propósito en MVP). El jugador debe explorar para entender el mapa
- **Futuro (Vertical Slice+)**: Un demonio especial ("Visión Mejorada" o equivalente) otorga la habilidad de ver rutas exactas y resaltar objetivos en el minimap, transformándolo de herramienta confusa a herramienta de navegación clara
- **No implementar en MVP**: El demonio de visión mejorada está conceptualmente diseñado pero no implementado hasta Vertical Slice

Nota: El minimap es concepto futuro. La navegación primaria en MVP sigue siendo implícita (iluminación, composición focal). El minimap es fallback solo si el jugador está completamente perdido.

> 📌 **UX Flag — Exploración del Mundo**: Este sistema tiene UI requirements complejos. En Pre-Producción, ejecuta `/ux-design` para crear UX spec detallada de: (a) highlight visual de inscripciones, (b) tamaño y posicionamiento de texto de inscripción, (c) efecto visual de susurro, (d) layout y opciones de "Ignorar" button durante susurro, (e) prompts de interacción con clamping, (f) minimap visual (pergamino diegético, legibilidad aproximada, interfaz de acceso). Las stories de UI deben citar `design/ux/hud.md` en vez del GDD directamente.

---

## Acceptance Criteria

### Zonas y Carga

**AC-EXPLO-01 (R1)** GIVEN que el mundo está cargado, WHEN se inspecciona cualquier zona en el filesystem, THEN cada zona corresponde a un `.tscn` distinto con `zona_id` único (propiedad @export `zona_id: String` en el nodo raíz); no existe tilemap global que contenga más de una zona.

**AC-EXPLO-02 (R2)** GIVEN que Edrick está en `TRAVERSAL`, WHEN alcanza un punto de transición válido, THEN los eventos ocurren en orden estricto: (1) input_blocked=true, (2) fade_out_start, (3) scene_unload, (4) scene_load, (5) edrick_position_set, (6) fade_in_start, (7) input_blocked=false. Verificable mediante test automatizado con event logging.

**AC-EXPLO-03 (FADE_DURATION — out)** GIVEN que Edrick inicia una transición, WHEN el fade-out comienza, THEN la pantalla pasa a negro en exactamente 0.3s (±0.05s), medido desde el Tween.duration en código, no visualmente.

**AC-EXPLO-04 (FADE_DURATION — in)** GIVEN que la escena destino ha cargado, WHEN el fade-in comienza, THEN la pantalla pasa a visible en exactamente 0.3s (±0.05s), medido desde el Tween.duration en código.

### Registro de Zonas

**AC-EXPLO-05 (R3)** GIVEN que Edrick nunca ha visitado `"zona_005"`, WHEN entra por primera vez, THEN se emite exactamente una señal `zona_visitada("zona_005")` hacia Estado del Mundo.

**AC-EXPLO-06 (R3)** GIVEN que Edrick ya visitó `"zona_005"` y Estado del Mundo lo registró, WHEN entra de nuevo, THEN NO se emite `zona_visitada` de nuevo. El registro permanece.

### Inscripciones Diegéticas (LORE_INSCRIPTION)

**AC-EXPLO-07 (R5 — Prompt de inscripción)** GIVEN que Edrick se acerca a un POI `LORE_INSCRIPTION`, WHEN entra al Area2D, THEN el prompt "Inspeccionar [E]" aparece sobre el objeto y el objeto mismo se destaca visualmente (highlight/glow).

**AC-EXPLO-08 (R5 — Lectura de inscripción)** GIVEN que el prompt está visible, WHEN Edrick presiona [E], THEN el texto de la inscripción aparece en UI (centrado) y el estado permanece `TRAVERSAL` (sin bloqueo de input). Edrick puede moverse mientras lee.

**AC-EXPLO-09 (R5 — Cierre de inscripción)** GIVEN que el texto está visible, WHEN Edrick presiona [E] de nuevo, THEN el texto desaparece instantáneamente.

**AC-EXPLO-10 (R5 — Salir del área)** GIVEN que el prompt está visible O el texto está abierto, WHEN Edrick sale del Area2D, THEN el prompt desaparece inmediatamente y el texto (si estaba abierto) se cierra sin animación.

### Susurros del Demonio (LORE_WHISPER)

**AC-EXPLO-11 (R5B — Activación de susurro)** GIVEN que Edrick entra a un POI `LORE_WHISPER`, WHEN el cuerpo toca el Area2D, THEN inmediatamente: (1) input se bloquea (INTERACTING), (2) efecto visual de nubladez aparece con fade-in 0.5s, (3) audio de voz inhumana y corrupta comienza a reproducir, (4) subtítulo aparece en pantalla (esquina inferior/central, legible), (5) botón "Ignorar las voces" aparece y es interactuable. PASS: verificable mediante spy/mock en test de integración.

**AC-EXPLO-12 (R5B — Duración de susurro)** GIVEN que el susurro está activo, WHEN el demonio comienza a hablar, THEN el audio dura entre 2.0 segundos y `WHISPER_MAX_DURATION` (5.0s) dependiendo del contenido grabado. PASS: verificable midiendo `AudioStreamPlayer.stream.get_length()` o mediante timestamp de inicio/fin del audio durante playback.

**AC-EXPLO-13 (R5B — Interrupción en cualquier momento)** GIVEN que el susurro está activo en cualquier momento durante el audio, WHEN el jugador presiona "Ignorar las voces" o [E], THEN: (1) audio se detiene inmediatamente, (2) se calcula el porcentaje de duración escuchada: `pct_listened = time_listened / WHISPER_MAX_DURATION`, (3) se emite `susurro_interrumpido(susurro_id, pct_listened)` hacia Estado del Mundo, (4) efecto visual se desvanece en 0.5s, (5) input se restaura. PASS: el evento contiene ambos parámetros (susurro_id y porcentaje escuchado).

**AC-EXPLO-14 (R5B — Corrupción proporcional)** GIVEN que `susurro_interrumpido(susurro_id, pct_listened)` fue emitido con pct_listened = 0.5 (50%), WHEN Estado del Mundo procesa el evento, THEN corrupción_floor aumenta en: `corruption_delta = whisper_base_delta × 0.5`. PASS: corrupción_floor registrada en Estado del Mundo es exactamente 50% del delta máximo (definido en GDD #4/GDD #22).

**AC-EXPLO-15 (R5B — Dejar que termine completo)** GIVEN que el susurro está activo, WHEN el jugador no lo interrumpe y el audio termina naturalmente, THEN: (1) al final del audio, se emite `susurro_interrumpido(susurro_id, pct_listened = 1.0)`, (2) efecto visual se desvanece en 0.5s, (3) input se restaura, (4) corrupción_floor aumenta por el delta máximo (100%). PASS: pct_listened = 1.0 exactamente.

**AC-EXPLO-16 (R5B — Trigger se desactiva después de interrumpir)** GIVEN que un susurro fue interrumpido (en cualquier punto: 0%, 25%, 100%), WHEN Edrick re-entra el mismo `LORE_WHISPER` Area2D en futuras visitas, THEN el susurro NO se activa nuevamente. El trigger permanece silencioso. PASS: `Area2D.body_entered` signal no dispara comportamiento de susurro (debe chequearse si susurro ya fue "completado" en Estado del Mundo).

### NPCs y Demonios

**AC-EXPLO-17 (R4-NPC sin activación)** GIVEN que Edrick está dentro del Area2D de un POI `NPC`, WHEN no presiona [E], THEN el sistema de Diálogo NO se activa; Edrick permanece en `TRAVERSAL`.

**AC-EXPLO-18 (R4-NPC con activación)** GIVEN que Edrick está dentro del Area2D de un POI `NPC`, WHEN presiona [E], THEN: (1) input se bloquea (INTERACTING), (2) se emite señal de activación a GDD #15, (3) Exploración espera `interaccion_completada()` para restaurar input.

**AC-EXPLO-19 (R4-DEMON)** GIVEN que Edrick está dentro del Area2D de un POI `DEMON`, WHEN presiona [E], THEN: (1) input se bloquea (INTERACTING), (2) se inicia la secuencia narrativa de encuentro de demonio (ver GDD `encuentro-demonio-primera-vez.md`), (3) Exploración espera `interaccion_completada()`.

### Secretos

**AC-EXPLO-20 (R4-SECRET)** GIVEN un POI `SECRET` no descubierto, WHEN Edrick entra al Area2D, THEN: (1) se emite `secreto_descubierto(secreto_id)` exactamente una vez, (2) se desbloquea acceso a la zona asociada (ej: punto de transición anteriormente inactivo), (3) se dispara feedback inmediato — audio ambiental bajo (rumble, sonido mecánico, eco distante) + cambio visual simultáneo (luz que cambia, partícula, color shift) en el área del trigger. PASS: ambos feedback (audio y visual) son detectables en un test de integración mediante spy de eventos de audio/partículas.

**AC-EXPLO-21 (R4-SECRET — sin repetición)** GIVEN que `secreto_descubierto(secreto_id)` ya fue emitido, WHEN Edrick vuelve al mismo POI `SECRET`, THEN NO se emite de nuevo y NO se disparan los feedback (audio y visual). El trigger permanece silencioso. Total emisiones = 1 por jugador.

### Prioridad de POIs

**AC-EXPLO-22 (Edge — Prioridad)** GIVEN que Edrick entra simultáneamente en dos o más POIs de tipos distintos solapados, THEN solo el POI de máxima prioridad se activa: `DEMON` > `LORE_WHISPER` > `NPC` > `SECRET` > `LORE_INSCRIPTION`. PASS: mediante test con dos Area2Ds superpuestos, verificar que solo una se activa.

### Estado INTERACTING

**AC-EXPLO-23 (Edge — Timeout INTERACTING)** GIVEN estado `INTERACTING` por susurro/diálogo/encuentro, WHEN pasan más de 10 segundos sin que se emita `interaccion_completada()`, THEN: (1) force-restaurar input, (2) cambiar estado a `TRAVERSAL`, (3) loguear error con `interaccion_id` en consola. PASS: el jugador puede moverse de nuevo; el sistema no queda bloqueado indefinidamente.

**AC-EXPLO-24 (Edge — INTERACTING bloquea transición)** GIVEN estado `INTERACTING`, WHEN Edrick toca un punto de transición, THEN NO se inicia ninguna transición. Input permanece bloqueado hasta que la interacción complete.

### Transiciones y Estados

**AC-EXPLO-25 (Edge — ZONE_LOCKED)** GIVEN que `is_zone_locked("zona_bloqueada")` retorna true (consultado a GDD #23), WHEN Edrick alcanza el punto de transición hacia esa zona, THEN NO ocurren los pasos de transición (sin fade, sin load, sin input block). El sistema permanece en `TRAVERSAL`. PASS: el jugador no ve ningún cambio visual.

**AC-EXPLO-26 (Edge — Estado del Mundo no disponible)** GIVEN que Estado del Mundo no está inicializado, WHEN Edrick entra a una zona, THEN: (1) se emite `zona_visitada` pero el receptor está null/no disponible, (2) se imprime WARN en consola con `zona_id`, (3) la zona carga con normalidad. Juego NO crashea. PASS: exploración funciona sin Estado del Mundo.

### Eventos de Integración

**AC-EXPLO-27 (Eventos hacia Estado del Mundo)** GIVEN que Edrick interactúa con POIs, WHEN se activan, THEN se emiten correctamente: `zona_visitada(zona_id)`, `poi_activado(poi_id, tipo)`, `secreto_descubierto(secreto_id)`, `susurro_interrumpido(susurro_id, pct_listened)`. PASS: verificable mediante spy/mock en tests de integración. Nota: `susurro_interrumpido` reemplaza los anteriores `susurro_resistido` y `susurro_aceptado` con un sistema gradual donde pct_listened es el porcentaje de duración escuchada (0.0 = interrumpido inmediatamente, 1.0 = escuchado completo).

**AC-EXPLO-28 (Evento zona_entered hacia Audio)** GIVEN que Edrick transiciona a una zona, WHEN el fade-in comienza (paso 6 de transición), THEN se emite `zona_entered(zona_id)` hacia Sistema de Audio (#5). PASS: audio puede iniciar crossfade en ese momento exacto.

**AC-EXPLO-29 (Zone bounds para Cámara)** GIVEN que una zona ha cargado, WHEN Cámara (#9) solicita `zone_bounds`, THEN se retorna `Rect2` con `size.x ∈ [320, 1920]` y `size.y ∈ [180, 720]`. Fallback si inválido: `Vector2(320, 180)`.

---

## Open Questions — RESUELTAS y PENDIENTES

### Resueltas en esta revisión:

2. ✅ **Duración y contenido de los susurros**: Resuelto. El sistema ahora usa interrupción gradual: el jugador puede interrumpir EN CUALQUIER MOMENTO durante los 2-5 segundos de audio. La duración máxima es flexible (depende del contenido grabado). El audio director debe coordinar duración con música; el narrative director debe coordinar contenido con progresión moral. No hay restricción de "must listen 2 seconds" — la elección es completamente del jugador desde el momento 0.

3. ✅ **Mecánica de corrupción y resistencia**: Resuelto. El sistema ahora es: corrupción proporcional al tiempo escuchado (0% interrupción inmediata = ~0 corrupción; 100% escuchar completo = delta máximo). Esto es narrativamente coherente: el jugador que resiste temprano recibe menos corrupción. El jugador que cede más tiempo recibe más corrupción. La fórmula está en Formulas → "Fórmula de Corrupción por Whisper". GDD #14 sigue siendo owner de cómo se refleja visualmente (animación, color, shader), pero ahora el input (% de corrupción por whisper) está bien definido.

5. ✅ **Tipología de zonas y wayfinding implícito**: Parcialmente resuelto. El GDD mantiene navegación implícita (iluminación, composición focal) como el camino principal. ADICIONALMENTE, se añade un minimap diegético (pergamino que Edrick lleva, accesible con tecla, no permanente en pantalla) como fallback de navegación. Este minimap es concepto MVP. En Vertical Slice+, se añadirá un demonio especial que otorga "visión mejorada" (resalta objetos importantes, camino a seguir) — esto es concepto futuro, documentado pero no MVP. **Owner**: level-designer (wayfinding), ux-design (minimap), demonio futuro en Vertical Slice.

6. ✅ **Feedback de `ZONE_LOCKED` sin GDD #23**: Resuelto como edge case. Si GDD #23 no está implementado, el sistema silenciosamente no permite la transición (jugador ve "nada" y puede intentar de nuevo). Este es behavior aceptable para MVP si no hay zonas bloqueadas en reino 1. Si hay zonas bloqueadas, GDD #23 debe implementar el feedback. **Owner**: GDD #23.

7. ✅ **SECRET sin feedback visual / descubribilidad**: Resuelto. Los SECRETs ahora disparan feedback audio+visual simultáneamente (sonido ambiental bajo + cambio visual en el environment). El level design debe usar estos como punto de partida y colocar pistas PREVIAS al secret (composición, iluminación focal) para indicar "hay algo aquí". **Owner**: level-designer (pistas previas), este GDD (feedback de descubrimiento).

### Pendientes (requerir input futuro):

1. **`TileMapLayer` API (Godot 4.4+)**: Esta versión reemplazó a `TileMap`. Antes de arquitectura, verificar en Godot 4.6 docs que la API soporte las colisiones y el parallax definidos en este GDD. **Owner**: `/setup-engine`. **Bloquea**: ningún sistema todavía (pre-código).

4. **Inscripciones diegéticas y longitud de texto**: El GDD especifica que `LORE_INSCRIPTION` son "inscripciones breves". Antes de escribir nivel 1, establecer: máximo 3 líneas, máximo 50-60 caracteres/línea en español. **Owner**: UX design (próximo `/ux-design`). **Bloquea**: narrative writing para inscripciones.

8. **Reacción de NPCs a corrupción de Edrick**: FUERA del MVP. Los NPCs que noten que Edrick está siendo invadido por el demonio es una feature de Vertical Slice. **Owner**: GDD #15 + #16. **Bloquea**: nada en MVP. **Diferido a**: Vertical Slice.

9. **Demonio futuro de "Visión Mejorada"**: Concepto para Vertical Slice+. Permite que Edrick vea resaltos de objetos importantes y camino a seguir en el mundo (visión estilo Hollow Knight's "focus" o Hades "highlight"). Este demonio mejora el navigation sin añadir minimap/marcadores. Debe ser diseñado en coordination con este GDD y con el future level-design. **Owner**: GDD futuro + demonio-bind system. **Bloquea**: nada en MVP. **Diferido a**: Vertical Slice.
