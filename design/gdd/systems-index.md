# Systems Index: Demons Of Dravaryn

> **Status**: Aprobado
> **Creado**: 2026-05-24
> **Última Actualización**: 2026-06-04
> **Concepto Fuente**: design/gdd/game-concept.md

---

## Visión General

Demons Of Dravaryn es una aventura narrativa 2D con combate en tiempo real y un sistema de progresión basado en la vinculación de demonios. El juego requiere tres ejes mecánicos que deben funcionar en cohesión: **exploración** (navegar 9 reinos con storytelling ambiental), **combate** (tiempo real con habilidades determinadas por la build de demonios activa), y **narrativa** (progresión moral, NPCs reactivos, Bestiario vivo). Los Pilares del juego — especialmente *Demonios como Poder Transformador* y *Transformación Moral de Edrick* — requieren que los sistemas mecánicos y narrativos estén fuertemente acoplados: cada demonio vinculado transforma tanto las habilidades disponibles como el aspecto visual de Edrick y el tono de sus diálogos. El MVP debe validar que el loop exploración + binding + combate es emocionalmente atractivo y mecánicamente satisfactorio en el contexto del Reino 1.

---

## Enumeración de Sistemas

| # | Sistema | Categoría | Milestone | Estado | GDD | Depende de |
|---|---------|-----------|-----------|--------|-----|------------|
| 1 | Movimiento y Físicas 2D | Core | MVP | Aprobado | [movimiento-fisicas-2d.md](movimiento-fisicas-2d.md) | — |
| 2 | Salud y Daño | Core | MVP | Aprobado | [salud-daño.md](salud-daño.md) | — |
| 3 | Base de Datos de Demonios | Core | MVP | Aprobado | [base-datos-demonios.md](base-datos-demonios.md) | — |
| 4 | Estado del Mundo | Core | MVP | Aprobado | [estado-del-mundo.md](estado-del-mundo.md) | — |
| 5 | Sistema de Audio | Audio | MVP | Aprobado | [sistema-audio.md](sistema-audio.md) | — |
| 6 | Combate en Tiempo Real | Gameplay | MVP | Aprobado | [combate-tiempo-real.md](combate-tiempo-real.md) | Movimiento, Salud/Daño, Base de Datos de Demonios |
| 7 | IA de Enemigos (inferred) | Gameplay | MVP | Aprobado | [ia-enemigos.md](ia-enemigos.md) | Movimiento, Salud/Daño |
| 8 | Exploración del Mundo | Gameplay | MVP | Aprobado | [exploracion-del-mundo.md](exploracion-del-mundo.md) | Movimiento, Estado del Mundo |
| 9 | Cámara (inferred) | Core | MVP | Aprobado | [camara.md](camara.md) | Movimiento, Exploración |
| 10 | Loadout & Build Management | Gameplay | MVP | Aprobado | [loadout-build-management.md](loadout-build-management.md) | Base de Datos de Demonios, Estado del Mundo |
| 11 | Motor de Sinergias | Gameplay | MVP | Aprobado | [motor-sinergias.md](motor-sinergias.md) | Base de Datos de Demonios, Loadout |
| 12 | Guardado y Carga (inferred) | Persistence | MVP | Aprobado | [guardado-y-carga.md](guardado-y-carga.md) | Estado del Mundo |
| 13 | Vinculación de Demonios | Gameplay | MVP | Aprobado | [vinculacion-demonios.md](vinculacion-demonios.md) | Base de Datos de Demonios, Motor de Sinergias, Combate |
| 14 | Transformación Visual de Edrick | Gameplay | MVP | Aprobado | [transformacion-visual-edrick.md](transformacion-visual-edrick.md) | Loadout, Base de Datos de Demonios |
| 15 | Sistema de NPC y Diálogo | Narrative | MVP | Aprobado | [sistema-npc-dialogo.md](sistema-npc-dialogo.md) | Estado del Mundo |
| 16 | Progresión Narrativa | Narrative | MVP | Aprobado | [progresion-narrativa.md](progresion-narrativa.md) | Estado del Mundo, Vinculación, NPC y Diálogo |
| 17 | Cinemáticas | Narrative | MVP | No Iniciado | — | Cámara, Progresión Narrativa, Audio |
| 18 | HUD de Combate (inferred) | UI | MVP | Diseñado | [hud-combate.md](hud-combate.md) | Combate, Salud/Daño, Loadout |
| 19 | Bestiario | UI | MVP | No Iniciado | — | Base de Datos de Demonios, Motor de Sinergias, Progresión Narrativa |
| 20 | Build Management UI (inferred) | UI | MVP | No Iniciado | — | Loadout, Transformación Visual |
| 21 | Menú Principal y Pausa (inferred) | UI | MVP | No Iniciado | — | Guardado/Carga |
| 22 | Seguimiento Moral | Narrative | Vertical Slice | No Iniciado | — | Estado del Mundo, Progresión Narrativa, NPC y Diálogo |
| 23 | Restricción por Demonio | Gameplay | Vertical Slice | No Iniciado | — | Loadout, Estado del Mundo, Exploración |
| 24 | Mapa (inferred) | UI | Vertical Slice | No Iniciado | — | Exploración, Estado del Mundo |
| 25 | Tutorial Integrado (inferred) | Meta | Alpha | No Iniciado | — | Combate, Exploración, Vinculación |
| 26 | Accesibilidad (inferred) | Meta | Full Vision | No Iniciado | — | Toda la UI |
| 27 | Sigilo | Gameplay | MVP | No Iniciado | — | Movimiento, IA Enemigos, Estado del Mundo |
| 28 | Atuendos y Disfraz | Gameplay | MVP | No Iniciado | — | Sigilo (#27), Estado del Mundo, NPC y Diálogo |
| 29 | Compañero (El Gato) | Gameplay | MVP | No Iniciado | — | Movimiento, IA Enemigos, Combate, Progresión Narrativa |
| 30 | Asedios / Batalla Masiva | Gameplay | MVP | No Iniciado | — | Combate, IA Enemigos, Estado del Mundo |

---

## Categorías Utilizadas

| Categoría | Descripción | Sistemas en Este Juego |
|-----------|-------------|------------------------|
| **Core** | Sistemas fundamentales de los que todo depende | Movimiento, Salud/Daño, Base de Datos de Demonios, Estado del Mundo, Cámara |
| **Gameplay** | Sistemas que hacen el juego divertido | Combate, IA de Enemigos, Exploración, Loadout, Motor de Sinergias, Vinculación, Transformación Visual, Sigilo, Atuendos y Disfraz, Compañero (El Gato), Asedios / Batalla Masiva, Restricción por Demonio |
| **Narrative** | Historia y entrega de diálogo | NPC y Diálogo, Progresión Narrativa, Cinemáticas, Seguimiento Moral |
| **Persistence** | Estado guardado y continuidad | Guardado y Carga |
| **UI** | Displays de información para el jugador | HUD, Bestiario, Build Management UI, Menú Principal/Pausa, Mapa |
| **Audio** | Sonido y música | Sistema de Audio |
| **Meta** | Sistemas fuera del loop central | Tutorial Integrado, Accesibilidad |

---

## Tiers de Prioridad

| Tier | Definición | Milestone Objetivo | Urgencia de Diseño |
|------|------------|---------------------|-------------------|
| **MVP** | Requerido para que el loop central funcione — sin estos no se puede testear "¿es divertido?" | Primera build jugable (4–6 semanas) | Diseñar PRIMERO |
| **Vertical Slice** | Requerido para una experiencia completa y pulida de 2 reinos | Vertical slice / demo (8–10 semanas) | Diseñar SEGUNDO |
| **Alpha** | Todas las características presentes en forma rough. Alcance mecánico completo. | Milestone Alpha (12–14 semanas) | Diseñar TERCERO |
| **Full Vision** | Polish, casos extremos, opciones de accesibilidad y contenido completo | Beta / Lanzamiento (16–20 semanas) | Diseñar según avance |

---

## Mapa de Dependencias

*Diseñar y construir de arriba hacia abajo. Los sistemas arriba son fundamentos; los de abajo son envoltorios.*

### Capa Foundation (sin dependencias)

1. **Movimiento y Físicas 2D** — Todo el gameplay físico del mundo depende de estas reglas
2. **Salud y Daño** — El combate y el peligro no tienen significado sin consecuencias definidas
3. **Base de Datos de Demonios** — Estructura de datos pura que define cada demonio: atributos, habilidad, transformación visual
4. **Estado del Mundo** — Registro central de elecciones del jugador, áreas visitadas, estado de NPCs — el tejido conectivo de la narrativa
5. **Sistema de Audio** — Framework independiente al que todos los demás sistemas conectan sus eventos

### Capa Core (depende de Foundation)

1. **Combate en Tiempo Real** — depende de: Movimiento, Salud/Daño, Base de Datos de Demonios
2. **IA de Enemigos** — depende de: Movimiento, Salud/Daño
3. **Exploración del Mundo** — depende de: Movimiento
4. **Cámara** — depende de: Movimiento, Exploración
5. **Loadout & Build Management** — depende de: Base de Datos de Demonios
6. **Motor de Sinergias** — depende de: Base de Datos de Demonios, Loadout
7. **Guardado y Carga** — depende de: Estado del Mundo (serializa todo el estado activo)
8. **Sigilo** — depende de: Movimiento, IA Enemigos, Estado del Mundo

### Capa Feature (depende de Core)

1. **Vinculación de Demonios** — depende de: Base de Datos de Demonios, Motor de Sinergias, Combate
2. **Transformación Visual de Edrick** — depende de: Loadout, Base de Datos de Demonios
3. **Sistema de NPC y Diálogo** — depende de: Estado del Mundo
4. **Progresión Narrativa** — depende de: Estado del Mundo, Vinculación de Demonios, NPC y Diálogo
5. **Seguimiento Moral** — depende de: Estado del Mundo, Progresión Narrativa, NPC y Diálogo
6. **Cinemáticas** — depende de: Cámara, Progresión Narrativa, Audio
7. **Restricción por Demonio** — depende de: Loadout, Estado del Mundo, Exploración
8. **Atuendos y Disfraz** — depende de: Sigilo, Estado del Mundo, NPC y Diálogo
9. **Compañero (El Gato)** — depende de: Movimiento, IA Enemigos, Combate, Progresión Narrativa
10. **Asedios / Batalla Masiva** — depende de: Combate, IA Enemigos, Estado del Mundo

### Capa Presentation (depende de Feature)

1. **HUD de Combate** — depende de: Combate, Salud/Daño, Loadout
2. **Bestiario** — depende de: Base de Datos de Demonios, Motor de Sinergias, Progresión Narrativa
3. **Build Management UI** — depende de: Loadout, Transformación Visual de Edrick
4. **Mapa** — depende de: Exploración, Estado del Mundo
5. **Menú Principal y Pausa** — depende de: Guardado/Carga

### Capa Polish (depende de todo)

1. **Tutorial Integrado** — depende de: Combate, Exploración, Vinculación de Demonios
2. **Accesibilidad** — depende de: toda la UI de Presentation

---

## Orden Recomendado de Diseño de GDDs

*Combina capa de dependencia + prioridad de milestone. Los sistemas en la misma capa pueden diseñarse en paralelo.*

| Orden | Sistema | Milestone | Capa | Esfuerzo Estimado |
|-------|---------|-----------|------|-------------------|
| 1 | Movimiento y Físicas 2D | MVP | Foundation | M |
| 2 | Salud y Daño | MVP | Foundation | S |
| 3 | Base de Datos de Demonios | MVP | Foundation | M |
| 4 | Estado del Mundo | MVP | Foundation | M |
| 5 | Combate en Tiempo Real | MVP | Core | L |
| 6 | IA de Enemigos | MVP | Core | M |
| 7 | Exploración del Mundo | MVP | Core | M |
| 8 | Loadout & Build Management | MVP | Core | M |
| 9 | Motor de Sinergias | MVP | Core | L |
| 10 | Vinculación de Demonios | MVP | Feature | L |
| 11 | Transformación Visual de Edrick | MVP | Feature | M |
| 12 | Sistema de NPC y Diálogo | MVP | Feature | M |
| 13 | Progresión Narrativa | MVP | Feature | L |
| 14 | Cinemáticas | MVP | Feature | M |
| 15 | HUD de Combate | MVP | Presentation | S |
| 16 | Bestiario | MVP | Presentation | M |
| 17 | Build Management UI | MVP | Presentation | S |
| 18 | Menú Principal y Pausa | MVP | Presentation | S |
| 19 | Seguimiento Moral | Vertical Slice | Feature | M |
| 20 | Restricción por Demonio | Vertical Slice | Feature | S |
| 21 | Mapa | Vertical Slice | Presentation | S |
| 22 | Tutorial Integrado | Alpha | Polish | M |
| 23 | Accesibilidad | Full Vision | Polish | M |
| 24 | Sigilo | MVP | Core | M |
| 25 | Atuendos y Disfraz | MVP | Feature | S |
| 26 | Compañero (El Gato) | MVP | Feature | M |
| 27 | Asedios / Batalla Masiva | MVP | Feature | M |

*Esfuerzo: S = 1 sesión, M = 2-3 sesiones, L = 4+ sesiones. Una "sesión" es una conversación de diseño enfocada que produce un GDD completo.*

> ⚠ **Sistemas 24–27** identificados durante el diseño narrativo del Acto 1. Son todos **MVP** y deben diseñarse **antes** que los sistemas de Presentation (#15–18 en esta tabla). Orden interno obligatorio: Sigilo (#24) antes que Atuendos (#25). El Gato (#26) y Asedios (#27) pueden diseñarse en paralelo entre sí y con el par anterior.

---

## Dependencias Circulares

Ninguna detectada. El grafo de dependencias es un DAG (Directed Acyclic Graph) limpio.

---

## Sistemas de Alto Riesgo

| Sistema | Tipo de Riesgo | Descripción | Mitigación |
|---------|----------------|-------------|------------|
| **Motor de Sinergias** | Diseño + Alcance | El sistema de sinergias entre demonios puede volverse opaco o difícil de balancear con 18 demonios potenciales | MVP limita a 3 demonios. Prototipa el motor con un conjunto mínimo antes de escalar. El Bestiario comunica sinergias al jugador para evitar caja negra. |
| **Combate en Tiempo Real** | Técnico + Diseño | El combate sluggish o sin respuesta destruye el loop central. La sincronización de habilidades de demonios con el combate físico es no trivial | Prototipa el combate con un único enemigo y 2 habilidades antes de escribir el GDD completo. Input response < 100ms es requisito. |
| **Progresión Narrativa** | Diseño + Alcance | La ramificación narrativa (elecciones que se propagan) puede volverse inmanejable | Limitar el seguimiento moral a tono de diálogo y reacción de NPC — un final, múltiples rutas sutiles. Documento de arco de personaje detallado antes de GDD. |
| **Vinculación de Demonios** | Diseño | El momento de binding es el núcleo del Pilar 2. Si no se siente especial y ganado, el sistema pierde su peso narrativo | Cada binding es un momento narrativo único, no un drop de loot. Diseñar la secuencia visual/audio del binding en el GDD antes de la implementación. |
| **Base de Datos de Demonios** | Alcance | Sistema cuello de botella — 5 sistemas dependen de él. Una estructura de datos mal diseñada aquí genera deuda técnica en cascada | Diseñar el schema de datos completo (atributos, habilidades, sinergias, transformaciones visuales) antes de cualquier implementación. Validar con un ADR. |
| **Sigilo** | Diseño + Feel | La percepción de detección debe ser inequívoca — una zona de visión injusta destruye M.1.2.1 (el tutorial del juego). Es el primer contacto real del jugador con el gameplay: si el sigilo no se siente justo desde el minuto uno, la confianza se rompe antes de que el juego arranque. | Prototipar la detección con valores extremos antes de escribir el GDD. Pregunta de validación: ¿el jugador siempre sabe por qué fue detectado? Si la respuesta es "no siempre", recalibrar las zonas y los tiempos de reacción. |
| **Compañero (El Gato)** | IA + Alcance | Combinar IA de combate, navegación, interacciones con entorno y aparición condicionada por narrativa es el sistema de mayor complejidad técnica en la Feature layer. Un compañero que se queda atascado, ataca en mal momento o aparece fuera de su ventana narrativa quiebra el Pilar 1 en los momentos más visibles del juego. | MVP limita al Gato a comportamiento scriptado por fases narrativas (no IA emergente). El GDD debe especificar exactamente qué puede y no puede hacer en cada fase — sin comportamiento emergente no definido. Diferir IA autónoma a post-MVP. |

---

## Tracker de Progreso

| Métrica | Cantidad |
|---------|----------|
| Total sistemas identificados | 30 |
| GDDs iniciados | 17 |
| GDDs en revisión | 0 |
| GDDs aprobados | 14 |
| GDDs diseñados (pendiente revisión) | 3 |
| GDDs NEEDS REVISION | 0 |
| Sistemas MVP diseñados | 16 / 25 |
| Sistemas Vertical Slice diseñados | 0 / 3 |
| Sistemas Alpha diseñados | 0 / 1 |
| Sistemas Full Vision diseñados | 0 / 1 |

---

## Próximos Pasos

- [x] Enumeración de sistemas aprobada
- [x] Mapa de dependencias aprobado
- [x] Prioridades de milestone aprobadas
- [ ] Diseñar GDD #1: Movimiento y Físicas 2D — `/design-system movimiento-fisicas-2d`
- [ ] Diseñar GDD #2: Salud y Daño — `/design-system salud-daño`
- [ ] Diseñar GDD #3: Base de Datos de Demonios — `/design-system base-datos-demonios`
- [ ] Completar todos los GDDs MVP (21 sistemas)
- [ ] Ejecutar `/gate-check pre-production` cuando todos los GDDs MVP estén completos
