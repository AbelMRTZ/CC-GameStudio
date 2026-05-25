# Cross-GDD Review Report — Demons of Dravaryn

**Fecha**: 2026-05-26
**Skill**: /review-all-gdds
**GDDs Revisados**: 6 (Movimiento, Salud/Daño, Base de Datos de Demonios, Estado del Mundo, Audio, Combate)
**Agentes consultados**: systems-designer (Phase 2 — Consistency), game-designer (Phase 3 — Design Theory)
**Verdict**: CONCERNS

---

## Resumen Ejecutivo

| Tipo | Conteo |
|------|--------|
| 🔴 Bloqueantes diseño | 2 |
| ⚠️ Warnings consistencia | 6 (1 crítica: W-05 Arcano) |
| ⚠️ Warnings diseño | 6 |
| ℹ️ Info no actionable | 6 |

**Hilo conductor**: 3 de los issues más críticos (B1, B2, W-05) están relacionados — todos giran alrededor de **Arcano y el sistema de Corrupción**. Resolverlos juntos es más eficiente que de a uno.

---

## 🔴 Issues Bloqueantes (decisiones de diseño requeridas)

### 🔴 B1 — Arcano es objetivamente dominante en cualquier loadout
**Check**: 3c (Dominant Strategy) | **Pilar afectado**: 4 (Sinergia y Libertad de Construcción) | **GDDs**: #3, #6

Arcano añade +0.25 daño + amplifica todos los demás demonios sin coste alguno (sin HP penalty, sin cooldown, sin resistencia negativa). Cualquier build sin Arcano es estrictamente inferior.

**Evidencia numérica**:
- Fuego solo: `mod_atacante = 0.20`
- Fuego+Arcano: `mod_atacante = 0.45` (+125% mejor)
- Triple Dash+Fuego+Arcano: `mod_atacante = 0.6725` (+67% en cada Ligero)

**Violación de Pilar 4**: Pilar 4 dice "algunas combinaciones son objetivamente superiores, otras son trampas". Pero no hay trampas reales — solo "Arcano o subóptimo". Esto colapsa la libertad de build.

**Mitigación parcial actual**: La sinergia negativa Arcano+Corrupción invierte la amplificación. Pero Corrupción NO es uno de los 7 demonios MVP, así que no aplica.

**Opciones de resolución**:
- (a) HP penalty: Arcano cuesta −10 a −15 HP (tradeoff vs Hielo +10)
- (b) Arcano NO se amplifica a sí mismo (reduce self-stacking de su propio +0.25)
- (c) Arcano amplifica también las **vulnerabilidades** de Edrick (no solo lo positivo): si Edrick tiene +0.10 vulnerabilidad Fuego, con Arcano pasa a +0.125

---

### 🔴 B2 — Pilar 5 desconectado del combate: ninguna habilidad MVP genera Corrupción
**Check**: 3f, 3g | **Pilar afectado**: 5 (Transformación Moral de Edrick) | **GDDs**: #3, #4, #6

GDD #6 emite `corruption_damaged(amount)` "solo para ciertas habilidades demoníacas — definidas en Base de Datos de Demonios". Pero GDD #3 **NO asigna Corrupción a ninguno de los 7 demonios MVP**.

**Resultado**: En todo el loop de combate MVP, NINGUNA habilidad del jugador genera corrupción. Las únicas fuentes en GDD #4 son eventos narrativos (matar NPC, abandonar aliado, mentir).

**Sistema circular**: el audio se ajusta al nivel de corrupción, pero el combate nunca corrompe. Edrick puede ser un "arma demoníaca" sin que sus actos demoníacos lo cambien.

**Violación de Pilar 2**: "Los demonios no son items — son cambios fundamentales con beneficios Y costos (mecánicos o narrativos)". Actualmente: 0 demonios MVP tienen coste moral en combate.

**Opciones de resolución**:
- (a) Fuego: +0.01-0.02 corrupción por kill por quemadura (acumulativo)
- (b) Mente "Predecir Movimiento": +0.02 corrupción por uso (precognición demoníaca tiene peso moral)
- (c) Arcano: +0.01 corrupción por habilidad amplificada (usar poder arcano corrompe)

---

## ⚠️ Warnings de Consistencia

### W-01 — Bidireccionalidad: GDD #4 no lista GDD #6 como dependiente
**GDDs**: #4, #6 | **Severidad**: Media

GDD #6 declara que emite señal `corruption_damaged(amount)` a Estado del Mundo. Pero GDD #4 Sección 6.1 lista 8 dependientes — **Combate en Tiempo Real no está**. Estado del Mundo no sabe que debe subscribirse a esta señal.

**Fix**: Añadir Combate en Tiempo Real a la lista de dependientes de GDD #4 y documentar la subscripción a `corruption_damaged`.

---

### W-02 — HP recalculation ownership undefined
**GDDs**: #2, #3 | **Severidad**: Alta

Cuando se equipa Visión (−5 HP) y HP_actual > nuevo HP_máximo, GDD #3 dice "se reduce automáticamente" pero **no asigna sistema propietario** de esa recalculación. GDD #2 solo aplica el clamp en eventos de daño, no en cambios de stat. La probable solución es que Loadout (GDD #10, no escrito aún) sea el propietario.

**Fix**: Documentar explícitamente en GDD #3 que la recalculación es propiedad de Loadout & Build Management (GDD #10), o si Loadout no existe aún, especificar en GDD #2 una función `recalculate_hp_max()` que pueda ser llamada por cualquier sistema que cambie stats.

---

### W-03 — Damage type taxonomy: capitalización inconsistente
**GDDs**: #2, #3 | **Severidad**: Media

GDD #2 capitaliza (`Corrupción`, `Físico`); GDD #3 usa lowercase en JSON schemas (`corrupción`, `físico`). Si los tipos se usan como string keys en código, hay riesgo de lookup failures silenciosos.

**Fix**: Estandarizar a una convención (recomendación: lowercase para keys de datos/JSON, capitalizado para narrativa/UI) y documentarlo en GDD #2 como ground-truth.

---

### W-04 — Dash cooldown triple-ownership
**GDDs**: #1, #3, #6 | **Severidad**: Alta

Tres documentos definen knobs sobre el mismo valor:
- GDD #1: `dash_cooldown = 0.6s`
- GDD #3: `Dash Attack | 0.6 segundos | 0.3-1.0s` (knob de demonio)
- GDD #6: `cooldown_global_scale = 1.0x` (multiplicador global)

GDD #3 dice "Dash ability cooldown es defined in GDD Movimiento" pero luego re-define el mismo knob.

**Fix**: Eliminar el knob redundante de GDD #3 Sección 7.4 (Dash debe usar el valor de GDD #1). Documentar claramente en GDD #6 que `cooldown_global_scale` afecta TODAS las cooldowns incluyendo Dash.

---

### W-05 — **CRÍTICA**: Arcano ¿aditivo o multiplicativo?
**GDDs**: #3, #6 | **Severidad**: Crítica

GDD #3 Sección 4.2 describe Arcano como ×1.25 **multiplicativo** de TODOS los efectos. GDD #6 Formula 4.1 lo usa como +0.25 **aditivo** al pool de modifiers.

Estos producen resultados numéricos distintos:
- Multiplicativo (GDD #3): Fuego+Arcano = `0.20 × 1.25 = 0.25`
- Aditivo (GDD #6): Fuego+Arcano = `0.20 + 0.25 = 0.45`

Es **incompatible** — ambos modelos no pueden coexistir.

**Decisión necesaria**: ¿Cuál es el modelo oficial?

---

### W-06 — HIT_STUN duration implícitamente = IFRAME_DURATION
**GDD**: #6 | **Severidad**: Baja

El acoplamiento entre HIT_STUN duration e IFRAME_DURATION no está documentado pero es estructural. Si un tuner cambia IFRAME_DURATION, HIT_STUN cambia silenciosamente con él.

**Fix**: Documentar explícitamente en Sección 3.4 de GDD #6: "HIT_STUN duration == IFRAME_DURATION; comparten timer".

---

## ⚠️ Warnings de Diseño

### D-W1 — Cognitive load: 7-8 sistemas activos simultáneamente
**Pilar**: Anti-pilar (NOT souls-like) | **GDDs**: #1, #3, #6

Durante un combate típico MVP (Edrick vs 2 enemigos, 3 demonios), el jugador maneja simultáneamente: Movement (4 inputs), Ligero/Pesado timing, 3 demon ability slots, 3 cooldown timers, HP management, I-frames awareness, audio cues. Eso es 7-8 sistemas activos vs el target de 3-4.

**Recomendación**: Primer reino cap inicial de slots de demonios a 2 para reducir cooldown tracking. Añadir indicador pasivo visual de cooldowns en el icono del demonio.

---

### D-W2 — Loop de progresión fragmentado
**Pilar**: 2, 3, 5 | **GDDs**: #3, #4, game-concept

Sesiones de 15 min sin demon binding no tienen señal tangible de "crecí". Demon binding es narrativo (raro), no exploración (loot). Hollow Knight tiene geo, Hades tiene dark crystal — Edrick no tiene equivalente.

**Recomendación**: Definir reward meso-loop entre binding events (NPC reputation visible, Bestiario lore, dialogue corrupción-driven). Resolver en Progresión Narrativa GDD antes de Vertical Slice.

---

### D-W3 — Dash+Fuego "Estela Ardiente" sin tradeoff
**Pilar**: 4 | **GDDs**: #3, #6

Cada 0.6s: damage + reposicionamiento + I-frames + drop burn zone. Sin penalización. Combinación cercana a dominante.

**Recomendación**: Documentar como conocido near-dominant para balance pass. Considerar cooldown separado para estelas (no compartido con dash).

---

### D-W4 — Identity vacuum pre-demonio
**Pilar**: 2 | **GDDs**: #1, #2, #6

GDD #6 combate técnico (startup/recovery timing, cancels) contradice narrativa "Edrick no es soldado entrenado". Sin demonios, Edrick ya tiene rotación funcional — el "antes" no se diferencia del "después" lo suficiente.

**Recomendación**: Vinculación de Demonios (GDD #13) debe diseñar para contraste explícito con baseline pre-demonio. Primera binding event debe ser un punto de inflexión claro.

---

### D-W5 — Respawn = HP completo: HP no es resource sink
**Anti-pilar**: NOT souls-like | **GDDs**: #2, #4

Muerte → checkpoint → HP completo sin coste. Eso es intencional (no souls-like) pero significa que HP solo importa entre checkpoints. Audio depende de presión de HP para feedback (formula 4.2 GDD #5) — devaluado si jugadores die-to-heal.

**Recomendación**: Level Design GDD debe especificar spacing mínimo de checkpoints. Flag para level-designer.

---

### D-W6 — Corrupción reversible permite min-max
**Pilar**: 5 | **GDD**: #4

3 ejecuciones (+0.30) + 3 salvaciones (−0.30) = mismo estado visual. Permite redemption grinding.

**Recomendación**: Considerar "moral history" floor: cada acto oscuro deja residual mínimo no redimible (e.g., +0.02 floor por ejecución). Resolver en Seguimiento Moral GDD (#22).

---

## ℹ️ Info (no actionable ahora)

- **I-01**: `world_bounds` en GDD #6 CA-044 sin Level/Stage GDD aún (esperado MVP scope)
- **I-02**: GDD #5 hardcodea `HP_max = 75` en formula 4.2; no actualiza con demonios (baja prioridad)
- **I-03**: GDD #4 CA-001 (gato inicial) vs GDD #3 CA-054 (gato post-binding) inconsistencia identificada
- **I-04**: IA de Enemigos ausente: curva dificultad no evaluable
- **I-05**: Audio requiere 21+ assets musicales (7 demonios × 3 contextos) — planificar producción
- **I-06**: Synergy Engine (#11), Loadout (#10) downstream pero no autorizados

---

## GDDs Flagged for Revision

| GDD | Razón | Tipo | Prioridad |
|-----|-------|------|-----------|
| base-datos-demonios.md | B1 (Arcano dominante), B2 (sin corrupción), W-05 (modelo Arcano) | Diseño + Consistencia | Bloqueante |
| combate-tiempo-real.md | B2, W-04 (dash ownership), W-05 (Arcano modelo) | Diseño + Consistencia | Bloqueante |
| estado-del-mundo.md | W-01 (bidireccionalidad), D-W6 (corruption floor) | Consistencia + Diseño | Alta |
| salud-daño.md | W-02 (HP recalc owner), W-03 (taxonomía) | Consistencia | Media |
| movimiento-fisicas-2d.md | W-04 (dash ownership) | Consistencia | Media |
| sistema-audio.md | I-02 (HP_max hardcoded), W-03 (taxonomía) | Consistencia | Baja |

---

## Próximos pasos recomendados

1. **Esta sesión**: Resolver B1, B2, W-05 (las 3 críticas relacionadas con Arcano y Corrupción)
2. **Próxima sesión balance**: Resolver W-02, W-03, W-04, W-06 + D-W1, D-W3
3. **Antes de Vertical Slice**: D-W2 (meso-loop), D-W6 (corruption floor)
4. **Continuar GDDs**: IA de Enemigos (#7), Loadout (#10), Sinergias (#11) son prerrequisitos para resolver W-02 y D-W3
