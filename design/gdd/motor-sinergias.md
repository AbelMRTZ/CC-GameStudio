# GDD: Motor de Sinergias

> **Estado**: En Revisión (revisión 3 — 8 bloqueantes en resolución, 2026-05-29)
> **Autor**: Abel + Claude Code Agents
> **Última Actualización**: 2026-05-29 (Revisión 3: coste visible Arcano -3 HP MVP, notificación HUD sinergia positiva, F-SE-07 precondición mente_fuego gate + max(0.1) guard, arcano_mente keys impulsividad, mente_gato §5 contradicción, 7 ACs reasignados, CA-MSE-019c/036/037 nuevos, ACs 002/014/021/029 especificados)
> **Implements Pillar**: Pilar 4 — Sinergia y Libertad de Construcción
> **Milestone**: MVP — Core Layer
> **Depende de**: Base de Datos de Demonios (#3), Loadout & Build Management (#10)
> **Dependen de este sistema**: Vinculación de Demonios (#13), Bestiario (#19)

---

## 1. Visión General

El Motor de Sinergias es el evaluador central que determina, en tiempo real, qué combinaciones de demonios están produciendo efectos sinérgicos y cuáles se están anulando mutuamente. Internamente, es un sistema de evaluación de reglas: cuando el Loadout emite `loadout_changed`, el Motor compara el array `equipped_demons` contra la tabla completa de sinergias definida en la Base de Datos de Demonios (#3) y produce un conjunto de salidas — `active_synergies`, `synergy_multiplier` para el cálculo de daño, y señales de activación para efectos especiales como las estelas de Dash. Desde la perspectiva del jugador, el Motor no es visible directamente: su existencia se percibe cuando la quemadura de Fuego se vuelve más letal con Arcano equipado, cuando el Dash deja una estela de llamas, o cuando Fuego e Hielo juntos se sienten decepcionantemente mediocres. El Motor convierte las decisiones de build del jugador en consecuencias mecánicas inmediatas y consistentes, haciendo que la experimentación con sinergias sea confiable y significativa.

---

## 2. Fantasy del Jugador

El Motor de Sinergias no tiene una fantasía propia — tiene la responsabilidad de hacer que la fantasía del Loadout sea real. GDD #10 define el momento objetivo: *el jugador dice "¡lo encontré!" y a la vez siente "¿pero a qué costo?"* El Motor de Sinergias es la garantía implícita de ese momento: hace que cada combinación de demonios produzca resultados consistentes, deterministas y descubribles. Sin él, experimentar con builds sería apostar en la oscuridad. Con él, el jugador puede confiar en que si Arcano + Fuego funcionó bien en un encuentro, funcionará igual en el siguiente — y ese contrato de confianza es lo que convierte el descubrimiento accidental en conocimiento acumulable. La fantasía que el Motor habilita (indirectamente) es la del jugador-arqueólogo: alguien que excava una combinación, la prueba, la confirma, y la colecciona como si fuera suya. El Bestiario después la nombra. El Motor es quien garantiza que tenga nombre.

---

## 3. Diseño Detallado

### 3.1 Reglas Centrales

**A. Inicialización**

1. El Motor de Sinergias es un **Autoload singleton** (`SynergyEngine`) que se inicializa en `_ready()`.
2. Al inicializarse, carga el archivo `assets/data/demons/synergies.json` — la tabla completa de reglas de sinergia. Esta carga ocurre una sola vez al arrancar el juego; el archivo no se recarga durante la sesión.
3. El Motor se suscribe al EventBus para escuchar `loadout_changed`. Inicializa su caché con `equipped_demons = []` y `active_synergies = []`.

**B. Evaluación al cambiar loadout**

4. Cada vez que llega `loadout_changed(equipped_demons: Array[String])`, el Motor:
   - Limpia el caché de sinergias anterior.
   - Recorre **todas** las reglas en `synergies.json` (sin orden fijo — el orden es irrelevante).
   - Para cada regla, verifica si `required_demons ⊆ equipped_demons`.
   - Si se cumple: marca la sinergia como activa y calcula sus valores de efecto.
   - Al finalizar: publica el nuevo estado cacheado y emite `synergies_updated`.
5. La evaluación es **determinista**: el mismo `equipped_demons` produce siempre el mismo resultado.
6. No hay evaluación lazy — todas las reglas se evalúan en cada `loadout_changed`, sin importar qué demonio cambió.

**C. Reglas de precedencia de sinergias múltiples**

7. Múltiples sinergias pueden estar activas simultáneamente. No se cancelan entre sí — coexisten y aplican de forma aditiva/multiplicativa según el efecto.
8. **Arcano no acumula sobre sí mismo**: Arcano amplifica ×1.25 cada demonio co-equipado de forma independiente. El Motor computa una entrada de amplificación por par `(arcano, demon_X)`. Arcano nunca se amplifica a sí mismo.
9. **Sinergias positivas y negativas coexisten**: Si Dash+Fuego (positiva) y Fuego+Hielo (negativa) están activos simultáneamente, ambas aplican. Los valores base de los efectos positivos se calculan primero; la penalización negativa se aplica sobre esos valores resultantes. (Ver §5 para el caso específico Dash+Fuego+Hielo.)
10. **Sinergias Gato no interfieren con sinergias mecánicas**: Las sinergias narrativas del Gato coexisten sin afectar cálculos mecánicos.

**D. Interfaz pública — Pull API**

11. El Motor expone las siguientes funciones de solo lectura, llamables por cualquier sistema en cualquier momento:

| Función | Retorno | Descripción |
|---------|---------|-------------|
| `get_synergy_multiplier() → float` | 1.00 (MVP) | Multiplicador global de daño. Hook para sinergias futuras que afecten daño base. |
| `is_synergy_active(synergy_id: String) → bool` | true/false | Verdadero si la sinergia con ese ID está activa en el loadout actual. |
| `get_active_synergies() → Array[String]` | Array de IDs | Lista de todos los IDs de sinergias activas en este momento. |
| `get_synergy_effect(synergy_id: String, effect_key: String) → Variant` | Variant o null | Valor de un efecto específico de una sinergia activa. Null si inactiva. |

12. El Motor no emite señales de resultado en el EventBus, **excepto**: `synergies_updated(active_synergies: Array[String])` al finalizar cada evaluación — para que el Bestiario actualice su UI sin polling.
13. El Motor no escribe en ningún otro sistema. Es de solo salida.

**E. Amplificación de Arcano**

14. Cuando `is_synergy_active("arcano_fuego")` es verdadero, el Motor expone vía `get_synergy_effect("arcano_fuego", "quemadura_rate_multiplier")` el valor 1.25. Combate lee ese valor al calcular la quemadura. El Motor no calcula el daño de quemadura — solo provee el factor de amplificación.
15. Los `daño_bonus` base de cada demonio **NO son amplificados por Arcano** — son aditivos en `mod_atacante_formula` sin multiplicación de sinergia. El Motor nunca modifica `daño_bonus`.

---

### 3.2 Estados y Transiciones

| Estado | Descripción | Cuándo ocurre |
|--------|-------------|---------------|
| **INITIALIZING** | Carga `synergies.json`, suscripción al EventBus, caché inicializado vacío | `_ready()` — ~1 frame |
| **EVALUATING** | Procesando `loadout_changed` — O(n) con n ≤ 20 reglas en MVP | < 1 ms por llamada |
| **READY** | Caché estable. Responde pull requests instantáneamente. | Entre evaluaciones |

No existe estado "inválido": el Motor siempre tiene un estado cacheado válido (inicialmente con `active_synergies = []`).

---

### 3.3 Interacciones con Otros Sistemas

| Sistema | Dirección | Qué fluye | Cuándo |
|---------|-----------|-----------|--------|
| **Loadout (#10)** | Loadout → Motor | `loadout_changed(equipped_demons: Array[String])` vía EventBus | Al finalizar `SWAP_ANIM_DURATION` |
| **Base de Datos (#3)** | BD → Motor | `synergies.json` — tabla de reglas | Una vez al inicializar |
| **Combate (#6)** | Motor ← Combate | `get_synergy_multiplier()`, `is_synergy_active()`, `get_synergy_effect()` | Al calcular daño, al ejecutar dash |
| **Progresión Narrativa (#16)** | Motor ← PN | `is_synergy_active("mente_gato")`, `"vision_gato"`, `"mente_vision_gato"` | Al evaluar desbloqueos narrativos |
| **HUD de Combate (#18)** | Motor → HUD | `synergies_updated(active_synergies)` — HUD detecta la primera vez por sesión que una sinergia de tipo `"negative"` o `"positive"` aparece en el array y muestra una notificación breve: sinergias negativas → "Anulación Térmica activa"; sinergias positivas → "[Demonio A] amplifica [Demonio B]" (nombre legible, no el ID). La lógica de "primera vez por sesión" vive en el HUD, no en el Motor. Motor emite la señal; HUD es responsable del tracking de vistas. | Tras cada evaluación que incluya una sinergia nueva (positive o negative) |
| **Bestiario (#19)** | Motor → Bestiario | `synergies_updated(active_synergies)` vía EventBus | Tras cada evaluación |

*Interfaz provista hacia GDD #19 (Bestiario)*: el Bestiario registra sinergias "descubiertas" cuando el jugador las experimenta. Escucha `synergies_updated` y filtra IDs no vistos antes. La persistencia de "sinergias descubiertas" vive en Estado del Mundo (#4), no en el Motor.

---

### 3.4 Esquema de `synergies.json`

Formato canónico del archivo de datos. El Motor carga este archivo completo en `_ready()` y no tolera entradas con formato incorrecto — el Motor entra en modo degradado si el archivo no es parseable como JSON válido.

**Estructura de cada entrada:**

```json
{
  "id":              "String — único, snake_case, formato: demon1_demon2 o demon1_demon2_gato",
  "required_demons": ["Array[String] — IDs de demonios en equipped_demons. Orden irrelevante (evaluado como conjunto)."],
  "required_gato":   "bool — si se omite, el Motor lo trata como false",
  "type":            "String — 'positive' | 'negative' | 'narrative'. Solo informativo para HUD/Bestiario; no afecta la evaluación del Motor.",
  "effects":         {
    "effect_key": "Float | Int — valor expuesto por get_synergy_effect(id, key)"
  }
}
```

**Ejemplo mínimo funcional (5 entradas representativas):**

```json
{
  "synergies": [
    {
      "id": "arcano_fuego",
      "required_demons": ["arcano", "fuego"],
      "required_gato": false,
      "type": "positive",
      "effects": {
        "quemadura_rate_multiplier": 1.25,
        "quemadura_duracion_multiplicador": 1.25
      }
    },
    {
      "id": "arcano_mente",
      "required_demons": ["arcano", "mente"],
      "required_gato": false,
      "type": "positive",
      "effects": {
        "esquivas_bonus": 1,
        "impulsividad_cooldown_multiplier": 1.25,
        "impulsividad_duration_multiplier": 1.25
      }
    },
    {
      "id": "fuego_hielo",
      "required_demons": ["fuego", "hielo"],
      "required_gato": false,
      "type": "negative",
      "effects": {
        "anulacion_factor": 0.85,
        "friccion_penalty": 0.15
      }
    },
    {
      "id": "arcano_vision",
      "required_demons": ["arcano", "vision"],
      "required_gato": false,
      "type": "positive",
      "effects": {
        "hp_penalty_bonus": -1
      }
    },
    {
      "id": "mente_gato",
      "required_demons": ["mente"],
      "required_gato": true,
      "type": "narrative",
      "effects": {}
    }
  ]
}
```

**Reglas del esquema:**

- `effects: {}` es válido — la sinergia se activa pero no expone efectos queryables (uso: sinergias narrativas puras).
- Valores Float: rates (HP/s), multiplicadores, porcentajes como fracciones (0.85, no 85).
- Valores Int: bonos/penalizaciones discretas (dodge_count_bonus, hp_penalty_bonus).
- El Motor no valida tipos de valores en runtime — el contenido de `effects` es correcto por contrato con el diseño.
- `required_gato` puede omitirse si es false — el Motor usa false como valor por defecto.
- `type` no tiene efecto en la evaluación del Motor; existe para que HUD y Bestiario puedan categorizar sinergias sin lógica adicional.

---

## 4. Fórmulas

### F-SE-01: Evaluación de Sinergia — Condición de Activación

La fórmula que determina si una sinergia se activa dado el loadout actual:

```
is_active(synergy_id) = (required_demons[synergy_id] ⊆ equipped_demons)
                      ∧ (required_gato[synergy_id] → gato_available)
```

**Variables:**

| Variable | Símbolo | Tipo | Rango | Descripción |
|----------|---------|------|-------|-------------|
| Demonios requeridos por la regla | `required_demons[id]` | Set[String] | subconjunto de IDs de demonio | Demonios que deben estar en el loadout para que la sinergia se active |
| Demonios equipados actualmente | `equipped_demons` | Set[String] | tamaño 0–5 | IDs de demonios activos en el loadout (excluyendo Gato) |
| Gato requerido por la regla | `required_gato[id]` | bool | {true, false} | Si la regla requiere que el Gato esté disponible |
| Disponibilidad del Gato | `gato_available` | bool | {true, false} | Recibido en la señal `loadout_changed`. Verdadero si Estado del Mundo marca al Gato como presente |
| Sinergia activa | `is_active` | bool | {true, false} | Resultado determinista de la evaluación |

**Rango de salida:** binario. La evaluación es O(1) por regla; O(n) total con n ≤ 20 reglas en MVP.

**Ejemplos:**
- `equipped_demons = {dash, fuego, hielo}`, `gato_available = false`:
  - `is_active("dash_fuego")` = {dash, fuego} ⊆ {dash, fuego, hielo} = **true**
  - `is_active("mente_gato")` = {mente} ⊆ {dash, fuego, hielo} = **false** (mente no equipada)
  - `is_active("fuego_hielo")` = {fuego, hielo} ⊆ {dash, fuego, hielo} = **true**
- `equipped_demons = {mente, vision}`, `gato_available = true`:
  - `is_active("mente_gato")` = {mente} ⊆ {mente, vision} ∧ `gato_available` = **true**
  - `is_active("mente_vision_gato")` = {mente, vision} ⊆ {mente, vision} ∧ `gato_available` = **true**

> **Nota de implementación**: La señal `loadout_changed` extiende su firma a `loadout_changed(equipped_demons: Array[String], gato_available: bool)`. GDD #10 (Loadout) §3.3 debe actualizarse para reflejar este payload. El Motor nunca consulta Estado del Mundo directamente para el estado del Gato — lo recibe pasivamente en la señal.

---

### F-SE-02: Amplificación de Arcano sobre Efectos Especiales

Cuando `is_active("arcano_X")` es verdadero para un demonio X co-equipado:

```
efecto_amplificado = efecto_base × ARCANO_AMP
```

**Variables:**

| Variable | Símbolo | Tipo | Rango | Descripción |
|----------|---------|------|-------|-------------|
| Valor base del efecto | `efecto_base` | float | específico por efecto | Valor sin amplificación (rate, duración, %) |
| Factor de amplificación de Arcano | `ARCANO_AMP` | float | [1.15, 1.35]; MVP = 1.25 | Tuning knob. NO hardcodeado |
| Valor amplificado | `efecto_amplificado` | float | `efecto_base × [1.15, 1.35]` | Output entregado al sistema consumidor |

**Efectos por sinergia Arcano+X (MVP):**

| Sinergia | Efecto | `efecto_base` | `efecto_amplificado` (×1.25) |
|----------|--------|---------------|-------------------------------|
| Arcano+Fuego | burn_rate (pasiva) | 3.0 HP/s | **3.75 HP/s** |
| Arcano+Fuego | burn_duration | 3.0 s | **3.75 s** |
| Arcano+Hielo | freeze_duration | 2.0 s | **2.5 s** |
| Arcano+Mente | dodge_count_bonus | +0 | **+1** (entero aditivo, NO ×1.25) |
| Arcano+Dash | dash_speed | 400 px/s | **500 px/s** |
| Arcano+Dash | dash_cooldown | −0.1 s (reducción plana) | **−0.1 s** (NO amplificado) |
| Arcano+X | resistencia | `resistencia_base` | `resistencia_base × 1.25`, clamp [-0.50, +0.50] |
| Arcano+Visión | hp_penalty_bonus | −5 HP (stat base de Visión, no amplificado) | **−1 HP adicional** (int aditivo — HP máximo total con Visión+Arcano = −6) |

**Regla de decisión — qué amplifica Arcano (categorías):**

| Categoría | Descripción | Tratamiento | Ejemplo |
|-----------|-------------|-------------|---------|
| **Magnitud amplificable** | Rates (HP/s), porcentajes, multiplicadores de velocidad, valores de resistencia/debilidad | `efecto_base × ARCANO_AMP` | burn_rate, freeze_duration, resistencia, velocity_multiplier |
| **Bonus entero discreto** | Contadores donde ×1.25 produce fracción sin sentido mecánico | Aditivo fijo (int) — NO ×ARCANO_AMP | dodge_count_bonus (+1), hp_penalty_bonus (−1) |
| **Delta plano fijo** | Reducciones fijas bloqueadas por GDDs upstream — no escalan | Sin amplificación | dash_cooldown_reduction (−0.1 s, fijo en GDD #1/GDD #3) |
| **Excluido del Motor** | Stats base del demonio en `modificadores_stats` — aplicados por Loadout/Health, no por el Motor | Motor nunca los expone | daño_bonus, hp_bonus (stats base — cada demonio los aplica directamente) |

> **Nota sobre Arcano+Visión**: `hp_penalty_bonus = -1` es un efecto de sinergia, no la amplificación del `hp_bonus = -5` base de Visión. Loadout aplica ambos por separado: −5 (stat de Visión, siempre) + −1 (sinergia Arcano+Visión, solo si activa) = −6 HP máximo total.

**Qué NO amplifica Arcano:**
- `daño_bonus` de ningún demonio (sumatorio aditivo en `mod_atacante_formula` — no se toca)
- El propio `daño_bonus` de Arcano (+0.20): Arcano no se amplifica a sí mismo
- Al Gato (restricción arquitectural de GDD #3 §3.2)
- El dodge_count_bonus: es entero aditivo fijo (+1), no fracción amplificada

**Amplificación de debilidades (espada de doble filo):**

```
resistencia_amplificada = resistencia_base × ARCANO_AMP
resistencia_final       = clamp(resistencia_amplificada, -0.50, +0.50)
```

Ejemplo — Arcano+Hielo (Hielo tiene +0.15 vulnerabilidad a Fuego):
- `resistencia_amplificada = +0.15 × 1.25 = +0.1875`
- `resistencia_final = clamp(+0.1875, -0.50, +0.50)` → **+0.19** (Edrick recibe 19% más daño de Fuego)

---

### F-SE-03: "Anulación Térmica" — Penalización por Fuego+Hielo

Una sola entrada `"fuego_hielo"` en `synergies.json` con dos efectos simultáneos:

```
// Efecto A: Penalización de efectividad (aplicada a magnitudes de Fuego y de Hielo)
efecto_penalizado = efecto_base × ANULACION_FACTOR

// Efecto B: Penalización de velocidad (aplicada al multiplicador de velocidad combinado)
velocity_multiplier_final = max(VELOCITY_MULT_MIN,
                               velocity_multiplier_combined - FRICCION_PENALTY)
```

**Variables:**

| Variable | Símbolo | Tipo | Rango | Descripción |
|----------|---------|------|-------|-------------|
| Factor de penalización | `ANULACION_FACTOR` | float | [0.75, 0.90]; MVP = 0.85 | Fracción de efectividad que se retiene |
| Penalización de velocidad | `FRICCION_PENALTY` | float | [0.10, 0.20]; MVP = 0.15 | Resta plana del multiplicador de velocidad combinado |
| Multiplicador combinado | `velocity_multiplier_combined` | float | [0.30, 2.0] | Producto de `velocidad_multiplicador_i` de todos los demonios equipados |
| Velocidad mínima | `VELOCITY_MULT_MIN` | float | 0.30 (bloqueado, GDD #3 §4.4) | Multiplicador mínimo absoluto |

**Definición formal de magnitud vs duración:** Una *magnitud* es cualquier valor que representa la *intensidad* de un efecto: rate (HP/s), porcentaje de reducción (%), multiplicador de velocidad, count de esquivas. Una *duración* es el tiempo en segundos que el efecto persiste una vez aplicado. `ANULACION_FACTOR` penaliza solo la intensidad — cómo de fuerte golpea el efecto — sin acortar el tiempo que dura. Arcano, por su parte, amplifica tanto magnitudes como duraciones (ver F-SE-02).

**Qué penaliza `ANULACION_FACTOR`:** solo **magnitudes** de efectos especiales (rates, porcentajes de slow, count de esquivas). NO penaliza duraciones. NO penaliza `daño_bonus`.

**Ejemplo — Fuego+Hielo equipados:**

| Efecto | Sin penalización | Con penalización (×0.85) |
|--------|-----------------|--------------------------|
| Quemadura de Fuego | 3.0 HP/s | **2.55 HP/s** |
| Duración quemadura | 3.0 s | 3.0 s (**no afectada**) |
| Duración congelación Hielo | 2.0 s | 2.0 s (**no afectada**) |
| Estela Ardiente (Dash+Fuego) | 2.0 HP/s | **1.70 HP/s** |
| Slow de Estela Congelada | 40% | **34%** |
| Velocidad combinada (Fuego×1.0 × Hielo×0.90) | 0.90× → 225 px/s | 0.90 − 0.15 = 0.75× → **187.5 px/s** |

---

### F-SE-04: Estelas de Dash — Valores de Trail con Modificadores Apilados

Orden de operaciones: **sinergias positivas primero → penalizaciones negativas encima**.

```
estela_burn_rate = ESTELA_ARDIENTE_BASE_RATE × arcano_fuego_mod × anulacion_mod

estela_slow_pct  = ESTELA_CONGELADA_BASE_SLOW × anulacion_mod

// donde:
arcano_fuego_mod = ARCANO_AMP  si is_active("arcano_fuego")  si no 1.0
anulacion_mod    = ANULACION_FACTOR  si is_active("fuego_hielo")  si no 1.0
```

**Variables:**

| Variable | Símbolo | Tipo | Rango | Descripción |
|----------|---------|------|-------|-------------|
| Rate base de quemadura de estela | `ESTELA_ARDIENTE_BASE_RATE` | float | [1.0, 3.0] HP/s; MVP = 2.0 | DoT al contacto con la estela |
| Duración del DoT en el enemigo | `ESTELA_ARDIENTE_BASE_DURATION` | float | [1.5, 2.5] s; MVP = 2.0 | Duración del burn aplicado al enemigo |
| Duración de la zona de estela | `ESTELA_ARDIENTE_TRAIL_DURATION` | float | [0.5, 1.0] s; MVP = 0.8 | Tiempo que la zona de trail permanece activa |
| Longitud de la estela | `ESTELA_TRAIL_LENGTH` | int | [80, 150] px; MVP = 120 | Largo del hitbox en la dirección del dash |
| Porcentaje base de slow | `ESTELA_CONGELADA_BASE_SLOW` | float | [0.30, 0.50]; MVP = 0.40 | Fracción de reducción de velocidad del enemigo |
| Duración del slow en el enemigo | `ESTELA_CONGELADA_BASE_SLOW_DURATION` | float | [1.0, 2.0] s; MVP = 1.5 | Duración del slow |

**Tabla de resultados — los 4 escenarios posibles de trail:**

| Escenario | Burn Rate Estela | Slow % |
|-----------|-----------------|--------|
| Dash+Fuego solo | 2.0 HP/s | — |
| Dash+Hielo solo | — | 40% |
| Dash+Fuego+Hielo (triple) | 2.0 × 0.85 = **1.70 HP/s** | 40% × 0.85 = **34%** |
| Arcano+Dash+Fuego+Hielo (cuádruple) | 2.0 × 1.25 × 0.85 = **2.13 HP/s** | 40% × 0.85 = **34%** |

> Arcano no amplifica la Estela Congelada — Arcano+Hielo amplifica `freeze_duration` (congelación directa), no el slow porcentual de la Estela. La Estela Congelada es sinergia Dash+Hielo; Arcano solo aplica al par que lo incluye directamente.

---

### F-SE-05: Velocidad — Dash y Movimiento Normal

**F-SE-05a — Arcano+Dash: velocidad de dash y cooldown (solo aplica durante el estado DASH):**

```
dash_speed_final      = DASH_SPEED_BASE × ARCANO_AMP
dash_cooldown_final   = DASH_COOLDOWN_BASE − arcano_dash_cd_reduction
```

| Variable | Valor | Fuente |
|----------|-------|--------|
| `DASH_SPEED_BASE` | 400 px/s | entities.yaml (bloqueado) |
| `DASH_COOLDOWN_BASE` | 0.6 s | entities.yaml (bloqueado) |
| `ARCANO_AMP` | 1.25 | F-SE-02 |
| `arcano_dash_cd_reduction` | 0.1 s (reducción plana, no derivada de ARCANO_AMP) | GDD #3 §3.4 |
| **`dash_speed_final`** | **500 px/s** | — |
| **`dash_cooldown_final`** | **0.5 s** | — |

**F-SE-05b — Fricción Extrema: velocidad de movimiento normal (solo si `is_active("fuego_hielo")`):**

```
velocity_multiplier_combined = ∏ velocidad_multiplicador_i  (todos los demonios equipados)
velocity_multiplier_final    = max(VELOCITY_MULT_MIN,
                                   velocity_multiplier_combined − FRICCION_PENALTY)
velocidad_final              = velocidad_base × velocity_multiplier_final
```

| Variable | Valor | Fuente |
|----------|-------|--------|
| `velocidad_base` | 250 px/s | entities.yaml (bloqueado) |
| `VELOCITY_MULT_MIN` | 0.30 | GDD #3 §4.4 (bloqueado) |
| `FRICCION_PENALTY` | 0.15 | F-SE-03 |

**Ejemplo — Fuego+Hielo equipados:** `(1.0 × 0.90) − 0.15 = 0.75` → `250 × 0.75 = **187.5 px/s**`

> F-SE-05a afecta velocidad durante el dash. F-SE-05b afecta velocidad caminando. No interactúan: un jugador con Arcano+Fuego+Hielo dashea a 500 px/s pero camina a 187.5 px/s.

---

### F-SE-06: synergy_multiplier — Multiplicador Global de Daño

```
synergy_multiplier = 1.00  // constante en MVP
                           // post-MVP: f(active_synergies, combo_count) — reservado
```

| Variable | Tipo | Rango | Descripción |
|----------|------|-------|-------------|
| `synergy_multiplier` | float | MVP = 1.00; post-MVP ≤ 2.00 est. | Multiplicador global sobre `(∑ demon_modifier_i)` en `mod_atacante_formula` |

En MVP, ninguna sinergia afecta el daño base directamente a través de `synergy_multiplier`. Toda la amplificación de Arcano opera a través de efectos especiales (F-SE-02), no de este multiplicador.

**Ejemplo — Arcano+Fuego+Dash equipados:**
- `mod_atacante = (0.20 + 0.15 + 0.00) × 1.00 = 0.35`
- Con `daño_base = 10`: `daño_final = round(10 × 1.35 × (1 + resist))`

> **Alineación cross-GDD (GDD #6 Combate)**: GDD #6 Fórmula 4.1 contiene un ejemplo desactualizado — `synergy_multiplier = ×1.15` para Dash+Fuego — escrito antes de que la Pull API del Motor fuera establecida. En MVP `synergy_multiplier = SynergyEngine.get_synergy_multiplier()` que devuelve **siempre 1.00**. GDD #6 debe actualizar: (1) la descripción de `synergy_multiplier` para indicar que es 1.00 MVP y se obtiene vía `get_synergy_multiplier()`; (2) su ejemplo de Fórmula 4.1 para eliminar ×1.15; (3) añadir una referencia a F-SE-04 de este GDD como el lugar donde se define el orden de operaciones para efectos especiales de sinergia (DoT de estela, amplificaciones de Arcano). Esta corrección fue coordinada con el propietario de GDD #6 — revisión 2026-05-28.

---

### F-SE-07: Predecir Movimiento — Cooldown y Duración bajo Sinergias Apiladas

```
// Precondición: IMPULSIVIDAD solo aplica si la sinergia mente_fuego está activa
impulsividad_active      = is_active("mente_fuego")

predecir_cooldown_final  = PREDECIR_BASE_COOLDOWN
                         + (IMPULSIVIDAD_COOLDOWN_ADD × arcano_impulsividad_mod  si impulsividad_active,  si no 0)

predecir_duration_final  = max(0.1,
                             PREDECIR_BASE_DURATION
                             − (IMPULSIVIDAD_DURATION_REDUCTION × arcano_impulsividad_mod  si impulsividad_active,  si no 0))

dodge_count_final        = max(0,
                             PREDECIR_BASE_DODGES
                             + arcano_dodge_bonus
                             − corrupcion_dodge_penalty)
```

| Variable | Valor base | Rango seguro | Descripción |
|----------|-----------|--------------|-------------|
| `PREDECIR_BASE_COOLDOWN` | 5.0 s | [3.0, 7.0] s | Cooldown base de Predecir Movimiento (GDD #3 §7.4) |
| `IMPULSIVIDAD_COOLDOWN_ADD` | +2.0 s | [+1.0, +3.0] s | Penalización de cooldown por Mente+Fuego |
| `arcano_impulsividad_mod` | 1.25 si `is_active("arcano_mente")`, si no 1.0 | — | Arcano amplifica la penalización de Impulsividad — **solo cuando `impulsividad_active = true`** |
| `PREDECIR_BASE_DURATION` | 5.0 s est. | [3.0, 7.0] s | Duración de la ventana de esquiva |
| `IMPULSIVIDAD_DURATION_REDUCTION` | 1.0 s | [0.5, 1.5] s | Reducción de duración de la ventana de Predecir Movimiento por Mente+Fuego. Amplificada por `arcano_impulsividad_mod` |
| `PREDECIR_BASE_DODGES` | 4 | [3, 6] | Esquivas base de Predecir Movimiento |
| `arcano_dodge_bonus` | +1 si `is_active("arcano_mente")`, si no 0 | — | Bonus entero aditivo (no ×1.25) |
| `corrupcion_dodge_penalty` | **+3** si `is_active("corrupcion_mente")`, si no 0 | — | Magnitud positiva — la fórmula la **resta** (`− corrupcion_dodge_penalty`). Penalización futura (demonio Corrupción — no en MVP); `max(0, ...)` garantiza que dodge_count_final ≥ 0 |

**Ejemplos:**

| Escenario | `impulsividad_active` | Cooldown final | Duración final | Esquivas |
|-----------|----------------------|----------------|----------------|----------|
| Solo Mente (base) | false | 5.0 s | 5.0 s | 4 |
| Mente+Arcano (sin Fuego) | **false** | 5.0 + 0 = **5.0 s** | 5.0 − 0 = **5.0 s** | 4 + 1 = **5** |
| Mente+Fuego (Impulsividad) | true | 5.0 + 2.0×1.0 = **7.0 s** | 5.0 − 1.0×1.0 = **4.0 s** | 4 |
| Mente+Fuego+Arcano | true | 5.0 + 2.0×1.25 = **7.5 s** | 5.0 − 1.0×1.25 = **3.75 s** | 4 + 1 = **5** |

---

## 5. Casos Extremos

- **Si `equipped_demons` está vacío**: El Motor limpia `active_synergies = []`, `synergy_multiplier = 1.00`, y emite `synergies_updated([])`. Todos los `is_synergy_active()` devuelven `false`. Sistema retorna a estado base limpio.

- **Si un demonio requerido por una sinergia se desequipa**: La sinergia deja de ser activa en la próxima evaluación de `loadout_changed`. Los efectos ya aplicados a enemigos (ej: quemadura activa) persisten hasta expirar — el Motor no tiene control sobre efectos ya aplicados en Combate.

- **Si `gato_available` cambia de `true` a `false` (Gato desaparece narrativamente)**: El Loadout debe emitir `loadout_changed` con `gato_available = false` cuando Estado del Mundo actualiza el estado del Gato. **Responsabilidad del Loadout**: escuchar cambios de `gato_available` en Estado del Mundo y re-emitir `loadout_changed` con el mismo `equipped_demons` y el nuevo valor de `gato_available`, aunque no haya habido swap de demonio. **Temporización de desactivación**: la desactivación es **instantánea** — sin periodo de gracia. El Motor procesa el `loadout_changed` inmediatamente, incluso mid-combat. El jugador puede percibir la pérdida del Gato en el mismo frame del evento narrativo (ej: `is_synergy_active("mente_gato")` pasa de `true` a `false`, lo que GDD #16 Progresión Narrativa interpreta como bloqueo de contenido gateado por esa sinergia — el Motor no gestiona la consecuencia narrativa; GDD #16 sí). Esto es intencional: la desaparición del Gato es un momento de peso que debe sentirse.

- **Si Arcano está equipado sin ningún otro demonio**: `active_synergies = []`. Arcano sin co-equipados no activa ninguna sinergia Arcano+X. No hay auto-amplificación. `get_synergy_effect("arcano_X", ...)` devuelve `null` para cualquier X.

- **Si Arcano está equipado solo con el Gato como "co-demonio"**: Arcano NO amplifica al Gato. `is_active("arcano_gato")` no existe como regla en `synergies.json` — cualquier consulta retorna `false`. El Motor no tiene esta regla.

- **Si `synergies.json` no se puede cargar al inicializar**: El Motor entra en modo degradado: `active_synergies = []` permanente, todos los `is_synergy_active()` devuelven `false`. El juego funciona sin sinergias. El Motor logea el error pero no hace crash. Esta condición solo es posible por corrupción de datos — en producción normal no ocurre.

- **Si `loadout_changed` llega mientras el Motor está evaluando (reentrancia)**: En MVP, la evaluación es sincrónica y completa en < 1ms. No existe evaluación concurrente en GDScript single-threaded. Reentrancia no es posible.

- **Si dos sinergias afectan el mismo recurso (ej: burn rate) con valores distintos**: Los sistemas consumidores llaman `get_synergy_effect(synergy_id, effect_key)` con el ID específico de cada sinergia. No hay colisión — las keys son únicas por par (synergy_id, effect_key). Ejemplo: `get_synergy_effect("arcano_fuego", "burn_rate_multiplier")` y `get_synergy_effect("dash_fuego", "estela_burn_rate")` son consultas distintas. Combate combina los valores según F-SE-04.

- **Si Dash+Fuego+Hielo están equipados simultáneamente**: Tres sinergias activas: `is_active("dash_fuego") = true`, `is_active("dash_hielo") = true`, `is_active("fuego_hielo") = true`. El Motor las reporta todas. Combate aplica F-SE-04 (positivos primero, penalización encima). Ver GDD #3 §5.11.

- **Si Mente+Fuego+Arcano están equipados (cooldown máximo de Predecir Movimiento)**: Cooldown = 7.5 s, duración = 4.0 s, esquivas = 5. Este es el caso más severo en MVP y es intencional: el jugador amplificó su penalización al elegir Arcano.

---

## 6. Dependencias

### 6.1 Dependencias Salientes (este sistema necesita)

| Sistema | Tipo | Qué necesita |
|---------|------|-------------|
| **Base de Datos de Demonios (#3)** | Dura | `assets/data/demons/synergies.json` — tabla de reglas de sinergia cargada en `_ready()`. Sin este archivo el Motor entra en modo degradado (todos los `is_synergy_active()` devuelven `false`, juego funciona sin sinergias). Los valores de efectos base (burn_rate, freeze_duration, estela params) vienen de GDD #3 §4.6 y §7. |
| **Loadout & Build Management (#10)** | Dura | Señal `loadout_changed(equipped_demons: Array[String], gato_available: bool)` vía EventBus. Es el único trigger de reevaluación del Motor. El Loadout también es responsable de re-emitir `loadout_changed` cuando `gato_available` cambia (por evento narrativo) sin que haya habido un swap de demonio. |

### 6.2 Dependencias Entrantes (dependen de este sistema)

| Sistema | Qué espera del Motor |
|---------|----------------------|
| **Combate en Tiempo Real (#6)** | Pull de `get_synergy_multiplier()`, `is_synergy_active()`, `get_synergy_effect()` al calcular daño y al ejecutar el dash. Combate aplica los valores — el Motor no ejecuta lógica de combate. |
| **Vinculación de Demonios (#13)** | Puede consultar `is_synergy_active()` para determinar si el momento de binding activa una sinergia nueva visible al jugador. Dependencia suave. |
| **Bestiario (#19)** | Escucha `synergies_updated(active_synergies: Array[String])` vía EventBus para actualizar sinergias "descubiertas". La persistencia de sinergias descubiertas vive en Estado del Mundo (#4), no en el Motor. |
| **Progresión Narrativa (#16)** | Pull de `is_synergy_active("mente_gato")`, `"vision_gato"`, `"mente_vision_gato"` para desbloquear contenido narrativo gateado por sinergia. El Motor solo detecta presencia de condición mecánica — la Progresión Narrativa decide qué contenido ejecutar y si los gates de acto están cumplidos. |

### 6.3 Bidireccionalidad

- **Loadout (#10) → Motor**: unidireccional. Loadout emite `loadout_changed`; Motor recibe y recalcula. Motor no escribe en Loadout.
- **Motor ← Combate (#6)**: unidireccional. Combate hace pull API. Motor no sabe cuándo Combate calcula daño.
- **Motor → Bestiario (#19)**: unidireccional. Motor emite `synergies_updated`; Bestiario escucha.
- **Motor ← Progresión Narrativa (#16)**: unidireccional. PN consulta pull API. Motor no sabe de actos o narrativa.
- **Motor ← Base de Datos (#3)**: unidireccional. Motor lee `synergies.json` en init. BD no sabe del Motor.

### 6.4 Sistemas que NO son responsabilidad del Motor

- **"Extinción" (Fuego+Agua)**: interacción ambiente-demonio. Agua no es un demonio con ID en `synergies.json`. Manejada por el sistema de entorno/Combate. El Motor no tiene regla `"fuego_agua"`.
- **Cálculo de resistencias combinadas**: vive en GDD #3 §4.4 y se aplica en Combate. El Motor no computa resistencias.
- **Persistencia de sinergias descubiertas**: vive en Estado del Mundo (#4). El Motor es stateless respecto a historial.

---

## 7. Parámetros de Ajuste

Todos los valores deben ser extrínsecos en `assets/data/demons/synergies.json` y en un archivo de config de tuning knobs. Nada hardcodeado en GDScript.

| Parámetro | Valor Base | Rango Seguro | Qué rompe si... |
|-----------|-----------|--------------|-----------------|
| `ARCANO_AMP` | 1.25 | [1.15, 1.35] | **< 1.15**: Arcano deja de sentirse como amplificador — no vale el costo de corrupción Tier S. **> 1.35**: Arcano domina cualquier build, trivializa builds sin él. |
| `ANULACION_FACTOR` | 0.85 | [0.75, 0.90] | **< 0.75**: Fuego+Hielo se vuelven prácticamente inútiles juntos — el jugador evita siempre la combinación. **> 0.90**: La penalización es tan suave que deja de ser un tradeoff visible. |
| `FRICCION_PENALTY` | 0.15 | [0.10, 0.20] | **< 0.10**: La ralentización no se nota. **> 0.20**: Con Hielo ya en 0.90× y sustrayendo 0.20, el jugador alcanza 0.70× (175 px/s) — demasiado lento para combate ágil. |
| `ESTELA_ARDIENTE_BASE_RATE` | 2.0 HP/s | [1.0, 3.0] HP/s | **< 1.0**: Estela es visual sin impacto. **> 3.0**: Junto con Arcano amplificado (3.75 HP/s), supera el daño del ataque normal — near-dominant. |
| `ESTELA_ARDIENTE_BASE_DURATION` | 2.0 s | [1.5, 2.5] s | **< 1.5**: La zona desaparece antes de que la mayoría de enemigos la toquen. **> 2.5**: Control de área dominante. |
| `ESTELA_TRAIL_LENGTH` | 120 px | [80, 150] px | **< 80**: No cubre la distancia del dash, pierde utilidad. **> 150**: Hitbox demasiado grande, difícil de evitar para el diseño de niveles. |
| `ESTELA_CONGELADA_BASE_SLOW` | 0.40 | [0.30, 0.50] | **< 0.30**: Slow inapreciable. **> 0.50**: Junto a otros efectos, el enemigo queda casi detenido — trivializa arquetipos de movilidad (hostigador). |
| `IMPULSIVIDAD_COOLDOWN_ADD` | 2.0 s | [+1.0, +3.0] s | **< +1.0**: La penalización de Mente+Fuego no disuade la combinación. **> +3.0**: Cooldown de 8+ s hace a Predecir Movimiento inútil con Impulsividad activa. |
| `synergy_multiplier` (post-MVP hook) | 1.00 (fijo MVP) | [1.00, 2.00] post-MVP | MVP no modifica este valor. Post-MVP: **> 2.00** rompe el rango estimado de `mod_atacante`. |

> **Nota de balance — Coste MVP de Arcano (decisión 2026-05-29)**: Arcano lleva un `hp_bonus: −3` que reduce el HP máximo de Edrick (análogo al `hp_bonus: −5` de Visión). Este coste es visible e inmediato — el jugador ve el HP reducido al equipar Arcano y elige conscientemente entre amplificación de efectos (×1.25) y HP máximo reducido. Con Arcano+Visión combinados, el HP máximo cae −8 (y −9 con la sinergia `arcano_vision` activa), creando un tradeoff legible sin GDD #22. GDD #3 §3.3 y `entities.yaml` han sido actualizados para reflejar este penalty. Cuando GDD #22 sea implementado, la corrupción Tier S actuará como coste moral adicional. Ver PQ-MSE-03 en §9 (resuelto).

---

## 8. Criterios de Aceptación

> **Convención de keys**: las claves de efectos en `synergies.json` y en las llamadas a `get_synergy_effect()` usan **español** (ej: `quemadura_rate_multiplier`, `congelacion_duracion_multiplicador`). `get_synergy_effect()` retorna `null` para cualquier key que no exista en la sinergia consultada.

---

### Inicialización y estado base

- **CA-MSE-001**: Motor arranca en READY con caché vacío
  - *Dado*: Motor inicializa (`_ready()`), `synergies.json` carga correctamente
  - *Cuando*: `get_active_synergies()`, `get_synergy_multiplier()`
  - *Entonces*: `[]` y `1.00` respectivamente; Motor en estado READY

- **CA-MSE-002**: Loadout vacío → sin sinergias activas
  - *Dado*: Motor en estado READY con `synergies.json` cargado correctamente; `loadout_changed([], false)`
  - *Cuando*: `is_synergy_active("dash_fuego")`, `is_synergy_active("arcano_fuego")`, `get_active_synergies()`
  - *Entonces*: `false` para ambos IDs; `get_active_synergies()` = `[]`

### Activación básica de sinergias

- **CA-MSE-003**: Sinergia básica activa cuando subconjunto cumple
  - *Dado*: `loadout_changed(["dash", "fuego"], false)`
  - *Cuando*: `is_synergy_active("dash_fuego")`
  - *Entonces*: retorna `true`

- **CA-MSE-004**: Sinergia no activa cuando falta demonio requerido
  - *Dado*: `loadout_changed(["dash"], false)`
  - *Cuando*: `is_synergy_active("dash_fuego")`
  - *Entonces*: retorna `false`

### Gate del Gato

- **CA-MSE-005**: Gato gate — sinergia bloqueada si `gato_available = false`
  - *Dado*: `loadout_changed(["mente"], false)` (Mente equipada, Gato no disponible)
  - *Cuando*: `is_synergy_active("mente_gato")`
  - *Entonces*: retorna `false`

- **CA-MSE-006**: Gato gate — sinergia activa si `gato_available = true`
  - *Dado*: `loadout_changed(["mente"], true)` (Mente equipada, Gato disponible)
  - *Cuando*: `is_synergy_active("mente_gato")`
  - *Entonces*: retorna `true`

- **CA-MSE-030**: Triple Gato (Mente+Visión+Gato) activa las tres sinergias Gato
  - *Dado*: `loadout_changed(["mente", "vision"], true)`
  - *Cuando*: `is_synergy_active("mente_gato")`, `is_synergy_active("vision_gato")`, `is_synergy_active("mente_vision_gato")`
  - *Entonces*: las tres retornan `true`; Motor activa subconjuntos y el supraconjunto simultáneamente

- **CA-MSE-030b**: Triple Gato — ninguna sinergia Gato activa si `gato_available = false`
  - *Dado*: `loadout_changed(["mente", "vision"], false)` (mismos demonios, Gato no disponible)
  - *Cuando*: `is_synergy_active("mente_gato")`, `is_synergy_active("vision_gato")`, `is_synergy_active("mente_vision_gato")`
  - *Entonces*: las tres retornan `false`; el gate de `required_gato` bloquea las tres sinergias

### Sinergia `fuego_hielo` (Anulación Térmica + Fricción Extrema)

- **CA-MSE-007**: `fuego_hielo` activa ambos efectos simultáneamente
  - *Dado*: `loadout_changed(["fuego", "hielo"], false)`
  - *Cuando*: `get_synergy_effect("fuego_hielo", "anulacion_factor")` y `get_synergy_effect("fuego_hielo", "friccion_penalty")`
  - *Entonces*: retornan `0.85` y `0.15` respectivamente; ninguno es null

### Amplificación de Arcano

- **CA-MSE-008**: Arcano+Fuego amplifica `quemadura_rate_multiplier` a 1.25
  - *Dado*: `loadout_changed(["arcano", "fuego"], false)`
  - *Cuando*: `get_synergy_effect("arcano_fuego", "quemadura_rate_multiplier")`
  - *Entonces*: retorna `1.25`

- **CA-MSE-009**: Arcano NO expone `daño_bonus` — retorna null
  - *Dado*: `is_synergy_active("arcano_fuego") = true`
  - *Cuando*: `get_synergy_effect("arcano_fuego", "daño_bonus")`
  - *Entonces*: retorna `null` (daño_bonus es aditivo en `mod_atacante_formula`, no es efecto de sinergia)

- **CA-MSE-010**: Arcano+Mente: `esquivas_bonus = 1` (entero, no ×1.25)
  - *Dado*: `loadout_changed(["arcano", "mente"], false)`
  - *Cuando*: `get_synergy_effect("arcano_mente", "esquivas_bonus")`
  - *Entonces*: retorna `1` (int), **no** `1.25`; Combate suma +1 al count de esquivas base

- **CA-MSE-011**: Arcano+Dash: `cooldown_reduccion_plana = 0.1 s` (no amplificado)
  - *Dado*: `loadout_changed(["arcano", "dash"], false)`
  - *Cuando*: `get_synergy_effect("arcano_dash", "cooldown_reduccion_plana")`
  - *Entonces*: retorna `0.1`, **no** `0.125` (= 0.1 × 1.25)

- **CA-MSE-012**: Arcano+Hielo: `congelacion_duracion_multiplicador = 1.25`
  - *Dado*: `loadout_changed(["arcano", "hielo"], false)`
  - *Cuando*: `get_synergy_effect("arcano_hielo", "congelacion_duracion_multiplicador")`
  - *Entonces*: retorna `1.25`; Combate aplica `2.0 s × 1.25 = 2.5 s`

- **CA-MSE-013**: Arcano+Hielo expone `congelacion_duracion_multiplicador = 1.25`; clave de resistencia no expuesta por el Motor
  - *Dado*: `loadout_changed(["arcano", "hielo"], false)`
  - *Cuando*: `get_synergy_effect("arcano_hielo", "congelacion_duracion_multiplicador")` y `get_synergy_effect("arcano_hielo", "resistencia_fuego")`
  - *Entonces*: primer retorna `1.25`; segundo retorna `null` — el Motor no expone valores de resistencia base de GDD #3; la amplificación `clamp(resistencia × 1.25, -0.50, +0.50)` es responsabilidad de Combate (verificada en tests de integración)

- **CA-MSE-031**: Arcano+Dash expone `velocidad_multiplicador = 1.25` para el dash
  - *Dado*: `loadout_changed(["arcano", "dash"], false)`
  - *Cuando*: `get_synergy_effect("arcano_dash", "velocidad_multiplicador")`
  - *Entonces*: retorna `1.25`; Combate deriva `dash_speed_final = 400 × 1.25 = 500 px/s`

### Multiplicador global y señal de actualización

- **CA-MSE-014**: `get_synergy_multiplier()` siempre retorna 1.00 en MVP
  - *Dado*: (caso A) Motor en READY con `loadout_changed([], false)`; (caso B) `loadout_changed(["arcano", "fuego", "dash", "hielo", "mente"], false)` (máximo loadout MVP con sinergias múltiples activas)
  - *Cuando*: `get_synergy_multiplier()` en cada caso
  - *Entonces*: retorna exactamente `1.00` en ambos casos; no varía por número de sinergias activas

- **CA-MSE-015**: `synergies_updated` emitida exactamente una vez por `loadout_changed`
  - *Dado*: Motor en estado READY
  - *Cuando*: se recibe `loadout_changed`
  - *Entonces*: `synergies_updated` se emite exactamente 1 vez al finalizar la evaluación; nunca antes de completar; nunca más de una vez por evento

- **CA-MSE-028**: `synergies_updated` contiene los IDs correctos de sinergias activas
  - *Dado*: `loadout_changed(["dash", "fuego"], false)`
  - *Cuando*: `synergies_updated(active_synergies)` es recibida por Bestiario
  - *Entonces*: `active_synergies` contiene exactamente los IDs que se cumplen para ese loadout (ej: `["dash_fuego"]`)

- **CA-MSE-029**: `get_active_synergies()` es coherente con el último `synergies_updated`
  - *Dado*: Motor en estado READY tras `loadout_changed(["arcano", "fuego"], false)`
  - *Cuando*: se compara `get_active_synergies()` con el payload del último `synergies_updated`
  - *Entonces*: ambos contienen exactamente `["arcano_fuego"]`; no existe caché de señal distinto del caché pull

### Orden de operaciones — estelas apiladas

- **CA-MSE-016**: Triple Dash+Fuego+Hielo — Motor activa las tres sinergias y expone el factor de anulación
  - *Dado*: `loadout_changed(["dash", "fuego", "hielo"], false)`
  - *Cuando*: `is_synergy_active("dash_fuego")`, `is_synergy_active("dash_hielo")`, `is_synergy_active("fuego_hielo")`, `get_synergy_effect("fuego_hielo", "anulacion_factor")`
  - *Entonces*: las tres `is_synergy_active` retornan `true`; `anulacion_factor = 0.85`
  - *Nota*: El burn rate final (`2.0 × 1.0 × 0.85 = 1.70 HP/s`) es responsabilidad de Combate — verificado en test de integración Motor+Combate

- **CA-MSE-017**: Cuádruple Arcano+Dash+Fuego+Hielo — Motor activa cuatro sinergias y expone ambos factores
  - *Dado*: `loadout_changed(["arcano", "dash", "fuego", "hielo"], false)`
  - *Cuando*: `is_synergy_active("arcano_fuego")`, `is_synergy_active("dash_fuego")`, `is_synergy_active("dash_hielo")`, `is_synergy_active("fuego_hielo")`, `get_synergy_effect("arcano_fuego", "quemadura_rate_multiplier")`, `get_synergy_effect("fuego_hielo", "anulacion_factor")`
  - *Entonces*: las cuatro `is_synergy_active` retornan `true`; `quemadura_rate_multiplier = 1.25`; `anulacion_factor = 0.85`
  - *Nota*: El burn rate final (`2.0 × 1.25 × 0.85 = 2.125 HP/s`) es responsabilidad de Combate — verificado en test de integración

- **CA-MSE-018**: Arcano+Hielo no expone `estela_slow_multiplier` — solo amplifica duración de congelación directa
  - *Dado*: `loadout_changed(["arcano", "dash", "hielo"], false)` — `is_synergy_active("arcano_hielo") = true`
  - *Cuando*: `get_synergy_effect("arcano_hielo", "estela_slow_multiplier")` y `get_synergy_effect("dash_hielo", "estela_slow_pct")`
  - *Entonces*: primer retorna `null` (Arcano+Hielo no tiene esa key — Arcano solo amplifica `congelacion_duracion_multiplicador`); segundo retorna `0.40` — sin modificación por Arcano

### Predecir Movimiento apilado

- **CA-MSE-019**: Mente+Fuego+Arcano — Motor activa ambas sinergias y expone multiplicadores de Impulsividad
  - *Dado*: `loadout_changed(["mente", "fuego", "arcano"], false)`
  - *Cuando*: `is_synergy_active("mente_fuego")`, `is_synergy_active("arcano_mente")`, `get_synergy_effect("arcano_mente", "esquivas_bonus")`, `get_synergy_effect("arcano_mente", "impulsividad_cooldown_multiplier")`, `get_synergy_effect("arcano_mente", "impulsividad_duration_multiplier")`
  - *Entonces*: `is_synergy_active("mente_fuego") = true`; `is_synergy_active("arcano_mente") = true`; `esquivas_bonus = 1` (int); `impulsividad_cooldown_multiplier = 1.25`; `impulsividad_duration_multiplier = 1.25`
  - *Nota*: Los valores finales (`cooldown = 7.5 s`, `duración = 3.75 s`) son calculados por Combate usando F-SE-07 — verificados en test de integración

- **CA-MSE-020**: Arcano+Mente SIN Fuego — `mente_fuego` inactivo; Impulsividad no aplica
  - *Dado*: `loadout_changed(["mente", "arcano"], false)` — Fuego NO equipado
  - *Cuando*: `is_synergy_active("arcano_mente")`, `is_synergy_active("mente_fuego")`
  - *Entonces*: `is_synergy_active("arcano_mente") = true`; `is_synergy_active("mente_fuego") = false`; la fórmula F-SE-07 con `impulsividad_active = false` produce cooldown = 5.0 s (sin penalización) — Combate NO aplica `IMPULSIVIDAD_COOLDOWN_ADD`

### Casos de borde de Arcano

- **CA-MSE-021**: Arcano equipado solo → `active_synergies = []`
  - *Dado*: `loadout_changed(["arcano"], false)`
  - *Cuando*: `get_active_synergies()`, `is_synergy_active("arcano_fuego")`, `is_synergy_active("arcano_hielo")`, `is_synergy_active("arcano_mente")`
  - *Entonces*: `get_active_synergies()` = `[]`; las tres `is_synergy_active` retornan `false`; Arcano sin co-equipados no activa ninguna sinergia

- **CA-MSE-022**: `is_synergy_active("arcano_gato")` siempre `false`
  - *Dado*: `loadout_changed(["arcano"], true)` (Arcano equipado, Gato disponible)
  - *Cuando*: `is_synergy_active("arcano_gato")`
  - *Entonces*: `false`; la regla `"arcano_gato"` no existe en `synergies.json`

### Propiedades del Motor

- **CA-MSE-023**: Determinismo — mismo loadout produce siempre el mismo resultado
  - *Dado*: `loadout_changed(["arcano", "fuego"], false)` evaluado en dos momentos distintos
  - *Cuando*: se comparan los `get_active_synergies()` de ambas evaluaciones
  - *Entonces*: arrays idénticos en contenido; sin variación por orden de evaluación ni estado previo

- **CA-MSE-024**: De-equipar un demonio invalida sus sinergias en el siguiente evento
  - *Dado*: `is_synergy_active("dash_fuego") = true` (Dash+Fuego equipados)
  - *Cuando*: `loadout_changed(["fuego"], false)` (Dash removido)
  - *Entonces*: `is_synergy_active("dash_fuego") = false` tras la nueva evaluación; el caché anterior fue invalidado

### Robustez y manejo de errores

- **CA-MSE-025**: `is_synergy_active(id_desconocido)` retorna `false` sin crash
  - *Dado*: `"sinergia_inventada_xyz"` no existe en `synergies.json`
  - *Cuando*: `is_synergy_active("sinergia_inventada_xyz")`
  - *Entonces*: `false`; no lanza excepción; Motor no hace crash

- **CA-MSE-026**: `get_synergy_effect(id_desconocido, key)` retorna `null` sin crash
  - *Dado*: ID o key que no existe en `synergies.json`
  - *Cuando*: `get_synergy_effect("sinergia_inventada_xyz", "efecto_xyz")`
  - *Entonces*: `null`; no lanza excepción; Motor no hace crash

- **CA-MSE-027**: Modo degradado — fallo de `synergies.json` → sin sinergias, sin crash
  - *Dado*: `synergies.json` ausente o corrupto al inicializar
  - *Cuando*: Motor completa `_ready()`
  - *Entonces*: `active_synergies = []` permanente; `is_synergy_active()` siempre `false`; juego arranca sin sinergias; ninguna excepción no manejada; Motor logea el error

### Determinismo extendido

- **CA-MSE-032**: Determinismo — resultado independiente del orden de `equipped_demons`
  - *Dado*: `loadout_changed(["dash", "fuego"], false)` evaluado; a continuación `loadout_changed(["fuego", "dash"], false)` evaluado
  - *Cuando*: se comparan los `get_active_synergies()` de ambas evaluaciones
  - *Entonces*: los arrays contienen exactamente los mismos IDs; el orden del input no afecta el resultado

### arcano_mente — keys de Impulsividad

- **CA-MSE-037**: `arcano_mente` expone los tres efectos correctos
  - *Dado*: `loadout_changed(["arcano", "mente"], false)`
  - *Cuando*: `get_synergy_effect("arcano_mente", "esquivas_bonus")`, `get_synergy_effect("arcano_mente", "impulsividad_cooldown_multiplier")`, `get_synergy_effect("arcano_mente", "impulsividad_duration_multiplier")`
  - *Entonces*: retornan `1` (int), `1.25` (float), `1.25` (float) respectivamente; ninguno retorna `null`

### Transición narrativa del Gato

- **CA-MSE-033**: Desactivación instantánea al cambiar `gato_available` → false
  - *Dado*: `loadout_changed(["mente"], true)` — `is_synergy_active("mente_gato") = true`
  - *Cuando*: `loadout_changed(["mente"], false)` (mismo `equipped_demons`, `gato_available` cambia)
  - *Entonces*: `is_synergy_active("mente_gato") = false`; `get_active_synergies()` no incluye `"mente_gato"`

### Arcano+Visión

- **CA-MSE-034**: Arcano+Visión expone `hp_penalty_bonus = −1`
  - *Dado*: `loadout_changed(["arcano", "vision"], false)`
  - *Cuando*: `is_synergy_active("arcano_vision")` y `get_synergy_effect("arcano_vision", "hp_penalty_bonus")`
  - *Entonces*: `is_synergy_active` retorna `true`; `get_synergy_effect` retorna `−1` (int); Loadout aplica HP máximo adicional de −1 sobre el −5 base de Visión (total −6 HP máximo adicional cuando ambos activos)

### Criterios de aceptación — HUD y Playtest (Advisory)

- **CA-MSE-035a**: [MANUAL/HUD] Sinergia negativa notificada al confirmar loadout — primera vez por sesión
  - *Dado*: Motor en READY; HUD sin `"fuego_hielo"` registrado en esta sesión (first-time flag false); `loadout_changed(["fuego", "hielo"], false)`
  - *Cuando*: HUD recibe `synergies_updated(["fuego_hielo"])` y detecta `type = "negative"` primera vez en sesión
  - *Entonces*: HUD muestra "Anulación Térmica activa" dentro de 1 frame de recibir `synergies_updated`; la notificación NO aparece en evaluaciones posteriores del mismo loadout en la misma sesión
  - *Nivel de gate*: ADVISORY — walkthrough manual verificado por QA

- **CA-MSE-035b**: [PLAYTEST] Sinergia negativa reconocida como intencional
  - *Dado*: sesión de playtest con jugador no informado de la mecánica de anulación
  - *Cuando*: jugador equipa Fuego+Hielo y experimenta la penalización en combate al menos una vez
  - *Entonces*: en cuestionario post-sesión de opción múltiple, el jugador selecciona "los demonios interactuaron / tuvieron conflicto" sobre "algo falló o el juego tuvo un bug"; umbral de paso: ≥ 2 de cada 3 testers en el mismo grupo
  - *Nivel de gate*: ADVISORY — requiere protocolo de playtest formalizado; no bloquea implementación

- **CA-MSE-036**: [MANUAL/HUD] Primera sinergia positiva activada en sesión — HUD notifica al jugador
  - *Dado*: Motor en READY; HUD sin `"arcano_fuego"` registrado en esta sesión; `loadout_changed(["arcano", "fuego"], false)`
  - *Cuando*: HUD recibe `synergies_updated(["arcano_fuego"])` y detecta `type = "positive"` primera vez en sesión
  - *Entonces*: HUD muestra notificación breve (ej. "Arcano amplifica Fuego") dentro de 1 frame de recibir `synergies_updated`; la notificación NO aparece en evaluaciones posteriores del mismo loadout en la misma sesión
  - *Nivel de gate*: ADVISORY — walkthrough manual; lógica "primera vez por sesión" vive en HUD, no en Motor

---

## 9. Preguntas Abiertas

- **~~PQ-MSE-01~~** ✅ **RESUELTO** — GDD #10 fue actualizado el 2026-05-28. La firma `loadout_changed(equipped_demons: Array[String], gato_available: bool)` ya está documentada en GDD #10 §3.1.F (Regla 23) y §3.3 (interfaz pública). No hay acción pendiente.

- **PQ-MSE-02: Diseño de `synergy_multiplier` post-MVP** — F-SE-06 reserva `synergy_multiplier` como hook para post-MVP (rango estimado ≤ 2.00, actualmente constante en 1.00). Candidatos para alimentarlo: número de sinergias activas simultáneamente, contador de combo activo de Combate, o nivel de corrupción de Edrick. Este diseño debe madurar antes del Vertical Slice.

- **~~PQ-MSE-03~~** ✅ **RESUELTO** — Decisión 2026-05-29: Arcano lleva `hp_bonus: −3` (GDD #3 §3.3 actualizado). El tradeoff amplificación-vs-HP es visible en MVP sin GDD #22. Ver §7 para detalle completo.

- **~~PQ-MSE-04~~** ✅ **RESUELTO PARCIALMENTE** — La temporización de desactivación está definida como instantánea (ver §5). El comportamiento del Motor ante el cambio de `gato_available` es completamente especificado. **Pendiente en GDD #4**: qué evento narrativo concreto dispara el cambio de disponibilidad del Gato. Esta pendiente no bloquea la implementación del Motor ni del Loadout — el canal `notify_gato_available(bool)` en GDD #10 §3.1.F está especificado; solo falta que GDD #4 defina qué escena/evento lo invoca.
