# Sistema de Audio

> **Estado**: En Revisión (revisión mayor completada 2026-05-25)  
> **Creado**: 2026-05-25  
> **Última Actualización**: 2026-05-25

---

## 1. Visión General

**Sistema de Audio** define todos los marcos de audio del juego: cómo se organizan eventos sonoros, cuándo se activan, cómo los demonios modifican sonidos, y cómo el audio señala la transformación moral de Edrick.

Este sistema es **reactivo**: cada acción de gameplay (golpe, demonio equipado, moralidad corrupta, área explorada) genera un evento de audio. El sistema no activa sonidos en aislamiento — activa en respuesta a la máquina de estado del juego (combate, exploración, narrativa, menú).

El audio está dividido en **4 capas que se mezclan dinámicamente**: Música (ambiente + narrativa), SFX de Combate (abilities, hits, enemy actions), SFX Ambiental (exploración, NPCs, mundo), y Diálogos (VO, gatos vocales narrativos). Cada capa tiene su propio árbol de eventos, ducking rules (reducción de volumen cuando otras capas dominan), y configuración dinámica.

Las **transformaciones del jugador** (demonios equipados, corrupción moral) modifican el audio en tiempo real: la música se tiñe progresivamente oscura conforme subes en corrupción, los efectos de demonios agresivos suenan más siniestros cuando combinados, la voz de Edrick cambia sutilmente de tono.

---

## 2. Fantasy del Jugador

El audio del juego entrega **cuatro fantasías centrales al jugador**:

1. **Narrativa cinematográfica sonora**: Cada reino tiene una identidad musical distinta que cuenta su historia. La música no es ambiental pasiva — es un personaje. Transiciones entre regiones, encuentros con NPCs, y momentos de revelación todas vienen con cambios musicales que refuerzan la emoción narrativa. El sonido de un demonio es su "voz" — cuando lo equipas, ese demonio suena diferente (tono vocal, efectos, música de combate).

2. **Combate visceral y responsivo**: Cada acción tiene feedback sonoro inmediato. Un golpe suena diferente si impacta a un enemigo vs. si fallas. Una habilidad de demonio es dramática — fuego crepita y ruge, hielo cruje y chilla, el Gato aúlla con poder. El jugador siente el peso de cada acción antes de verla.

3. **Mundo vivo y habitable**: La exploración suena viva. Pájaros en el bosque, viento en cavernas, agua cayendo, NPCs pasivos con voces sutiles. El mundo no es silencioso — está respirando. Conforme te alejas de lo seguro, el audio ambiental se vuelve más siniestro (un reino oscuro suena peligroso).

4. **Transformación moral audible**: La corrupción de Edrick se escucha. La música comienza limpia y hermosa. Conforme subes en corrupción (Act 1 → Act 2 → Act 3), la música se tiñe de oscuridad — acordes menores, disonancias sutiles, distorsión progresiva. Por el acto final, incluso la música "hermosa" del juego tiene un borde corrupto. La voz de Edrick cambia de tono — comienza juvenil y esperanzador, termina cansado y resentido. El gato suena progresivamente diferente — más demoníaco conforme la verdad se aproxima.

---

## 3. Reglas Detalladas

### 3.1 Las Cuatro Capas de Audio

Toda producción de audio se organiza en **cuatro capas independientes** que se mezclan dinámicamente. Cada capa tiene su propio bus, controles de volumen, y reglas de activación.

- **Capa 1: Música** (orquesta, síntetizers, temas narrativos)
  - Música ambiental por reino (loop continuo durante exploración)
  - Música de combate (activada cuando entra en combate; loop escalado dinámicamente)
  - Música narrativa (cinemáticas, momentos de revelación — no loopea, es secuencial)
  - Transiciones suaves entre estados (cross-fade de 0.5–2.0s según contexto)

- **Capa 2: SFX de Combate** (habilidades de demonios, golpes, estados)
  - Cada habilidad de demonio tiene su propia "firma sonora" (Fuego = rugido + crépito, Hielo = chillido + cristal, etc.)
  - Golpes: sonido diferente si impacta vs. falla, escalado a daño infligido
  - Estados (quemadura, envenenamiento, debuff): sonido de tick continuo mientras activo
  - Muertes de enemigos: sonido de muerte escalado a tipo de demonio usado

- **Capa 3: SFX Ambiental** (mundo, NPCs, eventos)
  - Efectos de entorno por región (viento, agua, animales, luz ambiental)
  - NPCs en rango: voces pasivas, respiración, movimiento
  - Puertas, interactuables, eventos del mundo (aperturas, caídas)
  - Nunca silencioso — siempre hay algo sutilmente ocurriendo

- **Capa 4: Diálogos** (voz de Edrick, NPCs, gato narrativo)
  - Diálogos de NPCs: VO con sincronización de sprites
  - Voz interna de Edrick: sutileza en monólogos (pensamiento, duda, corrupción)
  - Vocalizaciones del Gato: bufidos, aullidos, ronroneos — no VO, pero comunicación clara
  - Radio de activación: solo NPCs en rango visual moderado

### 3.2 Máquina de Estados de Audio y Eventos

El sistema de audio opera en tres **contextos de estado** que determinan qué capas están activas y cómo se mezclan:

- **Estado: Exploración**
  - Capa Música: Ambiental + Transición (activa)
  - Capa SFX Combate: Inactiva
  - Capa SFX Ambiental: Activa (full volume)
  - Capa Diálogos: Activa si NPC en rango
  - Ducking: Ninguno. Diálogos pueden atenuar música ligeramente si hablan.

- **Estado: Combate**
  - Capa Música: Música de Combate (loop escalado, comienza de cero siempre) (activa)
  - Capa SFX Combate: Activa (full volume — habilidades son dramáticas)
  - Capa SFX Ambiental: Atenuada a −12 dB (mundo recede)
  - Capa Diálogos: Inactiva (sin NPCs hablando durante combate)
  - Ducking: SFX Combate atenúa música a −6 dB cuando hay impactos simultáneos

- **Estado: Narrativa (Cinemática)**
  - Capa Música: Música Narrativa (secuencial, no loop) (activa)
  - Capa SFX Combate: Inactiva
  - Capa SFX Ambiental: Atenuada a −18 dB (subtítulos solo)
  - Capa Diálogos: Activa (full volume)
  - Ducking: Música atenúa a −9 dB cuando Diálogos activos

- **Estado: Menú**
  - Capa Música: Tema de Menú (loop)
  - Todas las demás capas: Inactivas
  - Ducking: Ninguno

### 3.3 Modificadores de Demonios en Audio

Cuando un **demonio está equipado**, modifica el audio en tiempo real:

- **Fuego**: SFX de combate adquieren perfil "roar_crackle" (distorsión cálida). Música: crossfade a variante pre-autorada `mus_[contexto]_fuego.ogg` (tonalidad menor, orquestación más oscura). SFX toma reverb "interior caliente".
- **Hielo**: SFX adquieren perfil "shrill_crystalline" (reverb frío, agudos). Música: crossfade a variante `mus_[contexto]_hielo.ogg` (tonos sintéticos/ethereales). SFX toma reverb "espacio vacío".
- **Arcano**: SFX adquieren perfil "dimensional_glitch" (pitch shifts, glitch sutil). Música: crossfade a variante `mus_[contexto]_arcano.ogg` (arpeggios discordantes). Amplifica todos los demás +25%.
- **Visión**: SFX adquieren perfil "radio_static". Música: crossfade a variante `mus_[contexto]_vision.ogg` (cadencias oníricas, reverb profundo).
- **Mente**: SFX adquieren perfil "binaural_pulses". Música: crossfade a variante `mus_[contexto]_mente.ogg` (ritmo hipnótico).
- **Dash**: SFX adquieren perfil "wind_sonic_boom". Música: crossfade a variante `mus_[contexto]_dash.ogg` (mismo tema, tempo 15% más rápido — entregado como asset pre-renderizado, NO como DSP en tiempo real).
- **Gato**: SFX adquieren perfil "demonic_howl" (brutales, sin efervescencia). Gato siempre audible (ver floor en sección 5.5).

**Implementación de variantes musicales**: El compositor entrega M variantes por contexto musical (M = número de demonios con music_variant_asset definido). Al equipar un demonio, AudioManager hace crossfade de 0.5s al asset de variante (`AudioStreamPlayer.stream` swap). Al desequipar, crossfade de vuelta al tema base del contexto. Si múltiples demonios con variants están equipados simultáneamente, se usa la variante del demonio con mayor `intensity_multiplier` (desempate: orden de equipado).

Los **modificadores se apilan**: si equipas Fuego + Dash, los sonidos de combate tienen tanto rugidos como ráfagas. Si equipas Arcano (que amplifica), todo suena 25% más potente (ver fórmula 4.4 y sección 7.5 para clarificación de unidades).

### 3.4 Corrupción Moral y Transformación Sonora

La **corrupción moral de Edrick** (almacenada en `Estado del Mundo.corruption_level`, rango 0.0–1.0) modifica el audio ambiental continuamente:

- **Corrupción 0.0 (Limpio)**: Música mayor, clara, esperanzadora. SFX ambiental "natural" (aves, agua, viento limpio). Voz de Edrick: juvenil, decidida.
- **Corrupción 0.3 (Acto 1 Progreso)**: Acordes menores comienzan a entrar en música. SFX ambiental comienza a sonar "vigilante" (ecos, pasos lejanos). Voz de Edrick: más grave.
- **Corrupción 0.6 (Acto 2)**: Música tiene disonancias claras, cambios a tonalidades oscuras. SFX ambiental: doblado/procesado, como si fuera escuchado a través de agua. Voz de Edrick: cansada, con borde.
- **Corrupción 1.0 (Acto 3)**: Música completamente distorsionada, reverb denso, temas originales inverted/reversos. SFX ambiental: corrompido, glichy, apenas reconocible. Voz de Edrick: siniestra, poco Edrick.

Las **transiciones entre niveles de corrupción son suaves** pero **event-driven**: el filtro de audio se actualiza ÚNICAMENTE cuando llega el evento `corruption_level_changed(nuevo_nivel)` desde Estado del Mundo. No hay polling continuo. La corrupción solo cambia en momentos morales discretos (ejecutar un NPC, abandonar aliado, etc.) — no drifta automáticamente. La suavidad perceptual proviene del `corruption_filter_update_speed_s: 0.2` (el filtro interpola durante 200ms al recibir el evento, no hay cambios frame-a-frame).

### 3.5 Integración: Cómo Otros Sistemas Activan Audio

Otros sistemas generan **eventos de audio** que este sistema consume:

- **Sistema de Combate**: Emite eventos `ability_used(demonio, habilidad)`, `hit_landed(target, attack_data)`, `miss()`, `enemy_died()` → Audio responde con SFX de Combate (Audio lee `attack_data.daño_final` para escalar intensidad)
- **Loadout & Demonios**: Emite evento `demon_equipped(demonio)` → Audio activa modificadores de ese demonio
- **Estado del Mundo**: Emite evento `corruption_changed(nuevo_nivel)` → Audio ajusta "filter" de corrupción
- **Sistema de NPC/Diálogo**: Emite evento `dialogue_started(npc)` → Audio atenúa música, activa VO
- **Exploración**: Emite evento `area_entered(reino)` → Audio fade a música ambiental de ese reino
- **Progresión Narrativa**: Emite evento `cinematic_started(id)` → Audio cambia a Estado Narrativa, activa música de cinemática

---

## 4. Fórmulas

### 4.1 Cálculo de Ducking Dinámico

Cuando múltiples capas compiten, el **ducking reduction** es calculado basado en qué capa tiene prioridad en el estado actual.

**Fórmula: Atenuación Dinámica**
```
master_music_level_dB = base_music_level_dB − (ducking_amount_dB × priority_factor)
```
Donde:
- `base_music_level_dB` = volumen base de música (e.g., −3 dB por defecto)
- `ducking_amount_dB` = cantidad máxima a atenuar (varía por estado)
- `priority_factor` = cuántos eventos de prioridad alta están activos (0.0–1.0)

**En Combate** (SFX Combate tiene prioridad):
- `ducking_amount_dB = 6` (música baja máximo 6 dB cuando hay golpes)
- `priority_factor = min(número_impactos_simultáneos / 3.0, 1.0)` — **clamp obligatorio a 1.0**; sin él, 6 impactos simultáneos producirían priority_factor = 2.0 y ducking de −15 dB (incorrecto)
- Ejemplo: 2 impactos → `priority_factor = min(2/3, 1.0) = 0.67` → música baja `6 × 0.67 = 4 dB`
- Ejemplo: 6 impactos (cap máximo) → `priority_factor = min(6/3, 1.0) = 1.0` → música baja `6 × 1.0 = 6 dB` (máximo correcto)

**En Narrativa** (Diálogos tienen prioridad):
- `ducking_amount_dB = 9` (música baja máximo 9 dB cuando hay VO activo)
- `priority_factor` = 1.0 si VO está activo, 0.0 si no hay VO

**Transición entre estados** (cross-fade):
- `fade_duration_seconds` = 0.5s en combate rápido; 2.0s en transiciones narrativas lentas
- `volume_at_t = lerp(old_volume, new_volume, t / fade_duration_seconds)` para t ∈ [0, fade_duration_seconds]

### 4.2 Escalado de SFX Basado en Daño

La **intensidad de un SFX de impacto** es proporcional al daño infligido:

**Fórmula: Intensidad de Impacto**
```
// Clamp obligatorio: daño_final puede exceder HP_max con multiplicadores de demonios
impact_intensity = min((daño_final / HP_max) × 100, 66.7)
impact_pitch_shift = 1.0 + (impact_intensity / 200)
impact_volume_dB = −12 + (impact_intensity × 0.15)
```
Donde:
- `daño_final` = daño final después de resistencias (de Salud/Daño.md) — puede superar HP_max con multiplicadores
- `HP_max` = 75 (HP base de Edrick)
- `impact_intensity` **clampada a 66.7** (máximo razonable: 50 HP / 75 HP_max × 100 = 66.7%). Sin el clamp, daño > HP_max produce `impact_volume_dB > 0 dB`, que clipea contra el limiter de 0 dB
- `impact_pitch_shift` rango: 1.0–1.334 (pitch sube hasta +33.4% al clamp máximo)
- `impact_volume_dB` rango: −12 dB a +3 dB (clamped; el limiter a 0 dB bloquea clips)

**Nota sobre daño 0**: Si `daño_final == 0` (golpe bloqueado, inmunidad), el sistema NO emite evento de impacto al bus de audio. Un impacto de daño 0 no produce SFX de combate — usa el asset separado "blocked_hit" si aplica (definido en GDD Combate #6).

**Ejemplo**: Un golpe de 20 daño en Edrick (75 HP max):
- `impact_intensity = (20 / 75) × 100 = 26.7%`
- `pitch_shift = 1.0 + (26.7 / 200) = 1.134` (pitch sube 13.4%)
- `volume_dB = −12 + (26.7 × 0.15) = −12 + 4 = −8 dB` (mucho más fuerte que impacto mínimo)

### 4.3 Filtro de Corrupción Moral

El **filtro de corrupción** ajusta parámetros de audio en tiempo real basado en `corruption_level` (0.0–1.0).

**Fórmula: Tone Shift (EQ)**
```
tone_shift_dB = corruption_level × −8 (reduce brillantez a medida que se corrompe)
bass_boost_dB = corruption_level × 6 (aumenta bajos a medida que se corrompe)
distortion_amount = corruption_level × 0.4 (max 40% distorsión a corrupción total)
```

**Fórmula: Reverb Corruption**
```
reverb_decay_ms = 800 + (corruption_level × 2200)  // 800ms a 0.0, 3000ms a 1.0
// Interpolación lineal-en-dB entre mínimo y máximo:
reverb_wet_level_dB = reverb_wet_min_dB + (corruption_level × (reverb_wet_max_dB - reverb_wet_min_dB))
                    = −48 + (corruption_level × 42)  // dB range: 48 dB
```
- `reverb_wet_min_dB = −48 dB` (ultra-silencio a corrupción 0.0; implementable en Godot AudioEffectReverb como valor numérico, no como −∞)
- `reverb_wet_max_dB = −6 dB` (50% wet a corrupción 1.0)
- A corrupción 0.0: reverb_wet = −48 dB (funcionalmente inaudible)
- A corrupción 0.5: reverb_wet = −48 + 21 = −27 dB (reverb sutil audible)
- A corrupción 1.0: reverb_wet = −48 + 42 = −6 dB (reverb denso, 50% wet)

**Ejemplo**: A corrupción 0.6 (Acto 2):
- `tone_shift = 0.6 × −8 = −4.8 dB` (brillantez reducida)
- `bass_boost = 0.6 × 6 = 3.6 dB` (bajos más presentes)
- `distortion = 0.6 × 0.4 = 0.24` (24% distorsión)
- `reverb_decay = 800 + (0.6 × 2200) = 1920 ms`
- `reverb_wet = −48 + (0.6 × 42) = −48 + 25.2 = −22.8 dB` (reverb moderado-sutil)

### 4.4 Apilamiento de Modificadores de Demonios

Cuando múltiples demonios están equipados, sus **modificadores se apilan multiplicativamente** (excepto Arcano, que es aditivo con todos):

**Fórmula: Intensidad de Demonio**
```
base_intensity = 1.0
intensity_with_modifiers = base_intensity × ∏(demonio_multiplier_i) × (1 + arcano_boost)
arcano_boost = 0.25 si Arcano equipado, 0.0 si no
```
Donde `demonio_multiplier_i`:
- Fuego, Hielo, Mente, Visión, Dash: multiplicadores 1.0 (neutrales; cambian timbre, no intensidad)
- Gato: multiplicador 2.0 (dobla intensidad de cualquier habilidad)
- Arcano: aplica como suma, no multiplicador (amplifica todos en +25%)

**Ejemplo**: Gato + Fuego + Arcano equipados:
- `intensity = 1.0 × 1.0 (Fuego) × 2.0 (Gato) × (1 + 0.25 Arcano) = 2.5`
- Habilidades de Fuego suenan 2.5× más intensas/más ruidosas

**Ejemplo**: Triple Dash + Fuego + Hielo (restricción de sinergia especial):
- El sistema reconoce esta combinación y activa "Anulación Térmica" (de base-datos-demonios.md)
- `estela_efectividad = 0.85` para ambas estelas Fuego e Hielo (audio ambas presentes pero 15% atenuadas cada una)

### 4.5 Duración de Transiciones Entre Estados

Las transiciones de audio entre estados tienen **duración variable** según contexto narrativo:

**Fórmula: Fade Duration**
```
fade_duration_s = base_fade_duration × urgency_factor
```
Donde:
- **Exploración → Combate**: `base_fade = 0.3s, urgency_factor = 1.0` → total 0.3s (rápido, entrada sorpresiva)
- **Combate → Exploración**: `base_fade = 1.0s, urgency_factor = 1.0` → total 1.0s (salida calma)
- **Cualquier estado → Cinemática**: `base_fade = 1.5s, urgency_factor = emotional_weight` → 1.5–2.5s (lento, ceremonial)
  - `emotional_weight` es un campo por cinemática en el asset de datos de la cinemática: `cinematic_data[cinematic_id].emotional_weight`
  - Valores válidos: `1.0` (normal — revelaciones menores, 1.5s total) o `1.67` (máximo peso emocional — giro del gato, finale — 2.5s total)
  - Default si no especificado: `1.0`. El sistema Audio lee este campo del payload del evento `cinematic_started(cinematic_id, emotional_weight: float)`
- **Cualquier estado → Menú**: `base_fade = 0.5s, urgency_factor = 1.0` → total 0.5s

---

## 5. Casos Extremos

### 5.1 Sobrecarga de SFX Simultáneos

Si **más de 6 sonidos de impacto se activan en el mismo frame**, el sistema **limita a los 6 más fuertes** (ordenados por `impact_intensity` descendente). Los demás se silencian completamente (no se atenúan, se descartan).

**Regla**: Máximo 6 impactos concurrentes. Si el usuario equipa Gato (multiplicador 2.0) y otros demonios crean múltiples habilidades simultáneamente, esto puede causar sobrecarga. El sistema prioriza por intensidad, no por orden de activación.

**Ejemplo**: Gato + Fuego con 3 golpes + Fuego habilidad (quemar) + Hielo habilidad = 5 sonidos totales. Todos se reproducen. Si un 6to evento intenta activarse, se filtra por intensidad.

### 5.2 Transiciones Rápidas Entre Estados

Si el jugador **entra en combate, es golpeado, gana/pierde (vuelve a exploración), y entra en un diálogo — todo en <3 segundos**, el sistema **mantiene las transiciones programadas** (no "salta" a la siguiente, respeta la máquina de estados).

**Regla**: Cada transición tiene su fade_duration (0.3s, 1.0s, 1.5s, etc.). Si una transición está en progreso y una nueva se dispara:
- La música **continúa fading al destino original**
- Las capas en **conflicto de prioridad se resuelven por el estado ACTUAL**
- **Caso**: En combate (música de combate), sale de combate (fade a exploración 1.0s), pero interrumpido por cinemática (0.5s después del inicio del fade). La música sigue hacia exploración (porque aún está en la transición original), pero si cinemática activada DESPUÉS de que explore termine, salta a narrativa.

### 5.3 Corrupción Aumentando Durante Combate

Si la **corrupción moral de Edrick cambia** (ej: ejecuta un enemigo, +0.10 corrupción) **mientras está en combate activo**, el filtro de corrupción se actualiza en tiempo real:

- Cambio de corrupción → actualiza `tone_shift_dB`, `bass_boost_dB`, `distortion_amount` inmediatamente
- La música de combate se re-filtra dentro de 0.2s (suave, no jarring)
- SFX de combate también se re-filtran (ejemplo: siguiente golpe ya suena más corrupto)
- **No interrumpe la música o SFX actuales** — continúan, pero con el nuevo filtro aplicado

### 5.4 Demonio Se Desequipa Durante Habilidad en Progreso

Si el jugador **desequipa un demonio mientras su habilidad está sonando**:

- El **SFX de la habilidad continúa** hasta completarse (no se corta)
- Los **modificadores de demonio se inhabilitan** para futuros eventos
- Si el usuario cambia a un demonio diferente **antes de que el SFX termine**, el nuevo demonio NO modifica el SFX en progreso (solo aplica a nuevos eventos)

**Ejemplo**: Fuego equipado, activas "quemar" (sonido de 3s de quemadura). En 1s, desequipas Fuego. El sonido de quemadura continúa por 2s más, sin modificadores de Fuego (suena "normal", no distorsionado). Si equipas Hielo durante esos 2s, Hielo NO afecta el sonido de quemadura.

### 5.5 Gato Vocalizando Mientras Otros Sonidos Dominan

El **Gato siempre se escucha** — sus vocalizaciones nunca se ducking más allá de −3 dB incluso en combate intenso. Si Gato vocaliza durante un impacto fuerte (ambos de prioridad alta):

- Música se duckea por el impacto (−6 dB)
- Vocalización del Gato **NO se atenúa** (permanece a −3 dB máximo)
- SFX de combate está a full
- **Resultado**: Gato permanece audible y claramente diferenciado

**Justificación narrativa**: El Gato es el compañero de Edrick. Su voz siempre debe ser clara.

### 5.6 NPC Hablando Mientras Hay Combate Cercano

Si un **NPC comienza un diálogo mientras hay combate aconteciendo en la misma región**:

- El estado **sigue siendo "Exploración"** (no ha entrado en combate Edrick)
- El NPC dialog toma prioridad de audio (música −9 dB, SFX ambiental −6 dB)
- Los **SFX de combate del enemigo lejano se atenúan a −15 dB** (muy lejano en mezcla)
- **Resultado**: El jugador escucha claramente el NPC, pero oye ecos de combate lejano. Es narrativamente realista (hay pelea afuera, tú estás adentro hablando).

### 5.7 Pausa del Juego

Cuando el usuario **pausa** (menú de pausa, no cinemática):

- **Música**: Se pausa (se congela el playhead, no desaparece)
- **SFX**: Se pausan todos (desaparecen)
- **Diálogos**: Se pausan
- Al despausar: Todo retoma desde donde se pausó (música retoma el playhead exacto, SFX se reinician, diálogos continúan)

**Excepción**: Si pausa durante una **cinemática**, la música continúa (el jugador quiere escuchar la narrativa con paciencia).

### 5.8 Cambio de Loadout de Demonios During Cinemática

Si el sistema **intenta cambiar el loadout** (ej: via scripting narrativo) **mientras una cinemática está activa**:

- El cambio de loadout **se cola** (no se aplica hasta que cinemática termina)
- Los nuevos modificadores de demonio se aplican **solo después de que cinemática transiciona a exploración/combate**
- La música narrativa NO es afectada por cambios de demonio durante la cinemática

**Justificación**: Las cinemáticas están bajo control narrativo, no del sistema de audio reactivo. Cambios de demonio son mecánicos, no narrativos.

### 5.9 VO Muy Largo (Diálogo que Excede Ducking)

Si un **diálogo de NPC dura más de 15 segundos** (raro, pero posible en narrativa compleja):

- El **ducking de música se mantiene constante** (−9 dB) por la duración completa
- Si la música de "exploración" tiene loops de 8s, puede sonar repetitiva bajo ducking
- **Solución diseño**: Mantener loops de diálogo en <10s, o usar música narrativa separada (no loopeable) para estos momentos

### 5.10 Desactivación de Audio (Config de Usuario)

Si el usuario **desactiva todas las capas de audio** (modo silencio):

- El sistema **continúa calculando** (ducking, modificadores, corrupción) pero con volumen output = 0
- Si el usuario **reactiva el audio**, el sistema retoma exactamente donde estaba (no desfase)
- **Subtítulos siempre visibles** si audio desactivado (compensación de accesibilidad)

### 5.11 Música Que No Loopea (Cinemática Muy Larga)

Si una **cinemática tiene música narrativa que no loopea** y **la cinemática dura más de la duración total de la música**:

- La música **se desvanece a silencio** en los últimos 2 segundos (fade-out de 2s)
- **Diálogos permanecen audibles** (la música no interfiere)
- Cuando cinemática termina, música narrative se descarga y el sistema vuelve a música ambiental de exploración

**Justificación**: Las cinemáticas muy largas son raras. Si ocurren, la música no debe repetirse de manera incomoda.

---

## 6. Dependencias

### 6.1 Sistemas Que Dependen de Audio

**8 sistemas consultan y activan eventos de Audio**:

1. **Sistema de Combate en Tiempo Real** (GDD #6)
   - Emite: `ability_used(demonio, habilidad_id)`, `hit_landed(target, attack_data)`, `miss()`, `enemy_died(tipo_enemigo)`, `player_damaged(daño)` (Audio lee `attack_data.daño_final`)
   - Espera: SFX de habilidad, SFX de impacto escalado, SFX de muerte del enemigo
   - Audio responde: Activa SFX de combate escalado por daño; aplica modificadores de demonio

2. **Loadout & Build Management** (GDD #10)
   - Emite: `demon_equipped(demonio)`, `demon_unequipped(demonio)`
   - Espera: Audio entiende qué demonios está el jugador usando para modificadores
   - Audio responde: Cambia timbre/intensidad de futuros SFX basado en demonios equipados

3. **Estado del Mundo** (GDD #4)
   - Emite: `corruption_level_changed(nuevo_nivel)`, `area_entered(reino_id)`, `save_loaded()`
   - Espera: Audio debe mantener sincronización de corrupción y contexto de región
   - Audio responde: Actualiza filtro de corrupción; cambia música ambiental por región

4. **Sistema de NPC y Diálogo** (GDD #15)
   - Emite: `dialogue_started(npc_id)`, `dialogue_ended()`, `vo_line_play(npc_id, line_id)`
   - Espera: Audio activa VO, sincroniza labios (si aplicable), atenúa otras capas
   - Audio responde: Atenúa música/SFX ambiental; activa VO; restaura volumen al terminar

5. **Progresión Narrativa** (GDD #16)
   - Emite: `cinematic_started(cinematic_id)`, `cinematic_ended()`
   - Espera: Audio cambia a estado narrativo, carga música de cinemática
   - Audio responde: Transiciona a música narrativa; atenúa SFX ambiental; establece contexto de diálogo prioritario

6. **Exploración del Mundo** (GDD #8)
   - Emite: `region_entered(región)`, `region_exited()`, `environmental_event(tipo, ubicación)`
   - Espera: Audio modifica música ambiental y SFX ambientales por región
   - Audio responde: Fade a música ambiental del nueva región; activa SFX ambientales regionales

7. **Transformación Visual de Edrick** (GDD #14)
   - Emite: `visual_state_changed(demonio_actual, corruption_level)`
   - Espera: Audio debe estar en sincronización con transformación visual
   - Audio responde: Ya está sincronizado (comparte datos de corruption_level y demonio_equipado desde Estado del Mundo)

8. **Seguimiento Moral** (GDD #22, Vertical Slice)
   - Emite: `moral_choice_made(tipo_accion, moral_weight)`
   - Espera: Audio responde a cambios de moralidad (parte del filtro de corrupción)
   - Audio responde: Actualiza corrupción → filtro aplica progresivamente

### 6.2 Sistemas de Que Audio Depende

**Audio tiene dependencias mínimas pero críticas**:

- **Salud y Daño** (GDD #2) — Audio necesita saber `daño_final` para escalar impactos. Lee la fórmula de daño de esta GDD.
- **Base de Datos de Demonios** (GDD #3) — Audio necesita los IDs de demonios y sus modificadores. Usa `demonio_id` para mapear a "firma sonora" de demonio.
- **Estado del Mundo** (GDD #4) — Audio lee `corruption_level` en tiempo real. Accede a `available_demons`, `equipped_demons` para saber qué está equipado.

### 6.3 Contrato de Interfaz: Cómo Otros Sistemas Comunican Con Audio

**Event Bus Architecture**: Todos los sistemas comunican con Audio via **event bus** (emisión de eventos, sin acoplamiento directo).

**Eventos que Audio Escucha** (lista completa):
```
# Combat
Event: ability_used(demonio: string, habilidad_id: string) → Audio activa SFX de habilidad + modificadores
Event: hit_landed(target: Node, attack_data: Dictionary) → Audio escala SFX de impacto por attack_data.daño_final
Event: miss() → Audio activa SFX de "fallo" (whoosh sin impacto)
Event: enemy_died(tipo: string) → Audio activa SFX de muerte
Event: player_damaged(daño: int) → Audio activa SFX de daño a jugador

# Loadout
Event: demon_equipped(demonio: string) → Audio activa modificadores de demonio
Event: demon_unequipped(demonio: string) → Audio desactiva modificadores

# World State
Event: corruption_level_changed(nuevo_nivel: float) → Audio actualiza filtro de corrupción
Event: area_entered(region_id: string) → Audio transiciona a música ambiental de región
Event: save_loaded() → Audio reinicia a estado inicial (corrupción, región actual)

# NPC/Dialogue
Event: dialogue_started(npc_id: string) → Audio atenúa música, prepara VO
Event: dialogue_ended() → Audio restaura volumen
Event: vo_line_play(npc_id: string, line_id: string) → Audio activa archivo VO específico

# Cinematics
Event: cinematic_started(cinematic_id: string) → Audio carga música narrativa
Event: cinematic_ended() → Audio transiciona de vuelta a exploración/combate

# Exploration
Event: region_entered(region_id: string) → Audio cambia música ambiental
Event: environmental_event(tipo: string, ubicacion: vec3) → Audio activa SFX ambiental

# Game State
Event: game_paused() → Audio pausa música/SFX (excepto cinemática)
Event: game_resumed() → Audio retoma desde pausado
Event: game_quit() → Audio descarga todos los recursos
```

### 6.3b Contrato de Error del Event Bus

Cuando Audio recibe un evento con un ID inválido o desconocido:

| Situación | Comportamiento requerido | Log |
|-----------|------------------------|-----|
| `ability_used("demonio_no_existe", ...)` | Silenciar el evento — no crash, no SFX | WARNING: "Unknown demon ID in ability_used: [id]" |
| `demon_equipped("id_invalido")` | Ignorar — no modificadores aplicados | WARNING: "Unknown demon in demon_equipped: [id]" |
| `area_entered("region_no_existe")` | Continuar con música actual sin cambio | WARNING: "No audio config for region: [id], keeping current music" |
| `vo_line_play("npc", "linea_no_existe")` | Silencio para esa línea de VO | WARNING: "VO asset not found: [npc]/[line_id], silence substituted" |
| Evento desconocido | Ignorar completamente | No log (evento desconocido puede ser de otro sistema no relacionado) |

**Regla**: El AudioManager nunca crashea por un evento mal formado. Todos los fallos son silenciosos con WARNING en log. Los errores en los IDs de audio son responsabilidad del sistema emisor, no del Audio.

### 6.4 Sincronización de Datos Compartidos

Audio mantiene **copia local en caché** de:
- `current_demons_equipped` (actualiza cuando escucha `demon_equipped/unequipped`)
- `current_corruption_level` (actualiza cuando escucha `corruption_level_changed`)
- `current_region` (actualiza cuando escucha `region_entered`)
- `current_game_state` (exploración/combate/narrativa/menú)

Estas cachés se sincroniza **con Estado del Mundo** en cada `save_loaded()` para evitar desfase después de cargar partida.

### 6.5 Garantías de Latencia

- **Eventos → Audio Playback**: <50ms (ejecución en frame actual, audio comienza en frame siguiente)
- **Transiciones de Estado**: Respeta fade_duration (0.3s–2.5s según contexto, deliberado para cinematografía)
- **Corrupción Filter Updates**: <200ms (suave, no jarring durante gameplay)
- **Ducking Dynamics**: <100ms (responsive a cambios de prioridad)

---

## 7. Parámetros de Ajuste

### 7.1 Parámetros de Volumen Base

**Ubicación**: `/assets/data/audio/volume_config.json`

Todos los volúmenes base se expresan en **dB** (decibelios). Los rangos seguros son:
- `0 dB` = referencia (no cambio)
- `−∞ dB` = silencio (mute)
- Rango típico: `−60 dB` a `+6 dB`

```json
{
  "music_base_level_dB": -3,          // Música por defecto 3dB debajo de referencia
  "sfx_combat_base_level_dB": 0,      // SFX combate a referencia (dramático)
  "sfx_ambient_base_level_dB": -6,    // SFX ambiental más bajo (fondo)
  "dialogue_base_level_dB": -1,       // Diálogos ligeramente por encima de música
  "cat_vocalization_floor_dB": -3     // Gato nunca cae debajo de este nivel
}
```

**Rangos de Ajuste Seguro**: ±6 dB desde valores base. Cambios mayores requieren re-balance de todo el sistema.

### 7.2 Parámetros de Ducking

**Ubicación**: `/assets/data/audio/ducking_config.json`

```json
{
  "combat_ducking_music_dB": 6,           // Música baja máximo 6dB en combate
  "narrative_ducking_music_dB": 9,        // Música baja máximo 9dB en diálogos
  "exploration_ducking_music_dB": 0,      // Sin ducking en exploración pura
  "combat_ducking_ambient_dB": 12,        // SFX ambiental baja 12dB en combate
  "narrative_ducking_ambient_dB": 18,     // SFX ambiental baja 18dB en diálogos
  "dialogue_ducking_music_dB": 9,         // Música baja cuando VO activo
  "dialogue_ducking_ambient_dB": 6        // Ambiental baja cuando VO activo
}
```

**Rangos de Ajuste**: ±3 dB desde valores base. Mayores cambios harán que capas se "pierdan" en la mezcla.

### 7.3 Parámetros de Transiciones

**Ubicación**: `/assets/data/audio/transition_config.json`

```json
{
  "fade_exploration_to_combat_s": 0.3,
  "fade_combat_to_exploration_s": 1.0,
  "fade_to_narrative_s": 1.5,
  "fade_to_menu_s": 0.5,
  "fade_resume_from_pause_s": 0.5,
  "corruption_filter_update_speed_s": 0.2,
  "cinematic_emotional_weight_normal": 1.0,
  "cinematic_emotional_weight_heavy": 1.67
}
```

**Notas**: 
- Tiempos muy cortos (<0.2s) suenan "jarring" (saltos abruptos)
- Tiempos muy largos (>3s) sienten "lentos" (lag percibido)
- La exploración → combate debe ser rápida (sorpresa); combate → exploración puede ser lenta (catársis)

### 7.4 Parámetros de Corrupción Moral

**Ubicación**: `/assets/data/audio/corruption_config.json`

```json
{
  "tone_shift_max_dB": -8,              // Brillantez máxima reducción: −8 dB
  "bass_boost_max_dB": 6,              // Bajos máxima elevación: +6 dB
  "distortion_max_amount": 0.4,        // Máxima distorsión: 40%
  "reverb_decay_min_ms": 800,          // Reverb mínimo (corrupción 0): 800ms
  "reverb_decay_max_ms": 3000,         // Reverb máximo (corrupción 1): 3000ms
  "reverb_wet_min_dB": -48,            // Reverb "off" (corrupción 0): −48 dB (ultra-silencio, implementable en AudioEffectReverb)
  "reverb_wet_max_dB": -6,             // Reverb máximo (corrupción 1): −6 dB (50% wet)
  "corruption_threshold_act1": 0.3,
  "corruption_threshold_act2": 0.6,
  "corruption_threshold_act3": 1.0
}
```

**Notas**: 
- Distorsión >0.5 hace audio "ilegible" (demasiado corrupto)
- Reverb >3s suena "bajo agua" (poco natural)
- Thresholds deben alinearse con los actos del juego (Estado del Mundo / Progresión Narrativa)

### 7.5 Parámetros de Modificadores de Demonios

**Ubicación**: `/assets/data/audio/demon_modifiers.json`

```json
{
  "demon_modifiers": {
    "fuego": {
      "intensity_multiplier": 1.0,
      "tone_tint": "warm",
      "signature_effect": "roar_crackle",
      "music_variant_asset": "fuego"    // crossfade a mus_[contexto]_fuego.ogg (variante pre-autorada, tonalidad menor)
    },
    "hielo": {
      "intensity_multiplier": 1.0,
      "tone_tint": "cold",
      "signature_effect": "shrill_crystalline",
      "music_variant_asset": "hielo"    // crossfade a mus_[contexto]_hielo.ogg (variante pre-autorada, tonos sintéticos)
    },
    "arcano": {
      "intensity_multiplier": 1.0,
      "additive_boost_linear": 0.25,    // +25% amplitud lineal (≈ +2 dB) aplicado a todos los demás SFX equipados
      "signature_effect": "dimensional_glitch",
      "music_variant_asset": "discordant"  // sufijo para crossfade: mus_[contexto]_arcano.ogg
    },
    "gato": {
      "intensity_multiplier": 2.0,      // Dobla intensidad
      "signature_effect": "demonic_howl",
      "always_audible": true
    },
    "vision": {
      "signature_effect": "radio_static",
      "music_variant_asset": "vision"    // crossfade a mus_[contexto]_vision.ogg (variante pre-autorada, cadencias oníricas)
    },
    "mente": {
      "signature_effect": "binaural_pulses",
      "music_variant_asset": "mente"     // crossfade a mus_[contexto]_mente.ogg (variante pre-autorada, ritmo hipnótico)
    },
    "dash": {
      "signature_effect": "wind_sonic_boom",
      "music_variant_asset": "dash"      // crossfade a mus_[contexto]_dash.ogg (variante pre-renderizada a tempo 1.15x — NO DSP en tiempo real)
    }
  }
}
```

**Rangos de Ajuste**: Multiplicadores pueden variar ±0.5 (ej: Gato 2.0 puede ir a 1.5–2.5). Cambios mayores requieren re-balancing de todas las sinergias.

### 7.6 Parámetros de Impacto (Damage-Based SFX)

**Ubicación**: `/assets/data/audio/impact_config.json`

```json
{
  "impact_pitch_shift_per_100_damage": 0.5,  // +0.5 pitch por cada 100% daño
  "impact_volume_base_dB": -12,
  "impact_volume_per_intensity_dB": 0.15,
  "impact_max_concurrent": 6,
  "impact_priority_by_damage": true   // Priorizar por daño, no por orden
}
```

**Ejemplo**: 20 daño en Edrick (75 HP max):
- `intensity = 26.7%`
- `pitch_shift = 1.0 + (26.7 / 200) = 1.134`
- `volume = −12 + (26.7 × 0.15) = −8 dB`

**Ajuste**: Si los impactos suenan "todos iguales", aumentar `pitch_shift_per_100_damage` a 0.6–0.7. Si suenan "demasiado variados", reducir a 0.3.

### 7.7 Parámetros Misceláneos

**Ubicación**: `/assets/data/audio/misc_config.json`

```json
{
  "master_volume_dB": 0,
  "music_mix_percentage": 40,      // Música es 40% de la mezcla final
  "sfx_mix_percentage": 35,        // SFX es 35%
  "dialogue_mix_percentage": 25,   // Diálogos es 25%
  "compressor_ratio": 2.0,         // Compresor suave para evitar picos
  "limiter_threshold_dB": 0,       // Hard limiter a 0 dB para proteger oído
  "enable_binaural_audio": false,  // Falso por defecto (complejo, costo render)
  "enable_dynamic_eq": true,       // Verdadero: EQ responde a contexto
  "max_simultaneous_sounds": 32    // Límite hard del motor de audio
}
```

### 7.8 Valores De Equilibrio Por Región

**Ubicación**: `/assets/data/audio/region_audio_config.json`

Cada reino tiene su propia **"firma sonora"** — música, SFX ambientales, reverb característico:

```json
{
  "regions": {
    "kingdom_1_grasslands": {
      "ambient_music_asset": "mus_grasslands_loop_8bar",
      "ambient_sfx_assets": ["sfx_wind_light", "sfx_birds_chirping", "sfx_grass_rustle"],
      "ambient_reverb_type": "open_outdoor",
      "ambient_reverb_decay_ms": 400,
      "danger_level": 0.2
    },
    "kingdom_5_abyss": {
      "ambient_music_asset": "mus_abyss_loop_12bar",
      "ambient_sfx_assets": ["sfx_wind_howling", "sfx_chains", "sfx_distant_screams"],
      "ambient_reverb_type": "deep_cavern",
      "ambient_reverb_decay_ms": 2000,
      "danger_level": 0.8
    }
  }
}
```

**Nota**: Cada región debe tener identidad sonora distintiva. Los jugadores deben **saber dónde están por el audio solo** sin ver la pantalla.

---

## 8. Criterios de Aceptación

### 8.1 Reproducción Básica de Audio

**CA-001**: Sistema de audio carga en startup. PASS: `AudioServer.get_bus_count() == 4`, los 4 buses existen con nombres "Música", "SFX_Combate", "SFX_Ambiental", "Diálogos", y 0 entradas de nivel ERROR en el log de Godot que contengan "AudioServer" dentro de los primeros 2 segundos de startup. Verificable con test GUT que inicializa AudioManager y llama `assert(AudioServer.get_bus_count() == 4)`.

**CA-002**: Música ambiental de región comienza a loopear cuando Edrick entra en exploración. Verifica: loop sin chasquidos, duración correcta, volumen en base_level_dB.

**CA-003**: Música de combate comienza cuando entra en combate (primer enemigo visible en rango). Fade de exploración a combate es 0.3s o menor.

**CA-004**: SFX de habilidad de demonio activa cuando habilidad se usa. Verifica: correcto demonio, correcto SFX, correcto timing (<50ms latencia).

**CA-005**: VO de NPC se activa cuando diálogo comienza. Verifica: correcto personaje, correcto línea, sincronización de labios (si aplica).

### 8.2 Ducking y Mezcla Dinámica

**CA-006**: En combate, música se atenúa a −6 dB (máximo) cuando impactos ocurren. Con 0 impactos, música vuelve a base_level. Con 3+ impactos simultáneos, ducking es máximo.

**CA-007**: Durante diálogo, música se atenúa a −9 dB y SFX ambiental a −18 dB. Cuando diálogo termina, volúmenes retoman a base levels en <0.5s.

**CA-008**: Gato nunca cae debajo de −3 dB incluso durante combate intenso. Verifica: vocalizaciones del Gato siempre claramente audibles.

**CA-009**: Pausa del juego pausa música/SFX (congela playhead). Resume continúa desde donde estaba (sin chasquidos, sin desfase).

**CA-010**: Pausa durante cinemática NO pausa música (diseño: jugador quiere escuchar narrativa).

### 8.3 Modificadores de Demonios

**CA-011**: Cuando Fuego se equipa, SFX de combate adquieren signature "roar_crackle" y música toma tonalidad menor.

**CA-012**: Cuando Hielo se equipa, SFX adquieren "shrill_crystalline" y música toma tonos sintéticos.

**CA-013**: Cuando Arcano se equipa, todos los SFX suenan 25% más intensos (volumen +25% dB en outputs posteriores).

**CA-014**: Cuando Gato se equipa, todas las habilidades suenan 2× más intensas (multiplicador 2.0). Verifica: golpes más fuertes, habilidades más dramáticas.

**CA-015**: Gato + Fuego + Arcano equipados: habilidades suenan 2.5× más intensas (1.0 × 2.0 Gato × 1.25 Arcano).

**CA-016**: Triple Dash + Fuego + Hielo: ambas estelas sonoras presentes pero cada una a 85% efectividad (Anulación Térmica activada).

**CA-017**: Desequipar demonio elimina sus modificadores para FUTUROS eventos. SFX en progreso (ej: sonido de quemadura de 3s) continúa sin cambios si demonio se desequipa a mitad.

### 8.4 Filtro de Corrupción Moral

**CA-018**: A corrupción 0.0 (limpio): música mayor, brillante. SFX ambiental "natural" (aves, viento limpio).

**CA-019**: A corrupción 0.3 (Acto 1): acordes menores comienzan a entrar. Tone shift −2.4 dB (reducción de brillantez). Bass boost +1.8 dB.

**CA-020**: A corrupción 0.6 (Acto 2): disonancias claras en música. Tone shift −4.8 dB. Distorsión 24%. Reverb decay 1920ms.

**CA-021**: A corrupción 1.0 (Acto 3): música distorsionada, reverb denso (3000ms, −6 dB wet). Voz de Edrick siniestra. Audio casi irreconocible.

**CA-022**: Transiciones de corrupción son suaves (no saltos discretos). Cambio de +0.10 corrupción durante combate: filtro se re-aplica en <200ms, sin jarring.

**CA-023**: Cargar partida con corrupción guardada: audio retoma con filtro correcto aplicado (no comienza limpio luego filtra).

### 8.5 Transiciones Entre Estados

**CA-024**: Exploración → Combate: fade 0.3s (rápido, sorpresa). Música cambia de ambiental a combate.

**CA-025**: Combate → Exploración: fade 1.0s (lento, catársis). Música retorna a ambiental del región.

**CA-026**: Cualquier estado → Cinemática: fade 1.5s–2.5s (ceremoniosa). Música narrativa carga.

**CA-027**: Transición rápida (Combate → Exploración → Diálogo en <3s): máquina de estados respeta fades programados (no "salta", mantiene coherencia).

**CA-028**: Cambio de región durante transición: la nueva región toma prioridad si entra DESPUÉS de que transición actual complete.

### 8.6 Escalado Basado en Daño

**CA-029**: Golpe de 1 daño (1.3% de 75 HP): impact_volume = −12 + (1.3 × 0.15) = −11.8 dB (muy suave).

**CA-030**: Golpe de 20 daño (26.7%): impact_volume = −12 + (26.7 × 0.15) = −8 dB. Pitch shift = 1.134 (+13.4%). Audiblemente diferente del golpe de 1.

**CA-031**: Golpe de 50 daño (66.7%, máximo razonable): impact_volume = −12 + (66.7 × 0.15) = −2 dB. Pitch shift = 1.33 (+33%). SFX dramático.

**CA-032**: Más de 6 impactos simultáneos: sistema limita a los 6 más fuertes (ordenados por daño). Demás se silencian (no overlaps).

### 8.7 Integración Con Otros Sistemas

**CA-033**: Evento `ability_used(Fuego, quemar)` recibido: Audio activa SFX de quemar (loop de 3s) con modificadores Fuego aplicados.

**CA-034**: Evento `demon_equipped(Hielo)` recibido: Audio almacena localmente y aplica modificadores a próximos SFX de combate.

**CA-035**: Evento `corruption_level_changed(0.7)` recibido: Audio actualiza filtro de corrupción en <200ms. Próximas capas de música/SFX usan nuevo filtro.

**CA-036**: Evento `area_entered(kingdom_3_mountains)` recibido: Audio fade a música ambiental de kingdom_3 (1.0s), carga SFX ambientales de montaña.

**CA-037**: Evento `dialogue_started(npc_blacksmith)` recibido: Audio atenúa música (−9 dB), prepara bus de diálogos, espera `vo_line_play`.

**CA-038**: Evento `cinematic_started(finale)` recibido: Audio transiciona a narrativa (1.5s), carga música "finale.ogg", establece VO como máxima prioridad.

**CA-039**: `game_paused()` evento: Música pausa (playhead congelado). `game_resumed()`: retoma desde playhead exacto.

### 8.8 Sincronización y Latenicia

**CA-040**: Evento de combate → SFX audible en <50ms (ejecución en frame, reproducción comienza en frame siguiente).

**CA-041**: Transición de estado respeta fade_duration (0.3s explícita para combate; no comprimida aunque múltiples eventos se disparen).

**CA-042**: Filtro de corrupción se aplica suavemente a lo largo de fade de transición (si transición es 1.0s, filtro también cambia a lo largo de esa duración).

**CA-043**: Después de cargar partida (`save_loaded` evento): audio state sincronizado con Estado del Mundo en <100ms (sin desfase audible).

### 8.9 Edge Cases y Robustez

**CA-044**: Demonio desequipado mientras habilidad suena: habilidad continúa hasta completarse; NO se corta.

**CA-045**: NPC hablando mientras combate cercano ocurre: NPC dialog toma prioridad (música −9 dB), combate lejano atenuado (−15 dB). Ambos audibles, NPC dominante.

**CA-046**: Múltiple diálogos simultáneamente (raro): sistema prioriza por proximidad. Diálogo más cercano a −1 dB; diálogos lejanos atenuados a −18 dB.

**CA-047**: Audio completamente desactivado por usuario: sistema continúa calculando pero volumen output = 0. Cuando reactiva: audio retoma sin desfase.

**CA-048**: Cinemática con música narrativa que no loopea dura más de la música: música fade-out los últimos 2s. Diálogos continúan audibles.

**CA-049**: Cargar partida en mitad de cinemática: audio retoma estado exacto (música en playhead correcto, diálogos en sincronización correcta).

### 8.10 Aseguramiento de Calidad Auditiva

**CA-050**: Prueba de carga: 32 sonidos simultáneos sin crash. PASS: test GUT `test_audio_load_32_simultaneous()` llama `AudioManager.play_sfx()` 32 veces en un mismo frame con clips distintos de 1 segundo. Criterios: (a) no crash, (b) 0 entradas de ERROR en log con "AudioStreamPlayer", (c) después de la llamada, `AudioManager.get_active_sound_count() == 32`. Si se llama un 33° sonido, verificar que el de menor prioridad tiene volumen == 0 (muted, no stopped).

**CA-051**: Prueba de memoria: no hay crecimiento de nodos de audio durante uso continuo. PASS: test GUT que itera 60 veces `region_entered` + `region_exited` (simulating 60 region changes), luego verifica que `get_tree().get_nodes_in_group("audio_players").size()` no creció respecto al inicio (delta == 0, ±2 por buffers transitorios). Herramienta: Godot Remote Debugger → Performance Monitor → "Audio/AudioStreamPlayers" debe permanecer estable tras los 60 cambios.

**CA-052**: Prueba de frecuencias: espectrograma de audio muestra distribución balanceada (no "tin-y", no muddy). Rango 20Hz–20kHz presente.

**CA-053**: Prueba de accesibilidad: Subtítulos activos cuando audio desactivado. VO en diálogos es claro y legible.

**CA-054**: Prueba de compatibilidad de plataformas: Audio reproduce correctamente en PC (WASAPI/PulseAudio), sin latencia anormal.
