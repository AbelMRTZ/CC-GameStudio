# GDD: Sistema de NPC y Diálogo

> **Status**: En Revisión
> **Autor**: Abel + agentes
> **Última Actualización**: 2026-05-27
> **Sistema**: NPC y Diálogo (#15)
> **Milestone**: MVP — Feature Layer
> **Implementa Pilar**: Pilar 1 — Narrativa Cinematográfica; Pilar 5 — Transformación Moral de Edrick
> **Depende de**: Estado del Mundo (#4)
> **Dependen de este sistema**: Progresión Narrativa (#16), Seguimiento Moral (#22)

---

## Overview

El Sistema de NPC y Diálogo es el **conducto primario** mediante el cual Dravaryn cuenta su historia. Cuando el jugador se acerca a un NPC y presiona [E], el sistema activa una conversación: bloquea el movimiento, consulta el Estado del Mundo para mostrar la rama de diálogo apropiada a ese momento narrativo (basada en demonios obtenidos, decisiones morales pasadas y relación acumulada con ese NPC), muestra el texto con identidad visual, registra cuál rama el jugador vio, y emite señales hacia Progresión Narrativa y Seguimiento Moral. Lo crucial: los NPCs no son solo dispensadores de lore — son **personajes recurrentes con arcos narrativos propios**. Algunos aparecen una sola vez (encuentro casual); otros reaparecen a lo largo del viaje, y su opinión de Edrick evoluciona conforme el jugador toma decisiones oscuras o nobles. El jugador puede empatizar con un NPC, traicionarlo, amarlo o temerlo — cada conversación amplía o refractura esa relación. El arco moral de Edrick está entrelazado con los NPCs que conoce: son testigos de su transformación, reflejos de sus elecciones, y a veces, espejos que le muestran lo que se está convirtiendo. Sin este sistema, Dravaryn sería un mundo vacío; con él, cada conversación ancla la narrativa en relaciones humanas (aunque algunos de esos "humanos" sean inteligencias demoníacas).

---

## Player Fantasy

Cuando Edrick habla con alguien — sea un aliado recurrente o un extraño encontrado una sola vez — el jugador debería sentir dos emociones entrelazadas: **descubrimiento del mundo** y **peso de la relación**. El descubrimiento viene de que cada NPC revela un pedazo de Dravaryn — su historia, sus miedos, sus secretos. El peso viene de notar que ese NPC tiene una opinión de Edrick *basada en lo que el jugador ha hecho*. Un NPC que antes saludaba amistosamente ahora lo mira con miedo (porque Edrick ha vinculado demonios oscuros). Un aliado que el jugador traicionó en misión anterior ahora rehúsa ayudar. Esos momentos de **reconocimiento** — "oh, sí, ellos me ven así porque yo elegí esto" — son el corazón de la fantasía. La conversación no es una cinemática que el jugador ve pasivamente; es un acto narrativo donde el jugador siente las consecuencias de sus elecciones *reflejadas en los ojos de alguien más*. Los NPCs recurrentes especialmente sirven como **espejos morales**: el jugador regresa a ellos y ve cómo su relación ha evolucionado, a veces mejorando (redención), a veces deteriorándose (caída). Y en esos momentos quietos — entre combate, entre crisis — el jugador tiene la oportunidad de conocer a alguien, de importarle, de reflexionar sobre quién se está convirtiendo. Sin eso, son solo ficheros de datos; con eso, son personajes que el jugador lamentará haber perdido (o amará haber destruido).


---

## Detailed Design

### Core Rules

**R1 — Existencia de NPCs**
Los NPCs existen como nodos `Area2D` con tipo `NPC` distribuidos en las zonas de Dravaryn. Cada NPC tiene un `npc_id: String` único (ej: "elder_vorn", "merchant_kess", "soldier_ren") y un conjunto de ramas de diálogo predefinidas (no generadas proceduralmente). Los NPCs están **pre-poblados** en el diccionario `npc_encounters` del Estado del Mundo desde el inicio (ver GDD #4 §3.2).

**R2 — Activación de Diálogo**
Cuando Edrick se acerca a un NPC (entra al Area2D) y presiona [E]:
1. El sistema bloquea el input del jugador (estado `INTERACTING`)
2. Consulta `world_state.npc_encounters[npc_id]` para obtener:
   - `met: bool` — ¿ha visto Edrick a este NPC antes?
   - `dialogue_branches_seen: list[str]` — qué ramas de diálogo ya ha visto
   - `reputation: float` — relación acumulada con este NPC (rango -1.0 a +1.0). Si `met == false`, aplicar F3 antes de continuar: la reputación inicial puede ser −0.2 si `corruption_level ≥ CORRUPTION_THRESHOLD_FOR_PENALTY`
   - `noble_streak: int` — número de ramas nobles consecutivas sin interacción oscura (usado para histéresis de umbral en R3)
   - `alive: bool` — ¿sigue vivo este NPC? (si es false, no activar diálogo; mostrar escena alternativa)
3. Basado en estos valores, selecciona una rama de diálogo disponible (lógica en R3)
4. Muestra la rama en la UI de diálogo (ver UI Requirements, sección H)
5. Al terminar, emite `interaccion_completada(npc_id)` y restaura el input (sección F)

**Input buffer de activación**: Para evitar que el mismo press de [E] que activa el diálogo salte automáticamente la primera línea de texto, el sistema ignora inputs de avance durante los primeros `DIALOGUE_ACTIVATION_BUFFER_FRAMES` frames tras abrir la UI (ver Tuning Knobs).

**Nota sobre interacciones forzadas**: En ciertos momentos narrativos vitales, el juego puede **forzar** una interacción (cinemática obligatoria donde Edrick DEBE hablar con un NPC específico). Estas interacciones omiten el paso [E] — se activan automáticamente cuando se cumplen las condiciones de la trama, bloquean input, muestran la rama designada, y restauran input después. El jugador no puede rechazarlas; son script de la historia, no interacciones opcionales.

**R3 — Selección de Rama de Diálogo**
Cada NPC tiene un árbol de ramas etiquetadas por condiciones.

**Paso previo — Filtrado por F2**: Antes de aplicar la lógica de prioridad, el sistema construye un pool de ramas candidatas aplicando F2 a todas las ramas del NPC (excluye ramas con `min_reputation` mayor a la reputación actual). La prioridad de R3 opera dentro de ese pool filtrado.

Si el pool del rango de reputación seleccionado queda vacío tras el filtrado por F2, el sistema prueba el siguiente rango en orden descendente (ej: cordial → neutral → hostil) antes de recurrir al DEFAULT. Esto evita que el jugador reciba ramas contradictorias con su reputación visible.

La selección sigue esta lógica de prioridad:
- **Rama de momento narrativo fijo**: Si la trama requiere que este NPC diga algo específico en un momento concreto (ej: "NPC_REVEAL_act_2_scene_3"), esa rama se muestra **siempre** en prioridad sobre reputación y contexto. Las ramas fijas **deben autorarse con sub-variantes por tier de reputación** (ej: `NPC_REVEAL_act_2_scene_3_hostile`, `NPC_REVEAL_act_2_scene_3_neutral`, `NPC_REVEAL_act_2_scene_3_warm`) para que el beat narrativo ocurra pero el tono refleje la relación acumulada. Si el tier actual no tiene sub-variante, el sistema usa `_neutral`. **Excepción**: Si la rama fija lleva `reputation_aware: false`, se muestra sin variantes — reservado para momentos donde el tono es narrativamente insustituible (debe ser la minoría de ramas fijas).
- **Primera vez (`met: false`)**: Si no hay rama fija, muestra rama `FIRST_ENCOUNTER`. Actualiza `met: true`.
- **NPCs recurrentes**: Si el NPC aparece en múltiples zonas/momentos sin rama fija asignada, muestra rama basada en `reputation`:
  - `reputation < -0.5`: rama oscura/hostil
  - `-0.5 ≤ reputation < 0.0`: rama fría/desconfiada
  - `0.0 ≤ reputation < 0.5`: rama neutral/cordial
  - `reputation ≥ 0.5`: rama caálida/aliada
- **Contexto narrativo**: Si ciertos eventos han ocurrido (ej: "demonio_arcano_obtenido", "acto_1_completo"), muestra rama específica

**Histéresis de Umbral (Regla de Estabilización):** Para cruzar un umbral en dirección ascendente (ej: cold → neutral, neutral → warm), el jugador necesita **2 interacciones nobles consecutivas sin interacción oscura entre ellas**. El campo `noble_streak: int` en `npc_encounters[npc_id]` trackea esto: se incrementa en cada rama noble completada y se reinicia a 0 tras cualquier rama oscura. El tier visual del NPC no asciende hasta que `noble_streak ≥ 2` Y la reputación supera el umbral. El descenso (acciones oscuras) es inmediato — sin histéresis en dirección negativa.

**Primer cambio de tier — Feedback de Espejo:** Cuando la reputación de un NPC cruza un umbral por primera vez (en cualquier dirección), el sistema emite `reputation_tier_changed(npc_id, old_tier, new_tier)`. Esto permite que la UI o el sistema de diálogo muestre un indicador contextual sutil (ej: texto breve en el header del diálogo: "Elder Vorn te mira diferente ahora") conectando mecánicamente la causa con el efecto. El contenido exacto es responsabilidad del autor de la rama, pero la señal garantiza que el sistema pueda entregarlo.
- **Bloqueo de rama**: Si una rama ya fue vista (`in dialogue_branches_seen`), prioriza ramas nuevas. Si todas fueron vistas, repite una aleatoriamente
- **Fallback**: Si ninguna rama aplica, muestra rama `DEFAULT`

**R4 — Registro de Ramas Vistas**
Después de que el jugador ve una rama completa, el sistema:
1. Añade el `branch_id` a `npc_encounters[npc_id].dialogue_branches_seen`
2. Si la rama tenía una opción de elección moral (ver R5), registra en `world_state.player_choices`

**R5 — Choques Narrativos y Consecuencias Morales**
Algunas ramas de diálogo contienen **puntos de decisión** donde el jugador elige una opción de respuesta (ej: "Matarlo" vs. "Dejarlo ir"). Cuando el jugador elige:
1. El sistema registra la elección en `player_choices[decision_id] = {value, act, timestamp, conscious: true}`
2. Calcula el impacto moral: si la opción está marcada como "oscura", aumenta `corruption_floor` (ver GDD #4 §4.1.D)
3. Puede cambiar el estado del NPC: `npc_encounters[npc_id].alive = false` si el jugador eligió ejecutar
4. Emite señal `decision_made(npc_id, decision_id, choice)` hacia Progresión Narrativa (#16) y Seguimiento Moral (#22)

> **Nota sobre variables de corrupción (GDD #4):** `corruption_floor` es el mínimo permanente alcanzado por la corrupción de Edrick. Es distinto de `corruption_level`, que es el nivel actual (fluctúa con demonios equipados y recuperación pasiva). Este sistema escribe a `corruption_floor` (eleva el piso permanente) — no a `corruption_level`. El filtrado de ramas de diálogo (F2) opera sobre `reputation`, no sobre la corrupción, por lo que esta distinción no afecta la selección de ramas.

**R6 — Cambios de Reputación**
Después de cada rama de diálogo, el sistema aplica un delta de reputación Y actualiza `noble_streak`:
- Rama "acto noble extraordinario" (ej: salvar vida de un NPC): **+0.30** → `noble_streak += 1`
- Rama "noble": +0.15 a reputation → `noble_streak += 1`
- Rama "neutral": ±0.0 → `noble_streak` sin cambio
- Rama "oscura": -0.15 a reputation → `noble_streak = 0` (reinicio)
- Rama "acto oscuro narrativo" (ej: ejecutar un NPC): **-0.30** → `noble_streak = 0` (reinicio)

**Justificación**: Como los demonios equipados ya corrompen pasivamente a Edrick (GDD #6), necesita deltas simétricos en diálogos para poder recuperarse narrativamente. Sin `+0.30` noble, la caída sería inevitable y la redención imposible — esto rompería el Pilar 5 (Transformación Moral como arco gris, no nihilista).

La reputación se clampea a [-1.0, +1.0].

**R7 — Estados de Diálogo y Transiciones**
```
INACTIVE (esperando interacción [E])
    ↓ [E presionado dentro del Area2D]
INTERACTING (diálogo en progreso)
    ↓ [rama completa / skip forzado]
DONE (emite interaccion_completada, restaura input)
    ↓ [automático]
INACTIVE (listo para siguiente interacción)
```

### Interactions with Other Systems

**Estado del Mundo (#4):**
- **Lee**: `npc_encounters[npc_id]` (met, reputation, dialogue_branches_seen, alive), `major_events`, `available_demons`
- **Escribe**: `npc_encounters[npc_id].met`, `.dialogue_branches_seen[]`, `.reputation`, `.alive`; `player_choices[decision_id]`, `corruption_floor` (si acto oscuro)

**Exploración del Mundo (#8):**
- **Recibe**: signal `interaccion_completada()` desde este sistema
- **Usa**: para restaurar input después de que el diálogo termina

**Progresión Narrativa (#16):**
- **Recibe**: signal `decision_made(npc_id, decision_id, choice)`, `dialogue_branch_viewed(npc_id, branch_id)`
- **Usa**: para disparar eventos narrativos, cinemáticas, cambios de zona

**Seguimiento Moral (#22):**
- **Recibe**: signal `moral_choice(decision_id, choice, impact: int)` cuando una rama contiene acto oscuro
- **Usa**: para acumular corrupción y cambios narrativos

**Transformación Visual de Edrick (#14):**
- **Recibe**: indirectamente, vía Seguimiento Moral, cambios en `corruption_level`
- **Nota**: El sistema NPC no toca directamente esta transformación; solo alimenta los datos que la conducen

---

## Formulas

**F1 — Cálculo de Reputación**

La reputación de Edrick con cada NPC se calcula:

```
reputation_new = clamp(reputation_old + delta, -1.0, +1.0)
```

**Variables:**

| Variable | Símbolo | Tipo | Rango | Descripción |
|----------|---------|------|-------|-------------|
| Reputación anterior | reputation_old | float | [-1.0, +1.0] | Reputación acumulada antes de esta interacción |
| Delta de rama | delta | float | {-0.30, -0.15, ±0.0, +0.15, +0.30} | Cambio aplicado por esta rama (ver R6 para asignación) |
| Reputación nueva | reputation_new | float | [-1.0, +1.0] | Reputación después de aplicar delta, clampeada |

**Output Range:** [-1.0, +1.0] — valores fuera del rango se clampean automáticamente.

**Ejemplos:**
- NPC comienza en `reputation=0.0` (neutral). Rama noble (+0.15) → `reputation_new = 0.15`
- NPC en `reputation=0.9` (muy aliado). Acto oscuro narrativo (-0.30) → `reputation_new = clamp(0.9 - 0.30, -1.0, +1.0) = 0.6`
- NPC en `reputation=-0.8` (enemigo). Rama noble (+0.15) → `reputation_new = clamp(-0.8 + 0.15, -1.0, +1.0) = -0.65`
- NPC en `reputation=-0.9` (enemigo). Acto oscuro narrativo (-0.30) → `reputation_new = clamp(-0.9 - 0.30, -1.0, +1.0) = -1.0` (clampeado, no -1.20)

**Restricción anti-farming:** El delta de reputación se aplica **solo la primera vez** que el jugador ve una rama regular. Si la rama entra en fallback (repetida), el delta NO se aplica nuevamente. Esto evita farming de reputación.

**Excepción — ramas de reconciliación:** Las ramas marcadas como `reconciliation: true` en su definición **siempre aplican su delta**, independientemente de si fueron vistas antes. Estas ramas se desbloquean tras eventos narrativos específicos del arco de ese NPC. Para prevenir pump infinito de reputación, cada rama de reconciliación registra `reconciliation_applied_this_arc: bool` en su schema — cuando `true`, la rama sigue siendo visible y accesible (el jugador puede revivirla narrativamente) pero el delta NO se aplica nuevamente hasta que el campo sea reseteado por GDD #16 al inicio de un nuevo arco narrativo de ese NPC.

**Tipos válidos de trigger de reconciliación** (cada rama debe definir al menos uno):
- `event_trigger: String` — un evento de `major_events` del Estado del Mundo que debe haber ocurrido (ej: `"acto_2_completo"`)
- `moral_score_threshold: float` — nivel de puntuación moral mínimo en Seguimiento Moral (para ramas que requieren cambio genuino)
- `relationship_event: String` — evento específico de la relación con ese NPC (ej: `"elder_vorn_confrontado"`)

**Requisito de contenido:** Cada NPC recurrente debe tener al menos **2 ramas de reconciliación** diseñadas, cada una asociada a un trigger diferente del arco de ese NPC. Esto garantiza que la redención tenga camino mecánico disponible aunque el primer trigger se retrase (Pilar 5 — no nihilista).

---

**F2 — Accesibilidad de Rama por Reputación**

No todas las ramas de diálogo están disponibles en todo momento. El acceso se determina por la reputación acumulada con ese NPC:

```
branch_accessible = (reputation >= branch.min_reputation)
```

**Variables:**

| Variable | Símbolo | Tipo | Rango | Descripción |
|----------|---------|------|-------|-------------|
| Reputación actual con NPC | reputation | float | [-1.0, +1.0] | Relación acumulada con este NPC |
| Umbral mínimo de reputación de rama | branch.min_reputation | float | [-1.0, +1.0] | Edrick no puede ver esta rama si su reputación con este NPC es menor (-1.0 = sin restricción) |
| Rama accesible | branch_accessible | bool | {true, false} | Si el jugador puede activar esta rama ahora |

**Output Range:** booleano — true (rama disponible) o false (excluida por reputación insuficiente).

**Diseño deliberado:** El nivel de corrupción de Edrick NO filtra ramas de diálogo. Las consecuencias de la corrupción ya se expresan en R3 (reputación baja → ramas hostiles) y en la narrativa. Bloquear ramas de redención por corrupción alta crearía ludonarrative dissonance contra el Pilar 5 — la redención debe siempre tener un camino mecánico disponible.

**Ejemplos:**
- Rama "confesión_redención" requiere `min_reputation=0.2`.
  - Edrick con `reputation=0.4` → accesible ✓ (independientemente de su corrupción)
  - Edrick con `reputation=0.1` → NO accesible (reputación insuficiente) ✗
- Rama "traición_venganza" requiere `min_reputation=-0.7`
  - Edrick con `reputation=-0.8` → NO accesible ✗
  - Edrick con `reputation=-0.6` → accesible ✓

**Nota técnica:** Las ramas excluidas por F2 se omiten del pool de candidatas en R3. El DEFAULT debe tener `min_reputation=-1.0` (sin restricción) para garantizar que siempre haya al menos una rama disponible.

---

**F3 — Reputación Inicial por Corrupción Visible**

Cuando el jugador encuentra un NPC por primera vez (`met: false`), la reputación inicial se calcula:

```
initial_reputation = IF corruption_level >= CORRUPTION_THRESHOLD_FOR_PENALTY
                     THEN max(-1.0, 0.0 - CORRUPTION_INITIAL_REPUTATION_PENALTY)
                     ELSE 0.0
```

**Variables:**

| Variable | Símbolo | Tipo | Rango | Descripción |
|----------|---------|------|-------|-------------|
| Corrupción actual de Edrick | corruption_level | float | [0.0, 1.0] | Nivel actual de corrupción (ver GDD #4) |
| Umbral de penalización | CORRUPTION_THRESHOLD_FOR_PENALTY | float | [0.3, 0.7] | Por defecto 0.5 — cuando la corrupción supera esto, los NPCs nuevos notan el cambio |
| Penalización de reputación inicial | CORRUPTION_INITIAL_REPUTATION_PENALTY | float | [0.0, 0.3] | Por defecto 0.2 — NPCs nuevos comienzan en −0.2 cuando Edrick es visiblemente corrupto |
| Reputación inicial | initial_reputation | float | [-1.0, 0.0] | Reputación al primer encuentro, modificada por corrupción visible |

**Ejemplos:**
- Edrick con `corruption_level=0.3` (bajo umbral) → `initial_reputation = 0.0` (neutral)
- Edrick con `corruption_level=0.7` (sobre umbral) → `initial_reputation = max(-1.0, 0.0 − 0.2) = −0.2` (ligeramente desconfiado)

**Diseño deliberado:** La penalización es leve (−0.2) para no bloquear la rama `FIRST_ENCOUNTER` pero sí mostrar que el mundo reacciona a un Edrick visiblemente corrupto desde el primer encuentro (Pilar 3). El jugador recupera reputación con acciones nobles normales.

---

**F4 — Branch Schema (Especificación de Datos)**

Cada rama de diálogo debe definir los siguientes campos. Todos son requeridos salvo los marcados como opcionales:

```
branch:
  branch_id: String            # identificador único (ej: "elder_vorn_noble_02")
  npc_id: String               # referencia al NPC propietario
  branch.type: String          # "hostile" | "dark" | "neutral" | "warm" | "allied"
  min_reputation: float        # [-1.0, +1.0] — DEFAULT debe ser -1.0
  delta: float                 # DEBE pertenecer al conjunto {-0.30, -0.15, 0.0, +0.15, +0.30}
  text: String                 # contenido del diálogo
  reconciliation: bool         # false por defecto; true activa el comportamiento de Excepción en F1
  reconciliation_applied_this_arc: bool  # false por defecto; true cuando delta ya fue aplicado en este arco
  reconciliation_trigger:      # requerido si reconciliation: true
    event_trigger: String?     # opcional — evento de major_events que debe haber ocurrido
    moral_score_threshold: float? # opcional — nivel mínimo en Seguimiento Moral
    relationship_event: String? # opcional — evento específico de la relación con este NPC
  reputation_aware: bool       # solo aplica a ramas fijas; false = fuerza tono sin variantes
```

**Regla de consistencia branch.type / min_reputation** — los autores deben respetar esta tabla:

| branch.type | min_reputation recomendado | Rango permitido |
|-------------|---------------------------|-----------------|
| `hostile` | -1.0 | [-1.0, -0.5) |
| `dark` | -1.0 | [-1.0, 0.0) |
| `neutral` | -0.5 | [-0.5, 0.5) |
| `warm` | 0.0 | [0.0, 1.0] |
| `allied` | 0.5 | [0.5, 1.0] |

El sistema emite un warning en log si un branch tiene `branch.type` fuera del rango permitido para su `min_reputation`. Esto no bloquea la carga pero alerta al autor.

**Dónde viven las ramas:** Cada NPC tiene un Resource file `NPCData_[npc_id].tres` (o equivalente en Godot 4) con un campo `branches: Array[BranchResource]`. El `BranchResource` implementa los campos de F4. El sistema NPC carga este recurso al inicializar el Area2D del NPC.

---

## Edge Cases

**E1 — NPC Muerto (alive: false)**
Si el jugador intenta interactuar con un NPC que está marcado como muerto (porque eligió ejecutarlo en un encuentro anterior):
- El Area2D del NPC sigue existiendo en la zona
- Al presionar [E], en lugar de abrir diálogo, muestra una escena visual alternativa (ej: tumba, ruinas, ausencia) durante 1-2 segundos
- Emite señal `npc_dead(npc_id)` hacia Progresión Narrativa
- No bloquea input (diferente de un diálogo normal)
- No registra nada en `dialogue_branches_seen`

**E2 — Rama No Accesible (Reputación Insuficiente para Todas las Ramas)**
Si todas las ramas regulares de un NPC son inaccesibles (porque `reputation < branch.min_reputation` para todas las ramas definidas):
- El NPC aún puede interactuarse ([E] funciona)
- El sistema muestra la rama `DEFAULT` (que debe tener `min_reputation=-1.0` sin restricción — siempre accesible)
- Si ni siquiera `DEFAULT` es accesible (error de autoría de contenido), el sistema emite error en log y falla seguro: muestra rama `FALLBACK_SILENT` que es texto narrativo genérico sin cambios de estado

**E3 — Dos Interacciones Simultáneas**
Si el jugador entra a dos Areas2D de tipo NPC al mismo tiempo (ej: dos NPCs muy cercanos):
- Solo la primera interacción (por orden de trigger, determinado por motor de física de Godot) se activa
- El segundo Area2D queda "bloqueado" mientras el primero esté en estado `INTERACTING`
- Cuando el primero emite `interaccion_completada()`, el jugador puede inmediatamente accionar el segundo NPC

**E4 — Pausa Durante Diálogo**
Si el jugador pausa el juego mientras un diálogo está en progreso (estado `INTERACTING`):
- El sistema de diálogo se pausa junto con el juego
- El menú de pausa muestra un botón "Resume" pero NO permite salir sin terminar el diálogo
- Al reanudar, el diálogo continúa desde donde fue pausado

**E5 — Cambio de Demonio Durante Diálogo**
Los demonios no pueden ser equipados/desequipados durante un diálogo. El input está bloqueado (estado `INTERACTING`), así que es imposible acceder al menú de Loadout. Por lo tanto, este edge case **no ocurre**.

**E6 — Rama Sin Texto o Vacía**
Si la rama de diálogo definida por el autor no tiene contenido de texto (está vacía o malformada):
- El sistema emite warning en log: "Branch [branch_id] is empty"
- Muestra un fallback de texto genérico: "…" (elipsis, indicando silencio)
- La rama se cuenta como vista (`added to dialogue_branches_seen`)
- El delta de reputación aún se aplica (normalmente)

**E7 — NPC Sin Ramas Definidas**
Si un NPC está en el diccionario `npc_encounters` pero no tiene ninguna rama de diálogo predefinida:
- El sistema emite error crítico en log: "NPC [npc_id] has no dialogue branches defined"
- Por seguridad, muestra rama `FALLBACK_EMPTY` que es un solo párrafo genérico: "El NPC permanece en silencio."
- No se registra nada en `dialogue_branches_seen` (es un fallback de error)
- La interacción termina sin cambios de reputación ni estado

**E8 — Interacción Forzada Colisiona con Interacción Opcional**
Si el juego intenta forzar una interacción con un NPC mientras el jugador está en el medio de una interacción opcional con otro NPC:
- El sistema aborta la interacción opcional en progreso
- Restaura el input brevemente (0.2s)
- Bloquea input nuevamente y activa la interacción forzada
- La rama de la interacción opcional que fue interrumpida NO se registra en `dialogue_branches_seen` (fue cancelada)
- El delta de reputación de esa rama NO se aplica


---

## Dependencies

| Sistema | Tipo | Interfaz |
|---------|------|----------|
| **Estado del Mundo (#4)** | Upstream (lee/escribe) | Lee: `npc_encounters[npc_id]`, `major_events`, `available_demons`, `corruption_level` (para F3 en primer encuentro); Escribe: `npc_encounters[npc_id].met`, `.dialogue_branches_seen[]`, `.reputation`, `.alive`, `.noble_streak`, `player_choices[decision_id]`, `corruption_floor` (si acto oscuro) |
| **Exploración del Mundo (#8)** | Sibling (emite señal) | Emite: `interaccion_completada()` para que Exploración restaure input tras diálogo |
| **Progresión Narrativa (#16)** | Downstream (emite señales) | Emite: `decision_made(npc_id, decision_id, choice)`, `dialogue_branch_viewed(npc_id, branch_id)`, `npc_dead(npc_id)` |
| **Seguimiento Moral (#22)** | Downstream (emite señales) | Emite: `moral_choice(decision_id, choice, impact: int)` cuando rama contiene acto oscuro |

**Notas de interdependencia:**
- Este sistema no bloquea a Exploración, Progresión ni Seguimiento — depende de ellos, no al revés
- Estado del Mundo es el hub central — este sistema la consulta y escribe constantemente
- Las señales emitidas hacia Progresión y Seguimiento son opcionales (pueden ignorarse sin romper este sistema)

---

## Tuning Knobs

| Knob | Valor Por Defecto | Rango Seguro | Efecto si Cambia |
|------|-------------------|--------------|------------------|
| `REPUTATION_DELTA_NOBLE_EXTRAORDINARY` | +0.30 | [+0.20, +0.50] | Cambiar qué tan rápido se recupera reputación con actos heroicos. Más alto = más fácil redención. |
| `REPUTATION_DELTA_NOBLE` | +0.15 | [+0.10, +0.25] | Cambiar velocidad de recuperación en acciones nobles normales. |
| `REPUTATION_DELTA_DARK` | -0.15 | [-0.25, -0.10] | Cambiar velocidad de pérdida de reputación en acciones oscuras. |
| `REPUTATION_DELTA_DARK_NARRATIVE` | -0.30 | [-0.50, -0.20] | Cambiar impacto de actos oscuros narrativos (ejecuciones, traiciones). |
| `REPUTATION_CLAMP_MIN` | -1.0 | [-1.0, 0.0] | Mínimo posible de reputación. Si cambias a -0.5, Edrick nunca puede caer a completa enemistad. |
| `REPUTATION_CLAMP_MAX` | +1.0 | [0.0, +1.0] | Máximo posible de reputación. **⚠ Advertencia**: Setting por debajo de 0.5 bloquea el tier warm/allied permanentemente sin que el jugador pueda saberlo. Solo cambiar con revisión de todas las ramas de cada NPC. |
| `NOBLE_STREAK_REQUIRED` | 2 | [1, 4] | Número de interacciones nobles consecutivas (sin oscura entre ellas) necesarias para cruzar un umbral ascendente. 1 = sin histéresis (más fácil remontar). 4 = requiere compromiso sostenido. |
| `NPC_DEAD_DISPLAY_DURATION` | 1.5 segundos | [0.5s, 3.0s] | Cuánto tiempo se muestra la escena alternativa cuando un NPC está muerto. Más corto = menos impactante; más largo = más solemne. |
| `INTERACTION_ABORT_RESTORE_DELAY` | 0.2 segundos | [0.1s, 0.5s] | Tiempo que input se restaura brevemente cuando una interacción forzada interrumpe una opcional (E8). |
| `DIALOGUE_UI_FADE_DURATION` | 0.3 segundos | [0.1s, 0.8s] | Duración del fade-in/fade-out cuando aparece/desaparece la UI de diálogo. |
| `DIALOGUE_ACTIVATION_BUFFER_MS` | 150 | [100, 300] | Milisegundos de gracia al abrir el diálogo antes de aceptar input de avance. Medidos desde que el fade-in completa (no desde que el estado cambia). Evita el skip accidental de la primera línea. Independiente del framerate. |
| `CORRUPTION_THRESHOLD_FOR_PENALTY` | 0.5 | [0.3, 0.7] | Nivel de `corruption_level` a partir del cual los NPCs nuevos comienzan con reputación penalizada (F3). Por debajo del umbral, todos los NPCs inician en 0.0. |
| `CORRUPTION_INITIAL_REPUTATION_PENALTY` | 0.2 | [0.0, 0.3] | Penalización de reputación inicial cuando Edrick supera el umbral de corrupción. NPCs nuevos comienzan en `0.0 − penalty`. Mantener bajo para no bloquear la rama FIRST_ENCOUNTER. |

**Notas de balance:**
- Los deltas son **simétricos por diseño** — cambiar uno requiere cambiar su par (+0.30 y -0.30 juntos)
- El clamp es técnicamente ajustable pero afecta R3 (selección de rama por reputación) — cambiar requiere revisar todas las ramas
- Duraciones de display son "feel" — no afectan mecánica, pero sí la percepción de dramatismo
- **No ajustar estos knobs sin playtesting** — los deltas de reputación afectan directamente la capacidad de redención

---

## Visual/Audio Requirements

**V1 — Entrada y Salida de Diálogo (Fade)**

Cuando se activa un diálogo, la transición es suave:
- Fade-in de overlay de diálogo: 0.3 segundos (ver `DIALOGUE_UI_FADE_DURATION` en Tuning Knobs G)
- Durante el fade, la música ambiente cambia (crossfade hacia tema de diálogo más íntimo)
- El fondo del reino no desaparece, solo se oscurece ligeramente (opacidad ~0.7)
- Al terminar el diálogo, fade-out espejo: 0.3 segundos + crossfade de vuelta a música de zona

**V2 — Identidad Visual de Rama por Tipo**

Las diferentes ramas tienen señales visuales que el jugador puede aprender a leer sin tooltips:
- **Rama noble**: texto en color dorado/cálido, posible efecto de luz suave alrededor del NPC
- **Rama oscura**: texto en color rojo oscuro/sangre, posible efecto de bruma o distorsión leve alrededor del NPC
- **Rama neutral**: texto blanco/gris, sin efecto especial
- **Rama de acto oscuro narrativo** (ejecución, traición): texto rojo brillante, efecto visual más intenso (pulso de luz negra o corrupción visible)

Estos efectos son **opcionales para MVP** — pueden simplificarse a solo color de texto si el tiempo no permite animación.

**V3 — Animación de NPC Durante Diálogo**

El NPC que está hablando muestra reacción física:
- Postura neutra en ramas normales
- Inclinación corporal hacia Edrick en ramas cálidas (alianza)
- Retroceso/guardia defensiva en ramas hostiles
- Tiemblo o efecto corrupto en ramas oscuras (si los demonios influyen en la percepción)

**V4 — Punto de Decisión Moral (Decision Point Visual)**

Cuando el jugador enfrenta una opción de decisión (ej: "ejecutar" vs. "perdonar"):
- Las dos opciones aparecen como botones/textos en la UI
- La opción "oscura" tiene una *aura visual* leve (color rojo, pulso) para indicar impacto moral sin obligar al jugador
- No hay "alarma sonora" — el feedback es visual, no punitivo

**V5 — Escena Alternativa: NPC Muerto (E1)**

Si el jugador regresa a un NPC ejecutado:
- El Area2D del NPC sigue existente pero muestra una **tumba, ruinas o ausencia físicamente visible** (ej: sangre seca en el suelo, lápida, cadáver descompuesto)
- Duración: 1.5 segundos (tunable en Tuning Knobs)
- Sin sonido — silencio sepulcral, o viento sutilmente inquietante
- Fade-in/fade-out: 0.2 segundos cada uno

**A1 — Crossfade Musical en Transiciones de Diálogo**

- Música de zona → Música de diálogo (más íntima, menos orquestal): 0.5 segundos crossfade
- Música de diálogo contiene leitmotivo del NPC (si es recurrente) — identidad sonora personal
- Música de diálogo → Vuelta a música de zona: 0.5 segundos crossfade al terminar

**A2 — Sonido de Activación de Diálogo**

Cuando se presiona [E] para activar un diálogo:
- SFX suave: "pop" o "chime" (no startling — 100-150ms de duración)
- Volumen: bajo (no compite con dialogue lines)
- Propósito: feedback táctil de que la interacción fue aceptada

**A3 — Subtítulos y Voice Acting (MVP)**

- MVP: **Sin voice acting** — solo texto + subtítulos
- Subtítulos: fuente legible, contraste alto (blanco sobre fondo oscuro semi-transparente)
- Duración de subtítulos: sincronizada con duración del texto (velocidad lectura ~200 words/min = 0.3s por palabra)
- Audio opcional post-MVP: voz narrativa en inglés (no localizamos en MVP)

**A4 — SFX de Decisión Moral**

Cuando el jugador selecciona una opción de decision point:
- Si opción "noble": sonido ascendente (harp arpeggio, nota major)
- Si opción "oscura": sonido descendente (low string, nota minor o disonante)
- Volumen bajo (feedback sutil, no punitivo)
- Propósito: refuerzo auditivo de la naturaleza moral de la elección


---

## UI Requirements

**UI1 — Layout Principal de Diálogo**

La interfaz de diálogo se compone de:

```
┌─────────────────────────────────────────────┐
│  [NPC_NAME] (ej: "Elder Vorn")              │  Header (NPC name + optional portrait)
├─────────────────────────────────────────────┤
│                                             │
│  [DIALOGUE TEXT — multi-line, word-wrapped] │  Body (dialogue content)
│  [up to 400 characters visible at once]     │
│                                             │
├─────────────────────────────────────────────┤
│  [Option A: "Accept his offer"]             │  Options section (if decision point)
│  [Option B: "Refuse and attack"]            │
└─────────────────────────────────────────────┘
```

- **Position**: Centro-inferior de la pantalla
- **Background**: Panel semi-transparente oscuro (opacidad ~0.85) para legibilidad sobre fondo del reino
- **Size**: Ocupar ~60-70% del ancho de pantalla, altura variable según contenido (min 100px, max 300px)

**UI2 — Elementos de Nombre y Retrato**

- **NPC Name**: Texto grande y legible (ej: font size 24-28px), color blanco o dorado si es aliado importante
- **NPC Portrait** (opcional MVP): Si existe, aparece a la izquierda del panel (100x100px o similar). Si no existe, solo el nombre.
- **Reputación Visual** (opcional MVP): Pequeño indicador bajo el nombre (ej: 3 estrellas llenas para reputation=0.5, 1 estrella para reputation=-0.5). Si no hay tiempo, omitir en MVP.

**UI3 — Texto de Diálogo**

- **Fuente**: Sans-serif, legible (ej: OpenSans, Roboto, Lato)
- **Tamaño**: 16-18px para jugabilidad cómoda, ajustable en opciones de accesibilidad
- **Color**: Blanco por defecto, pero **varía por rama**:
  - Noble: dorado/cálido
  - Oscuro: rojo/sangre
  - Neutral: blanco/gris
- **Símbolo de tipo de rama** (accesibilidad — no depende solo de color): El símbolo aparece en el **área del header**, junto al nombre del NPC — NO dentro del cuerpo del texto de diálogo. Esto mantiene la inmersión narrativa del Pilar 1 mientras garantiza legibilidad para daltónicos:
  - Noble: `★` en el header (color dorado)
  - Oscura: `✖` en el header (color rojo)
  - Neutral: sin símbolo en el header
- **Line Height**: 1.5x para espaciado cómodo
- **Word Wrapping**: Activo, respeta bordes del panel
- **Duración visible**: Hasta que el jugador presione [SPACE] o [E] para avanzar (no auto-advance en MVP)
- **Overflow**: Si el texto de una rama supera los 400 caracteres visibles con el tamaño de fuente activo, el texto se pagina internamente (el jugador avanza con [SPACE] por segmentos dentro de la misma rama — no una nueva rama). **Los autores de contenido no deben superar 400 chars por segmento de texto**, y el sistema emite `WARN: "Branch [branch_id] text exceeds display limit"` en log si el texto sin paginar supera 1200 chars.

**UI4 — Opciones de Decisión (Decision Point)**

Cuando una rama tiene punto de decisión:

```
  [→ Accept his offer] (NOBLE)
  [→ Refuse and attack] (DARK)
  [→ Ask for time to think] (NEUTRAL)
```

- Cada opción es un elemento interactivo (botón o línea seleccionable)
- **Navegación**: Arriba/Abajo con flecha de teclado o joystick analógico; [SPACE]/[E] para seleccionar
- **Feedback visual**:
  - Opción no seleccionada: color normal (blanco)
  - Opción hovering/selected: color más brillante + pequeño indicador visual (ej: `→` símbolo o highlight de fondo)
  - Opción "dark": aura roja sutil (no asusta, pero informa)
- **Estado de foco inicial**: Cuando la sección de opciones aparece, el foco de teclado se establece automáticamente en la **primera opción** (más arriba). El indicador de foco es un `→` animado + highlight de fondo que pasa WCAG 2.1 AA (contraste mínimo 3:1). El jugador navega antes de confirmar — no hay auto-selección.
- **Opción única**: Si solo existe una opción, se muestra igual que opciones múltiples (sin auto-selección). El jugador debe presionar [E]/[SPACE] explícitamente. Esto aplica a decisiones con un solo camino disponible.
- **Controller support**: Si el proyecto escala a gamepad, estas opciones deben ser navigables con D-pad

**UI5 — Avance de Diálogo y Señales de Finalización**

- Después de que todo el texto se muestra, el jugador presiona **[SPACE]** o **[E]** para avanzar a la siguiente rama/sección
- Indicador visual opcional: pequeño símbolo pulsante (ej: ▼ o "Press SPACE") en la esquina inferior derecha del panel cuando está listo para avanzar
- **Finalización**: Cuando la última línea es leída y avanzada, el sistema emite `interaccion_completada()` y el panel se desvanece (fade-out 0.3s)

**UI6 — Estados de Accesibilidad**

- **Subtítulos siempre visibles**: Integrados en el panel de diálogo (no hay toggle para "ocultar subtítulos" en MVP)
- **Tamaño de texto ajustable**: Accesibilidad → Tamaño de fuente (pequeño/normal/grande, afecta toda la UI)
- **Contraste alto**: Si el jugador lo activa en opciones, cambiar fondos a más oscuro/más opaco para mayor contraste con texto
- **Screen reader friendly** (4.5+ feature): Los elementos de UI de diálogo deben estar etiquetados accesiblemente para software de lectura de pantalla (Control nodes en Godot heredan esto automáticamente)
- **focus_mode requerido**: Todos los `Control` nodes interactivos de la UI de diálogo (opciones de decisión, botón de avance) deben tener `focus_mode = FOCUS_ALL` en Godot 4. El valor por defecto de muchos Control types es `FOCUS_NONE`, lo que rompe la navegación por teclado silenciosamente.

**UI7 — Transición Entre Ramas**

Si una rama tiene múltiples sub-secciones (ej: NPC habla, pausa, habla de nuevo):
- Fade-out del texto actual: 0.2s
- Pausa: 0.3s (silencio para énfasis)
- Fade-in del siguiente texto: 0.2s
- Total: 0.7s por transición interna

Esto crea ritmo narrativo sin interrumpir la inmersión.

**UI8 — Ejemplo Visual: Rama Noble**

```
┌─────────────────────────────────────────────┐
│  🌟 Elder Vorn (Reputation: ♡♡♡)            │  (golden text, optional stars)
├─────────────────────────────────────────────┤
│                                             │
│  "You have shown great wisdom, Edrick.      │  (golden text, warm)
│  I will aid you in your quest. The path     │
│  ahead is dark, but you will not walk it    │
│  alone."                                    │
│                                             │
│                               ▼ Press SPACE │
└─────────────────────────────────────────────┘
```

---

## Acceptance Criteria

**CA-NPC-001** — Activación de Diálogo: Input Bloqueado
**GIVEN** el jugador está en estado `INACTIVE` dentro del Area2D de un NPC vivo, **WHEN** presiona [E], **THEN** el sistema transiciona a estado `INTERACTING` y el input de movimiento del jugador queda bloqueado durante toda la conversación.
**Tipo**: Unit | **Bloquea**: Sí

**CA-NPC-002** — Activación de Diálogo: Consulta Estado del Mundo
**GIVEN** el diálogo se activa con `npc_id = "elder_vorn"`, **WHEN** el sistema inicializa la conversación, **THEN** consulta `world_state.npc_encounters["elder_vorn"]` y lee los campos `met`, `dialogue_branches_seen`, `reputation`, y `alive` antes de seleccionar una rama.
**Tipo**: Unit | **Bloquea**: Sí

**CA-NPC-003** — Restauración de Input al Terminar
**GIVEN** una conversación está en estado `INTERACTING`, **WHEN** la rama se completa y el sistema emite `interaccion_completada(npc_id)`, **THEN** el estado transiciona a `DONE` y `PlayerInput.is_blocked() == false` es observable síncronamente después de que la señal es procesada (sin esperar frames adicionales).
**Tipo**: Integration | **Bloquea**: Sí

**CA-NPC-004** — Selección de Rama: Primera Vez
**GIVEN** `npc_encounters[npc_id].met == false` y no hay rama de momento narrativo fijo asignada, **WHEN** el jugador activa el diálogo, **THEN** el sistema selecciona la rama `FIRST_ENCOUNTER` y actualiza `met = true` en el Estado del Mundo.
**Tipo**: Unit | **Bloquea**: Sí

**CA-NPC-005** — Selección de Rama: Prioridad de Momento Narrativo Fijo
**GIVEN** existe una rama de momento narrativo fijo asignada al NPC (ej: `"NPC_REVEAL_act_2_scene_3"`), **WHEN** el jugador activa el diálogo independientemente de `reputation` y `met`, **THEN** el sistema selecciona esa rama fija y no evalúa ramas de reputación ni contexto narrativo.
**Tipo**: Unit | **Bloquea**: Sí

**CA-NPC-006** — Selección de Rama: Reputación Hostil
**GIVEN** `npc_encounters[npc_id].reputation < -0.5` y `met == true` y no hay rama de momento narrativo fijo, **WHEN** el sistema selecciona rama del pool filtrado por F2, **THEN** elige una rama con `branch.type == "hostile"` o `branch.type == "dark"`.
**Tipo**: Unit | **Bloquea**: Sí

**CA-NPC-007** — Selección de Rama: Reputación Aliada
**GIVEN** `npc_encounters[npc_id].reputation >= 0.5` y `met == true` y no hay rama de momento narrativo fijo, **WHEN** el sistema selecciona rama del pool filtrado por F2, **THEN** elige una rama con `branch.type == "warm"` o `branch.type == "allied"`.
**Tipo**: Unit | **Bloquea**: Sí

**CA-NPC-008** — Fórmula F1: Reputación Se Actualiza Correctamente
**GIVEN** `reputation_old = 0.0` y se completa una rama marcada como `delta = +0.15` (noble), **WHEN** el sistema aplica F1, **THEN** `reputation_new == 0.15` y el valor queda guardado en `world_state.npc_encounters[npc_id].reputation`.
**Tipo**: Unit | **Bloquea**: Sí

**CA-NPC-009** — Fórmula F1: Clamp Superior
**GIVEN** `reputation_old = 0.9` y se completa una rama con `delta = +0.30` (acto noble extraordinario), **WHEN** el sistema aplica F1, **THEN** `reputation_new == 1.0` (clampeado, no 1.20).
**Tipo**: Unit | **Bloquea**: Sí

**CA-NPC-009b** — Fórmula F1: Clamp Inferior
**GIVEN** `reputation_old = -0.9` y se completa una rama con `delta = -0.30` (acto oscuro narrativo), **WHEN** el sistema aplica F1, **THEN** `reputation_new == -1.0` (clampeado, no -1.20).
**Tipo**: Unit | **Bloquea**: Sí

**CA-NPC-010** — Fórmula F1: Anti-Farming — Delta Solo en Primera Vista (Ramas Regulares)
**GIVEN** el jugador ya vio la rama regular `"branch_noble_01"` (está en `dialogue_branches_seen`) y esta rama vuelve a aparecer por fallback, **WHEN** el jugador la ve nuevamente, **THEN** el sistema NO aplica el delta de reputación de esa rama en esa segunda vista.
**Tipo**: Unit | **Bloquea**: Sí

**CA-NPC-010b** — Fórmula F1: Ramas de Reconciliación Siempre Aplican Delta
**GIVEN** el jugador ya vio la rama `"branch_reconciliation_01"` (marcada como `reconciliation: true`) y la vuelve a ver, **WHEN** el sistema aplica F1, **THEN** el delta de reputación de esa rama SE APLICA independientemente de su presencia en `dialogue_branches_seen`.
**Tipo**: Unit | **Bloquea**: Sí

**CA-NPC-011** — Fórmula F2: Rama Inaccesible por Reputación Baja (Alias: CA-NPC-012)
**GIVEN** `reputation = -0.8` y existe una rama con `min_reputation = -0.7`, **WHEN** el sistema construye el pool de candidatas con F2, **THEN** esa rama queda excluida del pool y no es seleccionable.
**Tipo**: Unit | **Bloquea**: Sí

**CA-NPC-012** — Fórmula F2: Fallback DEFAULT Cuando Todas las Ramas Son Inaccesibles
**GIVEN** todas las ramas definidas de un NPC tienen `min_reputation` mayor a la reputación actual del jugador (pool vacío), **WHEN** el sistema selecciona rama, **THEN** selecciona la rama `DEFAULT` (definida con `min_reputation=-1.0`) y no se emite error en log.
**Tipo**: Unit | **Bloquea**: Sí

**CA-NPC-013** — Consecuencias Morales: Registro de Decisión
**GIVEN** el jugador está en una rama con punto de decisión moral y elige una opción marcada como "oscura", **WHEN** confirma la elección, **THEN** el sistema registra `player_choices[decision_id] = {value, act, timestamp, conscious: true}` y emite `decision_made(npc_id, decision_id, choice)` hacia Progresión Narrativa y Seguimiento Moral.
**Tipo**: Integration | **Bloquea**: Sí

**CA-NPC-014a** — Interacción Forzada: Estado Post-Abort Limpio
**GIVEN** el jugador está en estado `INTERACTING` con NPC_A (opcional) y el sistema dispara interacción forzada con NPC_B, **WHEN** se activa la interacción forzada, **THEN** el estado transiciona de `INTERACTING` a `INACTIVE` (abort limpio) antes de iniciar el nuevo diálogo forzado con NPC_B.
**Tipo**: Integration | **Bloquea**: Sí

**CA-NPC-014b** — Interacción Forzada: Rama Parcial No Registrada
**GIVEN** la interacción con NPC_A fue abortada (CA-NPC-014a), **WHEN** se consulta `world_state.npc_encounters[NPC_A_id].dialogue_branches_seen`, **THEN** la rama abortada NO aparece en el array.
**Tipo**: Integration | **Bloquea**: Sí

**CA-NPC-014c** — Interacción Forzada: Delta No Aplicado
**GIVEN** la interacción con NPC_A fue abortada (CA-NPC-014a), **WHEN** se consulta `world_state.npc_encounters[NPC_A_id].reputation`, **THEN** el valor es idéntico al que tenía antes de iniciar el diálogo abortado.
**Tipo**: Integration | **Bloquea**: Sí

**CA-NPC-014d** — Interacción Forzada: Diálogo Forzado Inicia Correctamente
**GIVEN** la interacción con NPC_A fue abortada (CA-NPC-014a), **WHEN** se inicializa el diálogo con NPC_B, **THEN** el estado es `INTERACTING` con NPC_B, se muestra la rama de momento narrativo fijo asignada, e input del jugador queda bloqueado.
**Tipo**: Integration | **Bloquea**: Sí

**CA-NPC-015** — Dos Areas2D Simultáneas: Solo una Interacción Activa
**GIVEN** el jugador entra simultáneamente a los Areas2D de dos NPCs distintos, **WHEN** el jugador presiona [E], **THEN** solo el primer NPC (por orden de trigger del motor de física) inicia diálogo; el segundo queda bloqueado hasta que se emita `interaccion_completada()` del primero.
**Tipo**: Integration | **Bloquea**: No

---

**CA-NPC-016** — E6: Rama Vacía — Fallback Sin Crash
**GIVEN** una rama definida tiene `text == ""` o `text == null` (contenido vacío o malformado), **WHEN** el sistema intenta mostrarla, **THEN** emite `WARN: "Branch [branch_id] is empty"` en log, muestra el texto `"…"` en la UI, cuenta la rama como vista en `dialogue_branches_seen`, y aplica su delta de reputación normalmente.
**Tipo**: Unit | **Bloquea**: Sí

**CA-NPC-017** — E7: NPC Sin Ramas — Error Controlado Sin Crash
**GIVEN** un NPC está en `world_state.npc_encounters` pero no tiene ninguna rama de diálogo definida, **WHEN** el jugador activa interacción con [E], **THEN** el sistema emite `ERROR: "NPC [npc_id] has no dialogue branches defined"` en log, muestra `"El NPC permanece en silencio."`, no modifica `dialogue_branches_seen` ni aplica delta, y el estado regresa a `INACTIVE`.
**Tipo**: Unit | **Bloquea**: Sí

**CA-NPC-018** — Transición DONE → INACTIVE
**GIVEN** el sistema está en estado `DONE` (acaba de emitir `interaccion_completada(npc_id)`), **WHEN** el procesamiento de la señal completa, **THEN** el estado transiciona automáticamente a `INACTIVE` y el NPC queda disponible para una nueva interacción con [E].
**Tipo**: Unit | **Bloquea**: Sí

**CA-NPC-019** — Histéresis de Umbral: Tier Ascendente Requiere noble_streak ≥ 2
**GIVEN** un NPC con `reputation = 0.49` (justo debajo del umbral warm) y `noble_streak = 1` (una sola interacción noble), **WHEN** el jugador completa una segunda rama noble (+0.15) elevando `reputation` a 0.64, **THEN** el tier visual asciende a `warm` (porque `noble_streak ≥ 2` Y reputación supera el umbral) y se emite `reputation_tier_changed(npc_id, "neutral", "warm")`.
**Tipo**: Unit | **Bloquea**: Sí

**CA-NPC-020** — F3: Reputación Inicial Penalizada con Edrick Corrupto
**GIVEN** `world_state.corruption_level = 0.7` (sobre `CORRUPTION_THRESHOLD_FOR_PENALTY = 0.5`) y un NPC con `met = false`, **WHEN** el sistema inicializa la conversación (paso 2 de R2), **THEN** `npc_encounters[npc_id].reputation` se establece a `−0.2` (no 0.0) y `met` se actualiza a `true`.
**Tipo**: Unit | **Bloquea**: Sí

**CA-NPC-022** — Smoke Check de Contenido: Ramas Hostile/Warm por NPC Recurrente
**GIVEN** todos los archivos de contenido de NPC están cargados, **WHEN** se ejecuta la validación de contenido (pipeline de CI o inicio del juego en modo debug), **THEN** para cada NPC marcado como recurrente en `npc_encounters`, existe al menos 1 rama con `branch.type == "hostile"` o `"dark"` Y al menos 1 rama con `branch.type == "warm"` o `"allied"` — o bien el sistema emite `ERROR: "NPC [npc_id] missing hostile or warm branches"` en log.
**Tipo**: Config/Data | **Bloquea**: No (Advisory — smoke check)

**CA-NPC-023** — E8: Input Ignorado Durante Ventana de Restore de Interacción Forzada
**GIVEN** el sistema ha abortado una interacción opcional y se encuentra en la ventana de restore (0 < t < `INTERACTION_ABORT_RESTORE_DELAY`), **WHEN** el jugador presiona [E] durante esa ventana, **THEN** ningún diálogo se activa — ni con el NPC abortado ni con ningún otro NPC en rango — y el estado permanece en `INACTIVE` hasta que expire la ventana.
**Tipo**: Integration | **Bloquea**: Sí

---

**Nota para implementación**: CA-NPC-001 a CA-NPC-023 (26 ACs, incluyendo 009b, 010b, 014a-d) cubren el ciclo completo de la máquina de estados, edge cases E1-E8, histéresis de umbral (CA-NPC-019), reputación inicial por corrupción (CA-NPC-020), y smoke checks de contenido (CA-NPC-022). E1 se cubre con smoke check visual (no AC de lógica). E4 se traslada a GDD #21 (Sistema de Pausa). V1-V5, A1-A4, UI1-UI8 requieren evidencia visual/manual en `production/qa/evidence/` antes de que el sprint sea marcado Done.

---

## Open Questions

| Pregunta | Impacto | Propietario | Estado |
|----------|---------|-------------|--------|
| **¿Cuántos NPCs recurrentes confirmados en MVP?** | Definir la amplitud del árbol de diálogos. Plan: ~5-7 NPCs MVP (3-4 ramas cada uno). ¿Confirmado o ajustable? | Game Designer | Abierto |
| **¿Los demonios equipados afectan reputación inicial con NPCs?** | Narrativamente, un Edrick visiblemente corrupto debería empezar con reputación más baja (ej: -0.2 en lugar de 0.0). ¿Implementar? | Game Designer + Systems Designer | Abierto |
| **¿Ramas visuales diferentes para Edrick según corrupción?** | Algunos NPCs podrían notar físicamente si Edrick es "corrompido" (glow de demonios, aspecto oscurecido). ¿Integrar en branch selection o solo en V2 feedback visual? | Game Designer + Art Director | Abierto |
| **¿Voice acting post-MVP?** | MVP: solo texto. Post-MVP: ¿inglés locutado? ¿Otros idiomas? ¿Qué presupuesto? | Producer | Diferido |
| **¿Localization strategy para diálogos?** | Cómo se manejan strings de diálogo en otros idiomas. ¿CSV, sistema interno, herramienta tercera (Crowdin, etc.)? | Localization Lead | Diferido a Pre-Production |
| **¿Pueden los NPCs morir en el MVP?** | Actualmente soportado (E1, R5). ¿Hay contenido narrativo donde esto ocurra, o es solo mechanical? | Narrative Director | Diferido |
| **¿Audio leitmotivos para cada NPC?** | A1 propone leitmotivo en música de diálogo. ¿Presupuesto para ~5-7 leitmotivos únicos? | Audio Director | Diferido |
| **¿El gato/hermano tiene entradas en npc_encounters?** | El gato es el compañero principal de Edrick y debería ser el NPC más emotivamente relevante del sistema. Actualmente no está en `npc_encounters` — está fuera del sistema de moral mirrors completamente. Decide si entra en MVP o se diseña como sistema aparte en un GDD posterior. | Narrative Director + Game Designer | Abierto |
