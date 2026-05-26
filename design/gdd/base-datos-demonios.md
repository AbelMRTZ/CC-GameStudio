# GDD: Base de Datos de Demonios

> **Estado**: Aprobado (post-revisión — 7 bloqueantes resueltos 2026-05-24)
> **Creado**: 2026-05-24
> **Última Actualización**: 2026-05-24 (Bloqueantes resueltos: turnos→segundos, convención signos resistencias, Restricciones Críticas corregidas, triple Dash+Fuego+Hielo, compuertas narrativas Gato, saturación vs corrupción moral separados, coste real Visión)
> **Sistema**: Base de Datos de Demonios
> **Milestone**: MVP — Foundation Layer
> **Depende de**: — (sin dependencias)
> **Dependen de este sistema**: Combate en Tiempo Real, Loadout & Build Management, Motor de Sinergias, Transformación Visual, Vinculación, Bestiario, Restricción por Demonio

---

## 1. Visión General

La Base de Datos de Demonios es el registro centralizado de todos los demonios en Dravaryn. Define **qué demonios existen**, **qué aportan a Edrick** (habilidades activas, habilidades pasivas, modificadores de stats, resistencias, debilidades), **cómo se ven** (transformaciones visuales de Edrick, iconografía), y **cómo interactúan** con otros demonios (sinergias, restricciones). En MVP, la base de datos contiene **5-8 demonios** detallados. Cada demonio es un registro que define su identidad narrativa y mecánica, permitiendo que otros sistemas (Combate, Loadout, Motor de Sinergias, Transformación Visual) accedan a esa información de forma consistente. Los detalles de balance de habilidades (cooldown, daño específico) pertenecen al GDD de Combate; este sistema define la estructura y los efectos de alto nivel.

---

## 2. Fantasy del Jugador

Cuando Edrick está construyendo su build de demonios, el jugador debería sentir: **identidad única** — cada demonio se siente diferente, tiene su propia personalidad mecánica y visual. **Poder elegible** — la decisión de qué demonio equipar importa; no hay "mejor demonio", hay "mejor para esta situación". **Coherencia narrativa** — cada demonio cuenta una historia; el demonio del gato es legendario/misterioso, otros son corruptores, otros son aliados de Edrick. **Sínergia recompensada** — cuando dos demonios trabajan juntos, el resultado se siente intencional, no accidental. **Transformación visible** — cuando equipas un demonio, Edrick se ve diferente, se siente diferente, el juego te dice "acabas de cambiar".

---

## 3. Reglas Detalladas

### 3.1 Estructura de Demonio Estándar

Cada demonio (excepto Gato) que vive en Edrick sigue esta estructura:

```
id: string (único, ej: "fuego", "hielo")
nombre: string
descripcion: string (narrativa)
tipo: enum [COMBATE, NARRATIVO, HÍBRIDO]
ubicacion: "Edrick" (vive en Edrick)

habilidades_activas: []
  - nombre: string
    descripcion: string
    mecanica: string
    cooldown_segundos: float

habilidades_pasivas: []
  - nombre: string
    descripcion: string
    efecto: string

resistencias: dict
  - físico: float (-0.50 a +0.50)
  - fuego: float (-0.50 a +0.50)
  - hielo: float (-0.50 a +0.50)
  - arcano: float (-0.50 a +0.50)
  - corrupción: float (-0.50 a +0.50)

modificadores_stats:
  - velocidad_multiplicador: float (0.5 a 2.0)
  - gravedad_multiplicador: float (0.5 a 1.5)
  - friccion_multiplicador: float (0.3 a 1.5)
  - hp_bonus: integer (0 a +50)
  - daño_bonus: float (-0.25 a +0.50)

transformacion_visual:
  - descripcion: string (cómo cambia Edrick)
  - aura: string (color/efecto visual)
  - emote_icono: string (referencia a icono en bestiario)

efectos_narrativos: []
  - cuando_se_equipa: string (cambios en diálogo, tono)
  - reacciones_npc: dict (cómo reaccionan personajes específicos)

desbloqueos_narrativos: []
  - descripcion: string
  - condicion: string

portador: string (nombre del portador/origen)
ubicacion_obtencion: string (dónde se vincula en el mundo)
historia_obtencion: string (narrativa de cómo se obtiene)
```

### 3.2 Estructura de Demonio Gato

El demonio del gato es legendario y reside EN el gato, no en Edrick. Estructura única:

```
id: "cat"
nombre: "Gato Legendario" (nombre en mundo)
descripcion: string (misterioso, fragmentario)
tipo: NARRATIVO
ubicacion: "Gato" (vive EN el gato, no transferible a Edrick)

rol_narrativo: string (primordial — clave para entender la trama)
es_primer_demonio: true (primer encuentro del jugador)

combate:
  tipo_interaccion: enum [SEMIAUTÓNOMO, CONTROLADO]
  
  habilidades: []
    - nombre: string
      descripcion: string
      autonomia: bool (true = gato actúa solo, false = Edrick controla)
      mecanica: string
  
  combate_efectividad: "VARIABLE" (no sigue escalas estándar)

apariencia:
  pelaje: "blanco puro"
  emblema: "extraño, difuso, imposible de describir completamente"
  ojos_activacion: "rojo sangre cuando se invoca poder"

interacciones_narrativas:
  con_mente:
    nombre: "Telepatía Felina"
    descripcion: "Edrick lee la mente del gato — comunicación telepática profunda"
    desbloquea: "narrativa_gato_mind"
    compuerta_narrativa: "Requiere haber completado el evento 'La Voz del Gato' (Acto 2). En Acto 1, equipo Mente + Gato activo da solo una línea de diálogo vaga. La comunicación telepática real se activa en Acto 2."
  
  con_vision:
    nombre: "Revelación Felina"
    descripcion: "El gato revela secretos del pasado — visiones desde perspectiva felina"
    desbloquea: "narrativa_gato_vision"
    compuerta_narrativa: "Siempre disponible una vez ambos vinculados. Las visiones son FRAGMENTADAS e interpretables en múltiples sentidos — NUNCA revelan directamente que el gato es el hermano de Edrick antes del Acto 3."

restricciones:
  - "NO tiene sinergias mecánicas con otros demonios (vive en gato, no en Edrick)"
  - "SOLO interacciones narrativas con Mente y Visión"
  - "NO puede ser reemplazado — siempre accesible una vez vinculado"
  - "Arcano NO amplifica al Gato"
```

### 3.3 Los Siete Demonios MVP

#### 1. Gato Legendario
- **Tipo**: NARRATIVO
- **Ubicación**: EN el gato (no en Edrick)
- **Rol Narrativo**: Guía misterioso y protector. Es clave para la trama de Edrick — comparte secretos del pasado, revela la verdadera naturaleza de Draeven.
- **Combate**: Semiautónomo + Controlado. El gato actúa por su propia voluntad en algunos momentos, pero Edrick puede "sugerir" acciones.
- **Interacciones**:
  - **Con Mente**: "Telepatía Felina" — Edrick lee los pensamientos del gato, accediendo a recuerdos ancestrales
  - **Con Visión**: "Revelación Felina" — El gato muestra el pasado desde su perspectiva (visiones confusas, fragmentarias)
- **Restricciones**: Sin sinergias mecánicas, narrativas únicamente, sin resistencias de combate asignadas

#### 2. Fuego
- **Tipo**: COMBATE
- **Resistencias**: −0.35 fuego (toma 35% menos daño de fuego), −0.25 hielo (25% menos daño de hielo), débil a agua (daño de entorno: 3 HP/s en contacto — no una resistencia, sino una reacción elemental)
- **Habilidades Pasivas**: Al golpear con Edrick, los enemigos sufren "quemadura" (3 HP/s durante 3.0 segundos)
- **Modificadores**: HP bonus +5, daño bonus +0.15
- **Transformación Visual**: Aura naranja/rojo, sprite de Edrick con tonos ígneos
- **Historia**: Demonio de fuego antiguo, portador de la furia de los ancestros

#### 3. Hielo
- **Tipo**: HÍBRIDO
- **Resistencias**: −0.30 hielo (30% menos daño de hielo), −0.20 físico (20% menos daño físico), débil a fuego (+0.15: toma 15% más daño de fuego)
- **Habilidades Activas**:
  - "Congelación Táctica": Congela el agua (permite traversal), ralentiza críticos enemigos
  - "Escudo de Escarcha": Reduce daño recibido X% por N segundos (cooldown moderado)
- **Modificadores**: Velocidad -0.1 (más lento), HP bonus +10, daño bonus -0.05
- **Transformación Visual**: Aura azul/blanco, sprite con cristales de hielo
- **Historia**: Demonio del invierno, aliado en terrenos hostiles

#### 4. Arcano
- **Tipo**: AMPLIFICADOR (espada de doble filo — ver Sección 4.2)
- **Resistencias**: Neutro (no da resistencias propias)
- **Habilidades Pasivas**: 
  - **"Amplificación Arcana"** (PASIVA GLOBAL): Amplifica ×1.25 los EFECTOS ESPECIALES de otros demonios equipados — tanto fortalezas COMO debilidades. NO afecta el `daño_bonus` base (que se suma aditivamente). Detalle completo en Sección 4.2.
  - **"Atravesar lo Inmaterial"** (EXPLORACIÓN NARRATIVA): Puede pasar a través de paredes/obstáculos mágicos en ciertos puzzles del mundo. Valor exclusivo de exploración, no afecta combate.
- **Modificadores**: Daño bonus +0.20 (aditivo, sin auto-amplificación)
- **Corrupción Pasiva**: **Tier S — +0.005/min en combate** (el más alto del MVP — el coste moral de usar Arcano es significativo)
- **Transformación Visual**: Aura púrpura/dorada, sprite con símbolos runícos flotando
- **Historia**: Demonio de la magia antigua. Su poder amplifica todo lo que tocas — incluyendo tus propias debilidades. Y consume tu alma a un ritmo constante.

#### 5. Visión
- **Tipo**: NARRATIVO
- **Resistencias**: Ninguna
- **Modificadores**: HP bonus **-5** (máximo HP 75 → 70 — "el precio de ver demasiado")
- **Habilidades Pasivas**: 
  - **"Oráculo Fragmentado"**: Muestra el pasado de Dravaryn, revelando secretos ocultos
  - **"Visiones Confusas"**: Edrick recibe visiones sobre la moralidad de sus acciones (ambiguas, confusas — no respuestas claras). Las visiones pueden **interrumpir brevemente** a Edrick cuando se activan en exploración (0.5s sin input aceptado — raro, no en combate)
- **Transformación Visual**: Ojos brillantes, aura mística incolora/transparente
- **Historia**: Demonio de la visión ancestral, no ofrece certezas, solo preguntas. Ver el pasado tiene un coste: Edrick carga con más de lo que puede sostener.

#### 6. Mente
- **Tipo**: HÍBRIDO
- **Resistencias**: −0.20 arcano (20% menos daño arcano), débil a corrupción (+0.15: toma 15% más daño de corrupción)
- **Habilidades Activas**:
  - **"Predecir Movimiento"** (PASIVA DEFENSIVA): Edrick esquiva automáticamente N ataques cada X segundos (cooldown alto, ~5-8 segundos entre esquivas)
  - **"Lectura Mental"**: Edrick lee los pensamientos de NPCs durante diálogos. Si el NPC miente, existe un % de fallo que, paradójicamente, REVELA información falsa útil
- **Modificadores**: Velocidad bonus +0.05, daño bonus +0.10
- **Transformación Visual**: Aura platino/blanca, ojos que brillan con concentración
- **Historia**: Demonio de la mente colectiva, aliado en diplomacia y estrategia

#### 7. Dash (Demonio de la Espada Otorgado)
- **Tipo**: COMBATE
- **Ubicación**: "Edrick" (otorgado al inicio)
- **Habilidades Activas**:
  - **"Dash Attack"**: velocidad, duración, cooldown definidos en **GDD #1 Movimiento y Físicas 2D** §3.7. NO se re-define aquí. Valores canónicos viven únicamente en GDD #1. El multiplicador global de cooldown (GDD #6 §7.6 `cooldown_global_scale`) puede ajustar cooldowns de todas las habilidades simultáneamente.
- **Modificadores**: Nada (es base, no modifica)
- **Transformación Visual**: Sprite base de Edrick con espada/arma prominente
- **Historia**: No demonio, sino poder fundamental de Edrick como guerrero

### 3.4 Cómo Funcionan las Sinergias

**Sinergias Mecánicas Positivas** (se activan automáticamente):
1. **Arcano + Fuego** → "Fuego Amplificado": +25% más daño de quemadura
2. **Arcano + Hielo** → "Hielo Amplificado": Congelación dura 25% más tiempo
3. **Arcano + Mente** → "Mente Amplificada": "Predecir Movimiento" esquiva N+1 ataques
4. **Arcano + Visión** → "Visión Amplificada": Las visiones son más nítidas (narrativa + mecánica)
5. **Arcano + Dash** → "Dash Amplificado": Velocidad del dash +25% (multiplicador sobre valor de GDD #1), cooldown -0.1s (reducción sobre valor canónico de GDD #1)
6. **Mente + Visión** → "Telepatía Ancestral": Combina "Lectura Mental" + "Oráculo Fragmentado" — información sobre el pasado de NPCs
7. **Dash + Fuego** → "Estela Ardiente": El dash deja una estela de llamas que aplica quemadura a enemigos que la tocan
8. **Dash + Hielo** → "Estela Congelada": El dash deja una estela de hielo que congela brevemente a enemigos que la tocan

**Sinergias Mecánicas Negativas** (se activan automáticamente, debilidades amplificadas):
1. **Fuego + Hielo** → "Anulación Térmica": Ambos demonios pierden efectividad (-15% daño ambos)
2. **Fuego + Agua** → "Extinción": Quemadura se apaga, daño continuo cancelado
3. **Hielo + Fuego** → "Fricción Extrema": Movimiento ralentizado (-0.15 velocidad)
4. **Corrupción + Mente** → "Ruido Mental": "Predecir Movimiento" falla más frecuentemente (-3 esquivas)
5. **Arcano + Corrupción** → "Desestabilización": Amplificación se invierte (amplifica debilidades)
6. **Visión + Fuego** → "Visiones Ardientes": Las visiones son traumáticas, confusas (puro efecto narrativo, no mecánico)
7. **Visión + Hielo** → "Visiones Congeladas": El pasado se vuelve inaccesible (Oráculo Fragmentado no revela nada útil)
8. **Mente + Fuego** → "Impulsividad": "Predecir Movimiento" tiene mayor cooldown (-1s duración, +2s cooldown)

**Sinergias Narrativas (Gato)** (solo con Gato, sin otras demonios):
1. **Mente + Gato** → "Telepatía Felina": Edrick percibe vagamente la presencia del gato (Acto 1); comunicación telepática real se activa en **Acto 2** tras el evento "La Voz del Gato". Antes de ese evento, la sinergia existe mecánicamente pero el contenido narrativo es mínimo e interpretable.
2. **Visión + Gato** → "Revelación Felina": El gato muestra visiones del pasado desde su perspectiva — siempre fragmentadas, nunca una respuesta directa. Las visiones en Acto 1-2 son abstractas; solo en Acto 3 se vuelven coherentes. **RESTRICCIÓN CRÍTICA**: el contenido de las visiones nunca debe revelar explícitamente la identidad del gato antes del giro final.
3. **Mente + Visión + Gato** → "Comunión Ancestral": Los tres se sincronizan — requiere ambas sinergias previas activas. Disponible desde que todos estén vinculados, pero la profundidad de la revelación está gateada por el progreso narrativo. **La revelación completa (giro del hermano) solo ocurre en la Comunión Ancestral del Acto 3 final.**

**Restricciones Críticas**:
- Gato NO tiene sinergias mecánicas con NINGÚN demonio (solo sinergias narrativas con Mente y Visión)
- Arcano NO amplifica al Gato (están en capas diferentes de realidad narrativa)
- Hielo y Visión NO tienen sinergias juntas (mecánicas ni narrativas)
- Dash sinergiza con: **Arcano** (amplificación de velocidad/cooldown), **Fuego** (Estela Ardiente), **Hielo** (Estela Congelada). No tiene otras sinergias.
- Solo demonios con ubicacion "Edrick" pueden tener sinergias mecánicas entre sí
- Cuando Dash + Fuego + Hielo están equipados simultáneamente: las estelas se generan, pero la penalización de "Anulación Térmica" (Fuego + Hielo) aplica igualmente — ver Sección 5.11

---

## 4. Fórmulas

### 4.1 Cálculo de Resistencias

Las resistencias se aplican multiplicativamente a través de esta fórmula base (definida en GDD Salud y Daño):

```
daño_final = round(daño_base × (1 + modificador_atacante) × (1 + resistencia_defensiva))
```

Donde:
- `daño_base` = daño antes de modificadores (de habilidad, ítem, etc.)
- `modificador_atacante` = modificadores del demonio atacante (daño_bonus)
- `resistencia_defensiva` = valor de resistencia/debilidad del defensor (rango: -0.50 a +0.50)

**CONVENCIÓN DE SIGNOS — CRÍTICA** (alineada con GDD Salud y Daño):
- **Valor NEGATIVO** = resistencia (reduce daño recibido). Ej: -0.35 → el defensor toma 35% menos daño de ese tipo
- **Valor POSITIVO** = debilidad/vulnerabilidad (aumenta daño recibido). Ej: +0.15 → el defensor toma 15% más daño de ese tipo
- Valor 0 = neutral

Esta convención determina cómo se almacenan los valores en `demons.json`. El campo "resistencias" en el esquema usa valores negativos para resistencias y positivos para debilidades — la etiqueta narrativa ("fortaleza" vs "debilidad") no cambia el signo.

**Ejemplo A — Debilidad**: Fuego golpea a Hielo (Hielo es débil al fuego)
- daño_base = 20
- modificador_atacante (Fuego daño_bonus) = +0.15
- resistencia_defensiva (Hielo débil a Fuego) = **+0.15** (positivo = más daño recibido)
- daño_final = round(20 × 1.15 × 1.15) = round(26.45) = **26 daño**

**Ejemplo B — Resistencia**: Fuego (demonio) recibe ataque de fuego (Fuego es resistente al fuego)
- daño_base = 20
- modificador_atacante = 0
- resistencia_defensiva (Fuego resistente a fuego) = **-0.35** (negativo = menos daño recibido)
- daño_final = round(20 × 1.0 × 0.65) = round(13.0) = **13 daño** (35% menos)

### 4.2 Amplificación Arcana

**IMPORTANTE — DOS MECÁNICAS DISTINTAS** (resolución cross-review B1+W-05, 2026-05-26):

#### A) Daño Base — ADITIVO (alineado con GDD #6 Fórmula 4.1)

El `daño_bonus` de Arcano (+0.20) se suma como cualquier otro demonio al pool de modificadores del atacante. NO se aplica ×1.25 al daño base.

```
mod_atacante = (∑ demon_modifier_i) × synergy_multiplier
```

**Ejemplo**: Fuego (+0.20) + Arcano (+0.20) equipados, sin sinergia activa:
- `mod_atacante = (0.20 + 0.20) × 1.00 = 0.40` (no `0.20 × 1.25 = 0.25`)

#### B) Efectos Especiales — MULTIPLICATIVO ×1.25 (espada de doble filo)

Arcano amplifica ×1.25 los **efectos especiales** de otros demonios — tanto fortalezas COMO debilidades. Esto convierte a Arcano en una espada de doble filo: te hace más poderoso ofensiva/defensivamente PERO también más vulnerable.

```
efecto_amplificado = efecto_base × 1.25
```

**Aplicación a fortalezas (te benefician más)**:
- Resistencias: Arcano + Fuego (Fuego tiene −0.35 resistencia a fuego) = `round(-0.35 × 1.25) = -0.44` (44% menos daño)
- Cooldowns: Arcano + Mente (esquiva 4 ataques cada 5s) → 5 ataques cada 5s
- Duraciones de buff: Arcano + Hielo (congelación 2.0s) → 2.5s

**Aplicación a debilidades (te perjudican más — ESPADA DE DOBLE FILO)**:
- Vulnerabilidades: Arcano + Hielo equipados; Hielo tiene **+0.15 vulnerabilidad a Fuego** → `round(+0.15 × 1.25) = +0.19` (recibe 19% más daño Fuego en vez de 15%)
- Resistencias inversas: Mente tiene débil a corrupción (+0.15) → Arcano + Mente = +0.19 vulnerabilidad
- Cooldowns largos: Si una sinergia negativa aumenta cooldown (ej: Mente + Fuego "Impulsividad" +2s cooldown), Arcano lo amplifica a +2.5s

#### C) Por qué Arcano sigue siendo elegible (no dominante)

Combinando A+B y la habilidad narrativa "Atravesar lo Inmaterial" (línea 166), Arcano deja de ser estrictamente mejor:
- Te hace **más poderoso** si tu loadout tiene fortalezas claras
- Te hace **más frágil** si tu loadout tiene debilidades elementales
- Tiene **valor exclusivo de exploración** (atravesar paredes mágicas en puzzles)
- Genera **alta corrupción pasiva** (Tier S, ver Sección 4.3) — coste moral significativo

**No es acumulativo**: Si Arcano está equipado con múltiples demonios, CADA UNO se amplifica independientemente, pero Arcano mismo no se aplica dos veces. Y Arcano NO se amplifica a sí mismo (su propio `daño_bonus` queda en +0.20).

### 4.3 Corrupción Pasiva por Demonio Equipado

**Decisión cross-review B2 (2026-05-26)**: Resuelve la desconexión entre Pilar 5 (transformación moral) y el loop de combate. Cada demonio equipado contribuye corrupción pasiva acumulativa proporcional a su poder relativo en el meta.

**Tabla de Tiers de Corrupción Pasiva** (por minuto en combate activo):

| Demonio | Tier | Corrupción/min en combate | Justificación |
|---------|------|---------------------------|---------------|
| Arcano  | S    | +0.005 | Amplifica todo — máximo poder, máximo coste moral |
| Fuego   | A    | +0.003 | Daño constante + quemadura — violencia primitiva |
| Dash    | A    | +0.003 | Mobilidad + I-frames — poder ofensivo evasivo |
| Hielo   | B    | +0.002 | Control de tiempo — manipulación moderada |
| Mente   | B    | +0.002 | Lectura/predicción — violación mental sutil |
| Visión  | C    | +0.001 | Solo ver — peso moral menor (ya cuesta HP) |
| Gato    | —    | 0      | No es demonio mecánico — vínculo simbiótico |

**Cálculo total por minuto**:
```
corruption_passive_per_minute = ∑ tier_value_i
```
donde `i` itera sobre cada demonio en `equipped_demons`.

**Ejemplo**: Loadout Arcano + Fuego + Dash equipado, 30 minutos de combate activo:
- Corrupción/min: 0.005 + 0.003 + 0.003 = 0.011
- Ganancia total: 0.011 × 30 = **+0.33 corrupción** (sube de 0.20 a 0.53, cruzando umbral Manchado → Comprometido)

**Aplicación**:
- Combate emite señal `corruption_passive_tick(amount_per_minute)` cada **60 segundos en estado de combate activo** (cualquier estado que no sea IDLE en exploración tranquila)
- Estado del Mundo escucha y aplica delta a `corruption_level`
- El decay (Sección 4.X de GDD #4) permite recuperar corrupción si bajas a un loadout débil o desequipas, pero nunca por debajo del `corruption_floor` permanente

**Por qué este modelo cumple Pilar 5**: El jugador SIENTE el coste de su poder. Cada minuto con Arcano es +0.005 más cerca de Caído. Cada decisión de loadout es una decisión moral implícita.

### 4.4 Cálculo de Resistencia Final (con Arcano)

Cuando se calcula `resistencia_defensiva` para Edrick frente a un ataque, se aplica el siguiente proceso:

```
resistencia_base = ∑ resistencias_demonios_equipados[tipo_daño]
si Arcano equipado:
    resistencia_final = resistencia_base × 1.25  # amplifica fortalezas Y debilidades
sino:
    resistencia_final = resistencia_base
resistencia_final = clamp(resistencia_final, -0.5, +0.5)  # cap obligatorio
```

**Nota**: El clamp final asegura que la amplificación de Arcano nunca rompa los límites del rango de resistencia [-0.5, +0.5] definidos en GDD #2.

**No es acumulativo**: Si Arcano está equipado con múltiples demonios, CADA UNO se amplifica independientemente, pero Arcano mismo no se aplica dos veces.

### 4.3 Debilidades de Agua

Fuego tiene debilidad única a agua:

```
daño_agua_continuo = 3 HP/segundo (mientras Edrick esté en agua)
duracion = mientras permanezca en agua
max_acumulacion = sin límite (el daño continúa mientras esté en agua)
```

Esta es una **debilidad de ubicación**, no de tipo de demonio. Se aplica automáticamente en cualquier tile de agua si Fuego está equipado.

**Interrupción**: El daño cesa cuando Edrick sale del agua.

### 4.4 Modificadores de Movimiento por Demonio

Los modificadores se aplican MULTIPLICATIVAMENTE a los valores base de Movimiento y Física 2D:

```
velocidad_final = velocidad_base × velocidad_multiplicador_demonio
gravedad_final = gravedad_base × gravedad_multiplicador_demonio
friccion_final = friccion_base × friccion_multiplicador_demonio
```

Valores base (de GDD Movimiento y Física 2D):
- velocidad_base = 250 px/s
- gravedad_base = [valor del motor — veremos en implementación]
- friccion_base = 800 px/s² desaceleración

**Ejemplo**: Hielo reduce velocidad (~0.9 multiplicador)
- velocidad_final = 250 × 0.9 = 225 px/s (Edrick se mueve más lento)

**Restricción**: Los modificadores NUNCA pueden caer por debajo del 0.3× mínimo o superior al 2.0× máximo en valores base (establecido para prevenir valores extremos).

### 4.5 Bonificación de HP y Daño

Bonificadores directos que no son multiplicativos:

```
hp_total_combate = hp_base + hp_bonus_demonio + hp_bonus_items
daño_ataque = daño_base_arma × (1 + daño_bonus_demonio)
```

**Ejemplo**: Edrick con HP base 75 + Fuego (bonus +5) = 80 HP
**Ejemplo**: Edrick ataca con 20 daño + Arcano (bonus +0.20) = 24 daño

**Nota**: Los bonificadores de HP se calculan UNA VEZ al equipar el demonio, no dinámicamente (la salud actual no cambia, el máximo sí).

### 4.6 Sinergias Mecánicas — Cálculo de Amplificación/Reducción

#### Sinergias Positivas

**Arcano + Fuego:**
```
quemadura_tasa_base = 3 HP/s
quemadura_tasa_con_arcano = round(3 × 1.25) = 3.75 HP/s (≈ 4 HP/s)
duracion_quemadura_base = 3.0 segundos
duracion_quemadura_con_arcano = round(3.0 × 1.25) = 3.75s (≈ 3.8s)
```

**Dash + Fuego (Estela Ardiente):**
```
estela_aparece = cuando se usa dash attack
estela_duracion = 0.8 segundos (después del dash)
quemadura_tasa_estela = 2 HP/s (aplicada a enemigos que toquen la estela)
duracion_quemadura_estela = 2.0 segundos (una vez aplicada al enemigo)
area_estela = línea de 120 píxeles en dirección del dash
```

**Dash + Hielo (Estela Congelada):**
```
estela_aparece = cuando se usa dash attack
estela_duracion = 0.8 segundos (después del dash)
congelacion_estela = ralentiza 40% + posible congelación crítica
area_estela = línea de 120 píxeles en dirección del dash
duracion_efecto = 1.5 segundos ralentización (congelación total es rara)
```

#### Sinergias Negativas (ejemplo: Fuego + Hielo)

```
efectividad_fuego = 100% (base)
efectividad_fuego_con_hielo = 100% - 15% = 85%
efectividad_hielo = 100% (base)
efectividad_hielo_con_fuego = 100% - 15% = 85%
```

Ambos demonios sufren la penalización POR IGUAL (no es unidireccional).

#### Sinergias Narrativas (Mente + Visión)

No tienen fórmula matemática. Se evalúan narrativamente:
- Desbloquean diálogos específicos
- Revelan información sobre NPCs y el mundo
- Afectan reacciones de personajes
- No tienen componente mecánico de combate

---

## 5. Casos Extremos

### 5.1 Cambio de Demonio Durante Combate

**Escenario**: Edrick equipa Fuego, luego durante el combate cambia a Hielo (usa Loadout rápidamente).

**Qué sucede**:
1. El demonio anterior (Fuego) se desactiva inmediatamente
2. Sus modificadores cesan (velocidad vuelve a 250 px/s, resistencias se pierden)
3. La corrupción del demonio anterior se "memoria" en Estado del Mundo (narrativa) pero no afecta nuevo demonio
4. El nuevo demonio (Hielo) se activa, aplicando sus modificadores y resistencias
5. Cualquier efecto continuo del demonio anterior (ej: quemadura aplicada a enemigos) persiste (fue aplicado, el efecto es independiente)
6. Las sinergias se recalculan basadas en el nuevo demonio

**Restricción**: El cambio es instantáneo (sin transición). La UI muestra claramente qué demonio está activo.

### 5.2 Equipo de Demonio Cuando HP es Bajo

**Escenario**: Edrick tiene 5 HP (de 75) y equipa un demonio con +10 HP bonus.

**Qué sucede**:
- El máximo HP aumenta a 85, pero la salud ACTUAL permanece en 5 HP
- Edrick no se cura automáticamente al equipar demonio
- El jugador debe usar item de curación o descansar para recuperar HP

**Restricción**: Equipar un demonio NUNCA cura, solo aumenta el máximo.

### 5.3 Desactivar Demonio / Quedar sin Demonio Equipado

**Escenario**: El jugador abre Loadout y desactiva todos los demonios (solo Dash permanece).

**Qué sucede**:
1. Edrick pierde todos los bonificadores y resistencias (vuelve a stats base)
2. HP máximo vuelve a 75 (base)
3. Si está por encima de 75, la salud se reduce a 75 automáticamente
4. La velocidad vuelve a 250 px/s
5. Arcano NO amplifica nada (sin otros demonios)
6. Gato permanece siempre disponible (no es equipo, está en el gato)

**Restricción**: El juego SIEMPRE requiere Dash (demonio base otorgado), pero otros demonios son opcionales.

### 5.4 Gato en Combate con Múltiples Demonios en Edrick

**Escenario**: Edrick equipa Mente + Fuego, y el Gato está "equipado" en el gato (siempre).

**Qué sucede**:
1. Las sinergias de Mente + Fuego se aplican normalmente en Edrick
2. El Gato actúa INDEPENDIENTEMENTE (semiautónomo)
3. Las acciones del Gato NO sinergizaban con demonios de Edrick (no viven en el mismo "cuerpo")
4. PERO, las sinergias NARRATIVAS (Mente + Gato, Visión + Gato) se desbloquean/activan si ambos están "equipados"
5. Arcano NO amplifica al Gato (diferente plano de realidad)

**Restricción**: El Gato es una entidad mecánica separada, no susceptible a reglas de sinergias estándar.

### 5.5 Arcano + Arcano (Equipar Arcano Dos Veces)

**Escenario**: El jugador intenta equipar "Arcano" dos veces en Loadout (si la UI lo permite).

**Qué sucede**:
- NO permitido por diseño. Cada demonio se puede equipar UNA SOLA VEZ por Loadout.
- Si el sistema recibe intención de duplicar, se ignora (error silencioso) o se rechaza con mensaja clara.

**Restricción**: Cada demonio aparece COMO MÁXIMO UNA VEZ en el Loadout activo.

### 5.6 Daño a Edrick Cuando Está en Agua con Fuego

**Escenario**: Edrick con Fuego está en una charca de agua (tile de agua). Recibe daño del ambiente.

**Qué sucede**:
1. Daño de agua (3 HP/segundo) se aplica debido a debilidad Fuego
2. Edrick recibe daño simultáneamente de:
   - Fuego's debilidad a agua (3 HP/s)
   - Posible daño de enemigo (si hay uno)
3. Los daños se acumulan normalmente (no hay cap)
4. Si ambos aplican en el mismo frame, el UI debe mostrar ambas fuentes claramente

**Restricción**: Estar en agua con Fuego es una posición defensiva problemática (diseño intencional).

### 5.7 Lectura Mental (Mente) cuando NPC Miente

**Escenario**: Edrick equipa Mente y lee pensamientos de un NPC que está mintiendo sobre una quest.

**Qué sucede**:
1. "Lectura Mental" inicialmente tiene X% de éxito (ej: 80%)
2. Si falla (20%):
   - En lugar de "no saber", Edrick recibe información FALSA del demonio Mente
   - Esta información falsa es útil (contradice la mentira del NPC)
   - El jugador no sabe si es verdad o mentira (ambigüedad intencional)
3. Si tiene éxito (80%): Edrick lee correctamente los pensamientos del NPC

**Restricción**: Las fallas de "Lectura Mental" son feature, no bug. Crean ambigüedad narrativa.

### 5.8 Saturación Demoníaca vs Corrupción Moral — Dos Sistemas Separados

Este caso extremo distingue explícitamente entre dos mecanismos que coexisten pero son **independientes**:

---

**SISTEMA A: Saturación Demoníaca (por demonio, reversible)**

**Escenario**: Edrick equipa Fuego durante mucho tiempo. Sus efectos visuales se intensifican con el uso.

**Qué sucede**:
1. La saturación es **cosmética y reversible** — refleja cuánto tiempo Edrick ha cargado ese demonio en esta sesión/run
2. Afecta únicamente: aura visual (más intensa), partículas del sprite (más abundantes), tono de voz de Edrick con ese demonio
3. Si Edrick **desactiva** el demonio, la saturación se "congela" (memorizada en Estado del Mundo, no se pierde)
4. Si lo re-equipa, retoma donde la dejó
5. La saturación NO afecta estadísticas de combate (no daño, no resistencias, no cooldowns)
6. La saturación NO es un arco moral — es feedback visual de "cuánto usas este demonio"

**Restricción**: Saturación es por demonio, no global. Cada demonio tiene su propio contador de saturación.

---

**SISTEMA B: Corrupción Moral de Edrick (global, permanente — Pilar 5)**

**Escenario**: Edrick realiza acciones moralmente oscuras (ejecutar a rendidos, traicionar NPCs, usar poderes prohibidos). O acumula muchos demonios de alto poder durante mucho tiempo.

**Qué sucede**:
1. La corrupción moral es un **arco narrativo permanente** — NO se puede deshacer por cambio de loadout
2. La corrupción moral **persiste independientemente de qué demonio esté equipado** — es del personaje, no del demonio
3. Afecta: diálogos de Edrick (permanentemente más oscuros al progresar), reacciones de NPCs (percepción de amenaza, miedo), acceso a ciertas rutas narrativas
4. La Transformación Visual de Edrick (GDD #14) lee el nivel de corrupción moral global para determinar el aspecto base de Edrick, sobre el que se superpone la saturación del demonio equipado
5. **No es numérica en MVP**: solo tiene 3 niveles narrativos (Íntegro / En transición / Corrompido). Los valores exactos pertenecen al GDD de Progresión Narrativa y Seguimiento Moral.

**Restricción**: Corrupción Moral no es una mecánica de este GDD — se define en GDD Seguimiento Moral (GDD #22). Este sistema solo debe reconocerla como dato del Estado del Mundo al que la Transformación Visual puede acceder.

---

**Interacción entre ambos sistemas**: La saturación demoníaca AMPLIFICA la percepción visual de la corrupción moral. Un Edrick en nivel "Corrompido" con saturación alta de Fuego parece más amenazante que uno con saturación baja. Son capas visuales independientes que se combinan — no un mismo número.

### 5.9 Desbloqueos Narrativos sin Cumplir Condición

**Escenario**: El GDD define que "Visión + Fuego → narrativa confusa", pero el jugador nunca equipa ambos juntos.

**Qué sucede**:
- El desbloqueo NO se activa
- El contenido narrativo sigue oculto
- No hay penalización; es una rama opcional de la narrativa
- Si el jugador lo equipa después, el contenido se desbloquea normalmente

**Restricción**: Los desbloqueos narrativos son acumulativos, no regresivos (una vez desbloqueado, permanece).

### 5.11 Triple Sinergia: Dash + Fuego + Hielo Equipados Simultáneamente

**Escenario**: El jugador equipa Dash (siempre activo) + Fuego + Hielo al mismo tiempo. Dash tiene sinergias positivas con ambos, pero Fuego + Hielo tienen sinergia negativa entre sí.

**Qué sucede**:
1. **Estela Ardiente** (Dash + Fuego) se genera con cada dash — funciona normalmente
2. **Estela Congelada** (Dash + Hielo) se genera con cada dash — funciona normalmente
3. Las dos estelas coexisten en el mismo espacio (la estela es una sola: visualmente "bicolor" o se muestran en capas)
4. **"Anulación Térmica"** (Fuego + Hielo negativa) aplica igualmente: ambos demonios trabajan al 85% de efectividad
   - Quemadura de Estela Ardiente: 2 HP/s → 1.7 HP/s (85%)
   - Ralentización de Estela Congelada: 40% → 34% (85%)
5. El jugador recibe DOS efectos de estela a coste de penalización: más versátil, pero ninguno al máximo

**Regla de precedencia**: Las sinergias positivas de Dash (Estela Ardiente, Estela Congelada) se calculan independientemente. La penalización de Fuego + Hielo se aplica SOBRE los valores resultantes de cada estela.

**Restricción**: Nunca se "cancelan" las estelas entre sí — siempre se generan ambas. El jugador experimenta la tensión de querer dos efectos con la penalización de compatibilidad entre los demonios base.

### 5.12 Gato Desaparece de la Realidad

**Escenario**: En la trama, el Gato desaparece narrativamente (plot point). ¿Sigue siendo funcional?

**Qué sucede**:
- Mientras el Gato está "desaparecido" (estado del mundo), NO está disponible en Loadout
- Edrick no puede acceder a interacciones (Mente + Gato, Visión + Gato) mientras esté desaparecido
- Las sinergias narrativas que requerían Gato se bloquean
- Cuando el Gato "regresa" en la trama, vuelve a estar disponible

**Restricción**: El estado del Gato en la trama controla su disponibilidad mecánica (integración narrativa-mecánica).

---

## 6. Dependencias

### 6.1 Dependencias Entrantes (qué depende de este sistema)

Este sistema es un **Foundation-layer system** — es el cuello de botella para 7+ sistemas:

1. **Combate en Tiempo Real** (GDD #6)
   - Depende de: Estructura de demonios, habilidades activas/pasivas, modificadores de daño, cooldowns
   - Punto de integración: `demonio_equipado.habilidades_activas` se mapean a botones de combate
   - Validación: Cooldowns se manejan en Combate, pero definiciones se toman de aquí

2. **Loadout & Build Management** (GDD #10)
   - Depende de: Estructura de demonio, restricciones de equipo, disponibilidad
   - Punto de integración: UI de Loadout carga `demonios_disponibles` de esta BD
   - Validación: El sistema de Loadout debe respetar restricciones (ej: Gato NO se desactiva)

3. **Motor de Sinergias** (GDD #11)
   - Depende de: Definición completa de sinergias, tablas de amplificación/reducción
   - Punto de integración: Al cambiar demonio en Loadout, Motor recalcula sinergias aplicables
   - Validación: Las fórmulas de sinergias aquí definen comportamiento del Motor

4. **Transformación Visual de Edrick** (GDD #14)
   - Depende de: `demonio_equipado.transformacion_visual` (aura, emote, sprite)
   - Punto de integración: Al equipar demonio, sistema visual actualiza sprite/aura de Edrick
   - Validación: El demonio define CÓMO se ve la transformación, no el cómo técnico

5. **Vinculación de Demonios** (GDD #13)
   - Depende de: Historia de obtencion, condiciones de desbloqueo, portador
   - Punto de integración: Sistema de Vinculación lee estas propiedades para narrar momentos de binding
   - Validación: Cada demonio debe tener narrativa clara de cómo se obtiene

6. **Bestiario** (GDD #19)
   - Depende de: Descripción, historia, transformación visual, efectos narrativos
   - Punto de integración: Bestiario muestra todos los demonios desbloqueados + información completa
   - Validación: Bestiario es librería visual de la BD — requiere contenido descriptivo completo

7. **Restricción por Demonio** (GDD #23, Vertical Slice)
   - Depende de: Ubicación de obtención, restricciones narrativas por demonio
   - Punto de integración: Ciertos demonios pueden estar disponibles solo en ciertas áreas/condiciones
   - Validación: La BD define DÓNDE aparece cada demonio; restricción define CUÁNDO acceder

8. **Salud y Daño** (GDD #2)
   - Depende de: Resistencias y debilidades de demonio
   - Punto de integración: Al calcular daño, fórmula usa `demonio_equipado.resistencias`
   - Validación: Los valores de resistencia (-0.50 a +0.50) se respetan en cálculo de daño

### 6.2 Dependencias Salientes (qué este sistema necesita)

1. **Movimiento y Física 2D** (GDD #1)
   - Requerido para: Modificadores de velocidad/gravedad/fricción (los demonios modifican valores base definidos en Movimiento)
   - Integración: Los multiplicadores de demonios se aplican a valores base de Movimiento

2. **Salud y Daño** (GDD #2)
   - Requerido para: Cálculo de resistencias, debilidades, modificadores de daño
   - Integración: Las fórmulas de resistencia se definen allá, pero aplicación es aquí

3. **Estado del Mundo** (GDD #4)
   - Requerido para: Tracking de demonios desbloqueados, disponibilidad narrativa, corrupción acumulada
   - Integración: Al vincular demonio, Estado del Mundo lo marca como disponible. Si demonio "desaparece" en trama, Estado del Mundo lo desactiva

### 6.3 Bidireccionalidad

- **Combate → Base de Datos**: Combate lee habilidades de demonio. Base de Datos NO lee de Combate (unidireccional).
- **Loadout ↔ Base de Datos**: Bidireccional. Loadout lee demonios; Base de Datos no "sabe" cuál está equipado (lo sabe Estado del Mundo).
- **Motor de Sinergias ↔ Base de Datos**: Bidireccional. Motor aplica sinergias definidas aquí; Base de Datos no ejecuta cálculos.
- **Transformación Visual → Base de Datos**: Transformación visual lee definiciones de `transformacion_visual`. Base de Datos no sabe técnica renderización.

### 6.4 Restricciones de Dependencia

- Base de Datos NO debe conocer detalles de implementación técnica de otros sistemas (ej: NO especificar particle emitters exactas)
- Base de Datos es READ-ONLY para gameplay (otros sistemas leen; no escriben excepto Estado del Mundo que marca disponibilidad)
- Base de Datos define ESTRUCTURA y DATOS, no ALGORITMOS (algoritmos de sinergias/daño/física residen en sus respectivos GDDs)

---

## 7. Parámetros de Ajuste

**NOTA CRÍTICA**: Los siguientes valores son **preliminares** y serán ajustados durante balance pass post-MVP. No son valores finales. Durante implementación, TODOS estos deben ser extrínsecos (en archivos de configuración), no hardcoded.

### 7.1 Resistencias y Debilidades

**Convención de signos** (ver Sección 4.1): valor en `resistencias{}` del JSON — negativo = reduce daño recibido, positivo = amplifica daño recibido.

| Demonio | Resistencia 1 | Resistencia 2 | Debilidad | Rango Seguro |
|---------|---------------|---------------|-----------|--------------|
| Fuego | **−0.35** fuego (35% menos daño fuego) | **−0.25** hielo | Agua: 3 HP/s daño continuo (no resistencia; es efecto de entorno) | −0.50 a 0.0 |
| Hielo | **−0.30** hielo (30% menos daño hielo) | **−0.20** físico | **+0.15** fuego (15% más daño fuego) | −0.50 a +0.50 |
| Arcano | Neutral (0) | Neutral (0) | Neutral (0) | N/A |
| Mente | **−0.20** arcano (20% menos daño arcano) | — | **+0.15** corrupción (15% más daño corrupción) | −0.50 a +0.50 |
| Visión | Neutral (0) | Neutral (0) | Neutral (0) | N/A |
| Dash | Neutral (0) | Neutral (0) | Neutral (0) | N/A |
| Gato | (no aplica) | (no aplica) | (no aplica) | (narrativo) |

**Parámetro de Ajuste**: Cada valor puede variar ±0.10 para balance:
- Fuego: −0.25 a −0.45 fuego, −0.15 a −0.35 hielo
- Hielo: −0.20 a −0.40 hielo, −0.10 a −0.30 físico, +0.05 a +0.25 fuego
- Mente: −0.10 a −0.30 arcano, +0.05 a +0.25 corrupción

### 7.2 Multiplicadores de Movimiento

| Demonio | Velocidad | Gravedad | Fricción | Efecto Narrativo |
|---------|-----------|----------|----------|-----------------|
| Fuego | 1.0× | 1.0× | 1.0× | Mantiene velocidad base (ágil) |
| Hielo | 0.90× | 1.0× | 1.0× | Más lento, más pesado |
| Arcano | 1.0× | 1.0× | 1.0× | No modifica movimiento (solo amplificación) |
| Mente | 1.05× | 1.0× | 1.0× | Ligeramente más ágil (mental, anticipatorio) |
| Visión | 1.0× | 1.0× | 1.0× | No modifica movimiento |
| Dash | 1.0× | 1.0× | 1.0× | Base |
| Gato | (no aplica) | (no aplica) | (no aplica) | (independiente del gato) |

**Rango Seguro**: 0.3× a 2.0× (nunca menor/mayor para evitar extremos)

**Parámetro de Ajuste**: ±0.15× para cada multiplicador:
- Hielo: 0.75× a 1.05× velocidad
- Mente: 0.90× a 1.20× velocidad

### 7.3 Bonificadores de HP y Daño

| Demonio | HP Bonus | Daño Bonus | Justificación |
|---------|----------|-----------|---------------|
| Fuego | +5 HP | +0.15 (15%) | Atacante agresivo |
| Hielo | +10 HP | -0.05 (-5%) | Defensivo, tankea |
| Arcano | 0 HP | +0.20 (20%) | Amplificador puro |
| Mente | 0 HP | +0.10 (10%) | Equilibrado |
| Visión | **−5 HP** | 0 (0%) | El precio de ver demasiado (HP 75 → 70) |
| Dash | 0 HP | 0 (0%) | Base (demonio otorgado) |
| Gato | (no aplica) | (no aplica) | (narrativo independiente) |

**Rango Seguro**: HP bonus −10 a +20 (el negativo de Visión es excepción narrativa justificada), Daño bonus -0.25 a +0.50

**Parámetro de Ajuste**: ±5 HP, ±0.05 daño:
- Fuego: 0-10 HP, +0.10 a +0.20 daño
- Arcano: +0.15 a +0.25 daño

### 7.4 Habilidades Activas — Cooldowns

| Demonio | Habilidad | Cooldown Base | Rango Seguro |
|---------|-----------|---------------|--------------|
| Fuego | Quemadura (pasiva, no aplica) | — | — |
| Hielo | Congelación Táctica | 4.0 segundos | 2.0-6.0s |
| Hielo | Escudo de Escarcha | 6.0 segundos | 4.0-8.0s |
| Mente | Predecir Movimiento | 5.0 segundos entre esquivas | 3.0-7.0s |
| Mente | Lectura Mental | 2.0 segundos | 1.0-4.0s |
| Dash | Dash Attack | 0.6 segundos | 0.3-1.0s |

**Parámetro de Ajuste**: ±1.0 segundo para la mayoría:
- Hielo congelación: 3.0-5.0s
- Mente predicción: 4.0-6.0s

### 7.5 Amplificación Arcana

| Concepto | Valor Base | Rango Seguro |
|----------|-----------|--------------|
| Multiplicador de amplificación | 1.25× (25%) | 1.15× a 1.35× (+15% a +35%) |
| Cooldown reduction (Mente) | -0.1s | -0.05s a -0.15s |
| Duración extension (congelación Hielo) | +0.75 segundos | +0.25s a +1.0s |

### 7.6 Estelas de Dash (Dash + Elemental)

| Sinergia | Parámetro | Valor Base | Rango Seguro |
|----------|-----------|-----------|--------------|
| Dash + Fuego | Duración estela | 0.8 segundos | 0.5-1.0s |
| Dash + Fuego | Tasa quemadura en estela | 2 HP/s | 1-3 HP/s |
| Dash + Fuego | Rango estela | 120 píxeles | 80-150 px |
| Dash + Hielo | Duración estela | 0.8 segundos | 0.5-1.0s |
| Dash + Hielo | Ralentización | 40% | 30-50% |
| Dash + Hielo | Duración ralentización | 1.5 segundos | 1.0-2.0s |
| Dash + Hielo | Rango estela | 120 píxeles | 80-150 px |

**Parámetro de Ajuste**: Las estelas pueden ser más/menos duraderas (0.5-1.2s) según gameplay feel. El rango determina cuán "grande" es el efecto visual/mecánico de la estela.

**Parámetro de Ajuste**: Si Arcano es muy fuerte, reducir a 1.20× (+20%). Si muy débil, aumentar a 1.30× (+30%).

### 7.7 Sinergias Negativas — Penalizaciones

| Sinergias | Penalización | Rango Seguro |
|-----------|-------------|--------------|
| Fuego + Hielo | -15% efectividad ambos | -10% a -25% |
| Fuego + Hielo fricción | -0.15 velocidad | -0.10 a -0.20 |
| Corrupción + Mente | -3 esquivas | -2 a -5 |
| Arcano + Corrupción | Inversión amplificación (amplifica debilidades) | — (binario) |

**Parámetro de Ajuste**: Las penalizaciones pueden variar, pero DEBEN hacer la sinergia obviamente negativa (no vale la pena):
- Si -15% es muy suave, aumentar a -25%
- Si es demasiado punitivo, bajar a -10%

### 7.8 Configuración de Archivo de Datos

Todos estos parámetros se almacenan en `/assets/data/demons/demons.json` (o YAML, TML — según preferencia):

```json
{
  "demons": [
    {
      "id": "fuego",
      "resistencias": {
        "fuego": -0.35,
        "hielo": -0.25,
        "agua_debilidad_hp_por_segundo": 3.0
      },
      "modificadores": {
        "velocidad_multiplicador": 1.0,
        "daño_bonus": 0.15,
        "hp_bonus": 5
      },
      "habilidades_activas": [
        {
          "nombre": "Quemadura",
          "cooldown_segundos": null,
          "tasa_hp_por_segundo": 3.0,
          "duracion_segundos": 3.0
        }
      ]
    }
  ]
}
```

**Restricción**: NINGÚN valor debe estar hardcoded en código (GDScript). TODO es data-driven desde archivo de configuración.

### 7.9 Balance Pass Plan

Después de MVP (cuando combate y demonios estén jugables):

1. **Fase 1 (Post-MVP)**: Playtesting interno. Ajustar Arcano, Fuego, Hielo según gameplay feel.
2. **Fase 2 (Pre-Vertical Slice)**: Ajustar sinergias — ¿son demasiado poderosas? ¿Se sienten intencionales?
3. **Fase 3 (Vertical Slice)**: Balance final con playtesting externo. Algunos valores pueden variar ±20% de aquí.

**NO BLOQUEA MVP**: Los valores presentes son suficientemente razonables para validar mecánica y feedback.

---

## 8. Criterios de Aceptación

Estos criterios de aceptación verifican que la Base de Datos de Demonios está **completamente implementada, funcional y lista para depender de ella otros sistemas**.

### 8.1 Carga de Base de Datos

- [ ] **CA-001**: Se carga el archivo de configuración de demonios (`assets/data/demons/demons.json`) sin errores
- [ ] **CA-002**: Cada demonio se carga con todos los campos requeridos (id, nombre, descripcion, tipo, ubicacion, resistencias, modificadores, habilidades, transformacion_visual, efectos_narrativos, historia_obtencion)
- [ ] **CA-003**: Se valida que no hay demonios duplicados (id único)
- [ ] **CA-004**: Se valida que los valores numéricos están en rango seguro (resistencias ±0.50, multiplicadores 0.3-2.0)
- [ ] **CA-005**: El Gato (id: "cat") se carga con estructura diferente y sin errores de validación

### 8.2 Estructura de Demonio Estándar (Fuego, Hielo, Arcano, Mente, Visión, Dash)

- [ ] **CA-006**: Cada demonio tiene al menos 1 habilidad (pasiva, activa, o ambas)
- [ ] **CA-007**: Las habilidades activas tienen cooldown definido (en segundos o None para pasivas)
- [ ] **CA-008**: Las resistencias están dentro del rango -0.50 a +0.50
- [ ] **CA-009**: Los modificadores de movimiento están dentro de 0.3× a 2.0×
- [ ] **CA-010**: HP bonus y daño bonus están en rango seguro (HP 0-20, daño -0.25 a +0.50)
- [ ] **CA-011**: Cada demonio tiene transformacion_visual con descripción y aura
- [ ] **CA-012**: Cada demonio tiene historia_obtencion y ubicacion_obtencion documentadas
- [ ] **CA-013**: Dash está marcado como demonio_otorgado = true (siempre disponible)

### 8.3 Estructura de Gato Legendario

- [ ] **CA-014**: Gato tiene ubicacion = "Gato" (en el gato, no en Edrick)
- [ ] **CA-015**: Gato está marcado como es_primer_demonio = true
- [ ] **CA-016**: Gato NO tiene resistencias de combate asignadas (campo vacío o null)
- [ ] **CA-017**: Gato tiene rol_narrativo documentado
- [ ] **CA-018**: Gato tiene interacciones_narrativas documentadas (con_mente, con_vision)
- [ ] **CA-019**: Gato tiene restricción clara: "NO sinergias mecánicas"

### 8.4 Sinergias Mecánicas Positivas

- [ ] **CA-020**: Arcano + Fuego aplica amplificación a quemadura (verifica en gameplay)
- [ ] **CA-021**: Arcano + Hielo amplifica duración de congelación
- [ ] **CA-022**: Arcano + Mente amplifica esquivas (de 4 a 5 ataques evadidos)
- [ ] **CA-023**: Arcano + Visión muestra narrativa "amplificada" (visiones más nítidas)
- [ ] **CA-024**: Arcano + Dash amplifica velocidad (400 px/s → 500 px/s aproximadamente)
- [ ] **CA-025**: Mente + Visión desbloquea diálogos especiales/narrativa compartida
- [ ] **CA-026a**: Dash + Fuego genera "Estela Ardiente" visible durante 0.8s después del dash
- [ ] **CA-026b**: Estela Ardiente aplica quemadura (2 HP/s durante 2.0s) a enemigos que la toquen
- [ ] **CA-027a**: Dash + Hielo genera "Estela Congelada" visible durante 0.8s después del dash
- [ ] **CA-027b**: Estela Congelada ralentiza enemigos 40% por 1.5 segundos

### 8.5 Sinergias Mecánicas Negativas

- [ ] **CA-028**: Fuego + Hielo causa "Anulación Térmica" (-15% daño ambos)
- [ ] **CA-029**: Fuego + Agua causa daño continuo (3 HP/s) automáticamente
- [ ] **CA-030**: Hielo + Fuego ralentiza movimiento (-0.15 multiplicador)
- [ ] **CA-031**: Corrupción + Mente reduce esquivas disponibles
- [ ] **CA-032**: Arcano + Corrupción invierte amplificación (amplifica debilidades)

### 8.6 Sinergias Narrativas (Gato)

- [ ] **CA-033**: Mente + Gato — en Acto 1, muestra diálogo vago (no telepático). En Acto 2 post-evento "La Voz del Gato", comunicación telepática real activada — verificar que NO aparece diálogo telepático completo antes del evento
- [ ] **CA-034**: Visión + Gato desbloquea "Revelación Felina" (visiones fragmentadas, nunca revelan explícitamente la identidad del gato en Acto 1-2)
- [ ] **CA-035**: Mente + Visión + Gato desbloquea "Comunión Ancestral" — revelación profunda solo disponible en progresión Acto 3 (verificar que no hay revelación del giro del hermano antes)
- [ ] **CA-036**: Gato NO sinergiza con Arcano (verificar que Arcano no amplifica Gato)

### 8.7 Aplicación de Modificadores de Movimiento

- [ ] **CA-037**: Equipo Fuego → velocidad 250 × 1.0 = 250 px/s (velocidad base sin cambio)
- [ ] **CA-038**: Equipo Hielo → velocidad 250 × 0.90 = 225 px/s (verificar en gameplay)
- [ ] **CA-039**: Equipo Mente → velocidad 250 × 1.05 = 262.5 px/s (ligeramente más ágil)
- [ ] **CA-040**: Cambiar de demonio → modificadores nuevos se aplican inmediatamente (sin lag)
- [ ] **CA-041**: Desactivar todos demonios → modificadores vuelven a 1.0× (velocidad 250 px/s)

### 8.8 Aplicación de Resistencias y Daño

- [ ] **CA-042**: Fuego equipo → +35% resistencia a fuego aplicada en cálculo de daño
- [ ] **CA-043**: Hielo equipo → +30% resistencia a hielo aplicada
- [ ] **CA-044**: Fuego en agua → 3 HP/segundo daño continuo mientras en agua
- [ ] **CA-045**: Hielo equipo → debilidad a fuego (+0.15 en resistencia_defensiva) correctamente aplicada: Hielo recibe 15% más daño de fuego
- [ ] **CA-046**: Resistencias calculadas con convención correcta: negativo = reduce daño (ej: Fuego resiste fuego con −0.35 → recibe 35% menos), positivo = amplifica daño (debilidades). Fórmula: `daño_final = round(daño_base × (1 + mod_atacante) × (1 + resistencia_defensiva))`
- [ ] **CA-047**: Cambio de demonio → resistencias nuevas se aplican inmediatamente

### 8.9 Bonificadores de HP y Daño

- [ ] **CA-048**: Fuego equipo → +5 HP al máximo (75 → 80 HP)
- [ ] **CA-049**: Hielo equipo → +10 HP al máximo (75 → 85 HP)
- [ ] **CA-050**: HP actual NO cambia al equipar (si tenía 50 HP, sigue siendo 50 HP con nuevo máximo)
- [ ] **CA-050b**: Visión equipo → HP máximo DISMINUYE de 75 a 70 (−5 HP). Si HP actual > 70, se reduce a 70.
- [ ] **CA-051**: Daño base 20 + Arcano (+0.20) = 24 daño (fórmula aplicada en ataque)
- [ ] **CA-052**: Cambio de demonio → bonificadores de daño nuevos aplican al próximo ataque
- [ ] **CA-052b**: Dash + Fuego + Hielo equipados simultáneamente → ambas estelas se generan; quemadura al 85% (1.7 HP/s) y ralentización al 85% (34%). Verificar que no se cancelan entre sí.

### 8.10 Disponibilidad y Desbloqueos

- [ ] **CA-053**: Dash siempre está disponible (demonio_otorgado = true)
- [ ] **CA-054**: Gato está disponible después de evento narrativo de primer binding
- [ ] **CA-055**: Los otros demonios se desbloquean según historia_obtencion
- [ ] **CA-056**: Desbloqueos narrativos se marcan en Estado del Mundo (persistente en saves)
- [ ] **CA-057**: Si demonio "desaparece" en trama (evento narrativo), no está disponible en Loadout hasta que regrese

### 8.11 Integración con Otros Sistemas

- [ ] **CA-058**: Loadout UI carga lista de demonios disponibles desde esta BD
- [ ] **CA-059**: Combate lee habilidades_activas del demonio equipado al iniciar combate
- [ ] **CA-060**: Motor de Sinergias recalcula sinergias cuando Loadout cambia demonio
- [ ] **CA-061**: Transformación Visual lee transformacion_visual del demonio equipado
- [ ] **CA-062**: Bestiario muestra todos los demonios desbloqueados con información completa
- [ ] **CA-063**: Vinculación narrativa muestra historia_obtencion y efectos_narrativos al vincular

### 8.12 Edge Cases

- [ ] **CA-064**: Cambiar demonio durante combate → modificadores y resistencias nuevas aplican inmediatamente, efectos antiguos persisten
- [ ] **CA-065**: Equipo demonio con bajo HP → máximo aumenta, HP actual no cambia
- [ ] **CA-066**: Desactivar todos demonios → Edrick vuelve a stats base (excepto Dash)
- [ ] **CA-067**: Gato con múltiples demonios en Edrick → sinergias de Edrick no afectan a Gato
- [ ] **CA-068**: Daño en agua con Fuego → aplica simultáneamente 3 HP/s + posible daño enemigo
- [ ] **CA-069**: Lectura Mental con NPC que miente → si falla, revela información falsa útil (ambigüedad funcional)
- [ ] **CA-070**: Gato desaparece narrativamente → no disponible en Loadout hasta que regresa

### 8.13 Testing Checklist

**Unit Tests** (validación de datos):
- [ ] Esquema de demonio válido (campos requeridos, tipos correctos)
- [ ] Valores dentro de rangos seguros (resistencias, multiplicadores, bonificadores)
- [ ] Sinergia de Arcano calcula amplificación correctamente
- [ ] Fórmula de resistencia calcula daño correctamente

**Integration Tests** (con otros sistemas):
- [ ] Loadout → BD: Lee demonios correctamente
- [ ] Combate → BD: Lee habilidades y aplica cooldowns
- [ ] Resistencias → Salud/Daño: Aplica en cálculo de daño
- [ ] Transformación Visual → BD: Aplica transformacion_visual

**Manual Testing** (gameplay):
- [ ] Equipo cada demonio → cambios visuales, movimiento, resistencias correctos
- [ ] Sinergias positivas → se sienten intencionales y útiles
- [ ] Sinergias negativas → penalización es evidente
- [ ] Gato narrativo → interacciones con Mente/Visión funcionan
- [ ] Cambios mid-combate → transición suave sin glitches
- [ ] Dash + Fuego → estela de llamas visible 0.8s, quema enemigos
- [ ] Dash + Hielo → estela de hielo visible 0.8s, ralentiza enemigos

### 8.14 Consideraciones de Aceptación

- **NO BLOQUEA si**: Valores numéricos cambian ligeramente durante balance (±10-20%), siempre que lógica de sinergias y estructura sean correctas
- **BLOQUEA si**: Demonio falta campo requerido, sinergia no aplica, resistencia no se calcula
- **REVISIÓN REQUERIDA si**: Gato tiene sinergias mecánicas (violación de restricción) o interacciones narrativas no funcionan
- **POST-MVP BALANCE**: Los valores pasarán por ajustes durante playtesting; estructura y lógica son lo crítico para aceptación
