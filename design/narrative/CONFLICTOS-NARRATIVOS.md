# Conflictos Narrativos — Decisiones Pendientes

Registro de huecos, inconsistencias y decisiones no resueltas identificados en la narrativa del Acto 1 (MVP — Reino de Luxterra).

---

## Sección 1 — Decisiones de Diseño Narrativo

Corresponden a elecciones creativas que solo el autor del juego puede tomar. No tienen una respuesta técnicamente "correcta": dependen de la visión del juego. Pueden resolverse en conversación con los agentes **`narrative-director`** (para estructurar arcos y consecuencias narrativas) y **`writer`** (para explorar opciones de voz, diálogo y tono).

---

### D-01 — Personalidad y voz de Edrick (Solucionado)

Edrick toma la palabra en al menos cinco momentos narrativos del Acto 1. Sin una definición de su carácter no se puede escribir ninguna línea de diálogo.

**Qué decidir:** ¿Cómo habla Edrick? ¿Es callado y contenido, impulsivo, cínico, o algo más complejo? ¿Ha pasado 12 años rumiando venganza o tratando de dejarla atrás? ¿Qué rasgos de carácter arrastra del trauma del prólogo?

---

### D-02 — Qué sabe Edrick al inicio del Acto 1 (Solucionado)

Sir Aldric convivió con Edrick durante 12 años. La cantidad de información que le transmitió determina el punto de partida emocional del protagonista y el peso de cada revelación del acto.

**Qué decidir:** ¿Sabe Edrick que Draeven fue el responsable de la Traición? ¿Conoce el nombre de los Hijos del Fin? ¿Tiene claro que su hermano murió? ¿O Sir Aldric lo protegió de esa verdad?

---

### D-03 — El cadáver del monasterio: ¿cómo sabe El Gato que está ahí? (Solucionado)

En la Misión 1.2.3, El Gato lleva a Edrick hasta el monasterio y excava un cadáver que porta una carta dirigida a Sir Aldric. El Gato es Dorian, muerto hace 12 años — presumiblemente antes de que esa carta existiera.

**Qué decidir:** ¿Cómo justifica el lore que Dorian/El Gato sepa de esa carta y ese cuerpo? Las opciones no son excluyentes: percepción sobrenatural propia de los Demonios Errantes, conexión con el alma del mensajero fallecido, o algo específico del vínculo entre Dorian y la Casa Velmar. Hay que elegir y documentarlo.

---

### D-04 — Las "historias antiguas" que Tristan conoce sobre El Gato (Solucionado)

La Misión 1.4.1 usa el conocimiento de Tristan sobre El Gato como primer momento de revelación de su verdadera naturaleza. Pero El Gato es Dorian Velmar, muerto hace solo 12 años. Las historias "antiguas" no encajan con esa fecha.

**Qué decidir:** ¿Tristan confunde a El Gato con una leyenda más antigua sobre Demonios Errantes, identificando incorrectamente a Dorian con algo mayor? ¿O el lore de Dorian tiene una dimensión más profunda aún no documentada? Hay que establecer exactamente qué dice Tristan en esa escena y por qué lo sabe.

---

### D-05 — Consecuencias de la elección final (Misión 1.4.4) (Solucionado — 2026-05-31)

La elección de bando está definida mecánicamente (qué demonio obtiene Edrick) pero no narrativamente (qué pasa después).

**Decisión tomada:** El hermano derrotado muere. La muerte del portador es condición necesaria para el binding demoníaco (GDD #13) — el hermano al que Edrick traiciona perece en el clímax y su demonio se vincula a Edrick. El hermano aliado sobrevive y hereda el control político de Luxterra. La elección cambia qué demonio lleva Edrick (Espada o Destello según el hermano derrotado) y qué facción controla Luxterra al inicio del Acto 2. Resuelta durante sesión de diseño de GDD #16 (Progresión Narrativa).

---

### D-06 — La línea temporal: ¿conexión entre la Traición y la Guerra Civil? (Solucionado)

La Traición de los Blackhorn ocurrió hace 12 años (12 a.H.) y la Guerra Civil Luxiana hace 13 años (13 a.H.). Un año de diferencia. Puede ser coincidencia o puede haber una conexión causal no documentada.

**Qué decidir:** ¿Son eventos independientes que ocurrieron por azar en años consecutivos? ¿O hay algún vínculo entre ambos —por ejemplo, los Blackhorn aprovechando el caos de la guerra civil para ejecutar su golpe sin respuesta exterior?

---

### D-08 — El prólogo: ¿jugable o cinemático? (Solucionado — 2026-06-04)

El prólogo está documentado en dos párrafos sin misiones, objetivos ni mecánicas. El Acto 1 completo tiene misiones, puentes y gameplay detallado.

**Qué decidir:** ¿El prólogo es una cinemática de introducción o tiene segmentos jugables? Si es jugable, necesita el mismo nivel de detalle que el Acto 1. Si es cinemático, hay que definir su duración y qué decisiones narrativas comunica al jugador antes de que empiece el juego.

**Decisión tomada:** El prólogo es una cinemática de introducción no jugable. Comunica al jugador, antes de que empiece el gameplay: (1) la vida y el hogar original de Edrick en el castillo de la Casa Velmar; (2) el ataque traidor de Draeven Blackhorn y la masacre de la familia; (3) la huida de un Edrick de 8 años en brazos de Sir Aldric; (4) el trauma que marcará toda la personalidad del protagonista. Al terminar la cinemática, el juego salta 12 años al presente del Acto 1. El scope de GDD #17 (Cinemáticas) debe incluir la implementación de esta secuencia.

---

## Sección 2 — A Resolver en GDD

Aspectos de gameplay implicados por la narrativa que no tienen diseño técnico formal. Deben abordarse a través del flujo normal de diseño: `/design-system` para autoría, `/design-review` para revisión. Los agentes responsables son **`game-designer`** y **`systems-designer`** para el diseño mecánico, y **`gameplay-programmer`** para la evaluación de implementabilidad. Que esté en esta sección no implica que requiera un gdd propio, si no que requiere un estudio para comprobarlo. Para cada aspecto se propone un GDD, pero si el sistema decide que se puede implementar como parte de un gdd ya existente será totalmente válido.

---

### G-01 — Mecánica del compañero El Gato

El Gato combate junto a Edrick desde el Arco 1.1, lleva al jugador a lugares (Misión 1.2.2), y excava objetos del escenario (Misión 1.2.3). Sin GDD de compañero no se puede implementar ninguna de estas misiones.

**Pendiente:** GDD de sistema de compañero — comportamiento en combate, navegación, interacciones con el entorno, condiciones de aparición/desaparición.

---

### G-02 — Sistema de atuendos y disfraz

La Misión 1.2.1 es el tutorial del juego y enseña el sistema de atuendos: sigilo, percepción de NPCs, y al menos una elección con consecuencias en arcos futuros. Sin GDD este tutorial no se puede implementar.

**Pendiente:** GDD de sistema de atuendos — cómo afectan al sigilo, a las reacciones de NPCs, y qué implica la elección guardar/tirar.

---

### G-03 — Sistema de sigilo

La Misión 1.2.1 es una misión de sigilo completa. No existe GDD de sigilo en el proyecto.

**Pendiente:** GDD de mecánica de sigilo — detección, zonas de visión, consecuencias de ser descubierto.

---

### G-04 — Sistema de diálogos con NPCs como motor de progresión

Las Misiones 1.2.3, 1.3.3 y 1.4.2 usan la interacción con NPCs como mecanismo principal de avance narrativo y de deducción. Sin GDD no existe un sistema formal que soporte esto.

**Pendiente:** GDD de sistema de diálogo/conversación — cómo se activan, cómo guían al jugador, si hay opciones de respuesta y qué consecuencias tienen.

---

### G-05 — Batallas a gran escala (asedios)

La Misión 1.3.4 se describe explícitamente como "el primer gran asedio del juego" con una escala distinta al combate individual. Es una mecánica nueva que necesita diseño propio.

**Pendiente:** GDD de sistema de asedio/batalla masiva — cómo se diferencia del combate normal, qué objetivos tiene el jugador, cómo encajan los aliados.

---

### G-06 — Sistema de decisiones narrativas con consecuencias

El Acto 1 introduce tres elecciones con peso narrativo: guardar o tirar el atuendo (1.2.1), acudir o no a la reunión con Tristan (1.3.4), y elegir bando en el clímax (1.4.4). Sin GDD no hay un sistema que gestione el estado de estas decisiones ni sus efectos.

**Pendiente:** GDD de sistema de decisiones narrativas — cómo se registran, cómo afectan al estado del mundo, y cómo se propagan a actos siguientes.

