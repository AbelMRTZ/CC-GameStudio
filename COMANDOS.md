# 🎮 Comandos Disponibles — CC Game Studio

Guía de referencia rápida de todos los comandos (`/skills`) disponibles en el proyecto.
Para orientación general, ejecuta `/help` o `/project-stage-detect`.

---

## 🎯 ORIENTACIÓN Y PLANEACIÓN

| Comando | Descripción |
|---------|------------|
| `/start` | Onboarding inicial — de cero a proyecto (si es la primera vez) |
| `/help` | Dónde estás y qué hacer ahora |
| `/project-stage-detect` | Análisis completo del estado actual del proyecto |

---

## 🎨 FASE: CONCEPT & BRAINSTORM

| Comando | Descripción |
|---------|------------|
| `/brainstorm` | Ideación guiada: concepto → GDD estructurado |
| `/map-systems` | Descomponer concepto en sistemas (dependencias, orden diseño) |
| `/art-bible` | Crear guía visual de identidad del juego |

---

## 📋 FASE: SYSTEMS DESIGN

| Comando | Descripción |
|---------|------------|
| `/design-system` | Diseñar un sistema (8 secciones de GDD) |
| `/design-review` | Revisar un GDD completo con 5 especialistas |
| `/quick-design` | Diseño rápido para balance/tuning (no GDD completo) |
| `/review-all-gdds` | Review holístico de TODOS los GDDs a la vez |
| `/consistency-check` | Validar que valores en GDDs no conflictúan |
| `/scope-check` | ¿Cabe todo en MVP? Validar scope |

---

## 🏗️ FASE: TECHNICAL SETUP & ARCHITECTURE

| Comando | Descripción |
|---------|------------|
| `/setup-engine` | Configurar engine (Godot/Unity/Unreal) y versión |
| `/create-architecture` | Diseñar arquitectura técnica del proyecto |
| `/architecture-review` | Review de decisiones arquitectónicas |
| `/architecture-decision` | Registrar una ADR (Architecture Decision Record) |
| `/create-control-manifest` | Especificar controles de entrada (keyboard/gamepad/etc) |
| `/design-system` | Sistema de diseño UI/UX |

---

## 📖 FASE: PRE-PRODUCTION

| Comando | Descripción |
|---------|------------|
| `/gate-check pre-production` | ¿Pasamos de Systems Design a Pre-Prod? |
| `/create-epics` | Convertir sistemas en épicas de trabajo |
| `/create-stories` | Desglosar épicas en historias de sprint |
| `/asset-spec` | Generar specs de assets para artistas |
| `/asset-audit` | Auditar assets (tamaños, formatos, naming) |
| `/vertical-slice` | Prototipo jugable de loop principal |

---

## 👨‍💻 FASE: PRODUCTION (Coding & Implementation)

| Comando | Descripción |
|---------|------------|
| `/dev-story` | Guía paso-a-paso para implementar una story |
| `/sprint-plan` | Planificar sprint (qué stories, estimaciones) |
| `/sprint-status` | Estado actual del sprint |
| `/story-readiness` | ¿Es esta story implementable? |
| `/story-done` | Marcar story como completada |
| `/code-review` | Review de código (PR analysis) |
| `/test-setup` | Configurar framework de tests (GUT/pytest/etc) |
| `/test-flakiness` | Debug de tests que fallan aleatoriamente |
| `/test-evidence-review` | Review de test coverage (unit/integration/manual) |
| `/smoke-check` | Quick sanity test antes de merge |
| `/soak-test` | Test de resistencia (stress test) |
| `/run` | Ejecutar el app y ver cambios en vivo |
| `/verify` | Verificar que un fix funciona realmente |

---

## ⚡ FASE: POLISH

| Comando | Descripción |
|---------|------------|
| `/gate-check production` | ¿Pasamos a Polish? |
| `/tech-debt` | Auditar y priorizar deuda técnica |
| `/perf-profile` | Medir performance (FPS, memory, draw calls) |
| `/balance-check` | Validar balance de juego (dificultad, demonios, etc) |
| `/ux-review` | Review de UX (flows, accesibilidad, etc) |
| `/ui-ux-pro-max` | Diseño avanzado de UI (50+ estilos, 161 paletas) |
| `/team-polish` | Coordinación: última pasada de pulido |

---

## 🎬 FASE: RELEASE & LAUNCH

| Comando | Descripción |
|---------|------------|
| `/gate-check launch` | ¿Estamos listos para lanzar? |
| `/release-checklist` | Checklist pre-lanzamiento (certs, etc) |
| `/launch-checklist` | Plan de lanzamiento día-1 |
| `/day-one-patch` | Preparar patch para bugs post-lanzamiento |
| `/changelog` | Generar changelog automático |
| `/patch-notes` | Redactar notas de parche para jugadores |
| `/hotfix` | Emergencia: fix crítico post-launch |

---

## 📊 EQUIPO & COORDINACIÓN

| Comando | Descripción |
|---------|------------|
| `/team-narrative` | Coordinación: narrativa/diálogos |
| `/team-ui` | Coordinación: UI/UX |
| `/team-audio` | Coordinación: sonido/música |
| `/team-level` | Coordinación: level design |
| `/team-combat` | Coordinación: combate |
| `/team-qa` | Coordinación: QA y testing |
| `/team-release` | Coordinación: lanzamiento |
| `/team-live-ops` | Coordinación: post-launch (eventos, BP, etc) |

---

## 🔧 ESPECIALIDAD: DISEÑO

| Comando | Descripción |
|---------|------------|
| `/ux-design` | Diseño UX completo (flows, accesibilidad) |
| `/localize` | Internacionalización y localización |
| `/playtest-report` | Documentar resultados de playtest |
| `/retrospective` | Retrospectiva de sprint/fase |

---

## 🔍 ESPECIALIDAD: QA & TESTING

| Comando | Descripción |
|---------|------------|
| `/qa-plan` | Estrategia de testing completa |
| `/bug-triage` | Clasificar y priorizar bugs |
| `/bug-report` | Documentar bug con paso-a-paso |
| `/regression-suite` | Test suite para evitar regressions |

---

## 🔐 ESPECIALIDAD: SEGURIDAD

| Comando | Descripción |
|---------|------------|
| `/security-audit` | Auditar vulnerabilidades de seguridad |
| `/security-review` | Review de seguridad en código |

---

## 📚 EXTRAS & UTILITIES

| Comando | Descripción |
|---------|------------|
| `/prototype` | Prototipo rápido (sin GDD, sin pulido) |
| `/estimate` | Estimar esfuerzo de una story |
| `/simplify` | Review + mejora de código |
| `/reverse-document` | Generar GDD desde código existente |
| `/update-config` | Configurar settings.json del proyecto |
| `/claude-api` | Ayuda con Claude API / Anthropic SDK |
| `/loop` | Ejecutar comando recurrentemente cada N minutos |
| `/schedule` | Programar un comando para más tarde (cron) |

---

## 🗺️ FLUJO TÍPICO (MVP)

1. **Concept** → `/brainstorm` → `/map-systems` → `/art-bible`
2. **Systems Design** → `/design-system` (para cada sistema) → `/design-review`
3. **Technical Setup** → `/setup-engine` → `/create-architecture`
4. **Pre-Production** → `/gate-check pre-production` → `/create-stories`
5. **Production** → `/sprint-plan` → `/dev-story` → `/code-review` → `/story-done`
6. **Polish** → `/perf-profile` → `/balance-check` → `/ux-review`
7. **Release** → `/gate-check launch` → `/launch-checklist` → `/hotfix` (si es necesario)

---

## 📍 ESTADO ACTUAL DEL PROYECTO

**Fase:** Systems Design (GDD #4 y #5 completos)

**Próximo paso recomendado:**
```
/design-system combate-tiempo-real
```

---

## 💡 TIPS

- Ejecuta `/help` sin argumentos para saber exactamente dónde estás y qué hacer ahora
- Ejecuta `/project-stage-detect` para un análisis completo del estado
- Los comandos con `/gate-check` son "puertas" entre fases — ejecuta antes de avanzar
- Puedes usar `/loop` para ejecutar comandos recurrentemente (ej: `/loop 10m /sprint-status`)
