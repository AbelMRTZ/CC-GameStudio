# HUD de Combate

> **Status**: In Revision — post /design-review 2026-06-05
> **Author**: Abel Martínez + CC Game Studio Agents
> **Last Updated**: 2026-06-05
> **Implementa**: Pilar 2 — Demonios como Poder Transformador · **Sirve**: Pilar 4 (estado del loadout legible) · Pilar 5 — Edrick al Límite
> **GDD**: #18 | **Depende de**: Combate (#6), Salud/Daño (#2), Loadout (#10)

## Overview

El HUD de Combate es la capa de información en pantalla que convierte el estado mecánico de Edrick en decisiones visibles durante el combate. Técnicamente, es un `CanvasLayer` reactivo que se suscribe exclusivamente a señales del EventBus (ADR-002) — `health_changed` desde Salud y Daño, `cooldown_changed`/`combat_started`/`combat_ended` desde Combate, y `loadout_changed` desde Loadout — actualizando sus elementos visuales sin mantener referencias directas a otros sistemas. Desde la perspectiva del jugador, el HUD es el panel de mando del combate: la barra de HP comunica cuánto riesgo puede absorber Edrick antes de morir (y su caída genera tensión real), los indicadores de slot de demonio dictan la cadencia táctica — qué habilidad usar, cuándo esperar el cooldown, cuándo el slot está vacío —, y el estado del loadout confirma de un vistazo qué demonios están activos. Sin el HUD, el jugador pelea a ciegas: no sabe cuándo puede usar una habilidad, si debe retirarse, ni qué demonio tiene equipado. El sistema existe para hacer visible lo invisible: el estado mecánico que el combate genera en cada frame, expresado en una forma que se siente como instinto de batalla más que interfaz de usuario.

## Player Fantasy

El jugador debe sentir que el HUD es una extensión de su consciencia de combate, no una pantalla que consultar.

**Tensión escalada, no ansiedad genérica.** Cuando la barra de HP cae por debajo del umbral crítico, el HUD no solo informa — *amplifica*. El cambio visual llega antes de que el jugador procese el número: entiende instintivamente que está al borde. Esta tensión es el Pilar 5 expresado en la UI — Edrick está a punto de quebrarse, y el jugador lo siente en sus propias manos. La fantasía no es "saber cuántos HP tengo"; es "saber cuánto peligro tengo sin apartar los ojos del combate".

**El ritmo de los demonios, leído de un vistazo.** El jugador aprende el pulso de su loadout: Q se enfría rápido, E tarda, R es la reserva. Los indicadores de cooldown hacen ese ritmo *visible* sin que el jugador lo busque — son parte del tempo del combate, no un sistema aparte. Cuando el jugador perfecciona ese pulso — usar Q en el instante exacto en que se libera, encadenarlo con un Pesado, reservar R para cuando el enemigo está en recovery — el HUD desaparece: deja de ser interfaz y se convierte en instinto. Esto espeja lo que el GDD de Combate llama composición: "cada demonio añade una nota a la melodía del combate."

**El fracaso del sistema**: si el jugador tiene que pausar su lectura del combate para procesar el HUD, el HUD falló. La información debe llegar; el jugador no debe ir a buscarla.

## Detailed Design

### Core Rules

**Regla 1 — Barra de HP**

La barra de HP es una barra horizontal (orientación final se define en Visual/Audio Requirements) que refleja `HP_actual / HP_MAX` en tiempo real. No muestra número. Escala de feedback por nivel de peligro:

| Nivel | Condición | Comportamiento visual |
|-------|-----------|----------------------|
| Normal | HP > 50% | Color base. Sin animación adicional. |
| Alerta | 25% < HP ≤ 50% | El tono de la barra cambia (más saturado o desaturado según art bible). |
| Crítico | HP ≤ 25% | La barra pulsa suavemente a ~1 Hz para señalizar peligro inminente. |

Al recibir `health_changed` con daño positivo: la barra contrae su fill con una breve animación (no instantánea) usando **retarget mid-tween** — si llega un nuevo `health_changed` mientras la animación está en curso, el tween actualiza su valor objetivo al nuevo HP sin reiniciarse, asegurando que la barra termine siempre en el valor correcto sin saltos visuales. Al recibir `health_changed` con `tipo_daño = "restauracion"` (Santuario): la barra se llena con un glow suave distinto al daño. La fuente de verdad es exclusivamente la señal `health_changed(nuevo_hp, daño_recibido, tipo_daño)` del EventBus.

**Fuente de HP_MAX**: El HUD recibe `HP_MAX` desde el payload de `loadout_changed` (campo `hp_max: int`). Cambios mid-run (des-equipar demonio que modifica HP_MAX) se sincronizan adicionalmente via E1: cuando GDD #2 emite `health_changed(HP_MAX_nuevo, 0, "stat_reduction")`, el HUD debe actualizar su valor almacenado de HP_MAX tomando `nuevo_hp` como el nuevo máximo. Implementación: el HUD mantiene `var _hp_max: int` que se actualiza en ambos eventos.

**Regla 2 — Slots de Demonio**

El HUD muestra exactamente `slot_count` slots activos (2 en MVP, hasta 5 con Sellos de Vínculo). Cada slot tiene:
- **Ícono del demonio** equipado (o ícono placeholder si el slot está vacío)
- **Sweep overlay radial**: gira en sentido horario sobre el ícono mostrando el cooldown restante. Cuando `cooldown_changed(slot, value)` llega con `value = 0`: el sweep desaparece y el ícono recupera color completo.
- **Etiqueta de key**: `Q / E / R / F / G` para Slots 1–5 (según GDD #6 §2)

Estados de slot:

| Estado | Condición | Visual |
|--------|-----------|--------|
| `READY` | Cooldown = 0, demonio equipado | Ícono a color completo. Sin sweep. |
| `COOLING` | Cooldown > 0 | Ícono semitransparente. Sweep horario activo. |
| `EMPTY` | Sin demonio asignado | Ícono placeholder gris. Sin sweep. |
| `FLASH` | Tecla presionada con slot vacío | Flash de error ~0.3s (E3 de GDD #6). |

El HUD actualiza `slot_count` al recibir `loadout_changed` — si el slot_count aumenta (Sello de Vínculo), aparece el nuevo slot.

**Fuente de `cooldown_max`**: El HUD recibe el cooldown máximo de cada demonio via el campo `slot_cooldowns: Array[float]` del payload extendido de `loadout_changed`. El HUD almacena estos valores en una tabla interna `var _slot_cooldown_max: Array[float]` indexada por slot (0–4). Si `slot_cooldowns[i] ≤ 0.0` (slot vacío o dato inválido), el slot trata el cooldown como READY (guard clause de F-HUD-03). **Nota cross-GDD**: GDD #10 debe extender el payload de `loadout_changed` para incluir `slot_cooldowns` y `hp_max`.

**Regla 3 — Indicador de Swap en Combate**

Cuando el jugador está en estado COMBAT (señal `combat_started` recibida y `combat_ended` no recibida), los slots del loadout muestran un indicador visual sutil permanente que señala que un swap tiene penalización (GDD #10 Regla 10). Resuelve la pregunta abierta de GDD #10 §9.

Cuando `swap_cooldown_remaining > 0` (post-swap en combate), los slots añaden un borde animado que indica específicamente que el swap está en cooldown. Este overlay desaparece al llegar `swap_cooldown_remaining = 0` o al recibir `combat_ended` (cancelación inmediata alineada con CA-LBM-035).

El HUD lee `is_swap_on_cooldown` y `swap_cooldown_remaining` del Loadout via polling en `_process()` o por señal — a definir en ADR.

**Regla 4 — Indicador de Guardado**

Al recibir `show_save_indicator()` del EventBus, el HUD muestra un ícono de guardado en una esquina de la pantalla (posición definida en Visual/Audio Requirements) durante exactamente `SAVE_INDICATOR_DURATION` = 1.5s. Sin interacción del jugador, sin bloqueo de input.

**Regla 5 — Indicador de Santuario** *(provisional — asunción hasta GDD #8)*

El HUD escucha `sanctuary_in_range(is_in_range: bool)` emitida por el nodo Santuario al EventBus. Cuando `is_in_range = true`, muestra un indicador sutil (ícono o glow en la barra de HP) que indica que el Santuario está activo. Si HP ya está al máximo, el indicador igual se muestra pero sin animación de curación. Cuando `is_in_range = false`, el indicador desaparece. *Señal no registrada actualmente — candidato para Phase 5 registry.*

**Regla 6 — Ocultamiento durante Cinemáticas** *(provisional — GDD #17 no diseñado)*

Al recibir `cinematic_started` del EventBus, el HUD se oculta completamente (`CanvasLayer.visible = false`). Al recibir `cinematic_ended`, reaparece. El HUD no decide cuándo ocultarse — responde exclusivamente a estas señales.

---

### States and Transitions

El HUD tiene dos estados de visibilidad:

| Estado | Descripción | Transición hacia |
|--------|-------------|-----------------|
| `VISIBLE` | HUD activo, todos los elementos se actualizan en tiempo real | → `HIDDEN` al recibir `cinematic_started` |
| `HIDDEN` | `CanvasLayer.visible = false`. Sin actualizaciones procesadas. | → `VISIBLE` al recibir `cinematic_ended` |

Dentro de `VISIBLE`, los elementos individuales operan de forma independiente según sus señales (Reglas 1–5). No hay un estado "de combate" global en el HUD — el indicador de swap (Regla 3) es una propiedad reactiva, no un estado de máquina de estados.

El HUD se inicializa en `VISIBLE` al cargar la escena.

---

### Interactions with Other Systems

| Sistema | Dirección | Qué fluye |
|---------|-----------|-----------|
| **Combate (#6)** | Entrada | `cooldown_changed(slot, value)` — actualiza sweep de slot; `combat_started` — activa indicador de combate (Regla 3); `combat_ended` — desactiva indicador de combate. *(Nota: `hit_stun_started` e `i_frames_active` fuera de scope para MVP — reservado para iteración futura del HUD)* |
| **Salud y Daño (#2)** | Entrada | `health_changed(nuevo_hp, daño_recibido, tipo_daño)` — actualiza barra HP y aplica feedback por tipo |
| **Loadout (#10)** | Entrada | `loadout_changed(equipped_demons, gato_available, slot_cooldowns: Array[float], hp_max: int)` — actualiza íconos, slot_count, cooldown_max por slot y HP_MAX; señal `swap_cooldown_updated(remaining: float)` para Regla 3 *(requiere adición a GDD #10)* |
| **Guardado y Carga (#12)** | Entrada | `show_save_indicator()` — dispara el ícono de guardado |
| **Cinemáticas (#17)** *(provisional)* | Entrada | `cinematic_started` / `cinematic_ended` — ocultar/mostrar HUD |
| **Mundo / Santuario** *(provisional)* | Entrada | `sanctuary_in_range(is_in_range: bool)` — indicador de Santuario activo (señal no registrada) |
| **EventBus (ADR-002)** | Arquitectura | Toda suscripción vía `EventBus.signal.connect(callback)` en `_ready()`. El HUD no mantiene referencias directas a ningún sistema. |

## Formulas

El HUD de Combate no produce datos — los transforma a representación visual. Sus fórmulas son conversiones de estado a propiedades de nodo.

---

**Fórmula F-HUD-01: HP Bar Fill Ratio**

```
HP_fill_ratio = HP_actual / HP_MAX
```

| Variable | Símbolo | Tipo | Rango | Descripción |
|----------|---------|------|-------|-------------|
| HP actual de Edrick | `HP_actual` | int | [0, HP_MAX] | Payload `nuevo_hp` de `health_changed`. |
| HP máximo del loadout | `HP_MAX` | int | [1, 125] | Recibido via `loadout_changed` (campo `hp_max`). Actualizado por E1 (stat_reduction). HP_FLOOR = 1 garantiza denominador ≠ 0. |
| Fill ratio de la barra | `HP_fill_ratio` | float | [0.0, 1.0] | Fracción de la barra visible. Mapea a `TextureProgressBar.value` con `min=0, max=1`. |

**Rango de salida:** [0.0, 1.0]. La implementación debe aplicar `clamp(HP_fill_ratio, 0.0, 1.0)` como defensa ante posibles condiciones de carrera de señales entre sistemas, aunque el contrato de GDD #2 garantiza que `HP_actual ∈ [0, HP_MAX]`.
**Ejemplo:** HP_actual = 38, HP_MAX = 75 → `38 / 75 = 0.507` → barra al 50.7%.

---

**Fórmula F-HUD-02: HP Threshold Classifier**

```
HP_threshold =
    "Crítico"  si HP_actual × 4 ≤ HP_MAX
    "Alerta"   si HP_actual × 5 ≤ HP_MAX × 2  (y HP_actual × 4 > HP_MAX)
    "Normal"   en otro caso
```

*(Aritmética entera para evitar drift de punto flotante en valores límite. Esta fórmula es la implementación canónica. Los valores `HP_THRESHOLD_CRITICAL = 0.25` y `HP_THRESHOLD_ALERT = 0.40` en Tuning Knobs son equivalentes documentales: Critical → multiplicador ×4; Alert → multiplicador ×5 vs HP_MAX×2. Los knobs son referencia de diseño, no parámetros runtime. Cambiar los umbrales requiere modificar los multiplicadores en el código.)*

| Variable | Símbolo | Tipo | Rango | Descripción |
|----------|---------|------|-------|-------------|
| HP actual | `HP_actual` | int | [0, HP_MAX] | Fuente: señal `health_changed`. |
| HP máximo | `HP_MAX` | int | [1, 125] | Fuente: GDD #10. |
| Nivel de peligro | `HP_threshold` | enum | {"Normal", "Alerta", "Crítico"} | Determina el comportamiento visual de la barra (Regla 1). |

**Umbrales con HP_MAX = 75:**

| Threshold | HP range | Comportamiento visual |
|-----------|----------|-----------------------|
| Normal | 31–75 HP | Color base, sin animación |
| Alerta | 19–30 HP | Tono alterado |
| Crítico | 0–18 HP | Pulsación a ~1 Hz |

**Ejemplo — Alerta en límite:** HP_actual = 30, HP_MAX = 75 → `30×5=150 ≤ 75×2=150` y `30×4=120 > 75` → **Alerta**. HP_actual = 31: `31×5=155 > 150` → **Normal**.

---

**Fórmula F-HUD-03: Cooldown Sweep Angle**

```
sweep_angle_deg = (cooldown_remaining / cooldown_max) × 360.0
```

| Variable | Símbolo | Tipo | Rango | Descripción |
|----------|---------|------|-------|-------------|
| Cooldown restante | `cooldown_remaining` | float | [0.0, cooldown_max] | Payload `value` de `cooldown_changed(slot, value)`. Decrece a 0.0 en tiempo real. |
| Cooldown máximo del demonio | `cooldown_max` | float | (0.0, ∞) | Definido por demonio en GDD #3. Guard clause: si ≤ 0 → no evaluar, tratar como READY. |
| Ángulo del sweep overlay | `sweep_angle_deg` | float | [0.0, 360.0] | Arco en grados horario desde las 12 del ícono. 360° = cooldown completo. 0° = listo. |

**Rango de salida:** [0.0, 360.0]. Guard clause: si `cooldown_max ≤ 0.0` → renderizar como READY (sin overlay).
**Ejemplo — mitad cooldown:** cooldown_remaining = 1.5s, cooldown_max = 3.0s → `(1.5/3.0) × 360 = 180°` → semicírculo.
**Ejemplo — listo:** cooldown_remaining = 0.0s → `0°` → sin overlay, slot en estado READY.

---

**Fórmula F-HUD-04: Swap Cooldown Overlay Progress**

```
swap_overlay_ratio = swap_cooldown_remaining / SWAP_COMBAT_COOLDOWN
```

| Variable | Símbolo | Tipo | Rango | Descripción |
|----------|---------|------|-------|-------------|
| Cooldown de swap restante | `swap_cooldown_remaining` | float | [0.0, 5.0] | Leído del Loadout. Comienza en SWAP_COMBAT_COOLDOWN al finalizar SWAP_ANIM. |
| Cooldown de swap máximo | `SWAP_COMBAT_COOLDOWN` | float | 5.0 (constante) | Fuente: `entities.yaml` (GDD #10). Denominador siempre > 0. |
| Fill ratio del indicador | `swap_overlay_ratio` | float | [0.0, 1.0] | 1.0 = swap recién ejecutado. 0.0 = swap disponible. Mapea al borde animado de Regla 3. |

**Condición de uso:** Solo evaluar si `is_swap_on_cooldown = true` Y estado = COMBAT. Al recibir `combat_ended`, `swap_overlay_ratio → 0.0` inmediatamente (CA-LBM-035).
**Ejemplo — swap recién hecho:** swap_cooldown_remaining = 5.0s → `5.0/5.0 = 1.0` → borde completo.
**Ejemplo — expirado:** swap_cooldown_remaining = 0.0s → `0.0/5.0 = 0.0` → borde desaparece.

---

> **Nota cross-GDD:** `cooldown_max` por demonio (F-HUD-03) es un hecho cross-sistema cuya fuente de autoridad es GDD #3 (Base de Datos de Demonios). No está registrado actualmente en `entities.yaml` — candidato para Phase 5 registry update.

## Edge Cases

- **E1 — HP_MAX cambia mientras HP_actual es alto**: Si `HP_MAX` baja (des-equipar demonio) mientras `HP_actual > HP_MAX_nuevo`, GDD #2 §3.5.1 emite `health_changed(HP_MAX_nuevo, 0, "stat_reduction")` automáticamente. El HUD recibe la señal y actualiza la barra normalmente. Ningún handling especial necesario.

- **E2 — `health_changed` con `restauracion` cuando HP ya está al máximo**: GDD #2 AC-S4 garantiza que la señal NO se emite si HP ya es máximo. El HUD no recibe nada, no renderiza animación de curación espúrea.

- **E3 — `cooldown_changed` con `value > cooldown_max` (dato fuera de rango)**: F-HUD-03 produciría `sweep_angle_deg > 360°`. El HUD clampea: `sweep_angle_deg = min(sweep_angle_deg, 360.0)`. Resultado: sweep completo visible, sin crash ni arco infinito.

- **E4 — `loadout_changed` con todos los slots vacíos**: Todos los íconos muestran placeholder. Todos los sweeps desaparecen. El indicador de swap de Regla 3 sigue activo si `is_swap_on_cooldown = true`. Estado válido.

- **E5 — `slot_count` aumenta por Sello de Vínculo durante juego activo**: El HUD se inicializa con `SLOTS_MAX` = 5 slots, oculta los inactivos, y muestra/oculta según `slot_count` en cada `loadout_changed`. Nunca inicializar con número fijo de slots menor que `SLOTS_MAX`.

- **E6 — `show_save_indicator()` llega mientras el indicador ya está visible**: El timer del ícono se reinicia desde el segundo trigger. El ícono permanece visible el tiempo correcto desde el último guardado. No se duplica el ícono.

- **E7 — Señales recibidas durante estado `HIDDEN` (dentro de cinemática)**: El HUD procesa todas las señales y actualiza sus datos internos aunque `visible = false`. Al volver a `VISIBLE`, todos los elementos reflejan el estado correcto sin frame espúreo.

- **E8 — `combat_ended` mientras `swap_cooldown_remaining > 0`**: `swap_overlay_ratio → 0.0` inmediatamente (F-HUD-04 condición de uso). El borde animado desaparece en el frame de `combat_ended`. Alineado con CA-LBM-035.

- **E9 — Slot `FLASH` (sin habilidad) mientras el slot ya es `EMPTY`**: Si el jugador limpió el slot pero el sweep anterior no había terminado, el slot está correctamente en EMPTY. El HUD muestra EMPTY + flash; ignora `cooldown_changed` residual para ese slot hasta que `loadout_changed` le asigne un nuevo demonio.

- **E10 — `sanctuary_in_range(true)` durante estado `HIDDEN`**: El HUD actualiza su dato interno pero no renderiza. Al recibir `cinematic_ended`, si Edrick sigue en el radio del Santuario, el indicador aparece correctamente.

## Dependencies

### Dependencias salientes (este sistema necesita)

| Sistema | Relación | Interfaz |
|---------|----------|---------|
| **Combate (#6)** | Dura | Señales `cooldown_changed(slot, value)`, `combat_started`, `combat_ended` vía EventBus. Sin Combate, el HUD no puede mostrar cooldowns ni el indicador de combate. *(`hit_stun_started` e `i_frames_active` removidos del scope MVP — ver Open Questions §6)* |
| **Salud y Daño (#2)** | Dura | Señal `health_changed(nuevo_hp, daño_recibido, tipo_daño)` vía EventBus. Sin esta señal, la barra de HP no se actualiza. |
| **Loadout & Build Management (#10)** | Dura | Señal `loadout_changed(equipped_demons, gato_available, slot_cooldowns, hp_max)` + señal `swap_cooldown_updated(remaining: float)` *(requiere adición a GDD #10)*. Sin Loadout, el HUD no puede mostrar slots, cooldowns ni el indicador de swap. |
| **Guardado y Carga (#12)** | Blanda | Señal `show_save_indicator()`. Sin esta señal, el HUD no muestra el ícono de guardado — funciona en todos los demás aspectos. |
| **Cinemáticas (#17)** *(provisional)* | Blanda | Señales `cinematic_started` / `cinematic_ended` para visibilidad. Provisional hasta que GDD #17 sea diseñado. |
| **EventBus (ADR-002)** | Arquitectural | Toda suscripción vía EventBus. El HUD no mantiene referencias directas a ningún nodo de otro sistema. |

### Dependientes (qué sistemas resolvemos con este GDD)

| Sistema | Qué espera de HUD |
|---------|-------------------|
| **GDD #2 (Salud y Daño)** | §5.9 delega el indicador de Santuario al HUD #18. Resuelto por Regla 5. |
| **GDD #10 (Loadout)** | §9 pregunta abierta 2 ("¿HUD muestra indicador de combate activo?") resuelta por Regla 3. |
| **GDD #12 (Guardado)** | `show_save_indicator()` requiere que HUD defina la duración de display. Constante `SAVE_INDICATOR_DURATION = 1.5s` ya en `entities.yaml`. |

### Verificación de bidireccionalidad

- GDD #6 lista "HUD de Combate (downstream) → señales de cooldown" — alineado ✅
- GDD #2 lista "HUD de Combate (GDD #18) — leerá HP_actual y HP_max" — alineado ✅
- GDD #10 lista "HUD de Combate (#18) — lee equipped_demons, slot_count, is_swap_on_cooldown, swap_cooldown_remaining" — alineado ✅

## Tuning Knobs

Todos los valores deben vivir en un archivo de configuración (`assets/data/hud_config.gd` o similar), nunca hardcodeados.

| Parámetro | Valor Base | Rango Seguro | Qué rompe si... |
|-----------|-----------|--------------|-----------------|
| `HP_THRESHOLD_CRITICAL` | 0.25 (25%) | 0.15–0.35 | **Muy alto (>0.35)**: el jugador está casi siempre en Crítico — pierde impacto. **Muy bajo (<0.15)**: el jugador rara vez llega a Crítico — el feedback emocional se pierde. |
| `HP_THRESHOLD_ALERT` | 0.40 (40%) | 0.35–0.55 | **Muy alto (>0.55)**: el jugador está casi siempre en Alerta — se normaliza, pierde significado. **Muy bajo (<0.35)**: rango Alerta demasiado estrecho; Normal→Crítico es casi directo. *Pilar 5 alignment: el default en 0.40 mantiene la Alerta como señal con peso — el jugador no la siente la mitad del combate.* Invariante: `HP_THRESHOLD_ALERT > HP_THRESHOLD_CRITICAL`. |
| `HP_PULSE_FREQUENCY_HZ` | 1.0 Hz | 0.5–2.0 Hz | **Muy alta (>2Hz)**: el parpadeo es irritante, rompe la inmersión. **Muy baja (<0.5Hz)**: no transmite urgencia. |
| `SLOT_READY_FLASH_DURATION` | 0.3s | 0.1–0.5s | **Muy corto (<0.1s)**: el flash de "sin habilidad" no se percibe. **Muy largo (>0.5s)**: el flash persiste demasiado, parece un estado. |
| `SAVE_INDICATOR_DURATION` | 1.5s | 0.8–3.0s | **Fuente: `entities.yaml`** — no ajustar aquí directamente; cambiar el registro. |
| `HP_BAR_CHANGE_ANIM_DURATION` | 0.12s | 0.05–0.25s | **Muy larga (>0.25s)**: la barra se siente lenta. **Muy corta (<0.05s)**: instantánea, sin impacto visual. |

**Invariante de umbrales**: `HP_THRESHOLD_CRITICAL < HP_THRESHOLD_ALERT < 1.0`. Validar al cargar la configuración — si la invariante se viola, los clasificadores de F-HUD-02 producen resultados indefinidos.

**Nota**: Los cooldowns individuales de demonios se ajustan en GDD #3. El `cooldown_global_scale` vive en GDD #6 §7.6. El HUD solo renderiza el estado que recibe — no controla los cooldowns.

## Visual/Audio Requirements

*(Producido con input del Art Director. Especificaciones accionables — suficientes para comenzar producción sin preguntas de seguimiento.)*

### Layout del HUD

```
┌─────────────────────────────────────┐
│                              [SAVE] │  ← top-right, 8px margin
│                                     │
│                                     │
│ [HP BAR]                            │  ← bottom-left, 8px margin
│ [SANCTUARY]                         │  ← 4px below HP bar, alineado izq.
│                                     │
│       [D1] [D2] [D3] [D4] [D5]     │  ← bottom-center, 12px above bottom
└─────────────────────────────────────┘
```

**Principio rector**: El HUD ocupa la periferia de la visión, nunca el centro. Legible en visión periférica. La acción de combate es el centro de pantalla.

---

### HP Bar

- **Dimensiones**: 72×8px a 1x pixel scale (144×16px renderizada a 2x). Bordes planos (no redondeados).
- **Track**: 1px border `#2A1F1A`, fill interior `#1A1210`. Siempre visible.

| Estado | Condición | Color fill |
|--------|-----------|-----------|
| Normal | HP > 50% | `#B87333` (copper-amber — acento Kingdom 1) |
| Alerta | 25% < HP ≤ 50% | `#C8A020` (gold desaturado, más amarillo) |
| Crítico | HP ≤ 25% | `#9B1B1B` (crimson oscuro) + border `#3A0A0A` |

**Animaciones:**
- **Daño** (CA-HUD-007): contracción ease-out en `HP_BAR_CHANGE_ANIM_DURATION` = 0.12s. Sin ghost bar.
- **Restauración** (CA-HUD-008): expansión 0.20s + outline superior 1px `#E8D8A0` que pulsa 0.15s y desaparece. Expansión vs. contracción es el lenguaje completo.
- **Crítico pulse**: alpha oscila 100% → 65% a 1 Hz. Fórmula de implementación: `alpha = 0.825 + 0.175 × (sin(t × TAU × HP_PULSE_FREQUENCY_HZ) × 0.5 + 0.5)` donde `t` es tiempo acumulado via `_process(delta)` (se pausa con el árbol de escena). Solo alpha — sin cambio de color durante el pulso.

**Art Bible:** Principle 1 — colores oscuros, valor bajo incluso en Crítico. No compite con el brillo de los demonios.

---

### Slots de Demonio

- **Tamaño**: 20×20px a 1x (40×40px a 2x). Cuadrado. Gap entre slots: 4px.
- **Ícono**: 16×16px centrado en área interior 18×18px (1px breathing room por lado).
- **Asset naming**: `ui_demon-slot_[demon-name]_icon_16.png`

| Estado | Visual |
|--------|--------|
| **READY** | Ícono alpha 1.0, color completo. Border `#3A2E28`. Sin overlay. |
| **COOLING** | Ícono alpha 0.45. Sweep overlay `#000000` a 55% alpha, horario desde 12h, recede al drenar. Border `#2A2020`. |
| **EMPTY** | Placeholder `ui_demon-slot_empty_placeholder_16.png` — silueta cadena, `#3A3030`, alpha 1.0. Border `#2A2020`. |
| **FLASH** | Solo el border cambia a `#C83232` durante 0.3s. Sin sacudida. Sin cambio de ícono. |

**Key labels** (Q/E/R/F/G): Pixel font bitmap uppercase, 5px tall. Color `#C8B090`. Posición: esquina inferior-derecha del frame, 1px inset. Opacity: 70% READY / 40% COOLING / 55% EMPTY. Sin background badge.

---

### Indicador de Swap en Combate

- **COMBAT sin cooldown**: border exterior 1px `#786040` (amber dorado) a 2px del frame, estático en todos los slots.
- **COMBAT con swap cooldown activo**: border exterior convierte a sweep radial horario en `#786040` que drena con `swap_overlay_ratio`. Spatial separation: inner sweep = demon cooldown, outer sweep = swap cooldown.
- **Al recibir `combat_ended`**: border exterior desaparece en el mismo frame.

---

### Save Indicator

- **Posición**: Top-right corner, 8px de cada borde.
- **Ícono**: 12×12px — libro/pergamino medieval con marcador. Color `#A09070`. Asset: `ui_save-indicator_icon_12.png`.
- **Animación**: Fade-in 0.15s → hold 1.2s → fade-out 0.15s. Total = 1.5s. Sin background, sin texto.

---

### Sanctuary Indicator

- **Posición**: 4px bajo la HP bar, alineado izquierda.
- **Ícono**: 10×10px — llama de vela o runa ascendente. Color `#E8C860` (gold cálido brillante). Asset: `ui_sanctuary-indicator_icon_10.png`.
- **Animación activa**: Pulse alpha 80% → 100% a 0.5 Hz. Fórmula: `alpha = 0.9 + 0.1 × (sin(t × TAU × 0.5) × 0.5 + 0.5)` donde `t` es el mismo acumulador de `_process(delta)` del HUD. Distinguible del crítico de HP por frecuencia (0.5 Hz vs 1 Hz) y color (`#E8C860` vs `#9B1B1B`).
- **HP al máximo** (CA-HUD-025): ícono presente, alpha estático en 90%, sin pulso.
- **Art Bible:** El Sanctuary indicator es el elemento más brillante del HUD por diseño intencional — la seguridad ganada merece luz.

---

### Audio Cues

| Evento HUD | Categoría | Tono / Calidad |
|------------|-----------|----------------|
| HP entra en Alerta (primera vez por encuentro) | Heartbeat low-pass | 1 thud sordo, ~40Hz, 0.3s. No loop. |
| HP entra en Crítico (primera vez por encuentro) | Heartbeat + tinnitus | Tono HF corto (1.5kHz, 0.1s) + thud bajo |
| Restauración (Santuario) | Campana suave | Campana medieval ascendente, 0.4s, mid-warmth |
| Slot READY (cooldown expira) | Click suave | Click seco o cadena asentándose, 0.05s — casi subliminal |
| Slot FLASH error | Buzz corto | Buzz bajo-medio, 0.08s, aperiódico. Corto e incómodo. |
| Swap cooldown expira | Cadena | Chain-link rattle que resuelve a silencio, 0.15s |
| Save indicator aparece | Pergamino | Rustle muy suave, 0.2s. Apenas audible. |

**Regla global**: Ningún sonido del HUD debe ser audible sobre SFX de combate a volumen máximo. HUD vive en el 20% inferior del headroom de audio.

---

### Assets para el Pixel Artist

| Asset | Tamaño (1x) | Paleta |
|-------|-------------|--------|
| `ui_hp-bar_fill_normal.png` | 1×8px tileable | `#B87333` |
| `ui_hp-bar_fill_alerta.png` | 1×8px tileable | `#C8A020` |
| `ui_hp-bar_fill_critico.png` | 1×8px tileable | `#9B1B1B` |
| `ui_hp-bar_track.png` | 72×8px | `#2A1F1A` border, `#1A1210` fill |
| `ui_demon-slot_frame_ready.png` | 20×20px | `#3A2E28` border |
| `ui_demon-slot_frame_combat.png` | 24×24px | `#786040` outer ring |
| `ui_demon-slot_sweep-mask.png` | 20×20px | `#000000` solid circle |
| `ui_demon-slot_empty_placeholder_16.png` | 16×16px | `#3A3030` |
| `ui_save-indicator_icon_12.png` | 12×12px | `#A09070` |
| `ui_sanctuary-indicator_icon_10.png` | 10×10px | `#E8C860` |

*Los íconos de demonios individuales pertenecen al sprite sheet de personaje/demonio.*

> **📌 Asset Spec** — Requisitos visuales definidos. Después de aprobar el art bible, ejecutar `/asset-spec system:hud-combate` para producir especificaciones por-asset completas con prompts de generación.

## UI Requirements

> **📌 UX Flag — HUD de Combate**: Este sistema tiene requisitos de UI directos. En Fase 4 (Pre-Production), ejecutar `/ux-design` para crear un UX spec para cada pantalla o elemento HUD antes de escribir epics. Las historias que referencien elementos del HUD deben citar `design/ux/hud.md`, no este GDD directamente.

El HUD de Combate no es una pantalla interactiva — es una capa de display pasivo. Sin embargo, define contratos de información visual que el UX spec debe respetar:

1. **Barra de HP**: display solo, sin interacción del jugador. El jugador lee, no interactúa.
2. **Slots de demonio**: display de cooldowns. La interacción (presionar Q/E/R/F/G) pertenece al sistema de input, no al HUD. El HUD solo muestra el efecto (sweep, flash).
3. **Build Screen**: cuando el jugador abre la Build Screen para swappear demonios, la Build Screen es un sistema separado (GDD #20). El HUD muestra el estado del loadout pero NO es la Build Screen.
4. **Ningún elemento del HUD es clickeable**. El HUD es solo-lectura desde la perspectiva de la UI.

El UX spec para `design/ux/hud.md` debe especificar: layout final a distintas resoluciones de pantalla (1280×720 mínimo, 1920×1080 recomendado), feedback de accesibilidad (colores alternativos para daltónicos), y comportamiento en ratios de aspecto no estándar (ultrawide, 4:3).

## Acceptance Criteria

### HP Bar — Regla 1 (F-HUD-01, F-HUD-02)

**CA-HUD-001** — BLOCKING
**GIVEN** HP_actual = 38 y HP_MAX = 75,
**WHEN** la señal `health_changed(38, daño, tipo)` es recibida del EventBus,
**THEN** `TextureProgressBar.value = 38.0 / 75.0 = 0.5067` (tolerancia ± 0.001) y la barra refleja ese fill.

**CA-HUD-002** — BLOCKING
**GIVEN** HP_actual = 0 y HP_MAX = 75,
**WHEN** la señal `health_changed(0, daño, tipo)` es recibida,
**THEN** `HP_fill_ratio = 0.0` y la barra muestra fill cero. Inversamente: HP_actual = HP_MAX → `HP_fill_ratio = 1.0` y barra llena.

**CA-HUD-003** — BLOCKING
**GIVEN** HP_actual = 38, HP_MAX = 75 (`38 × 5 = 190 > 75 × 2 = 150`),
**WHEN** la señal `health_changed` es recibida,
**THEN** el nivel clasificado es `"Normal"`, color base, sin animación.

**CA-HUD-004** — BLOCKING
**GIVEN** HP_actual = 30, HP_MAX = 75 (`30 × 5 = 150 ≤ 75 × 2 = 150`; `30 × 4 = 120 > 75`),
**WHEN** la señal `health_changed` es recibida,
**THEN** el nivel clasificado es `"Alerta"` y la barra muestra tono alterado. *(Confirma el límite exacto del 40% con aritmética entera.)*

**CA-HUD-004b** — BLOCKING
**GIVEN** HP_actual = 31, HP_MAX = 75 (`31 × 5 = 155 > 150`),
**WHEN** la señal `health_changed` es recibida,
**THEN** el nivel clasificado es `"Normal"`. *(Confirma que 31 HP ya no es Alerta con el umbral al 40%.)*

**CA-HUD-005** — BLOCKING
**GIVEN** HP_actual = 20, HP_MAX = 75 (`20 × 4 = 80 > 75`; `20 × 5 = 100 ≤ 150`),
**WHEN** la señal `health_changed` es recibida,
**THEN** el nivel clasificado es `"Alerta"` (no `"Crítico"`).

**CA-HUD-006** — BLOCKING
**GIVEN** HP_actual = 18, HP_MAX = 75 (`18 × 4 = 72 ≤ 75`),
**WHEN** la señal `health_changed` es recibida,
**THEN** el nivel clasificado es `"Crítico"` y la barra inicia pulsación. Con HP_actual = 19 (`19 × 4 = 76 > 75`) → nivel es `"Alerta"`. Confirma que el límite usa aritmética entera, no punto flotante.

**CA-HUD-007a** — BLOCKING
**GIVEN** el HUD está en VISIBLE y no hay un tween de barra en curso,
**WHEN** la señal `health_changed(nuevo_hp, daño_positivo, tipo)` es recibida,
**THEN** un tween de contracción inicia en el mismo frame (`hp_bar_tween.is_running() == true`) y el valor objetivo es `nuevo_hp / HP_MAX`.

**CA-HUD-007b** — BLOCKING
**GIVEN** un tween de contracción está en curso (barra animando desde HP_A hacia HP_B),
**WHEN** la señal `health_changed(nuevo_hp_2, daño_positivo, tipo)` es recibida (nuevo golpe),
**THEN** el tween actualiza su valor objetivo a `nuevo_hp_2 / HP_MAX` sin reiniciarse (retarget mid-tween); al completar, `TextureProgressBar.value == nuevo_hp_2 / HP_MAX` (tolerancia ± 0.001). La barra no salta ni se interrumpe visualmente.

**CA-HUD-007c** — BLOCKING
**GIVEN** el tween iniciado en CA-HUD-007a ha terminado,
**THEN** `TextureProgressBar.value == nuevo_hp / HP_MAX` (tolerancia ± 0.001). El fill resultante es exacto.

**CA-HUD-008** — ADVISORY
**GIVEN** HP_actual < HP_MAX,
**WHEN** la señal `health_changed(nuevo_hp, cantidad, "restauracion")` es recibida,
**THEN** (a) `TextureProgressBar.value` aumenta hacia `nuevo_hp / HP_MAX` (fill se expande, no contrae); (b) un borde superior de 1px en color `#E8D8A0` aparece encima de la barra y pulsa 0.15s antes de desaparecer; (c) ningún tween de contracción ease-out está activo durante este evento. *Sign-off del lead de arte requerido para calidad visual del pulso del borde.*

---

### Slots de Demonio — Regla 2 (F-HUD-03)

**CA-HUD-009** — BLOCKING
**GIVEN** un slot tiene demonio equipado y `cooldown_remaining = 0`,
**WHEN** la señal `cooldown_changed(slot, 0.0)` es recibida,
**THEN** el ícono muestra color completo, el sweep overlay desaparece, estado = `READY`.

**CA-HUD-010** — BLOCKING
**GIVEN** `cooldown_remaining = 1.5s`, `cooldown_max = 3.0s`,
**WHEN** la señal `cooldown_changed(slot, 1.5)` es recibida,
**THEN** `sweep_angle_deg = (1.5 / 3.0) × 360.0 = 180.0°` — el overlay radial muestra exactamente un semicírculo horario.

**CA-HUD-011** — BLOCKING
**GIVEN** un slot recibe `cooldown_max ≤ 0.0` (dato inválido),
**WHEN** el HUD evalúa F-HUD-03,
**THEN** el slot se renderiza como READY sin crash ni división por cero (guard clause activa).

**CA-HUD-012** — BLOCKING
**GIVEN** la señal `cooldown_changed(slot, value)` llega con `value > cooldown_max`,
**WHEN** el HUD evalúa F-HUD-03,
**THEN** `sweep_angle_deg` es clampeado a 360.0°; sin arco mayor a 360°, sin crash.

**CA-HUD-013** — BLOCKING
**GIVEN** un slot tiene demonio equipado con `cooldown_max = 4.0`, y `cooldown_remaining = 2.0`,
**WHEN** la señal `cooldown_changed(slot, 2.0)` es recibida,
**THEN** (a) el slot está en estado `COOLING`; (b) el ícono tiene `modulate.a == 0.45` (tolerancia ± 0.01); (c) `sweep_angle_deg == (2.0 / 4.0) × 360.0 = 180.0°` (tolerancia ± 0.1°). *(Este caso es deliberadamente distinto de CA-HUD-010 — mismos ángulos, diferentes denominadores, para confirmar que F-HUD-03 es paramétrica respecto a cooldown_max.)*

**CA-HUD-014a** — BLOCKING
**GIVEN** un slot no tiene demonio asignado (via `loadout_changed` con ese slot vacío),
**WHEN** el HUD procesa el estado del slot,
**THEN** estado del slot = `EMPTY`, ícono muestra `ui_demon-slot_empty_placeholder_16.png`, sweep overlay no renderizado, `icon.modulate.a == 1.0`.

**CA-HUD-014b** — BLOCKING
**GIVEN** un slot está en estado `EMPTY`,
**WHEN** el evento de input para la tecla de ese slot es recibido,
**THEN** el slot transiciona a estado `FLASH`; un timer de duración `SLOT_READY_FLASH_DURATION` (0.3s) inicia; el borde del slot cambia a `#C83232`; al expirar el timer el slot retorna a `EMPTY` con borde `#2A2020`.

**CA-HUD-015** — ADVISORY
**GIVEN** el HUD está activo con `slot_count` entre 2 y 5,
**WHEN** el jugador puede ver los slots,
**THEN** cada slot muestra su etiqueta de tecla (Q/E/R/F/G según índice) en posición legible.

---

### Indicador de Swap — Regla 3 (F-HUD-04)

**CA-HUD-016** — ADVISORY
**GIVEN** el HUD ha recibido `combat_started` y no ha recibido `combat_ended`,
**WHEN** el jugador mira el HUD,
**THEN** todos los slots muestran un indicador visual sutil permanente señalizando que el swap tiene penalización en combate.

**CA-HUD-017** — BLOCKING
**GIVEN** `swap_cooldown_remaining = 5.0`, `SWAP_COMBAT_COOLDOWN = 5.0`, `is_swap_on_cooldown = true`, estado = COMBAT,
**WHEN** el HUD evalúa F-HUD-04,
**THEN** `swap_overlay_ratio = 5.0 / 5.0 = 1.0` y el borde animado de los slots muestra fill completo.

**CA-HUD-018** — BLOCKING
**GIVEN** `swap_cooldown_remaining = 2.5`, `SWAP_COMBAT_COOLDOWN = 5.0`, `is_swap_on_cooldown = true`, estado = COMBAT,
**WHEN** el HUD evalúa F-HUD-04,
**THEN** `swap_overlay_ratio = 2.5 / 5.0 = 0.5` y el borde animado refleja 50% de fill.

**CA-HUD-019** — BLOCKING
**GIVEN** `swap_cooldown_remaining = 0.0`, `is_swap_on_cooldown = false`, estado = COMBAT,
**WHEN** el HUD evalúa F-HUD-04,
**THEN** `swap_overlay_ratio = 0.0` y el borde animado desaparece; el indicador permanente de combate continúa visible.

**CA-HUD-020** — BLOCKING *(Integration test requerido — no headless)*
**GIVEN** `swap_cooldown_remaining > 0`, `is_swap_on_cooldown = true`, y `swap_overlay_ratio > 0.0`,
**WHEN** la señal `combat_ended` es recibida,
**THEN** inmediatamente después del handler: `swap_overlay_ratio == 0.0` y el borde animado no se renderiza; confirmar via `await get_tree().process_frame` que ningún frame intermedio muestra ratio > 0. *(Alineado con CA-LBM-035.)*

---

### Indicador de Guardado — Regla 4

**CA-HUD-021** — BLOCKING
**GIVEN** el HUD está en VISIBLE,
**WHEN** la señal `show_save_indicator()` es recibida del EventBus,
**THEN** el ícono de guardado aparece en la esquina definida y desaparece exactamente después de `SAVE_INDICATOR_DURATION = 1.5s`, sin interacción del jugador.

**CA-HUD-022** — BLOCKING
**GIVEN** el ícono de guardado está visible con tiempo restante > 0,
**WHEN** la señal `show_save_indicator()` es recibida una segunda vez,
**THEN** el timer se reinicia desde 1.5s; el ícono no se duplica.

---

### Indicador de Santuario — Regla 5 *(provisional — señal pendiente de registro en EventBus)*

**CA-HUD-023** — ADVISORY
**GIVEN** `sanctuary_in_range` está registrada en el EventBus y el HUD está en VISIBLE,
**WHEN** la señal `sanctuary_in_range(true)` es recibida,
**THEN** el HUD muestra el indicador de Santuario (ícono o glow definido en Visual/Audio Requirements).

**CA-HUD-024** — ADVISORY
**GIVEN** el indicador de Santuario está visible,
**WHEN** la señal `sanctuary_in_range(false)` es recibida,
**THEN** el indicador de Santuario desaparece en el frame siguiente.

**CA-HUD-025** — ADVISORY
**GIVEN** HP_actual = HP_MAX y el HUD está en VISIBLE,
**WHEN** la señal `sanctuary_in_range(true)` es recibida,
**THEN** el indicador de Santuario se muestra igual (sin animación de curación), confirmando que indicador de presencia y de curación son visualmente distintos.

---

### Ocultamiento durante Cinemáticas — Regla 6 *(provisional — señal pendiente de GDD #17)*

**CA-HUD-026** — BLOCKING
**GIVEN** el HUD está en VISIBLE,
**WHEN** la señal `cinematic_started` es recibida del EventBus,
**THEN** `CanvasLayer.visible = false` en el mismo frame; ninguna región del HUD es renderizada.

**CA-HUD-027** — BLOCKING
**GIVEN** el HUD está en HIDDEN (`CanvasLayer.visible = false`),
**WHEN** la señal `cinematic_ended` es recibida del EventBus,
**THEN** `CanvasLayer.visible = true` y todos los elementos reflejan el estado actual sin ningún frame espúreo.

**CA-HUD-028** — BLOCKING
**GIVEN** el HUD está en HIDDEN y durante ese período llegan: `health_changed(20, 10, "daño")` con HP_MAX = 75 (→ Alerta, fill = 0.267), `cooldown_changed(slot_1, 1.5)` con cooldown_max = 3.0 (→ sweep 180°), `loadout_changed` con slot_2 vacío,
**WHEN** la señal `cinematic_ended` es recibida,
**THEN** el HUD transiciona a VISIBLE mostrando: barra al 26.7% en Alerta, sweep de 180° en slot_1, placeholder en slot_2; sin ningún frame con estado previo a la cinemática.

---

### Criterios cruzados

**CA-HUD-029** — ADVISORY
**GIVEN** la configuración HUD es cargada desde `hud_config.gd`,
**WHEN** el sistema valida los valores al inicio,
**THEN** se verifica `HP_THRESHOLD_CRITICAL < HP_THRESHOLD_ALERT < 1.0`; si la invariante se viola, el sistema registra un error y usa valores por defecto (0.25, 0.50) sin crashear.

**CA-HUD-030** — BLOCKING
**GIVEN** HP_actual = 60, HP_MAX = 100 (loadout con demonio de +25 HP),
**WHEN** la señal `health_changed(60, 0, "restauracion")` es recibida,
**THEN** `HP_fill_ratio = 60.0 / 100.0 = 0.60` y nivel = `"Normal"` (`60 × 5 = 300 > 100 × 2 = 200`), confirmando que F-HUD-01 y F-HUD-02 son paramétricas respecto a HP_MAX, no asumen HP_MAX = 75 o HP_MAX = 100.

---

> **Notas de implementación (QA Lead):**
> - CA-HUD-016/019/020 — **BLOQUEADAS** hasta que GDD #6 registre `combat_started`/`combat_ended` (Acción X1). No gatear sprint review en estas.
> - CA-HUD-026/027/028 — **BLOQUEADAS** hasta que GDD #17 registre `cinematic_started`/`cinematic_ended` en EventBus. No gatear sprint review en estas.
> - CA-HUD-017/018 — **BLOQUEADAS** hasta que GDD #10 emita `swap_cooldown_updated` (Acción X3).
> - CA-HUD-023/024/025 — ADVISORY. `sanctuary_in_range` no está en EventBus aún.
> - CA-HUD-020 — Integration test requerido (`await get_tree().process_frame`); no ejecutar headless.
> - CA-HUD-028 requiere test de integración completo en GUT. Expandir para cubrir: (a) dos `health_changed` durante HIDDEN, (b) cambio de swap state durante HIDDEN, (c) cruce de umbral Crítico durante HIDDEN.
> - F-HUD-02: usar aritmética entera (`HP × 4 ≤ HP_MAX` para Crítico; `HP × 5 ≤ HP_MAX × 2` para Alerta) en código, nunca división con punto flotante. Cualquier implementación con división producirá drift en los límites de CA-HUD-005/006.

## Open Questions

1. **Señal `sanctuary_in_range` — propietario y registro**: El HUD necesita saber cuándo Edrick está dentro del radio de un Santuario. La señal provisional es `sanctuary_in_range(is_in_range: bool)`. ¿La emite el nodo Santuario directamente al EventBus, o la gestiona el sistema de Exploración (#8)? **Resolver antes de implementar Regla 5**. Candidato para añadir a ADR-002 y `entities.yaml`.

2. ~~**`swap_cooldown_remaining` — polling vs. señal**~~ **RESUELTO**: Decisión de arquitectura — señal `swap_cooldown_updated(remaining: float)` en el EventBus. El polling rompe ADR-002 (requeriría referencia directa al nodo Loadout) y además es incompatible con la garantía "mismo frame" de CA-HUD-020. **Acción pendiente: GDD #10 debe registrar y emitir `swap_cooldown_updated(remaining: float)` en el EventBus.** Owner: Lead Programmer.

3. **Señales cinemáticas — GDD #17**: `cinematic_started` y `cinematic_ended` son provisionales hasta que GDD #17 sea diseñado. CA-HUD-026/027/028 están bloqueadas en este dependencia. **Resolver al diseñar GDD #17 (Cinemáticas)**.

4. ~~**Comportamiento del HUD durante muerte de Edrick**~~ **RESUELTO**: El HUD muestra HP = 0 y mantiene ese estado durante la transición de muerte. Si la muerte desencadena una cinemática (GDD #17), el HUD se ocultará al recibir `cinematic_started` (Regla 6) — el HUD nunca decide cuándo ocultarse, solo responde a la señal. Si en MVP no hay cinemática de muerte, el HUD permanece visible con HP = 0 hasta que otra señal lo oculte. Esto asegura que el jugador siempre tiene confirmación de que su HP llegó a cero antes de que la pantalla cambie.

5. **Multi-resolución y aspect ratio**: El layout del HUD está especificado en posiciones absolutas (px). Para pantallas que no sean 1920×1080 o 1280×720, los anclajes de Godot deben estar correctamente configurados. El UX spec (`design/ux/hud.md`) debe validar esto antes de implementación.

6. *(Resuelto en /design-review 2026-06-05)* **`hit_stun_started` e `i_frames_active`**: Ambas señales removidas del scope MVP. El HUD no reacciona a ellas en esta versión. Reservado para iteración futura del HUD. Para implementación futura: `hit_stun_started` podría triggear un flash en el borde de la barra HP; `i_frames_active` podría mostrar un glow de invulnerabilidad en la barra durante la duración activa.

---

## Accciones Cross-GDD Pendientes *(generadas en /design-review 2026-06-05)*

> Estos items no bloquean la implementación de los subsistemas que ya tienen señales registradas (HP bar, save indicator), pero **bloquean Regla 3 y F-HUD-03** hasta ser resueltos.

| # | Acción requerida | GDD afectado | Bloqueante para |
|---|---|---|---|
| X1 | Registrar `combat_started` y `combat_ended` como señales salientes de Combate en el EventBus | GDD #6 + ADR-002 + `entities.yaml` | Regla 3, CA-HUD-016/019/020 |
| X2 | Extender payload de `loadout_changed` para incluir `slot_cooldowns: Array[float]` y `hp_max: int` | GDD #10 | F-HUD-01, F-HUD-03, todos los CAs de cooldown |
| X3 | Registrar y emitir `swap_cooldown_updated(remaining: float)` desde Loadout al EventBus | GDD #10 + ADR-002 | Regla 3, F-HUD-04, CA-HUD-017–020 |
| X4 | Añadir bus `SFX_HUD` a la arquitectura de audio con nivel −14 dBFS respecto a `SFX_Combate`; actualizar CA-001 de GDD #5 | GDD #5 (Sistema de Audio) | Implementación de audio del HUD |
| X5 | Garantizar que `cooldown_max` por demonio esté en `entities.yaml` (o en la DB de Demonios) con un floor mínimo de 0.1s | GDD #3 + `entities.yaml` | F-HUD-03 guard clause |
