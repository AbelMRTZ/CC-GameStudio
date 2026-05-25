# Combate en Tiempo Real

> **Status**: In Review (Post-Design Fixes)
> **Author**: Abel Martínez + CC Game Studio Agents
> **Last Updated**: 2026-05-26
> **Implements Pillar**: Progresión de Poder Demoníaco · Protagonista Moralmente Gris

## Overview

El sistema de Combate en Tiempo Real es el núcleo interactivo del juego: gestiona el flujo de daño entre el jugador y los enemigos, los estados de combate del personaje (ataque, dash-attack, recibir golpe, invulnerabilidad), y la integración de las habilidades demoníacas en el loop de pelea. Técnicamente, opera a través de una máquina de estados que controla qué acciones están disponibles en cada frame, hitboxes y hurtboxes basados en `Area2D` con activación por animación, y una pipeline de daño que consume la fórmula de Salud y Daño. Desde la perspectiva del jugador, el combate es la expresión directa del poder demoníaco: Edrick comienza limitado (ataques físicos básicos) y gradualmente escala hacia un arsenal devastador que combina tipos de daño, sinergias entre demonios y habilidades de movilidad, con la tensión moral de que las acciones más poderosas pueden acelerar su corrupción.

## Player Fantasy

El jugador debe sentir que Edrick es un arma que se afila sola — pero con un costo. Cada enfrentamiento comienza sintiéndose calculado y mortal (Edrick es experto, no un novato), y escala hacia algo más visceral y poderoso a medida que los demonios se activan. La fantasía tiene dos capas:

1. **Poder creciente y fluido**: encadenar un ataque físico con un estallido de Fuego y rematar con un dash-attack se siente como composición — cada demonio añade una nota a la melodía del combate. El jugador nunca siente que los demonios son extras pegados; son extensiones naturales del cuerpo de Edrick.

2. **Peso y consecuencia**: golpear no es gratuito. Los ataques tienen recovery frames que exigen timing. Recibir daño duele visualmente. Las acciones más destructivas (usar Corrupción, encadenar demonios oscuros) dejan una marca en el mundo moral del personaje — el jugador *siente* que cruzar ciertos límites tiene un precio, aunque en el momento se sienta poderoso.

El peligro a evitar: que el combate se sienta como button-mashing. Cada acción debe tener intención.

## Detailed Design

### Core Rules

**1. Tipos de Ataque Base del Jugador**

Edrick tiene dos ataques melee base, disponibles siempre:

| Ataque | Input | daño_base | Startup | Active | Recovery | Alcance H |
|--------|-------|-----------|---------|--------|----------|-----------|
| Ligero | Botón A | 10 | 0.08s | 0.05s | 0.12s | 32 px |
| Pesado | Botón B | 22 | 0.22s | 0.10s | 0.30s | 48 px |

- Ambos funcionan igual en suelo y en el aire.
- **Heavy-to-Light Cancel**: Durante recovery de Pesado, presionar Botón A cancela la recovery y entra en LIGHT_ATTACK (permite skill expression, evita passivity)
- Sin cancels entre Ligero y otro ataque (Ligero no puede cancelarse a sí mismo)
- Sin cancels de demonios (las habilidades usan el sistema de slots, no cancelan ataques)

**2. Habilidades Demoníacas**

Las habilidades demoníacas se gestionan a través del **sistema de slots**:
- Cada demonio equipado (3-5) ocupa un slot numerado (1, 2, 3… hasta 5).
- Keybindings por defecto: Slot 1 = Q, Slot 2 = E, Slot 3 = R, Slot 4 = F, Slot 5 = G.
- Presionar la tecla de un slot activa la habilidad del demonio correspondiente si: (a) el cooldown es 0, (b) Edrick no está en HIT_STUN o DEAD.
- Cada demonio define su propia habilidad, cooldown y comportamiento (ver Base de Datos de Demonios GDD #3).
- Ejemplo: si el demonio **Dash** está equipado en Slot 1, presionar Q activa el dash-attack (usando los valores de Movimiento: 400 px/s, 0.15s, 0.6s cooldown).
- Las habilidades demoníacas **no** interrumpen un ataque ligero o pesado ya en curso (se encolan y se ejecutan al salir de recovery).

**3. Sistema de Hitbox/Hurtbox**

- **Hurtbox de Edrick**: `Area2D` permanentemente activa en el `CharacterBody2D`, excepto durante I-frames (usar `hurtbox.set_monitoring(false)` al entrar HIT_STUN).
- **Hitbox de ataque**: `Area2D` separada, desactivada por defecto. **El state machine controla su activación**: 
  - Al entrar LIGHT_ATTACK/HEAVY_ATTACK: habilitar hitbox
  - Durante animación: el hitbox permanece activo durante los **active frames** definidos
  - Al salir del estado: deshabilitar hitbox
  - (NO usar AnimationPlayer method tracks para control del hitbox — state machine es la única fuente de verdad)
- Cuando una hitbox de ataque entra en contacto con una hurtbox enemiga: emite señal `hit_landed(target, attack_data)`. Un único golpe por instancia de ataque:
  - Flag `hit_registered` se reinicia cuando LIGHT_ATTACK/HEAVY_ATTACK entra en estado (enter), no en hit time
  - Evita multi-hit si el hitbox es re-activado por accidente
- La pipeline de daño en Salud y Daño se invoca desde esta señal.

**4. Pipeline de Daño**

Al registrar un golpe:
1. Calcular `daño_final` según fórmula 4.1 de Salud y Daño: `round(daño_base × (1 + mod_atacante) × (1 + resistencia_defensiva))`
2. `mod_atacante` = suma de modificadores del demonio activo + sinergias activas (según Base de Datos de Demonios)
3. Aplicar `daño_final` al HP del objetivo
4. Otorgar I-frames al objetivo (`IFRAME_DURATION` segundos)
5. Aplicar vector de knockback al objetivo

**5. Reacción a Impacto (Hit Reaction)**

Al recibir daño:
1. **Descartar queue de habilidades**: si hay una habilidad demoníaca encolada, se cancela (evita "ghost" abilities post-HIT_STUN)
2. Interrumpir acción actual (salir del estado actual → entrar en HIT_STUN)
3. Aplicar vector de knockback: dirección opuesta al atacante, magnitud según tipo de ataque
4. Activar I-frames durante `IFRAME_DURATION` (0.3s): hurtbox.set_monitoring(false), solo el Area2D hurtbox se desactiva (terreno/geometría se mantienen activos)
5. Flash visual de invulnerabilidad (blink del sprite a 10 Hz)
6. Al expirar I-frames: reactivar hurtbox (hurtbox.set_monitoring(true)), restaurar flash, transicionar a IDLE

| Fuente del golpe | Knockback (px) | Dirección |
|-----------------|----------------|-----------|
| Ataque Ligero | 40 px | Opuesta al atacante |
| Ataque Pesado | 90 px | Opuesta al atacante + leve hacia arriba |
| Habilidad Demoníaca | Variable (definida en Base de Datos de Demonios) | Variable |

---

### States and Transitions

| Estado | Descripción | Transiciones posibles |
|--------|-------------|----------------------|
| IDLE | En reposo, sin input | → MOVING (input de movimiento) · → LIGHT_ATTACK · → HEAVY_ATTACK · → DEMON_ABILITY · → HIT_STUN |
| MOVING | Desplazándose | → IDLE (sin input) · → LIGHT_ATTACK · → HEAVY_ATTACK · → DEMON_ABILITY · → HIT_STUN |
| AIRBORNE | En el aire (saltando/cayendo) | → IDLE (al aterrizar) · → LIGHT_ATTACK (aéreo) · → HEAVY_ATTACK (aéreo) · → DEMON_ABILITY · → HIT_STUN |
| LIGHT_ATTACK | Ejecutando ataque ligero | → HIT_STUN (si golpeado durante startup/recovery) · → IDLE (al completar) |
| HEAVY_ATTACK | Ejecutando ataque pesado | → HIT_STUN (si golpeado durante startup/recovery) · → IDLE (al completar) |
| DEMON_ABILITY | Activando habilidad demoníaca | → IDLE (al completar) · → HIT_STUN |
| HIT_STUN | Recibiendo knockback + I-frames | → IDLE (al expirar knockback) |
| DEAD | HP ≤ 0 | → (sistema narrativo/checkpoint toma control) |

Reglas adicionales:
- DEAD es un estado terminal desde combate — el sistema de Salud y Daño emite `player_died` y el sistema narrativo maneja la transición (cutscene + checkpoint).
- No hay cancels entre ataques en MVP (sin "combo" input que cancele recovery).
- HIT_STUN no puede ser interrumpido por ningún input del jugador; solo expira por tiempo.

---

### Interactions with Other Systems

| Sistema | Dirección | Qué fluye |
|---------|-----------|-----------|
| Movimiento y Físicas 2D | Bidireccional | Combate lee `velocity` para habilidades con movimiento (Dash); escribe `velocity` para knockback |
| Salud y Daño | Salida | Combate llama a `apply_damage(target, daño_base, tipo, mod_atacante)` |
| Base de Datos de Demonios | Entrada | Combate lee `cooldown`, `damage_modifier`, `ability_type`, `knockback_magnitude` por demonio |
| Estado del Mundo | Salida | Ataques con Corrupción y ciertas muertes de enemigos pueden llamar a `apply_corruption_delta()` |
| IA de Enemigos (downstream) | Entrada | IA lee el estado de combate del jugador para tomar decisiones (p.ej. atacar durante recovery) |
| HUD de Combate (downstream) | Salida | Combate emite señales `cooldown_changed(slot, value)`, `hit_stun_started`, `i_frames_active` |

## Formulas

**Fórmula 4.1: Modificadores Totales del Atacante**

```
mod_atacante = (∑ demon_modifier_i) × synergy_multiplier
```

Donde:
- `demon_modifier_i` = modificador de daño del i-ésimo demonio equipado (p.ej. Fuego +0.20, Arcano +0.25)
- `∑` = suma de todos los demonios activos en el ataque
- `synergy_multiplier` = multiplicador por sinergia (p.ej. Dash+Fuego = ×1.15, sin sinergia = ×1.00)

Ejemplo:
- Edrick tiene equipados: Fuego (+0.20), Dash (+0.10), Arcano (+0.25)
- Ataca con Fuego: `mod = (0.20 + 0.25) × 1.00 = 0.45` (Arcano amplifica todos)
- Ataca con Dash+Fuego (sinergia activa): `mod = (0.10 + 0.20 + 0.25) × 1.15 = 0.6725` (≈ +67% daño)

**Fórmula 4.2: Daño Final Infligido**

```
daño_final = round(daño_base × (1 + mod_atacante) × (1 + resistencia_defensiva))
```

(Heredada de Salud y Daño GDD #2 — se aplica en Combate al momento del `hit_landed`.)

Donde:
- `daño_base` = daño del ataque (Ligero=10, Pesado=22)
- `mod_atacante` = modificadores del atacante (Fórmula 4.1)
- `resistencia_defensiva` = suma aditiva de resistencias del defensor, capped en [-0.5, +0.5]
- `round()` = redondeo matemático al entero más cercano

Ejemplo:
- Edrick (Fuego+Arcano equipados) ataca Ligero a un enemigo con -0.3 resistencia Fuego
- `daño_final = round(10 × (1 + 0.45) × (1 + (-0.3))) = round(10 × 1.45 × 0.7) = round(10.15) = 10 daño`

**Fórmula 4.3: Magnitud de Knockback Dinámica**

```
knockback_magnitude = base_knockback / (1 + resistencia_defensiva)
```

Donde:
- `base_knockback` = valor fijo según tipo de ataque (Ligero=40px, Pesado=90px, variable para habilidades)
- `resistencia_defensiva` = del defensor (mismo valor de Salud y Daño)
- Resultado: enemigos con resistencias negativas se empujan más; resistencias positivas los protegen del knockback

Ejemplo:
- Edrick Pesado (base=90px) golpea a enemigo con -0.2 resistencia general
- `knockback = 90 / (1 + (-0.2)) = 90 / 0.8 = 112.5 px` ≈ 113 px de empuje
- Enemigo con +0.3 resistencia: `90 / 1.3 = 69.2 px` ≈ 69 px (se resiste más)

**Fórmula 4.4: Tiempo Activo de I-Frames**

```
time_invulnerable = IFRAME_DURATION (constante, Tuning Knob 7.2)
```

Los I-frames son una ventana temporal de invulnerabilidad tras recibir daño. No hay decay — es un valor booleano temporal. Cuando expira, la hurtbox se restaura.

**Fórmula 4.5: Knockback Decay (Linear)**

```
knockback_velocity_current = knockback_velocity_initial × (1.0 - (elapsed_time / IFRAME_DURATION))
```

El knockback vector se aplica al inicio de HIT_STUN y decae linealmente a cero durante los IFRAME_DURATION segundos. El decay es independiente del control del jugador (HIT_STUN desactiva input). Al final de IFRAME_DURATION, knockback_velocity_current ≈ 0.

## Edge Cases

**E1: Daño resultante en 0 o negativo**

Si la fórmula de daño resulta en valor ≤ 0 (solo posible si future demonios o items otorgan resistencias fuera del cap [-0.5, +0.5]):
- Comportamiento: aplicar mínimo 1 daño siempre (el golpe conecta, aunque sea simbólico)
- Esto previene que una resistencia extremadamente alta haga que los ataques sean completamente inútiles
- Ejemplo teórico: Ligero (10 daño) contra enemigo -1.0 resistencia (fuera de cap) = `round(10 × 1 × (1 + (-1.0)))` = `round(0)` = mínimo 1 daño aplicado
- Nota: En MVP, con cap en [-0.5, +0.5], el daño mínimo con resistencia máxima = `round(10 × 1 × 0.5)` = 5 daño; nunca alcanza 0

**E2: Múltiples golpes simultáneos**

Si dos enemigos golpean a Edrick en el mismo frame:
- Cada golpe se procesa independientemente (ambos aplican daño, ambos aplican I-frames)
- La reacción HIT_STUN se activa una única vez (transición a HIT_STUN si no está ya en ese estado)
- El knockback final es la suma vectorial de ambos vectores
- Si Edrick está EN I-FRAMES y recibe otro golpe: el daño se ignora (hurtbox desactivada), pero no se reinician I-frames

**E3: Slot de habilidad demoníaca sin demonio asignado**

Si el jugador presiona Q pero el Slot 1 no tiene ningún demonio:
- Acción: nada ocurre (sin sonido, sin animación, sin cooldown)
- El jugador recibe feedback visual (flash de HUD) indicando "sin habilidad"

**E4: Knockback fuera de los límites del mapa**

Si el vector de knockback empujaría a Edrick fuera del área de juego:
- El movimiento se clampea al límite de la pantalla (CharacterBody2D con collision walls)
- No hay "caída al vacío" — la pared detiene el knockback

**E5: Recibir daño durante DEMON_ABILITY**

Si Edrick está activando una habilidad y es golpeado:
- Se interrumpe la habilidad (DEMON_ABILITY → HIT_STUN)
- El cooldown de esa habilidad **se consume** (no se reembolsa)
- Tras expirar HIT_STUN, Edrick puede usar la habilidad cuando su cooldown esté listo de nuevo

**E6: Intentar atacar durante HIT_STUN**

Si el jugador presiona Ataque Ligero mientras Edrick está en knockback:
- Input se ignora (en HIT_STUN no hay acciones permitidas)
- El input NO se cola (el jugador debe esperar a recuperarse)

**E7: Daño de tipo Corrupción y delta_corruption**

Si un ataque inflige daño Corrupción:
- Además del daño normal, el sistema de Estado del Mundo recibe `apply_corruption_delta(+0.XX)`
- El ataque melee base NO causa corrupción (solo es Físico)
- Solo ciertas habilidades demoníacas pueden causar Corrupción (definidas en Base de Datos de Demonios)

**E8: Resistencia más negativa que -0.5**

Si un enemigo tiene -0.7 resistencia a Fuego pero la fórmula sólo permite cap en -0.5:
- La resistencia se clampea: `resistencia_clamped = max(-0.5, -0.7) = -0.5`
- El enemigo es "débil" pero no ultra-vulnerable (máximo ×1.5 multiplicador de daño)

## Dependencies

| Sistema Dependencia | Relación | Qué necesita este GDD |
|-------------------|----------|----------------------|
| **Movimiento y Físicas 2D** (GDD #1) | Entrada | Valores de dash (400 px/s, 0.15s, 0.6s cooldown) · control de `velocity` para knockback |
| **Salud y Daño** (GDD #2) | Entrada/Salida | Fórmula daño_final · HP ranges · resistencia schema |
| **Base de Datos de Demonios** (GDD #3) | Entrada | damage_modifier por demonio · cooldown values · sinergia multipliers · ability definitions |
| **Estado del Mundo** (GDD #4) | Salida | Emite `apply_corruption_delta()` si habilidades causan Corrupción |
| **Sistema de Audio** (GDD #5) | Bidireccional | Audio emite eventos: `hit_landed` → sonido impacto · `i_frames_started` → efecto visual · Combate lee volumen/pan/modifiers de demonios activos |

**Dependientes (GDDs que dependen de este):**

| Sistema Dependiente | Qué espera de este GDD |
|-------------------|----------------------|
| **IA de Enemigos** (GDD #?) | Estado actual del jugador (IDLE/LIGHT_ATTACK/HIT_STUN/etc) · cooldowns visibles · para tomar decisiones de ataque |
| **Vinculación de Demonios** (GDD #?) | Eventos cuando demonios se equipan/desactivan · modificadores dinámicos |
| **HUD de Combate** (GDD #?) | Señales de cooldown (`cooldown_changed(slot, value)`) · HP actual · estado de I-frames |

## Tuning Knobs

Todos los valores ajustables para balance de combate:

**7.1 Damage Base Values**

| Parámetro | Valor Base | Rango Seguro | Efecto |
|-----------|-----------|--------------|--------|
| `damage_light` | 10 | 5–20 | Daño ataque ligero (presión constante) |
| `damage_heavy` | 22 | 15–40 | Daño ataque pesado (riesgo/recompensa) |
| Ratio pesado/ligero | 2.2x | 1.5x–3.0x | Balance entre velocidad y poder de los ataques |

Guía de tuning:
- Si el juego es muy fácil: `+2–3 en damage_light` y `+4–5 en damage_heavy`
- Si es muy difícil: `-1–2 en ambos`
- El ratio debe mantener Pesado siempre > Ligero (al menos 1.5x)

**7.2 I-Frames Duration**

| Parámetro | Valor Base | Rango Seguro | Efecto |
|-----------|-----------|--------------|--------|
| `IFRAME_DURATION` | 0.3s | 0.15s–0.5s | Ventana de invulnerabilidad tras daño |

Guía:
- **0.15s**: muy punishing, juego difícil, requiere skill
- **0.3s**: balance estándar 2D action
- **0.5s**: muy perdonador, juego casual

**7.3 Knockback Base Values**

| Parámetro | Valor Base | Rango Seguro | Efecto |
|-----------|-----------|--------------|--------|
| `knockback_light` | 40 px | 20–60 px | Empuje ataque ligero |
| `knockback_heavy` | 90 px | 60–150 px | Empuje ataque pesado |

Guía:
- Knockback bajo: enemigos presionan constantemente (caos, acción frenética)
- Knockback alto: enemigos se espacian, juego más "ajedrez"
- Rango seguro: Heavy ≥ 2x Light

**7.4 Attack Animation Timings**

| Parámetro | Valor Base | Rango Seguro | Efecto |
|-----------|-----------|--------------|--------|
| `light_startup` | 0.08s | 0.05s–0.15s | Frames antes de daño en Ligero |
| `light_recovery` | 0.12s | 0.08s–0.2s | Frames después de Ligero antes de siguiente acción |
| `heavy_startup` | 0.22s | 0.15s–0.35s | Frames antes de daño en Pesado |
| `heavy_recovery` | 0.30s | 0.2s–0.5s | Frames después de Pesado antes de siguiente acción |

Guía:
- Startup rápido = ataque es difícil de predecir/esquivar
- Recovery larga = penalty para fallar/es riesgo
- Mantener Heavy recovery ≥ 2x Light recovery

**7.5 Hitbox Size & Reach**

| Parámetro | Valor Base | Rango Seguro | Efecto |
|-----------|-----------|--------------|--------|
| `hitbox_light_reach` | 32 px | 16–48 px | Alcance horizontal ataque ligero |
| `hitbox_heavy_reach` | 48 px | 32–80 px | Alcance horizontal ataque pesado |

Nota: Edrick tiene 24 px de ancho (CharacterBody2D), así que un reach de 32 px significa ~1.33x de su cuerpo.

**7.6 Demon Ability Cooldowns**

Gestionar en Base de Datos de Demonios (GDD #3). Aquí se puede ajustar la escala global si necesitas que **todas** las habilidades se enfríen más rápido/lento:

| Parámetro | Valor Base | Rango Seguro | Efecto |
|-----------|-----------|--------------|--------|
| `cooldown_global_scale` | 1.0x | 0.5x–2.0x | Multiplicador de todos los cooldowns de demonios |

Ejemplo: si pones 0.75x, todas las habilidades se enfríen 25% más rápido.

## Acceptance Criteria

**H1: Ataques Base — Ligero**

- CA-001: Al presionar Botón A, Edrick entra en estado LIGHT_ATTACK
- CA-002: El estado LIGHT_ATTACK dura 0.25s (0.08s startup + 0.05s active + 0.12s recovery)
- CA-003: El hitbox de Ligero se activa solo durante active frames (0.08s–0.13s desde inicio)
- CA-004: Cuando un enemigo es golpeado por Ligero, recibe 10 daño base (sin modificadores)
- CA-005: Ligero knockback es 40px en dirección opuesta al atacante
- CA-006: Solo se cuenta un golpe por swing (si la hitbox persiste más frames, no multi-hit)
- CA-007a: Ligero en aire: daño = 10 (mismo que en suelo)
- CA-007b: Ligero en aire: hitbox active window = 0.08s–0.13s desde inicio (mismo que en suelo)
- CA-007c: Ligero en aire: alcance horizontal = 32 px (mismo que en suelo)
- CA-008: Durante recovery de Ligero (últimos 0.12s), no se puede cancelar a otro ataque
- CA-008b: Durante recovery de Pesado, presionar Botón A cancela la recovery y entra en LIGHT_ATTACK (heavy-to-light cancel)

**H2: Ataques Base — Pesado**

- CA-009: Al presionar Botón B, Edrick entra en estado HEAVY_ATTACK
- CA-010: El estado HEAVY_ATTACK dura 0.62s (0.22s startup + 0.10s active + 0.30s recovery)
- CA-011: El hitbox de Pesado se activa solo durante active frames (0.22s–0.32s desde inicio)
- CA-012: Cuando un enemigo es golpeado por Pesado, recibe 22 daño base (sin modificadores)
- CA-013: Pesado knockback: magnitud = 90px, dirección = opuesta al atacante + 15° hacia arriba (ángulo medido desde horizontal). Assert: `knockback_velocity.angle_to(Vector2.RIGHT)` = 15° ± 1°
- CA-014: Solo se cuenta un golpe por swing (multiestrike prevention)
- CA-015a: Pesado en aire: daño = 22 (mismo que en suelo)
- CA-015b: Pesado en aire: hitbox active window = 0.22s–0.32s desde inicio (mismo que en suelo)
- CA-015c: Pesado en aire: alcance horizontal = 48 px (mismo que en suelo)
- CA-015d: Pesado en aire: knockback = 90 px (mismo que en suelo, dirección opuesta + leve hacia arriba)
- CA-016: Durante recovery de Pesado (últimos 0.30s), no se puede iniciar otro ataque

**H3: Fórmula de Daño — Modificadores**

- CA-017: Si Edrick ataca con Fuego equipado (+0.20 mod), daño = `round(10 × 1.20) = 12` en Ligero
- CA-018: Si Edrick ataca con Fuego(+0.20) + Arcano(+0.25) equipados sin sinergia, mod_atacante = (0.20 + 0.25) × 1.00 = 0.45. Ligero (10 daño) sin resistencia enemiga = `round(10 × (1 + 0.45) × 1.0) = round(14.5) = 15 daño`. Assert: computed value == 15
- CA-019: Si enemigo tiene +0.3 resistencia Fuego (debilidad Fuego) y recibe Ligero (10 base), daño = `round(10 × 1.0 × (1 + 0.3)) = round(13) = 13`
- CA-020: Si enemigo tiene -0.3 resistencia Fuego (armadura Fuego) y recibe Pesado (22 base), daño = `round(22 × 1.0 × (1 + (-0.3))) = round(15.4) = 15`
- CA-021: Daño nunca es menor a 1 (mínimo 1 daño por golpe)

**H4: Habilidades Demoníacas — Slots y Cooldowns**

- CA-022: Presionar Q activa el demonio en Slot 1 si su cooldown es 0
- CA-023: Presionar E activa el demonio en Slot 2 si su cooldown es 0
- CA-024: Presionar R activa el demonio en Slot 3 si su cooldown es 0
- CA-025: Presionar F activa el demonio en Slot 4 si su cooldown es 0
- CA-026: Presionar G activa el demonio en Slot 5 si su cooldown es 0
- CA-027: Si un slot no tiene demonio, presionar su tecla no hace nada (sin error)
- CA-028: Después de activar una habilidad demoníaca en Slot N, el Timer para ese slot inicia con tiempo_restante = cooldown_max (valor configurado en Base de Datos de Demonios para esa habilidad). El timer decrementa cada frame por delta_time. Assert: cooldown_remaining == ability_cooldown_max inmediatamente post-activation
- CA-029: El cooldown de una habilidad debe llegar a exactamente 0 antes de poder activar de nuevo
- CA-030: Si Dash está equipado en Slot 1 y se presiona Q, Edrick entra en estado de dash (400 px/s, 0.15s) — es una habilidad, no un ataque base

**H5: I-Frames y Hit Reaction**

- CA-031: Al recibir daño, Edrick entra en estado HIT_STUN
- CA-032: En HIT_STUN, la hurtbox de Edrick está desactivada (collision mask = 0)
- CA-033: I-frames duran exactamente `IFRAME_DURATION` (0.3s por defecto)
- CA-034: Durante I-frames, Edrick no puede recibir daño adicional (segundo golpe = nada)
- CA-035: Durante I-frames, `modulate.a` (alpha) alterna entre 1.0 (visible) y 0.0 (invisible) a una frecuencia de 10 Hz (período = 0.1s). Assert: después de 0.1s, `modulate.a` ha alternado al menos 1 vez
- CA-036: Al expirar I-frames, la hurtbox se reactiva (collision mask restaurada)
- CA-037: Recibir daño interrumpe cualquier estado previo (LIGHT_ATTACK, DEMON_ABILITY, etc.) → HIT_STUN
- CA-038: HIT_STUN no puede ser interrumpido por input del jugador (se espera a que expire)
- CA-039: Después de expirar HIT_STUN, Edrick regresa a IDLE

**H6: Knockback**

- CA-040: Knockback es un vector: `magnitude = base / (1 + resistencia_defensiva)`, dirección opuesta al atacante
- CA-041: Ligero knockback (40px) con enemigo -0.2 resistencia = `40 / 0.8 = 50px`
- CA-042: Pesado knockback (90px) con enemigo +0.3 resistencia = `90 / 1.3 ≈ 69px`
- CA-043: Knockback se aplica como delta_velocity en CharacterBody2D durante HIT_STUN
- CA-044: Si knockback vector empujaría a Edrick más allá de world_bounds (definidos en Level/Stage data), la posición final es clampada a world_bounds. Assert: `position.x >= world_bounds.position.x` y `position.x <= world_bounds.position.x + world_bounds.size.x` después de aplicar knockback
- CA-045: Knockback se aplica durante HIT_STUN (0.3s I-frames). Knockback magnitude decae linealmente a cero durante HIT_STUN. Assert: en t=0.3s (fin de HIT_STUN), knockback_velocity.length() ≈ 0 (< 1.0 px/s)

**H7: Múltiples Golpes Simultáneos**

- CA-046: Si dos enemigos golpean en el mismo frame, ambos aplican daño
- CA-047: Si Edrick está EN I-FRAMES y recibe un tercer golpe, el daño se ignora pero I-frames no se reinician
- CA-048: Si Enemigo A (izquierda) y Enemigo B (derecha) golpean en el mismo frame con knockback_A = Vector2(-50, 0) y knockback_B = Vector2(+50, 0), el knockback final resultante = Vector2(0, 0) (suma vectorial). Assert: `abs(resultant_knockback.length()) < 1.0 px/s`

**H8: Interrupciones y Colisiones**

- CA-049: Si Edrick está en DEMON_ABILITY y es golpeado, la habilidad se interrumpe
- CA-050: El cooldown de una habilidad interrumpida **se consume** (no reembolso)
- CA-051: Presionar un botón de ataque durante DEMON_ABILITY lo encola; se ejecuta cuando DEMON_ABILITY termina
- CA-052: Presionar un botón durante HIT_STUN no se encola (input descartado)
- CA-052b: Si hay una habilidad demoníaca encolada y Edrick recibe daño (entrada a HIT_STUN), la habilidad encolada se descarta (no dispara post-HIT_STUN)

**H9: Integración con Estado del Mundo**

- CA-053: Si una habilidad aplica daño de tipo Corrupción, se emite señal `corruption_damaged(amount)` a Estado del Mundo
- CA-054: Ataques melee base (Ligero/Pesado) nunca emiten `corruption_damaged` (solo demonios específicos)

**H10: Estados Finales**

- CA-055: Si HP del jugador ≤ 0, entra en estado DEAD
- CA-056: En DEAD, el sistema de Salud y Daño emite señal `player_died`
- CA-057: No se permiten más acciones de combate en DEAD (sistema narrativo toma control)
