# Art Bible: Demons Of Dravaryn

*Created: 2026-05-24*  
*Status: In Progress*

---

## 1. Visual Identity Statement

**One-line visual rule:**  
*"If it does not carry weight, it does not merit light."*

This rule means: visual prominence is earned through narrative or mechanical significance. Nothing gets brightness or visual emphasis without justification.

### Principle 1: Paleta de Colores por Reino
*(Serves Pillar 3: Mundo Hermoso y Vivo)*

Cada reino tiene su propia identidad cromática que comunica su naturaleza emocional, clima y peligrosidad — sin necesidad de texto. La paleta base (grises, negros, marrones terrosos) es consistente, pero cada reino introduce un **color acento dominante** que define su atmósfera:

- **Reino 1 (Noble Ruins):** Ámbar/Oro apagado — decadencia, gloria pasada
- **Reino 2 (Corruption):** Verde enfermizo — putrefacción, peligro biológico
- **Reino 3 (Winter):** Azul gélido y blanco — frío extremo, aislamiento
- **Reino 4 (Harvest):** Naranja cálido — abundancia, seguridad relativa
- *Y así sucesivamente…*

Dentro de cada paleta de reino, el brillo y la saturación se siguen ganando narrativamente (demonios, personajes críticos, objetos de poder), pero siempre dentro del rango cromático del reino. Un demonio en el Reino de Invierno no usa colores cálidos — viola el frío del reino.

*Design test:* Toma una screenshot de cualquier reino. Sin leer el nombre, ¿qué sientes sobre ese lugar solo por los colores? ¿Miedo? ¿Calidez? ¿Frío? Si el jugador "lee" el mood del reino instantáneamente por color, la paleta funciona.

---

### Principle 2: Animación como Claridad Narrativa
*(Serves Pillar 1: Narrativa Cinematográfica)*

Cada movimiento comunica. Las animaciones priorizan el sentimiento sobre el detalle. El jugador debe entender qué está pasando — emocional y mecánicamente — solo por el movimiento. Esto es especialmente crítico para las habilidades de demonios: la animación *es* la firma visual de la habilidad.

*Design test:* Muestra la animación de una habilidad de demonio a alguien que no conoce el juego. ¿Pueden adivinar qué hace (sanar, dañar, manipular, observar)? Si sí, la claridad animada es correcta.

---

### Principle 3: Violación de Gramática para Demonios
*(Serves Pillar 2: Demonios como Poder Transformador)*

El jugador aprende las reglas visuales del mundo primero. Los demonios rompen esas reglas. Un demonio podría usar colores antinaturales, moverse con fluidez inhumana, o violar las reglas de escala de píxeles establecidas para los NPCs. Esta violación visual se interpreta como "algo fundamentalmente incorrecto y poderoso" — que es exactamente lo que es un demonio.

*Design test:* Para cada diseño de demonio, nombra la regla que rompe. Si un demonio no viola una norma establecida, no es visualmente distintivo.

---

## 2. Mood & Atmosphere

*Governed by: "If it does not carry weight, it does not merit light."*  
*All six states exist on a single emotional continuum. The world is grim, not hopeless — beauty earns its right to exist alongside darkness.*

---

### 2.1 Exploration

**Primary emotional target:** Melancholic wonder. The player feels like a trespasser inside someone else's tragedy — aware that this kingdom was once alive, and aware that they are partly responsible for what it is becoming.

**Lighting character:** Low ambient, side-lit. Light sources are diegetic and sparse: torches, windows, afternoon sun filtered through ruin arches. Color temperature follows the kingdom accent (amber in Kingdom 1, icy blue in Kingdom 3, etc.) but is desaturated to 40-60% of maximum. Mid-value ceilings hold. No bloom. Shadows are long and directional — late-afternoon sun angle regardless of time of day, because it communicates "running out of time" without narration.

**Atmospheric descriptors:** Inhabited ruin. Quiet grief. Inhabited stillness. Textured silence. Earned solitude.

**Energy level:** Contemplative with slow pulse. The world breathes at a resting rate — torch flicker on a 3-5 second cycle, distant NPC movement at the edge of the screen, leaves or debris on ambient parallax that never calls attention to itself.

**One visual element that carries the mood:** The background layer always contains exactly one window or opening with warm light visible through it, unreachable by the player. It signals that life exists elsewhere, that this space was once lived in, that the world is larger than Edrick's path through it. It costs one background tile. It earns the entire emotional register of the screen.

---

### 2.2 Combat

Light tells the player where danger lives. During exploration, ambient light is diffuse and mid-value — environments breathe at rest. When combat begins, ambient light reduces by 30-40% over a **0.5-second fade**, pulling the world back and leaving only the combatants and their skill effects with full luminance. The battlefield contracts visually to the size of what matters.

Color temperature shifts slightly cooler on the fade — warmth is peace; desaturation is threat. Skill animations are the only source of saturated color in combat. This means every ability reads against a darker, greyer field without competing with ambient decoration.

The 0.5-second transition is slow enough to feel like a shift in air pressure, not a technical toggle. The player does not notice the mechanic — they feel the atmosphere change.

When combat ends, light fades back at the same rate. The return to warmth is a relief beat, mechanically inert but emotionally necessary.

**Atmospheric descriptors:** Percussive. Tight. Merciless. Brief. Earned.

**Energy level:** Frenetic in execution, measured in structure. The tempo is high, but every ability has a read frame — one frame of held anticipation before the action resolves. This frame is the "cost visible" moment. Combat never feels random or unreadable.

**One visual element that carries the mood:** Edrick's shadow during combat is always slightly larger than his sprite and leads him by half a step in the direction of his attack vector. It is a subtle, sub-pixel displacement. It communicates that something in him is already ahead of his body — that the violence comes from somewhere deeper and faster than his conscious mind. No tooltip required. The corruption principle from Principle 1 is active even in early-game combat.

---

### 2.3 Demonio Binding

**Primary emotional target:** Transgressive awe. This is wrong, and it is magnificent, and the player has chosen it fully knowing both. The feeling is irreversibility — something that cannot be undone has just been done.

**Lighting character:** Full violation of the kingdom's visual grammar. The binding sequence is the single moment in the game where the ambient palette is overridden entirely. The demonio's "wrong color" expands from its silhouette and floods the frame. Contrast inverts briefly — shadows become light sources, highlights become voids. Duration: 3-6 seconds, then collapses back to the kingdom palette, but Edrick's sprite has changed (one new color channel that was not there before, dynamically updated). The world reasserts itself; Edrick reflects what he has chosen to channel.

**Atmospheric descriptors:** Searing. Silent. Inevitable. Transformative. Strange.

**Energy level:** Oppressive stillness. All ambient motion stops for the duration of the binding. The torch does not flicker. Parallax halts. Nothing moves except the demonio and Edrick. The world is holding its breath. There is no UI during this sequence — not even health. Only the two figures and the grammar-breaking light.

**One visual element that carries the mood:** At the exact moment the binding completes, one frame of Edrick's idle animation changes. A specific pixel cluster — 3-5 pixels on his silhouette — shifts from his base palette to a color that belongs to the demonio. It is barely perceptible in motion. But players who look closely will see it before the game ever tells them "Edrick is being transformed." This is the visual foundation of the entire demonio system: the art communicates before the narrative does.

---

### 2.4 NPC Dialogue

**Primary emotional target:** Cautious intimacy. The player is an outsider who is permitted, briefly, to matter to someone. NPCs do not trust Edrick — or they did once and should not anymore. Every dialogue carries the subtext: "I see what you're becoming, and I'm not sure I should be telling you this."

**Lighting character:** Close and warm, intentionally small in radius. The dialogue frame isolates characters in a cone of proximity light — 2-3 pixel feathered vignette at screen edges. The kingdom's ambient light persists but dims to 70%, putting the focus on faces (or posture, in pixel art terms). Warm color temperature even in cold kingdoms — human warmth is the specific thing being spent in this moment.

**Atmospheric descriptors:** Taut. Confessional. Guarded. Textured. Close.

**Energy level:** Measured. Breathing-pace. No ambient motion except character idle animations, which are reduced to their minimum loop — a single breath cycle. The world waits while two people speak. This is the visual equivalent of lowering one's voice.

**One visual element that carries the mood:** During dialogue, the NPC's idle animation has a single frame — the resting frame, not a motion peak — where their gaze direction is not quite at Edrick. It is a one-pixel offset in the eye cluster. It communicates unease or withheld knowledge without a single word of dialogue needing to carry that weight. This is Principle 2 (Motion Is Meaning) applied to stillness: the absence of full eye contact is the signal.

---

### 2.5 Menu / UI (Bestiario)

**Primary emotional target:** Archaeological investment. The player feels like a scholar cataloguing dangerous things — proud of their collection, aware of its cost, drawn deeper by what remains undiscovered. The Bestiario is not a menu; it is a record of scars.

**Lighting character:** Paper-and-candlelight. Warm amber, extremely low contrast, high texture. The Bestiario is conceived as a handwritten manuscript — parchment-toned background (dark ochre, not white), ink-line illustrations of demonios, marginalia that expands as the player progresses. No in-world lighting simulation here; the UI exists in a liminal frame-within-frame space, lit as if by a single nearby candle off-screen left. Soft vignette. No harsh lines.

**Atmospheric descriptors:** Antiquarian. Obsessive. Quiet authority. Earned knowledge. Candlelit.

**Energy level:** Still. No animation in the Bestiario except ink that appears to dry as new entries are written in — a 0.5-second ink-spread effect on newly discovered demonios. Nothing else moves. This is a space of reflection, not action.

**One visual element that carries the mood — Heraldic Marks:**

Cada entrada de demonio lleva una marca heráldica que registra su historia, no su taxonomía. La marca es la historia:

- **Demonios con un portador anterior conocido:** muestran el sello de la casa de ese portador — Velmar, Blackhorn, Asterion, y otros. El sello se renderiza en el estilo de tinta del manuscrito, no como un emblema limpio. Un sello Velmar que está agrietado o manchado comunica historia sin requerir texto.

- **Demonios errantes** — aquellos sin portador registrado — llevan una marca especial distinta de todos los sellos de casa: un símbolo de soledad y caos desmoronado. Es visualmente sobrio, llevando el peso de la ausencia en lugar de linaje.

- **El demonio del gato:** muestra un sello que no pertenece a ninguna casa conocida, ni existe en ningún registro de portadores. Hay leyendas antiguas sobre un demonio errante de naturaleza desconocida — algunos dicen que fue visto hace siglos, otros que es pura ficción. Su sello heráldico es visualmente único y desasosegante: algo que podría ser real, o podría ser una alucinación compartida de la historia. Los jugadores lo verán, registrarán que es diferente, y tendrán que viajar a través del juego sin saber si está catalogando un mito o algo que debería haber sido imposible. Su significado y la verdad de su existencia se revelan solo late-game.

El sistema heráldico recompensa a los jugadores que leen el mundo cuidadosamente. El jugador que nota que el sello del gato es diferente está haciendo la pregunta correcta antes de que el juego esté listo para responder.

---

### 2.6 Edrick's Corruption (Progressive Visual Mood Shift)

**Primary emotional target:** Slow unease that only resolves in retrospect. The player should not feel a sudden "corruption moment." They should finish the game and realize the visual language had been telling them for hours. The emotional target at each stage is slightly different: early — *wrongness at the edge of perception*; mid — *recognition without language*; late — *grief without surprise*.

**How it works:** La apariencia de Edrick es un índice vivo de su build actual, no un registro de su historia. Su sprite cambia dinámicamente para reflejar qué demonios está canalizando AHORA — no qué demonios ha usado en el pasado.

Esto se implementa mediante **swapping dinámico de canales**: los canales de color del sprite de Edrick se desplazan en tiempo real según la build activa. Un demonio de fuego desplaza su paleta hacia sangrado naranja-rojo en el borde de silueta. Un demonio de sombra invierte su respuesta especular — absorbe luz en lugar de capturarla. Un demonio del tiempo introduce un desincronización de un frame en su animación ociosa, visible solo para jugadores atentos.

**The implication:** Edrick no tiene corrupción permanente. No está acumulando daño. Está usando algo. El horror de esto se hace claro late-game cuando el jugador realiza que ha usado estas cosas voluntariamente, repetidamente, y lo visual les ha estado diciendo así todo el tiempo.

Un jugador que intercambia todos los demonios ve a Edrick volver a su apariencia base — que, para ese punto en el juego, se lee como su propio tipo de extraño. El Edrick sin adornos no es el Edrick seguro.

**Atmospheric descriptors (late stage):** Wrong geometry. Inherited heaviness. Beautiful corruption. Unnameable loss. Proximity dread.

**Energy level:** Increasingly oppressive without events to trigger it. The background layers compress — a 1-2% parallax reduction that makes the world feel slightly flatter. This is the visual consequence of corruption: the beautiful lush world slowly loses its depth around Edrick.

**One visual element that carries the mood across the whole arc:** In Kingdom 1, the torch in Edrick's starting area has a warm amber flicker. By Kingdom 9, if Edrick stands beneath any torch in any kingdom, the flame color shifts from amber to the primary color of his current demonio palette — just the flame, just for Edrick's proximity radius. No other character triggers this. No text explains it. Fire recognizes what he is.

---

## 3. Shape Language

*Governed by: "If it does not carry weight, it does not merit light."*  
*All shape decisions must pass the grammar test: does this shape communicate something the player needs to understand? If not, it does not belong.*

---

### 3.1 Character Silhouette Philosophy

**Core rule:** Every character archetype must read as a distinct shape at 32x32 pixels with no color information — thumbnail silhouette is the first and final test of a character design.

**Archetype shape vocabulary:**

- **Edrick (base):** Vertical emphasis with narrow waist and slightly top-heavy shoulders. A young noble's posture — controlled, upright, but not yet earned. His silhouette reads as a compressed triangle pointing upward: ambition without mass. The base design deliberately lacks visual complexity at the edges; his outline is clean and legible. This cleanliness is intentional — it gives the demonio system room to *add* visual noise over time without starting from clutter.

- **Edrick (demonio-active):** The silhouette acquires peripheral disturbances specific to the bound demonio. A fire demonio extends heat shimmer at shoulder edges — 2-4 pixels of semi-transparent fringe that breaks the clean outline. A shadow demonio makes his cast shadow slightly wider than his body geometry allows. A time demonio introduces a trailing ghost of his last-frame position, offset 1-2 pixels. In all cases: the disturbance happens at the *edge* of the silhouette, not the interior — it reads as something pressing outward from inside. This serves Pillar 2 (Demonios como Poder Transformador): the power is literally reshaping the boundary of who he is.

- **NPCs (civilians, allies):** Round and horizontal — barrel-shaped torsos, low center of gravity, wider stance. These are people who *belong* somewhere. Their silhouettes communicate rootedness. A merchant is wider than tall; a village elder has curved shoulders that read as the weight of years, not strength. They contrast with Edrick's verticality: the world is grounded, he is straining.

- **Enemies (corrupted, soldiers, bound creatures):** Asymmetry is the marker. Normal enemies have slightly asymmetrical silhouettes — one shoulder higher, one hand heavier. This distinguishes them from NPCs (symmetrical groundedness) and communicates latent threat without animation. Fully corrupted enemies have extreme silhouette irregularity: protrusions, collapsed limbs, shapes that don't resolve into recognizable body parts at thumbnail size. The corruption gradient from soldier to creature is legible in silhouette alone.

- **The Cat:** Deliberately deceptive silhouette. In resting state he reads as a normal cat — compact, four-legged, clean arching back. In combat he elongates impossibly: a silhouette that is too long for any cat, too fluid in its transitions between poses. This visual dissonance is the first signal players receive that something is different about this companion. His combat silhouette violates the physics of the natural world (Principle 3: Demonios Break the Visual Grammar), and he does this from the moment they meet — before any narrative reveal.

**Visual test — The Thumbnail Silhouette Check:** Export any character sprite as a pure black silhouette on white, scale to 32x32 pixels, place all characters side by side. Every archetype must be distinguishable from every other. If two characters are indistinguishable at that scale, one of them requires redesign before any color or texture work proceeds.

*(Serves Pillar 1: Narrativa Cinematografica — the player reads character before they read dialogue.)*

---

### 3.2 Environment Geometry

**Core rule:** The world uses angular, rectilinear geometry as its baseline — and reads that geometry as *human construction under stress*. Nothing in the built world is perfectly right-angled; everything has settled, cracked, or been reclaimed.

**Why angular:** Medieval architecture is fundamentally geometric — stone blocks, arches, towers, walls. This geometry communicates order, hierarchy, the ambition of institutions. The kingdoms were once coherent, organized, powerful. Their base geometry reflects that: grid-aligned environments, regular tile sizes, structural clarity in the read.

**Why stress matters:** The angular geometry is systematically *interrupted* by organic intrusion. Vines on walls do not grow in straight lines. Cracks in stone floors do not follow grid seams. Rubble does not stack symmetrically. The environment reads as geometric structure being slowly reclaimed by organic growth or erosion — this communicates *decay without nihilism*. The institutions existed. They are falling. But falling implies they stood.

**Specific guidance by layer:**

- **Background layers (parallax 3, 4):** Soft and organic. Distant mountains, treelines, sky elements use curved, rounded shapes. These layers breathe; they are the natural world behind the constructed one. Curves here. Low contrast edges.

- **Midground environment (parallax 2):** Structural geometry with organic interrupt. A castle wall is rectilinear but has moss clusters at the base (organic blobs, rounded forms) and cracks that run diagonally against the grid. The ratio is approximately 70% angular to 30% organic intrusion.

- **Foreground interactive layer (parallax 1):** Primarily angular because this is where the player navigates — legibility demands clean geometry for platforms, edges, passages. However, a single organic element per screen section (a root pushing through a floor tile, a collapsed column with rubble scatter) marks that the world has history beyond the player's path.

- **Demonio encounter zones:** Geometry becomes irregular and non-Euclidean in the immediate area of a demonio encounter. Angles that do not add up to 360 degrees around a vertex. A doorframe that is slightly too tall on one side. A floor tile cluster where the tiles are the right shape but cannot possibly tessellate the way they appear to. This is the environmental expression of Principle 3 (grammar violation): the demonio's presence is warping the logic of the space before the player sees it.

**Medieval reference calibration:** The target is *Game of Thrones* interior architecture — massive stone, weight communicated through scale and shadow, a sense that these structures took lifetimes to build and are beginning to outlast their purpose. Not *Castlevania*'s exuberant gothic excess; not *Hollow Knight*'s entirely organic cavernous softness. The midpoint: structured order under organic pressure.

**Visual test — The Geometry Autopsy:** Take a screenshot of any completed environment screen. Trace all visible edges with a vector tool. Classify each edge as angular (straight, grid-aligned, or at a consistent angle) or organic (curved, irregular, non-repeating). The ratio should be 65-75% angular to 25-35% organic in the foreground and midground combined. If organic forms dominate, the world reads as a cave, not a ruin. If angular forms dominate without interruption, the world reads as pristine, which contradicts the decay-and-survival narrative.

*(Serves Pillar 3: Mundo Hermoso y Vivo — the tension between construction and reclamation is the visual argument that this world was once worth building.)*

---

### 3.3 UI Shape Grammar

**Core rule:** The UI is not neutral. Its shapes must feel like objects that exist in the world of Demons Of Dravaryn — specifically, like objects a scholar or record-keeper of this world would have made.

**The governing aesthetic: manuscript and instrument.** Every UI element should feel as though it could have been drawn by hand in ink or cast in iron. This produces two allowed shape categories:

- **Ink-drawn forms** (Bestiario, dialogue frames, status text): Slightly irregular borders with ink-weight variation at corners. The border of a dialogue box is not a perfect rectangle — it has slightly heavier line weight at corners as if a quill slowed there. Text containers have a faint ruled-line texture, as manuscript paper has. No perfect circles; ellipses with slight asymmetry.

- **Forged/cast forms** (HUD health indicators, skill slot frames, ability cooldown rings): Angular and heavy, as if cut from metal or stamped into leather. Thick borders. Corner brackets rather than rounded corners. Right angles with inward chamfers. These communicate permanence and cost — the HUD exists in the same material world as the armor Edrick wears.

**What UI shapes must NOT do:**

- No glowing rounded rectangles (this reads as contemporary digital UI, not medieval craft)
- No pure geometric circles used as decorative elements (circles belong to demonio grammar — any perfect circle in the UI will read as demonio-related, which is a signal reserved for the Bestiario's demonio entries specifically)
- No drop shadows used for depth — depth is conveyed through overlap and scale, not soft-shadow effects. Soft shadows are reserved for the lighting system, not UI.

**Demonio entries in the Bestiario are the exception.** Demonio-specific UI frames deliberately violate the manuscript/instrument grammar: their borders pulse at irregular intervals, use shapes that don't resolve cleanly (an almost-circle, a border that closes at a slightly wrong angle), and include a color channel that doesn't exist elsewhere in the UI system. This is intentional grammar violation — the Bestiario is a human-made record of something inhuman, and the inhuman nature bleeds through the human format.

**Visual test — The Period Immersion Test:** Show the UI at full scale to someone familiar with the game's visual language and ask: "Does anything in this UI look like it was made after the year 1500?" If yes, identify the specific element and correct the shape language before proceeding to color. The test is strict — a rounded rectangle with modern proportions fails this test even at the right color.

*(Serves Pillar 1: Narrativa Cinematografica — the UI is part of the world's visual grammar, not a layer placed over it.)*

---

### 3.4 Hero Shapes vs. Supporting Shapes — Visual Hierarchy in Scene

**Core rule:** The player's eye must find Edrick within 200 milliseconds of any screen transition, and must understand where danger lives within 500 milliseconds of entering any new screen. Shape complexity and silhouette contrast are the primary tools — not color alone.

**The hero shape contract:**

Edrick occupies the single most visually distinct silhouette in any scene. This is maintained through three mechanisms:

1. **Silhouette contrast with background:** Edrick's outline must always contrast with the pixel immediately behind it. The environment art team must review every background zone for value proximity — if the background mid-layer value is 40% grey, Edrick's silhouette pixels in that area must not be between 30-50% grey. A 20% minimum value gap is required. This is the animator's and environment artist's shared responsibility, resolved during integration review.

2. **Shape specificity:** Edrick's silhouette contains specific shapes that do not appear in background geometry: the shoulder line at his particular angle, the slight forward lean of his idle stance, the visible pommel of his weapon. These shapes are unique to him. Background elements — columns, walls, NPC silhouettes — are designed to *not* replicate these specific shapes. This prevents visual merging even when color contrast is low (dark scene, dark kingdom).

3. **Movement ownership:** When Edrick is the only element in motion in a still scene, the eye goes to him automatically (Gestalt figure-ground). The ambient motion system (Principle 2 from the Visual Identity Statement) is designed to keep background motion at sub-peripheral attention levels — slow enough that it doesn't compete with Edrick's deliberate movement. The rule is: if Edrick is standing still, the background can breathe. When Edrick moves, ambient motion reduces by 50% in the immediate surrounding area (a 3-screen-width radius), yielding visual dominance to the character.

**Supporting shapes and their role:**

Supporting environmental elements — background columns, ground tiles, distant architecture — use shapes that are *complete but unresolved*. They read as full objects, but they don't pull the eye because they lack specificity. A column in the background is a rectangle with a capital; it is legible as a column, but it has no detail that rewards scrutiny. The player registers it as "column" and their attention moves on. This is the Gestalt principle of figure-ground: the background earns its right to exist by being readable without being interesting.

NPC silhouettes are designed to be distinguishable from Edrick but not competitive. They are wider (horizontal emphasis vs. Edrick's vertical), shorter in the torso (lower center of gravity), and positioned in screen zones that are compositionally secondary — off-center, in foreground or background layer rather than the main action plane.

**The danger-read requirement:**

Enemies must be visually distinct from NPCs at first glance. The shape markers are:
- Asymmetrical silhouette (enemies) vs. symmetrical (civilians)
- Forward lean in idle stance (enemies lean into the player's position) vs. upright or away (civilians)
- Interrupted silhouette edges (corrupted enemies have protrusions that break the outline) vs. clean edges (civilians are continuous outlines)

The 500-millisecond danger-read test is: place an enemy in any completed environment screen. A player who has not seen that enemy before must be able to identify it as a threat (not an NPC, not a decorative element) within half a second of the screen appearing. Shape alone must carry this — the test is run in greyscale.

**Visual test — The Greyscale Hierarchy Test:** Convert any completed scene to greyscale and identify, in order, the first three elements the eye visits. The sequence must be: (1) Edrick, (2) the primary threat or point of interest, (3) the path or exit. If any other element appears before Edrick, the scene's visual hierarchy is inverted and must be corrected at the shape-and-value level before color work continues.

*(Serves Pillar 5: Transformacion Moral de Edrick — the player must remain connected to Edrick as the central figure even as his shape increasingly violates the visual grammar of the world around him. When Edrick becomes harder to read against the world he has corrupted, that difficulty is intentional and signals the late-game transformation — but the progression to that point must be gradual, not sudden.)*

---

## 4. Sistema de Colores

*Gobernado por: "Si no tiene peso, no merece luz."*  
*Cada color debe ganar su brillo a través de significado narrativo o mecánico.*

---

### 4.1 Paleta Primaria por Reino

Cada uno de los 9 reinos utiliza una identidad cromática diferenciada. El sistema base (grises, negros, marrones terrosos) permanece constante, pero cada reino introduce un **color acento dominante** que comunica su naturaleza sin texto. Este acento define el rango permitido para todos los colores saturados que aparecen en ese reino.

**Nota sobre nombres de reino:** Los siguientes son nombres en clave (Ruinas Nobles, Putrefacción, Invierno, etc.). Los nombres narrativos verdaderos de los reinos se definirán en documentos posteriores. Sin embargo, dos de estos reinos tienen importancia narrativa particular:
- **Reino de origen:** El reino que gobernaba la casa de Edrick, del cual huye
- **Reino de inicio:** El reino donde comienza la historia, después del exterminio de su familia

---

**Reino 1 - Ruinas Nobles (Ámbar/Oro Apagado) [Nombre en clave]**
- Emoción: Decadencia, gloria pasada, instituciones caídas
- Rango de acento: Oros y ámares desaturados (40-60% saturación máxima)
- Significado narrativo: La riqueza existe solo en la memoria
- Colores secundarios permitidos: Marrones cálidos, herrumbre roja (óxido, corrosión)
- Transformaciones demoniacas aquí: Los demonios que habitan este reino dejarán marcas cromáticas que contrastan con la frialdad dorada — transformaciones que comunican poder a través de la desviación del acento del reino

**Reino 2 - Putrefacción (Verde Enfermizo) [Nombre en clave]**
- Emoción: Peligro biológico, enfermedad, corrupción viviente
- Rango de acento: Verdes desgastados (50-70% saturación, siempre con matiz de mucosidad)
- Significado narrativo: Lo viviente está consumiendo lo construido
- Colores secundarios permitidos: Grises verdosos, negros con tinte verde, amarillos desaturados (pus)
- Transformaciones demoniacas aquí: Los demonios marcan a sus portadores con colores que niegan la decadencia biológica — purezas cromáticas que señalan poder sobrenatural

**Reino 3 - Invierno (Azul Gélido y Blanco) [Nombre en clave]**
- Emoción: Frío extremo, aislamiento, preservación en hielo
- Rango de acento: Azules fríos (40-60% saturación, siempre cianoso, nunca cálido)
- Significado narrativo: El calor es supervivencia; su ausencia es la regla
- Colores secundarios permitidos: Blancos gélidos, grises azulados, violetas muy pálidos
- Transformaciones demoniacas aquí: Los demonios introducen calidez donde el reino niega toda calidez — su marca visual es la antítesis cromática del frío

**Reino 4 - Cosecha (Naranja Cálido) [Nombre en clave]**
- Emoción: Abundancia, seguridad relativa, otoño perpetuo
- Rango de acento: Naranjas y ocres cálidos (45-65% saturación, siempre luminosos)
- Significado narrativo: Este reino es el más cercano a la seguridad; es un refugio momentáneo
- Colores secundarios permitidos: Rojos cálidos, marrones dorados, amarillos suaves
- Transformaciones demoniacas aquí: Los demonios marcan con colores que rompen la calidez confortable — frialdad o extrañeza cromática que señala que nada en este reino es realmente seguro

**Reinos 5-9 [Nombres en clave]:**
- Reino 5: Vorágine de Piedra (Gris Metálico)
- Reino 6: Marisma Sagrada (Púrpura Enfermizo)
- Reino 7: Catedral Sumergida (Azul Profundo)
- Reino 8: Bosque Ardiente (Rojo Oscuro)
- Reino 9: El Trono de Draeven (Oro Negro)

*Estructura de reino: A medida que se definen narrativamente los reinos verdaderos, sus nombres en clave serán reemplazados, pero sus identidades cromáticas y principios visuales permanecen.*

---

### 4.2 Semántica del Color — Lo que los Colores Comunican

**En el mundo (no en UI):**

- **Oro/Ámbar:** Poder institucional decadente, herencia noble, historia
  - Aparece en: coronas, sellos de casas, reliquias, objetos heredados
  - Nunca aparece como color base ambiental excepto en Reino 1 — siempre ganado

- **Verde:** Corrupción biológica, crecimiento anormal, enfermedad
  - Aparece en: vides que consumen ruinas, luz emitida por criaturas corrompidas, marcas de transformación demoniacas en este contexto
  - Si el jugador ve verde que no es Reino 2 o transformación demoniacal, algo está mal

- **Azul:** Frío, magia, lo no-vivo, lo preservado
  - Aparece en: nieve, hielo, efectos de transformación demoniacal que rompen la calidez local
  - Frío es seguridad en el Invierno; es amenaza en otros reinos

- **Rojo/Naranja:** Violencia, pasión, consumo, transformación por fuego
  - Aparece en: efectos de transformación demoniacal violenta, sangre (mostrada raramente), luz de peligro extremo
  - Saturación alta = peligro inmediato; saturación baja = recuerdo de violencia

- **Negro/Gris Oscuro:** Vacío, sombra, lo desconocido
  - Aparece en: sombras (nunca con color falso), zonas no exploradas, posibles transformaciones demoniacales que nieguen la luz
  - La sombra es siempre literal, nunca emocional

- **Púrpura:** Lo arcano, lo demonológico, lo que está entre mundos
  - Aparece SOLO en: demonio bindings, páginas especiales del Bestiario, transformaciones demoniacales especiales
  - Nunca en ambiente natural — es siempre anómalo

---

### 4.3 Estructura de Luminancia — El Peso Gana Luz

La saturación y el brillo siguen una regla única: **nada es brillante sin justificación narrativa.**

**Niveles de ganancia de luz:**

1. **Base ambiental (30-45% de luminancia máxima):** Grises, marrones, desaturados. El mundo respira en reposo. Este es el nivel de la mayoría de los píxeles en pantalla.

2. **Detalles arquitectónicos (45-60% de luminancia):** Piedra más clara, maderas destacadas, herrumbre, musgo. Estos elementos comunican historia — fueron construidos o crecieron con intención.

3. **Elementos narrativos (65-80% de luminancia):** Objetos que el jugador puede recoger, plataformas críticas, NPCs. Estos ganan brillo porque importan mecánicamente o narrativamente.

4. **Demonio binding (85%+ de luminancia, toda la saturación permitida):** El único momento en que la saturación completa es permitida es durante la secuencia de binding. El demonio no está restringido por la paleta del reino — es la violación fundamental. Las transformaciones demoniacales en el cuerpo humano del portador permanecen legibles como humanas, pero ganan brillo y saturación como marca visual de posesión.

5. **Efectos de habilidades en combate (90-100% de luminancia con color completo):** Las únicas cosas más brillantes que un demonio binding son los efectos visuales de las habilidades durante combate. Aquí, la saturación completa y el brillo máximo están justificados — el jugador está actuando, gastando energía.

**Implicación:** Una transformación demoniacal permanentemente visible en el cuerpo del portador nunca es tan brillante como la explosión visual de una habilidad. Esto comunica que las habilidades cuestan algo — su brillo es el costo visible.

**Clarificación importante:** Los demonios no son seres mostruosos en aspecto. Requieren un cuerpo humano (o en principios humanoides) para manifestarse. Las transformaciones que dejan en sus portadores son marcas cromáticas, cambios de luminancia y alteraciones visuales sutiles que permanecen dentro de la legibilidad de una "forma humana reconocible." Un portador nunca se vuelve visualmente inhumano — se vuelve inequívocamente transformado, pero sigue siendo claramente una persona.

---

### 4.4 Paleta UI (Bestiario y Menús)

**Paleta del manuscrito:**
- Fondo: Ochre oscuro (aproximadamente RGB 90, 75, 60)
- Texto: Sepia (RGB 180, 160, 140)
- Líneas decorativas: Marrones oscuros (RGB 60, 45, 30)
- Sin colores saturados excepto en demonio entries

**Demonio entries (Excepciones intencionales):**
- Cada entrada de demonio usa un color secundario que pertenece al contexto donde el demonio fue encontrado o vinculado
- Los demonios errantes usan un púrpura opaco (la falta de hogar en color)
- Demonio singular o legendario: color que no existe en la paleta UI normal — un plateado-púrpura iridiscente que cambia ligeramente según el ángulo de "lectura"

**Ningún color de UI coincide exactamente con colores ambientales.** El Bestiario existe en un espacio liminario — no es el mundo, es un registro del mundo. La textura del pergamino (leve patrón de tela) y la ausencia de luz de ventana comunican que es un espacio aparte.

---

### 4.5 Seguridad para Daltónicos

**Problema:** Las diferentes transformaciones demoniacales podrían ser indistinguibles solo por color para jugadores con daltonismo.

**Solución — Código de Saturación + Forma + Movimiento:**

Cada demonio, una vez definido, deberá incluir en su especificación visual:

1. **Patrón visual distintivo** (teselación, ondulación, deformación de píxeles animada) que sea legible incluso en escala de grises
2. **Alteración de movimiento** (velocidad de animación, cadencia, desincronización de frames) que sea visible sin información cromática
3. **Transformación de forma** en el cuerpo del portador (simetría rota, protuberancias sutiles, cambios en postura) que comunique tipo demoniacal

**Costo audible:** Cada demonio, una vez definido, tendrá una firma sonora única durante el binding — el cambio visual es reforzado por una distorsión de audio. Un jugador sordo nunca pierde información por color solo; un jugador daltónico nunca pierde información por falta de patrón distintivo.

---

### 4.6 Transformaciones Visuales de Edrick

**Sistema dinámico, no acumulativo:**

El cuerpo de Edrick refleja su build actual de demonio(s), no su historial. Cada demonio aplicará transformaciones visuales propias a su sprite:

- **Transformación individual:** Cada demonio, cuando se defina, incluirá su propia paleta de cambios cromáticos, alteraciones de luminancia, o modificaciones de forma en el cuerpo de Edrick
- **Transformaciones combinadas:** Ciertas combinaciones de demonios pueden generar transformaciones visuales emergentes — sinergia visual que va más allá de las dos transformaciones individuales superpuestas
- **Reversal:** Si Edrick descarga todos los demonios, su sprite regresa a su paleta base completamente — un personaje descolorido es el Edrick más extraño, visualmente sin poder ni identidad

**Late-game implicación:** En los reinos finales, las torches y fuentes de luz ambiental pueden cambiar su color cromático para coincidir con la transformación demoniacal activa de Edrick si está dentro del radio de proximidad (aproximadamente 3 tiles). Es el mundo reconociendo — y reflejando — lo que ha elegido ser.

---

**Tests de validación del Color System:**

1. **Test de Lectura de Reino:** Muestra screenshots de cada reino sin nombres. ¿Puede un jugador describir la atmósfera solo por paleta?
2. **Test de Distinción de Demonio:** Presenta dos transformaciones demoniacales diferentes (daltonismo simulado incluido). ¿Pueden ser distinguidas por forma, patrón o movimiento además de color?
3. **Test de Jerarquía de Luminancia:** Cuenta píxeles de máxima saturación en una pantalla de exploración. Deberían ser <2% de la pantalla (ganados solo por objetos narrativos).
4. **Test de Humanidad:** Verifica que todas las transformaciones demoniacales mantienen la legibilidad del cuerpo humano como forma base — si algo parece completamente inhuman o monstruoso, requiere revisión.

---

## 5. Dirección de Diseño de Personajes

*Gobernado por: "Si no tiene peso, no merece luz."*  
*Cada personaje es un arquetipo visual — legible a 32x32 pixels en silhueta pura.*

---

### 5.1 Edrick Velmar — Protagonista

**Silhueta base:** Triángulo vertical comprimido — hombros estrechos, cintura aún más estrecha, postura erguida pero sin peso acumulado. Es la forma de un joven noble que nunca ha trabajado un día en su vida, cuya ambición supera su masa corporal. Legible en thumbnail: se lee como "joven, ambicioso, sin experiencia."

**Apariencia visual vs. realidad interior:** Aunque Edrick viste y vive como un plebeyo — ropa de viaje desgastada, tonos grises y marrones terrosos, telas que muestran uso pero no pobreza extrema — su lenguaje corporal nunca pierde completamente la educación noble. Los detalles visuales comunican esta dualidad:
- Su postura mantiene una elegancia mínima incluso mientras descansa
- Ocasionalmente visible: el forro interior de mejor calidad bajo ropa exterior desgastada
- Sus manos, aunque endurecidas por el viaje, nunca muestran las marcas de trabajo manual verdadero

**Inteligencia y cultura:** Edrick es profundamente culto — leyó, escribió, estudió estrategia, historia y diplomacia bajo tutores nobles. Esto debe comunicarse en sus diálogos: vocabulario sofisticado, referencias a conceptos complejos, capacidad de razonamiento estratégico. Es un contraste visual con su aspecto de vagabundo — el jugador aprende rápidamente que la apariencia de Edrick es un camuflaje, no su verdadera naturaleza.

**Propósito narrativo visual:** Edrick debe ser encontrable en cualquier pantalla dentro de 200ms. Su silueta vertical lo distingue de NPCs (horizontales, arraigados) y de enemigos (asimétricos, amenazantes). A medida que se transforma por demonios, su silueta adquiere disturbios periféricos, pero permanece reconociblemente el mismo cuerpo subyacente. El horror late-game es visual: la forma que reconocemos se vuelve cada vez menos segura de sí misma.

**Edad y aspecto:** Aproximadamente 22-25 años. Rasgos de nobleza clara pero juventud. Cara que podría ser cruel o compasiva dependiendo de la animación — el rostro como máscara que cambia con la moral.

---

### 5.2 NPCs — Civiles y Aliados

**Silhueta base:** Horizontal y redonda — torsos de forma de barril, cintura más ancha que los hombros, centro de gravedad bajo. Legible en thumbnail: se lee como "arraigado, belongs aquí, acumuló años."

**Jerarquía de diseño según importancia narrativa:**

**NPCs menores (sin peso en la trama):**
- Silueta y paleta distintivas por rol (comerciante, guardia, campesino), pero genéricas dentro de ese rol
- Sin detalles faciales específicos — si la cara es visible, es simple y fácilmente repetible
- Animaciones básicas y reutilizables
- Propósito: Poblar el mundo. El jugador no los recuerda individualmente

**NPCs con peso narrativo (aliados, confidentes, figuras pivotales):**
- Silueta única dentro de su rol — características que los hacen inmediatamente reconocibles
- Detalles faciales específicos y memorables — un cicatriz, un tatuaje, una asimetría característica, ojos de color único
- Animaciones customizadas que reflejan su personalidad (movimiento cauteloso, gesticulación expansiva, rigidez, fluidez)
- Paleta cromática personal — colores que los siguen a través de regiones
- Propósito: Personajes con los que el jugador forma conexión. El jugador los recuerda, reconoce, y siente el peso de sus interacciones

**Variedad por rol (NPCs menores):**
- **Comerciantes:** Anchos en la parte media, hombros más estrechos. Silueta que comunica equilibrio de peso sobre la mercancía, no sobre la fuerza
- **Ancianos/Ancianas:** Hombros ligeramente redondeados, postura que se inclina marginalmente hacia adelante. El peso de los años es literal en la forma
- **Trabajadores:** Brazos más definidos (visibles en silueta), postura de trabajo — ligeramente inclinados, como si siempre estuvieran haciendo algo

**Paleta general:** Colores pertenecientes al reino donde viven. Un NPC en Reino 1 usa tonos ámbar/marrón; en Reino 3 usa azules y grises. Los colores del NPC nunca contrastan violentamente con su reino — son parte de la trama ambiental.

**Expresión y movimiento:** Las animaciones de NPCs son lentas y metódicas — el movimiento de alguien que conoce su trabajo. Las transiciones de pose son suaves, sin urgencia. Los ojos durante diálogos tienen esa sutil desalineación de un pixel que comunica incomodidad (como se establece en Sección 2.4).

---

### 5.3 Enemigos — Soldados y Criaturas Corrompidas

**Estructura de escala de corrupción:**

Los portadores de demonios son extraordinariamente raros y poderosos. La vast mayoría de enemigos que encuentra Edrick son soldados no corrompidos, bandidos, o bestias naturales del mundo. Solo los jefes de la historia, los guardianes de los reinos, y ciertos enemigos narrativamente significativos serán demonios o estarán corrompidos.

**Distribución esperada:**
- ~85-90% de enemigos: Nivel 1 (sin corrupción)
- ~8-12% de enemigos: Nivel 2 (parcialmente corrompidos)
- ~2-5% de enemigos: Nivel 3 (completamente corrompidos)

---

**Nivel 1 — Soldados y enemigos naturales (sin corrupción):**
- Silhueta: Asimetría leve — un hombro ligeramente más alto, un brazo que cuelga diferente
- Postura: Inclinación hacia adelante en idle, como si siempre estuvieran listo para atacar
- Paleta: Tonos de armadura (grises, hierro oxidado, cuero), sin marcas demoniacales
- Animaciones: Estándar, reutilizable. El movimiento comunica "amenaza" pero no "transgresión"
- Detalles faciales: Generales — estos soldados no son individuos, son fuerzas del mundo
- Propósito visual: Legibilidad de amenaza sin complejidad de diseño
- Legibilidad: Distinguible de NPCs por asimetría; distinguible de criaturas corrompidas por claridad de forma

**Nivel 2 — Parcialmente corrompidos (enemigos narrativamente significativos):**
- Silhueta: Asimetría moderada — protuberancias sutiles en un lado del cuerpo, distorsión de proporción (un brazo o pierna levemente más largo)
- Paleta: Contamina colores del reino con matices que no pertenecen — verde en Reino 1, por ejemplo
- Movimiento: Animaciones que rompen la simetría — giro desigual, desplazamiento de peso en un solo lado
- Detalles faciales: Específicos y memorables — cicatrices, marcas demoniacales alrededor de los ojos, asimetría facial que comunica dolor o transformación
- Animaciones customizadas: El movimiento refleja la lucha interna — ocasionalmente se detiene, como si luchando contra el demonio, luego continúa con movimiento inhumano
- Propósito visual: Estos son personajes — ex-humanos, ahora víctimas del poder demoniacal. El jugador debe reconocerlos como pérdidas, no simplemente como obstáculos
- Legibilidad: Claramente una amenaza, pero aún vagamente humanoide. El jugador siente la tragedia de su existencia

**Nivel 3 — Completamente corrompidos (jefes de historia, guardianes de reino):**
- Silhueta: Extrema irregularidad — formas que no resuelven en un cuerpo reconocible a escala de thumbnail
- Paleta: Colores antinaturales y luminancia errática
- Movimiento: Fluidez inhumana, transiciones que no siguen la física esperada
- Detalles: Completamente customizados — ninguno de estos enemigos es genérico. Cada guardián corrompido es una obra de arte visual oscura con su propio patrón de transformación demoniacal
- Animaciones: Únicas, complejas, que comunican poder y alienación total de la humanidad
- Propósito visual: Pura amenaza — estos son los encuentros climáticos del juego. Visualmente, el jugador entiende inmediatamente que esta es una batalla diferente de cualquier otra
- Legibilidad: Pura amenaza — no hay expectativa de humanidad subyacente

**Nota narrativa:** La corrupción visual es una cuestión de grado, no de tipo. Un soldado corrompido es un soldado que se vuelve monstruoso lentamente — el jugador ve la progresión. Los Nivel 2 y Nivel 3 requieren más iteración de diseño, más detalles únicos, porque cada uno es un jefe o punto de quiebre narrativo. Los Nivel 1 pueden ser reutilizables — son la trama, no el patrón.

---

### 5.4 El Gato — Compañero Misterioso

**En reposo:** Silueta completamente normal de gato blanco. Cuatro patas, lomo arqueado, cabeza pequeña, pelaje que sugiere aseo meticuloso. Legible como un felino doméstico normal a cualquier escala. Paleta: blanco puro, con sombras grises y negras mínimas. Es visualmente el compañero más seguro de Edrick — casi ingenuo en su normalidad.

**En combate — La revelación:** Los ojos del gato cambian a un rojo brillante — no sangre, sino una luminancia interna que comunica poder. Este es el primer y único cambio visual en combate: los ojos. Todo lo demás en el cuerpo del gato se vuelve imposiblemente alargado — cuerpo que fluye, extremidades que se estiran, movimiento que sugiere articulaciones donde ningún animal tiene articulaciones.

**Arquitectura visual del cambio:**
- **Transición:** Instantánea — los ojos pasan de gato normal a rojo brillante en un frame, como un cambio de canal
- **Permanencia:** Los ojos permanecen rojos mientras está en combate. Cuando el combate termina, desvanecen de nuevo a gato blanco normal
- **Implicación narrativa:** Es el primer violador de gramática visual que el jugador experimenta (Sección 3, Principio 3 — Demonios rompen las reglas). El jugador sabe desde el comienzo que algo está mal, visualmente. Cuando el juego revela lo que es, no es sorpresa — es confirmación de lo que sus ojos ya habían visto

**Paleta:** Blanco base, rojo de ojos en combate, pero ningún otro color demoniacal completa el cuerpo. A diferencia de Edrick, el gato no adquiere canales de color del demonio. Solo los ojos revelan lo que lleva.

---

**Tests de validación de personaje:**

1. **Test de silhueta:** Todos los personajes como negro sólido sobre blanco, 32x32. ¿Cada uno es distinto? ¿Rol/amenaza clara sin color?
2. **Test de legibilidad de animación:** Muestra un ataque de enemigo sin efectos visuales. ¿Puede el jugador entender qué sucede solo por movimiento?
3. **Test de encontrabilidad de Edrick:** Coloca Edrick en cualquier pantalla completada. ¿Lo encuentran los ojos dentro de 200ms sin buscar activamente?
4. **Test de distinción de peso:** Presenta dos NPCs — uno menor, uno importante narrativamente. ¿Es visualmente obvio cuál es el personaje que importa?
5. **Test de gato en reposo:** ¿Es el gato completamente indistinguible de un gato doméstico normal cuando sus ojos no son rojos?

---

## 6. Lenguaje de Diseño Ambiental

*Gobernado por: "Si no tiene peso, no merece luz."*  
*El ambiente comunica historia a través de composición, no de decoración. El mundo es diverso — próspero y decadente, vivido y arruinado.*

---

### 6.1 Filosofía Arquitectónica Base

**Regla central:** La arquitectura del mundo es historia visible. Cada muro, cada columna, cada puerta comunica quiénes fueron las personas que construyeron este reino, qué les sucedió, y cómo viven hoy.

**Geometría base:** Angular y rectilineal. La construcción humana es la línea recta — orden, intención, ambición institucional. Los 9 reinos fueron una vez coherentes, estructurados, bajo control. Cada reino refleja esto en su arquitectura base: alineación de grillas, regularidad estructural, proporciones que comunican poder.

**Diversidad es clave:** No todos los reinos están en declive. Algunos están prósperos — mercados activos, construcción nueva al lado de lo antiguo, vida comunitaria evidente. Otros están en decadencia. Otros están en transición. El mundo es medieval — no utópico, pero tampoco uniformemente desesperado.

**Referencia visual de arquitectura:** Game of Thrones (castillos de piedra masiva, peso arquitectónico, escala de poder, pero también vida cotidiana) / Estudios de arqueología medieval (no romantizada, sino vivida) / Hollow Knight como referencia NEGATIVA — no queremos un mundo tan uniformemente muerto. Queremos la edad media: miseria y alegría mezcladas.

**Implicación narrativa:** Este es un mundo donde las personas viven. Algunos reinos están siendo reclamados por el tiempo. Otros son refugios momentáneos de seguridad relativa. La historia visual debe reflejar que estos son lugares donde sucedía la vida humana ordinaria, no solo campos de ruinas.

---

### 6.2 Espectro de Condición — Tres Estados de Reino

En lugar de asumir deterioro uniforme, cada reino (o partes del reino) cae en uno de tres estados:

**Estado 1: Próspero / Mantenido**
- Arquitectura en buen estado de conservación o recientemente reparada
- Orden visible: calles limpias de escombros, puertas con bisagras, techos intactos
- Props comunitarios: mercados activos, tiendas, plazas donde la gente se reúne
- Paleta: Colores vivos dentro de la gama del reino (oros brillantes en Reino 1, verdes vivos en reinos naturales)
- Luz: Luz diurna clara, sombras definidas pero no opresivas
- Intrusión orgánica: Mínima — un árbol ocasional, macetas con plantas, no vides salvajes
- Ejemplo narrativo: "La cosecha fue buena este año. Los comerciantes vinieron. Hay esperanza aquí, aunque sea frágil"

**Estado 2: En Declive / Transición**
- Arquitectura que muestra esfuerzo de mantenimiento pero está perdiendo la batalla
- Mezcla visible: estructuras reparadas al lado de secciones deterioradas
- Props mixtos: herramientas de reparación, andamios, signos de abandono gradual
- Paleta: Colores del reino pero desaturados, menos brillo de lo que deberían tener
- Luz: Luz filtrada, sombras más largas, ángulo de "tarde perpetua" que comunica incertidumbre
- Intrusión orgánica: Moderada — vides en algunos muros, grietas que comienzan a tomar forma, pero la forma humana aún es legible
- Ejemplo narrativo: "Este reino fue próspero. Ahora lucha. Algunos residentes se fueron. Otros no tienen a dónde ir"

**Estado 3: Arruinado / Abandonado**
- Arquitectura colapsada, suelos desnudos, estructuras sin techo
- Poco orden — escombros dispersos, objetos esparcidos sin propósito
- Props escasos — solo reliquias de lo que fue, no signos de mantenimiento actual
- Paleta: Grises, negros, desaturación casi completa del color del reino
- Luz: Baja, ambiental, a menudo filtrada a través de ruinas
- Intrusión orgánica: Extrema — la naturaleza está reclamando el espacio
- Ejemplo narrativo: "Esto fue abandonado hace años. No hay vida aquí excepto el mundo natural reclamando lo que fue quitado"

---

### 6.3 Aplicación por Tipo de Locación

**Ciudades principales de un reino:** Típicamente Estado 1 o Estado 2
- Los mercados están activos
- Las casas tienen puertas cerradas, no solo abiertas vacías
- Hay NPCs viviendo vidas ordinarias — no en peligro perpetuo
- La narrativa del jugador aquí es: "¿Qué sucedió a este reino? ¿Está mejorando o empeorando?"

**Aldeas remotas o fronterizas:** Típicamente Estado 2 o Estado 3
- Lejos del apoyo del poder central
- Más abandonadas, pero no necesariamente todas vacías
- Algunos residentes han permanecido (comunicado por props personales, fuegos en chimeneas)

**Fortalezas y sitios de poder:** Pueden ser cualquier estado, pero comunican control
- Si State 1: El régimen actual tiene control
- Si State 2: El poder se está dispersando
- Si State 3: El régimen anterior fue derrocado

**Ruinas de templos/sitios antiguos:** Típicamente State 3
- Estas locaciones son artefactos históricos, no lugares donde la gente viva actualmente
- Comunicar antigüedad a través de la arquitectura colapsada

---

### 6.4 Ratio Angular/Orgánica por Capa

El principio de Sección 3.2 se detalla aquí, pero con modulación basada en estado:

**Estado 1 (Próspero):**
- Capa de fondo: 100% orgánico (montañas, cielo)
- Capa media: 85-90% angular / 10-15% orgánico (domina la construcción humana, naturaleza ordenada como jardines)
- Capa de primer plano: 90-95% angular / 5-10% orgánico (claridad de plataformas)

**Estado 2 (Transición):**
- Capa de fondo: 100% orgánico
- Capa media: 65-75% angular / 25-35% orgánico (tensión visible entre construcción y reclamación)
- Capa de primer plano: 75-85% angular / 15-25% orgánico

**Estado 3 (Arruinado):**
- Capa de fondo: 100% orgánico
- Capa media: 40-50% angular / 50-60% orgánico (la naturaleza ha ganado, pero la construcción aún es legible como arqueología)
- Capa de primer plano: 50-60% angular / 40-50% orgánico

**Detalle de propósito:**

Un muro de piedra en Estado 1 es recto, limpio, ocasionalmente pintado. El mismo muro en Estado 2 tiene grietas diagonales que rompen la regularidad, musgo en los bordes. El mismo muro en Estado 3 es prácticamente una ruina — más grieta que piedra, completamente consumido por vides.

---

### 6.5 Paleta por Reino — Ejemplos de Estados

*(Estos detalles se complementan con la Sección 4.1 — Color System)*

**Reino 1 - Ruinas Nobles (Ámbar/Oro Apagado):**

*Estado 1 (Próspero):* Oro brillante en techos intactos, calles pavimentadas con precisión, mercaderes con toldos dorados. Luz de tarde atravesando torres intactas.

*Estado 2 (Transición):* Oro desaturado, algunos techos colapsados, calles parcialmente deterioradas pero aún navegables. Luz mezclada — claros y sombras profundas.

*Estado 3 (Arruinado):* Oro prácticamente invisible bajo capas de polvo y vides. Torres que se derrumban. Luz muy baja, filtrada a través de ruinas.

**Reino 4 - Cosecha (Naranja Cálido):**

*Estado 1 (Próspero):* Campos visibles, graneros llenos de cosecha, mercados activos. Casas con jardines. Luz cálida y segura — el rey de los reinos prósperos.

*Estado 2 (Transición):* Campos parcialmente cosechados, graneros medio vacíos, casas con huertos descuidados. Luz aún cálida pero con más sombra.

*Estado 3 (Arruinado):* Campos completamente reclamados por naturaleza salvaje. Graneros vacíos. Luz aún tibia pero melancólica — un recuerdo de abundancia.

---

### 6.6 Densidad de Props y Elementos Decorativos

**Regla de peso:** Cada objeto decorativo debe ganar su presencia narrativa. Los props también comunican estado.

**Densidad por tipo de espacio y estado:**

**Estado 1 (Próspero):**
- Espacios abiertos/plazas: 20-35% tiles con props (mercados, tiendas, gente trabajando)
- Interiores/casas: 35-50% (muebles, herramientas en uso, decoración deliberada)
- Jardines/espacios verdes: 25-40% (árboles cuidados, huertos, plantas en orden)

**Estado 2 (Transición):**
- Espacios abiertos: 15-25% (menos comercio, pero aún actividad)
- Interiores: 25-35% (algunos muebles abandonados, otros aún en uso)
- Espacios verdes: 15-25% (mezcla de cultivo y reclamación salvaje)

**Estado 3 (Arruinado):**
- Espacios abiertos: 5-15% (solo reliquias)
- Interiores: 10-20% (arqueología — solo cosas que sobrevivieron al colapso)
- Espacios verdes: 50-70% (la naturaleza es el único prop activo)

**Jerarquía de props:**

- **Props narrativos:** Objetos que el jugador puede recoger, que comunican historia (una carta, una reliquia de familia, un arma abandonada). Estos son SIEMPRE legibles
- **Props ambientales:** Muebles, vasijas, herramientas. Comunican "gente vivía aquí" o "gente vive aquí"
- **Props decorativos:** Piedras, hojas caídas, plantas ornamentales. Comunican textura

---

### 6.7 Storytelling Ambiental — Lo Que el Espacio Comunica

**Guía para diseñadores:**

Cada screen debe responder una de estas preguntas visualmente, sin texto:

1. "¿Quiénes viven aquí?" — El tipo de arquitectura responde (nobleza, soldados, campesinos, comerciantes)
2. "¿Están prosperar o sufriendo?" — El estado de conservación y densidad de props responde
3. "¿Cuánto tiempo lleva así?" — El grado de reclamación orgánica responde (reciente, antiguo, muy antiguo)
4. "¿Es seguro?" — La forma del espacio y presencia de amenazas visuales responde

**Ejemplos de narrativa ambiental Estado 1:**
- Un mercado con vendedores y cosas siendo compradas = "Este reino está vivo"
- Una casa con puerta cerrada y humo de chimenea = "Alguien vive aquí y se siente seguro"

**Ejemplos de narrativa ambiental Estado 2:**
- Una calle con tiendas cerradas y puertas rotas = "Algo sucedió aquí recientemente"
- Un templo con reparaciones visibles pero incompletas = "Alguien intenta mantener esto, pero está perdiendo"

**Ejemplos de narrativa ambiental Estado 3:**
- Una habitación completamente llena de vides = "Esto fue abandonado hace años"
- Una plaza de mercado vacía, con solo piédestales donde estaban estatuas = "El poder se fue"

---

### 6.8 Transiciones Entre Reinos y Estados

**Transición entre reinos (1-2 screens de ancho):**
- Los colores comienzan a cambiar gradualmente
- La arquitectura comienza a reflejar el nuevo estilo
- La densidad y tipo de prop cambia
- La música ambiental transiciona simultáneamente

**Transición entre estados dentro de un reino:**
- Puede ser más abrupta — un reino puede tener un lado próspero y un lado arruinado
- Ejemplo: Reino 1 tiene una ciudad principal próspera (Estado 1) pero las periferias están en ruinas (Estado 3)
- El jugador entiende que el poder está concentrado en el centro

---

### 6.9 Efectos Ambientales y Luz Dinámica

**Luz dinámica por reino (complementa Sección 2 — Mood & Atmosphere):**

**Estado 1 (Próspero):**
- Luz clara, ángulos que comunican actividad
- Iluminación del mercado (antorchas encendidas, puertas abiertas dejando luz)

**Estado 2 (Transición):**
- Luz mixta — algunas áreas claras, otras oscuras
- Iluminación inconsistente (algunas torches encendidas, otras apagadas)

**Estado 3 (Arruinado):**
- Luz muy baja, filtrada a través de ruinas
- Sin iluminación activa (no hay gente manteniéndola)

**Efecto de proximidad de Edrick en late-game:**
- En reinos más prósperos, el efecto es más sutil — la luz cambia ligeramente
- En reinos arruinados, el efecto es más drástico — incluso en ruinas abandonadas, el mundo reacciona a su corrupción

---

**Tests de validación ambiental:**

1. **Test de lectura de estado:** ¿Puede un jugador entender si un reino es próspero, transicional, o arruinado, solo por la visual?
2. **Test de diversidad:** Toma 5 screens de reinos diferentes. ¿Cada uno se siente diferente y vivido?
3. **Test de densidad de prop:** ¿La densidad de props corresponde al estado del reino?
4. **Test de no-euclidiano (zonas demoniacas):** ¿Siente el jugador desasosiego sin poder nombrar por qué?
5. **Test de humanidad:** ¿El reino se siente como un lugar donde vivía la gente, no solo un cementerio?

---

## 7. Dirección Visual de UI/HUD

*Gobernado por: "Si no tiene peso, no merece luz."*  
*El UI es parte del mundo — no un layer digital sobre él.*

---

### 7.1 Filosofía General del UI

**Regla central:** El UI de Demons Of Dravaryn no existe en una pantalla digital. Existe en el mundo narrativo. El jugador está leyendo un manuscrito (Bestiario), mirando objetos forjados en metal (HUD de combate), viendo escrituras talladas en piedra (etiquetas de ubicación), consultando un mapa de cuero y tinta.

**Dos sistemas visuales separados:**

1. **Sistema de Manuscrito (Bestiario, Diarios, Lore):** Ink-drawn, orgánico, imperfecto. Esto es lo que Edrick ha escrito o descubierto.
2. **Sistema de Forja (HUD de combate, indicadores, slots de habilidad):** Angular, metal, permanente. Esto es lo que lleva Edrick — su equipo, su estado de batalla.

**Lo que NO es UI:**
- Ningún elemento digital moderno (fuentes sans-serif de espacios, bordes redondeados, gradientes suaves)
- Ningún elemento que parezca ser de un videojuego — es todo medieval-diegético
- Ningún glow o bloom utilizado para decoración — la luz es narrativa, no efectista

---

### 7.2 Bestiario — El Diario Personal de Edrick

**Concepto:** El Bestiario no es un menú ni una enciclopedia estática. Es el diario de investigación vivo de Edrick — un manuscrito físico que el jugador lee, que se completa progresivamente conforme Edrick descubre cosas en la historia. Es tanto un registro personal como una guía del juego sobre los detalles que el jugador va aprendiendo.

**Progresión del Bestiario:**
- Al inicio, el Bestiario está vacío — solo páginas en blanco
- Conforme Edrick encuentra/vincula demonios, aparecen nuevas entradas
- Conforme Edrick aprende sobre portadores históricos (a través de NPCs, documentos, observación), las entradas se expanden con marginalia, anotaciones, dibujos mejorados
- El jugador ve la *evolución* de la comprensión de Edrick — las primeras anotaciones sobre un demonio son caóticas; después de aprender más, están mejor organizadas
- Algunas entradas permanecen incompletas o especulativas — Edrick no tiene todas las respuestas

**Apariencia visual:**

- **Fondo:** Parchment oscuro (RGB 90, 75, 60 — ochre oscuro)
- **Textura:** Patrón de tela suave — el papel tiene weave, no es una imagen limpia
- **Iluminación:** Luz de candela desde la izquierda — vignette suave en los bordes, sin luz de ventana
- **Tipografía:** Fuente que simula escritura manuscrita (no Comic Sans — algo más caligráfico, con variación de grosor)
- **Bordes de página:** Ligeramente irregulares, como si fueron cortados a mano

**Estructura de página (evoluciona con descubrimientos):**

*Primer encuentro con un demonio (poco conocimiento):*
- Título: "¿Demonio?" (Edrick está inseguro)
- Ilustración: Borrador rápido, imperfecto
- Marginalia: Observaciones caóticas — "¿Qué fue eso? ¿Fue una ilusión? Tenía ojos [ilegible]"

*Después de vinculación y más encuentros (conocimiento medio):*
- Título: Nombre del demonio (si es conocido), o descripción ("Demonio del fuego")
- Ilustración: Mejorada, más detallada
- Marginalia: Notas más estructuradas — "Aparece en [reino], marca al portador con [efecto visual], parece ser [especulación]"
- Símbolo heráldico: Si hay portador conocido, sello de la casa

*Después de investigación profunda (conocimiento completo):*
- Entrada completamente desarrollada
- Dibujo cuidado
- Notas bien organizadas
- Posible historia del demonio o portador anterior
- Si el demonio es legendario/misterioso: aún puede tener "?" y especulaciones sin resolver

**Secciones del Bestiario:**
- **Demonios descubiertos:** Mostrados en el orden de primer encuentro (refleja el viaje de Edrick, no un orden lógico)
- **Portadores documentados:** Cuando Edrick aprende sobre quién llevó un demonio antes
- **Demonios errantes:** Demonios sin portador conocido — menor información
- **Demonio del gato:** Sello único, misterioso, iridiscente — las anotaciones de Edrick son especialmente confusas aquí ("¿Es real? ¿Cuándo lo conocí?")

**Interacción:** El jugador scrollea el manuscrito (no un menú deslizante). Las páginas se voltean. El sonido es papel, no clics digitales.

**Propósito dual:** El Bestiario es tanto narrativa (muestra la mente y el viaje de Edrick) como gameplay (el jugador consulta aquí para entender enemigos, demonios, y el lore que ha descubierto).

---

### 7.3 Build Management — Menú de Gestión de Demonios

**Concepto:** Existe un menú concreto donde Edrick gestiona su build actual — qué demonios equipa en este momento. Es un espacio mecánico, no narrativo (aunque visualmente sigue la estética medieval).

**Estructura del menú:**

**Sección 1 — Demonios disponibles:**
- Grid de demonios que Edrick ha vinculado
- Cada demonio se muestra como un sello heráldico + icono visual
- Iconos muestran: tipo de demonio (atributo visual), estado (activo/inactivo)

**Sección 2 — Build actual (Loadout):**
- Slots para los demonios actualmente equipados (número de slots = número máximo de demonios simultáneos)
- El jugador arrastra demonios desde "disponibles" a los slots
- Cada slot ocupado muestra: sello del demonio + habilidad que otorga

**Sistema de habilidades ligadas:**
- **CRÍTICO:** Cada demonio proporciona UNA habilidad concreta, no seleccionable
- Cuando Edrick equipa demonio A, automáticamente gana habilidad A
- Cuando equipa demonio A + demonio B + demonio C, tiene acceso a habilidades A, B, y C
- Las habilidades NO son independientes del demonio — son inseparables
- Ejemplo: Demonio de Fuego = Habilidad "Llamarada". No hay opción de elegir otra habilidad para ese demonio.

**Transformaciones visuales:**
- El menú muestra cómo la build actual transforma a Edrick
- Vista previa visual: pequeña silueta de Edrick mostrando cómo se ve con la build actual
- Cuando cambia la build, la silueta cambia en tiempo real — canales de color, disturbios periféricos, etc.

**Estilo visual del menú:**
- Formas de metal (brackets angulares, bordes chamfered) — es un menú funcional, no narrativo
- Fuente simple y legible
- Sellos heráldicos de demonios coloridos contra fondo oscuro
- Claridad máxima — este es el espacio donde el jugador entiende mecánicamente qué equipa

**Interacción:**
- Drag-and-drop o selección+confirmación
- Confirmación visual cuando la build se aplica (la silueta de Edrick cambia, sonido de transformación demoniacal)

---

### 7.4 HUD de Combate — Equipo del Jugador

**Concepto:** El HUD de combate representa objetos físicos que Edrick lleva y usa — no información abstracta.

**Elementos del HUD:**

**Indicador de salud:**
- Representación: No una barra. Más bien, un escudo o placa de pecho que "se daña" visualmente a medida que la salud disminuye
- Estilo: Forjado en metal, pesado, con bordes chamfered
- Comportamiento: Grietas visibles crecen en el escudo a medida que el daño acumula
- Ubicación: Inferior izquierda (donde se lleva el escudo)

**Build actual / Demonios equipados:**
- Representación: Sellos heráldicos de los demonios actualmente equipados, junto con su habilidad correspondiente
- Ubicación: Central superior o lateral
- Comportamiento: Cuando Edrick cambia demonios en combate (si es posible), los sellos se intercambian
- Paleta: Cada sello usa el color dominante del demonio/reino

**Habilidades disponibles (ligadas a demonios):**
- Representación: Ranuras de habilidad (slots) que muestran qué habilidades están disponibles (una por demonio equipado)
- Cada slot muestra: icono de la habilidad, sello del demonio que la proporciona, estado de cooldown
- Estilo: Metal pesado con corner brackets, no redondeados
- Comportamiento: Los slots se oscurecen cuando están en cooldown (todo el slot se oscurece, no una barra de cooldown)
- Animación: Cuando una habilidad está lista, el slot brilla sutilmente (ganando luz)
- Claridad: El jugador ve claramente QUÉ demonio proporciona QUÉ habilidad

**Indicador de enemigos:**
- No un mini-map. En su lugar, una brújula simple: direcciones cardinales donde los enemigos están
- Estilo: Grabado en metal
- Comportamiento: Ping cuando un enemigo entra en rango

**Lo que NO aparece:**
- Valores numéricos de daño (el jugador ve el resultado de sus ataques, no números flotantes)
- Porcentaje de salud (el jugador lee el escudo, no un "74/100 HP")
- Nombres de enemigos (el jugador lee silhueta, no etiquetas)

---

### 7.5 Mapa

**Concepto:** El mapa existe como un objeto físico — un pergamino o mapa de cuero que Edrick consulta. Es un espacio separado de la pantalla de juego.

**Apariencia:**

- **Fondo:** Cuero o pergamino antiguo (color marrón oscuro, RGB 70, 50, 40)
- **Líneas del mapa:** Tinta sepia — topografía simple, no ultra-detallada
- **Leyenda:** Símbolos simples para ciudades, templos, peligros, etc.
- **Anotaciones:** Anotaciones de Edrick en tinta — notas personales sobre cada región

**Contenido del mapa:**

- **Explorado:** Mostrado con detalles (terreno, estructuras, nombres de ubicaciones)
- **No explorado:** Mostrado en blanco o con "?" — Edrick no sabe qué hay
- **Puntos de interés:** Marcados con símbolos (templo, ciudad, demonio avistado, muerte de aliado, etc.)
- **Anotaciones de Edrick:** Marcas personales — una X para "lugar peligroso", una flecha para "dirección a perseguir", notas manuscritas

**Funcionalidad:**
- El jugador puede zoom in/out
- Al seleccionar una ubicación, ve anotaciones de Edrick sobre ella
- El mapa actualiza conforme Edrick explora
- Si el jugador ha encontrado atajos o rutas alternativas, están marcados

**Estilo visual:**
- No es una cuadrícula moderna — es un mapa medieval imperfecto
- Las líneas de costa y montañas son dibujadas a mano (ligeramente irregulares)
- Los caminos están marcados pero no son perfectos
- Las distancias son aproximadas, no a escala

**Transición:**
- Abrir/cerrar mapa es instantáneo o muy rápido (fade, no animación)
- El sonido es papel enrollado/desenrollado

---

### 7.6 Menús Principales — Navegación

**Pausa menu:**
- Fondo: Oscuro pero no negro — color realm actual (si pause en Kingdom 1, fondo ámbar muy oscuro)
- Opciones: Alineadas verticalmente, sin decoración
- Fuente: Simple, legible, no decorativa
- Interacción: Navegación con flecha del teclado, confirmación con enter

**Opciones de inventario/equipo:**
- Iconografía: Símbolos simples — una espada es una espada, un vial es un vial
- Organización: Grid simple, sin drag-and-drop complicado
- Estilo: Líneas de tinta, bordes ligeramente irregulares (no cuadrículas perfectas)

---

### 7.7 Demonio Entries — La Excepción

**Las entradas de demonio rompen intencionalmente la gramática de UI:**

**Bordes:** Pulsean irregularmente — la forma del borde no se cierra limpiamente, como si fuera inestable

**Colores:** El sello heráldico del demonio usa un color que NO existe en la paleta UI normal — iridiscente, incierto, que cambia ligeramente

**Animación:** La tinta no se seca uniformemente. Las líneas ocasionalmente parecen estar escribiéndose de nuevo — un efecto sutil de re-trazado

**Implicación:** El UI intenta contener información sobre algo que no debería existir. La falta de estabilidad visual comunica que los demonios están *fuera de lugar* incluso en el manuscrito de Edrick.

**Demonio del gato específicamente:**
- El sello es plateado-púrpura iridiscente
- El nombre del demonio está en blanco, no en sepia — como si Edrick no estuviera seguro de cómo escribirlo
- Las anotaciones marginales son caóticas — letras más grandes, tachadas, preguntas sin respuesta

---

### 7.8 Tipografía y Legibilidad

**Fuentes a usar:**
- **Bestiario/Manuscrito:** Fuente caligráfica con variación de grosor (simula tinta de pluma)
- **HUD/Metal:** Fuente simple, recta, sin serif — como si fuera grabada, no escrita
- **Mapas:** Fuente geométrica ligera — como tallado en cuero

**Tamaño y contraste:**
- El texto siempre contrasta completamente con el fondo — legibilidad es no negociable
- En el Bestiario: texto sepia sobre ochre oscuro
- En el HUD: Metal gris sobre fondo más oscuro o vice versa
- Sin fuentes muy pequeñas — el jugador debe poder leer sin esfuerzo a distancia

**Jerarquía de tamaño:**
- Títulos: 1.5x grande
- Cuerpo: Normal
- Marginalia/notas: Más pequeño, pero aún legible

---

### 7.9 Navegación y Flujo

**Sin submenús anidados profundos:**
- Máximo 2 niveles de profundidad (Bestiario → Demonio seleccionado)
- El flujo es directo — el jugador accede a lo que necesita rápidamente

**Sin animaciones complicadas:**
- Las transiciones entre pantallas son instantáneas o muy rápidas (0.2 segundos max)
- La apertura del Bestiario es una transición de fade, no una animación de "libro abriéndose"

**Interacción con inputs:**
- Teclado/mouse primario
- Las opciones en menú se naveguen con flechas, confirmación con enter
- El UI responde inmediatamente — sin lag

---

### 7.10 Accesibilidad en UI

**Consideraciones:**

**Daltonismo:**
- Los indicadores del HUD que usan color (cooldown, enemigos cercanos) también usan forma o icono
- El demonio loadout usa forma de sello además de color
- Los slots de habilidad se oscurecen completamente cuando están en cooldown, no solo cambian de color

**Legibilidad:**
- Contraste mínimo WCAG AA (4.5:1 para texto pequeño)
- Sin texto que sea menor a 10-12pt equivalente
- Las opciones de fuente más grande están disponibles en settings

**Sin elementos solo-sonido:**
- Cada feedback sonoro tiene un visual complementario
- El brillo del sello cuando la habilidad está lista es visual Y sonoro

---

### 7.11 Estilo Prohibiciones y Guardrails

**Lo que NUNCA debe aparecer en el UI:**

- Gradientes suaves o "modern" (los gradientes, si existen, son planos)
- Glow/bloom como decoración (solo para indicadores específicos de "habilidad lista")
- Bordes redondeados en elementos principales
- Animaciones innecesarias que ralentizan la interfaz
- Números flotantes de daño o efectos
- Mini-maps tradicionales con jugador en centro (usa mapa consulta + brújula en HUD)
- Barras de progreso tradicionales (comunica a través de forma, grieta, cambio de estado)

**Guardrails:**

- Cada elemento de UI debe responder la pregunta: "¿Por qué esto existe en el mundo?" Si la respuesta es "porque es UI," requiere revisión
- El Bestiario es el único espacio donde el UI puede ser "real" narrativamente como manuscrito — todo lo demás debe sentir como equipo/herramientas
- Prueba: muestra el UI a alguien y pregunta si se siente como un videojuego o como objetos dentro del mundo. Debe ser lo último

---

**Tests de validación de UI:**

1. **Test de inmersión:** ¿El UI se siente como parte del mundo, o como un layer digital?
2. **Test de legibilidad:** ¿Un jugador puede leer todo el texto sin esfuerzo a una distancia normal de pantalla?
3. **Test de Bestiario:** ¿El Bestiario se siente como un manuscrito antiguo que evoluciona, no como un menú estático?
4. **Test de build:** ¿El jugador entiende claramente que cada demonio = una habilidad, no son sistemas independientes?
5. **Test de demonio:** ¿Las entradas de demonio se sienten visualmente inestables y fuera de lugar?
6. **Test de mapa:** ¿El mapa se siente como un objeto consultado, no como un mini-map siempre visible?
7. **Test de daltonismo:** Sin depender solo de color, ¿puede el jugador entender el estado de cooldown y selección de habilidad?

---

## 8. Estándares de Assets

*Gobernado por: "Si no tiene peso, no merece luz."*  
*Cada asset debe cumplir especificaciones técnicas y narrativas.*

---

### 8.1 Filosofía de Producción de Assets

**Regla central:** Los assets comunican a través de limitación intencional. Pixel art es abstracción — cada pixel cuenta. La fidelidad viene de la intención, no del detalle.

**Restricción técnica como dirección artística:** El pixel art impone límites que sirven la visión del juego. No son limitaciones que aceptamos — son herramientas que usamos.

**Engine y plataforma:** Godot 4.6.3, exportación a PC (Steam). Las especificaciones se optimizan para Godot 2D en esta plataforma.

---

### 8.2 Especificaciones de Sprite

**Resolución base para personajes:**
- Edrick: 48x64 pixels en idle (base de tamaño)
- NPCs: 40x56 pixels (ligeramente más pequeños, más cortos)
- Enemigos: 40x56 a 64x80 pixels (escala con amenaza/rol)
- Demonios: Varía según forma, pero típicamente 32x48 a 96x128 pixels

**Escala de mundo:**
- 1 tile = 16x16 pixels
- Los sprites ocupan 1-2 tiles de ancho, 2-5 tiles de alto
- Consistencia: todos los sprites en la misma escena deben obedecer la misma "altura de cámara" — si Edrick tiene los ojos a 48px de altura visual, todos los objetos a distancia equivalente deben tener proporciones consistentes

**Paleta de colores:**
- Máximo 256 colores por sprite (limitación de PNG indexado, buen límite de calidad)
- Colores base del reino + transformaciones demoniacas
- Sin gradientes suaves — transiciones son por pasos de color discretos
- Test: cada sprite debe ser legible en escala de 50% (mitad de tamaño)

**Animaciones:**
- Idle: 2-4 frames, ciclo de 3-5 segundos
- Movimiento: 4-6 frames, duraciones variadas por tipo
- Ataque/habilidad: 6-12 frames, duraciones que comunican peso
- Transición demoniacal: 8-12 frames, transición suave que dura 0.5-1 segundo

**Líneas de referencia para artistas:**
- Línea de ojos: Crítica — debe estar clara incluso en silhueta. Mínimo 2 píxeles de diferenciación
- Centro de masa: Invisibly importante — las animaciones deben mantener el centro de masa coherente para que el movimiento se sienta pesado
- Outline/borde externo: Bordes limpios para personajes (a excepción de demonios que rompen estos bordes)

---

### 8.3 Especificaciones de Ambientes

**Tiles base:**
- Tamaño estándar: 16x16 pixels (pueden ser múltiples tiles para estructuras más grandes)
- Máximo 256 colores por tileset
- Test de legibilidad: 4 tiles de ancho (64x16 pixels) deben ser claramente legibles

**Capas de ambiente (parallax/depth):**
- Parallax 4 (background lejano): Muy bajo detalle, solo siluetas, desaturado 60%+
- Parallax 3 (background): Bajo detalle, elementos grandes, desaturado 50-60%
- Parallax 2 (midground): Detalle medio, la mayoría de la arquitectura, colores reino 60-80% saturación
- Parallax 1 (foreground/gameplay): Alto detalle, plataformas, objetos interactivos, colores reino 80%+ saturación
- Parallax 0 (overlay, si existe): Muy raramente — solo para efectos visuales críticos

**Ratio de píxeles por capa (para renders de 320x180 en 1x, escaleado a 1280x720 en 4x):**
- Parallax 4-3: ~40% de píxeles totales (background)
- Parallax 2: ~35% de píxeles totales (midground)
- Parallax 1: ~25% de píxeles totales (gameplay layer, donde ocurren las cosas)

**Densidad de detalles por estado:**
- Estado 1 (Próspero): Promedio de 15-20 props únicos por screen de 320x180
- Estado 2 (Transición): Promedio de 10-15 props
- Estado 3 (Arruinado): Promedio de 5-10 props (mucha naturaleza reclamando, pocos objetos humanos)

---

### 8.4 Especificaciones de Efectos Visuales

**Efectos de habilidades:**
- Tamaño: Escalable desde 32x32 pixels hasta 256x256 pixels dependiendo del tipo
- Duración: Máximo 1 segundo (24 frames a 60fps)
- Claridad: Debe leer claramente qué tipo de efecto es (fuego, hielo, tiempo, sombra, etc.) en el primer frame
- Animación: 6-12 frames, suave pero con "peso" — no una transición suave, sino una que comunica impacto
- Paleta: Usa colores reino + colores demonios, saturation 100% (esto es lo único en combate que puede ser brillante)

**Efectos ambientales (lluvia, nieve, polvo):**
- Partículas muy pequeñas (2-4 pixels cada una)
- Bajas en saturación (si es nieve, gris-blanco; si es polvo, gris-marrón)
- No deben ocultar el gameplay — son decorativos
- Densidad variable por estado del reino (Estado 1 = menos polvo; Estado 3 = más)

---

### 8.5 Convenciones de Naming y Organización de Archivos

**Estructura de directorios:**
```
assets/
├── sprites/
│   ├── characters/
│   │   ├── edrick/
│   │   │   ├── edrick_idle.png
│   │   │   ├── edrick_walk.png
│   │   │   ├── edrick_attack.png
│   │   │   ├── edrick_bind_demonio.png
│   │   ├── npcs/
│   │   ├── enemies/
│   ├── demonios/
│   │   ├── [demonio_name]/
│   │   │   ├── [demonio_name]_idle.png
│   │   │   ├── [demonio_name]_attack.png
│   ├── environments/
│   │   ├── [kingdom_name]/
│   │   │   ├── [kingdom_name]_tileset.png
│   │   │   ├── [kingdom_name]_background_parallax3.png
│   │   │   ├── [kingdom_name]_background_parallax4.png
│   ├── ui/
│   │   ├── bestiario_elements/
│   │   ├── hud_elements/
│   │   ├── demonio_seals/
├── vfx/
│   ├── abilities/
│   ├── environmental/
```

**Convención de nombres:**
- Minúsculas, guion-separadas: `edrick_idle.png`, no `EdrickIdle.png` o `edrick_Idle.png`
- Descriptivos: `edrick_walk_east.png` mejor que `edrick_anim2.png`
- Versiones: `edrick_idle_v2.png` si hay iteración (permite tracking de cambios)
- Demonio transformaciones: `edrick_idle_demonio_fuego.png` para versiones transformadas

**Formato de archivos:**
- Sprites: PNG con transparencia (8-bit indexed cuando sea posible para reducir tamaño)
- Tilesets: PNG 16x16 tiles en grillas, múltiples tiles por fila
- Documentación de tileset: archivo `.txt` o `.json` listando qué tile es cuál (ej: "tile_0_0 = stone_wall")

---

### 8.6 Importación a Godot — Configuración

**Configuración de importación de sprite:**
- Filtro: Nearest (mantiene nitidez de pixel art)
- Compresión: VRAM (optimiza para rendering en tiempo real)
- No use mipmaps (pixel art no se beneficia)

**Configuración de tileset:**
- Tamaño de tile: 16x16 pixels (debe coincidir con la especificación de tile)
- Physics layers: Definidas por tile (colisión)
- Navigation layers: Definidas para pathfinding si es necesario

**Configuración de AnimatedSprite2D:**
- Framerate: 12-15 fps para idle (movimiento lento, contemplativo)
- Framerate: 15-20 fps para walk (movimiento normal)
- Framerate: 20-30 fps para ataque/habilidad (movimiento rápido, impacto)

---

### 8.7 Exportación e Integración

**Proceso de exportación:**
1. Sprite final en software de pixel art (Aseprite, Pyxel, etc.)
2. Exportar como PNG con transparencia
3. Importar a Godot como sprite resource
4. Configurar en editor (hitboxes, animaciones, etc.)
5. Crear escena con AnimatedSprite2D o StaticSprite2D según necesidad
6. Guardar como .tscn (scene file)

**Control de versión:**
- Los archivos PNG se commitan al repositorio (pequeños, no binarios problemáticos)
- Los archivos .tscn de Godot se commitan
- Los archivos fuente de Aseprite (.ase) NO se commitan (no son necesarios en el build)

---

### 8.8 Resolución y Escala de Pantalla

**Resolución base de juego:** 320x180 pixels (internal)
**Resolución de pantalla (Steam):** 1280x720 pixels (4x scale)
**Soporte futuro posible:** 1920x1080 (6x scale)

**Implicación para artistas:**
- Diseña todos los sprites asumiendo escala 4x
- En un monitor 1920x1200, el juego ocupará la mitad de la pantalla (1280x720)
- Los sprites serán nítidos y legibles a cualquier escala de múltiplo de 4x

---

### 8.9 Tests de Validación de Assets

1. **Test de silhueta en thumbnail:** Exporta sprite a 32x32 pixels. ¿Es aún legible el arquetipo/amenaza?
2. **Test de paleta:** ¿Todos los colores pertenecen a la paleta del reino? ¿La saturación es apropiada?
3. **Test de animación:** ¿Cada frame es un progreso visual claro hacia el siguiente? ¿Sin "frames muertos"?
4. **Test de consistencia de escala:** Coloca varios sprites en la misma escena. ¿Las proporciones tienen sentido relativo?
5. **Test de claridad de línea:** ¿El outline/borde de cada sprite es limpio y legible?
6. **Test de importación a Godot:** ¿El sprite importa sin artefactos? ¿Las animaciones se reproducen suavemente?

---

### 8.10 Guía de Iteración y Feedback

**Ciclo de aprobación:**
1. Artista crea sprite inicial
2. Captura en-motor en Godot
3. Revisión visual: ¿comunica lo que pretende?
4. Revisión técnica: ¿cumple especificaciones de resolución, paleta, etc.?
5. Aprobación o feedback iterativo
6. Commit a repositorio

**Feedback común:**
- "La silueta no es clara" → Aumentar contraste de borde, simplificar
- "El color no pertenece al reino" → Ajustar saturación o matiz
- "La animación se siente flotante" → Añadir más frames, aumentar duración, ajustar arcos de movimiento
- "Está demasiado detallado" → Reducir líneas internas, simplificar, mantener solo lo narrativo

---

## 9. Prohibiciones de Estilo y Referencias Visuales

*Gobernado por: "Si no tiene peso, no merece luz."*  
*La restricción es claridad. Estos son los límites que definen lo que Demons Of Dravaryn ES.*

---

### 9.1 Prohibiciones Visuales Explícitas

**Lo que NUNCA debe aparecer en el juego, sin excepción:**

- **Gradientes suaves y soft shadows** — La luz en este mundo es narrativa, no decorativa. El contraste debe ser directo.
- **Rounded rectangles en el mundo** — Contradicen la gramática de formas angulares. (Exception: UI metal puede tener chamfers, no curves)
- **Efectos de bloom/glow ornamental** — El glow solo existe en momentos narrativos (demonio binding, habilidades en combate). Nunca como decoración.
- **Texturas fotorrealistas o hyper-detalladas** — Pixel art vive en la abstracción. Una piedra es una piedra, no una foto de granito.
- **Color puro neón** — Los colores pertenecen a reinos. Ni nada artificial/digital.
- **Animaciones de 60fps suave** — El movimiento tiene peso. Las transiciones lentas comunican contemplación; las rápidas comunican urgencia. Nunca "suavidad" desconectada.
- **Parallax inconsistente** — Si parallax 2 se mueve a velocidad X, todos los elementos en parallax 2 se mueven a velocidad X. Sin excepciones.
- **Bordes de sprites con anti-aliasing** — Pixel art es píxeles. Los bordes son limpios o rompen intencionalmente (demonios). Sin transiciones suaves.
- **Sombras de color falso** — Las sombras son siempre en escala de grises o colores más oscuros del kingdom accent. Nunca sombras azules bajo rojo.

**Por qué estas prohibiciones existen:**

Cada una de estas cosas ha aparecido en pixel art y "arruina" la legibilidad o la coherencia de los mundos. Son atajos que parecen profesionales pero comunican confusión visual.

---

### 9.2 Direcciones de Estilo a Evitar

**NO queremos que el juego se parezca a:**

| Referencia a Evitar | Por qué | Qué Aprendemos |
|---|---|---|
| Castlevania series | Demasiada exuberancia gótica, decoración excesiva, visual noise compite con gameplay | Mantener claridad sobre decoración |
| Hyper Light Drifter | Brillos/efectos excesivos, demasiado ciencia ficción, color artificial | La luz debe estar ligada al mundo, no ser pura atmósfera |
| Stardew Valley | Demasiado amable, colores pasteles, sin drama visual | Nuestro mundo tiene peso y oscuridad, no bucolicidad |
| Hollow Knight (completamente) | Mundo uniformemente muerto, sin esperanza narrativa, todo decadencia | Necesitamos variedad: algunos reinos prósperos, otros en declive |

---

### 9.3 Referencias Visuales de Inspiración

**Estas referencias definen elementos específicos a adoptar:**

### Referencia 1: Game of Thrones (TV Series, Seasons 1-4)

**Qué tomar:** Arquitectura medieval pesada, escala de poder, sombras largas comunicando hora del día, color en contexto (los Stark = grises fríos, los Lannister = oros cálidos)

**Específicamente:**
- Cómo la arquitectura comunica poder: castillos masivos, muros gruesos, puertas grandes pero pesadas
- Cómo el color comunica familia/región: cada casa tiene paleta cromática
- Cómo la luz comunica estado: interiores oscuros = castillo se está desmoronando, iluminación clara = poder central estable

**Lo que NO tomar:** Realismo excesivo, fidelidad fotográfica, complejidad de trama (nos interesa lo visual, no el argumento)

**Aplicación directa:** Los reinos deben tener "casas" visuales — su propia paleta, su propia arquitectura, su propio peso.

---

### Referencia 2: Hollow Knight (Video Game)

**Qué tomar:** Siluetas claramente legibles a cualquier escala, animación que comunica sin diálogos, mundo que cuenta historia a través de arquitectura abandonada

**Específicamente:**
- Cómo los personajes son siluetas puras legibles: el Knight es un ovaloide con punta arriba, cada enemigo es distinto
- Cómo la animación es minimalista pero comunica MUCHO: un salto comunica esperanza, una caída comunica derrota
- Cómo el ambiente comunica decadencia: tuberías rotas, calles rotas, naturaleza reclamando

**Lo que NO tomar:** El uniformidad total de la decadencia, la ausencia de esperanza, la monotonía cromática (verde/negro en todos lados)

**Aplicación directa:** Cada personaje debe ser reconocible por silueta en thumbnail. Cada animación debe contar un microcuento.

---

### Referencia 3: Medieval Architecture (Real Historical References)

**Qué tomar:** Proporciones de edificios medievales reales, materiales visibles (piedra, madera, metal), deterioro que sigue física real (grietas en piedra, óxido en metal)

**Específicamente:**
- Cómo se construyeron castillos reales: piedra grande en base para fuerza, madera en pisos superiores para economía, techos en ángulo para lluvia
- Cómo se deterioran: piedra se raja en líneas de estrés, madera se pudre desde esquinas, plantas crecen en grietas (no aleatoriamente, en lugares donde el agua se acumula)
- Escala y proporciones que comunican poder: una puerta de 3 hombres de alto no es decoración, es autoridad

**Lo que NO tomar:** Ornamentación excesiva gótica, detalles que nadie puede ver desde distancia, perfección (los castillos reales estaban sucios y parcheados)

**Aplicación directa:** Cada arruina tiene una razón física por qué se desmorona así. Cada detalle arquitectónico comunica algo.

---

### Referencia 4: Berserk Manga (Visual Style Only)

**Qué tomar:** Cómo el blanco y negro pueden comunicar atmósfera tan poderosamente como el color, cómo las sombras pueden ser casi personajes propios, cómo la perspectiva y composición cuentan historias

**Específicamente:**
- Uso de negro no como vacío sino como presencia — las sombras pesan
- Cómo pequeños detalles de personajes comunican carácter (cicatrices, postura, tamaño)
- Cómo el espacio alrededor de un personaje comunica su poder o vulnerabilidad (un personaje rodeado de blanco = aislado; rodeado de negro = amenazado)

**Lo que NO tomar:** La violencia gráfica, el erotismo, la densidad excesiva de líneas (nuestro mundo es pixel art, no manga detailed)

**Aplicación directa:** La composición importa. Dónde colocas un personaje en la pantalla comunica su narrativa.

---

### Referencia 5: Limbo (Video Game)

**Qué tomar:** Siluetas contra luz, minimalismo extremo (solo lo que importa es visible), horror a través de ausencia, no presencia excesiva

**Específicamente:**
- Cómo menos información puede ser más aterrador: no ves todo, imaginas
- Cómo la luz es el personaje principal: define qué es visible, qué es desconocido
- Cómo el espacio vacío es tan importante como el espacio lleno

**Lo que NO tomar:** Puzzle design (ese es mecánica, no visual), la uniformidad de escala de grises (queremos color)

**Aplicación directa:** No todo debe estar visible. La oscuridad y la luz son herramientas narrativas.

---

### Referencia 6: Études de Arquitectura Medieval (Pinterest/Historia del Arte)

**Qué tomar:** Referencias fotográficas de cómo lucen castillos reales hoy (ruinas), proporciones, deterioro natural

**Cómo usarla:** Cuando diseñes un nuevo reino, busca en Google "medieval ruins [kingdom theme]" y mira proporciones, proporción de piedra a vides, cómo crecen las plantas en grietas específicas

---

### 9.4 Referencia Negativa Explícita: Qué NO Hacer

| Referencia Negativa | El Problema | Qué Hacer en Su Lugar |
|---|---|---|
| Stardew Valley estética | Demasiado amable, diminuye drama | Usa colores más oscuros, sombras más profundas, incluso en reinos "prósperos" |
| Retro City Rampage estilo visual | Demasiado ocupado, demasiados detalles pequeños | Simplifica, mantén solo lo que importa narrativamente |
| Celeste / Jump King pixel style | Muy "limpio", colores saturados uniformes | Añade variación: algunos colores brillantes, otros desaturados, creando jerarquía |
| Undertale tileset approach | Tilesets pequeños sin variación dentro del tipo | Nuestros tilesets tienen 3-4 variantes (limpio, deteriorado, muy deteriorado) para comunicar estado |

---

### 9.5 Proceso de Validación Contra Referencias

Cuando un artista completa un asset:

1. **¿Se parece a una referencia inspiradora?** (Game of Thrones, Hollow Knight, arquitectura medieval) → Buena señal
2. **¿Se parece a una referencia a evitar?** (Castlevania exceso, Stardew amabilidad, Hyper Light brillo) → Requiere revisión
3. **¿Comunica claramente a 32x32 pixels?** (Test Hollow Knight) → Sí = bueno
4. **¿El deterioro sigue física real o es aleatorio?** (Test arquitectura medieval) → Debe ser física real
5. **¿El color pertenece al reino?** (Test GoT casas) → Sí = bueno

---

**Test final de validación de todo el Art Bible:**

1. **Test de cohesión visual:** Junta screenshots de todos los 9 reinos. ¿Cada uno se siente como un mundo único pero unificado?
2. **Test de narrativa visual:** Muestra un screenshot de un NPC importante vs. un NPC menor. ¿Es visualmente obvio cuál importa?
3. **Test de demonio:** Muestra un demonio en su reino. ¿Rompe la gramática visual de manera que comunica poder?
4. **Test de late-game:** Muestra a Edrick completamente corrompido en varios reinos. ¿El mundo REACCIONA a su presencia?
5. **Test de manuscrito vivo:** Abre el Bestiario. ¿Se siente como un registro en evolución, no un menú estático?
6. **Test de inmersión:** Alguien que no juega videojuegos mira 5 screenshots aleatorios. ¿Entienden que es un mundo medieval, no un juego digital?

---

*La Art Bible está completa. Este documento es la brújula — cuando haya dudas sobre si algo "se siente correcto," regresa aquí y pregunta: ¿esto sirve la regla central? ¿Si no tiene peso, no merece luz?*
