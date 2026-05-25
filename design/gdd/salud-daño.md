# GDD: Salud y Daño

> **Estado**: Aprobado
> **Creado**: 2026-05-24
> **Última Actualización**: 2026-05-24
> **Aprobación**: Completado en sesión de diseño
> **Sistema**: Salud y Daño
> **Milestone**: MVP — Foundation Layer
> **Depende de**: — (sin dependencias)
> **Dependen de este sistema**: Combate en Tiempo Real, IA de Enemigos

---

## 1. Visión General

El sistema de Salud y Daño define cómo Edrick y los enemigos sufren daño y mueren. Edrick comienza con una reserva de puntos de vida (HP) que disminuyen cuando recibe daño de enemigos, trampas u otras fuentes. El daño se categoriza en múltiples tipos (Físico, Fuego, Hielo, Arcano, Corrupción), cada uno con su propia mecánica y resistencias. Los demonios vinculados pueden otorgar resistencias o modificadores de daño recibido, afectando cómo Edrick responde a distintos tipos de ataques. Cuando Edrick llega a 0 HP, se dispara una cutscena de muerte narrativa, seguida de un reload al último checkpoint. El sistema de daño se calcula mediante fórmulas transparentes (atacante, defensor, tipo, modificadores), permitiendo balance predictible y ajuste de dificultad claro.

---

## 2. Fantasy del Jugador

Cuando Edrick toma daño, el jugador debería sentir: **consecuencia real** — cada golpe importa, no hay infinitismo. **Amenaza visceral** — la barra de HP (o indicador visual) comunica que está en peligro; cuando baja, la tensión sube. **Inteligencia de build** — si Edrick equipó demonios que dan resistencia al fuego, un ataque de fuego debería sentirse menos amenazante, recompensando la elección de build. **Muerte cinematográfica** — cuando cae a 0 HP, no es un fade to black genérico; es un momento narrativo que refuerza que sus actos tienen peso. **Checkpoint sensible** — el respawn está cerca pero no trivial; el jugador siente que fracasó pero puede intentar de nuevo rápidamente.

---

## 3. Reglas Detalladas

### 3.1 Salud Base de Edrick

Edrick comienza con **75 puntos de vida (HP)**:
- Este es el valor base, sin modificadores
- Los demonios pueden modificar HP máximo (aumentar o disminuir)
- HP nunca puede estar por debajo de 1 (no es posible tener 0 HP vivo; 0 HP = muerte)
- HP nunca puede ser negativo (se clampea en 0 cuando se calcula)
- No hay regeneración pasiva durante exploración (no en MVP)

### 3.2 Tipos de Daño

El daño se categoriza en **5 tipos**:

| Tipo | Descripción | Ejemplo de Fuente | Resistencia Típica |
|------|-------------|-------------------|-------------------|
| **Físico** | Daño de armas, golpes, caídas | Enemigo con espada, daño de caída | Armadura/Items especiales, Demonios |
| **Fuego** | Daño térmico, quemaduras | Enemigo con fuego, trampas de llama | Items/Armaduras de fuego, Demonio resist. fuego |
| **Hielo** | Daño congelante, ralentización | Enemigo con hechizo hielo | Items/Armaduras de hielo, Demonio resist. hielo |
| **Arcano** | Daño mágico/sobrenatural | Enemigo arcano, trampas mágicas | Items/Armaduras arcanas, Demonio resist. arcano |
| **Corrupción** | Daño relacionado con la corrupción demoníaca | Enemigo corrupto (portador de demonio) | Items/Armaduras anti-corrupción, Demonio con afinidad contraria |

**Nota importante:** La **Corrupción** es un tipo de daño especial relacionado con la narrativa de Edrick. Los enemigos portadores de demonios (raros, ~2-5%) infligen daño por Corrupción. Este daño refleja la lucha interna de Edrick con su propia corrupción.

**Items y Armaduras:** El jugador puede encontrar **items especiales y piezas de armadura** mediante la exploración del mundo. Estos items otorgan resistencias a tipos de daño específicos (ej: "Brazales de Fuego +20% resist. Fuego"). Las resistencias de items se suman a las de demonios de forma aditiva, permitiendo que el jugador personalice su defensa según el build y el entorno.

### 3.3 Cálculo Base de Daño

El daño se calcula mediante esta fórmula:

```
daño_final = round(daño_base × (1 + modificadores) × (1 + resistencia_defensiva))

Donde:
  daño_base = daño inherente del ataque
  modificadores = bonificadores del atacante (ej: +0.30 = +30% daño)
  resistencia_defensiva = delta de resistencia del defensor ∈ [-0.5, +0.5]
                          (ej: -0.10 = recibe 10% menos daño)
```

**Ejemplo:**
- Enemigo ataca con 15 daño Físico
- Edrick tiene resistencia_defensiva = -0.10 al Físico (10% menos daño)
- daño_final = round(15 × 1.0 × (1 + (-0.10))) = round(15 × 0.9) = round(13.5) = 14 daño

### 3.4 Resistencias de Edrick

Edrick puede tener resistencias o vulnerabilidades a tipos de daño:

- **Rango de resistencia:** -50% a +50% del daño
  - -50% = recibe 50% menos daño (resistencia máxima)
  - 0% = sin modificación
  - +50% = recibe 50% más daño (vulnerabilidad máxima)

- **Resistencia base (sin demonios ni items):**
  - Físico: 0% (neutral)
  - Fuego: 0%
  - Hielo: 0%
  - Arcano: 0%
  - Corrupción: -10% (Edrick tiene resistencia innata, es medio-corrupto)

- **Los demonios pueden modificar resistencias:**
  - Demonio A: +20% resistencia al Fuego (acumula: 20%)
  - Demonio B: -15% resistencia al Hielo (acumula: -15%)
  - Si ambos están equipados: Fuego 20%, Hielo -15%, otros 0%

- **Los items pueden modificar resistencias:**
  - Item: "Brazales de Fuego" +20% resist. Fuego
  - Si Edrick lleva brazales + demonio +20% fuego: Fuego total = 40%
  - Las resistencias se suman aditivamente: demonios + items

### 3.5 Aplicar Daño a Edrick

Cuando Edrick recibe daño:
1. Calcular daño_final usando fórmula de 3.3
2. Restar daño_final de HP actual
3. Si HP ≤ 0 → HP = 0, dispara evento "edrick_died"
4. Si HP > 0 → animar cambio de HP en HUD

**Evento de daño:**
- Se emite una señal (signal) `health_changed(nuevo_hp, daño_recibido, tipo_daño)`
- El HUD escucha y actualiza visualmente
- Efectos de sonido/VFX se disparan desde el GDD de Audio/VFX (fuera de este sistema)

### 3.6 Muerte de Edrick

Cuando Edrick llega a 0 HP:

1. **Estado de muerte activado:** Edrick entra en estado "dead"
2. **Input bloqueado:** El jugador no puede controlar a Edrick
3. **Cutscena narrativa:** Se dispara la cinemática de muerte (definida en GDD de Cinemáticas)
4. **Duración:** La cutscena dura ~3-5 segundos
5. **Reload:** Después de la cutscena, la escena se recarga desde el último checkpoint guardado
6. **Estado reset:** Edrick reaparece con HP completo (75 HP) en el checkpoint

**Checkpoints:** Se guardan automáticamente al entrar a nuevas áreas o después de jefes derrotados (definido en Estado del Mundo).

### 3.7 Indicador Visual de HP

El sistema de salud emite una señal `health_changed` que el HUD escucha:
- El HUD muestra un indicador (barra, números, otro) del HP actual
- Cuando HP baja, el indicador cambia visualmente (se define en HUD de Combate, no aquí)
- No hay indicador de daño flotante en este GDD (pertenece a VFX)

### 3.8 Enemigos (Referencia)

Los enemigos usan el mismo sistema de salud y daño:
- Cada enemigo tiene su propio HP
- Tipos de daño se aplican igual (pueden tener resistencias diferentes)
- Cuando llegan a 0 HP, mueren (se define en IA de Enemigos)

---

## 4. Fórmulas

### 4.1 Cálculo de Daño Final

```
daño_final = round(daño_base × (1 + mod_atacante) × (1 + resistencia_defensiva))

Donde:
  daño_base = daño del ataque (definido por fuente)
  mod_atacante = modificador de daño del atacante ∈ [-1, +2]
    (ejemplo: +30% = 0.3, -20% = -0.2)
  resistencia_defensiva = resistencia del defensor ∈ [-0.5, 0.5]
    (ejemplo: -40% resist = -0.4, +30% vulnerabilidad = +0.3)
  round() = redondear al entero más cercano
```

**Ejemplo:**
- Enemigo ataca Fuego 18 daño
- Edrick lleva "Brazales de Fuego" +25% resist. Fuego + Demonio +15% resist. Fuego = total -40%
- daño_final = round(18 × (1 + 0) × (1 + (-0.4))) = round(18 × 0.6) = 11 daño

### 4.2 Acumulación de Resistencias

```
resistencia_total = sum(demonio_resistencias) + sum(item_resistencias)
resistencia_final = clamp(resistencia_total, -0.5, 0.5)

Donde:
  sum(demonio_resistencias) = suma de todas las resistencias de demonios
  sum(item_resistencias) = suma de todas las resistencias de items
  clamp(x, min, max) = si x < min, devuelve min; si x > max, devuelve max; sino x
```

**Ejemplo:**
- Demonio A: +20% Fuego
- Demonio B: +15% Fuego
- Item: +10% Fuego
- resistencia_total = 0.2 + 0.15 + 0.1 = 0.45
- resistencia_final = 0.45 (dentro de rango)
- Edrick recibe daño Fuego al 55% (100% - 45% = 55%)

### 4.3 Cambio de HP

```
HP_nuevo = clamp(HP_actual - daño_final, 0, HP_máximo)

Donde:
  HP_actual = HP antes del daño
  daño_final = daño calculado (≥ 0)
  clamp() = garantiza que HP nunca sea < 0 o > máximo
```

### 4.4 Muerte

```
Si HP_nuevo == 0:
  estado = "dead"
  dispara evento: edrick_died()
  activa cutscena: play_death_cinematic()
  después de cutscena: reload_scene()
```

---

## 5. Casos Extremos

### 5.1 Daño Overkill

**Situación**: Edrick tiene 10 HP. Enemigo ataca con 100 daño.

**Comportamiento**:
- daño_final se calcula normalmente: 100 daño (con resistencias aplicadas, digamos 80)
- HP_nuevo = clamp(10 - 80, 0, 75) = clamp(-70, 0, 75) = 0
- Edrick muere con 0 HP
- El daño excedente se descarta (no hay "acumulación" de overkill)

### 5.2 Cambiar Items/Demonios Mientras Toma Daño

**Situación**: Edrick equipado con +40% resist. Fuego. Enemigo ataca Fuego. Mientras el daño se calcula, Edrick cambia de demonio (pierde resist.).

**Comportamiento**:
- El cálculo de daño debe ser instantáneo (ocurre en 1 frame)
- La resistencia se evalúa en el momento en que el daño se aplica
- Si el cambio de demonio ocurre DESPUÉS de que el daño se calcula, la nueva resistencia aplica para ataques futuros
- Orden: Ataque → Calcular daño (con resistencias ACTUALES) → Aplicar daño → Cambiar demonio

### 5.3 Daño Negativo (Curación)

**Situación**: Un item o demonio podría otorgar "curación al ser golpeado" (+5 HP cuando recibe daño) — esto es posible narrativamente.

**Comportamiento**:
- El sistema permite daño_final negativo (curación)
- HP_nuevo = clamp(HP_actual - daño_final, 0, HP_máximo)
- Si daño_final es negativo, resta un número negativo = suma
- Ejemplo: HP_actual = 50, daño_final = -10, HP_nuevo = clamp(50 - (-10), 0, 75) = 60
- **Nota**: No está en MVP, pero el sistema es flexible para permitirlo

### 5.4 Invulnerabilidad Temporal (I-Frames)

**Situación**: Edrick acaba de ser golpeado. ¿Puede ser golpeado de nuevo inmediatamente?

**Comportamiento**:
- Este GDD **no define i-frames** — eso pertenece al GDD de Combate
- Si el sistema de Combate implementa i-frames (ej: 0.5s después de recibir daño), el sistema de Salud simplemente rechaza daño si está en estado "invulnerable"
- Salud y Daño NO aplica daño si `es_invulnerable() == true`

### 5.5 Múltiples Daños Simultáneos

**Situación**: Edrick toca fuego (daño Fuego 15/frame) Y es golpeado por enemigo (daño Físico 20 único). Ambos ocurren en el mismo frame.

**Comportamiento**:
- Cada daño se calcula por separado con su propio tipo y resistencia
- Ambos se restan de HP en el mismo frame:
  - daño_fuego_final = round(15 × (1 + 0) × (1 + resist_fuego))
  - daño_fisico_final = round(20 × (1 + 0) × (1 + resist_fisico))
  - HP_nuevo = clamp(HP - daño_fuego_final - daño_fisico_final, 0, HP_máximo)
- Se emite una sola señal `health_changed` con el daño total, no dos

### 5.6 Resistencia que Excede el Cap

**Situación**: Edrick tiene resist. Fuego -60% (total aditivo: demonio +40% + item +20%).

**Comportamiento**:
- Suma total = 0.6
- Cap máximo = 0.5
- resistencia_final = clamp(0.6, -0.5, 0.5) = 0.5
- Edrick recibe daño Fuego al 50% (tope de resistencia)

### 5.7 Edrick Golpeado a 0 HP Varias Veces

**Situación**: Edrick está a 5 HP. Enemigo A ataca 10 daño. Enemigo B ataca 15 daño. Ambos llegan en frames consecutivos.

**Comportamiento**:
- Frame 1: HP = clamp(5 - 10, 0, 75) = 0 → dispara edrick_died(), cutscena, reload
- Frame 2: Nunca ocurre porque la escena se recargó
- Si por alguna razón Frame 2 ocurriese: HP ya es 0, y el daño se rechaza (HP ya está muerto)

### 5.8 Muerte Mientras Cutscena de Daño

**Situación**: Edrick recibe daño cinematográfico durante una cutscena.

**Comportamiento**:
- Las cutscenas de Cinemáticas tienen control total de Edrick
- El sistema de Salud no debe reducir HP durante cutscenas de narración (definido en Cinemáticas)
- Si el GDD de Cinemáticas QUIERE mostrar daño (ej: Edrick es apuñalado en cutscena), llama manualmente al sistema de Salud
- Salud y Daño es agnóstico: simplemente aplica daño cuando se solicita

---

## 6. Dependencias

### 6.1 Dependencias de Este Sistema

**Salud y Daño** es Foundation Layer — no depende de ningún otro sistema. Es puro cálculo de daño, sin dependencias externas.

### 6.2 Sistemas que Dependen de Este

- **Combate en Tiempo Real** — Define cuándo se aplica daño, integra cálculos de hit/miss, aplica condiciones (stun, knockback)
- **IA de Enemigos** — Los enemigos usan el mismo sistema de salud y daño; su muerte se dispara cuando HP llega a 0
- **Movimiento y Físicas 2D** — Knockback del daño es opcional (definido en Combate, no aquí)

### 6.3 Integración con Otros Sistemas

**Base de Datos de Demonios**: Define qué resistencias otorga cada demonio. Este sistema LAS APLICA, pero la definición vive en ese GDD.

**Loadout & Build Management**: Cuando Edrick cambia demonio, las resistencias se actualizan. Este sistema recibe los nuevos valores.

**Exploración del Mundo**: Los items encontrados son definidos en Exploración, pero sus resistencias se aplican aquí.

---

## 7. Parámetros de Ajuste

Todos estos valores deben vivir en un archivo de configuración (ej: `assets/data/health_config.gd`), no hardcodeados.

| Parámetro | Valor Base | Rango Seguro | Aspecto Afectado | Notas |
|-----------|-----------|--------------|-----------------|--------|
| `hp_inicial` | 75 | 30–200 | Fragilidad/Dureza de Edrick | Mayor = más HP, combate menos inmediato. Menor = muy frágil |
| `daño_enemigo_débil` | 8 | 2–15 | Daño de enemigos menores | Define cuántos golpes necesita un enemigo débil |
| `daño_enemigo_medio` | 15 | 8–30 | Daño de enemigos normales | Enemigos de nivel medio |
| `daño_enemigo_fuerte` | 25 | 15–50 | Daño de enemigos jefe/elite | Jefes y enemigos peligrosos |
| `resist_cap_max` | 0.5 | 0.3–0.7 | Resistencia máxima permitida | Mayor cap = puede resistir más, cambios por build son más notables |
| `resist_cap_min` | -0.5 | -0.7 a -0.3 | Vulnerabilidad máxima permitida | Cap negativo (vulnerabilidad) balancea resistencias |
| `hp_jefe_multiplicador` | 2.5 | 1.5–5.0 | HP de jefes vs enemigos normales | Multiplicador aplicado al HP base (ej: jefe = 75 × 2.5 = 187.5 HP) |
| `daño_cutscena_muerte` | variable | — | Daño en cinemática de muerte | Definido por Cinemáticas, no aquí |

**Fichero de configuración recomendado**: `assets/data/health_config.gd` (Resource o script con constantes exportables).

---

## 8. Criterios de Aceptación

### 8.1 Sistema de Salud Base

- [ ] **AC 1.1**: Edrick comienza con 75 HP. Verificar: al iniciar el nivel, HP = 75.
- [ ] **AC 1.2**: HP nunca es negativo. Verificar: aplicar 1000 daño a alguien con 10 HP → HP = 0, no -990.
- [ ] **AC 1.3**: HP nunca excede el máximo. Verificar: equipar demonio que da +100 HP máx, HP total = 175, no más.
- [ ] **AC 1.4**: Signal `health_changed` se emite cuando HP cambia. Verificar: recibir daño → señal se emite con nuevo HP.

### 8.2 Tipos de Daño

- [ ] **AC 2.1**: Existen 5 tipos de daño (Físico, Fuego, Hielo, Arcano, Corrupción). Verificar: atacar con cada tipo → se registra correctamente.
- [ ] **AC 2.2**: Daño Físico ocurre. Verificar: enemigo con espada ataca → daño Físico aplica.
- [ ] **AC 2.3**: Daño Fuego ocurre. Verificar: trampa de llama → daño Fuego aplica.
- [ ] **AC 2.4**: Daño Hielo ocurre. Verificar: enemigo arcano hielo → daño Hielo aplica.
- [ ] **AC 2.5**: Daño Arcano ocurre. Verificar: trampa arcana → daño Arcano aplica.
- [ ] **AC 2.6**: Daño Corrupción ocurre. Verificar: enemigo portador de demonio → daño Corrupción aplica.

### 8.3 Cálculo de Daño

- [ ] **AC 3.1**: Daño se calcula correctamente sin resistencias. Verificar: 20 daño base × 1.0 (sin modificadores) × (1 + 0 resist) = 20 daño aplicado.
- [ ] **AC 3.2**: Modificadores del atacante afectan daño. Verificar: enemigo con +30% daño ataca 20 base → 26 daño aplicado.
- [ ] **AC 3.3**: Resistencias del defensor reducen daño. Verificar: Edrick con -40% resist. Fuego recibe 20 Fuego → round(20 × 0.6) = 12 daño.
- [ ] **AC 3.4**: Daño se redondea correctamente. Verificar: 13.5 daño → 14 daño (round to nearest).

### 8.4 Resistencias y Items

- [ ] **AC 4.1**: Demonio con +20% resist. Fuego reduce daño Fuego. Verificar: con demonio, daño Fuego 20 → round(20 × 0.8) = 16 daño.
- [ ] **AC 4.2**: Item con +15% resist. Hielo reduce daño Hielo. Verificar: con item, daño Hielo 25 → round(25 × 0.85) = 21 daño.
- [ ] **AC 4.3**: Resistencias se apilan aditivamente. Verificar: +20% demonio + 15% item = -35% resist total, aplicado correctamente.
- [ ] **AC 4.4**: Resistencia capped en max (0.5). Verificar: +60% resist. total → clamp a 0.5 → recibe daño al 50%.
- [ ] **AC 4.5**: Vulnerabilidad capped en min (-0.5). Verificar: +60% vulnerabilidad → clamp a -0.5 → recibe daño al 150%.

### 8.5 Muerte de Edrick

- [ ] **AC 5.1**: Edrick muere cuando llega a 0 HP. Verificar: HP > 0 pero recibe daño que lo trae a 0 → estado "dead".
- [ ] **AC 5.2**: Cutscena de muerte se dispara. Verificar: al morir → cinemática de muerte se ejecuta (~3-5s).
- [ ] **AC 5.3**: Escena se recarga después de muerte. Verificar: esperar a que cutscena termine → nivel recarga, Edrick reaparece en checkpoint.
- [ ] **AC 5.4**: Edrick respawns con HP completo. Verificar: después de reload, HP = 75 (o máximo).

### 8.6 Múltiples Daños

- [ ] **AC 6.1**: Múltiples daños en mismo frame se aplican. Verificar: fuego + enemigo atacan simultáneamente → ambos daños se restan de HP.
- [ ] **AC 6.2**: Una sola señal se emite para daños múltiples. Verificar: dos daños → signal `health_changed` emitida una vez (no dos).
- [ ] **AC 6.3**: Daño overkill se maneja correctamente. Verificar: 10 HP, 1000 daño → 0 HP, no crash.

### 8.7 Integración General

- [ ] **AC 7.1**: El sistema se puede desactivar en cinemáticas. Verificar: durante cutscena narrativa, daño no se aplica a menos que Cinemáticas lo solicite explícitamente.
- [ ] **AC 7.2**: Items se aplican correctamente. Verificar: encontrar item +resist → resistencia se suma al total.
- [ ] **AC 7.3**: Cambiar build actualiza resistencias. Verificar: cambiar demonio mid-combate → nuevas resistencias aplican al siguiente daño.
- [ ] **AC 7.4**: No hay glitches de HP. Verificar: HP es siempre un entero entre 0 e HP_máximo, nunca NaN o infinito.
