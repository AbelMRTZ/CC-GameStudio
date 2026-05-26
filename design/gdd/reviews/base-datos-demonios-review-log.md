# Review Log: Base de Datos de Demonios

---

## Review — 2026-05-24 — Verdict: REVISIÓN MAYOR REQUERIDA → APROBADO (misma sesión)

Scope signal: L
Specialists: game-designer, systems-designer, qa-lead, narrative-director, creative-director
Blocking items: 7 | Recommended: 3
Prior verdict resolved: Sí — primera revisión, blockers resueltos en la misma sesión

Summary: El GDD era estructuralmente sólido pero tenía inconsistencias críticas: "turnos" como unidad de tiempo en un juego de tiempo real, convención de signos de resistencias invertida respecto a salud-daño.md, contradicción en Restricciones Críticas sobre las sinergias de Dash, y los sistemas de saturación demoníaca y corrupción moral estaban conflacionados (violando el Pilar 5). Visión carecía de coste real, convirtiendo el slot en free. Las sinergias narrativas del Gato no tenían compuertas, arriesgando revelar el giro del hermano demasiado pronto. Todos los bloqueantes fueron resueltos en la misma sesión.

### Bloqueantes Resueltos

1. **"Turnos" reemplazados por segundos**: quemadura = 3 HP/s durante 3.0s, extensiones en segundos, JSON usa `duracion_segundos` y `tasa_hp_por_segundo`
2. **Restricciones Críticas corregidas**: Dash sinergiza con Arcano, Fuego (Estela Ardiente) Y Hielo (Estela Congelada) — texto actualizado
3. **Triple Dash+Fuego+Hielo definido**: Sección 5.11 — ambas estelas se generan al 85% efectividad por Anulación Térmica
4. **Convención de signos resuelta**: negativo = resistencia (reduce daño), positivo = debilidad (amplifica daño). Fórmula, tabla 7.1, descripciones en 3.3, JSON de ejemplo y CAs actualizados
5. **Compuertas narrativas del Gato**: Telepatía Felina requiere Acto 2; Comunión Ancestral no revela identidad del gato antes del Acto 3 final; Revelación Felina siempre fragmentada en Acto 1-2
6. **Saturación demoníaca vs Corrupción moral separados**: Sección 5.8 reescrita con dos subsistemas explícitos — saturación (cosmética, reversible, por demonio) vs corrupción moral (global, permanente, arco narrativo Pilar 5)
7. **Visión con coste real**: −5 HP máximo (75→70) + interrupción breve (0.5s) en visions durante exploración. CA-050b y CA-052b añadidos.

### Recomendados (pendientes para sesiones futuras)

1. Enriquecer lore individual de demonios elementales (Fuego, Hielo, Arcano) — actualmente tienen historias escuetas
2. Definir coste narrativo/dramático de cambio de loadout para Pilar 1 (Cinemático) — actualmente es instantáneo
3. Evaluar si 7 demonios MVP es alcanzable en timeline 3-6 meses solo — considerar reducir a 5 para validación de loop
