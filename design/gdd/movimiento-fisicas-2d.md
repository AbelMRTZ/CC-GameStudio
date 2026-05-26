# GDD: Movimiento y Físicas 2D

> **Estado**: Aprobado
> **Creado**: 2026-05-24
> **Última Actualización**: 2026-05-24
> **Aprobación**: Completado en sesión de diseño
> **Sistema**: Movimiento y Físicas 2D
> **Milestone**: MVP — Foundation Layer
> **Depende de**: — (sin dependencias)
> **Dependen de este sistema**: Combate en Tiempo Real, IA de Enemigos, Exploración del Mundo, Cámara

---

## 1. Visión General

El sistema de Movimiento y Físicas 2D define cómo Edrick Velmar se desplaza por el mundo de Dravaryn: movimiento horizontal responsivo, salto con control de altura, colisiones con el entorno y las reglas de física base que gobiernan velocidad, gravedad y aceleración. El sistema establece los valores por defecto sobre los cuales los demonios vinculados pueden aplicar modificadores — alterando velocidad, gravedad, inercia, u otorgando mecánicas de movimiento adicionales como dash de ataque. El dash es un ejemplo de mecánica de movimiento demonio-otorgada; otros demonios podrán conceder movimientos especiales aún no definidos. Toda la física opera sobre `CharacterBody2D` en GodotPhysics 2D, resolución interna 320×180 px, tile base 16×16 px.

---

## 2. Fantasy del Jugador

Cuando el jugador controla a Edrick, debería sentir: **respuesta instantánea** — cada entrada de teclado produce movimiento inmediato sin lag ni inercia excesiva. **Control preciso** — el jugador puede posicionarse exactamente donde quiere, pivotear, frenar o cambiar dirección en milisegundos. **Peso narrativo** — aunque responsivo, Edrick no es un acróbata; cada movimiento comunica que es un joven educado de casa noble, no un soldado entrenado. El salto debería ser satisfactorio pero no excesivamente ágil. El dash de ataque debería sentir como una técnica marcial aprendida (quizá del demonio mismo), no un truco de acrobacia. **Confianza en el control** — el jugador nunca debería culpar a las físicas por un fallo; los controles deben ser tan predecibles y responsivos que los errores sean siempre del jugador, no del sistema.

---

## 3. Reglas Detalladas

### 3.1 Movimiento Horizontal Base

Edrick se controla con **A/D** (izquierda/derecha) o **flechas de dirección**. El movimiento es acelerado, no instantáneo:
- **Velocidad máxima sin modificadores**: 250 px/s
- **Aceleración**: 1200 px/s² (mientras la tecla está presionada)
- **Desaceleración (fricción)**: 800 px/s² (cuando se suelta la tecla)
- **Cambio de dirección**: Aplicar aceleración en dirección opuesta (permite cambios rápidos)

*Justificación*: La aceleración rápida (~1200) produce respuesta instantánea al jugador. La fricción alta (~800) permite frenar y pivotar en menos de 0.3 segundos, lo que es necesario para combate en tiempo real. La velocidad máxima (~250) mantiene a Edrick ágil pero con peso narrativo.

### 3.2 Salto

**Entrada**: Barra espaciadora. El jugador puede saltar **solo si está en contacto con suelo** (verificación `is_on_floor()`).

- **Altura máxima del salto**: 80 px (~5 tiles)
- **Velocidad inicial de salto**: -180 px/s (negativa = arriba)
- **Control de altura**: Si se mantiene **espacio presionado**, el salto alcanza altura máxima. Si se suelta **inmediatamente**, el salto es más bajo (~45 px). Aplicamos reducción de velocidad vertical en 0.1 segundos si se suelta.
- **Tiempo de salto máximo**: ~0.9 segundos (desde inicio hasta altura máxima)
- **Gravedad**: 400 px/s² (constante, salvo modificadores de demonios)

*Justificación*: Control de altura es fundamental en plataformas. La gravedad relativamente alta (400) evita saltos flotantes. 80 px de altura permite franquear obstáculos de ~5 tiles, necesario para traversal.

### 3.3 Caída

Cuando Edrick está en el aire (en ningún suelo):
- **Velocidad de caída máxima (terminal velocity)**: 300 px/s (sin modificadores)
- **Aceleración gravitacional**: 400 px/s² (aplicada cada frame en `_physics_process`)
- **Fricción aérea**: Ninguna (el aire no frena el movimiento horizontal)

*Nota*: El movimiento horizontal sigue siendo 250 px/s máx incluso en el aire. La gravedad y velocidad de caída son independientes.

### 3.4 Contacto con Suelo

- Detectado por `CharacterBody2D.is_on_floor()` (raycast de Godot)
- Se considera "en suelo" si el raycast de Godot detecta colisión bajo los pies dentro de 1 frame
- **Coyote Time**: El jugador puede saltar durante **0.15 segundos** después de abandonar el suelo. Esto permite saltos "gratos" en plataformas que bajan muy rápido
- **Jump Buffer**: El jugador puede presionar espacio hasta **0.1 segundos** antes de tocar suelo, y el salto se ejecutará cuando haga contacto

*Justificación*: Coyote time y jump buffer son estándares en juegos responsivos (Hollow Knight, Celeste). Mejoran la sensación de control sin romper la física.

### 3.5 Colisiones

- **Nodo**: `CharacterBody2D` de Godot
- **Forma**: `CapsuleShape2D` (48 px alto × 24 px ancho — aproximadamente el sprite de Edrick)
- **Capas de colisión**: 
  - Edrick: layer 1, mask = layers 2, 3, 4 (enemigos, objetos, plataformas)
  - Plataformas: layer 2, no interactúan entre sí
  - Enemigos: layer 3
  - Objetos: layer 4
- **One-way platforms**: Algunas plataformas son "de una sola dirección" — Edrick puede pasar a través desde abajo pero no desde arriba. Detectado por tileset metadata.

### 3.6 Modificadores de Demonios

Cada demonio puede aplicar **uno o más** de estos modificadores al movimiento base:
- **Multiplicador de velocidad máxima**: Rango permitido 0.5× a 2.0× (ej: -50% a +100%)
- **Multiplicador de gravedad**: Rango permitido 0.5× a 1.5× (no permitir gravedad invertida como regla de diseño)
- **Multiplicador de fricción**: Rango permitido 0.3× a 1.5×
- **Salto alternativo**: Algunos demonios podrían cambiar el tipo de salto (ej: triple jump, wall jump) — estos son **nuevas mecánicas** y se definen por demonio

*Nota técnica*: Los modificadores se aplican en `CharacterBody2D.process(delta)` como multiplicadores a los valores base. Si múltiples demonios se activan simultáneamente, los modificadores se **apilan multiplicativamente** (no aditivamente).

### 3.7 Dash de Ataque (Demonio-Otorgado)

El dash es una mecánica de movimiento que **un demonio específico** otorga a Edrick:
- **Entrada**: Presionar **SHIFT** (requiere que el demonio otorgador esté equipado)
- **Dirección**: Edrick hace dash en la dirección actual que está mirando (si se mantiene dirección presionada) o hacia el último input de dirección registrado
- **Velocidad de dash**: 400 px/s
- **Duración**: 0.15 segundos (60 frames a 60fps)
- **Cooldown**: 0.6 segundos (no se puede usar dash nuevamente hasta después de 0.6s)
- **Efecto de ataque**: Durante el dash, Edrick está en estado "attack-dash". La colisión con enemigos dispara lógica de daño (definida en el sistema de Combate, no aquí)
- **Animación**: La animación de Edrick cambia a "dash" durante la duración. Después, vuelve a caminar/idle
- **Cancelación**: El dash no puede ser cancelado una vez iniciado. El jugador recupera control cuando termina

*Nota*: El dash de ataque combina **movimiento puro** (el desplazamiento) con **lógica de combate** (el daño). La parte de movimiento se define aquí; la de daño pertenece al GDD de Combate.

### 3.8 Interrupción de Movimiento

Cuando Edrick entra en estado de **stun**, **knockback** o similar (definidos en Combate):
- El input de movimiento es ignorado
- La velocidad se reaplica según el tipo de efecto
- El movimiento se recupera cuando el efecto termina

---

## 4. Fórmulas

### 4.1 Movimiento Horizontal

```
velocity_x += sign(input_dir) * accel * delta    (si input ≠ 0)
velocity_x = lerp(velocity_x, 0, friction * delta)  (si input = 0)
velocity_x = clamp(velocity_x, -max_speed, max_speed)

Donde:
  velocity_x = velocidad horizontal actual (px/s)
  input_dir = -1 (izq), 0 (ninguno), +1 (der)
  accel = 1200 px/s²
  friction = 800 px/s²
  max_speed = 250 px/s (base, modificable por demonios)
  delta = tiempo por frame (~0.0167s a 60fps)
```

### 4.2 Salto

```
Si es_en_suelo() y espacio_presionado:
  velocity_y = -180 px/s
  tiempo_salto_inicio = tiempo_actual

Si espacio_soltado y tiempo_desde_inicio < 0.1s:
  velocity_y = max(velocity_y, -90 px/s)  # Reducir altura si se suelta rápido
```

### 4.3 Gravedad y Caída

```
velocity_y += gravedad * delta    (mientras no en suelo)
velocity_y = clamp(velocity_y, -∞, terminal_velocity)

Donde:
  gravedad = 400 px/s²
  terminal_velocity = 300 px/s
  delta = tiempo por frame
```

### 4.4 Coyote Time

```
Si NO es_en_suelo():
  tiempo_en_aire += delta
  Si tiempo_en_aire > 0.15s:
    puede_saltar = false
  Sino:
    puede_saltar = true

Si es_en_suelo():
  tiempo_en_aire = 0
  puede_saltar = true
```

### 4.5 Jump Buffer

```
Si espacio_presionado:
  tiempo_buffer_salto = 0
  
Si NO espacio_presionado:
  tiempo_buffer_salto += delta

Si es_en_suelo() y tiempo_buffer_salto < 0.1s:
  Ejecutar salto (aun si espacio ya fue soltado)
```

### 4.6 Modificador de Velocidad (Demonio)

```
max_speed_actual = max_speed * demonio_speed_mult
accel_actual = accel * demonio_speed_mult
friction_actual = friction * demonio_speed_mult

Donde:
  demonio_speed_mult ∈ [0.5, 2.0]
  
Si múltiples demonios activos:
  mult_resultante = mult_1 × mult_2 × mult_3 ...  (multiplicativo)
```

### 4.7 Modificador de Gravedad (Demonio)

```
gravedad_actual = gravedad * demonio_gravity_mult
terminal_velocity_actual = terminal_velocity * demonio_gravity_mult

Donde:
  demonio_gravity_mult ∈ [0.5, 1.5]
  (No permitir < 0.5 ni > 1.5 — evitar gravedad invertida o extrema)
```

### 4.8 Dash de Ataque

```
Si SHIFT presionado y demonio_otorgador equipado y tiempo_desde_último_dash > 0.6s:
  dash_activo = true
  dash_velocidad = 400 px/s
  dash_dirección = dirección_actual_o_último_input
  dash_tiempo_inicio = tiempo_actual

Si dash_activo:
  velocity_x = dash_velocidad * cos(dash_dirección)
  velocity_y = dash_velocidad * sin(dash_dirección)
  
Si (tiempo_actual - dash_tiempo_inicio) > 0.15s:
  dash_activo = false
  tiempo_desde_último_dash = 0
  # Control retorna al jugador, velocidades vuelven a valores normales
```

---

## 5. Casos Extremos

### 5.1 Cambio de Dirección Rápido

**Situación**: Edrick está corriendo a la derecha a 250 px/s, el jugador presiona izquierda.

**Comportamiento**: 
- La aceleración se aplica en dirección opuesta (1200 px/s² hacia la izquierda)
- Edrick se detiene en ~0.2 segundos y comienza a moverse hacia la izquierda
- No hay "deslizamiento" de inercia excesiva
- La animación cambia según la dirección del movimiento

### 5.2 Salto en Borde de Plataforma (Coyote Time)

**Situación**: Edrick camina hacia el borde de una plataforma y sale. En los siguientes 0.15 segundos, el jugador presiona espacio.

**Comportamiento**:
- El salto se ejecuta COMO SI estuviera aún en suelo
- Altura máxima = 80 px (igual al salto normal)
- Después de 0.15s en el aire, el salto ya no es posible
- Esto permite saltos "gratos" y perdona imprecisión del jugador

### 5.3 Presionar Salto Antes de Tocar Suelo (Jump Buffer)

**Situación**: Edrick está cayendo en el aire. 0.08 segundos ANTES de tocar suelo, el jugador presiona espacio.

**Comportamiento**:
- El input se registra en el buffer
- Cuando Edrick toca suelo (incluso 0.08s después), el salto se ejecuta automáticamente
- Simula "llegada a tiempo" sin que el jugador necesite timing perfecto

### 5.4 Aterrizaje en Plataforma Moving

**Situación**: Una plataforma se mueve (ej: ascensor). Edrick salta hacia ella y aterriza mientras se mueve.

**Comportamiento**:
- Edrick es "pegado" a la plataforma mientras `is_on_floor() = true`
- El movimiento horizontal de la plataforma NO afecta a Edrick (la plataforma lo arrastra, no lo controla)
- Cuando Edrick salta, deja de estar "pegado"
- Las plataformas moving se manejan **fuera de este sistema** (Exploración)

### 5.5 Dash en el Aire

**Situación**: Edrick salta. Mientras está en el aire, presiona SHIFT (demonio otorgador equipado).

**Comportamiento**:
- El dash se ejecuta normalmente
- Edrick hace dash a 400 px/s en la dirección del último input horizontal
- Si no hay input horizontal activo, el dash es en la dirección que estaba mirando
- La gravedad NO aplica durante el dash (solo dura 0.15s)
- Después del dash, la gravedad se reaplica normalmente

### 5.6 Dos Demonios con Modificadores en Conflicto

**Situación**: Edrick lleva dos demonios equipados. El demonio A da multiplicador de velocidad 1.5×. El demonio B da multiplicador de velocidad 0.7×.

**Comportamiento**:
- Los multiplicadores se apilan: 1.5 × 0.7 = 1.05
- Velocidad máxima actual = 250 × 1.05 = 262.5 px/s
- Si A da gravedad 1.2× y B da gravedad 0.8×: 1.2 × 0.8 = 0.96
- Gravedad actual = 400 × 0.96 = 384 px/s²

### 5.7 Equipar/Desequipar Demonio en Movimiento

**Situación**: Edrick está corriendo a 200 px/s (con demonio que da 1.2× velocidad). Cambia a otro demonio que da 0.8× velocidad.

**Comportamiento**:
- El nuevo multiplicador se aplica en el siguiente frame
- Si velocidad actual > nuevo max speed, se clampea inmediatamente
- Ejemplo: 200 px/s > (250 × 0.8 = 200 px/s) → Edrick no cambia velocidad en este caso
- La transición es instantánea, sin suavizado

### 5.8 Knockback o Stun en Medio del Dash

**Situación**: Edrick está haciendo dash attack. Un enemigo lo golpea y lo stun.

**Comportamiento**:
- El estado de dash se interrumpe inmediatamente
- Edrick recibe knockback (velocidad forzada en cierta dirección por N segundos)
- No puede usar dash de nuevo hasta que pasen 0.6 segundos desde que se INICIÓ el último dash (o desde que el stun termina, lo que sea más largo)

---

## 6. Dependencias

### 6.1 Dependencias de Este Sistema

Este sistema de **Movimiento y Físicas 2D** es Foundation Layer — **no depende de ningún otro sistema**. Es el fundamento sobre el cual se construyen todos los demás.

### 6.2 Sistemas que Dependen de Este

Los siguientes sistemas dependen de las físicas base y el control de movimiento definidos aquí:

1. **Combate en Tiempo Real** — Depende del movimiento para evitar enemigos, posicionamiento en combate, integración del dash attack
2. **IA de Enemigos** — Depende de las mismas físicas (CharacterBody2D, gravedad, aceleración) para movimiento de enemigos consistente
3. **Exploración del Mundo** — Depende del control responsivo de Edrick para atravesar niveles, plataformas, puzzles de traversal
4. **Cámara** — Depende de la posición y velocidad de Edrick para seguimiento suave

### 6.3 Integración con Otros Sistemas

**Base de Datos de Demonios**: Define los multiplicadores que los demonios pueden aplicar a las físicas. Este sistema APLICA esos multiplicadores, pero la definición de qué modificadores otorga cada demonio vive en ese sistema.

**Loadout & Build Management**: Cuando cambia el build de Edrick, los modificadores de demonios se reaplican. Este sistema recibe los nuevos valores y los aplica al siguiente frame.

---

## 7. Parámetros de Ajuste

Todos los valores listados aquí pueden ser configurados en archivos de datos, **NO hardcodeados**. Cada valor debe ser ajustable en tiempo de ejecución o a través de archivos de configuración para permitir iteración de balance rápida.

| Parámetro | Valor Base | Rango Seguro | Aspecto Afectado | Notas |
|-----------|-----------|--------------|-----------------|--------|
| `accel_horizontal` | 1200 px/s² | 600–2000 | Responsividad de control | Mayor = más "snappy", menor = más pesado. Valores < 600 se sienten lentos; > 2000 se sienten arcade |
| `friction_horizontal` | 800 px/s² | 400–1200 | Tiempo para frenar | Mayor = frena más rápido. Determina si el jugador puede pivotar rápido |
| `max_speed` | 250 px/s | 150–400 | Velocidad de movimiento | Mayor = Edrick se mueve más rápido. Afecta distancia de combate y sincronización de ataques |
| `jump_velocity` | -180 px/s | -150 a -220 | Altura máxima de salto | Mayor magnitud = saltos más altos. Rango actual da ~80 px |
| `gravity` | 400 px/s² | 300–600 | Tiempo de caída / peso de salto | Mayor = cae más rápido, saltos se sienten más "poundantes". Menor = saltos flotan más |
| `terminal_velocity` | 300 px/s | 250–400 | Velocidad máxima de caída | Mayor = cae más rápido después de cierto punto. Afecta cómo cae desde alturas |
| `coyote_time` | 0.15 s | 0.1–0.2 s | Gracia en borde (salto tardío) | Mayor = más perdón al saltar desde borde. Mejora "feel" pero puede volverse impreciso |
| `jump_buffer_time` | 0.1 s | 0.05–0.15 s | Gracia en llegada (presionar antes) | Mayor = más perdón si presionabas salto demasiado pronto |
| `dash_speed` | 400 px/s | 350–500 | Velocidad del dash attack | Mayor = dash más rápido, penetra defensa más fácil |
| `dash_duration` | 0.15 s | 0.1–0.2 s | Cuánto dura el dash | Mayor = cubre más distancia, más tiempo de invulnerabilidad potencial |
| `dash_cooldown` | 0.6 s | 0.4–1.0 s | Tiempo entre usos | Mayor = menos disponible, requiere más estrategia |
| `short_jump_multiplier` | 0.5 | 0.3–0.7 | Altura si sueltas espacio rápido | Define cuánto baja el salto si no lo mantienes presionado |
| `demonio_speed_mult_min` | 0.5 | 0.3–0.7 | Ralentización máxima permitida | Define cuánto puede ralentizar un demonio |
| `demonio_speed_mult_max` | 2.0 | 1.5–3.0 | Aceleración máxima permitida | Define cuánto puede acelerar un demonio |
| `demonio_gravity_mult_min` | 0.5 | 0.3–0.8 | Gravedad mínima permitida | Define gravedad más baja que un demonio puede dar |
| `demonio_gravity_mult_max` | 1.5 | 1.2–2.0 | Gravedad máxima permitida | Define gravedad más alta que un demonio puede dar |

**Fichero de configuración**: Todos estos valores deben vivir en un archivo de datos (ej: `assets/data/movement_config.tres` o `movement_config.gd`), accesible al `PlayerController` script.

---

## 8. Criterios de Aceptación

**Todos los criterios deben ser probables y verificables. QA debe poder ejecutar cada uno y confirmar PASS o FAIL.**

### 8.1 Movimiento Horizontal

- [ ] **AC 1.1**: Presionar A/D hace que Edrick acelere en esa dirección a ~1200 px/s². Verificar: la aceleración toma ~0.2s para alcanzar 250 px/s.
- [ ] **AC 1.2**: Soltar A/D hace que Edrick descelere suavemente en ~0.3s (fricción visible pero no snap). Verificar: a 250 px/s soltar toma ~0.3s para llegar a 0 px/s.
- [ ] **AC 1.3**: Cambiar dirección (A→D o D→A) es responsivo. Verificar: pivoteo visible en <0.2s sin deslizamiento excesivo.
- [ ] **AC 1.4**: La velocidad máxima nunca excede 250 px/s (sin modificadores). Verificar: acelerar indefinidamente no pasa de 250.

### 8.2 Salto

- [ ] **AC 2.1**: Presionar espacio mientras está en suelo hace que Edrick salte ~80 px de altura. Verificar: midiendo altura máxima alcanzada.
- [ ] **AC 2.2**: Soltar espacio rápido (<0.1s) resulta en salto más bajo (~45 px). Verificar: presionar y soltar inmediatamente → altura menor.
- [ ] **AC 2.3**: Mantener espacio presionado alcanza altura máxima (~80 px). Verificar: espacio presionado → 80 px.
- [ ] **AC 2.4**: No se puede saltar si no está en suelo. Verificar: en el aire, presionar espacio NO ejecuta salto.
- [ ] **AC 2.5**: Coyote time funciona. Verificar: caminar hacia borde, esperar 0.1s en el aire, presionar espacio → salta (< 0.15s después). Presionar espacio > 0.15s después del borde → no salta.
- [ ] **AC 2.6**: Jump buffer funciona. Verificar: presionar espacio 0.08s antes de tocar suelo, luego tocar suelo → automáticamente salta.

### 8.3 Caída y Gravedad

- [ ] **AC 3.1**: Edrick cae cuando está en el aire. Verificar: after jump, cae constantemente a 400 px/s² hasta terminal velocity.
- [ ] **AC 3.2**: Terminal velocity es ~300 px/s. Verificar: caída desde altura INFINITA → velocidad máxima 300 px/s.
- [ ] **AC 3.3**: Aterrizaje es suave. Verificar: Edrick entra en suelo, velocidad vertical se resetea, `is_on_floor()` = true.

### 8.4 Colisiones

- [ ] **AC 4.1**: Edrick no puede atravesar plataformas. Verificar: saltar hacia plataforma sólida → colisiona y se detiene.
- [ ] **AC 4.2**: One-way platforms permiten pasar desde abajo. Verificar: saltar hacia plataforma "de una vía" desde abajo → pasa a través.
- [ ] **AC 4.3**: One-way platforms bloquean desde arriba. Verificar: caer sobre plataforma "de una vía" desde arriba → colisiona.
- [ ] **AC 4.4**: Colisión con enemigos interrumpe movimiento (si hay knockback). Verificar: Edrick choca con enemigo, recibe knockback en dirección correcta.

### 8.5 Dash de Ataque

- [ ] **AC 5.1**: Dash requiere demonio. Verificar: sin demonio otorgador equipado, SHIFT no hace nada. Con demonio: SHIFT hace dash.
- [ ] **AC 5.2**: Dash se ejecuta en dirección del input. Verificar: presionar D + SHIFT → dash hacia la derecha. A + SHIFT → izquierda.
- [ ] **AC 5.3**: Dash dura 0.15s y cubre ~60 px de distancia (400 px/s × 0.15s). Verificar: medir distancia recorrida en dash.
- [ ] **AC 5.4**: Cooldown funciona. Verificar: dash, esperar < 0.6s, presionar SHIFT → no hace nada. Esperar > 0.6s → dash funciona.
- [ ] **AC 5.5**: Dash no puede ser cancelado. Verificar: durante dash, presionar A/D NO cambia dirección. Se completa automáticamente.
- [ ] **AC 5.6**: Dash en el aire funciona. Verificar: saltar, presionar SHIFT en el aire → dash se ejecuta.
- [ ] **AC 5.7**: Dash interactúa con enemigos. Verificar: dash toca enemigo, el GDD de Combate registra colisión.

### 8.6 Modificadores de Demonios

- [ ] **AC 6.1**: Demonio con +20% velocidad incrementa max_speed a 300 px/s. Verificar: acelerar con demonio → 300 px/s máx.
- [ ] **AC 6.2**: Demonio con -30% gravedad reduce gravedad a 280 px/s². Verificar: saltar con demonio → caída más lenta.
- [ ] **AC 6.3**: Múltiples modificadores se apilan multiplicativamente. Verificar: dos demonios con 1.2× y 0.8× → 1.2 × 0.8 = 0.96× aplicado.
- [ ] **AC 6.4**: Cambiar de demonio cambia valores inmediatamente. Verificar: cambiar build en combate → movimiento se siente diferente en el siguiente frame.

### 8.7 Integración General

- [ ] **AC 7.1**: El sistema es responsivo (< 50ms input lag). Verificar: presionar A, Edrick comienza a moverse visiblemente en <1 frame.
- [ ] **AC 7.2**: No hay glitches de clipping (Edrick traspasando paredes). Verificar: movimiento contra paredes = bloqueo limpio.
- [ ] **AC 7.3**: El sistema escala correctamente con framerate. Verificar: física funciona igual a 30, 60 y 120 fps (usando delta time).
- [ ] **AC 7.4**: Todas las pruebas deben pasar a resolución 320×180 (escala 4×). Verificar: movimiento y colisiones funcionan a resolución interna.
