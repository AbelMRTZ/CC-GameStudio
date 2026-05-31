# GDD: Transformación Visual de Edrick

> **Status**: En Revisión (NEEDS REVISION — revisión completada 2026-05-31)
> **Author**: Abel + Claude Code Agents
> **Last Updated**: 2026-05-31 (revisión post-review: anchor moment, Dash tint, POST_BINDING, fórmulas)
> **Implements Pillar**: Pilar 2 — Demonios como Poder Transformador; Pilar 5 — Transformación Moral de Edrick
> **Milestone**: MVP — Feature Layer
> **Depende de**: Loadout & Build Management (#10), Base de Datos de Demonios (#3)
> **Dependen de este sistema**: Build Management UI (#20)

---

## 1. Visión General

La Transformación Visual de Edrick da cuerpo visible al Pilar 2: cada demonio que Edrick porta cambia su apariencia de una manera propia y distinta. Internamente, el sistema define un **schema estructurado de capas de transformación** — aura de fondo, tono de sprite, overlay de ojos, overlay de partículas, efectos floating, y otras — que la Base de Datos de Demonios (#3) popula por demonio. Este sistema lee esas capas y las aplica en tiempo real al sprite de Edrick. No todos los demonios modifican todas las capas: `none` es un valor válido para cualquier capa de cualquier demonio. Un demonio puede cambiar solo los ojos, otro puede añadir solo un efecto flotante, otro puede no cambiar nada visible.

Con múltiples demonios equipados simultáneamente (hasta 2 slots en MVP), el sistema aplica cada transformación activa en su propia capa independiente: los efectos no se anulan entre sí ni uno domina sobre el otro. Si Fuego añade tono ígneo al sprite y Arcano añade runas flotantes, ambas capas son visibles al mismo tiempo. El resultado visual es Edrick compuesto — la representación directa de que porta más de un demonio.

El sistema gestiona también la **frontera con Vinculación (#13)**: durante la secuencia de binding, GDD #13 produce una distorsión de aura oscura sobre Edrick. Al recibir `demon_bound`, este sistema toma ownership de esa distorsión y la integra como la manifestación permanente del demonio recién vinculado — continuidad visual sin corte entre la secuencia de binding y el estado equipado normal.

El sistema está además preparado para la capa de **corrupción moral** (GDD #22, Vertical Slice): el nivel de corrupción afectará la apariencia base de Edrick sobre la que se superponen las transformaciones por demonio. En MVP la apariencia base es fija; el hook para la capa de corrupción se define aquí para evitar deuda técnica.

---

## 2. Fantasy del Jugador

La transformación visual es la **confirmación de que el cambio es real**. No la promesa de un poder nuevo: la evidencia de que algo en Edrick ya no es como antes.

Cuando el jugador equipa un demonio y ve cómo el sprite de Edrick se altera — ojos que brillan con una luz que no era suya, runas que orbitan sus hombros, un tono en la piel que no existía hace un momento — no siente que "desbloqueó algo". Siente que *cargó con algo*. La transformación visible es el juego diciendo: *"Esto no se puede deshacer. Ahora eso es parte de ti."*

Esa incomodidad es intencional. El jugador quiere el poder. Pero la transformación visual le recuerda, sin texto y sin UI, cuánto de Edrick está cediendo para tenerlo. La fantasy opera en esa tensión: el placer de ver a tu personaje volverse más poderoso, mezclado con el leve malestar de que ya no reconoces completamente quién es.

Con múltiples demonios activos, la fantasy se extiende: el jugador ve un Edrick **compuesto**, portador de más de una marca. Eso se debe sentir acumulativo — no como una paleta de colores revuelta, sino como capas de identidad superpuestas, cada una legible pero todas presentes. La build de demonios se hace visible. La elección del jugador se imprime en el personaje.

**Momento ancla**: el jugador swappea un demonio mid-exploración, ve a Edrick cambiar durante la animación de 0.8 segundos, y antes de que la animación termine ya se está preguntando si quiere este cambio o si prefería cómo se veía antes. Esa duda es el éxito del sistema.

**Alineación de pilares**: Pilar 2 — *"Transformación visible — cuando equipas un demonio, Edrick se ve diferente, se siente diferente"*. Pilar 5 — *"la corrupción se siente necesaria, incluso ganada"* — la acumulación visual de marcas demoníacas debe sentirse como el precio pagado, no como una colección de badges.

---

## 3. Diseño Detallado

### 3.1 Reglas Centrales

**A. Arquitectura de capas**

La transformación de Edrick se representa como un conjunto de cinco capas independientes aplicadas sobre su sprite base. Cada demonio puede aportar a 0, 1, o múltiples capas. `none` es válido para cualquier capa.

| Capa | Descripción | Ejemplos MVP |
|------|-------------|--------------|
| **`aura_bg`** | Aura ambiente alrededor/detrás del sprite | Fuego = naranja, Hielo = azul/blanco, Arcano = púrpura/dorado |
| **`sprite_tint`** | Tinte de color sobre el sprite (piel/ropa) | Fuego = tonos ígneos |
| **`eye_overlay`** | Efecto visual sobre los ojos | Visión = brillo incoloro, Mente = brillo platino |
| **`floating_fx`** | Elementos que orbitan a Edrick | Arcano = runas flotantes, Hielo = cristales |
| **`motion_trail`** | Estela de movimiento detrás de Edrick, solo activa mientras se mueve por encima de un umbral de velocidad. Desaparece al detenerse. | Dash = estela sutil de velocidad |

**B. Regla de `sprite_tint` sutil para Dash**

El demonio Dash no aporta `aura_bg`, `eye_overlay` ni `floating_fx` en MVP. Su efecto visual primario es la estela de movimiento (`motion_trail`) que comunica velocidad y agilidad sin competir con efectos de otros demonios.

**Indicador en reposo**: Dash aporta un `sprite_tint` muy sutil (blend_weight=0.08, color gris-blanco frío `#D8E4F0`) que distingue a Edrick del estado base IDLE incluso sin movimiento. El tinte es deliberadamente débil — comunica "algo ha cambiado" sin proclamarlo. El aspecto base sigue siendo reconocible. Esta capa resuelve la violación del Pilar 2 para el primer binding del Acto 1: el primer demonio vinculado debe dejar una marca visible.

**C. El Gato no genera capas en Edrick**

El demonio Gato vive en el Gato, no en Edrick. No contribuye ninguna capa de transformación. `gato_available` es ignorado por este sistema.

**D. Timing de actualizaciones — modelo de blend progresivo**

La transformación visual **es** la animación, no una actualización al finalizarla. El sistema opera con un blend lineal que corre durante los 0.8s de SWAP_ANIM:

1. **Al recibir `loadout_swap_started(new_demons)`** (señal nueva emitida por GDD #10 al **INICIO** de SWAP_ANIM): el sistema comienza un blend lineal de 0.8s de duración.
   - `t = 0.0s`: capas anteriores al 100% — capas nuevas al 0%
   - `t = 0.4s`: capas anteriores al 50% — capas nuevas al 50%
   - `t = 0.8s`: capas anteriores al 0% — capas nuevas al ~100% (bloqueo al recibir la señal siguiente)
2. **Al recibir `loadout_changed(equipped_demons, gato_available)`** (señal existente, emitida por GDD #10 al **FINAL** de SWAP_ANIM): el sistema bloquea las capas nuevas al 100% y descarta las anteriores.
3. **Al recibir `demon_bound(demon_id)`**: handoff del aura post-binding. Ver §3.4.

**⚠ Dependencia de GDD #10 (P-TVE-07)**: La señal `loadout_swap_started` es nueva y debe ser añadida a GDD #10 antes del primer sprint de implementación.

---

### 3.2 Schema de Transformación Visual

Este GDD define el schema estructurado que GDD #3 debe usar para el campo `transformacion_visual`. Reemplaza el campo `aura: string` actual de GDD #3.

```
transformacion_visual:
  aura_bg:
    color: Color            # Color del aura. Ej: Color(1.0, 0.4, 0.0) para naranja
    intensity: float        # [0.0 – 1.0] intensidad máxima de la capa
    style: String           # "glow" | "distortion" | "particles"
  
  sprite_tint:
    color: Color
    blend_weight: float     # [0.0 – 1.0] cuánto afecta el tinte al sprite

  eye_overlay:
    color: Color
    pulse: bool             # true = efecto que pulsa; false = estático
    style: String           # "glow" | "shine" | "void"

  floating_fx:
    type: String            # "runes" | "crystals" | "sparks"
    color: Color
    count: int              # [1 – 12] elementos flotantes
    orbit_radius: float     # [16 – 64] px desde el centro de Edrick
    speed: float            # radianes/s de rotación

  motion_trail:
    color: Color            # color de los ghost frames de la estela
    opacity_peak: float     # [0.0 – 1.0] opacidad máxima del ghost más reciente
    trail_length: int       # [1 – 8] número de ghost frames en la estela
    activation_speed: float # px/s mínimos para mostrar la estela
    fade_time: float        # segundos hasta desaparecer al detenerse
```

Cualquier capa puede ser `null` (no aplica para ese demonio).

**Mapping provisional de los demonios MVP** *(valores a confirmar con Art Direction en §8)*:

| Demonio | aura_bg | sprite_tint | eye_overlay | floating_fx | motion_trail |
|---------|---------|-------------|-------------|-------------|--------------|
| Fuego | naranja, 0.7, glow | naranja, 0.25 | none | none | none |
| Hielo | azul/blanco, 0.6, particles | none | none | crystals, blanco, 6 | none |
| Arcano | púrpura/dorado, 0.8, glow | none | none | runes, dorado, 4 | none |
| Visión | incoloro, 0.3, distortion | none | incoloro, pulse=false, void | none | none |
| Mente | platino/blanco, 0.5, glow | none | platino, pulse=true, shine | none | none |
| Dash | none | gris-blanco frío `#D8E4F0`, 0.08 | none | none | gris/blanco, opacity 0.15, length 3, speed_min 50 px/s, fade 0.1s |
| Gato | (no aplica — vive en el Gato) | — | — | — | — |

---

### 3.3 Reglas de Combinación (Multi-Demonio)

Con 2+ demonios equipados que aportan la misma capa:

**Fórmula de normalización (aplica a aura_bg, sprite_tint, floating_fx):**
```
intensidad_efectiva_i = intensidad_base_i / N_activos_en_capa
```
donde `N_activos_en_capa` = número de demonios equipados con esa capa como no-null.

**Por capa:**

- **`aura_bg`**: blend aditivo de colores a intensidad normalizada. Ambas auras visibles simultáneamente.
- **`sprite_tint`**: blend aditivo de tintes a peso normalizado. Los colores se mezclan.
- **`eye_overlay`** (regla híbrida):
  ```
  N = 1 demonio → ambos ojos toman el color de ese demonio al 100%
  N = 2 demonios → heterocromia:
      ojo izquierdo = demonio en slot 1 (100% de su color)
      ojo derecho   = demonio en slot 2 (100% de su color)
  N ≥ 3 demonios → blend aditivo normalizado en ambos ojos
      (ambos ojos toman el color mezclado de todos los eye_overlays activos)
  ```
  *En MVP (máximo 2 slots), el caso N≥3 no es alcanzable. La regla se define para compatibilidad futura con slots adicionales.*

- **`floating_fx`**: todos los efectos flotantes de todos los demonios activos coexisten. Cada set mantiene su `count` y `orbit_radius` propios. El total de elementos en pantalla es la suma de los `count` individuales.
- **`motion_trail`**: si 2+ demonios tienen `motion_trail`, las estelas se renderizan de forma aditiva (ambas visibles). En MVP solo Dash tiene esta capa — no hay combinación posible.

**Riesgo de legibilidad**: dos demonios con `aura_bg` y `sprite_tint` simultáneos pueden generar ruido visual. El tuning knob `MAX_BLEND_INTENSITY` (§7) limita la intensidad combinada máxima. Art Direction debe validar que los pares de demonios comunes en MVP (Fuego+Arcano, Fuego+Hielo, Hielo+Arcano, Dash+cualquiera) sean legibles. Ver §8.

---

### 3.4 Estados y Transiciones

| Estado | Descripción | Entrada | Salida |
|--------|-------------|---------|--------|
| **IDLE** | Sin demonios equipados. Sprite base sin capas activas. | Inicio del juego; todos los slots vacíos | SINGLE_DEMON al equipar el primero |
| **SINGLE_DEMON** | Un demonio activo. Sus capas al 100% de intensidad (sin normalización). | IDLE; MULTI_DEMON al limpiar a 1 | MULTI_DEMON al equipar segundo; TRANSITION al swap |
| **MULTI_DEMON** | 2+ demonios activos. Capas con blend normalizado. | SINGLE_DEMON | SINGLE_DEMON si queda uno; TRANSITION al swap |
| **TRANSITION** | SWAP_ANIM en curso (0.8s). Capas del estado anterior hacen fade-out; capas del nuevo estado hacen fade-in progresivo lineal. Al recibir `loadout_changed`, las capas nuevas se bloquean al 100%. | SINGLE_DEMON, MULTI_DEMON | Estado correspondiente al finalizar SWAP_ANIM |
| **POST_BINDING** | `demon_bound` recibido. Transición de la distorsión de GDD #13 a representación permanente. | IDLE, SINGLE_DEMON, **MULTI_DEMON** | SINGLE_DEMON o MULTI_DEMON al completar (según cuántos demonios estén equipados en ese momento) |

**Estado POST_BINDING — contrato de handoff con GDD #13 (P-VD-03 resuelto):**

1. Al recibir `demon_bound(demon_id)`, el sistema lee las capas del demonio recién vinculado de GDD #3.
2. GDD #13 tiene activa una distorsión oscura de `aura_bg` sobre Edrick (persistida tras el binding). Esta distorsión usa el mismo canal que este sistema.
3. Este sistema ejecuta un **crossfade** de duración `BINDING_AURA_TRANSITION` (tunable, MVP = 0.5s): la distorsión de GDD #13 hace fade-out mientras el `aura_bg` permanente del nuevo demonio hace fade-in simultáneamente.
4. Si el demonio recién vinculado no tiene `aura_bg` (p.ej. Dash), la distorsión hace fade-out a transparente sin reemplazarse.
5. Al completar el crossfade, el estado transiciona a SINGLE_DEMON (o MULTI_DEMON si ya había otro demonio equipado).

**Interfaz de handoff (P-VD-03 definido):**
- **Trigger**: señal `demon_bound(demon_id)` del EventBus
- **Mecanismo**: sobreescritura de los parámetros del shader del sprite de Edrick con animación de crossfade
- **GDD #13 no emite ninguna señal adicional de "release"** — su responsabilidad termina al emitir `demon_bound`
- El handoff ocurre post-descongelación del mundo (después de que el gameplay reanuda)

**Nota de implementación — cancelación del crossfade (ver AC-TVE-029):**
Si `loadout_changed` llega mientras el crossfade está en curso, la implementación debe cancelarlo. Antes de escribir el código, decidir entre: (a) llamar `crossfade_tween.kill()` y resetear opacidades, (b) usar un flag `_crossfade_active` que el Tween chequea en cada step, o (c) ambos. La decisión tiene implicaciones en el comportamiento edge case de frames parciales — documentar en el ADR correspondiente.

---

### 3.5 Interacciones con Otros Sistemas

| Sistema | Dirección | Qué fluye |
|---------|-----------|-----------|
| **Loadout (#10)** | Loadout → VisualTransform | `loadout_changed(equipped_demons, gato_available)` al finalizar SWAP_ANIM. VisualTransform actualiza sus capas activas. |
| **Base de Datos de Demonios (#3)** | BD → VisualTransform | `transformacion_visual` por demon_id (schema §3.2). Read-only. GDD #3 debe retrofittarse para usar este schema. |
| **Vinculación (#13)** | Vinculación → VisualTransform | `demon_bound(demon_id)` — dispara POST_BINDING y handoff de distorsión. |
| **Estado del Mundo (#4)** | EstadoMundo → VisualTransform | `corruption_level` — **HOOK MVP**: interfaz definida aquí, no implementada hasta GDD #22 (Seguimiento Moral). |
| **VisualTransform → Sprite** | VisualTransform → Shader/Sprite | Escribe parámetros del shader de Edrick (colores, intensidades, blend weights, trail params) en cada actualización de capas. |

---

## 4. Fórmulas

### F-TVE-01: Normalización de Intensidad de Capa (multi-demonio)

Cuando N demonios aportan la misma capa, cada uno aplica su intensidad proporcional:

```
intensity_effective_i = intensity_base_i × (1 / N_active) × BLEND_SCALE
```

**Variables:**
| Variable | Símbolo | Tipo | Rango | Descripción |
|----------|---------|------|-------|-------------|
| Intensidad base del demonio i | `intensity_base_i` | float | [0.0, 1.0] | Valor del campo `intensity` en schema §3.2 |
| N demonios activos en esta capa | `N_active` | int | [1, 5] | Demonios equipados con esa capa como no-null |
| Factor de escala de blend | `BLEND_SCALE` | float | [0.5, 1.0] | Tuning knob global. 1.0 = puramente proporcional |
| Intensidad efectiva | `intensity_effective_i` | float | [0.0, 1.0] | Valor final aplicado a la capa |

**Rango de salida:** (0.0, MAX_BLEND_INTENSITY] = (0.0, 0.9] — con BLEND_SCALE ≤ 1.0 (ver §7), el factor `1/N` garantiza que `intensity_effective_i ≤ intensity_base_i`; MAX_BLEND_INTENSITY actúa como cap final antes del shader para el caso boundary `intensity_base=1.0, N=1`.

**Ejemplo (MVP, N=2):** Fuego (`intensity=0.7`) + Arcano (`intensity=0.8`), BLEND_SCALE=1.0
- Fuego efectivo: `0.7 × 0.5 = 0.35`
- Arcano efectivo: `0.8 × 0.5 = 0.40` — ambas visibles y distinguibles

**⚠ Flag Art Direction**: Visión (`intensity=0.3`) con N=2 queda en `0.15` — muy tenue. Si en playtest resulta invisible, ajustar `BLEND_SCALE > 1.0` o añadir un floor mínimo. Validar en §8.

---

### F-TVE-02: Blend de Color de `sprite_tint` (multi-demonio)

Promedio ponderado por `blend_weight` en espacio RGB:

```
w_i_norm = blend_weight_i / Σ(blend_weight_j)
C_tint_out = Σ(C_i × w_i_norm,  para i = 1..N)
```

**Variables:**
| Variable | Símbolo | Tipo | Rango | Descripción |
|----------|---------|------|-------|-------------|
| Color de tinte del demonio i | `C_i` | Color RGBA | [0,0,0,0]–[1,1,1,1] | Campo `color` del `sprite_tint` del demonio i |
| Peso de blend del demonio i | `blend_weight_i` | float | [0.0, 1.0] | Campo `blend_weight` del schema §3.2 |
| Peso normalizado | `w_i_norm` | float | [0.0, 1.0] | `blend_weight_i / suma_total`; suma de todos = 1.0 |
| Color resultante | `C_tint_out` | Color RGBA | [0,0,0,0]–[1,1,1,1] | Tinte final aplicado al sprite de Edrick |

**Rango de salida:** siempre dentro del gamut de color por construcción. Sin riesgo de overflow.

**Guard contra división por cero**: si `Σ(blend_weight_j) = 0.0` (todos los pesos son cero), el sistema trata el tinte como null y no aplica ninguna modificación. Esta condición puede ocurrir con `blend_weight=0.0` configurado en el schema. La implementación debe verificar `Σ > 0.0` antes de la división.

**Por qué weighted average, no promedio aritmético**: Fuego (naranja) + Hielo (azul/blanco) con promedio aritmético produce gris neutro. El weighted average preserva la identidad del demonio con mayor `blend_weight`.

**Ejemplo:** Fuego `Color(1.0, 0.45, 0.0)` weight=0.25 + tinte rojo `Color(0.8, 0.1, 0.1)` weight=0.20 → `C_out ≈ Color(0.91, 0.30, 0.04)` — naranja-rojizo reconocible.

---

### F-TVE-03: Decay de Opacidad de Ghost Frames (`motion_trail`)

Cada ghost frame de la estela decae exponencialmente desde el más reciente al más antiguo:

```
opacity_j = opacity_peak × DECAY_RATE ^ j
```

**Variables:**
| Variable | Símbolo | Tipo | Rango | Descripción |
|----------|---------|------|-------|-------------|
| Opacidad del ghost más reciente | `opacity_peak` | float | [0.0, 1.0] | Campo `opacity_peak` del schema §3.2 por demonio |
| Factor de decaimiento global | `DECAY_RATE` | float | [0.3, 0.8] | Tuning knob del sistema. Menor = cola más corta/pronunciada |
| Índice del ghost frame | `j` | int | [0, trail_length-1] | 0 = más reciente, trail_length-1 = más antiguo |
| Longitud de la estela | `trail_length` | int | [1, 8] | Campo `trail_length` del schema §3.2 por demonio |
| Opacidad del ghost j | `opacity_j` | float | [0.0, 1.0] | Opacidad final aplicada al ghost frame j |

**Rango de salida:** (0, `opacity_peak`] — nunca supera `opacity_peak`.

**Ejemplo (Dash):** `opacity_peak=0.15`, `trail_length=3`, `DECAY_RATE=0.5`
- j=0: `0.15` (más reciente)
- j=1: `0.075`
- j=2: `0.038` (casi invisible)

Decay exponencial produce una estela con "cabeza" clara y cola difuminada — más natural que el decay lineal.

---

### F-TVE-04: Crossfade de Aura Post-Binding (`BINDING_AURA_TRANSITION`)

Transición suavizada (ease-out / smoothstep) entre la distorsión de GDD #13 y el aura permanente del demonio:

```
t_norm = clamp(t / BINDING_AURA_TRANSITION, 0.0, 1.0)
s = smoothstep(0.0, 1.0, t_norm)           -- 3t² - 2t³; nativo en Godot

opacity_gdd13(t) = 1.0 - s                 -- fade-out de la distorsión oscura
opacity_demon(t) = s                       -- fade-in del aura permanente
```

**Variables:**
| Variable | Símbolo | Tipo | Rango | Descripción |
|----------|---------|------|-------|-------------|
| Tiempo desde `demon_bound` | `t` | float | [0.0, `BINDING_AURA_TRANSITION`] | segundos |
| Duración del crossfade | `BINDING_AURA_TRANSITION` | float | [0.3, 0.8] s | MVP = 0.5s. Tuning knob. |
| Tiempo normalizado | `t_norm` | float | [0.0, 1.0] | `t / BINDING_AURA_TRANSITION` |
| Curva de ease | `s` | float | [0.0, 1.0] | `smoothstep(0, 1, t_norm)` |
| Opacidad distorsión GDD #13 | `opacity_gdd13(t)` | float | [0.0, 1.0] | 1.0 → 0.0 durante el crossfade |
| Opacidad aura permanente | `opacity_demon(t)` | float | [0.0, 1.0] | 0.0 → 1.0 durante el crossfade |

**Invariante:** `opacity_gdd13(t) + opacity_demon(t) = 1.0` en todo momento — la luminosidad total se conserva.

**Caso especial — demonio sin `aura_bg` (Dash):** `opacity_demon` llega a 1.0 pero la capa es null. La distorsión de GDD #13 hace fade-out a transparente sin reemplazarse. Sin lógica adicional.

---

### F-TVE-05: Blend de Color de `eye_overlay` (N≥3, future-proofing)

*Solo aplica cuando N≥3 demonios tienen `eye_overlay` activo. En MVP (máximo 2 slots) no es alcanzable.*

Blend en espacio HSV preservando saturación máxima:

```
w_i_norm = blend_weight_i / Σ(blend_weight_j)

H_out = circular_weighted_mean(H_i, w_i_norm)  -- media circular ponderada de Hues
S_out = max(S_i)                                -- saturación máxima (no promedio)
V_out = Σ(V_i × w_i_norm)                       -- valor/brillo promedio ponderado
C_eye_out = HSVtoRGB(H_out, S_out, V_out)
```

**Rango de salida:** Color RGB válido. `S_out = max(S_i)` garantiza que el resultado es siempre al menos tan saturado como el demonio más saturado — nunca degrada a gris.

**Nota de implementación (Godot 4.6.3):** `Color.to_hsv()` y `Color.from_hsv(h, s, v)` son nativos. La media circular del Hue usa `atan2(Σsin(H_i), Σcos(H_i))` — 6 líneas de GDScript.

---

## 5. Casos Extremos

> **⚠ Esta sección depende fuertemente de Art Direction.**
>
> El sistema de capas define *las reglas* de cómo se combinan los efectos visuales — pero la *viabilidad visual* de esas combinaciones (legibilidad, contraste, identidad cromática de cada demonio) solo puede validarse con el arte final. Varios de los casos extremos documentados abajo no son errores de código: son situaciones que Art Direction debe anticipar al diseñar los assets y los valores del schema (§3.2).
>
> **Antes de producir cualquier asset de transformación de demonio**, revisar esta sección con el/la art director/a y determinar qué pares de demonios son problemáticos visualmente. Los flags `[🎨 AD]` marcan los casos que requieren decisión de Arte antes de producción.

---

**E1 — Sin demonios equipados (estado IDLE)**
**Si** todos los slots están vacíos, **entonces** Edrick muestra el sprite base sin ninguna capa activa. Sin `aura_bg`, `sprite_tint`, `eye_overlay`, `floating_fx` ni `motion_trail`. Estado completamente válido.

**E2 — Solo Dash equipado**
**Si** Dash es el único demonio activo, **entonces** el único efecto visible es `motion_trail` durante el movimiento. Edrick se ve "base" en reposo. Esto es intencional — Dash es el estado visual silencioso.

**E3 — Dos demonios con `eye_overlay` del mismo color (heterocromia idéntica)** `[🎨 AD]`
**Si** dos demonios activos tienen `eye_overlay` con el mismo color o colores prácticamente idénticos, **entonces** ambos ojos toman ese color — no hay heterocromia perceptible. El código es correcto; el problema es de diseño de color.
> **Decisión de Arte requerida antes de producción**: asegurarse de que ningún par de demonios MVP con `eye_overlay` use colores idénticos o tan similares que la heterocromia sea imperceptible. Ejemplo problemático: Visión (incoloro) + cualquier demonio con color muy claro. Definir en la tabla de colores del art bible.

**E4 — Visión a N=2 (intensidad efectiva muy baja)** `[🎨 AD]`
**Si** Visión (`aura_bg intensity=0.3`) está equipado junto a otro demonio, la intensidad efectiva es `0.3 × 0.5 = 0.15` — potencialmente tenue hasta el límite de lo invisible. El sistema aplica el valor correctamente.
> **Decisión de Arte requerida antes de producción**: validar en pantalla si `intensity=0.15` para la distorsión de Visión es legible o desaparece. Si no es legible, ajustar `intensity` de Visión en GDD #3 o el `BLEND_SCALE` global (§7). No tocar el código — el problema se resuelve en los datos del schema.

**E5 — `demon_bound` recibido para un demonio ya en `equipped_demons`**
**Si** `demon_bound(demon_id)` llega con un ID ya activo (bug en GDD #13 o race condition), **entonces** el sistema ignora el evento. Guard: verificar `equipped_demons.has(demon_id)` antes de entrar a POST_BINDING. Sin decisión de Arte.

**E6 — `loadout_changed` recibido mientras POST_BINDING crossfade está en curso**
**Si** el jugador completa un swap mientras el crossfade de aura post-binding (0.5s) aún corre, **entonces** el sistema cancela el crossfade inmediatamente, elimina la distorsión de GDD #13 al 100%, y aplica el nuevo estado. El binding sigue registrado — solo se corta la transición de aura.

**E7 — Edrick muere y reaparece durante TRANSITION o POST_BINDING**
**Si** `edrick_died()` llega mientras hay una transición en curso, **entonces** el sistema cancela la transición y vuelve a IDLE. Al recibir `edrick_respawned()`, re-lee `equipped_demons` de Loadout y recalcula todas las capas desde cero.

**E8 — `floating_fx` con el mismo `orbit_radius` en dos demonios activos** `[🎨 AD]`
**Si** dos demonios activos tienen `floating_fx` con el mismo `orbit_radius`, **entonces** ambos sets de elementos orbitales se superponen en el mismo radio, produciendo densidad visual potencialmente confusa. El sistema renderiza ambos correctamente.
> **Decisión de Arte requerida antes de producción**: asignar `orbit_radius` distintos a todos los demonios con `floating_fx` en MVP. Separación mínima recomendada: 12px entre radios. Arcano (runas) e Hielo (cristales) son los únicos demonios MVP con `floating_fx` — validar que sus radios no colisionen. Ver §7.

**E9 — `motion_trail` durante SWAP_ANIM**
**Si** Edrick tiene estela activa cuando inicia un swap (queda inmóvil durante 0.8s), **entonces** la estela desaparece en `fade_time` (0.1s para Dash) al detenerse. No requiere lógica especial — el trigger es `speed < activation_speed`.

**E10 — `floating_fx` orbitan fuera de geometría visible** `[🎨 AD]`
**Si** los elementos flotantes cruzan geometría de nivel (paredes, techos), **entonces** se renderizan igualmente — son efectos de CanvasItem, no respetan física. Esto es comportamiento esperado.
> **Decisión de Arte / Level Design requerida**: si en algún nivel la geometría produce elementos flotantes mitad dentro de pared, ajustar `orbit_radius` como tuning knob por demonio. Flagear en la reunión de art+level design al construir los primeros niveles.

---

> **Resumen de flags para Art Direction** (para la reunión de diseño de assets):
> - E3: definir colores de `eye_overlay` no-idénticos para todos los demonios MVP con esta capa
> - E4: validar visibilidad de Visión (`intensity=0.15`) en pantalla real antes de fijar valores
> - E8: separar `orbit_radius` de Arcano e Hielo con mínimo 12px entre ellos
> - E10: coordinar con Level Design para `orbit_radius` seguros en geometría de niveles

---

## 6. Dependencias

### 6.1 Dependencias Salientes (este sistema necesita)

| Sistema | Tipo | Qué necesita | Nota |
|---------|------|-------------|------|
| **Base de Datos de Demonios (#3)** | Dura | Lee `transformacion_visual` por `demon_id`: los datos de las 5 capas del schema §3.2 para cada demonio activo. Sin estos datos, el sistema no puede calcular ninguna capa. | ⚠ GDD #3 debe retrofittarse para reemplazar `aura: string` por el schema estructurado de §3.2. Deuda aceptada — el campo `aura: string` actual se trata como `aura_bg.color` hasta la actualización. |
| **Loadout (#10)** | Dura | Recibe `loadout_changed(equipped_demons, gato_available)` del EventBus al finalizar `SWAP_ANIM_DURATION`. El array `equipped_demons` determina qué capas están activas. | Timing crítico: los cambios visuales aplican al recibir esta señal, no antes. |
| **Estado del Mundo (#4)** | Suave (MVP: sin efecto) | `corruption_level` — nivel de corrupción moral de Edrick para modular la apariencia base. En MVP el hook está definido pero sin implementación. | Activar al diseñar GDD #22 (Seguimiento Moral, Vertical Slice). |

### 6.2 Dependencias Entrantes (dependen de este sistema)

| Sistema | Qué espera |
|---------|-----------|
| **Build Management UI (#20)** | Probablemente muestra una previsualización del aspecto de Edrick según el loadout configurado. GDD #20 aún no está diseñado — este sistema debe exponer una API de consulta de capas activas para que la UI pueda renderizar una previsualización sin depender del sprite en escena. |

### 6.3 Interfaces de Señal

| Señal | Dirección | Qué hace este sistema |
|-------|-----------|----------------------|
| `loadout_swap_started(new_demons: Array[String])` | EventBus → VisualTransform | **SEÑAL NUEVA** (P-TVE-07). Emitida por GDD #10 al INICIO de SWAP_ANIM. Inicia el blend progresivo de 0.8s hacia las capas de `new_demons`. |
| `loadout_changed(equipped_demons, gato_available)` | EventBus → VisualTransform | Emitida por GDD #10 al FINAL de SWAP_ANIM. Bloquea las capas nuevas al 100% y descarta las anteriores. |
| `demon_bound(demon_id)` | EventBus → VisualTransform | Dispara POST_BINDING: crossfade F-TVE-04 desde distorsión de GDD #13 hacia aura permanente. **Contrato crítico**: el binding NO auto-equipa el demonio — `loadout_changed` NO se emite como consecuencia directa de `demon_bound`. POST_BINDING completa sin auto-cancelación bajo flujo normal. |
| `edrick_died()` | EventBus → VisualTransform | Cancela cualquier transición en curso (TRANSITION o POST_BINDING) y vuelve a IDLE. |
| `edrick_respawned()` | EventBus → VisualTransform | Re-lee `equipped_demons` de Loadout y recalcula todas las capas. **Pendiente P-TVE-06**: esta señal aún no está definida en GDD #2. |

### 6.4 Bidireccionalidad

- **VisualTransform → Loadout**: unidireccional. Loadout notifica; VisualTransform no escribe en Loadout.
- **VisualTransform → BD Demonios (#3)**: unidireccional. VisualTransform lee; BD no sabe qué se está renderizando.
- **VisualTransform ↔ Vinculación (#13)**: handoff vía `demon_bound`. GDD #13 no espera ninguna señal de vuelta — ownership transferido al recibir la señal. **⚠ Brecha de bidireccionalidad**: GDD #13 no lista actualmente a GDD #14 como sistema dependiente. Debe actualizarse en GDD #13 para documentar que `demon_bound` transfiere ownership del canal `aura_bg` a este sistema.
- **Build Management UI (#20) ↔ VisualTransform**: potencialmente bidireccional (UI consulta estado para previsualización). Contrato a definir en GDD #20.

### 6.5 Deuda de Diseño Aceptada

| Deuda | Impacto | Resolución |
|-------|---------|-----------|
| GDD #3 usa `aura: string` en lugar del schema §3.2 | Bajo — tratado como `aura_bg.color` hasta retrofit | Retrofittar GDD #3 antes del primer sprint de implementación de assets |
| Hook de `corruption_level` sin implementar en MVP | Ninguno en MVP | Activar al diseñar GDD #22 |
| API de previsualización para GDD #20 no especificada | Bajo — GDD #20 aún no diseñado | Definir en GDD #20 |

---

## 7. Parámetros de Ajuste

Todos los valores deben estar en configuración externa (archivo de datos o exports del nodo), no hardcodeados.

### 7.1 Parámetros Globales del Sistema

| Parámetro | Valor MVP | Rango Seguro | Qué rompe si... |
|-----------|-----------|-------------|-----------------|
| `BLEND_SCALE` | 1.0 | [0.5, 1.0] | **<0.5**: capas multi-demonio son casi invisibles. **>1.0**: no permitido (cap en 1.0). Para compensar capas tenues (ej. Visión en E4), ajustar `intensity_base` en demons.json por demonio en lugar de subir BLEND_SCALE globalmente. El valor 1.0 es puramente proporcional. |
| `MAX_BLEND_INTENSITY` | 0.9 | [0.5, 1.0] | **<0.5**: limita tanto las auras combinadas que se pierden visualmente. **>1.0** (sin límite): con dos demonios de alta intensidad puede producir auras que opacan el sprite base. Actúa como cap final tras aplicar F-TVE-01. |
| `DECAY_RATE` | 0.5 | [0.3, 0.8] | **<0.3**: estela muy corta, el efecto desaparece casi inmediatamente. **>0.8**: los ghost frames más antiguos son casi tan visibles como el más reciente — la estela parece "plana" en vez de disolverse. |
| `BINDING_AURA_TRANSITION` | 0.5 s | [0.3, 0.8] s | **<0.3s**: el crossfade es tan rápido que el jugador no lo percibe — la distorsión de GDD #13 desaparece abruptamente. **>0.8s**: el aura permanente tarda en asentarse, creando disonancia entre el gameplay reanudado y la visual aún en transición. |

### 7.2 Parámetros por Demonio (en `demons.json` — GDD #3)

Todos viven en el campo `transformacion_visual` del schema §3.2. Cambiarlos solo requiere editar el archivo de datos.

| Campo | Rango Seguro | Qué rompe si... |
|-------|-------------|-----------------|
| `aura_bg.intensity` | [0.0, 1.0] | **<0.1**: aura imperceptible incluso en SINGLE_DEMON. **>0.9** con BLEND_SCALE>1: puede opacar el sprite en multi-demonio. |
| `sprite_tint.blend_weight` | [0.0, 0.4] | **>0.4**: el tinte empieza a "lavar" el sprite base, perdiendo detalles del personaje. **0.0**: tinte invisible (usar null si no se quiere tinte). |
| `floating_fx.count` | [1, 12] | **>12**: demasiados nodos simultáneos; riesgo de rendimiento combinado con otros efectos. **<1**: usar null si no se quieren flotantes. |
| `floating_fx.orbit_radius` | [16, 64] px | **<16px**: los flotantes se superponen con el sprite de Edrick (ilegibles). **>64px**: sobresalen demasiado del personaje, contaminan el fondo. Separación mínima entre dos demonios: 12px (ver E8). |
| `floating_fx.speed` | [0.3, 2.0] rad/s | **<0.3**: los flotantes parecen casi estáticos. **>2.0**: el efecto es mareante y distrae del gameplay. |
| `motion_trail.opacity_peak` | [0.05, 0.4] | **<0.05**: estela invisible. **>0.4**: estela demasiado llamativa — compite con `aura_bg` activos. El objetivo de Dash es discreción. |
| `motion_trail.trail_length` | [1, 8] | **<1**: inútil (usar null). **>8**: demasiados sprites simultáneos de estela. |
| `motion_trail.activation_speed` | [30, 150] px/s | **<30**: la estela aparece hasta con micro-movimientos. **>150**: la estela solo aparece al esprintar — se pierde el feedback de velocidad base. |
| `motion_trail.fade_time` | [0.05, 0.3] s | **<0.05**: la estela desaparece abruptamente al detenerse. **>0.3**: la estela "flota" visible mucho después de que Edrick se detiene. |

### 7.3 Nota de Validación

> **🎨 Art Direction**: Los valores por demonio de la tabla §3.2 son provisionales. Deben ser validados y ajustados por Art Direction con el arte final en pantalla antes de fijarlos en `demons.json`. Los rangos seguros de esta tabla son el contrato de diseño — Art Direction puede elegir cualquier valor dentro del rango sin cambiar código.

---

## 8. Requisitos Visuales y de Audio

> **`art-director` no consultado — Lean mode.**
>
> **⚠ Esta sección define los requisitos técnicos de assets. Los valores cromáticos, el estilo exacto de cada capa y los assets finales DEBEN ser validados por Art Direction antes de producción.**
>
> **📌 Asset Spec**: Una vez aprobado el art bible, ejecutar `/asset-spec system:transformacion-visual-edrick` para producir descripciones de assets, dimensiones y prompts de generación desde esta sección.

---

### 8.1 Base — Sprite de Edrick

El sprite base de Edrick es la capa 0 sobre la que se superponen todas las transformaciones. Todos los efectos visuales de este sistema son aditivos o sobreimpuestos; el sprite base no se modifica.

**Requisito para Art Direction**: el sprite base debe diseñarse con un canal alpha limpio en todas las zonas que puedan ser afectadas por `sprite_tint` y `eye_overlay`. Una zona de ojos con brillo precalculado en el sprite dificultará la superposición del `eye_overlay`.

---

### 8.2 Capa `aura_bg` — Aura Ambiente

**Técnica recomendada**: El campo `style` requiere tres implementaciones técnicas distintas — no es un parámetro de slider:
- `"glow"` → ShaderMaterial en CanvasItem con parámetro de color e intensidad
- `"distortion"` → shader de refracción UV con BackBufferCopy o equivalente en Godot 4.6.3 (⚠ validar render order con `godot-specialist` — ha cambiado entre versiones 4.x)
- `"particles"` → GPUParticles2D con material propio y presupuesto de rendimiento separado

Estos son tres sistemas de implementación independientes con presupuestos de performance distintos. Delegar especificación técnica a `technical-artist` antes de la implementación. Implementado como nodo hijo del sprite de Edrick.

| Demonio | Color | Style | Descripción visual objetivo |
|---------|-------|-------|----------------------------|
| Fuego | naranja/rojo `#FF6A00` (ref) | glow | Brillo suave naranja-rojizo alrededor del sprite. Caliente, no agresivo. |
| Hielo | azul/blanco `#B0D8FF` (ref) | particles | Pequeñas partículas frías orbitando el sprite. Quieto, distante. |
| Arcano | púrpura/dorado `#8B31C7` (ref) | glow | Brillo púrpura profundo con toque dorado. Poderoso, sobrenatural. |
| Visión | incoloro `#FFFFFF` (ref) | distortion | Distorsión sutil del fondo alrededor de Edrick, casi imperceptible. |
| Mente | platino/blanco `#E8E8FF` (ref) | glow | Aura blanca-azulada muy suave. Serena, concentrada. |

*Los colores en hex son referencias de partida para Art Direction — no son valores finales.*

> **🎨 Decisión de Arte requerida**: validar que las 5 auras son distinguibles entre sí en el contexto visual del juego (sobre fondos oscuros y claros de los distintos reinos). Proporcionar paleta aprobada antes de implementar los shaders.

---

### 8.3 Capa `sprite_tint` — Tinte de Sprite

**Técnica recomendada**: ShaderMaterial personalizado con blend mode Screen o Soft Light a bajo peso. **`modulate` de Godot no es válido para este uso** — es multiplicativo (`final = pixel × modulate_color`) y desplaza el matiz en zonas oscuras, destruyendo la estructura de sombras del pixel art. Sin nuevos assets; opera sobre el sprite base existente. Delegar especificación del blend mode exacto a `technical-artist`.

| Demonio | Color de tinte | Blend weight | Efecto objetivo |
|---------|---------------|--------------|-----------------|
| Fuego | naranja `#FF7F2A` (ref) | 0.25 | Tonos ígneos sutiles sobre piel y ropa. No enmascarar los detalles del sprite. |
| Dash | gris-blanco frío `#D8E4F0` (ref) | 0.08 | Matiz frío casi imperceptible. Distingue el estado Dash-equipado del estado IDLE sin proclamarlo. Resting indicator del primer binding del Acto 1. |

> **🎨 Decisión de Arte requerida**: confirmar `blend_weight` óptimo para Fuego y Dash con el sprite base final. El rango seguro es [0.0, 0.4]. Ver E4 y §7.

---

### 8.4 Capa `eye_overlay` — Overlay de Ojos

**Técnica recomendada**: sprite overlay separado (o zona UV maskeada del sprite de Edrick) con un shader de brillo/glow independiente. Requiere identificar la región de los ojos en el sprite.

| Demonio | Color | Pulse | Style | Efecto objetivo |
|---------|-------|-------|-------|-----------------|
| Visión | incoloro/blanco lechoso | false | void | Ojos sin pupila visible, reflejo fantasmal. Perturbador, no brillante. |
| Mente | platino/blanco | true | shine | Ojos que brillan con concentración intensa. Pulso lento (≈ 1 Hz). |

**Heterocromia (2 demonios activos con eye_overlay):**
> **🎨 Decisión de Arte requerida**: el sprite de Edrick debe tener los ojos izquierdo y derecho como elementos separados (o la máscara de eye_overlay debe poder aplicarse por ojo individualmente) para que la heterocromia sea implementable. Si el sprite trata los ojos como una sola región, la heterocromia requiere rediseño del sprite. Definir **antes de encargar el sprite base**.

---

### 8.5 Capa `floating_fx` — Efectos Flotantes

**Técnica recomendada**: N nodos `Node2D` sprites con `Tween` de posición angular (rotación circular). Un único `ShaderMaterial` compartido entre todos los nodos del mismo demonio. Patrón establecido en GDD #13.

| Demonio | Tipo | Asset requerido | Descripción objetivo |
|---------|------|-----------------|----------------------|
| Arcano | runes | Sprite individual de runa arcana (ref: 8×8 px a 16×16 px) | Runas doradas/púrpuras que orbitan lentamente a Edrick. Legibles individualmente como símbolos. |
| Hielo | crystals | Sprite de fragmento de cristal de hielo (ref: 6×8 px a 10×12 px) | Seis fragmentos de hielo orbitando en blanco-azul. Simples, angulares. |

> **🎨 Decisión de Arte requerida**: definir el tamaño en píxeles de cada asset de `floating_fx` coherente con la escala base del sprite de Edrick (16–32 px). Sprites demasiado grandes reducen la legibilidad del personaje. Validar `orbit_radius` con los assets finales.

---

### 8.6 Capa `motion_trail` — Estela de Movimiento

**Técnica recomendada**: snapshots del sprite de Edrick en posiciones anteriores (ghost frames) con opacidad decreciente (F-TVE-03). Sin nuevos assets — reutiliza el sprite base de Edrick. Implementado como instancias duplicadas del sprite con el ShaderMaterial de opacity aplicado.

| Demonio | Descripción objetivo |
|---------|----------------------|
| Dash | Estela gris/blanco muy sutil. Solo visible en movimiento. 3 ghost frames. No compite visualmente con ninguna capa de aura activa. Comunica agilidad, no poder bruto. |

> La estela de Dash es la única en MVP, pero el mecanismo de ghost frames reutiliza el sprite base de Edrick — no requiere assets adicionales por demonio en el futuro.

---

### 8.7 Audio

Este sistema es visualmente-only. No tiene eventos de audio propios.

- La animación de swap (SWAP_ANIM 0.8s) y su audio pertenecen a GDD #10
- El audio de la secuencia de binding pertenece a GDD #13
- El crossfade de aura post-binding (F-TVE-04, 0.5s) es silencioso

Sin requisitos de audio para este GDD.

---

### 8.8 Checklist de Validación para Art Direction

Antes de producir assets:
- [ ] Paleta de colores aprobada para las 5 auras (§8.2)
- [ ] Sprite base con canal alpha limpio y ojos separables como región independiente (§8.4)
- [ ] Tamaño en píxeles de assets `floating_fx` coherente con escala del sprite (§8.5)
- [ ] Radios de `orbit_radius` definidos y no solapados (mínimo 12px entre Arcano e Hielo) (E8)
- [ ] Colores de `eye_overlay` no idénticos para Visión y Mente (E3)
- [ ] Visión con `intensity=0.15` en pantalla real es legible (E4)

---

## 9. Criterios de Aceptación

> **Revisado por especialistas completo (2026-05-31). Conteo actualizado post-revisión: 40 ACs.**
>
> **Flag de implementación**: AC-TVE-029 — la cancelación del crossfade debe implementarse vía `Tween.kill()` + flag `_crossfade_active` (Opción C — mandatado por revisión). Ver P-TVE-01 en §10.
>
> **Flag de Art Direction**: Las ACs visuales (033–036) no pueden marcarse Done hasta que Art Direction haya aprobado la paleta de §8.2. AC-TVE-033 **no** es parte del gate de QA hand-off.
>
> **Nota de tests unitarios**: Los ACs de timing (TVE-001, TVE-004) requieren mock timer injection de `SWAP_ANIM_DURATION`. Los tests NO deben usar wall-clock waits — viola la regla de determinismo.

---

### Grupo A — Timing y Control de Estado

**AC-TVE-001** — Capas no cambian antes de finalizar SWAP_ANIM_DURATION
**GIVEN** Edrick tiene Fuego equipado (aura_bg naranja activa), **WHEN** `loadout_changed` es recibido y han transcurrido menos de 0.8 s, **THEN** los parámetros del shader no cambian hasta que se cumplen los 0.8 s completos.
**Tipo**: Unit | **Bloquea**: Sí

**AC-TVE-002** — Las capas se actualizan al finalizar SWAP_ANIM_DURATION
**GIVEN** Edrick en SINGLE_DEMON con Fuego equipado, **WHEN** `loadout_changed([Arcano], false)` es recibido y transcurren 0.8 s, **THEN** aura_bg cambia a púrpura/dorado de Arcano, sprite_tint se elimina, floating_fx runas se activa; los parámetros anteriores no son visibles.
**Tipo**: Unit | **Bloquea**: Sí

**AC-TVE-003** — `loadout_changed` con `equipped_demons` vacío transiciona a IDLE
**GIVEN** Edrick tiene Fuego equipado, **WHEN** `loadout_changed([], false)` llega y pasan 0.8 s, **THEN** todas las capas son null; el estado es IDLE.
**Tipo**: Unit | **Bloquea**: Sí

**AC-TVE-004** — Durante TRANSITION las capas hacen blend progresivo lineal
**GIVEN** `SWAP_ANIM_DURATION` inyectado via mock (no wall-clock), Edrick en SINGLE_DEMON con Hielo equipado, **WHEN** `loadout_swap_started([Arcano])` se recibe, **THEN** en t=0.4s (mitad del blend) la intensidad efectiva de las capas de Arcano ≈ 50% de su valor final y la de Hielo ≈ 50% de su valor original (±5%); el sistema reporta estado TRANSITION.
**Tipo**: Unit | **Bloquea**: Sí

---

### Grupo B — Capas Individuales

**AC-TVE-005** — `aura_bg` se aplica con los parámetros correctos para demonio único
**GIVEN** Edrick en IDLE, **WHEN** `loadout_changed([Fuego], false)` llega y pasan 0.8 s, **THEN** el nodo `aura_bg` tiene color naranja (`#FF6A00` ref), intensity=0.7, style="glow". Art Direction firma la captura.
**Tipo**: Visual | **Bloquea**: No

**AC-TVE-006** — `sprite_tint` = null no produce modificación en el shader
**GIVEN** Arcano equipado (sprite_tint=null), **WHEN** se comprueba el parámetro de tinte, **THEN** `blend_weight=0.0` o la capa está desactivada; el sprite base no varía de color.
**Tipo**: Unit | **Bloquea**: Sí

**AC-TVE-007** — `eye_overlay` con N=1: ambos ojos toman el color del demonio al 100%
**GIVEN** Mente equipado (eye_overlay platino, pulse=true), **WHEN** se consultan los parámetros del ojo izquierdo y derecho, **THEN** ambos tienen color=platino y pulse=true; son idénticos entre sí.
**Tipo**: Unit | **Bloquea**: Sí

**AC-TVE-008** — `floating_fx`: count y orbit_radius correctos para demonio único
**GIVEN** Arcano equipado (floating_fx: runes, count=4), **WHEN** el sistema renderiza SINGLE_DEMON, **THEN** existen exactamente 4 nodos Node2D de runa activos, todos a distancia `orbit_radius` del centro de Edrick (±1 px), rotando a `speed`.
**Tipo**: Integration | **Bloquea**: Sí

**AC-TVE-009** — `motion_trail` aparece solo cuando speed > `activation_speed` (50 px/s para Dash)
**GIVEN** Dash equipado, **WHEN** Edrick se mueve a 49 px/s, **THEN** no hay ghost frames; **WHEN** se mueve a 51 px/s, **THEN** exactamente 3 ghost frames con opacidades j=0:0.150, j=1:0.075, j=2:0.038.
**Tipo**: Unit | **Bloquea**: Sí

**AC-TVE-010** — `motion_trail` desaparece al detenerse dentro de `fade_time` (0.1 s)
**GIVEN** Dash equipado con ghost frames activos (speed > 50 px/s), **WHEN** Edrick se detiene (speed=0), **THEN** todos los ghost frames desaparecen completamente en ≤ 0.1 s.
**Tipo**: Unit | **Bloquea**: Sí

**AC-TVE-011** — Gato no aporta ninguna capa al sprite de Edrick
**GIVEN** `equipped_demons` contiene solo el Gato, **WHEN** `loadout_changed([Gato], true)` llega y pasan 0.8 s, **THEN** todas las capas son null; `gato_available` es ignorado.
**Tipo**: Unit | **Bloquea**: Sí

---

### Grupo C — Fórmulas

**AC-TVE-012** — F-TVE-01: Intensidad efectiva con N=2 calcula correctamente
**GIVEN** Fuego (intensity=0.7) + Arcano (intensity=0.8), BLEND_SCALE=1.0, N=2, **WHEN** el sistema calcula intensidades efectivas, **THEN** Fuego=0.350 (±0.001), Arcano=0.400 (±0.001); ninguno supera su `intensity_base`.
**Tipo**: Unit | **Bloquea**: Sí

**AC-TVE-013** — F-TVE-01: La intensidad se cappa en MAX_BLEND_INTENSITY (0.9) cuando el resultado pre-cap supera 0.9
**GIVEN** un demonio hipotético con intensity_base=1.0 equipado en SINGLE_DEMON (N=1, BLEND_SCALE=1.0), **WHEN** el sistema calcula la intensidad efectiva, **THEN** el valor pre-cap es 1.0 × (1/1) × 1.0 = 1.0; el valor aplicado al shader es ≤ 0.9 (capado por MAX_BLEND_INTENSITY).
**Tipo**: Unit | **Bloquea**: Sí

**AC-TVE-014** — F-TVE-02: Blend de sprite_tint produce el weighted average correcto
**GIVEN** Fuego (Color(1.0,0.45,0.0), weight=0.25) + segundo demonio (Color(0.8,0.1,0.1), weight=0.20) equipados, **WHEN** se calcula C_tint_out, **THEN** el resultado es ≈Color(0.91, 0.30, 0.04) (±0.01 por canal).
**Tipo**: Unit | **Bloquea**: Sí

**AC-TVE-015** — F-TVE-03: Opacidades de ghost frames siguen decay exponencial con DECAY_RATE=0.5
**GIVEN** Dash equipado (opacity_peak=0.15, trail_length=3, DECAY_RATE=0.5), **WHEN** se leen las opacidades de los 3 ghosts en movimiento, **THEN** j=0: 0.150 (±0.001), j=1: 0.075 (±0.001), j=2: 0.0375 (±0.001).
**Tipo**: Unit | **Bloquea**: Sí

**AC-TVE-016** — F-TVE-04: Invariante `opacity_gdd13 + opacity_demon = 1.0` durante crossfade
**GIVEN** `demon_bound(Fuego)` recibido (BINDING_AURA_TRANSITION=0.5 s), **WHEN** se muestrean ambas opacidades en t=0.0, 0.1, 0.25, 0.4, 0.5 s, **THEN** en cada muestra la suma es 1.0 (±0.001); t=0.0: (1.0, 0.0), t=0.5: (0.0, 1.0).
**Tipo**: Unit | **Bloquea**: Sí

**AC-TVE-017** — F-TVE-04: La curva de crossfade es smoothstep, no lineal
**GIVEN** `demon_bound` recibido con BINDING_AURA_TRANSITION=0.5 s, **WHEN** se mide opacity_demon en t=0.1 s (t_norm=0.2), **THEN** el valor es ≈0.104 (±0.01) — diferente al 0.2 que produciría interpolación lineal.
**Tipo**: Unit | **Bloquea**: Sí

**AC-TVE-018** — F-TVE-05: Blend HSV de eye_overlay preserva S=max(S_i) `[DEFERRED — N≥3 no alcanzable en MVP]`
**GIVEN** tres demonios hipotéticos con eye_overlay activo (S1=0.8, S2=0.5, S3=0.3), **WHEN** el sistema calcula el color resultante en HSV, **THEN** S_out=0.8=max(S_i); el resultado no es menos saturado que el demonio más saturado.
**Tipo**: Unit | **Bloquea**: **No** — DEFERRED. El test GUT debe existir como `pending` en MVP con condición de activación: `# Activar cuando slot_count >= 3`. Un AC Blocking que no puede verificarse en MVP no puede formar parte del gate de Done.

---

### Grupo D — Combinación Multi-Demonio

**AC-TVE-019** — R1: Dos demonios con capas distintas muestran ambas simultáneamente
**GIVEN** Fuego (aura_bg naranja + sprite_tint) + Arcano (aura_bg púrpura + floating_fx runas) equipados, **WHEN** el sistema renderiza MULTI_DEMON, **THEN** el aura de Fuego y el aura de Arcano son visibles simultáneamente; las runas y el tinte naranja coexisten.
**Tipo**: Visual | **Bloquea**: No

**AC-TVE-020** — `floating_fx` de dos demonios coexiste sin cancelarse mutuamente
**GIVEN** Arcano (runes, count=4) + Hielo (crystals, count=6) equipados, **WHEN** el sistema renderiza MULTI_DEMON, **THEN** existen exactamente 10 elementos flotantes activos (4 runas + 6 cristales), cada set con su `orbit_radius` propio.
**Tipo**: Integration | **Bloquea**: Sí

**AC-TVE-021** — R5: N=2 eye_overlay → heterocromia: slot 1 = ojo izquierdo, slot 2 = ojo derecho
**GIVEN** Visión en slot 1 (incoloro) y Mente en slot 2 (platino), **WHEN** el sistema calcula MULTI_DEMON con N_eye=2, **THEN** ojo izquierdo = incoloro/void, ojo derecho = platino/shine+pulse; los ojos no son idénticos.
**Tipo**: Unit | **Bloquea**: Sí

**AC-TVE-022** — R7: Dash no aporta aura_bg, sprite_tint ni eye_overlay en ninguna condición
**GIVEN** Dash es el único demonio equipado, **WHEN** el sistema evalúa todas las capas al finalizar SWAP_ANIM, **THEN** aura_bg=null, sprite_tint=null, eye_overlay=null, floating_fx=null; solo motion_trail tiene parámetros activos.
**Tipo**: Unit | **Bloquea**: Sí

---

### Grupo E — POST_BINDING

**AC-TVE-023** — POST_BINDING se dispara por `demon_bound`, no por `loadout_changed`
**GIVEN** Edrick en IDLE, **WHEN** EventBus emite `demon_bound(Fuego)` sin `loadout_changed` previo, **THEN** el sistema entra en POST_BINDING y comienza el crossfade F-TVE-04 hacia las capas de Fuego.
**Tipo**: Integration | **Bloquea**: Sí

**AC-TVE-024** — POST_BINDING con demonio sin aura_bg: distorsión GDD #13 hace fade-out a transparente
**GIVEN** distorsión oscura de GDD #13 activa (opacity_gdd13=1.0), **WHEN** EventBus emite `demon_bound(Dash)`, **THEN** crossfade de 0.5 s: opacity_gdd13 va de 1.0 a 0.0; opacity_demon permanece en 0.0; al completar, no hay ninguna aura activa.
**Tipo**: Unit | **Bloquea**: Sí

**AC-TVE-025** — POST_BINDING completa en exactamente BINDING_AURA_TRANSITION (0.5 s)
**GIVEN** `demon_bound(Arcano)` recibido, **WHEN** transcurren 0.5 s, **THEN** opacity_gdd13=0.0, opacity_demon=1.0; el sistema transiciona a SINGLE_DEMON o MULTI_DEMON.
**Tipo**: Unit | **Bloquea**: Sí

---

### Grupo F — Casos Extremos

**AC-TVE-026** — E1: Estado IDLE — sprite base sin capas activas
**GIVEN** Edrick sin ningún demonio equipado, **WHEN** se inspeccionan los parámetros del sprite, **THEN** todas las capas son null o cero; el sprite base se renderiza sin modificación de shader.
**Tipo**: Unit | **Bloquea**: Sí

**AC-TVE-027** — E2: Solo Dash — aspecto base en reposo, estela solo en movimiento
**GIVEN** Dash es el único demonio, **WHEN** Edrick está en reposo (speed=0), **THEN** sin aura, tinte ni overlay; **WHEN** Edrick se mueve > 50 px/s, **THEN** exactamente 3 ghost frames con F-TVE-03.
**Tipo**: Unit | **Bloquea**: Sí

**AC-TVE-028** — E5: `demon_bound` duplicado para demonio ya en `equipped_demons` es ignorado
**GIVEN** Fuego en `equipped_demons` (SINGLE_DEMON), **WHEN** EventBus emite `demon_bound(Fuego)` de nuevo, **THEN** el sistema no entra en POST_BINDING, no reinicia ningún crossfade, el estado permanece intacto.
**Tipo**: Unit | **Bloquea**: Sí

**AC-TVE-029** — E6: `loadout_changed` durante POST_BINDING cancela el crossfade
**GIVEN** `demon_bound(Fuego)` recibido y crossfade en curso en t=0.2 s, **WHEN** EventBus emite `loadout_changed([Arcano], false)`, **THEN** el crossfade cancela inmediatamente (opacity_gdd13=0.0) y tras 0.8 s se aplica el estado de Arcano; el binding de Fuego no se deshace.
**Tipo**: Integration | **Bloquea**: Sí
> ⚠ *Nota de implementación*: especificar en §3.4 si la cancelación es vía reset de Tween, flag `_crossfade_active`, o ambos, antes de abrir la story.

**AC-TVE-030** — E7: `edrick_died()` durante TRANSITION cancela la transición y pasa a IDLE
**GIVEN** SWAP_ANIM en curso (t=0.4 s de 0.8 s), **WHEN** EventBus emite `edrick_died()`, **THEN** el sistema pasa a IDLE inmediatamente: todas las capas = null.
**Tipo**: Unit | **Bloquea**: Sí

**AC-TVE-031** — E7: `edrick_respawned()` resincroniza capas desde Loadout
**GIVEN** Edrick ha muerto (sistema en IDLE) y Loadout reporta `equipped_demons=[Hielo]`, **WHEN** EventBus emite `edrick_respawned()`, **THEN** el sistema activa las capas de Hielo sin requerir otro `loadout_changed`.
**Tipo**: Integration | **Bloquea**: Sí

**AC-TVE-032** — E9: motion_trail desaparece durante SWAP_ANIM (Edrick inmóvil)
**GIVEN** Dash equipado con ghost frames activos, **WHEN** SWAP_ANIM inicia (speed=0 < 50 px/s), **THEN** los ghost frames desaparecen en ≤ 0.1 s; no se requiere lógica adicional.
**Tipo**: Unit | **Bloquea**: Sí

---

### Grupo G — Visual / Feel

> **Dependencia Art Direction**: estos ACs no pueden marcarse Done hasta que Art Direction haya aprobado la paleta de §8.2. Evidencia en `production/qa/evidence/`.

**AC-TVE-033** — Las 5 auras MVP son distinguibles entre sí
**GIVEN** cada demonio con aura_bg equipado en sesiones separadas, **WHEN** se captura el sprite sobre fondos representativos de los reinos, **THEN** cada aura es cromáticamente distinguible de las otras 4 sin UI de ayuda; Art Direction firma.
**Tipo**: Visual | **Bloquea**: No

**AC-TVE-034** — Pares MVP críticos con aura_bg combinada son legibles
**GIVEN** pares Fuego+Arcano, Fuego+Hielo, Hielo+Arcano equipados simultáneamente, **WHEN** se captura MULTI_DEMON en reposo, **THEN** se identifica la contribución de cada demonio por separado; Art Direction firma cada par.
**Tipo**: Visual | **Bloquea**: No

**AC-TVE-035** — Heterocromia Visión+Mente es visualmente perceptible
**GIVEN** Visión en slot 1 + Mente en slot 2, **WHEN** se captura el sprite en close-up, **THEN** el ojo izquierdo y el ojo derecho son cromáticamente distintos de forma perceptible; Art Direction firma.
**Tipo**: Visual | **Bloquea**: No

**AC-TVE-036** — El crossfade POST_BINDING no produce frames abruptos
**GIVEN** binding de demonio con aura_bg ejecutado, **WHEN** un QA tester observa el crossfade de 0.5 s, **THEN** la transición es continua sin flash ni corte; evidencia en `production/qa/evidence/TVE-postbinding-crossfade.mp4`.
**Tipo**: Visual | **Bloquea**: No

---

### Grupo H — Transiciones de Estado (añadidos post-revisión)

**AC-TVE-037** — Transición SINGLE_DEMON → MULTI_DEMON aplica normalización
**GIVEN** Fuego equipado (SINGLE_DEMON, aura_bg intensity=0.7), **WHEN** `loadout_swap_started([Fuego, Arcano])` se recibe y transcurren 0.8s, **THEN** estado == MULTI_DEMON y la intensidad efectiva de Fuego == 0.35 (normalizada con N=2, ±0.001).
**Tipo**: Unit | **Bloquea**: Sí

**AC-TVE-038** — Transición MULTI_DEMON → SINGLE_DEMON elimina normalización
**GIVEN** Fuego+Arcano equipados (MULTI_DEMON, Fuego intensity_effective=0.35), **WHEN** `loadout_swap_started([Fuego])` se recibe y transcurren 0.8s, **THEN** estado == SINGLE_DEMON y la intensidad efectiva de Fuego == 0.7 (al 100%, sin normalización, ±0.001).
**Tipo**: Unit | **Bloquea**: Sí

**AC-TVE-039** — POST_BINDING iniciado desde MULTI_DEMON transiciona a estado correcto
**GIVEN** Fuego equipado (SINGLE_DEMON), **WHEN** EventBus emite `demon_bound(Arcano)` (binding, NO equip automático), **THEN** el sistema entra en POST_BINDING y al completar el crossfade transiciona a SINGLE_DEMON (Arcano vinculado pero no equipado — sin `loadout_changed`).
**Tipo**: Integration | **Bloquea**: Sí

**AC-TVE-040** — `edrick_died()` durante POST_BINDING cancela el crossfade y va a IDLE
**GIVEN** `demon_bound(Fuego)` recibido y crossfade en curso en t=0.2s, **WHEN** EventBus emite `edrick_died()`, **THEN** el crossfade cancela inmediatamente (`Tween.kill()` + flag), opacity_gdd13=0.0, todas las capas=null, estado==IDLE.
**Tipo**: Unit | **Bloquea**: Sí

---

**Resumen**: 40 ACs — 33 BLOCKING, 1 DEFERRED (TVE-018), 6 ADVISORY.
**Gate mínimo para hand-off a QA**: AC-TVE-001, AC-TVE-002, AC-TVE-003 (timing/blend-in), AC-TVE-026 (estado IDLE correcto), AC-TVE-006 (null-safety sprite_tint), AC-TVE-011 (Gato null). Nota: AC-TVE-033 (Visual/Advisory) es sign-off de Art Direction — no es un gate de QA hand-off.

---

## 10. Preguntas Abiertas

**P-TVE-01 — Mecanismo de cancelación del crossfade POST_BINDING**
*Urgencia: Antes de implementación* — **DECIDIDO POST-REVISIÓN: Opción C (ambos)**
La Opción B (flag-only) tiene una ventana de un frame de escritura stale. La Opción A (`Tween.kill()` solo) no limpia el flag. El ADR DEBE mandatar: (1) `Tween.kill()` para prevenir pasos del Tween en el mismo frame, más (2) `_crossfade_active = false` antes del kill para prevenir callbacks en vuelo. Documentar en ADR de implementación.

**P-TVE-02 — Sprite base de Edrick con ojos separables**
*Urgencia: Antes de producción del sprite base*
La heterocromia (AC-TVE-021) requiere que el ojo izquierdo y el ojo derecho sean regiones separadas o independientemente enmascarables. Si el sprite base trata los ojos como una sola región, la heterocromia requiere rediseño del sprite. Decidir con Art Direction **antes de encargar el sprite base** — cambiar la estructura después tiene coste alto.

**P-TVE-03 — API de previsualización para Build Management UI (#20)**
*Urgencia: Antes de diseñar GDD #20*
GDD #20 probablemente necesita mostrar una previsualización del aspecto de Edrick según el loadout configurado. Este sistema debe exponer una API de consulta de capas activas para que la UI pueda renderizar una previsualización sin depender del sprite en escena. El contrato exacto se define en GDD #20.

**P-TVE-04 — Retrofit de GDD #3 con el nuevo schema**
*Urgencia: Antes del primer sprint de implementación*
GDD #3 actualmente usa `aura: string` para el campo `transformacion_visual`. Debe retrofittarse para usar el schema estructurado de §3.2. No bloquea el diseño del GDD, pero bloquea la implementación.

**P-TVE-05 — Valores exactos del mapping de demonios en §3.2**
*Urgencia: Antes de producción de assets*
Todos los valores de color, intensidad, orbit_radius y parámetros de capas en la tabla §3.2 son provisionales. Art Direction debe revisar y aprobar todos los valores antes de que pasen a `demons.json`. Ver §8.8 para el checklist completo.

**P-TVE-06 — Señal `edrick_respawned()` no está definida en GDD #2**
*Urgencia: Antes del primer sprint de implementación*
entities.yaml confirma que `edrick_respawned()` aún no existe en GDD #2 (Salud y Daño). AC-TVE-031 y AC-TVE-030 dependen de esta señal — son BLOCKING y no pueden verificarse hasta que GDD #2 la añada. Este sistema no puede abrirse como story hasta que GDD #2 esté actualizado.

**P-TVE-07 — Señal `loadout_swap_started(new_demons)` es nueva y requiere actualización en GDD #10**
*Urgencia: Antes del primer sprint de implementación*
El modelo de blend progresivo (§3.1.D) requiere que Loadout emita `loadout_swap_started(new_demons: Array[String])` al INICIO de SWAP_ANIM. Esta señal no existe en GDD #10 actualmente. GDD #10 debe añadirla antes de que las stories de este sistema se abran.
