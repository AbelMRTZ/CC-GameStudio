# GDD: Progresión Narrativa

> **Estado**: Aprobado (Re-Revisión #3 — 2026-06-05)
> **Autor**: Abel + Claude Code Agents
> **Última Actualización**: 2026-06-05
> **Implements Pillar**: Pilar 1 — Narrativa Cinematográfica; Pilar 5 — Transformación Moral de Edrick
> **Milestone**: MVP — Feature Layer
> **Depende de**: Estado del Mundo (#4), Vinculación de Demonios (#13), NPC y Diálogo (#15)
> **Dependen de este sistema**: Cinemáticas (#17), Bestiario (#19), Seguimiento Moral (#22)

---

## Overview

El Sistema de Progresión Narrativa es el **director de escena de Demons of Dravaryn**: monitorea el estado de la historia, avanza las fases de los actos, y dispara los gates que desbloquean cinemáticas, diálogos, bindings de demonios y contenido del Bestiario en el momento correcto. Internamente, es una máquina de estado narrativo que escucha señales del mundo — el jugador vinculó un demonio, tomó una decisión moral con un NPC, exploró un área nueva, completó una misión — y actualiza `major_events` y `acts_completed` en el Estado del Mundo como respuesta. Para el jugador, el sistema se manifiesta como el mundo que *recuerda*: NPCs que reaccionan a lo que hiciste en el acto anterior, áreas que solo se abren después de ciertos momentos narrativos, el gato que aparece cuando la historia lo pide y no antes. El jugador no interactúa con este sistema directamente — lo experimenta a través de cada consecuencia narrativa que se siente ganada, no arbitraria. Es el tejido conectivo que convierte una colección de sistemas mecánicos en una historia que progresa con coherencia.

## Player Fantasy

Cuando el jugador toma una decisión — perdonar a un enemigo, ejecutar a un aliado que se interpone, elegir un bando en un conflicto donde ninguna opción es limpia — la fantasía de la Progresión Narrativa es que esa decisión **le importa al mundo**. No aparece una pantalla de "logro desbloqueado". No hay puntuación moral visible. Lo que sucede es que, tres misiones después, el NPC que casi ejecutaste aparece en un momento inesperado y dice algo que solo tiene sentido si el jugador recuerda lo que hizo. La puerta del monasterio que estaba cerrada ahora está abierta porque terminaste el arco correcto. El Gato, que ha estado contigo silenciosamente, tiene una línea diferente de diálogo porque el juego sabe en qué acto estás y qué has visto. Esa sensación — *"el juego me está recordando quién soy"* — es el núcleo emocional de este sistema.

La progresión también tiene el ritmo de una historia bien construida: capítulos que tienen principio, peso, y cierre. El final de un acto se siente como el final de un capítulo de libro — algo se resolvió, algo nuevo se abre, y hay una pregunta abierta que empuja al siguiente. El jugador no persigue un marcador de misiones; persigue la respuesta a una pregunta narrativa que el sistema ha plantado.

**Alineación de pilares**: Pilar 1 — *"La narrativa va primero; la mecánica la sirve."* Este sistema es la columna vertebral que hace que esa promesa sea verdad mecánicamente. Pilar 5 — la transformación de Edrick se manifiesta a través de qué eventos narrativos se desbloquean, en qué orden, y con qué tono — el sistema traduce corrupción en consecuencias del mundo.

## Detailed Design

### Core Rules

**R1 — Definición de Gate Narrativo**

El sistema gestiona una colección de **gates narrativos** — unidades atómicas de progresión. Cada gate define:
- Un **trigger**: el tipo de señal que puede activarlo (ver R2)
- Cero o más **condiciones previas**: checks de `major_events` que deben cumplirse antes de que el gate pueda disparar
- Una lista de **consecuencias**: qué ocurre cuando el gate dispara (ver R4)

Los gates se definen en archivos de datos (`/assets/data/narrative/acto_[N].yaml`), uno por acto. El código del sistema es genérico; el contenido narrativo es data. Esto permite editar la historia sin tocar GDScript.

---

**R2 — Tipos de Trigger**

El sistema está suscrito al EventBus global. Cuando llega una señal, evalúa todos los gates cuyo trigger coincide:

| Tipo de Trigger | Señal EventBus que lo dispara | Parámetros de matching |
|----------------|-------------------------------|------------------------|
| `zona_trigger` | `zona_visitada(zona_id)` de Exploración (#8) | `zona_id` coincide con el definido en el gate |
| `binding_complete` | `demon_bound(demon_id)` de Vinculación (#13) | `demon_id` coincide |
| `decision_made` | `decision_made(npc_id, decision_id, choice)` de NPC y Diálogo (#15) | `decision_id` y opcionalmente `choice` coinciden |
| `dialogue_viewed` | `dialogue_branch_viewed(npc_id, branch_id)` de NPC y Diálogo (#15) | `branch_id` coincide |
| `npc_dead` | `npc_dead(npc_id)` de NPC y Diálogo (#15) | `npc_id` coincide |
| `combat_sequence` | `combat_sequence_complete(sequence_id)` de Combate (#6) | `sequence_id` coincide |

**Nota**: `combat_sequence_complete` es una señal nueva que debe añadirse al EventBus (ADR-002). Combate (#6) debe emitirla cuando completa una secuencia de combate con ID narrativo predefinido (ej: "batalla_final_arco_1_1"). Las secuencias de combate narrativas se identifican en los datos del acto.

**API pública de trigger para testing**: `NarrativeSystem` expone el método `trigger_event(trigger_type: String, trigger_id: String) -> void` como API pública. Los listeners del EventBus son wrappers que internamente llaman `trigger_event`. Los tests de unidad e integración DEBEN usar `trigger_event` directamente — no llamar métodos internos (`_on_[signal]`) ni re-emitir al EventBus. Esto permite testing determinista sin depender del autoload global. CA-PN-001 a CA-PN-025 usan `trigger_event` como punto de entrada en todos sus setups.

---

**R3 — Evaluación de Condiciones**

Cuando llega un trigger, el sistema evalúa las condiciones previas del gate de forma AND: **todas** deben cumplirse para que el gate dispare. Cada condición verifica: `world_state.major_events.get(key, 0) >= fase_min`.

El sistema **no hace polling** — solo evalúa gates cuando llega una señal relevante. Sin `_process()`.

**Invariante de orden de evaluación (B6)**: cuando múltiples gates coinciden con el mismo trigger, la evaluación y ejecución son **secuenciales**: el sistema evalúa y ejecuta completamente el gate N (incluyendo todas sus consecuencias) antes de evaluar el gate N+1. Nunca "evalúa todos, luego ejecuta todos". Este orden es un invariante de corrección: los gates que dependen de escrituras de otros gates con el mismo trigger (p. ej.: `arco_1_1` depende de que `aldric_muerto` y `encuentro_gato` ya estén escritos) solo son correctos con este modelo de ejecución. El orden de los gates dentro de un mismo trigger se define por su posición en el YAML del acto.

**Extensión: condición con `max_fase` (igualdad exacta / rango)**: cuando una condición define un campo opcional `max_fase: int`, la evaluación se extiende a comprobación de rango: `world_state.major_events.get(c.key, 0) >= c.fase_min AND <= c.max_fase`. Usado para igualdad exacta (`fase_min: 0, max_fase: 0` → el key debe ser exactamente 0) o rangos acotados. Ejemplo: gate de emboscada con `{key: "reunion_tristan", fase_min: 0, max_fase: 0}` — excluye jugadores donde `reunion_tristan` ya es 1 o 2. **Validación de autoría**: `fase_min = 0` sin `max_fase` es degenerate (siempre verdadero para keys ausentes) y se rechaza en tiempo de carga con `push_error` + estado `NO_NARRATIVE`. `fase_min = 0` con `max_fase` definido es válido.

**Nota de autoría de ordenamiento YAML**: el orden de los gates en el YAML es semánticamente significativo cuando comparten un trigger. Para el trigger `combat_sequence_complete("batalla_hijos_fin_arco_1_1")` del Acto 1, el orden obligatorio en el YAML es: (1) `gate_aldric_muerto`, (2) `gate_encuentro_gato_fase2`, (3) `gate_arco_1_1_completo`. Un YAML donde `gate_arco_1_1_completo` aparece antes que los otros dos producirá que ese gate evalúe sus precondiciones con el estado anterior, falle silenciosamente, y el Arco 1.1 nunca se complete. No hay validación automática de este orden — la responsabilidad es del autor del YAML.

---

**R4 — Tipos de Consecuencias**

Cuando un gate dispara, ejecuta sus consecuencias en orden:

| Tipo de Consecuencia | Qué hace |
|---------------------|---------|
| `major_event` | Actualiza `world_state.major_events[key] = fase` via Estado del Mundo (#4) |
| `player_choice` | Registra `world_state.record_event("choice", key, value, act, conscious)` |
| `acts_completed` | Añade `acto_id` a `world_state.acts_completed` |
| `signal` | Emite una señal específica al EventBus (ej: `cat_encounter_complete`, `cinematic_trigger`) |

**Invariante de emisión de señales (B8)**: las consecuencias de tipo `signal` se emiten **después de todos los `major_event` writes** en la secuencia de consecuencias, garantizando que los listeners downstream leen el estado actualizado. Para prevenir re-entrada si una señal desencadena otro gate evaluation del mismo gate, el sistema mantiene un conjunto `gates_in_progress: Array[String]` que se chequea junto con `fired_gates` en el guard de evaluación. El gate se añade a `gates_in_progress` al inicio de la ejecución de consecuencias (antes de ejecutar la primera) y se mueve a `fired_gates` al completar todas; si las consecuencias fallan parcialmente, el gate sale de `gates_in_progress` sin entrar en `fired_gates`, permitiendo re-disparo en el próximo trigger.

**Contrato de listeners de consecuencias `signal` (B8-ext)**: los sistemas downstream que se suscriben a señales emitidas como consecuencias de gate (ej: GDD #17 Cinemáticas escuchando `cinematic_trigger`, GDD #13 Vinculación escuchando `cat_encounter_complete`) DEBEN manejar esas señales de forma no-bloqueante. Un listener que ejecuta trabajo síncrono largo (ej: cargar y reproducir una cinemática completa en el mismo frame) bloquea el dispatch de señales y puede causar congelamiento de la evaluación de gates. El trabajo pesado debe delegarse a la siguiente frame con `call_deferred()` o `await`. Este contrato es especialmente crítico para GDD #17 (Cinemáticas).

---

**R5 — Deduplicación**

Cada gate dispara **exactamente una vez** por partida. El sistema mantiene `fired_gates: Array[String]` en memoria (no persistido — se reconstruye al cargar). Al iniciar, el sistema recorre los gates y reconstruye `fired_gates` basándose en el estado actual de `major_events`: cualquier gate cuyas consecuencias ya están reflejadas en el Estado del Mundo se marca como disparado y no puede re-disparar.

**Validación de sentinel en tiempo de carga (B7)**: al inicializar, el sistema verifica que todo gate sin consecuencias de tipo `major_event` tiene un key sentinel definido. Si encuentra un gate sin sentinel y sin `major_event`, emite `push_error("NarrativeSystem: gate [id] has no major_event consequence and no sentinel — cannot safely reconstruct fired state")` y **entra en estado `NO_NARRATIVE`** (el mismo camino de fallo que un YAML no cargable — ver Edge Case correspondiente). La narrativa queda deshabilitada hasta que el YAML sea corregido con el sentinel correspondiente. No es un error recuperable en runtime: un gate sin sentinel en producción es un bug de datos, no un caso de borde tolerable.

---

**R6 — Misiones Secundarias Opcionales (con consecuencias)**

Las misiones secundarias (ej: Arco 1.4.2) son opcionales — sus gates no son condición previa de los arcos siguientes. Sin embargo, el sistema las trackea con sus propios `major_events`. Sistemas downstream (NPC y Diálogo, Cinemáticas) pueden consultar esos events para ofrecer variantes de diálogo o escenas alternativas. La progresión del arco principal avanza sin necesitarlas; la calidad narrativa varía si el jugador las completó.

---

**R7 — Inicio del Juego**

Al iniciar una nueva partida, el sistema comienza en **Arco 1.1 activo**: todos los gates del Acto 1 están disponibles para disparar desde el primer frame jugable. No requiere trigger explícito de "inicio de acto". El prólogo es una cinemática pre-juego, no un gate del sistema.

---

**R8 — Contrato de Autoría de Callbacks Narrativos**

El sistema provee infraestructura para callbacks narrativos diferidos. La garantía de que el patrón "el mundo te recuerda" se demuestra dentro del Acto 1 — para todos los perfiles de jugador — es una responsabilidad compartida con GDD #15 (NPC y Diálogo), no del motor. El contrato completo es el siguiente:

- **Callback garantizado para todos los jugadores — Cierre de Acto 1**: GDD #15 DEBE implementar el diálogo de cierre del Gato (`gato_cierre_acto1`) como una conversación obligatoria en la escena de cierre del Acto 1, con al menos 2 variantes condicionadas en combinaciones de `major_events` del acto (recomendados: `aldric_muerto`, `eleccion_bando_acto1`, `reunion_tristan`). El Gato dice algo que solo tiene sentido si el jugador recuerda un evento específico del acto — esta es la demostración mínima garantizada del sistema para todos los jugadores, sin excepción. La escena NO puede ser missable — Level Design debe situarla en la ruta crítica.

- **Calidad de callbacks para decisiones `conscious: true`**: Los callbacks de decisiones conscientes deben cumplir dos restricciones de calidad mínimas: (a) **Personaje nombrado** — el callback debe involucrar a un personaje ya conocido por el jugador que fue afectado por la decisión; una referencia ambiental o un desconocido no satisfacen este contrato. (b) **Legibilidad** — el jugador que tomó la decisión debe poder conectar el callback con su elección original sin que se le explique; el callback emerge como consecuencia orgánica, no como nota al pie narrativa.

- **Propietario del patrón de callback diferido**: GDD #15 es responsable de implementar al menos una reacción diferida por `player_choice` de tipo `conscious: true`. El callback debe aparecer en un arco distinto al que registró la decisión — no en la misma conversación ni en el mismo arco narrativo.
- **Distancia mínima**: la respuesta a una `player_choice` de Acto N no debe surfacearse antes de que el jugador haya avanzado al menos un arco principal completo después de la decisión.
- **Tono de sorpresa**: el callback no debe anunciarse como consecuencia directa de la elección — debe emerger de forma orgánica en un contexto diferente.
- **Scope MVP**: para el MVP, basta con UN callback diferido por decisión principal (además del `gato_cierre_acto1` garantizado para todos). `decision_atuendo` tiene su callback garantizado vía el gate `atuendo_consecuencia`. `eleccion_bando_acto1` y `reunion_tristan` tienen sus callbacks diferidos a Acto 2 — GDD #15 debe implementarlos al diseñar el Acto 2, y deben cumplir las restricciones de calidad (a) y (b) anteriores.
- **Calidad del clímax con `cat_reveal = 0`**: La asimetría de `cat_reveal` es diseño intencional — el coste real de rechazar la reunión con Tristán. Sin embargo, GDD #15 y GDD #17 DEBEN implementar el clímax con `cat_reveal = 0` como una experiencia emocionalmente coherente en sus propios términos: un jugador que rechazó la reunión llega a un clímax que funciona narrativamente desde su perspectiva, sin lagunas visibles que sugieran contenido omitido. "Diferente" es el objetivo; "inferior/roto" no es aceptable.

---

### States and Transitions

Los estados del sistema son el conjunto de valores de `major_events` para el Acto 1 MVP:

| Key de major_event | Fases y significado | Gate que lo avanza |
|--------------------|--------------------|--------------------|
| `encuentro_gato` | 0=no visto, 1=primer avistamiento, 2=unido al equipo | zona_trigger "sunville_parque" → fase 1; combat_sequence "batalla_hijos_fin_arco_1_1" + fase≥1 → fase 2 + emite `cat_encounter_complete()` |
| `aldric_muerto` | 0=vivo, 1=ha muerto | combat_sequence "batalla_hijos_fin_arco_1_1" (simultáneo al gate de gato) |
| `arco_1_1` | 0=activo, 1=completo | combat_sequence "batalla_hijos_fin_arco_1_1" completa, condición: encuentro_gato≥2 + aldric_muerto≥1 |
| `decision_atuendo` | 0=sin decidir, 1=atuendo guardado, 2=atuendo tirado | `decision_made "decision_atuendo"` desde NPC y Diálogo (#15) en Misión 1.2.1, cuando el jugador confirma su elección. La decisión también se registra en `player_choices["decision_atuendo"]` con `conscious: true`. No bloquea la progresión del Acto 1. **Callback del Acto 1** (si `decision_atuendo = 2`): el gate `atuendo_consecuencia` (ver siguiente fila) se activa en Vestalia — el mundo recuerda el atuendo descartado. Implicaciones narrativas adicionales pendientes de diseño en Acto 2. |
| `atuendo_consecuencia` | 0=no ocurrido, 1=testigo de la ejecución del inocente | `zona_trigger "vestalia_camino_principal"`, precondición `decision_atuendo >= 2` + `arco_1_3 >= 1` → escribe `atuendo_consecuencia = 1` + emite `npc_dead("inocente_atuendo")`. Solo se activa si el jugador tiró el atuendo en M1.2.1. Si el jugador lo conservó (`decision_atuendo = 1`), este gate nunca dispara. **Callback del Pilar 1**: el atuendo descartado fue recogido por un inocente pobre en Vestalia que fue ejecutado confundido con Edrick — el mundo recuerda una acción casual que el jugador pudo haber olvidado. **Nota de coordinación con Level Design**: el ID de zona `"vestalia_camino_principal"` debe estar en una ruta natural del Acto 1.3–1.4 que el jugador recorra normalmente; coordinar antes de implementar el Acto 1. |
| `arco_1_2` | 0=activo, 1=completo | dialogue_viewed "carta_aldric_branch" por Edrick en el monasterio |
| `carta_aldric_encontrada` | 0=no, 1=sí | dialogue_viewed branch_id `"monasterio_carta_descubierta"` en zona `"monasterio_aurenfall"` — primer avistamiento/descubrimiento de la carta, antes de leer su contenido completo. Este gate precede a `arco_1_2`. **Relación entre los dos gates**: `carta_aldric_encontrada` y `arco_1_2` son dos pasos secuenciales del mismo momento narrativo: (1) Edrick descubre/ve la carta → dispara `monasterio_carta_descubierta`, escribe `carta_aldric_encontrada = 1`; (2) Edrick lee el contenido completo → dispara `carta_aldric_branch`, completa `arco_1_2 = 1`. Pueden ser una o dos interacciones de diálogo distintas según el diseño de Level Design y GDD #15 — el contrato de cuántas interacciones son y si están en la misma escena debe especificarse antes de implementar el Acto 1. |
| `elian_conocido` | 0=no, 1=sí | dialogue_viewed "elian_primer_encuentro_plaza" |
| `arco_1_3` | 0=activo, 1=completo | combat_sequence "fortaleza_alba_conquistada" completa |
| `invitacion_tristan_recibida` | 0=no, 1=sí | zona_trigger "fortaleza_alba_exterior_post_batalla" + arco_1_3≥1 |
| `reunion_tristan` | 0=no ocurrió, 1=reunión voluntaria, 2=rechazó (emboscada) | `decision_made "acudir_reunion_tristan"` → fase 1; `decision_made "rechazar_reunion_tristan"` activa un gate que, tras `combat_sequence_complete("emboscada_tristan")`, avanza a fase 2. Ambas rutas requieren `invitacion_tristan_recibida ≥ 1` como precondición — el rechazo es una elección explícita al ver la invitación. **B2**: la decisión de rechazo DEBE ser presentada claramente como una opción con consecuencias visibles; no es un path pasivo. El gate de emboscada tiene precondición `{key: "reunion_tristan", fase_min: 0, max_fase: 0}` (igualdad exacta via extensión F1 de R3) para excluir jugadores donde `reunion_tristan` ya es 1 (acudió voluntariamente) o 2 (ya completó la emboscada). |
| `investigacion_1_4` | 0=ninguna, 1=primera completada, 2=dos completadas, 3=todas completadas | **Gates ordenados (B5)**: `gate_mision_1_4_2a`: `decision_made "mision_1_4_2a_completa"` (sin precondición) → escribe `investigacion_1_4 = 1`; `gate_mision_1_4_2b`: `decision_made "mision_1_4_2b_completa"`, precondición `investigacion_1_4 ≥ 1` → escribe = 2; `gate_mision_1_4_2c`: `decision_made "mision_1_4_2c_completa"`, precondición `investigacion_1_4 ≥ 2` → escribe = 3 + emite señal `bestiary_unlock("investigacion_aldric_completa")` (consecuencia tipo `signal`, emitida después del `major_event` write per R4/B8). Escrituras absolutas, no incrementales. No bloquea progresión principal. |
| `eleccion_bando_acto1` | 0=sin decidir, 1=elian_aliado, 2=tristan_aliado | **Dos gates separados**: `gate_bando_elian`: trigger `decision_made "elegir_bando_elian"`, precondición `arco_1_3 >= 1` (previene fire prematuro por error de datos YAML antes de que el Arco 1.3 esté completo) → escribe `eleccion_bando_acto1 = 1` + consecuencia `player_choice("eleccion_bando_acto1", 1, "acto_1", conscious: true)`. `gate_bando_tristan`: trigger `decision_made "elegir_bando_tristan"`, precondición `arco_1_3 >= 1` → escribe `eleccion_bando_acto1 = 2` + consecuencia `player_choice("eleccion_bando_acto1", 2, "acto_1", conscious: true)`. Solo un gate puede disparar por partida — una vez que `eleccion_bando_acto1 >= 1`, el otro queda sin relevancia. |
| `arco_1_4` | 0=activo, 1=completo | combat_sequence "climax_acto_1_post_eleccion" completa, condición: eleccion_bando_acto1 ≥ 1 |
| `acto_1` | 0=en progreso, 1=completo | consecuencia de `arco_1_4 = 1`; el gate escribe DOS consecuencias: (1) `major_event` → `acto_1 = 1` (permite reconstrucción F2); (2) `acts_completed` → añade `"acto_1"` a `world_state.acts_completed`. Ambas son necesarias: la primera para que F2 pueda inferir el gate como disparado al cargar; la segunda para la semántica de "acto completado". |
| `gato_cierre_acto1` | 0=no visto, 1=visto | trigger `dialogue_viewed "gato_cierre_acto1"` — **gate garantizado para todos los jugadores**. En la escena de cierre del Acto 1, el Gato pronuncia un diálogo cuya variante es seleccionada por GDD #15 basándose en el estado de `major_events` del acto (ver R8 contrato — mínimo 2 variantes condicionadas en combinaciones de `aldric_muerto`, `eleccion_bando_acto1`, y/o `reunion_tristan`). El jugador ve una línea del Gato que solo tiene sentido si se recuerda lo que ocurrió — esta es la garantía mínima de que el sistema "el mundo te recuerda" se demuestra para todos los perfiles de jugador. Consecuencia: `major_event` → `gato_cierre_acto1 = 1` + emite `narrative_gate_open("gato_cierre_acto1")`. **Contrato con GDD #15**: el diálogo de cierre del Gato es OBLIGATORIO en la escena de cierre del Acto 1 y no puede ser missable; Level Design debe garantizar que se dispara en ruta crítica. |
| `cat_reveal` | 0=sin pistas, 1=pistas dadas (Arco 1.4.1), 2=revelación completa (Actos futuros) | `dialogue_viewed "tristan_historias_gato"` en Misión 1.4.1 → fase 1; revelación completa diferida post-MVP. **Asimetría de camino (B3)**: fase 1 solo es accesible en el camino de reunión voluntaria (`reunion_tristan = 1`). Jugadores que rechazan la reunión llegan al clímax con `cat_reveal = 0` — **estado válido por diseño intencional**. Los sistemas downstream (NPC #15, Cinemáticas #17) deben implementar ramas que funcionen con `cat_reveal = 0` en el clímax. |
| `climax_tono` | 0=neutro (default), 1=oscuro (alta corrupción) | **Gate F1-ext (B4)**: trigger `decision_made "elegir_bando_climax"`, condición float `narrative.corruption_level >= 0.5` — si se cumple, escribe `climax_tono = 1`; si no, `climax_tono` permanece en 0. **Sentinel adicional** (redundante — `climax_tono = 1` ya es una consecuencia de tipo `major_event` en `world_state.major_events`, suficiente para que F2 reconstruya este gate al cargar; el sentinel se conserva por claridad de autoría y para hacer explícita la intención): `{ key: "gate_climax_tono_oscuro_fired", target_fase: 1 }`. **Expresión del Pilar 5 — diseño intencional**: el umbral `>= 0.5` corresponde al inicio de la banda "Comprometido" en GDD #4 §4.1 y requiere juego deliberadamente oscuro (~5 ejecuciones de NPCs rendidos = +0.50, o ~45 min de combate Tier-S pasivo, o una combinación). Este es el umbral correcto: el cambio de tono de Edrick en el clímax es la *recompensa narrativa del Acto 1* para el jugador que eligió activamente la oscuridad — no debe ser visible para el jugador moralmente neutro (corruption 0.1–0.3). Un jugador que llega con corrupción baja experimenta el clímax con tono humano/conflictuado, que es la experiencia por defecto. **Matriz narrativa del clímax por tono**: `climax_tono = 0` (tono humano — jugador neutro/leve): Edrick expresa el peso de la elección — incertidumbre, el coste de elegir cuando ninguna opción es limpia, referencia implícita a lo perdido en Sunville. `climax_tono = 1` (tono oscuro — jugador deliberadamente corrupto): Edrick evalúa la elección como cálculo de ventaja pura — sin peso emocional visible, el pasado no aparece, el lenguaje es táctico. El outcome mecánico (`eleccion_bando_acto1`) no cambia en ningún caso. |

---

### Interactions with Other Systems

| Sistema | Dirección | Qué fluye | Cuándo |
|---------|-----------|-----------|--------|
| **Estado del Mundo (#4)** | ← Escribe | `major_events`, `acts_completed`, `player_choices` via `record_event()` | Cuando un gate dispara sus consecuencias |
| **NPC y Diálogo (#15)** | → Recibe | `decision_made`, `dialogue_branch_viewed`, `npc_dead` | Estas señales activan gates de tipo `decision_made`, `dialogue_viewed`, `npc_dead` |
| **Vinculación de Demonios (#13)** | → Recibe | `demon_bound(demon_id)` | Activa gates de tipo `binding_complete` post-binding |
| **Vinculación de Demonios (#13)** | ← Emite | `cat_encounter_complete()` | Cuando el gate de "gato unido" dispara; Vinculación registra al Gato sin secuencia visual |
| **Exploración del Mundo (#8)** | → Recibe | `zona_visitada(zona_id)` | Activa gates de tipo `zona_trigger` |
| **Combate en Tiempo Real (#6)** | → Recibe | `combat_sequence_complete(sequence_id)` | Activa gates de tipo `combat_sequence` |
| **Cinemáticas (#17)** | ← Emite | `cinematic_trigger(cinematic_id)` | Cuando un gate tiene consecuencia de tipo `signal → cinematic_trigger` |
| **Bestiario (#19)** | ← Emite | `bestiary_unlock(demon_id)` | Cuando `demon_bound` gate dispara con demonios que desbloquean entradas de Bestiario |
| **Audio (#5)** | ← Emite | `narrative_gate_open(gate_id)` | Audio puede suscribirse para cambiar música en gates narrativos importantes |
| **Guardado y Carga (#12)** | ← Lee | `major_events`, `acts_completed` via Estado del Mundo | Guardado persiste el estado; carga lo restaura; el sistema reconstruye `fired_gates` |

## Formulas

### F1 — Evaluación de Condición de Gate

```
GATE_READY(gate) = ∀ c ∈ gate.preconditions:
    world_state.major_events.get(c.key, 0) >= c.fase_min
```

**Variables:**

| Variable | Símbolo | Tipo | Rango | Descripción |
|----------|---------|------|-------|-------------|
| Lista de precondiciones del gate | `gate.preconditions` | `Array[Precondition]` | `[]` a ≤8 por gate | Lista AND-ed de condiciones; lista vacía = siempre listo |
| Key del evento | `c.key` | `String` | Cualquier key de `major_events` | El event key que se verifica |
| Fase mínima requerida | `c.fase_min` | `int` | `[1, N]` | La fase que debe estar alcanzada |
| Fase actual del evento | `major_events.get(c.key, 0)` | `int` | `[0, N]` | 0 si el key no existe en el dict |
| **Resultado** | `GATE_READY` | `bool` | `{true, false}` | Si el gate puede disparar |

**Rango de output:** bool. Short-circuit en el primer false.

**Casos de frontera:**
- Lista vacía: vacuamente true (el gate no tiene prerequisitos — correcto para los primeros eventos)
- Key ausente: `get(key, 0) = 0`, siempre < `c.fase_min ≥ 1` → false (safe)

**Ejemplo:** Gate "arco_1_1_completo" con condiciones `encuentro_gato≥2` y `aldric_muerto≥1`. Estado: `{encuentro_gato: 2, aldric_muerto: 1}` → `2≥2 AND 1≥1` → **true**.

**Valores degenerate y validación**: `fase_min = 0` sin `max_fase` es degenerate (`get(key, 0) = 0 >= 0` es siempre verdadero para keys ausentes) — el sistema rechaza esta configuración en tiempo de carga con `push_error` + `NO_NARRATIVE`. `fase_min = 0` con `max_fase` definido es válido (es una comprobación de igualdad con 0, ver R3 extensión max_fase). La condición vacía (`preconditions = []`) sigue siendo vacuamente verdadera — ese es el comportamiento correcto para gates sin prerequisitos.

---

### F1-ext — Condición de Float de World State (B4)

Para gates que requieren verificar campos float del World State (como `corruption_level`) que no son parte de `major_events`, se define un tipo de condición extendido:

```
FLOAT_CONDITION_READY(c) = world_state.get_nested(c.path) >= c.threshold
```

| Variable | Símbolo | Tipo | Rango | Descripción |
|----------|---------|------|-------|-------------|
| Ruta en world_state | `c.path` | `String` | Path dot-notation válido | Ej: `"narrative.corruption_level"` |
| Umbral mínimo | `c.threshold` | `float` | `[0.0, 1.0]` | El valor que debe alcanzarse |
| **Resultado** | `FLOAT_CONDITION_READY` | `bool` | `{true, false}` | Si la condición float se cumple |

**Uso MVP**: el único gate que usa este tipo en el Acto 1 es `climax_tono_oscuro` (ver tabla de Estados). La implementación de `world_state.get_nested(path)` debe estar disponible en Estado del Mundo (#4) antes de implementar este gate — ver Open Questions. Las condiciones de tipo float se evalúan AND con las condiciones de tipo `major_events` en F1 para determinar `GATE_READY`.

**Valores degenerate y validación**: `threshold = 0.0` es in-spec pero degenerate (siempre verdadero para `corruption_level` inicializado a 0.0 en `new_game()`). El sistema emite `push_warning` en tiempo de carga si detecta `threshold = 0.0`, pero no entra en `NO_NARRATIVE` (puede haber usos válidos en Actos futuros). `threshold = 1.0` con `corruption_level` que nunca llega a 1.0 resulta en un gate que nunca dispara — no es un error pero sí una trampa de autoría. El rango efectivo recomendado para MVP es `(0.1, 0.9]`.

**Política de fallo por método ausente en runtime**: si `world_state` no implementa el método `get_nested` en el momento de la evaluación, `FLOAT_CONDITION_READY` retorna `false` para todas las condiciones de este tipo y el sistema emite `push_warning("NarrativeSystem: get_nested not available — F1-ext condition skipped for gate [id]")`. El gate no dispara, el juego no crashea. Esta política es consistente con el resto del diseño de fallos del sistema (rutas de error explícitas, no silenciosas). En GDScript 4, llamar un método inexistente en un objeto produce un error irrecuperable en debug — esta verificación previa (`world_state.has_method("get_nested")`) previene ese crash.

---

### F1-combined — Evaluación Completa de Condiciones (F1 + F1-ext AND-ed)

Cuando un gate tiene condiciones de tipo `major_event` Y condiciones de tipo `float` (F1-ext), la evaluación combina ambas fórmulas con AND:

```
GATE_READY_FULL(gate) =
    (∀ c ∈ gate.preconditions WHERE c.type == "major_event":
        major_events.get(c.key, 0) >= c.fase_min
        [AND c.max_fase defined: major_events.get(c.key, 0) <= c.max_fase])
    AND
    (∀ c ∈ gate.preconditions WHERE c.type == "float":
        get_nested_safe(c.path) >= c.threshold)
```

donde `get_nested_safe(path)` retorna `world_state.get_nested(path)` si el método existe, o `false` (0.0) con `push_warning` si no (ver F1-ext política de fallo).

**Short-circuit**: se evalúan las condiciones `major_event` antes que las `float` (más baratas; la mayoría de fallos ocurren aquí). Si la primera cláusula falla, las condiciones `float` no se evalúan y `get_nested_safe` no se llama — evitando el warning innecesario en el caso común.

**Gates sin condiciones float**: la segunda cláusula es vacuamente verdadera — `GATE_READY_FULL` = `GATE_READY` (F1). En MVP, solo `climax_tono_oscuro` usa condiciones de tipo `float`.

---

### F2 — Reconstrucción de `fired_gates` al Cargar

```
ALREADY_FIRED(gate) = ∀ e ∈ gate.consequences WHERE e.type == "major_event":
    world_state.major_events.get(e.key, 0) >= e.target_fase
```

**Variables:**

| Variable | Símbolo | Tipo | Rango | Descripción |
|----------|---------|------|-------|-------------|
| Consecuencias del gate | `gate.consequences` | `Array[Consequence]` | `[]` a sin límite | Solo se filtran las de tipo `major_event` |
| Key de consecuencia | `e.key` | `String` | Key válida | El key que la consecuencia habría escrito |
| Fase objetivo | `e.target_fase` | `int` | `[1, N]` | La fase que la consecuencia habría asignado |
| **Resultado** | `ALREADY_FIRED` | `bool` | `{true, false}` | Si el gate debe considerarse ya disparado |

**Rango de output:** `fired_gates: Array[String]` — IDs de todos los gates donde `ALREADY_FIRED = true`.

**Invariante crítico**: Las escrituras a `major_events` son **absolutas, no incrementales**: siempre `major_events[key] = target_fase` (nunca `+= 1`). Esto garantiza que re-disparar un gate ya aplicado es idempotente.

**Restricción de autoría**: Todo gate que no tiene consecuencias de tipo `major_event` DEBE añadir un sentinel (`{ key: "gate_[id]_fired", target_fase: 1 }`) para poder ser inferido al recargar. Sin él, sería vacuamente `ALREADY_FIRED = true` en toda carga — el gate nunca volvería a disparar aunque no haya corrido.

**Ejemplo:** Gate "arco_1_1_completo" tiene consecuencias `{arco_1_1→1, encuentro_gato→2}`. Estado cargado: `{arco_1_1: 1, encuentro_gato: 2}`. Ambas ≥ target → `ALREADY_FIRED = true` → añadido a `fired_gates`, no se re-dispara.

---

### F3 — Progreso de Acto (Porcentaje)

```
ACT_PROGRESS_PCT = clamp(
    (P_main / P_main_total) × W_main
    + min(P_optional / P_optional_total, 1.0) × W_optional,
    0.0, 1.0
) × 100
```

**Variables:**

| Variable | Símbolo | Tipo | Rango | Descripción |
|----------|---------|------|-------|-------------|
| Gates principales completados | `P_main` | `int` | `[0, P_main_total]` | Conteo de gates con `counts_toward_main_progress: true` (campo booleano en el YAML del gate) que están actualmente en `fired_gates`. Se recalcula en cada llamada a F3. |
| Total de gates principales | `P_main_total` | `int` | `1` a sin límite; MVP Acto 1 = 13 | Conteo estático de gates con `counts_toward_main_progress: true` en el archivo de datos del acto. Calculado en inicialización — constante para la vida del acto. |
| Peso del arco principal | `W_main` | `float` | `(0.0, 1.0)`; recomendado `0.90` | Tuning knob en archivo de datos |
| Gates opcionales completados | `P_optional` | `int` | `[0, P_optional_total]` | Valor de `investigacion_1_4` (0–3 en MVP) |
| Total de gates opcionales | `P_optional_total` | `int` | `1` a sin límite; MVP Acto 1 = 3 | Definido en archivo de datos |
| Peso del contenido opcional | `W_optional` | `float` | `1.0 - W_main`; recomendado `0.10` | `W_main + W_optional = 1.0` siempre |
| **Resultado** | `ACT_PROGRESS_PCT` | `float` | `[0.0, 100.0]` | Porcentaje de progreso del acto actual |

**Rango de output:** `[0.0, 100.0]`. El contenido opcional añade máximo 10pp adicionales sobre el 90% del arco principal. 100% requiere todo el contenido principal Y todo el opcional.

**Guard de división por cero**: si `P_main_total = 0` o `P_optional_total = 0`, el sistema retorna `ACT_PROGRESS_PCT = 0.0` y emite `push_error("NarrativeSystem: F3 division by zero — P_main_total or P_optional_total is 0 in act data")`. Esta condición indica un archivo de datos de acto malformado; en GDScript, `float / 0` retorna `inf`, que el `clamp` convertiría silenciosamente en `100.0` — comportamiento incorrecto que debe prevenirse con este guard explícito.

**Ejemplo:** 9/13 gates principales + 2/3 opcionales, `W_main=0.90`, `W_optional=0.10`:
`(9/13)×0.90 + (2/3)×0.10 = 0.6923×0.90 + 0.6667×0.10 = 0.62308 + 0.06667 = 0.68974 → 68.97%`

**Mecanismo de cómputo de P_main en runtime**: el campo `counts_toward_main_progress: bool` en cada gate YAML determina qué gates contribuyen al arco principal. Gates de misiones secundarias (ej: los tres gates de `investigacion_1_4`) tienen `counts_toward_main_progress: false`. El método `get_act_progress()` del NarrativeSystem computa P_main iterando sobre `fired_gates` y contando los que tienen el flag en true. P_main_total se calcula en `_ready()` al cargar el YAML del acto y se cachea.

---

### F4 — Variante Narrativa por Investigación Secundaria

```
NARRATIVE_VARIANT(investigacion_1_4) = VARIANT_TABLE[clamp(investigacion_1_4, 0, 3)]
```

**Variables:**

| Variable | Símbolo | Tipo | Rango | Descripción |
|----------|---------|------|-------|-------------|
| Misiones secundarias completadas | `investigacion_1_4` | `int` | `[0, 3]` | Valor del major_event; 0=ninguna, 3=todas |
| Tabla de variantes | `VARIANT_TABLE` | `Array[String]` | 4 entradas | Definida en `/assets/data/narrative/acto_1.yaml` |
| **Resultado** | `NARRATIVE_VARIANT` | `String` | `{"ninguna", "parcial", "avanzada", "completa"}` | Key de variante consumida por NPC y Diálogo (#15) y Cinemáticas (#17) |

**Tabla de variantes:**

| `investigacion_1_4` | Variante | Impacto narrativo |
|--------------------|----------|-------------------|
| `0` | `"ninguna"` | Baseline — NPCs del clímax no referencian la investigación |
| `1` | `"parcial"` | Un NPC del clímax reacciona con una línea breve |
| `2` | `"avanzada"` | Dos NPCs reaccionan; diálogo de Tristan tiene párrafo alternativo |
| `3` | `"completa"` | Diálogo de Tristan completamente reescrito; desbloquea entrada especial de Bestiario (`bestiary_unlock("investigacion_aldric_completa")`) |

`VARIANT_TABLE` vive en el archivo de datos del acto, no hardcodeado. Los strings son pointers a branches del árbol de diálogos.

**Guard de acceso a array**: si `VARIANT_TABLE` no contiene exactamente 4 entradas, el sistema entra en estado `NO_NARRATIVE` en tiempo de carga con `push_error("NarrativeSystem: VARIANT_TABLE must have exactly 4 entries — found [N]")`. En GDScript, acceder a un índice fuera de rango crashea en debug y retorna `null` en release — ambos son peores que `NO_NARRATIVE`. Esta validación ocurre en la misma fase de inicialización que B7.

## Edge Cases

- **Si un trigger llega y no coincide con ningún gate**: el sistema lo descarta silenciosamente. Sin error. Los triggers de otros sistemas no son exclusivos de gates narrativos — muchos combates terminan sin gate asociado.

- **Si dos triggers llegan en el mismo frame y activan el mismo gate**: F1 se evalúa dos veces. El gate dispara en el primer trigger; en el segundo, el guard de `fired_gates` lo bloquea. Las consecuencias se aplican exactamente una vez; la señal EventBus solo se emite una vez.

- **Si un gate dispara pero Estado del Mundo falla al escribir `major_events`**: las consecuencias se aplican en orden hasta la falla. El gate sale de `gates_in_progress` sin entrar en `fired_gates` (ver R4 — invariante de emisión). En la próxima carga, F2 reconstruye `fired_gates`; el gate parcialmente aplicado se re-dispara e intenta las consecuencias faltantes (las ya escritas son idempotentes). **Caso no recuperable (B9)**: si el trigger es un `combat_sequence` narrativo único que no puede repetirse (p. ej.: batalla del hermano), el partial failure resulta en pérdida permanente del estado narrativo — "eventual consistency en el próximo trigger" no aplica. Mitigación: GDD #12 garantiza que no hay guardado automático durante una secuencia de gate activa; si el sistema crashea a mitad de gate, el checkpoint pre-secuencia permite al jugador reiniciar desde antes del trigger. Esta garantía es responsabilidad de Level Design y GDD #12.

- **Si el jugador rechaza la reunión con Tristán (Misión 1.3.4)**: el sistema registra `decision_made "rechazar_reunion_tristan"` — decisión **explícita**. Esto activa el gate de emboscada: tras `combat_sequence_complete("emboscada_tristan")`, `reunion_tristan` avanza a fase 2. El rechazo es una opción presentada al jugador cuando recibe la invitación, con consecuencias implícitas visibles (rechazar a Tristán tiene un precio). Ambas rutas (`reunion_tristan = 1` o `= 2`) satisfacen `reunion_tristan ≥ 1`, condición previa del Arco 1.4. **Nota**: `cat_reveal` permanece en fase 0 en el camino de rechazo — consecuencia narrativa intencional (el jugador nunca escuchó las historias de Tristán sobre el Gato).

- **Si el jugador rechaza la reunión y llega al clímax con `cat_reveal = 0`**: **consecuencia narrativa deliberada del rechazo**. El jugador que rechazó a Tristán nunca escuchó sus historias sobre el Gato; su experiencia del clímax es narrativamente más limitada — es el precio de haber rechazado la reunión. Esta asimetría es diseño intencional y no se enmascara: la experiencia con `cat_reveal = 0` es intencionalmente reducida para dar peso real a la elección. GDD #15 y #17 deben implementar ramas funcionales para `cat_reveal = 0` (el clímax es completable mecánicamente), pero no se espera equivalencia dramática con el camino de reunión.

- **Si el jugador tiró el atuendo (`decision_atuendo = 2`) pero no visita `"vestalia_camino_principal"` antes del clímax**: el gate `atuendo_consecuencia` nunca dispara y la consecuencia queda permanentemente sin ver. Esta es una consecuencia missable por diseño — el callback existe si el jugador recorre el área natural, pero no se fuerza. Level Design debe asegurar que la zona esté en un camino razonablemente natural del Acto 1.3–1.4.

- **Si `cat_encounter_complete()` recibiría un trigger duplicado** (gate ya en `fired_gates`): el guard bloquea el gate. `cat_encounter_complete()` NO se emite de nuevo. El guard interno de Vinculación (#13) también protege de double-binding (`available_demons.has("cat")`). Protección doble.

- **Si el jugador carga una partida con `major_events` parcialmente lleno** (ej: crash a mitad de Arco 1.2): F2 reconstruye `fired_gates` desde el estado persistido. Los gates cuyas consecuencias están reflejadas no se re-disparán; los demás pueden re-dispararse si llega el trigger — comportamiento correcto.

- **Si el archivo de datos del acto (`acto_1.yaml`) no se puede cargar**: el sistema falla con error crítico en log y entra en estado `NO_NARRATIVE` — el juego es jugable pero ningún gate se evalúa. No crashea el juego. Log: `"NarrativeSystem: Failed to load act data for acto_1 — narrative gates disabled"`.

- **Si `investigacion_1_4` excede 3** (error de datos): F4 clampea a 3. Se loggea warning: `"NarrativeSystem: investigacion_1_4 value [N] exceeds maximum 3 — clamped"`.

- **Si el jugador llega al clímax (1.4.4) con `reunion_tristan = 0`** (bug de orden de secuencia): el gate de `eleccion_bando_acto1` tiene condición previa `reunion_tristan ≥ 1` — no dispara. El clímax está bloqueado hasta que la reunión ocurra. En diseño correcto de niveles esto no debería ser alcanzable.

- **Estado del Mundo §9.5 — "Edrick Puro" y expresión del Pilar 5 (B4)**: El Acto 1 completa independientemente del nivel de corrupción — no hay bloqueo de progresión. Sin embargo, el gate `climax_tono_oscuro` (ver tabla de Estados) condiciona el tono del diálogo de Edrick en la escena de elección de bando a `corruption_level >= 0.5`. Este es el único efecto narrativo visible de la corrupción en el Acto 1. Las restricciones de progresión por corrupción (si las hay) se introducen en los datos del Acto 3.

- **Resolución de contradicción narrativa — reunión con Tristán (CONFLICTOS-NARRATIVOS D-06)**: La reunión con Tristán en Misión 1.3.4 es una **bifurcación genuina**, no un evento inevitable. Una versión anterior del documento narrativo (`acto-1-luxterra.md`) describía la reunión como "inevitable mediante emboscada de todas formas" — esa descripción es obsoleta y debe corregirse en el documento narrativo. El GDD es la fuente de verdad para la mecánica: rechazar la reunión toma el path de emboscada (`reunion_tristan = 2`); acudir voluntariamente toma el path cordial (`reunion_tristan = 1`). El coste del rechazo es la experiencia reducida de `cat_reveal = 0` en el clímax — diseño intencional aceptado explícitamente (ver R8 para la garantía de calidad del path `cat_reveal = 0`).

- **Si un trigger llega para un gate cuyas precondiciones no están satisfechas**: la señal se descarta permanentemente. No hay mecanismo de re-cola. Si la precondición se satisface después (ej: el jugador completa `mision_1_4_2a_completa` después de que `mision_1_4_2b_completa` ya llegó y fue descartada), el gate de `mision_1_4_2b` nunca disparará a menos que el jugador vuelva a activar el trigger. **Contrato con GDD #15**: NPC y Diálogo DEBE garantizar que las opciones de diálogo que emiten `decision_made` para misiones secuenciales (ej: `mision_1_4_2b_completa`) solo se ofrezcan al jugador cuando el estado del acto previo ya esté en la fase requerida. Este es un contrato de prevención — el NarrativeSystem no recupera señales perdidas.

- **`arco_1_2` sin precondición explícita de `arco_1_1`** (diseño intencional): el gate de `arco_1_2` solo requiere `dialogue_viewed "carta_aldric_branch"` en el monasterio, sin precondición de `arco_1_1 >= 1`. Esto es intencional — la secuencia narrativa (arco_1_1 → viaje a Aurenfall → arco_1_2) la garantiza el diseño de niveles mediante control de acceso de zona, no el NarrativeSystem. Si Level Design abre el monasterio antes del final de Arco 1.1, `arco_1_2` podría completarse prematuramente — esto es una responsabilidad de Level Design que debe documentarse en el diseño del área.

- **D-05 — Condición de muerte del portador y binding del demonio (B11)**: La resolución de CONFLICTOS-NARRATIVOS D-05 establece que la muerte del portador es condición necesaria para el binding. Sin embargo, `combat_sequence_complete("climax_acto_1_post_eleccion")` es una señal de "secuencia de combate completada", no de "NPC muerto". **Contrato con Level Design y Combate (#6)**: la secuencia de combate del clímax debe garantizar que el portador muere antes de que la señal se emita — el combate no puede completarse con el portador incapacitado, rendido o huyendo. Adicionalmente, el gate del clímax emite `npc_dead(portador_id)` como una de sus consecuencias, lo que notifica a Vinculación (#13) del portador disponible. Si el Level Design no puede garantizar la muerte por diseño de combate, se debe añadir un gate intermedio de tipo `npc_dead` como precondición del gate de `arco_1_4`.

## Dependencies

### Dependencias Upstream (este sistema depende de)

| Sistema | GDD | Qué provee | Tipo de dependencia |
|---------|-----|-----------|---------------------|
| Estado del Mundo | [#4](estado-del-mundo.md) | `major_events`, `acts_completed`, `player_choices` — lectura de condiciones y escritura de consecuencias via `record_event()`. Contrato de escritura absoluta (no incremental) requerido. **Contrato adicional (B10)**: `world_state.acts_completed` debe implementarse con semántica de Set — añadir un ID ya presente no crea duplicados. **Contrato adicional (B4)**: `world_state.get_nested(path: String) -> float` debe estar disponible para condiciones de tipo F1-ext. | Datos — lectura/escritura |
| NPC y Diálogo | [#15](sistema-npc-dialogo.md) | Señales `decision_made(npc_id, decision_id, choice)`, `dialogue_branch_viewed(npc_id, branch_id)`, `npc_dead(npc_id)` | Eventos — señales entrantes vía EventBus |
| Vinculación de Demonios | [#13](vinculacion-demonios.md) | Señal `demon_bound(demon_id)` | Eventos — señal entrante vía EventBus |
| Exploración del Mundo | [#8](exploracion-del-mundo.md) | Señal `zona_visitada(zona_id)` | Eventos — señal entrante vía EventBus |
| Combate en Tiempo Real | [#6](combate-tiempo-real.md) | Señal `combat_sequence_complete(sequence_id)` — **nueva señal que debe añadirse a GDD #6 y al EventBus (ADR-002)** | Eventos — señal entrante vía EventBus |

### Dependencias Downstream (sistemas que dependen de este)

| Sistema | GDD | Qué consumen | Cuándo |
|---------|-----|-------------|--------|
| Vinculación de Demonios | [#13](vinculacion-demonios.md) | Señal `cat_encounter_complete()` — dispara el registro del Gato sin secuencia visual | Cuando el gate de "gato unido" dispara |
| Cinemáticas | [#17](—) | Señal `cinematic_trigger(cinematic_id)` — activa reproducción de cinemática | Cuando un gate tiene consecuencia de tipo signal→cinematic_trigger |
| NPC y Diálogo | [#15](sistema-npc-dialogo.md) | `major_events` y `acts_completed` vía Estado del Mundo para selección de ramas de diálogo; `NARRATIVE_VARIANT` (F4) para variantes de Arco 1.4 | Post-gate, en la siguiente interacción con NPCs |
| Bestiario | [#19](—) | Señal `bestiary_unlock(demon_id)` para desbloquear entradas; F3 (`ACT_PROGRESS_PCT`) para desbloqueos por progreso | Cuando gate dispara consecuencia de bestiary_unlock |
| Seguimiento Moral | [#22](—) | `major_events` vía Estado del Mundo para evaluar actos narrativos oscuros | Consulta directa al Estado del Mundo |
| Tutorial Integrado | [#25](—) | `demon_bound(demon_id)` — ya manejado por Vinculación; Progresión Narrativa no tiene interacción directa | — |
| Guardado y Carga | [#12](guardado-y-carga.md) | `major_events` y `acts_completed` persisten vía Estado del Mundo. Al cargar, este sistema reconstruye `fired_gates` via F2. GDD #12 garantiza que el guardado automático NO ocurre durante una secuencia de gate activa para evitar estados parciales | Post-carga: reconstrucción de fired_gates |

### Nota de Bidireccionalidad

- **GDD #4** menciona este sistema como dependiente suyo (§6.1 punto 5) ✓
- **GDD #13** menciona `cat_encounter_complete()` como emitida por este sistema (§3.1.G y §6.2) ✓
- **GDD #15** menciona este sistema como receptor de sus señales (Dependencies §tabla NPC) ✓
- **GDD #6** debe actualizarse para añadir la señal `combat_sequence_complete(sequence_id)` — actualmente no existe en ese GDD. **Pendiente: actualizar GDD #6.**
- **GDD #15** debe implementar el diálogo obligatorio del Gato en el cierre del Acto 1 (`gato_cierre_acto1`) con al menos 2 variantes según `major_events` del acto (ver R8 contrato). **Pendiente: especificar en GDD #15 antes de implementar el cierre del Acto 1.**

### Nueva Señal Requerida en EventBus

```gdscript
# Añadir a res://autoload/event_bus.gd (ADR-002)
# Emitida por: Combate en Tiempo Real (#6) al completar secuencia narrativa
signal combat_sequence_complete(sequence_id: String)
```

Las secuencias de combate con ID narrativo (ej: "batalla_hijos_fin_arco_1_1") se configuran en los datos del área de combate, no en código hardcodeado.

## Tuning Knobs

Todos los parámetros ajustables viven en los archivos de datos del acto (`/assets/data/narrative/acto_[N].yaml`) — nunca hardcodeados en GDScript.

| Knob | Valor MVP | Rango Seguro | Efecto si Cambia |
|------|-----------|-------------|------------------|
| `W_main` (F3 — peso de arco principal) | `0.90` | `[0.70, 0.95]` | Determina cuánto del 100% puede alcanzarse solo con el arco principal. Por debajo de 0.70, el contenido opcional se vuelve semi-obligatorio perceptualmente. Por encima de 0.95, el opcional apenas aporta. |
| `W_optional` (F3 — peso de contenido opcional) | `0.10` | `[0.05, 0.30]`; siempre = `1.0 - W_main` | Peso del bonus opcional. Cambiar `W_main` ajusta automáticamente este valor. |
| `P_main_total` (F3 — gates principales del acto) | `13` (Acto 1 MVP) | Fijo por acto — no es un knob, es el conteo de gates definidos | Solo cambia si se añaden o eliminan gates del arco principal. |
| `P_optional_total` (F3 — milestones opcionales) | `3` (Acto 1 MVP) | Fijo por acto | Solo cambia si se añaden misiones secundarias. |
| `VARIANT_TABLE` (F4 — strings de variante) | `["ninguna","parcial","avanzada","completa"]` | Los strings son fijos (contrato con GDD #15 y #17); el CONTENIDO narrativo detrás de ellos es ajustable | Cambiar estos strings rompe el contrato con NPC y Diálogo y Cinemáticas. No modificar sin actualizar ambos GDDs. |
| Gates por acto (máx precondiciones) | ≤8 precondiciones por gate | `[1, 12]` | Por encima de 12 condiciones AND-ed, la depuración de por qué un gate no dispara se vuelve muy costosa. Diseñar gates con ≤5 precondiciones como objetivo. |
| `NARRATIVE_LOG_MODE` | `"warn_only"` en producción, `"verbose"` en desarrollo | `{"silent", "warn_only", "verbose"}` | En modo `verbose`, el sistema loggea cada gate evaluado, cada trigger recibido, y cada consecuencia ejecutada. Invaluable para debug narrativo; deshabilitar en producción por volumen de logs. |

### Knobs que No Existen (por diseño)

- **No hay peso diferencial entre gates individuales**: todos los gates principales pesan igual en F3. Añadir pesos por gate introduciría complejidad de autoría sin beneficio visible en MVP.
- **No hay timeout o decay narrativo**: la progresión no se "olvida" ni decae con el tiempo. Lo que ocurrió persiste. Si se quiere puntuación de "frescura" de arcos, ese es un sistema diferente.
- **No hay límite de gates por trigger**: un solo trigger puede disparar N gates simultáneamente si todos matchean. En MVP el diseño de niveles garantiza que no ocurre más de 2-3 simultáneamente.

## Visual/Audio Requirements

Este sistema no tiene requisitos visuales propios — es infraestructura pura. Los efectos visuales y de audio de los eventos narrativos que dispara son responsabilidad de los sistemas downstream:
- **Cinemáticas (#17)**: al recibir `cinematic_trigger`, renderiza la cinemática
- **NPC y Diálogo (#15)**: al recibir señal de gate, presenta diálogo reactivo
- **Transformación Visual de Edrick (#14)**: reacciona a `major_events` vía Estado del Mundo

El sistema sí emite `narrative_gate_open(gate_id)` que Audio (#5) puede suscribir para hacer crossfades de música en momentos narrativos clave (ej: `arco_1_1_completo` → transición musical de Sunville a Aurenfall).

## UI Requirements

Este sistema no tiene UI propia en MVP. La única superficie que consume su output visualmente es:
- **Bestiario (#19)**: usa `bestiary_unlock` para revelar entradas. **`ACT_PROGRESS_PCT` (F3) es una métrica interna de desarrollo — NO debe surfacearse al jugador en ninguna forma** (sin barra de progreso, sin porcentaje, sin indicador de completitud). Su único uso permitido es en herramientas de debug/desarrollo. GDD #19 queda vinculado por esta restricción: no puede mostrar `ACT_PROGRESS_PCT` al jugador sin revisión de diseño explícita. (B1 — evita contradecir la Player Fantasy de "no visible moral score".)
- **Sistema de Pausa (#21)**: puede mostrar el acto actual (ej: "Acto 1 — Luxterra") consultando `acts_completed` vía Estado del Mundo. No mostrar porcentaje de completitud.

En post-MVP: un panel de "notas del diario" o mapa narrativo podría consumir el árbol de `major_events` directamente para mostrar qué arcos están completos.

## Acceptance Criteria

### Evaluación de Gates — F1 (CA-PN-001 a CA-PN-004)

**CA-PN-001 — Gate sin precondiciones dispara al recibir su trigger**
**GIVEN** el sistema cargado con un gate G que tiene `preconditions = []`, **WHEN** se llama `_on_trigger(trigger_type, trigger_id)` con un trigger que coincide con G, **THEN** inmediatamente después de que `_on_trigger` retorna (sin ningún `await` en el test): G está en `fired_gates` y `major_events` refleja todas las consecuencias de G aplicadas.
Tipo: Unit | Bloquea: Sí

**CA-PN-002 — Gate con todas las precondiciones satisfechas dispara**
**GIVEN** `major_events = { encuentro_gato: 2, aldric_muerto: 1 }` y un gate con `preconditions = [{ key: "encuentro_gato", fase_min: 2 }, { key: "aldric_muerto", fase_min: 1 }]`, **WHEN** llega el trigger matching del gate, **THEN** `GATE_READY` retorna `true` y el gate dispara.
Tipo: Unit | Bloquea: Sí

**CA-PN-003 — Gate con una precondición no satisfecha no dispara**
**GIVEN** `major_events = { encuentro_gato: 1, aldric_muerto: 1 }` y el mismo gate de CA-PN-002 (requiere `encuentro_gato >= 2`), **WHEN** llega el trigger matching, **THEN** `GATE_READY` retorna `false` y el gate no dispara.
Tipo: Unit | Bloquea: Sí

**CA-PN-004 — Key ausente en `major_events` se interpreta como fase 0**
**GIVEN** `major_events = {}` (dict vacío) y un gate con precondición `{ key: "arco_1_1", fase_min: 1 }`, **WHEN** se evalúa `GATE_READY`, **THEN** `major_events.get("arco_1_1", 0)` retorna `0`, la condición `0 >= 1` es `false`, y el gate no dispara.
Tipo: Unit | Bloquea: Sí

---

### Deduplicación — R5 (CA-PN-005 a CA-PN-006)

**CA-PN-005 — Gate ya disparado no vuelve a disparar**
**GIVEN** el gate G ya está en `fired_gates`, **WHEN** llega el trigger que coincide con G, **THEN** el guard de `fired_gates` bloquea el gate antes de evaluar `GATE_READY`; ninguna consecuencia se ejecuta.
Tipo: Unit | Bloquea: Sí

**CA-PN-006 — Consecuencia `signal` no causa re-entrada del gate que la emitió**
**GIVEN** el gate G tiene una consecuencia `{ type: "signal", signal_name: "test_reentrant_signal" }` y un `EventBusMock` inyectado al `NarrativeSystem` via constructor configurado para llamar `narrative_system.trigger_event(trigger_type, trigger_id_de_G)` cuando recibe `test_reentrant_signal` (usando la API pública `trigger_event`, no `_on_trigger` interno), **WHEN** se llama `narrative_system.trigger_event(trigger_type, trigger_id_de_G)` por primera vez desde el test, **THEN**: G se añade a `gates_in_progress` al inicio de la ejecución de consecuencias; cuando la señal se emite y el mock re-invoca `trigger_event`, el guard de `gates_in_progress` bloquea la re-evaluación de G; G se mueve a `fired_gates` al completar la ejecución original; las consecuencias de G se aplican exactamente una vez; no hay stack overflow ni excepción.
Tipo: Unit | Bloquea: Sí

---

### Tipos de Consecuencia — R4 (CA-PN-007 a CA-PN-008)

**CA-PN-007 — Consecuencia `major_event` escribe valor absoluto**
**GIVEN** `major_events = { encuentro_gato: 1 }` y un gate con consecuencia `{ type: "major_event", key: "encuentro_gato", target_fase: 2 }`, **WHEN** el gate dispara, **THEN** `major_events["encuentro_gato"]` es `2` (escritura absoluta, no `+= 1`).
Tipo: Unit | Bloquea: Sí

**CA-PN-008 — Consecuencia `signal` emite la señal al EventBus exactamente una vez**
**GIVEN** un gate con consecuencia `{ type: "signal", signal_name: "cat_encounter_complete" }` y un `EventBusMock` inyectado al `NarrativeSystem` via constructor (path autoritativo — no `watch_signals(EventBus)` global), **WHEN** el gate dispara via `narrative_system.trigger_event(trigger_type, trigger_id)`, **THEN** `cat_encounter_complete()` está en el log del mock exactamente una vez — verificado con el equivalente del mock a `assert_signal_emit_count(mock, "cat_encounter_complete", 1)`.
Tipo: Unit | Bloquea: Sí

---

### Reconstrucción al Cargar — F2 (CA-PN-009 a CA-PN-011)

**CA-PN-009 — Gate con consecuencias ya en `major_events` se marca como disparado al cargar**
**GIVEN** estado cargado con `major_events = { arco_1_1: 1, encuentro_gato: 2 }` y gate "arco_1_1_completo" con consecuencias `[{ key: "arco_1_1", target_fase: 1 }, { key: "encuentro_gato", target_fase: 2 }]`, **WHEN** el sistema reconstruye `fired_gates` al cargar (F2), **THEN** "arco_1_1_completo" está en `fired_gates` y no puede volver a disparar.
Tipo: Unit | Bloquea: Sí

**CA-PN-010 — Gate con consecuencias parcialmente aplicadas NO se marca como disparado**
**GIVEN** estado cargado con `major_events = { arco_1_1: 1 }` (falta `encuentro_gato: 2`) y el gate "arco_1_1_completo" que requiere ambas consecuencias, **WHEN** el sistema reconstruye `fired_gates` al cargar (F2), **THEN** "arco_1_1_completo" NO está en `fired_gates` y puede re-dispararse si llega el trigger.
Tipo: Unit | Bloquea: Sí

**CA-PN-011 — Gate con sentinel `major_event` se reconstruye correctamente al cargar**
**GIVEN** un gate de tipo signal-only con sentinel `{ key: "gate_cat_signal_fired", target_fase: 1 }` y `major_events = { gate_cat_signal_fired: 1 }`, **WHEN** el sistema reconstruye `fired_gates` al cargar (F2), **THEN** el gate está en `fired_gates` y no re-dispara.
Tipo: Unit | Bloquea: Sí

---

### Progreso de Acto — F3 (CA-PN-012 a CA-PN-013)

**CA-PN-012 — F3: progreso correcto con arco principal parcial y opcional parcial**
**GIVEN** `P_main = 9`, `P_main_total = 13`, `W_main = 0.90`, `P_optional = 2`, `P_optional_total = 3`, `W_optional = 0.10`, **WHEN** se calcula `ACT_PROGRESS_PCT`, **THEN** el resultado es `68.97` (tolerancia ±0.05). Valor exacto: `(9/13)×0.90 + (2/3)×0.10 = 0.62308 + 0.06667 = 68.97%`.
Tipo: Unit | Bloquea: Sí

**CA-PN-013 — F3: progreso no supera 100% con contenido completo**
**GIVEN** `P_main = 13`, `P_main_total = 13`, `P_optional = 3`, `P_optional_total = 3`, `W_main = 0.90`, `W_optional = 0.10`, **WHEN** se calcula `ACT_PROGRESS_PCT`, **THEN** el resultado es exactamente `100.0` (el clamp garantiza que no excede).
Tipo: Unit | Bloquea: Sí

---

### Variante Narrativa — F4 (CA-PN-014 a CA-PN-016)

**CA-PN-014 — F4: `investigacion_1_4 = 0` retorna variante `"ninguna"`**
**GIVEN** `major_events = { investigacion_1_4: 0 }`, **WHEN** se evalúa `NARRATIVE_VARIANT(0)`, **THEN** retorna `"ninguna"`.
Tipo: Unit | Bloquea: Sí

**CA-PN-015a — F4: `investigacion_1_4 = 3` retorna variante `"completa"`**
**GIVEN** `VARIANT_TABLE = ["ninguna", "parcial", "avanzada", "completa"]`, **WHEN** se evalúa `NARRATIVE_VARIANT(3)`, **THEN** retorna exactamente la string `"completa"`.
Tipo: Unit | Bloquea: Sí

**CA-PN-015b — `gate_mision_1_4_2c` emite `bestiary_unlock` al completar la tercera investigación**
**GIVEN** `major_events = { investigacion_1_4: 2 }` (dos investigaciones completadas), EventBus capturado via `watch_signals(EventBus)`, **WHEN** llega `decision_made "mision_1_4_2c_completa"`, **THEN** `major_events["investigacion_1_4"] = 3` y `bestiary_unlock("investigacion_aldric_completa")` está en el log de captura exactamente una vez.
Tipo: Integration | Bloquea: Sí

**CA-PN-016 — F4: valor superior a 3 se clampea y genera warning**
**GIVEN** `major_events = { investigacion_1_4: 5 }` (valor inválido), **WHEN** se evalúa `NARRATIVE_VARIANT(5)`, **THEN** `clamp(5, 0, 3) = 3` → retorna `"completa"`, y el log contiene `"NarrativeSystem: investigacion_1_4 value 5 exceeds maximum 3 — clamped"`.
Tipo: Unit | Bloquea: Sí

---

### Edge Cases (CA-PN-017 a CA-PN-018)

**CA-PN-017 — Trigger sin gate matching se descarta silenciosamente**
**GIVEN** el sistema cargado con los gates del Acto 1, **WHEN** llega una señal `combat_sequence_complete("secuencia_sin_gate_asociado")`, **THEN** no se dispara ningún gate, no se emite ninguna señal, y no se produce ningún error ni log de error.
Tipo: Unit | Bloquea: Sí

**CA-PN-018 — YAML de acto no cargable produce estado `NO_NARRATIVE`**
**GIVEN** el `NarrativeSystem` es inicializado con un mock de `ResourceLoader` configurado para devolver `null` para `acto_1.yaml` (inyectado via dependency injection), **WHEN** se llama `_ready()`, **THEN**: (a) el estado interno del sistema es `NO_NARRATIVE`; (b) llamar `_on_trigger` con cualquier tipo de trigger no tiene efecto; (c) el juego no crashea ni lanza excepción; (d) el log capturado contiene exactamente la cadena `"NarrativeSystem: Failed to load act data for acto_1 — narrative gates disabled"`.
Tipo: Unit | Bloquea: Sí

---

### Integración (CA-PN-019 a CA-PN-022)

**CA-PN-019 — Secuencia completa del Arco 1.1 en orden correcto**
**GIVEN** partida nueva con todos los `major_events` en fase 0 y `acto_1.yaml` cargado, **WHEN** se envía en secuencia: `zona_visitada("sunville_parque")` luego `combat_sequence_complete("batalla_hijos_fin_arco_1_1")`, **THEN** tras `zona_visitada`: `encuentro_gato = 1`; tras `combat_sequence_complete`: `aldric_muerto = 1`, `encuentro_gato = 2`, `arco_1_1 = 1`, `cat_encounter_complete()` emitida exactamente una vez; `fired_gates` contiene exactamente `["sunville_parque_gato_avistado", "batalla_hijos_aldric_muerto", "arco_1_1_completo"]`. **Estrategia de fixture**: los IDs de gate se declaran como constantes en `tests/unit/narrative/narrative_gate_ids.gd` (crear este archivo antes de implementar CA-PN-019 — no leerlos del YAML en tiempo de test). El test debe usar un YAML de fixture `tests/fixtures/acto_1_arco1_test.yaml` con solo los tres gates del Arco 1.1 para limitar el scope. **Verificación de orden B6**: CA-PN-019 solo prueba el happy path (orden correcto en YAML); el orden incorrecto es una trampa de autoría documentada en R3 — no se incluye un test de orden invertido en este suite.
Tipo: Integration | Bloquea: Sí

**CA-PN-020 — Fallo a mitad de consecuencias: gate no se marca como `fired`; re-disparo es idempotente**
**GIVEN** un gate con dos consecuencias `major_event` y un mock de WorldState inyectado donde `set_major_event(key, value)` retorna `false` (sin modificar el valor almacenado) en el segundo call — el mock usa un contador de fallos como variable de instancia (persiste entre llamadas a `_on_trigger` y se resetea explícitamente en `teardown()` del test), configurado para fallar de forma determinista en la segunda llamada a `set_major_event` dentro de la invocación actual del gate (no global entre tests; el contador se reinicia en `setup()` de cada test y solo cuenta llamadas dentro del gate en evaluación) — **WHEN** el gate dispara y la segunda escritura falla, **THEN** el gate NO está en `fired_gates` (sale de `gates_in_progress` sin entrar en `fired_gates`; este comportamiento se verifica observando que `fired_gates` no contiene el gate ID — no es necesario ni posible observar `gates_in_progress` directamente desde fuera del sistema); la primera consecuencia persiste en el mock; en el siguiente trigger el gate re-evalúa, re-intenta ambas consecuencias, la primera escritura (key ya en la fase correcta = idempotente) tiene éxito, la segunda escritura tiene éxito, el gate entra en `fired_gates`.
Tipo: Integration | Bloquea: Sí

**CA-PN-021 — Gate del clímax no dispara si `reunion_tristan = 0`**
**GIVEN** `major_events = { eleccion_bando_acto1: 0, reunion_tristan: 0 }` (bug de diseño de niveles), **WHEN** llega `combat_sequence_complete("climax_acto_1_post_eleccion")`, **THEN** el gate de `arco_1_4` no dispara (precondición `reunion_tristan >= 1` no satisfecha); el clímax queda bloqueado hasta que `reunion_tristan` sea avanzado.
Tipo: Integration | Bloquea: No (Advisory — este estado solo es alcanzable por bug de level design)

**CA-PN-022 — Gate ya disparado no emite su señal en trigger duplicado**
**GIVEN** el gate "encuentro_gato_fase2" ya en `fired_gates` y el EventBus capturado via `watch_signals(EventBus)`, **WHEN** llega un segundo trigger `combat_sequence_complete("batalla_hijos_fin_arco_1_1")`, **THEN** el guard de `fired_gates` bloquea el gate antes de evaluarlo; `cat_encounter_complete()` NO está en el log de captura; `assert_signal_emit_count(EventBus, "cat_encounter_complete", 0)`. La verificación de que Vinculación de Demonios (#13) no re-registra al Gato es responsabilidad del test de integración de GDD #13, no de este suite.
Tipo: Unit | Bloquea: Sí

---

### Tipos de Consecuencia Adicionales — R4 (CA-PN-023 a CA-PN-025) *(B12, B13)*

**CA-PN-023 — Consecuencia `acts_completed` escribe de forma idempotente**
**GIVEN** un gate con consecuencia `{ type: "acts_completed", acto_id: "acto_1" }` y `world_state.acts_completed = []`, **WHEN** el gate dispara, **THEN** `world_state.acts_completed` contiene exactamente `["acto_1"]`. Si el gate re-dispara (simulado via mock que no marca el gate como fired), `acts_completed` sigue conteniendo exactamente una entrada `"acto_1"` — sin duplicados.
Tipo: Unit | Bloquea: Sí

**CA-PN-024 — Consecuencia `player_choice` llama `record_event` con los parámetros correctos**
**GIVEN** un gate con consecuencia `{ type: "player_choice", key: "eleccion_bando_acto1", value: 1, act: "acto_1", conscious: true }` y un mock de WorldState que captura las llamadas a `record_event`, **WHEN** el gate dispara, **THEN** `world_state.record_event("choice", "eleccion_bando_acto1", 1, "acto_1", true)` es invocado exactamente una vez con esos parámetros exactos.
Tipo: Unit | Bloquea: Sí

**CA-PN-025 — Camino de emboscada (rechazo de reunión) avanza `reunion_tristan` a fase 2**
**GIVEN** `major_events = { invitacion_tristan_recibida: 1, reunion_tristan: 0 }` y el jugador ha disparado `decision_made "rechazar_reunion_tristan"` (gate de rechazo activo), **WHEN** llega `combat_sequence_complete("emboscada_tristan")`, **THEN** `major_events["reunion_tristan"] = 2`. **Assertion adicional de integración**: dado el estado anterior (`reunion_tristan = 2`) y `major_events["eleccion_bando_acto1"] = 1` (cualquier valor >= 1), **WHEN** llega `combat_sequence_complete("climax_acto_1_post_eleccion")`, **THEN** el gate de `arco_1_4` dispara (precondición `reunion_tristan >= 1` satisfecha) y `major_events["arco_1_4"] = 1`.
Tipo: Integration | Bloquea: Sí

---

---

### Progreso P_main — F3 (CA-PN-026)

**CA-PN-026 — F3: P_main se computa correctamente desde `fired_gates` con flag de datos**
**GIVEN** acto_1.yaml con 13 gates con `counts_toward_main_progress: true` y 3 con `counts_toward_main_progress: false`, y `fired_gates` contiene los IDs de 9 gates con flag `true` y 2 con flag `false`, **WHEN** se calcula `P_main`, **THEN** `P_main = 9` y `P_main_total = 13` (los gates con flag `false` no cuentan). Verificación adicional: con `W_main=0.90`, `P_optional=2`, `P_optional_total=3`, `W_optional=0.10`, `ACT_PROGRESS_PCT = 68.97` (±0.05).
Tipo: Unit | Bloquea: Sí

---

### Condición Float — F1-ext (CA-PN-027 a CA-PN-029)

**CA-PN-027 — Gate `climax_tono_oscuro` dispara cuando `corruption_level = 0.5`**
**GIVEN** un mock de WorldState con `get_nested("narrative.corruption_level")` retornando `0.5`, `major_events = {}` (sin precondiciones major_event adicionales para este gate), EventBusMock inyectado, **WHEN** se llama `narrative_system.trigger_event("decision_made", "elegir_bando_climax")`, **THEN** `FLOAT_CONDITION_READY = (0.5 >= 0.5) = true` → gate dispara → `major_events["climax_tono"] = 1` y `major_events["gate_climax_tono_oscuro_fired"] = 1` (sentinel).
Tipo: Unit | Bloquea: Sí

**CA-PN-028 — Gate `climax_tono_oscuro` NO dispara cuando `corruption_level = 0.4`**
**GIVEN** las mismas condiciones que CA-PN-027 excepto que `get_nested("narrative.corruption_level")` retorna `0.4`, **WHEN** se llama `trigger_event("decision_made", "elegir_bando_climax")`, **THEN** `FLOAT_CONDITION_READY = (0.4 >= 0.5) = false` → gate no dispara → `major_events` no contiene `climax_tono` (permanece ausente o en 0).
Tipo: Unit | Bloquea: Sí

**CA-PN-029 — Gate `climax_tono_oscuro` maneja ausencia de `get_nested()` sin crash**
**GIVEN** un mock de WorldState que NO implementa el método `get_nested` (verificable con `has_method("get_nested") = false`), EventBusMock inyectado, **WHEN** se llama `trigger_event("decision_made", "elegir_bando_climax")`, **THEN**: (a) el gate no dispara; (b) no hay excepción ni crash; (c) el log capturado contiene exactamente la cadena `"NarrativeSystem: get_nested not available — F1-ext condition skipped for gate climax_tono_oscuro"`.
Tipo: Unit | Bloquea: Sí

---

**Resumen de criterios**: 30 totales — 29 bloqueantes, 1 advisory (CA-PN-021). (CA-PN-015 dividida en CA-PN-015a y CA-PN-015b; CA-PN-026/027/028/029 añadidos en Re-Revisión #3.)
**Gate mínimo para hand-off a QA**: CA-PN-019 (smoke check Arco 1.1 end-to-end) + CA-PN-025 (smoke check camino de emboscada) + CA-PN-027 (smoke check climax_tono).
**Prerequisito de infraestructura**: señal `combat_sequence_complete` añadida al EventBus (ADR-002) + método `world_state.get_nested(path)` añadido a GDD #4 antes de escribir CA-PN-027/028/029.

## Open Questions

| Pregunta | Impacto | Propietario | Estado |
|----------|---------|-------------|--------|
| **¿El Acto 1 tiene un ending con path de "Edrick Puro" (baja corrupción)?** | Afecta si los datos del Acto 3 necesitarán `corruption_level >= X` como condición de algún gate narrativo. El sistema MVP no bloquea por corrupción — decisión diferida. | Creative Director + Narrative Director | Abierto (diferido a diseño de Acto 3) |
| **¿Qué `combat_sequence_id` corresponde exactamente a cada batalla narrativa?** | Los IDs de `combat_sequence_complete` deben coordinarse con Level Design y Combate (#6). Si los IDs no coinciden, los gates no disparan. | Lead Programmer + Level Designer | Abierto (definir antes de implementar Acto 1) |
| **¿El sistema de Seguimiento Moral (#22) leerá `NARRATIVE_VARIANT` directamente o suscribe a gates?** | Puede afectar la arquitectura de F4 si Seguimiento Moral necesita la variante como input. | Systems Designer + Game Designer | Diferido a diseño de GDD #22 |
| **¿Los datos de gates incluyen el texto de "por qué bloqueado" para debug UI?** | Útil en desarrollo para mostrar en overlay por qué un gate no disparó. Fuera de scope MVP pero bueno documentarlo. | Tools Programmer | Diferido a post-MVP |
| **D-05 (hermano derrotado) — contrato de muerte con Combate (#6)** | La resolución D-05 (muerte del portador como precondición de binding) debe verificarse en el diseño de la secuencia de combate del clímax. ¿Puede el combate completar sin que el NPC muera? Si sí, añadir gate `npc_dead(portador_id)` como precondición de `arco_1_4`. | Lead Programmer + Level Designer | Abierto — resolver antes de implementar el clímax |
| **`world_state.get_nested(path)` en Estado del Mundo (#4)** | F1-ext requiere que GDD #4 exponga un método `get_nested(path: String) -> float` para que los gates de tipo float-condition funcionen. GDD #4 debe actualizarse para incluir este método. | Lead Programmer | Abierto — actualizar GDD #4 antes de implementar el gate de `climax_tono` |
| **Validador de orden de gates YAML (B6)** | El invariante de orden secuencial de gates que comparten un trigger es silenciosamente incorrecto si los gates están mal ordenados en el YAML — sin error ni warning. Para el Acto 1 el orden correcto está documentado en R3. Para Actos futuros, un validador de carga que verifique dependencias de precondición entre gates del mismo trigger previene bugs imposibles de diagnosticar en runtime. Este validador NO es requerido para MVP-Acto 1, pero DEBE existir antes de que se autoricen gates de Acto 2. | Tools Programmer + Lead Programmer | Abierto — requerido antes de escalar a Acto 2 |
| **Especificación de variantes de `gato_cierre_acto1` con GDD #15 y Level Design** | El gate `gato_cierre_acto1` garantiza el callback del Acto 1 para todos los jugadores. Su contenido exacto (qué major_events condicionan las variantes, qué dice el Gato en cada una) debe especificarse en GDD #15 antes de implementar el Acto 1. Level Design debe confirmar que la escena de cierre del Acto 1 es una ruta crítica ineludible. | Narrative Director + Level Designer | Abierto — especificar antes de implementar el cierre del Acto 1 |
