# IA de Enemigos

> **Status**: Aprobado (design-review 2026-05-26 — 13 bloqueantes resueltos)
> **Author**: Abel Martínez + CC Game Studio Agents
> **Last Updated**: 2026-05-26
> **Implements Pillar**: Demonios como Poder Transformador · Transformación Moral de Edrick

## Overview

El sistema de IA de Enemigos define cómo los enemigos de Dravaryn perciben, persiguen y atacan a Edrick a lo largo del mundo. Los enemigos no son obstáculos estáticos: son amenazas reactivas que leen activamente el campo de batalla — cierran distancia cuando Edrick es vulnerable, retroceden cuando están bajo presión, y explotan los recovery frames de sus ataques para castigar el juego descuidado. El sistema define **3 arquetipos de enemigos en el MVP**, cada uno con un rol táctico distinto: agresor melee, hostigador a distancia, y oponente defensivo. Cada arquetipo tiene una `debilidad_tipo`: el tipo de daño al que es especialmente vulnerable. Cuando Edrick golpea esa debilidad — generalmente mediante una habilidad demoníaca — el enemigo entra en `STAGGERED`, creando la ventana táctica de alta recompensa que materializa Pilar 2 ("Demonios como Poder Transformador"). Sin este sistema, el combate sería un ejercicio unilateral sin presión ni consecuencias. El cuarto arquetipo (Portador de Corrupción) está diferido al Vertical Slice para un rediseño con intención más clara.

## Player Fantasy

Cuando Edrick enfrenta a un enemigo, el jugador debería sentir la tensión del cazador que estudia a su presa antes de actuar. No hay button-mashing: cada arquetipo tiene un patrón legible que el jugador aprende a interpretar — el Agresor Melee siempre avanza y castiga los recovery frames del jugador con 100% de fidelidad, el Hostigador crea zonas de peligro y huye cuando se le presiona pero castiga los swings pesados, el Defensivo es resistente al spam y recompensa el uso estratégico de ataques pesados y habilidades demoníacas. La recompensa no es simplemente sobrevivir: hay dos momentos de alta recompensa que el jugador debe aprender a crear. **Primero**: leer la recovery window del enemy y atacar en el gap seguro — el instante en que el enemy está en `RECOVER` y `attack_cooldown` todavía corre. **Segundo (y más poderoso)**: usar el demonio correcto para golpear la debilidad del enemigo, provocar `STAGGERED`, y encadenar daño sin riesgo. Ese segundo momento debe sentirse **inevitable, no afortunado** — una ejecución del plan que el jugador construyó con su build, no un golpe de suerte. Cada arquetipo enseña sin tutoriales qué tipos de daño son relevantes, qué ventanas son seguras, y por qué los demonios importan.

## Detailed Design

### Core Rules

**1. Arquetipo de Enemigo Base**

Todos los enemigos MVP son terrestres (no voladores). Cada enemigo tiene:
- Un **tipo de arquetipo** que determina su comportamiento
- Una instancia propia de state machine
- Stats base (HP, speed, detection_range, attack_range, daño_base, tipo_daño)
- Un punto de origen (spawn point) que define el centro de su zona de patrulla

| Arquetipo | HP | Speed (px/s) | Detection Range | Attack Range | Daño Base | Tipo Daño | `debilidad_tipo` | Rareza |
|-----------|-----|-------------|----------------|--------------|-----------|-----------|-----------------|--------|
| Agresor Melee | 40 | 120 | 180 px | 36 px | 12 | `physical` | `fire` | Común |
| Hostigador Distancia | 25 | 80 | 220 px | 180 px (proyectil) | 9 | `fire` o `ice` | tipo opuesto (fire→`ice`, ice→`fire`) | Común |
| Defensivo | 60 | 70 | 160 px | 40 px | 15 | `physical` | `corruption` | Poco común |

> **Nota sobre `debilidad_tipo`**: Ver §7 Sistema de Debilidades. Cuando Edrick inflige daño del tipo de debilidad de un enemigo, el daño se multiplica ×1.5 y el enemigo entra en `STAGGERED` (0.4s). El `debilidad_tipo` de cada instancia es una propiedad pública expuesta para que `CombatSystem` pueda consultarla al resolver daño. Para el Hostigador: el `tipo_daño` de cada instancia (fire o ice) se define en la escena por el level designer; el `debilidad_tipo` se asigna automáticamente como el tipo opuesto.

**2. Estado Base del State Machine**

Cada enemigo corre un state machine de 6 estados:

| Estado | Descripción |
|--------|-------------|
| `PATROL` | Moviéndose en patrulla (izquierda-derecha o waypoints). Sin conciencia de Edrick. |
| `DETECT` | Edrick entró al rango de detección Y hay línea de visión. Delay de alerta breve (0.2s) antes de perseguir. Si Edrick sale de la zona o se rompe LoS durante estos 0.2s, el enemigo regresa a `PATROL` inmediatamente. |
| `PURSUE` | Moviéndose hacia Edrick (o su última posición conocida). `last_known_position` se actualiza cada frame mientras LoS esté activa; se limpia al regresar a `PATROL`. Verificando oportunidades de ataque. |
| `ATTACK` | Dentro del rango de ataque. Ejecutando animación de ataque. El hitbox se activa en los active frames de la animación mediante señal de `AnimationPlayer` (`"hitbox_active"`). Prevención de multi-hit: flag `has_hit_this_swing` (bool) reset al entrar en `ATTACK`; `apply_damage()` solo se llama si `!has_hit_this_swing`, luego se pone a `true`. |
| `RECOVER` | Post-ataque. El enemigo no puede atacar ni moverse (`velocity.x = 0`; la gravedad vertical sigue aplicando). `attack_cooldown` inicia al salir de este estado. |
| `STAGGERED` | Enemigo aturdido: golpeado en su `debilidad_tipo` O el Defensivo absorbió `guard_stagger_threshold` ataques ligeros consecutivos. No puede atacar ni moverse. Emite `enemy_staggered(enemy_id)`. |

**3. Sistema de Detección**

- **Zona de detección**: `Area2D` con `CircleShape2D` (radio = `detection_range` del arquetipo), centrada en el enemigo
- **Line-of-sight (LoS)**: Raycast desde el centro del enemigo al centro de Edrick. Si el raycast impacta geometría estática antes de llegar a Edrick, LoS está bloqueada
- **Condición de detección**: Edrick dentro de la zona AND LoS clara → iniciar `DETECT`
- **Pérdida de tracking**: Si Edrick sale de la zona de detección Y LoS se rompe durante `PURSUE`, el enemigo continúa hacia la última posición conocida (`last_known_position`). Si alcanza esa posición sin recuperar LoS, inicia `pursuit_timeout` (3.0s) y regresa a `PATROL`

**4. Ataque Oportunista (Leer Estado del Jugador)**

Durante `PURSUE`, el enemigo verifica el estado de combate de Edrick cada frame:

```
edrick_combat_state → leído de CombatSystem (propiedad expuesta)
edrick_in_recovery  → bool, leído de CombatSystem; true solo durante recovery frames del ataque activo
```

**Condición de oportunismo:** `edrick_combat_state ∈ trigger_set AND edrick_in_recovery == true AND attack_cooldown == 0 AND enemy en attack_range`

El trigger de oportunismo se activa **exclusivamente durante los recovery frames** del ataque de Edrick (los últimos 0.12s del ciclo). No se activa durante startup ni active frames. Esto significa que el enemigo castiga el sobreextenderse, no el atacar.

| Arquetipo | Condición de trigger | Reacción |
|-----------|---------------------|----------|
| Agresor Melee | `edrick_in_recovery == true` en cualquier estado de ataque | Siempre (100%) → `ATTACK` |
| Hostigador Distancia | `edrick_in_recovery == true` solo en `HEAVY_ATTACK` | Siempre (100%) → `ATTACK` |
| Defensivo | `edrick_in_recovery == true` solo en `HEAVY_ATTACK` | Siempre (100%) → `ATTACK` |

**Prioridad de condiciones en `PURSUE` (de mayor a menor):**
1. Condición de huida del Hostigador (`distance < 100px`) → flee
2. Oportunismo (`edrick_in_recovery + condición de arquetipo + attack_cooldown==0 + en attack_range`) → `ATTACK`
3. Rango de ataque normal (`distance <= attack_range + attack_cooldown==0`) → `ATTACK`
4. Movimiento hacia Edrick (comportamiento default)

**5. Modos de Patrulla**

El sistema soporta dos modos de patrulla configurables por instancia de enemigo:

- **Izquierda-Derecha** (predeterminado): El enemigo camina `patrol_range` px en cada dirección desde su spawn point. Al llegar al límite, invierte dirección. Si detecta borde de plataforma (raycast hacia abajo a 8 px del borde), invierte dirección antes de caer.
- **Waypoints** (para jefes o enemigos importantes, definidos por el level designer): El enemigo sigue una lista de `Vector2[]` definida en la escena. Útil para rutas con subidas/bajadas específicas.

**6. Comportamientos Específicos por Arquetipo**

*Agresor Melee*: Sin comportamiento especial. Siempre avanza hacia Edrick durante `PURSUE`. Sin retreat. El arquetipo más simple y frontal. Enseña al jugador la mecánica de spacing: atacar libremente en melee range tiene consecuencias. Débil a `fire`.

*Hostigador Distancia*: Mantiene `preferred_distance` (150 px) de Edrick moviéndose a su velocidad base (80 px/s). Si `distance_to_edrick < 100 px` → huye en dirección opuesta a Edrick a velocidad base hasta restaurar ≥ 150 px, luego vuelve a `PURSUE`. Solo contraataca con oportunismo durante `HEAVY_ATTACK` del jugador (idéntico al Defensivo, diferenciado porque su ataque es proyectil a distancia). Dispara proyectil en `ATTACK` — el proyectil se instancia mediante señal de `AnimationPlayer` (`"fire_projectile"` al 60% de la animación) apuntando a `edrick.global_position` en ese frame. Débil al tipo opuesto de su `tipo_daño`.

*Defensivo*: Avanza lentamente. Si recibe daño durante `PURSUE` → entra en `RECOVER` (0.5s extra) antes de volver a `PURSUE`. (**Excepción a la regla general de "daño no interrumpe estado": solo el Defensivo tiene este comportamiento.**) Solo contraataca durante `HEAVY_ATTACK` del jugador. Además, lleva un contador `light_attack_hits_absorbed` (reseteado si el Defensivo ataca, entra en RECOVER, o Edrick no ataca durante >2.0s). Al acumular `guard_stagger_threshold` (3) ataques ligeros consecutivos → entra en `STAGGERED` por `guard_stagger_duration` (0.6s) — la guardia se rompe. Débil a `corruption`.

> **Nota sobre stunlock del Defensivo**: La RECOVER extra de 0.5s al recibir daño durante PURSUE no tiene cooldown propio, pero como el timer no se resetea durante RECOVER (AC-AI-008), spam de daño durante el RECOVER extra no produce más RECOVER. El exploit de chaineo solo es posible si Edrick golpea al Defensivo exactamente cuando sale de RECOVER y vuelve a PURSUE — que es exactamente la ventana de contraataque legítima. Esto es aceptable por diseño.

**7. Sistema de Debilidades (Pilar 2 — Demonios como Poder Transformador)**

Cada arquetipo expone `debilidad_tipo: String` como propiedad pública. Cuando `CombatSystem` resuelve el daño de Edrick contra un enemigo, consulta esta propiedad:

- Si `tipo_daño_ataque == enemigo.debilidad_tipo` → aplica `mult_debilidad = 1.5` al daño Y envía señal `enemy_weakness_hit(enemy_id)` al enemigo
- Al recibir `enemy_weakness_hit`: el enemigo transiciona a `STAGGERED` por `stagger_duration` (0.4s)

El estado `STAGGERED` es la ventana táctica de alto valor que materializa "usar el demonio correcto". Durante `STAGGERED` el enemigo no puede atacar ni moverse, y el jugador tiene una ventana garantizada de 0.4s para encadenar ataques sin riesgo de contraataque.

**8. Mecánica de Guardia del Defensivo (detalle de §6)**

El contador `light_attack_hits_absorbed` incrementa cuando el Defensivo recibe daño de tipo `physical` con `edrick_combat_state == LIGHT_ATTACK`. Reset conditions:
- El Defensivo transiciona a `ATTACK` o `RECOVER` (por propio ataque)
- Edrick no inflige golpes al Defensivo durante >2.0s
- El Defensivo entra en `STAGGERED` (el contador ya se consumió)

Al alcanzar `guard_stagger_threshold` (3): entra en `STAGGERED` por `guard_stagger_duration` (0.6s). Este STAGGERED es más largo que el de debilidad (0.4s) para recompensar al jugador que construyó la presión con ataques ligeros.

> **Interacción**: Si el jugador combina — 2 ataques ligeros (construye guardia) + 1 golpe de demonio corruption (activa debilidad) — ambos efectos se aplican: el golpe de debilidad hace ×1.5 daño Y el STAGGERED más largo (guard_stagger entra porque el contador llegó a 3 incluido el golpe de debilidad). El resultado: la mejor estrategia contra el Defensivo es un mix de ligeros + un demonio de tipo corruption.

**9. Proyectil del Hostigador**

- Nodo: `Area2D` con `CircleShape2D` (radio 8 px) + `Sprite2D`
- Velocidad: 200 px/s en dirección a `edrick.global_position` en el momento del disparo
- Al impactar hurtbox de Edrick: llama `apply_damage(edrick, daño_base, tipo_daño, 0.0)`; aplica knockback de 30 px en dirección del proyectil; el proyectil se destruye
- Al impactar geometría estática (collision layer 2): el proyectil se destruye
- Si viaja > 400 px sin impactar: `queue_free()` automáticamente

**8. Muerte del Enemigo**

- Cuando `HP ≤ 0`: el sistema de Salud y Daño emite `entity_died(entity_id)`. El estado del enemy → `DEAD`
- En `DEAD`: desactivar hitbox (`set_monitoring(false)`) y hurtbox, reproducir animación de muerte, esperar 1.5s, `queue_free()`
- Emite señal `enemy_died(enemy_id, global_position, damage_type)` — consumida por Audio (para sfx de muerte) y Estado del Mundo (para tracking de kills del Bestiario)

---

### States and Transitions

| Estado | Condición de Entrada | Condiciones de Salida |
|--------|---------------------|-----------------------|
| `PATROL` | Spawn (default) | Edrick en detection_zone + LoS clara → `DETECT` |
| `DETECT` | Desde `PATROL` cuando se detecta a Edrick | `alert_delay` (0.2s) expira → `PURSUE`; Edrick sale de zona o LoS se rompe durante delay → `PATROL` |
| `PURSUE` | Desde `DETECT`; desde `ATTACK`/`RECOVER`/`STAGGERED` si Edrick fuera de attack_range | En `attack_range` + `attack_cooldown==0` → `ATTACK`; Edrick fuera de zona + LoS rota por `pursuit_timeout` (3s) → `PATROL`; `edrick_in_recovery==true` + condición de arquetipo + `attack_cooldown==0` + en `attack_range` → `ATTACK`; recibe golpe de `debilidad_tipo` → `STAGGERED`; Defensivo: `light_attack_hits_absorbed >= 3` → `STAGGERED` |
| `ATTACK` | En `attack_range`; condición de oportunismo cumplida; `attack_cooldown==0` | Animación de ataque completa (`AnimationPlayer` señal `"attack_end"`) → `RECOVER` |
| `RECOVER` | Desde `ATTACK` | `recover_duration` expira → `PURSUE` (o `PATROL` si Edrick abandonó la zona); al salir: inicia `attack_cooldown` timer |
| `STAGGERED` | Desde `PURSUE`/`ATTACK`/`RECOVER` al recibir golpe en `debilidad_tipo`; Defensivo al acumular 3 ataques ligeros | `stagger_duration` expira → `PURSUE` (o `PATROL` si Edrick salió de zona) |

Reglas adicionales:
- El estado `DEAD` es terminal — gestionado por Salud y Daño, no por este state machine
- Recibir daño generalmente no interrumpe el state machine (aplica knockback visual con `velocity.x` temporal sin cambiar estado). **Excepciones**: (1) Defensivo en `PURSUE` al recibir daño → `RECOVER` (0.5s extra); (2) cualquier arquetipo al recibir golpe de su `debilidad_tipo` → `STAGGERED` (interrumpe cualquier estado excepto `DEAD`)
- Un enemigo en `RECOVER` no puede entrar en `ATTACK` aunque Edrick esté en recovery
- `attack_cooldown` es un float decrementado en `_physics_process`; bloquea tanto el ataque normal como el oportunismo mientras sea > 0

---

### Interactions with Other Systems

| Sistema | Dirección | Qué fluye |
|---------|-----------|-----------|
| Movimiento y Físicas 2D | Entrada | Enemigos usan `CharacterBody2D` + `move_and_slide()` con las mismas reglas físicas del mundo (gravity, collision layer 3); la física de Edrick no se modifica |
| Salud y Daño | Salida | Al conectar hitbox: `apply_damage(edrick, daño_base, tipo_daño, 0.0)` — los enemigos MVP no tienen `mod_atacante`; el knockback se aplica a través de Combate en Tiempo Real |
| Combate en Tiempo Real | Entrada | IA lee `edrick_combat_state` y `edrick_in_recovery: bool` (propiedades públicas de `CombatSystem`) durante `PURSUE`. `edrick_in_recovery` es `true` solo en los últimos 0.12s del ciclo de ataque (recovery frames exclusivamente). También expone `debilidad_tipo` del enemigo para que CombatSystem aplique el multiplicador de debilidad al calcular daño de Edrick contra el enemigo. |
| Sistema de Audio | Salida | Emite: `enemy_detected(enemy_id)`, `enemy_attacked(enemy_id, damage_type)`, `enemy_died(enemy_id, global_position, damage_type)`, `enemy_staggered(enemy_id)` |
| Estado del Mundo | Salida | `enemy_died(enemy_id, global_position, damage_type)` — Estado del Mundo rastrea kills por tipo para el Bestiario |
| HUD de Combate | Ninguna | La IA no emite directamente al HUD; el HUD consume datos de Salud y Daño cuando se aplica el daño |

## Formulas

**Fórmula 1: Daño de Enemigo a Edrick**

Heredada de Salud y Daño (GDD #2). Los enemigos MVP no tienen `mod_atacante` (se omite el factor, equivalente a `mod_atacante = 0.0`):

```
daño_final = round(daño_base × (1 + resistencia_defensiva))
```

> ⚠️ **Convención de `resistencia_defensiva`**: valor **negativo** = Edrick tiene resistencia al tipo (toma menos daño). Valor **positivo** = Edrick tiene vulnerabilidad al tipo (toma más daño). Ejemplo: `resistencia_defensiva = -0.3` → toma 70% del daño. `resistencia_defensiva = +0.5` → toma 150% del daño. El mínimo de 1 HP de daño final se aplica dentro de `apply_damage()`, no en la expresión.

**Variables:**

| Variable | Símbolo | Tipo | Rango | Descripción |
|----------|---------|------|-------|-------------|
| Daño base del arquetipo | `daño_base` | int | 9–15 | Stat fijo del arquetipo (tabla §4.4) |
| Resistencia/vulnerabilidad de Edrick al tipo de daño | `resistencia_defensiva` | float | [-0.5, +0.5] | Suma aditiva. Negativo = protección; positivo = vulnerabilidad |
| Daño final aplicado | `daño_final` | int | ≥1 | HP que pierde Edrick; mínimo 1 (clamp en `apply_damage()`) |

**Rango de output:** 5 (9 × 0.5) a 23 (15 × 1.5) bajo stats normales; nunca inferior a 1.  
**Ejemplo:** Defensivo (daño_base=15) ataca a Edrick sin resistencias → `round(15 × 1.0) = 15 HP`.

---

**Fórmula 2: TTK de Edrick sobre un enemigo (Time-to-Kill)**

```
TTK_edrick = ceil(HP_enemigo / daño_ligero) × cadencia_ligero
```

**Variables:**

| Variable | Símbolo | Tipo | Rango | Descripción |
|----------|---------|------|-------|-------------|
| HP del enemigo objetivo | `HP_enemigo` | int | 25–60 | HP base del arquetipo |
| Daño del Ataque Ligero de Edrick | `daño_ligero` | int | 10 (base) | Sin demonios; con demonios puede ser mayor |
| Duración del ciclo completo de ataque ligero | `cadencia_ligero` | float | 0.25s | Ciclo completo: startup (0.08s) + active (0.05s) + recovery (0.12s) = 0.25s (GDD Combate §3.1) |
| Golpes necesarios | `N_golpes` | int | ≥1 | `ceil(HP_enemigo / daño_ligero)` |
| TTK resultante | `TTK_edrick` | float | segundos | Tiempo mínimo para matar al enemigo spam-eando Ataque Ligero sin interrupción |

> ⚠️ **Baseline teórico**: Este TTK asume golpes consecutivos sin interrupción. En combate real, el Agresor y el Defensivo interrumpirán el spam mediante oportunismo. Usar como referencia de balance mínima, no como predicción de duración real del combate.

| Arquetipo | HP | N_golpes | TTK mínimo |
|-----------|-----|----------|-----------|
| Agresor Melee | 40 | 4 | 1.00s |
| Hostigador Distancia | 25 | 3 | 0.75s |
| Defensivo | 60 | 6 | 1.50s |

**Rango de output:** 0.75s–1.50s (base, sin demonios). Demonios con `damage_modifier > 0` o que activen `debilidad_tipo` del enemigo reducen el TTK.

---

**Fórmula 3: TTK de Enemigo sobre Edrick**

```
TTK_enemigo = ceil(HP_edrick / daño_base) × tiempo_ciclo_ataque
tiempo_ciclo_ataque = attack_anim_duration + recover_duration + attack_cooldown
```

**Variables:**

| Variable | Símbolo | Tipo | Rango | Descripción |
|----------|---------|------|-------|-------------|
| HP de Edrick | `HP_edrick` | int | 75 (base) | HP base sin modificadores de demonios |
| Daño base del arquetipo | `daño_base` | int | 9–15 | Stat fijo del arquetipo |
| Duración de la animación de ATTACK | `attack_anim_duration` | float | 0.30s (provisional) | Tiempo desde que entra en ATTACK hasta que conecta el hitbox y transiciona a RECOVER. Valor provisional — definir en spec de animación por arquetipo antes del Vertical Slice. |
| Duración del estado RECOVER | `recover_duration` | float | 0.5s–0.8s | Por arquetipo (tabla §4.4) |
| Cooldown tras RECOVER | `attack_cooldown` | float | 1.2s–2.0s | Timer que corre desde el final de RECOVER; dueño: variable float en el state machine, decrementada en `_physics_process`, inicia tras salir de `RECOVER` |
| Ciclo completo de ataque | `tiempo_ciclo_ataque` | float | 2.0s–3.1s | `attack_anim_duration + recover_duration + attack_cooldown` |
| Golpes para matar a Edrick | `N_golpes` | int | 5–9 | `ceil(HP_edrick / daño_base)` |
| TTK del enemigo | `TTK_enemigo` | float | segundos | Tiempo que tarda el enemigo en matar a Edrick si Edrick no esquiva ni resiste |

| Arquetipo | N_golpes | Ciclo (con anim 0.30s provisional) | TTK_enemigo |
|-----------|----------|-----------------------------------|-------------|
| Agresor Melee | 7 | 2.00s (0.30+0.50+1.20) | 14.0s |
| Hostigador Distancia | 9 | 2.80s (0.30+0.70+1.80) | 25.2s |
| Defensivo | 5 | 3.10s (0.30+0.80+2.00) | 15.5s |

**Rango de output:** 14.0s–25.2s en 1v1 sin resistencias. La peligrosidad real surge de múltiples enemigos simultáneos. Los valores cambiarán cuando se defina `attack_anim_duration` real por arquetipo.

---

**Fórmula 4: Stats Completos por Arquetipo**

| Arquetipo | HP | Speed | Det. Range | Atk. Range | Daño Base | `attack_cooldown` | `recover_duration` | `patrol_range` | `debilidad_tipo` | `stagger_duration` |
|-----------|-----|-------|-----------|-----------|-----------|------------------|--------------------|----------------|-----------------|-------------------|
| Agresor Melee | 40 | 120 px/s | 180 px | 36 px | 12 | 1.2s | 0.5s | 120 px | `fire` | 0.4s |
| Hostigador Distancia | 25 | 80 px/s | 220 px | 180 px | 9 | 1.8s | 0.7s | 120 px | tipo opuesto | 0.4s |
| Defensivo | 60 | 70 px/s | 160 px | 40 px | 15 | 2.0s | 0.8s | 120 px | `corruption` | 0.4s (0.6s por guard-break) |

*`attack_cooldown` corre desde el final del estado `RECOVER`. El ciclo real entre ataques = `attack_anim_duration (0.30s provisional) + recover_duration + attack_cooldown`.  
`attack_anim_duration` es provisional (0.30s) — confirmar con spec de animación antes del Vertical Slice.*

---

**Fórmula 6: Multiplicador de Debilidad (usado por CombatSystem, definido aquí)**

```
mult_debilidad = 1.5  si tipo_daño_ataque == enemigo.debilidad_tipo
                 1.0  en otro caso

daño_vs_enemigo = round(daño_edrick × mult_debilidad)
```

Cuando `mult_debilidad == 1.5`: CombatSystem envía señal `enemy_weakness_hit(enemy_id)` al enemigo → enemigo transiciona a `STAGGERED` (duración = `stagger_duration`).

**Variables:**

| Variable | Símbolo | Tipo | Rango | Descripción |
|----------|---------|------|-------|-------------|
| Tipo de daño del ataque de Edrick | `tipo_daño_ataque` | String | enum del GDD S&D | Tipo del ataque o habilidad demoníaca que Edrick acaba de ejecutar |
| Tipo de debilidad del enemigo | `enemigo.debilidad_tipo` | String | ver tabla §4.4 | Propiedad pública de cada instancia de enemigo |
| Multiplicador | `mult_debilidad` | float | {1.0, 1.5} | 1.5 si coincide, 1.0 si no |
| Duración del stagger | `stagger_duration` | float | 0.4s (estándar) / 0.6s (guard-break Defensivo) | Ver tabla §4.4 |

**Ejemplo:** Edrick usa una habilidad demoníaca de tipo `fire` contra un Agresor Melee (`debilidad_tipo=fire`, daño_edrick=10): `round(10 × 1.5) = 15` de daño + Agresor entra en `STAGGERED` 0.4s.

---

**Fórmula 5: Escalado de Stats por Zona (esquema para expansión futura)**

```
HP_zona_N     = HP_base     × (1.0 + (N - 1) × 0.30)
daño_zona_N   = round(daño_base   × (1.0 + (N - 1) × 0.15))
speed_zona_N  = speed_base  × (1.0 + (N - 1) × 0.10)
```

**Variables:**

| Variable | Símbolo | Tipo | Rango | Descripción |
|----------|---------|------|-------|-------------|
| Número de zona/reino | `N` | int | 1–9 | N=1 = Reino 1 (MVP) |
| Stat base (Reino 1) | `HP_base / daño_base / speed_base` | float | ver tabla §4.4 | Valores definidos en este GDD |
| Stat escalado | `HP_zona_N / daño_zona_N / speed_zona_N` | float | > stat_base | Stat aplicado en el reino N |

**Ejemplo (Agresor Melee, Reino 3):** `HP = 40 × 1.60 = 64`, `daño = round(12 × 1.30) = 16`, `speed = 120 × 1.20 = 144 px/s`

*Los valores exactos por reino se definirán al diseñar las zonas futuras. Esta fórmula establece el esquema; los coeficientes pueden ajustarse en playtest.*

## Edge Cases

- **Si Edrick muere (HP=0) mientras el enemigo está en `PURSUE`**: El enemigo transiciona a `PATROL` tras 2.0s. El sistema de Salud y Daño emite `edrick_died` y el sistema narrativo toma control. La IA no persigue a un Edrick muerto.

- **Si el `pursue_timeout` expira mientras el enemigo está en mitad de un salto o sobre una plataforma en el aire**: El enemigo completa el aterrizaje primero, luego regresa al spawn point en modo `PATROL`. No aborta movimiento en el aire.

- **Si dos enemigos tienen sus hitboxes superpuestas y ambos atacan a Edrick en el mismo frame**: Ambos golpes se aplican por separado (dos llamadas independientes a `apply_damage()`). Edrick recibe el total sumado. Los I-frames del segundo golpe inician solo cuando los del primero expiran — si el primer golpe aún no activó sus I-frames, el segundo conecta normalmente.

- **Si el proyectil del Hostigador impacta a Edrick mientras Edrick tiene I-frames activos**: El proyectil se destruye pero no aplica daño ni knockback. Los I-frames son absolutos para todos los sources de daño.

- **Si un enemigo entra en `STAGGERED` mientras está en `ATTACK` (golpe de debilidad mid-swing)**: El `STAGGERED` interrumpe el `ATTACK` inmediatamente. La hitbox se desactiva si aún estaba activa. El enemigo entra en `STAGGERED` sin completar el daño del ataque — el jugador que golpeó la debilidad también evitó el contraataque.

- **Si un enemigo recibe un golpe de debilidad durante `STAGGERED` activo**: El `STAGGERED` no se extiende ni reinicia. El daño se aplica normalmente (con mult_debilidad = 1.5) pero el timer del stagger continúa desde donde estaba.

- **Si el Defensivo acumula su tercer ataque ligero en el mismo frame en que entra en RECOVER propio (por su ataque)**: El RECOVER del ataque tiene prioridad. El `light_attack_hits_absorbed` se resetea porque el Defensivo transitó a RECOVER (su propia acción). El guard-break no se activa en ese frame.

- **Si el Hostigador Distancia no tiene espacio para retirarse** (pared o borde detrás, Edrick a menos de 100 px): El hostigador permanece en su posición actual y ejecuta el ataque en lugar de huir. No se teleporta ni atraviesa geometría.

- **Si el Defensivo recibe daño mientras está en `RECOVER`** (su propio post-ataque): El timer de `recover_duration` ya corre y no se resetea. El Defensivo no puede quedarse en RECOVER indefinidamente por spam de daño.

- **Si `edrick_combat_state = DEAD`**: Ningún enemigo transiciona a `ATTACK` por oportunismo. El trigger de oportunismo solo aplica a `{LIGHT_ATTACK, HEAVY_ATTACK, DEMON_ABILITY}`. Los enemigos en `PURSUE` vuelven a `PATROL` tras 2.0s.

- **Si múltiples enemigos detectan a Edrick simultáneamente**: Cada uno corre su state machine de forma independiente. No hay coordinación entre enemigos en MVP (sin flanking diseñado). La dificultad proviene de la independencia, no de la coordinación.

- **Si Edrick usa una habilidad demoníaca que lo mueve fuera del `attack_range` de un enemigo durante el frame en que el enemigo transicionó a `ATTACK`**: El enemigo completa su animación de ataque pero el hitbox no conecta. El enemigo pasa a `RECOVER` normalmente. No hay "chase" durante la animación de ataque.

- **Si un enemigo cae de una plataforma durante `PURSUE`** (pathfinding simple no prevé el salto): El enemigo cae con gravedad normal (`CharacterBody2D`), aterriza en la plataforma inferior, y reanuda `PURSUE`. Si pierde LoS al caer, inicia `pursuit_timeout`.

## Dependencies

**Dependencias duras (el sistema no funciona sin ellas):**

| Sistema | Dirección | Naturaleza | Interfaz específica |
|---------|-----------|------------|---------------------|
| **Movimiento y Físicas 2D** | Upstream | Dura | Enemigos usan `CharacterBody2D` + `move_and_slide()`, collision layer 3; gravedad y física del mundo aplican igual que a Edrick |
| **Salud y Daño** | Upstream | Dura | Llama `apply_damage(edrick, daño_base, tipo_daño, 0.0)` al conectar hitbox; S&D gestiona I-frames, HP update y muerte de Edrick |
| **Combate en Tiempo Real** | Upstream | Dura | Lee `edrick_combat_state` (propiedad pública de `CombatSystem`) para timing de ataques oportunistas |

**Dependencias blandas (el sistema funciona sin ellas, pero pierde features):**

| Sistema | Dirección | Naturaleza | Interfaz específica |
|---------|-----------|------------|---------------------|
| **Sistema de Audio** | Downstream | Blanda | IA emite señales `enemy_detected`, `enemy_attacked`, `enemy_died` — Audio las consume para SFX |
| **Estado del Mundo** | Downstream | Blanda | IA emite `enemy_died(enemy_id, global_position, damage_type)` — Estado del Mundo rastrea kills por tipo para el Bestiario |
| **HUD de Combate** | Downstream | Blanda (transitiva) | El HUD no recibe señales directas de IA; lee de Salud y Daño cuando se aplica el daño |

**Verificación bidireccional:**
- GDD #1 (Movimiento): lista "IA de Enemigos" como sistema dependiente — ✅ confirmado
- GDD #2 (Salud y Daño): lista "IA de Enemigos" como sistema dependiente — ✅ confirmado
- GDD #6 (Combate en Tiempo Real): lista "IA de Enemigos (downstream)" consumiendo `edrick_combat_state` — ✅ confirmado

**Supuestos provisionales:**
- `CombatSystem` expone `edrick_combat_state` y `edrick_in_recovery: bool` como propiedades públicas. Si el API cambia a señales, la lógica de oportunismo necesita un buffer de último estado. Resolver antes de implementar el state machine (ver Open Question 1).
- Cada enemy script expone `debilidad_tipo: String` y `stagger_duration: float` como propiedades públicas. `CombatSystem` las consulta al resolver daño de Edrick vs enemigo.
- Todos los enemigos MVP son terrestres. Enemigos voladores requerirán una extensión de este GDD (nuevo arquetipo, sin gravedad) al diseñar el Vertical Slice.

## Tuning Knobs

| Knob | Valor Base | Rango Seguro | Muy alto → | Muy bajo → |
|------|-----------|-------------|------------|------------|
| `HP` (por arquetipo) | 25–60 | 10–150 | Enemigos esponja; combate tedioso | Mueren en un golpe; pierde tensión |
| `daño_base` (por arquetipo) | 8–15 | 3–30 | Edrick muere muy rápido; juego punitivo | Sin amenaza real; combate trivial |
| `speed` (por arquetipo) | 70–120 px/s | 30–200 px/s | Imposible esquivar en HEAVY_ATTACK recovery | Nunca alcanzan a Edrick; sin presión |
| `detection_range` (por arquetipo) | 160–220 px | 80–300 px | Detectan fuera de pantalla (320×180); "¿de dónde vino?" | Edrick puede ignorarlos acercándose al límite |
| `attack_range` (por arquetipo) | 36–180 px | 16–240 px | Ataques parecen injustos a distancia | Apenas conectan aunque estén encima |
| `attack_cooldown` (por arquetipo) | 1.0–2.0s | 0.5–4.0s | Sin amenaza; el jugador ataca libremente | Sin ventana de contraataque; sensación injusta |
| `recover_duration` (por arquetipo) | 0.45–0.8s | 0.2–2.0s | Ventana de contraataque demasiado grande | Ataques en ráfaga; sensación de spam |
| `alert_delay` | 0.2s | 0.0–0.5s | Reacción demasiado lenta; rompe tensión | Detección instantánea; sin tiempo de reacción |
| `pursuit_timeout` | 3.0s | 1.0–8.0s | Persecución indefinida aunque Edrick escape | Los enemigos se rinden demasiado rápido |
| `patrol_range` | 120 px | 40–240 px | El enemigo patrulla fuera de pantalla | Enemigo parece estático |
| `preferred_distance` (Hostigador) | 150 px | 80–200 px | Se aleja demasiado; proyectiles sin riesgo | Se acerca demasiado; pierde rol de distancia |
| `projectile_speed` (Hostigador) | 200 px/s | 100–350 px/s | Imposible esquivar | Trivial de evadir |
| `projectile_range` (Hostigador) | 400 px | 200–600 px | Cruza pantalla completa; sin zona segura | Se destruye antes de amenazar |
| `stagger_duration` (debilidad) | 0.4s | 0.2s–0.8s | Ventana de stagger demasiado larga; trivializa el combate | El jugador apenas puede aprovechar la ventana; debilidad no se siente recompensada |
| `guard_stagger_threshold` (Defensivo) | 3 ataques | 2–5 | Demasiado fácil abrir la guardia; el Defensivo es trivial | Requiere demasiados golpes ligeros; el jugador no descubre el patrón naturalmente |
| `guard_stagger_duration` (Defensivo) | 0.6s | 0.3s–1.2s | Ventana demasiado larga; Defensivo pierde amenaza | No recompensa suficiente el haber construido la presión |
| `zone_scale_HP` | +30%/reino | +10% a +50%/reino | Esponjas en zonas avanzadas; TTK se dispara | Sin progresión de dificultad |
| `zone_scale_daño` | +15%/reino | +5% a +30%/reino | Juego punitivo en zonas avanzadas | Sin presión adicional |
| `zone_scale_speed` | +10%/reino | +5% a +20%/reino | Imposible esquivar en zonas avanzadas | Sin progresión de amenaza |

**Knobs que interactúan:** `attack_cooldown` y `recover_duration` siempre ajustar en par — uno afecta la ventana de contraataque y el otro el ritmo de ataque.

## Visual/Audio Requirements

**Señales de audio consumidas por Sistema de Audio (GDD #5):**

| Señal emitida | Evento | SFX sugerido |
|---------------|--------|--------------|
| `enemy_detected(enemy_id)` | Enemigo transiciona de PATROL a DETECT | Sonido breve de "alerta" — distinto por arquetipo (metal para Agresor, silbido para Hostigador) |
| `enemy_attacked(enemy_id, damage_type)` | Enemigo ejecuta ataque | Swipe de espada para melee (Agresor, Defensivo); proyectil al disparar (Hostigador) |
| `enemy_died(enemy_id, position, damage_type)` | Enemigo muere | Sonido de impacto + caída — breve, no dramático en MVP |
| `enemy_staggered(enemy_id)` | Enemigo entra en STAGGERED | Golpe de impacto pesado (hit de debilidad) o crack de guardia rota (guard-break del Defensivo) |

**Feedback visual de estado:**
- `PATROL`: animación de idle/caminata suave
- `DETECT`: animación de alerta (cabeza levantada, pausa breve de 0.2s = alert_delay)
- `PURSUE`: animación de carrera
- `ATTACK`: animación de ataque con hitbox activo en active frames
- `RECOVER`: animación de post-ataque (recuperación de postura)
- `STAGGERED`: animación de stagger (tambaleo, flash visual de debilidad) — distintiva para que el jugador reconozca la ventana de alto daño
- `DEAD`: animación de muerte + fade/desaparición a 1.5s

**Requisito de hitbox/hurtbox visual:** En debug build, mostrar hitbox activa (rojo) y hurtbox (azul) como gizmos. Requerido para QA — no se puede testear AC-AI-036 sin visibilidad del hitbox.

📌 **Asset Spec** — Visual/Audio requirements definidos. Después de aprobar el art bible, ejecutar `/asset-spec system:ia-enemigos` para especificaciones de sprites, animaciones y dimensiones por arquetipo.

## UI Requirements

La IA de Enemigos no tiene UI propia en MVP. El HUD consume señales de Salud y Daño cuando el daño se aplica — ninguna señal directa al HUD desde este sistema. Sin flags UX.

## Acceptance Criteria

### Grupo A: Reglas Core — Arquetipo Base y State Machine

**AC-AI-001 — PATROL: movimiento izquierda-derecha**
DADO que un Agresor Melee acaba de hacer spawn y Edrick no está en su zona de detección,
CUANDO han transcurrido 0.1 segundos desde el spawn,
ENTONCES el enemigo se mueve horizontalmente a 120 px/s y su estado es `PATROL`.
Assert: `velocity.x != 0`, `current_state == PATROL`.

**AC-AI-002 — PATROL: inversión en borde de plataforma**
DADO que un enemigo está en `PATROL` con patrol_range=120px,
CUANDO su raycast hacia abajo (8px desde el borde) no detecta geometría en su dirección de marcha,
ENTONCES el enemigo invierte su dirección horizontal dentro de ese mismo frame.
Assert: signo de `velocity.x` invierte antes de cruzar el borde.

**AC-AI-003 — DETECT: transición y alert_delay**
DADO que un Hostigador Distancia está en `PATROL` y Edrick entra en su zona de detección (220px),
CUANDO el raycast de LoS alcanza a Edrick sin impactar geometría,
ENTONCES el enemigo pasa a `DETECT` y permanece exactamente 0.2 segundos antes de transicionar a `PURSUE`.
Assert: `current_state == DETECT` durante 0.2s ± 1 frame; `current_state == PURSUE` al cumplirse.

**AC-AI-004 — DETECT: bloqueo por geometría (LoS)**
DADO que Edrick está dentro del rango de detección de un Defensivo (160px),
CUANDO hay geometría estática que bloquea el raycast entre el enemigo y Edrick,
ENTONCES el enemigo no transiciona a `DETECT`; permanece en `PATROL`.
Assert: `current_state == PATROL` con Edrick en rango pero sin LoS.

**AC-AI-005 — PURSUE: pérdida de tracking y pursuit_timeout**
DADO que un Agresor Melee está en `PURSUE` siguiendo a Edrick,
CUANDO Edrick sale de la zona de detección Y el raycast de LoS se rompe simultáneamente,
ENTONCES el enemigo continúa hacia `last_known_position`; al llegar sin recuperar LoS, inicia un timer de 3.0s; al expirar, el estado pasa a `PATROL`.
Assert: `pursuit_timeout_remaining` inicia en 3.0s; `current_state == PATROL` cuando llega a 0.

**AC-AI-006 — ATTACK: transición desde PURSUE en rango**
DADO que un Agresor Melee está en `PURSUE` con `attack_cooldown == 0`,
CUANDO la distancia a Edrick es ≤ 36px,
ENTONCES el enemigo transiciona a `ATTACK` en ese mismo frame.
Assert: `current_state == ATTACK` en el frame donde `distance_to_edrick <= 36.0`.

**AC-AI-007 — RECOVER: invulnerabilidad y bloqueo de acción**
DADO que un Defensivo completó su animación de ataque y entró en `RECOVER`,
CUANDO han transcurrido 0.79 segundos (justo antes de que expire el recover_duration de 0.8s),
ENTONCES el enemigo no puede moverse ni transicionar a `ATTACK` durante todo `RECOVER`.
Assert: `velocity.length() < 0.1` y `current_state == RECOVER` durante toda la ventana de 0.8s.

**AC-AI-008 — RECOVER: daño no resetea el timer**
DADO que un Defensivo está en su propio `RECOVER` post-ataque con 0.4s restantes,
CUANDO recibe un golpe de Edrick,
ENTONCES el timer de `recover_duration` no se resetea; el Defensivo no puede re-entrar en `ATTACK` antes de que expiren esos 0.4s.
Assert: `recover_timer_remaining` sigue decreciendo a partir del valor previo al golpe.

**AC-AI-009 — DEAD: estado terminal**
DADO que un Agresor Melee acaba de pasar a `DEAD` (HP ≤ 0),
CUANDO el sistema de Salud y Daño emite `entity_died`,
ENTONCES el state machine no procesa más transiciones; la hitbox se desactiva; el nodo se libera (queue_free) después de 1.5s.
Assert: `current_state == DEAD` permanente; `monitoring == false` en hitbox; nodo eliminado a los 1.5s ± 0.1s.

---

### Grupo B: Reglas Core — Comportamientos Específicos por Arquetipo

**AC-AI-010 — Agresor: sin retreat**
DADO que un Agresor Melee está en `PURSUE` y Edrick se acerca a 10px,
CUANDO Edrick continúa avanzando,
ENTONCES el Agresor no retrocede; `velocity.x` mantiene signo en dirección a Edrick o es 0 (en ATTACK), nunca en dirección contraria.
Assert: nunca `sign(velocity.x) == sign(edrick_direction) * -1` durante PURSUE.

**AC-AI-011 — Hostigador: preferred_distance y huida**
DADO que un Hostigador Distancia está en `PURSUE` y Edrick se acerca a menos de 100px,
CUANDO Edrick cruza ese umbral,
ENTONCES el Hostigador invierte su dirección y se aleja hasta restaurar al menos 150px, luego reanuda `PURSUE`.
Assert: `velocity.x` apunta contrario a Edrick cuando `distance_to_edrick < 100`; re-entra en `PURSUE` cuando `distance_to_edrick >= 150`.

**AC-AI-012 — Hostigador: sin espacio para huir (pared detrás)**
DADO que un Hostigador Distancia tiene una pared detrás y Edrick a 80px,
CUANDO Edrick avanza y la distancia cae a < 100px,
ENTONCES el Hostigador permanece en su posición (no atraviesa geometría) y transiciona a `ATTACK`.
Assert: `global_position.x` no cambia más allá de la colisión; `current_state == ATTACK`.

**AC-AI-013 — Hostigador: proyectil — velocidad y rango**
DADO que un Hostigador Distancia ejecuta su ataque,
CUANDO el proyectil es instanciado,
ENTONCES viaja a 200 px/s en línea recta hacia `edrick.global_position` en el momento del disparo; si recorre 400px sin impactar, llama `queue_free()`.
Assert: `projectile.velocity.length() ≈ 200.0 ± 1.0 px/s`; nodo destruido al alcanzar 400px.

**AC-AI-014 — Hostigador: proyectil — destrucción por geometría**
DADO que el proyectil de un Hostigador está en vuelo,
CUANDO impacta collision layer 2 (geometría estática),
ENTONCES el proyectil llama `queue_free()` sin aplicar daño ni knockback.
Assert: `apply_damage()` no invocado en ese impacto; nodo destruido al siguiente frame.

**AC-AI-015 — Defensivo: daño durante PURSUE activa RECOVER breve**
DADO que un Defensivo está en `PURSUE`,
CUANDO recibe cualquier daño de Edrick,
ENTONCES entra en `RECOVER` por 0.5s adicionales antes de volver a `PURSUE`.
Assert: `current_state == RECOVER` inmediatamente post-daño durante exactamente 0.5s ± 1 frame.

**AC-AI-016 — Defensivo: ignora LIGHT_ATTACK para oportunismo**
DADO que un Defensivo está en `PURSUE` dentro de attack_range (40px),
CUANDO `edrick_combat_state == LIGHT_ATTACK`,
ENTONCES el Defensivo NO transiciona a `ATTACK`.
Assert: `current_state` permanece `PURSUE`.

> **AC-AI-017 y AC-AI-018 — Portador de Corrupción**: diferido al Vertical Slice. Los criterios de aceptación de este arquetipo se crearán al rediseñar el Portador.

---

### Grupo C: Reglas Core — Ataque Oportunista

**AC-AI-019 — Oportunismo: Agresor ataca en recovery frames de LIGHT_ATTACK**
DADO que un Agresor Melee está en `PURSUE` dentro de attack_range (36px) con `attack_cooldown == 0`,
CUANDO `edrick_combat_state == LIGHT_ATTACK` AND `edrick_in_recovery == true`,
ENTONCES transiciona a `ATTACK` en ese mismo frame.
Cuando `edrick_in_recovery == false` (startup o active frames), el Agresor NO transiciona.
Assert: `current_state == ATTACK` solo cuando ambas condiciones se cumplen; permanece `PURSUE` si `edrick_in_recovery == false`.

**AC-AI-020 — Oportunismo: Agresor — HEAVY_ATTACK y DEMON_ABILITY también disparan (en recovery)**
DADO que un Agresor Melee está en `PURSUE` dentro de attack_range con `attack_cooldown == 0`,
CUANDO `edrick_combat_state == HEAVY_ATTACK` Y `edrick_in_recovery == true` (y luego con `DEMON_ABILITY`),
ENTONCES transiciona a `ATTACK` inmediatamente en ambos casos.
Assert: `current_state == ATTACK` para los dos valores del enum; NO transiciona si `edrick_in_recovery == false`.

**AC-AI-021 — Oportunismo: Agresor en RECOVER no puede atacar**
DADO que un Agresor Melee está en `RECOVER`,
CUANDO `edrick_combat_state == LIGHT_ATTACK`,
ENTONCES el Agresor NO transiciona a `ATTACK`.
Assert: `current_state == RECOVER` se mantiene.

**AC-AI-022 — Oportunismo: Hostigador — solo en HEAVY_ATTACK recovery (determinístico)**
DADO que un Hostigador Distancia está en `PURSUE` dentro de attack_range con `attack_cooldown == 0`,
CUANDO `edrick_combat_state == HEAVY_ATTACK` AND `edrick_in_recovery == true`,
ENTONCES el Hostigador transiciona a `ATTACK` en ese mismo frame (determinístico, 100%).
Cuando `edrick_combat_state == LIGHT_ATTACK` o `DEMON_ABILITY` (con o sin recovery), el Hostigador NO transiciona.
Assert: `current_state == ATTACK` exclusivamente con `HEAVY_ATTACK + edrick_in_recovery==true`; comportamiento idéntico al Defensivo (AC-AI-023).

**AC-AI-023 — Oportunismo: Defensivo solo en HEAVY_ATTACK**
DADO que un Defensivo está en `PURSUE` dentro de attack_range (40px) con `attack_cooldown == 0`,
CUANDO `edrick_combat_state == HEAVY_ATTACK` → transiciona a `ATTACK`;
CUANDO `edrick_combat_state == LIGHT_ATTACK` o `DEMON_ABILITY` → NO transiciona.
Assert: `current_state == ATTACK` solo con `HEAVY_ATTACK`; permanece `PURSUE` en los otros dos.

> **AC-AI-024 — Portador de Corrupción**: diferido al Vertical Slice.

**AC-AI-025 — Oportunismo: no dispara con edrick_combat_state == DEAD**
DADO que Edrick está en estado `DEAD`,
CUANDO cualquier arquetipo verifica el trigger de oportunismo,
ENTONCES ningún enemigo transiciona a `ATTACK`; los que están en `PURSUE` pasan a `PATROL` tras 2.0s.
Assert: `current_state != ATTACK` para todos los enemigos con `edrick_combat_state == DEAD`.

---

### Grupo D: Fórmulas

**AC-AI-026 — Fórmula 1: daño de enemigo a Edrick sin resistencias**
DADO que un Hostigador Distancia (daño_base=9) ataca a Edrick con `resistencia_defensiva == 0.0`,
CUANDO `apply_damage()` procesa el golpe,
ENTONCES Edrick pierde exactamente 9 HP.
Assert: `edrick.hp_before - edrick.hp_after == 9`.

**AC-AI-027 — Fórmula 1: daño de enemigo con resistencia de Edrick**
DADO que un Agresor Melee (daño_base=12) ataca con `resistencia_defensiva == -0.3` al tipo `physical`,
CUANDO `apply_damage()` procesa el golpe,
ENTONCES Edrick pierde `round(12 × 0.7) = round(8.4) = 8 HP`.
Assert: `edrick.hp_before - edrick.hp_after == 8`.

**AC-AI-028 — Fórmula 1: daño mínimo = 1**
DADO que Edrick tiene `resistencia_defensiva == -0.5` (cap máximo) y un enemigo ataca con daño_base=1,
CUANDO `apply_damage()` procesa el golpe,
ENTONCES Edrick pierde al menos 1 HP.
Assert: `edrick.hp_before - edrick.hp_after >= 1`.

**AC-AI-029 — Fórmula 2: TTK Edrick sobre Agresor Melee**
DADO que Edrick tiene daño_ligero=10 y el Agresor tiene HP=40,
CUANDO Edrick ejecuta 4 ataques Ligeros consecutivos sin demonios,
ENTONCES el Agresor muere exactamente al cuarto golpe en ≈ 1.00s.
Assert: `enemy.hp <= 0` tras cuarto golpe; `elapsed_time ≈ 1.00s ± 0.05s`.

**AC-AI-030 — Fórmula 2: TTK Edrick sobre Hostigador Distancia**
DADO que el Hostigador tiene HP=25 y daño_ligero=10,
CUANDO Edrick conecta 3 ataques Ligeros,
ENTONCES el Hostigador muere al tercer golpe en ≈ 0.75s.
Assert: `enemy.hp <= 0` tras tercer golpe; `elapsed_time ≈ 0.75s ± 0.05s`.

**AC-AI-031 — Fórmula 2: TTK Edrick sobre Defensivo**
DADO que el Defensivo tiene HP=60 y daño_ligero=10,
CUANDO Edrick conecta 6 ataques Ligeros,
ENTONCES el Defensivo muere al sexto golpe en ≈ 1.50s.
Assert: `enemy.hp <= 0` tras sexto golpe; `elapsed_time ≈ 1.50s ± 0.05s`.

**AC-AI-032 — Fórmula 3: TTK del Agresor sobre Edrick**
DADO que Edrick tiene HP=75 sin resistencias y el Agresor tiene daño_base=12, ciclo=2.00s (0.30s anim + 0.50s recover + 1.20s cooldown),
CUANDO el Agresor ataca repetidamente sin que Edrick esquive,
ENTONCES necesita 7 golpes y ≈ 14.0s en total para matar a Edrick.
Assert: `edrick.hp <= 0` tras séptimo golpe; `total_elapsed ≈ 14.0s ± 0.5s`.

**AC-AI-033 — Fórmula 3: attack_cooldown corre desde el final de RECOVER**
DADO que un Agresor Melee acaba de salir del estado `RECOVER`,
CUANDO el timer de `attack_cooldown` inicia,
ENTONCES el próximo ataque ocurre 1.2s después de que `RECOVER` expiró.
Assert: `time_since_recover_expired >= 1.2s` en el momento del siguiente `ATTACK`.

**AC-AI-034 — Fórmula 4: stats completos del Hostigador Distancia**
DADO que se instancia un Hostigador Distancia,
CUANDO se inspeccionan sus stats,
ENTONCES todos sus valores coinciden exactamente: HP=25, speed=80px/s, detection_range=220px, attack_range=180px, daño_base=9, attack_cooldown=1.8s, recover_duration=0.7s, patrol_range=120px.
Assert: comparación punto a punto de todos los campos.

**AC-AI-035 — Fórmula 5: escalado por zona (Reino 3)**
DADO que se instancia un Agresor Melee en Zona N=3,
CUANDO el sistema de escalado aplica los multiplicadores,
ENTONCES: HP = round(40 × 1.60) = 64; daño = round(12 × 1.30) = 16; speed = 120 × 1.20 = 144 px/s.
Assert: `hp_final == 64`, `damage_final == 16`, `speed_final == 144.0`.

---

### Grupo E: Integración Cross-System

**AC-AI-036 — Integración Salud y Daño: apply_damage invocado correctamente**
DADO que un Agresor Melee (daño_base=12, tipo=`physical`, mod_atacante=0.0) conecta su hitbox con la hurtbox de Edrick,
CUANDO se procesa la señal de hit,
ENTONCES `apply_damage(edrick, 12, "physical", 0.0)` se invoca exactamente una vez por swing.
Assert: `apply_damage` llamado con esos parámetros exactos; llamado 1 vez aunque el hitbox permanezca activo varios frames.

**AC-AI-037 — Integración Combate: lectura de edrick_combat_state en el mismo frame**
DADO que `CombatSystem.edrick_combat_state` cambia de `IDLE` a `LIGHT_ATTACK` en el frame T,
CUANDO la lógica de oportunismo del Agresor se procesa,
ENTONCES la transición a `ATTACK` ocurre en el frame T, no en T+1.
Assert: `current_state == ATTACK` en el mismo frame del cambio de estado de Edrick.

**AC-AI-038 — Integración Audio: señal enemy_detected emitida**
DADO que un enemigo transiciona de `PATROL` a `DETECT`,
CUANDO la transición ocurre,
ENTONCES emite `enemy_detected(enemy_id)` exactamente una vez.
Assert: listener recibe exactamente 1 emisión con el `enemy_id` correcto.

**AC-AI-039 — Integración Audio: señal enemy_attacked emitida**
DADO que un Hostigador Distancia transiciona a `ATTACK` y dispara su proyectil,
CUANDO el proyectil impacta la hurtbox de Edrick,
ENTONCES emite `enemy_attacked(enemy_id, "fire")` (o `"ice"` según el arquetipo).
Assert: listener recibe `enemy_attacked` con el `damage_type` correcto.

**AC-AI-040 — Integración Audio: señal enemy_died emitida**
DADO que un Agresor Melee tiene HP ≤ 0,
CUANDO el sistema de Salud y Daño emite `entity_died`,
ENTONCES la IA emite `enemy_died(enemy_id, global_position, "physical")` al siguiente frame.
Assert: listener recibe `enemy_died` con los tres argumentos correctos.

**AC-AI-041 — Integración Estado del Mundo: kill tracking**
DADO que un Defensivo muere (HP ≤ 0),
CUANDO la señal `enemy_died(enemy_id)` es emitida,
ENTONCES Estado del Mundo incrementa el contador de `enemies_defeated` en el área actual en 1.
Assert: `world_state.areas_visited[area_id].enemies_defeated` aumenta de N a N+1.

> **AC-AI-042 y AC-AI-043 — Portador de Corrupción**: diferido al Vertical Slice.

**AC-AI-044 — Integración Movimiento: gravedad aplica al enemigo**
DADO que un Agresor Melee cae de una plataforma durante `PURSUE`,
CUANDO cae verticalmente,
ENTONCES el `CharacterBody2D` aplica gravedad con `move_and_slide()`; aterriza en la plataforma inferior y reanuda `PURSUE` o inicia `pursuit_timeout` si perdió LoS.
Assert: `is_on_floor() == true` después del aterrizaje; `current_state` es `PURSUE` o inicia `pursuit_timeout`.

---

### Grupo F: Edge Cases Críticos

**AC-AI-045 — Double-hit: dos enemigos en mismo frame**
DADO que un Agresor Melee (daño_base=12) y un Defensivo (daño_base=15) conectan sus hitboxes con Edrick en el mismo frame,
CUANDO ambos ataques se procesan,
ENTONCES Edrick pierde la suma de ambos daños (12 + 15 = 27 HP sin resistencias) en dos llamadas independientes.
Assert: `apply_damage` llamado 2 veces en el mismo frame; `edrick.hp` decrece en 27.

**AC-AI-046 — I-frames: proyectil vs I-frames activos**
DADO que Edrick está en I-frames (0.3s activos) tras recibir un golpe,
CUANDO el proyectil del Hostigador impacta la hurtbox de Edrick durante esa ventana,
ENTONCES el proyectil se destruye sin aplicar daño ni knockback; los I-frames no se reinician.
Assert: `apply_damage` no invocado; `edrick.hp` no cambia; `i_frames_remaining` no se resetea.

**AC-AI-047 — I-frames: segundo golpe melee vs I-frames activos**
DADO que Edrick acaba de recibir un golpe y está en I-frames,
CUANDO un segundo enemigo conecta su hitbox dentro de la ventana de I-frames,
ENTONCES el segundo golpe no aplica daño ni knockback; los I-frames no se reinician.
Assert: `edrick.hp` idéntico antes y después del segundo impacto dentro de la ventana.

**AC-AI-048 — Oportunismo: Edrick sale del attack_range durante ATTACK**
DADO que un Agresor transicionó a `ATTACK` por oportunismo estando dentro de attack_range (36px),
CUANDO Edrick usa una habilidad demoníaca que lo mueve fuera del attack_range en ese mismo frame,
ENTONCES el Agresor completa su animación; el hitbox no conecta; el Agresor pasa a `RECOVER` normalmente.
Assert: `current_state == RECOVER` tras la animación; `apply_damage` no invocado en ese swing.

**AC-AI-049 — Edrick muere en PURSUE: todos los enemigos retornan a PATROL**
DADO que múltiples enemigos están en `PURSUE`,
CUANDO Edrick llega a 0 HP y el sistema emite `edrick_died`,
ENTONCES todos los enemigos en `PURSUE` transicionan a `PATROL` tras 2.0s.
Assert: `current_state == PATROL` para todos los enemigos ≥ 2.0s después de `edrick_died`.

**AC-AI-050 — Pursuit_timeout: enemigo en el aire al expirar**
DADO que un Agresor Melee está en el aire (sin `is_on_floor()`) cuando `pursuit_timeout` llega a 0,
CUANDO se procesa la transición a `PATROL`,
ENTONCES el enemigo completa el aterrizaje antes de regresar al spawn point.
Assert: `is_on_floor() == true` antes de iniciar el retorno; `current_state == PATROL` post-aterrizaje.

**AC-AI-051 — Multiple enemies: state machines independientes**
DADO que tres Agresores Melee detectan a Edrick simultáneamente,
CUANDO cada uno ejecuta su state machine,
ENTONCES cada uno corre su propio ciclo de forma independiente; sus estados pueden diferir entre sí en el mismo frame.
Assert: `enemy_a.current_state`, `enemy_b.current_state`, `enemy_c.current_state` son independientes.

**AC-AI-052 — Proyectil vs I-frames: continuación de trayectoria**
DADO que el proyectil del Hostigador está en vuelo y Edrick entra en I-frames antes del impacto,
CUANDO el proyectil llega a la posición de Edrick (hurtbox con `monitoring == false`),
ENTONCES el proyectil no detecta solapamiento; continúa su trayectoria hasta alcanzar 400px y se destruye.
Assert: `apply_damage` no invocado; proyectil vive hasta los 400px de travel si no hay otra geometría.

> **AC-AI-053 — Portador de Corrupción**: diferido al Vertical Slice.

---

### Grupo G: Sistema de Debilidades y STAGGERED

**AC-AI-054 — STAGGERED: golpe de debilidad transiciona al enemigo inmediatamente**
DADO que un Agresor Melee (`debilidad_tipo=fire`) está en `PURSUE`,
CUANDO Edrick conecta un ataque con `tipo_daño == "fire"` contra el Agresor,
ENTONCES el Agresor transiciona a `STAGGERED` inmediatamente, emite `enemy_staggered(enemy_id)`, y el daño aplicado es `round(daño_edrick × 1.5)`.
Assert: `current_state == STAGGERED`; `enemy_staggered` emitido 1 vez; `hp_before - hp_after == round(daño_edrick × 1.5)`.

**AC-AI-055 — STAGGERED: bloqueo de movimiento y ataque durante todo el estado**
DADO que cualquier arquetipo está en `STAGGERED`,
CUANDO han transcurrido 0.1s desde la entrada en el estado,
ENTONCES el enemigo no puede moverse ni transicionar a `ATTACK`.
Assert: `velocity.length() < 0.1`; `current_state == STAGGERED`; no se producen transiciones a `ATTACK` durante el período.

**AC-AI-056 — STAGGERED: expiración → PURSUE si Edrick sigue en zona**
DADO que un Hostigador Distancia está en `STAGGERED` (`stagger_duration=0.4s`) y Edrick permanece en zona de detección,
CUANDO expiran los 0.4s,
ENTONCES el Hostigador transiciona a `PURSUE`.
Assert: `current_state == PURSUE` en el frame en que `stagger_timer <= 0`.

**AC-AI-057 — STAGGERED: interrumpe ATTACK mid-swing**
DADO que un Agresor Melee está ejecutando su animación de ataque (`ATTACK`) y la hitbox está activa,
CUANDO Edrick conecta un golpe de tipo `fire` (debilidad del Agresor) en ese mismo frame,
ENTONCES el estado transiciona a `STAGGERED` inmediatamente; la hitbox se desactiva; el Agresor no aplica daño en ese swing.
Assert: `current_state == STAGGERED`; `apply_damage` no invocado por el Agresor en ese swing.

**AC-AI-058 — STAGGERED: no se extiende con golpes adicionales de debilidad**
DADO que un Agresor Melee está en `STAGGERED` con 0.2s restantes,
CUANDO Edrick conecta otro golpe de tipo `fire`,
ENTONCES el daño se aplica con `mult_debilidad=1.5` pero el timer de `STAGGERED` NO se reinicia; sigue desde 0.2s.
Assert: `stagger_timer_remaining ≈ 0.2s` (continúa desde el valor previo); no se extiende ni se reinicia.

**AC-AI-059 — STAGGERED: guard-break del Defensivo tras 3 ataques ligeros consecutivos**
DADO que un Defensivo está en `PURSUE` con `light_attack_hits_absorbed == 0`,
CUANDO Edrick conecta 3 ataques ligeros consecutivos contra el Defensivo (sin gap > 2.0s entre golpes y sin que el Defensivo ataque),
ENTONCES al tercer golpe el Defensivo transiciona a `STAGGERED` por `guard_stagger_duration` (0.6s).
Assert: `current_state == STAGGERED` tras tercer golpe; `stagger_timer == 0.6s`; `light_attack_hits_absorbed` reseteado a 0.

**AC-AI-060 — STAGGERED: guard_stagger más largo que debilidad_stagger**
DADO que el Defensivo tiene `stagger_duration=0.4s` (por debilidad) y `guard_stagger_duration=0.6s` (por guard-break),
CUANDO el Defensivo entra en `STAGGERED` por guard-break (3 ataques ligeros) vs. por debilidad (hit de tipo `corruption`),
ENTONCES la duración del stagger es 0.6s en guard-break y 0.4s en debilidad.
Assert: `stagger_timer == 0.6` en guard-break; `stagger_timer == 0.4` en debilidad-hit.

**AC-AI-061 — Fórmula 6: multiplicador de debilidad correcto**
DADO que un Agresor Melee tiene HP=40 y `debilidad_tipo=fire`,
CUANDO Edrick conecta un ataque con `tipo_daño=fire` y `daño_edrick=10`,
ENTONCES el daño aplicado es `round(10 × 1.5) = 15 HP`.
Assert: `enemy.hp_before - enemy.hp_after == 15`.

**AC-AI-062 — Fórmula 6: sin multiplicador fuera de debilidad**
DADO que un Agresor Melee tiene `debilidad_tipo=fire`,
CUANDO Edrick conecta un ataque con `tipo_daño=physical` (no es la debilidad) y `daño_edrick=10`,
ENTONCES el daño aplicado es `round(10 × 1.0) = 10 HP`; el enemigo NO entra en `STAGGERED`.
Assert: `enemy.hp_before - enemy.hp_after == 10`; `current_state != STAGGERED`; `enemy_staggered` no emitido.

---

### Notas de Implementación para QA

- **AC-AI-019–AC-AI-023** (oportunismo con `edrick_in_recovery`) requieren que `CombatSystem.edrick_in_recovery` sea accesible sincrónicamente cada frame. Si se implementa como señal en lugar de propiedad, el oportunismo necesita un buffer de último estado — ajustar asserts a frame T+1.
- **AC-AI-037** asume que `CombatSystem.edrick_combat_state` es una propiedad pública accesible cada frame, no una señal de evento. Si cambia a señal asíncrona, ajustar el assert a frame T+1.
- **AC-AI-054–AC-AI-062** (Sistema de Debilidades y STAGGERED) son determinísticos y automatizables. No requieren RNG ni mocks.
- Todos los demás criterios son determinísticos y automatizables como unit tests en `tests/unit/enemy_ai/`.

## Open Questions

1. **API de lectura de edrick_combat_state** *(Owner: gameplay-programmer, resolver en Architecture)* — El GDD asume que `CombatSystem.edrick_combat_state` es una propiedad pública accesible cada frame. Si el diseño arquitectural usa señales (event-driven), el sistema de oportunismo necesita un buffer de último estado. Resolver antes de escribir el código del state machine.

2. **Comportamiento en plataformas con gaps** *(Owner: level-designer + ai-programmer, resolver en Vertical Slice)* — El pathfinding simple con `move_and_slide()` no prevé saltos entre plataformas. Los enemigos caen en gaps y pueden quedar atrapados. Para el MVP (Reino 1), el level design debe evitar gaps entre plataformas donde haya enemigos. A largo plazo, considerar `NavigationAgent2D` con navigation mesh para enemigos en entornos más complejos.

3. **Comportamiento del Hostigador Distancia en plataformas elevadas** *(Owner: ai-programmer, resolver en Pre-Production)* — Si el Hostigador está en una plataforma superior y Edrick en una inferior (o vice versa), ¿apunta el proyectil directamente o lo lanza horizontalmente? El GDD actual especifica "hacia `edrick.global_position` en el momento del disparo" — verificar en playtest si esto crea proyectiles con ángulos de caída impredecibles.

4. **Coordinación entre múltiples enemigos del mismo tipo** *(Owner: game-designer, diferido a Vertical Slice)* — El MVP define enemigos como independientes. Explorar si en zonas posteriores se quiere "flanking" básico (dos Agresores que no atacan desde el mismo lado). No para MVP.

5. **`attack_anim_duration` provisional (0.30s)** *(Owner: technical-artist + ai-programmer, resolver antes del Vertical Slice)* — El valor 0.30s es una estimación placeholder. Todos los TTK de Fórmula 3 dependen de este valor. Debe confirmarse con la spec de animación por arquetipo antes del Vertical Slice — cualquier cambio requiere actualizar la tabla de §4.4 y los asserts de AC-AI-032.

6. **Asignación automática de `debilidad_tipo` del Hostigador** *(Owner: gameplay-programmer, resolver en Pre-Production)* — El GDD establece que `debilidad_tipo` del Hostigador se asigna automáticamente como el tipo opuesto a su `tipo_daño`. ¿Se calcula en `_ready()` del script del enemigo, o se setea manualmente en la escena por el level designer? Si es automático, verificar que el level designer no pueda override accidentalmente. Si es manual, agregar validación en editor (tool script).
