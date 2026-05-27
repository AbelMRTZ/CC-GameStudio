# GDD: Sistema de Cámara

> **Estado**: Diseñado
> **Autor**: Abel + agentes
> **Última Actualización**: 2026-05-27
> **Sistema**: Cámara
> **Milestone**: MVP — Core Layer
> **Implementa Pilar**: Pilar 3 — Mundo Hermoso y Vivo; Pilar 1 — Narrativa Cinematográfica
> **Depende de**: Movimiento y Físicas 2D (#1), Exploración del Mundo (#8)
> **Dependen de este sistema**: Cinemáticas (#17)

---

## Overview

El Sistema de Cámara gobierna la ventana a través de la cual el jugador experimenta Dravaryn: una `Camera2D` de Godot que sigue suavemente a Edrick Velmar durante la exploración y el combate (con submodos dinámicos), respeta los límites geográficos de cada zona, y cede el control al sistema de Cinemáticas (#17) cuando la narrativa lo exige. La cámara es **composición emergente del movimiento natural** — el look-ahead suave y el framing de zona comunican intención sin director visible. En momentos narrativos clave (cinemáticas, encuentros de jefe), **CameraAnchors** manuales intensifican la composición cinematográfica para apoyar el Pilar 1 (Narrativa Cinematográfica). 

**Implementación técnica: suavizado manual via lerp en `_process`, no suavizado nativo de Camera2D**, permitiendo transiciones dinámicas de `smoothing_speed` entre submodos (exploración/combate) sin saltos visuales. Los límites de zona se almacenan localmente; **`Camera2D.limit_*` se desactivan** (clamping ocurre en F5, no en el nodo, para evitar double-clamping con zoom variable). El sistema gestiona dos modos exclusivos — **FOLLOW** (gameplay, auto-follow) y **CINEMATIC** (controlado por #17) — con transiciones suaves entre ambos. Opera sobre la resolución interna de 320×180 px del proyecto, tomando la posición del `CharacterBody2D` de Edrick como objetivo.

---

## Player Fantasy

Cuando la cámara de Dravaryn hace su trabajo, el jugador no debería pensar en la cámara — debería detenerse en seco y pensar *"este sitio es hermoso"*, o *"no quiero avanzar todavía"*, o simplemente olvidar por un momento que está controlando algo.

La fantasía del Sistema de Cámara es **composición emergente sin director visible**: la cámara sigue a Edrick con inercia natural (look-ahead suave en la dirección de movimiento, subtiles límites de zona), permitiendo que el jugador explore libremente mientras la composición emerge del movimiento. El framing se siente orgánico, no impuesto. Esto sirve directamente al **Pilar 3** (*Mundo Hermoso y Vivo*) y al **Pilar 1** (*Narrativa Cinematográfica*): la exploración es fluida y hermosa por su naturalidad, no por dirección asfixiante.

En momentos narrativos **clave** — encuentros de jefe, cinemáticas de giro de historia, santuarios narrativos — **CameraAnchors** manuales intensifican la composición, posicionando el encuadre intencionalmente. Estos momentos contrastan con la exploración libre, haciendo que el cambio de dirección sea reconocible como narrativa. El jugador *siente* el cambio de agencia (de libre a dirigido) pero nunca lo cuestiona.

**Momento ancla** (redefinido): Edrick aproxima una catedral abandonada. La cámara lo sigue naturalmente con look-ahead: él está en el tercio izquierdo de pantalla, el rosetón roto visible al fondo derecho. Es composición emergente — hermosa porque el arte + el layout + el movimiento convergen. Edrick entra al interior. Un **CameraAnchor** se activa: la cámara hace un subtle push-in (zoom 1.05×) durante 2s, mostrando la grandiosidad del espacio. Edrick camina; el follow retoma. Sin transición brusca. El jugador siente "el juego me está mostrando algo importante" sin que se sienta artificial.

**Lo que el jugador NO debe sentir en exploración**: que la cámara lo está empujando. El seguimiento parece siempre natural. **Lo que SÍ debe sentir en momentos de anchor**: reconocer que el juego tomó intención narrativa; que ese momento importa. La belleza emerge de la convergencia — nunca de violencia directorial.

---

## Detailed Design

### Core Rules

**R1 — Dos Modos de Operación**

La cámara opera en dos modos exclusivos:
- **FOLLOW** (gameplay): sigue autónomamente a Edrick con suavizado y look-ahead.
- **CINEMATIC** (narrativo): el Sistema de Cinemáticas (#17) toma control completo. La cámara responde a comandos externos — posición destino, zoom, corte — en lugar de seguir a Edrick.

La transición entre modos siempre es suavizada. Nunca hay snap de posición al cambiar de modo.

**R2 — Follow Suave con Look-Ahead Horizontal**

En modo FOLLOW, la cámara no centra a Edrick directamente — calcula una **posición objetivo** (F1) desplazada en la dirección del movimiento. La posición actual de la cámara **interpola suavemente** desde su posición actual hacia ese objetivo cada frame, usando un **lerp manual** controlado por `smoothing_speed` (que varía dinámicamente entre submodos exploración/combate). El movimiento por frame se limita a `lerp_delta_clamp_max` para evitar teleports visuales durante grandes saltos de target.

**Implementación técnica requerida:**

```gdscript
# En _process(delta):
var target_clamped = apply_zone_limits(target_position)  # F5

# Compute lerp factor, clamped for frame-rate stability
var lerp_factor = smoothing_speed * delta
var delta_vec = target_clamped - camera.position
var delta_magnitude = delta_vec.length()

# Cap the per-frame movement to lerp_delta_clamp_max for safety during pivots
if delta_magnitude > lerp_delta_clamp_max:
  var clamped_delta = delta_vec.normalized() * lerp_delta_clamp_max
  camera.position += clamped_delta
else:
  # Normal lerp when movement is under the cap
  camera.position = camera.position.lerp(target_clamped, lerp_factor)

# Snap to integer pixels for clean pixel-art rendering
camera.position = camera.position.round()
```

**Detalles clave:**
- La posición objetivo se calcula con F1: `target.x = Edrick.x + (dir_input × look_ahead_current)`
- `smoothing_speed` es variable: 6.0 en exploración, 9.0 en combate, interpolado dinámicamente por F3
- `lerp_delta_clamp_max` = 12px: previene teleports durante pivotes rápidos (E1). Si `|target − current|` excede este máximo por frame, movimiento se limita a 12px.
- **`Camera2D.position_smoothing_enabled` debe estar DESACTIVADO** (suavizado es manual, no nativo)
- **`Camera2D.limit_left/right/top/bottom` deben estar DESACTIVADOS** (clamping ocurre en F5, no en Camera2D)
- **Drag margins (`drag_*_enabled`) deben estar DESACTIVADOS** — interfieren con el lerp manual
- **Pixel snap requerido** — `.round()` después del lerp evita shimmer en pixel-art 320×180
- Cuando input horizontal = 0, `look_ahead_current` decae gradualmente a 0 (F2), re-centrando automáticamente
- La cámara **nunca hace snap** porque lerp garantiza transición gradual, incluso durante cambios rápidos de dirección (E1)

**R3 — Submodo Exploración vs. Combate**

Dentro de FOLLOW, el look-ahead se ajusta por contexto. La cámara no detecta combate internamente — el Sistema de Combate (#6) emite señales que el controlador de cámara recibe:

| Contexto | `look_ahead_dist` | `smoothing_speed` | Comportamiento |
|----------|------------------|-------------------|----------------|
| Exploración | 50 px | 6.0 | Muestra más mundo en la dirección de avance |
| Combate | 20 px | 9.0 | Más centrado en Edrick, más ágil |

La transición entre submodos es gradual (lerp sobre `combat_transition_time` = 0.4s).

**R4 — Límites de Zona**

Cada zona define sus límites de cámara en los metadatos de la escena. Los límites se almacenan como variables locales (`zone_limit_left`, `zone_limit_right`, `zone_limit_top`, `zone_limit_bottom`) pero **NO** se asignan a `Camera2D.limit_*` (para evitar double-clamping con F5 cuando zoom es variable). En su lugar, F5 clampea la posición objetivo contra estos límites cada frame. La cámara nunca muestra píxeles fuera de los límites definidos. Si Edrick alcanza una esquina, la cámara se detiene en el límite aunque Edrick siga moviéndose. Los límites coinciden con los bordes del arte de fondo.

**R5 — Camera Anchors (Puntos de Composición)**

Para implementar el framing cinematográfico de la Player Fantasy, el sistema soporta nodos `CameraAnchor` colocados manualmente en escena. Cuando Edrick entra en el `Area2D` de un anchor:

1. La cámara añade un **offset de composición** suave a su posición objetivo (ej: +30 px a la izquierda para dar aire al fondo).
2. Opcionalmente aplica un **zoom suave** (`push-in`) si el anchor lo define.
3. Al salir del área, el offset y el zoom se revierten gradualmente.

Un `CameraAnchor` define: `offset_x`, `offset_y`, `zoom_target`, `transition_time`. Son opcionales — el 80% de las zonas no los necesitan; se reservan para momentos narrativos de alto impacto.

**R6 — Transición en Cambio de Zona**

Cuando Exploración (#8) inicia una transición:
1. `zona_transition_started` → la cámara congela posición (desactiva position smoothing).
2. Durante el fade-out (FADE_DURATION = 0.3s): la cámara no se mueve.
3. Nueva zona cargada → la cámara hace **snap inmediato** a la posición inicial de Edrick (invisible al jugador — ocurre durante el frame negro).
4. `zona_transition_ended` → reactiva position smoothing, reanuda follow normal.

---

### States and Transitions

| Estado | Descripción | Entradas desde | Salidas a |
|--------|-------------|----------------|-----------|
| **FOLLOW_EXPLORE** | Follow con look-ahead máximo | FOLLOW_COMBAT, TRANSITION | FOLLOW_COMBAT, CINEMATIC, TRANSITION |
| **FOLLOW_COMBAT** | Follow centrado, look-ahead reducido | FOLLOW_EXPLORE | FOLLOW_EXPLORE, CINEMATIC, TRANSITION |
| **CINEMATIC** | Control cedido a Sistema #17 | FOLLOW_EXPLORE, FOLLOW_COMBAT | FOLLOW_EXPLORE |
| **TRANSITION** | Cámara congelada durante cambio de zona | FOLLOW_EXPLORE, FOLLOW_COMBAT | FOLLOW_EXPLORE |

Restricciones: No se puede entrar en CINEMATIC desde TRANSITION. CINEMATIC siempre vuelve a FOLLOW_EXPLORE (nunca a FOLLOW_COMBAT).

---

### Interactions with Other Systems

**GDD #1 — Movimiento (upstream)**
- Lee `CharacterBody2D.position` y `.velocity.x` de Edrick cada frame. Solo lectura — la cámara no modifica el movimiento.

**GDD #8 — Exploración (upstream)**
- Recibe `zona_transition_started` → entra en TRANSITION.
- Recibe `zona_transition_ended` → sale de TRANSITION.
- Lee los límites de `Camera2D` de los metadatos de cada escena cargada.

**GDD #6 — Combate (upstream, señal)**
- Recibe `combat_started` (emitido por Sistema de Combate vía EventBus) → transición a FOLLOW_COMBAT. F3 interpola `smoothing_speed` desde 6.0 a 9.0 durante `combat_transition_time = 0.4s`.
- Recibe `combat_ended` (emitido por Sistema de Combate vía EventBus) → transición a FOLLOW_EXPLORE. F3 interpola `smoothing_speed` desde 9.0 a 6.0 durante 0.4s.
- **Mecanismo de entrega: EventBus Autoload** — Define dos señales (`Signal combat_started` y `Signal combat_ended`) que el Sistema de Combate emite, y el controlador de cámara se suscribe en `_ready()`. (Ver ADR: Arquitectura de Señales Globales.)

**GDD #17 — Cinemáticas (downstream)**
- Recibe `EventBus.cinematic_started(camera_data: Dictionary)` → entra en CINEMATIC.
  - `camera_data` fields: `target_position: Vector2`, `target_zoom: float`, `transition_time: float`, `easing_type: String` (e.g., "linear", "ease_out")
  - Modo CINEMATIC interpola `camera.position` y `camera.zoom` desde valores actuales hacia targets sobre `transition_time`.
  - No hay look-ahead, no hay limit clamping — la cinemática es absoluta.
- Recibe `EventBus.cinematic_ended()` → sale a FOLLOW_EXPLORE (nunca a FOLLOW_COMBAT).
  - Nota: Si el Sistema de Combate #6 está activo cuando la cinemática termina, debe re-emitir `EventBus.combat_started` para restaurar FOLLOW_COMBAT.
  - El timing de esa re-emisión determinará cuánto tiempo la cámara está en framing equivocado durante la transición post-cinematic.

---

## Formulas

### F1 — Posición Objetivo (Follow Target)

```gdscript
target.x = edrick.position.x + (dir_input * look_ahead_current)
target.y = edrick.position.y + vertical_offset
```

**Variables:**

| Variable | Tipo | Rango | Descripción |
|----------|------|-------|-------------|
| `target.x / target.y` | float | [−∞, +∞] | Posición objetivo de la cámara en espacio de mundo |
| `edrick.position.x / .y` | float | [−∞, +∞] | Posición actual del CharacterBody2D de Edrick |
| `dir_input` | float | {−1, 0, +1} | Input horizontal discreto: −1 (izquierda), +1 (derecha), 0 (ninguno). **Nota**: representa *intención*, no velocidad |
| `look_ahead_current` | float | [0, look_ahead_max] px | Magnitud de look-ahead interpolada (output de F2). Multiplicado por `dir_input` para dirección |
| `vertical_offset` | float | −8 px (fijo) | Elevación del encuadre. Negativo = más suelo visible adelante |

**Output range:** Sin clamping, [pos_edrick.x − look_ahead_max, pos_edrick.x + look_ahead_max] en X. Se clampea en F5.

**Nota de implementación:** `dir_input` es la entrada de input del jugador (discreta), **no** la velocidad. Esto permite que la intención del jugador mantenga el look-ahead incluso contra una pared (`velocity = 0`, pero `dir_input ≠ 0`). La magnitud viene de F2 (basada en velocidad). Ver E1 para comportamiento con paredes.

---

### F2 — Look-Ahead Dinámico (Magnitud Proporcional a Velocidad)

```gdscript
speed_ratio = clamp(abs(velocity_x) / max_speed, 0.0, 1.0)
look_ahead_raw = look_ahead_max * speed_ratio
look_ahead_next = lerp(look_ahead_current, look_ahead_raw, look_ahead_speed * delta)
```

**Variables:**

| Variable | Tipo | Rango | Descripción |
|----------|------|-------|-------------|
| `velocity_x` | float | [−400, +400] px/s | Velocidad horizontal de Edrick (incluye dash). **Nota**: usamos velocidad para magnitud, no dirección (F1 da dirección) |
| `max_speed` | float | 250 px/s | Velocidad base máxima. El dash (400 px/s) supera esto; se clampea al máximo |
| `speed_ratio` | float | [0.0, 1.0] | Velocidad normalizada como fracción de max_speed |
| `look_ahead_max` | float | {50, 20} px | Máximo look-ahead: 50 px exploración, 20 px combate. Interpolado dinámicamente por F3 |
| `look_ahead_raw` | float | [0, look_ahead_max] px | Look-ahead proporcional a velocidad actual (antes de suavizado) |
| `look_ahead_current` | float | [0, look_ahead_max] px | Look-ahead del frame anterior (estado persistente) |
| `look_ahead_next` | float | [0, look_ahead_max] px | Look-ahead calculado este frame (output → input para F1) |
| `look_ahead_speed` | float | 5.0 | Velocidad de interpolación (factor de lerp). Mayor = converge más rápido |
| `delta` | float | ~0.0167 s | Tiempo por frame a 60 fps |

**Output range:** [0, look_ahead_max] px. El lerp es asintótico (nunca alcanza 100% exactamente).

**Convergencia (tiempos reales con `look_ahead_speed = 5.0`):**
- De 0 → 95% de target: ~34 frames (~0.57s a 60 fps)
- De 0 → 80% de target: ~19 frames (~0.32s a 60 fps)
- De 50 → 0 px (detenerse): misma duración (~0.57s al 95%)

Cuando `dir_input = 0`, el look-ahead decae hacia 0 con la misma constante de tiempo. El lerp exponencial asintótico evita saltos abruptos.

---

### F3 — Transición de Modo (Interpolación de Parámetros de Seguimiento)

```gdscript
# Cuando combat_started o combat_ended se emite:
transition_elapsed = 0.0
look_ahead_from = look_ahead_max        # Valor actual en el momento
smoothing_from = smoothing_speed        # Valor actual en el momento
look_ahead_to = look_ahead_max_target   # 20 (combate) o 50 (exploración)
smoothing_to = smoothing_speed_target   # 9.0 (combate) o 6.0 (exploración)

# Cada frame durante la transición:
transition_elapsed += delta
t_norm = clamp(transition_elapsed / combat_transition_time, 0.0, 1.0)
look_ahead_max = lerp(look_ahead_from, look_ahead_to, t_norm)
smoothing_speed = lerp(smoothing_from, smoothing_to, t_norm)
```

**Variables:**

| Variable | Tipo | Rango | Descripción |
|----------|------|-------|-------------|
| `transition_elapsed` | float | [0, combat_transition_time] s | Tiempo acumulado desde inicio de transición. Reseteado cada vez |
| `combat_transition_time` | float | 0.4 s | Duración total del lerp entre modos |
| `look_ahead_from / look_ahead_to` | float | {20, 50} px | **Crítico**: `from` es el valor actual, no fijo. Permite interrupción suave |
| `smoothing_from / smoothing_to` | float | {6.0, 9.0} | **Crítico**: `from` es el valor actual, no fijo |
| `t_norm` | float | [0.0, 1.0] | Tiempo normalizado (0 = inicio, 1 = fin) |
| `look_ahead_max` | float | [20, 50] px | Resultado interpolado de look-ahead. Usado por F2 |
| `smoothing_speed` | float | [6.0, 9.0] | Resultado interpolado de smoothing. Usado en lerp manual de R2 |

**Output range:** `look_ahead_max` ∈ [20, 50] px, `smoothing_speed` ∈ [6.0, 9.0].

**Comportamiento de Interrupción:**
Si `combat_ended` se emite mientras transición está a `t_norm = 0.5`:
- `look_ahead_from_new = lerp(50, 20, 0.5) = 35 px` (valor actual)
- Transición inversa calcula desde (35 px, 7.5) hacia (50 px, 6.0)
- Resultado: transición suave sin saltos, aunque se haya interrumpido. Garantiza que señales rápidas no oscilen.

---

### F4 — Camera Anchor Blend (Entry and Exit)

**Entry transition (Edrick enters Area2D):**
```gdscript
anchor_t = clamp(anchor_elapsed / anchor.transition_time, 0.0, 1.0)
anchor_offset.x = lerp(0.0, anchor.offset_x, anchor_t)
anchor_offset.y = lerp(0.0, anchor.offset_y, anchor_t)
zoom_current = lerp(1.0, anchor.zoom_target, anchor_t)
```

**Exit transition (Edrick leaves Area2D):**
```gdscript
# Revert from current offset/zoom back to neutral (0.0, 1.0)
# Do NOT reset anchor_elapsed to 0 — continue countdown from current value
anchor_t = clamp(anchor_elapsed / anchor.transition_time, 0.0, 1.0)
anchor_offset.x = lerp(anchor_offset_current.x, 0.0, anchor_t)
anchor_offset.y = lerp(anchor_offset_current.y, 0.0, anchor_t)
zoom_current = lerp(zoom_current_prev, 1.0, anchor_t)
```
where `anchor_offset_current` = value of offset at exit time, `zoom_current_prev` = value of zoom at exit time.

**Architecture note (for multi-anchor support):** When a second anchor overlaps the first (E4), push the first anchor's state onto a stack before activating the new one. On exit, pop the stack to restore the previous anchor's state. A single `current_anchor` variable is insufficient.

**Integration with F1:**
```gdscript
target.x += anchor_offset.x
target.y += anchor_offset.y

# Zoom assignment (single authoritative site per frame):
# Applied AFTER all offset calculations, BEFORE F5 clamping
camera.zoom = Vector2(zoom_current, zoom_current)
```

**Variables:**

| Variable | Tipo | Rango | Descripción |
|----------|------|-------|-------------|
| `anchor_elapsed` | float | [0, transition_time] s | Tiempo desde que Edrick entró al Area2D del anchor |
| `anchor.offset_x / anchor.offset_y` | float | [−80, +80] px | Offset de composición definido manualmente en el anchor |
| `anchor.zoom_target` | float | [0.85, 1.2] | Zoom destino del anchor |
| `anchor.transition_time` | float | [0.2, 2.0] s | Duración de la transición de entrada/salida |
| `anchor_offset.x / anchor_offset.y` | float | [−80, +80] px | Offset interpolado (output) |
| `zoom_current` | float | [0.85, 1.2] | Zoom interpolado (output) |

**Output range:** Offset entre −80 y +80 px; zoom entre 0.85 (aleja) y 1.2 (acerca).

---

### F5 — Posición Final con Límites de Zona

```
# Safety: evitar división por cero o zoom negativo
zoom_safe = max(zoom_current, 0.01)

half_vp_w = 160.0 / zoom_safe
half_vp_h = 90.0 / zoom_safe
min_x = limit_left + half_vp_w
max_x = limit_right - half_vp_w
min_y = limit_top + half_vp_h
max_y = limit_bottom - half_vp_h

# Caso zona pequeña: si min > max, centrar automáticamente
if max_x < min_x:
    camera_final.x = (limit_left + limit_right) / 2.0
else:
    camera_final.x = clamp(target.x, min_x, max_x)

if max_y < min_y:
    camera_final.y = (limit_top + limit_bottom) / 2.0
else:
    camera_final.y = clamp(target.y, min_y, max_y)
```

**Variables:**

| Variable | Tipo | Rango | Descripción |
|----------|------|-------|-------------|
| `target.x / target.y` | float | [−∞, +∞] | Posición objetivo antes de clamping (output de F1 + F4) |
| `zoom_safe` | float | [0.01, 1.2] | Zoom actual, clampeado a mínimo 0.01 para evitar división por cero |
| `half_vp_w / half_vp_h` | float | [133, 188] / [75, 105] px | Mitad del viewport visible, ajustado por zoom. En Godot: asignar `camera.zoom = Vector2(zoom_safe, zoom_safe)` |
| `limit_left / limit_right / limit_top / limit_bottom` | float | definido por zona | Límites de cámara de la zona actual (cargados desde metadatos, no asignados a Camera2D.limit_*) |
| `camera_final.x / camera_final.y` | float | dentro de límites | Posición final de la cámara, clampeada contra límites de zona o centrada si zona < viewport |

**Output range:** Siempre dentro de los límites de la zona. El viewport interno es 320×180 px; ajustado por zoom en tiempo real.

**Caso límite:** Si la zona es más pequeña que el viewport (ej: sala de 240×140 px), la cámara se centra automáticamente en el punto medio `((limit_left + limit_right) / 2, (limit_top + limit_bottom) / 2)`. La fórmula detecta `max_x < min_x` e implementa el centrado.

---

## Edge Cases

**E1 — Cambio Rápido de Dirección (Pivote)**

**Situación**: Edrick corre a la derecha a 250 px/s (look-ahead = +50 px). En el siguiente frame presiona A y suelta D — input cambia a −1 (izquierda).

**Resultado**: `target.x` salta de `pos_edrick.x + 50` a `pos_edrick.x − 50` (delta de 100 px). **Sin embargo**, la cámara NO salta — el lerp manual (R2) interpola suavemente desde la posición actual hacia el nuevo target. Con `smoothing_speed = 6.0` en exploración, la interpolación toma ~8-12 frames (0.13-0.2s a 60 fps). Edrick pivota fluidamente; la cámara lo sigue con una curva suave sin glitch ni parpadeo. Si el delta del target excede `lerp_delta_clamp_max = 12 px`, el lerp se clampea por frame para evitar un jump visual excesivo (E1 el clamping es transparente porque el lerp lo absorbe naturalmente).

---

**E2 — Zona Más Pequeña que el Viewport**

**Situación**: Edrick entra a una sala de 240×140 px (más pequeña que el viewport de 320×180 px).

**Resultado**: Fórmula 5 detecta que `max_x < min_x`. La cámara automáticamente se centra en el punto medio de la zona: `camera_final.x = (limit_left + limit_right) / 2.0`. Lo mismo para Y. La zona está centrada en pantalla; el espacio vacío se rellena con fondo. Las zonas MVP se diseñan para ser ≥ 320×180 px; este caso es raro pero manejado.

---

**E3 — Transición de Zona Durante Combate Activo**

**Situación**: Sistema de Combate (#6) tiene `combat_started` activo (look_ahead = 20 px). Edrick toca una zona de transición.

**Resultado**: Cámara entra en TRANSITION (congela posición). Estado FOLLOW_COMBAT permanece pendiente. Durante fade-out (0.3s), la cámara no se mueve. Cuando la nueva zona carga, la cámara snap a la posición inicial de Edrick y restaura automáticamente los parámetros de combate. El combate continúa fluidamente en la nueva zona.

---

**E4 — Dos Camera Anchors Superpuestos**

**Situación**: Edrick está dentro de dos `Area2D` de anchors diferentes simultáneamente.

**Resultado**: Solo el anchor entrado más recientemente es activo. Prioridad por orden de entrada, no por overlap. Si Edrick sale del anchor más reciente pero sigue dentro del otro, la cámara revierte al anchor anterior. Cada entrada/salida triggeriza un nuevo lerp de `transition_time`. **Recomendación**: evitar sobreposiciones en el diseño de niveles.

---

**E5 — Dash a 400 px/s (Velocidad Extrema)**

**Situación**: Edrick corre a 250 px/s (look-ahead = 50 px). Presiona SHIFT → dash a 400 px/s durante 0.15s.

**Resultado**: Fórmula 2: `speed_ratio = clamp(400 / 250, 0.0, 1.0) = 1.0`. Look-ahead se mantiene en máximo (50 px); no aumenta más allá. El dash se siente rápido sin que el encuadre "tiemble" por intentar anticipar velocidades imposibles. La cámara sigue con suavizado normal.

---

**E6 — Zona Cargando Lentamente (Fade Termina Primero)**

**Situación**: Nueva zona tarda 0.45s en instanciar; FADE_DURATION = 0.3s. El fade termina pero la zona no está lista.

**Resultado**: Cámara permanece congelada en pantalla negra esperando a `zona_transition_ended`. Una vez que la zona carga, la cámara snap a la posición inicial de Edrick. Nota técnica: problema de timing de Exploración (#8), no de Cámara.

---

**E7 — Zoom Extremo + Offset Grande de Anchor**

**Situación**: Anchor define `zoom_target = 1.2` y `offset_x = +80 px`. Edrick entra desde la esquina izquierda de una zona pequeña.

**Resultado**: `zoom_current = 1.2` → `half_vp_w = 133 px`. El offset se desplaza a la derecha. Fórmula 5 clampea contra límites. El offset se trunca en el límite derecho sin glitch — la cámara se detiene donde el límite la obliga.

---

## Dependencies

### Dependencias de Este Sistema

Este sistema **depende de**:
- **GDD #1 — Movimiento y Físicas 2D**: Lee `CharacterBody2D.position` y `.velocity.x` de Edrick cada frame. La cámara es read-only; no modifica el movimiento.
- **GDD #6 — Combate en Tiempo Real**: Recibe señales `combat_started` y `combat_ended` vía EventBus. Usa estas señales para transicionar entre submodos FOLLOW_EXPLORE ↔ FOLLOW_COMBAT.
- **GDD #8 — Exploración del Mundo**: Recibe señales `zona_transition_started` y `zona_transition_ended`. Lee los límites de zona de los metadatos de cada escena cargada.

Sistemas que **dependen de este sistema**:
- **GDD #17 — Cinemáticas**: Recibe control de la cámara cuando se inicia una cinemática. La cámara entra en modo CINEMATIC y acepta comandos de posición/zoom/easing. Al terminar, vuelve a modo FOLLOW_EXPLORE.

---

## Tuning Knobs

| Parámetro | Valor Base | Rango Seguro | Aspecto Afectado |
|-----------|-----------|--------------|-----------------|
| `look_ahead_explore` | 50 px | 30–80 px | Máximo desplazamiento anticipado en exploración. Mayor = más mundo visible adelante. |
| `look_ahead_combat` | 20 px | 10–40 px | Máximo desplazamiento en combate. Menor = más foco visual en Edrick. |
| `smoothing_speed_explore` | 6.0 | 4.0–10.0 | Velocidad de interpolación lerp en exploración (multiplicador de delta). Mayor = más "snappy", menor = más suave. |
| `smoothing_speed_combat` | 9.0 | 6.0–12.0 | Velocidad de interpolación lerp en combate. Mayor = más ágil, menor = más pesado. |
| `combat_transition_time` | 0.4 s | 0.2–0.8 s | Duración del lerp F3 al cambiar de exploración ↔ combate. |
| `look_ahead_speed` | 5.0 | 3.0–8.0 | Velocidad de decaimiento del look-ahead cuando input horizontal = 0. Controla qué tan rápido se centra la cámara al detenerse (F2). |
| `vertical_offset` | −8 px | −15 a 0 px | Elevación del encuadre. Más negativo = más suelo visible delante de Edrick. |
| `anchor_offset_max` | ±80 px | ±50 a ±100 px | Rango máximo de desplazamiento de un CameraAnchor (F4). |
| `anchor_zoom_min` | 0.85 | 0.75–0.95 | Zoom mínimo permitido en anchors (aleja). |
| `anchor_zoom_max` | 1.2 | 1.1–1.5 | Zoom máximo permitido en anchors (acerca). |
| `lerp_delta_clamp_max` | 12 px | 8–15 px | Máxima diferencia de posición de cámara por frame en lerp (safety). Si `|(target - current)| > lerp_delta_clamp_max` por frame, clampear el delta para evitar teleport durante grandes saltos de target (ej: pivote rápido). |

---

## Visual/Audio Requirements

No hay requisitos visuales o de audio específicos para el sistema de cámara en el GDD. Las cinemáticas (#17) definirán sus propios requisitos cuando necesiten cámara con propósitos narrativos. El sistema de cámara es infraestructura transparente — el feedback visual (cambios de encuadre) es intencional, no accidental. No hay VFX, partículas o audio específicos del sistema de cámara.

---

## UI Requirements

No hay requisitos de UI específicos. El sistema de cámara no genera elementos de interfaz. Las cinemáticas podrían superponer controles narrativos (diálogos, cinemáticas) que la cámara debe respetar sin interferencias — ver GDD #17 para detalles.

---

## Acceptance Criteria

### Criterios de Seguimiento Suave (AC 1-4)

- [ ] **AC 1: Suavizado Nativo Desactivado**
  - Verificar `Camera2D.position_smoothing_enabled = false` en runtime startup.
  - Verificar `Camera2D.drag_horizontal_enabled = false` y `drag_vertical_enabled = false`.
  - Verificar `Camera2D.limit_left = Camera2D.limit_right = Camera2D.limit_top = Camera2D.limit_bottom = -1` (disabled).
  - **Pass**: no propiedades nativas de Camera2D interfieren con el lerp manual. Log output: "✓ Camera2D properties correctly configured for manual smoothing."

- [ ] **AC 2: Look-Ahead Horizontal en Exploración**
  - Overlay de debug: mostrar `look_ahead_current`, `edrick.position.x`, `camera.position.x`.
  - Edrick inmóvil: `look_ahead_current = 0 px`.
  - Edrick camina (50 px/s): `look_ahead_current` alcanza ~9.5 px (~0.57s, 34 frames, 95% de 10px target).
  - Edrick corre (250 px/s): `look_ahead_current` alcanza ~47.5 px (~0.57s, 34 frames, 95% de 50px target).
  - Pivote: `look_ahead_current` decae 50 → 0 px durante ~34 frames (asintótico, 95% en 34 frames).
  - **Pass**: `look_ahead_current` ∈ [0, 50] px, asintótico convergence con `look_ahead_speed = 5.0`, no cambios discretos/saltados.

- [ ] **AC 3: Look-Ahead Reducido en Combate**
  - Emitir `combat_started` en FOLLOW_EXPLORE.
  - Verificar `look_ahead_max` interpola 50 → 20 px en 0.4s (±0.05s).
  - Verificar `smoothing_speed` interpola 6.0 → 9.0 en 0.4s (±0.05s).
  - Emitir `combat_ended`: interpola 20 → 50 px y 9.0 → 6.0 en 0.4s (±0.05s).
  - **Pass**: transiciones suaves bidireccionales con duraciones correctas.

- [ ] **AC 4: Pivote Suave (Cambio Rápido de Dirección)**
  - Edrick corriendo 250 px/s a la derecha (look-ahead = 50 px).
  - Frame N: presionar A, soltar D → `dir_input` = −1.
  - Registrar `camera.position.x` frames N a N+70.
  - Verificar `max(delta_position)` ≤ 12 px en cualquier frame (lerp_delta_clamp_max).
  - Verificar `camera.position` alcanza destino (`edrick.x − 50`) en ~30-40 frames (asintótico convergence via F2, 95% en 34 frames).
  - **Pass**: transición fluida, sin saltos > 12 px, duración ~34 frames al 95% de convergencia. No snap, no parpadeo.

### Criterios de Límites y Anchors (AC 5-7)

- [ ] **AC 5: Límites de Zona Funcionan**
  - Edrick alcanza borde derecho de zona.
  - Verificar `camera.position.x` ≤ `limit_right − half_vp_w`.
  - Cámara no muestra píxeles fuera del límite.
  - **Pass**: límites respetados.

- [ ] **AC 6: Zona Pequeña se Centra**
  - Crear zona 240×140 px (< 320×180 viewport).
  - Edrick entra.
  - Verificar `camera.position.x ≈ (limit_left + limit_right) / 2` (±2 px tolerancia).
  - Verificar lo mismo para Y.
  - **Pass**: centrado automático detecta `max < min` e implementa centroide.

- [ ] **AC 7A: Entrada a Camera Anchor**
  - Anchor: `offset_x = +30 px`, `zoom_target = 1.1`, `transition_time = 0.5s`.
  - Edrick entra al `Area2D`.
  - Overlay muestra `anchor_offset.x`, `zoom_current`.
  - Verificar `anchor_offset.x` interpola 0 → 30 px en 0.5s (±0.05s).
  - Verificar `zoom_current` interpola 1.0 → 1.1 en 0.5s (±0.05s).
  - Verificar `camera.zoom = Vector2(zoom_current, zoom_current)` se aplica.
  - **Pass**: interpolaciones suaves, duraciones correctas.

- [ ] **AC 7B: Salida de Camera Anchor**
  - Edrick dentro de anchor activo (offset = 30 px, zoom = 1.1).
  - Edrick sale del `Area2D`.
  - Verificar `anchor_offset.x` interpola 30 → 0 px en 0.5s (±0.05s).
  - Verificar `zoom_current` interpola 1.1 → 1.0 en 0.5s (±0.05s).
  - Transición de salida simétrica a entrada (sin saltos).
  - **Pass**: reversión suave, simetría confirmada.

### Criterios de Transiciones de Zona y Cinemáticas (AC 8-10)

- [ ] **AC 8: Transición de Zona — Freeze y Snap Correcto**
  - FOLLOW_EXPLORE: Edrick toca zona de transición en `camera.position = (150, 90)`.
  - `zona_transition_started` emitida: cámara entra en TRANSITION state.
  - Log assertion: "Camera frozen, position = (150, 90)". Verify `camera.position` unchanged for 0.3s (fade duration).
  - Nueva zona cargada, `zona_transition_ended` emitida: cámara snap a `edrick_new_zone.position = (120, 80)` (verificar con log).
  - Verify post-snap: `camera.position` ∈ [(118, 78), (122, 82)] (±2 px tolerancia).
  - `zona_transition_ended`: cámara reactiva suavizado, entra en FOLLOW_EXPLORE.
  - Verify frame siguiente (N+1): `camera.position` comienza lerp suave (≤ 12px delta desde snap).
  - **Pass**: log chain correcta, snap invisible (ocurre durante fade negro), suavizado reanudado sin glitch.

- [ ] **AC 9A: Cinemática Inicio/Fin**
  - FOLLOW_EXPLORE: cinemática emite `cinematic_started(camera_data: {position_target, zoom_target, transition_time, easing})`.
  - Cámara entra CINEMATIC.
  - Cámara interpola hacia `position_target` y `zoom_target` durante `transition_time`.
  - Cinemática termina: `cinematic_ended()`.
  - Cámara vuelve a FOLLOW_EXPLORE.
  - **Pass**: estados correctos, interpolación suave según `camera_data`.

- [ ] **AC 9B: CINEMATIC Rechazado Desde TRANSITION (No puede entrar)**
  - Iniciar transición zona: estado = TRANSITION.
  - Emitir `cinematic_started(camera_data: {...})` durante TRANSITION.
  - Camera controller recibe signal: verifica current state == TRANSITION.
  - Behavior: **rechaza entrada** (ignora la señal, no la encola). Log: "Cinematic request ignored — camera in TRANSITION state."
  - `zona_transition_ended`: zona carga, cámara entra FOLLOW_EXPLORE.
  - Ahora emitir nuevamente `cinematic_started(...)`.
  - Cámara **acepta** (verifica state == FOLLOW_EXPLORE). Log: "Cinematic started."
  - **Pass**: primera señal rechazada silenciosamente; segunda señal aceptada. Sem crashes, sem deadlock.

- [ ] **AC 9C: Estado Post-Cinemática con Combate Activo**
  - FOLLOW_COMBAT (combate activo).
  - Cinemática: `cinematic_started(...)`.
  - Cámara CINEMATIC, ejecuta cinemática.
  - Cinemática `cinematic_ended()`.
  - Verificar estado = FOLLOW_EXPLORE (NO FOLLOW_COMBAT).
  - Nota: Si combate sigue activo, Sistema de Combate debe re-emitir `combat_started`.
  - **Pass**: estado post-cinemática siempre FOLLOW_EXPLORE.

- [ ] **AC 10: Input Responsiveness — No Perceptible Lag**
  - Add frame-counter instrumentation to `_input()` and `_process()`.
  - Frame N: player presses right arrow → `dir_input` changes from 0 to +1 in input polling.
  - Frame N or N+1: `look_ahead_current` begins updating (lerp toward new target).
  - Frame N+1 **latest**: `camera.position.x` begins moving toward new target (visible in next rendered frame).
  - Acceptable latency: 1-2 frames (≤33ms at 60fps). 
  - Test via: Manual keybind press + frame logger in _process(). No automated test required — visual inspection confirms "responsive feel."
  - **Pass**: input press → camera reaction within 2 frames (≤33ms).

### Criterios de Edge Cases (AC 11-15)

- [ ] **AC 11: Transición de Zona Durante Combate Activo (E3)**
  - `combat_started` emitido: cámara FOLLOW_COMBAT.
  - Edrick toca zona transición: `zona_transition_started`.
  - Verificar estado = TRANSITION (independientemente de combate).
  - Cámara congelada durante fade (delta = 0).
  - Post-`zona_transition_ended`: estado = FOLLOW_EXPLORE.
  - Si combate sigue: Sistema de Combate emite `combat_started` nuevamente.
  - **Pass**: TRANSITION tiene prioridad, post-transición FOLLOW_EXPLORE restaurable.

- [ ] **AC 12: Dos Camera Anchors Superpuestos (E4)**
  - Zona con Anchor A (`offset_x = +30 px`) y Anchor B (`offset_x = −40 px`) superpuestos.
  - Edrick entra A: `anchor_offset.x` → +30 px.
  - Edrick entra B (sigue en A): `anchor_offset.x` → −40 px (nuevo target).
  - Edrick sale B (sigue en A): `anchor_offset.x` → +30 px (revierte a A).
  - Edrick sale A: `anchor_offset.x` → 0 px.
  - **Pass**: último anchor entrado es activo, reversión correcta, sin saltos.

- [ ] **AC 13: Dash a 400 px/s (E5)**
  - Edrick corre 250 px/s: `look_ahead_current` = 50 px.
  - Edrick dash a 400 px/s (SHIFT).
  - Verificar `speed_ratio = clamp(400/250, 0, 1) = 1.0` (máximo).
  - Verificar `look_ahead_current` NO excede 50 px.
  - Dash mantiene look-ahead en máximo, no salta.
  - **Pass**: clamping de velocidad funciona, look-ahead controlado.

- [ ] **AC 14: Zona Cargando Lentamente (E6)**
  - Nueva zona carga en 0.45s (FADE_DURATION = 0.3s).
  - Fade-out 0.3s: pantalla negra, cámara congelada.
  - Cámara espera (zona aún cargando).
  - `zona_transition_ended` emitido en 0.45s.
  - Cámara snap a posición nueva zona.
  - Fade-in 0.3s: sin frames negros residuales.
  - **Pass**: cámara espera, sin prematuramente, sin glitch.

- [ ] **AC 15: Zoom Extremo + Offset Grande en Zona Pequeña (E7)**
  - Zona 240×180 px.
  - Anchor: `zoom_target = 1.2`, `offset_x = +80 px`.
  - Edrick entra: `zoom_current = 1.2` → `half_vp_w = 133 px`.
  - F5 detecta `max_x = 107 < min_x = 133`.
  - Cámara centra: `camera_final.x = 120 px` (punto medio).
  - **Pass**: centrado automático bajo zoom extremo + offset, sin degeneración.

### Acceptance Criteria — Player Fantasy Validation

- [ ] **AC 16: Camera Feel — New Observer Test**
  - Recruit a playtest observer unfamiliar with the GDD.
  - Have them play 10–15 minutes of exploration gameplay (mixed zones, some with anchors, some without).
  - Ask: "How would you describe how the camera behaves?"
  - **Pass**: Observer mentions organic/natural/smooth/cinematic phrasing OR silently enjoys the exploration without mentioning camera. 
  - **Fail**: Observer mentions camera negatively (e.g., "jerky," "feels tracked," "too anticipatory," "mechanical").
  - This validates the "composición emergente" Player Fantasy.

- [ ] **AC 17: Rule-of-Thirds Composition (80% of Zones)**
  - Automated or manual audit: traverse 10 non-anchor exploration zones (no CameraAnchors active).
  - Measure: what percentage of playtime is Edrick's position within the rule-of-thirds box (33–66% of screen horizontally, 33–66% vertically)?
  - Threshold: ≥70% of playtime falls within rule-of-thirds (comfortable cinematic framing).
  - Measure at `smoothing_speed = 6.0` (exploration mode), normal movement speeds.
  - **Pass**: ≥8 of 10 zones maintain rule-of-thirds for ≥70% of playtime.
  - This validates that the camera naturally frames Edrick cinematically without forced anchors.

---

## Open Questions

- **¿Parallax de fondo sigue automáticamente a la cámara, o se define por zona?** Esperando clarificación del Sistema de Exploración (#8) sobre cómo se estructuran los assets de parallax.
- **¿Las cinemáticas pueden desactivar los límites de zona?** (ej: para una toma de cámara panorámica que sale fuera de la zona). Será definido en GDD #17.
- **¿Hay un fade de transición de modo exploración↔combate visualmente, o solo el cambio de parámetros de cámara es suficiente?** Depende del feedback visual del GDD de Combate #6.
