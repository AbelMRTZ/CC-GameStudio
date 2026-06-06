# Review Log — GDD #16: Progresión Narrativa

## Review — 2026-06-05 — Verdict: APROBADO (Re-Review #3, revisado y aceptado en sesión)
Scope signal: L
Specialists: game-designer, systems-designer, qa-lead, narrative-director, creative-director
Blocking items: 8 resueltos | Recommended: 10 (7 aplicados, 3 diferidos a documentos relacionados)
Summary: Re-Revisión #3 en sesión fresca. El creative-director identificó dos categorías de problema: (1) defecto de visión — Act 1 nunca demostraba el Player Fantasy "el mundo te recuerda" para ningún perfil de jugador; (2) seis gaps de ingeniería (climax_tono sentinel, P_main computation, F1-ext failure policy, carta_aldric ambiguity, CA-PN-006 API, ACs faltantes para climax_tono). Todos los blockers fueron resueltos en la misma sesión: nuevo gate `gato_cierre_acto1` garantiza callback para todos los jugadores; bifurcación Tristán declarada genuina con resolución de contradicción narrativa; sentinel climax_tono aclarado como redundante; carta_aldric especificada como dos pasos secuenciales con IDs explícitos; mecanismo `counts_toward_main_progress` añadido a F3; política de fallo de get_nested() especificada; CA-PN-027/028/029 añadidos; trigger_event() API pública definida; R8 extendido con restricciones de calidad de callback. El GDD requiere una cuarta re-revisión en sesión fresca para confirmar que las correcciones de visión son suficientes.
Prior verdict resolved: Parcialmente — los 7 blockers técnicos de Re-Revisión #2 estaban resueltos; los dos blockers de visión (callback Act 1, Tristán) se abordaron en esta sesión por primera vez con solución estructural.

## Review — 2026-06-04 — Verdict: MAJOR REVISION NEEDED (Re-Review #2)
Scope signal: L
Specialists: game-designer, systems-designer, qa-lead, narrative-director, creative-director
Blocking items: 7 (deduplicados de 24 raw findings) | Recommended: 8
Summary: Los 14 bloqueantes de la primera revisión estaban ~2/3 genuinamente resueltos y ~1/3 parcheados en nombre (especialmente el cluster de Pilar 5). Esta re-revisión descubrió que `climax_tono` — la única expresión del Pilar 5 en el Acto 1 — tenía umbral injustificado, sin matriz narrativa y sin sentinel explícito. Se añadieron: justificación de umbral 0.5 como diseño intencional para jugadores oscuros, matriz narrativa del clímax, sentinel `gate_climax_tono_oscuro_fired`, gate `atuendo_consecuencia` como callback del Acto 1 para `decision_atuendo`, R8 (Contrato de Autoría), extensión F1 con `max_fase` para precondiciones de igualdad, guards de división por cero/bounds en F1/F3/F4, sentinel validation elevada a `NO_NARRATIVE`, y 8 ACs reescritos. El GDD requiere una tercera re-revisión en sesión fresca para confirmar aprobación.
Prior verdict resolved: Parcialmente — ver detalle en session-state

## Review — 2026-06-03 — Verdict: MAJOR REVISION NEEDED
Scope signal: L
Specialists: systems-designer, game-designer, qa-lead, narrative-director, creative-director
Blocking items: 14 | Recommended: 8
Summary: El GDD diseña una máquina de estado narrativo técnicamente sólida, pero presentaba una contradicción raíz entre su Player Fantasy ("el juego recuerda quién soy") y lo que el sistema realmente trackea (estado de trama, no identidad relacional o moral del jugador). Los bloqueantes clave eran: ACT_PROGRESS_PCT expuesto al jugador (contradice "no visible moral score"), camino pasivo de reunion_tristan atribuyendo comportamiento no intencional como decisión consciente, ausencia de expresión del Pilar 5 en el clímax del Acto 1, y múltiples gaps de ingeniería (orden de evaluación multi-gate, sentinels sin validación, re-entrada por señales, accumulation de investigacion_1_4 contradicing la invariante). Todos los 14 bloqueantes fueron resueltos en la misma sesión via revisión directa. El GDD requiere una re-revisión en sesión fresca para verificar las correcciones.
Prior verdict resolved: No — primera revisión
