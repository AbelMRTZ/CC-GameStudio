# Review Log: GDD #12 — Guardado y Carga

## Review — 2026-06-03 — Verdict: MAJOR REVISION NEEDED → APPROVED (revisiones aplicadas en sesión)

Scope signal: L
Specialists: game-designer, systems-designer, qa-lead, godot-gdscript-specialist, creative-director
Blocking items: 14 | Recommended: 9
Summary: El GDD describía algoritmos que no podían ejecutarse en Godot 4.x — `DirAccess.make_dir_recursive()` y `DirAccess.rename()` llamados como métodos estáticos cuando son de instancia, con `rename()` devolviendo `Error` (no `bool`) lo que habría marcado cada save exitoso como fallido. `FileAccessWrapper` fue extendido para envolver también `store_string_in()`, ya que `FileAccess` no es subclaseable en GDScript. Se añadió rotación de backup (`save_data.bak`), espera bloqueante en save-on-exit durante SAVING, guards de sub-campo en ambos algoritmos, `companion_state` en CRITICAL_FIELDS, clamp independiente para `corruption_floor`, y 3 nuevos ACs (CA-SL-023/024/025). CA-SL-018 partido en 4 criterios independientes. CA-SL-003 reescrito con mecanismo de comparación explícito.
Prior verdict resolved: No — primera revisión formal registrada.
