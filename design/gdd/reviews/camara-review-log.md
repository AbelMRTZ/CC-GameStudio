# Review Log: Sistema de Cámara (GDD #9)

---

## Review — 2026-05-27 (R3) — Verdict: APROBADO

Scope signal: M
Specialists: game-designer, systems-designer, gameplay-programmer, godot-specialist, qa-lead, creative-director
Blocking items: 4 → 0 (todos resueltos en esta sesión) | Recommended: 5 → 0

Summary: La revisión R2 encontró que los fixes de R1 introdujeron 3 bugs nuevos (limit_* = -1 incorrecto, F4 exit snap, lerp frame-rate dependency) y dejaron una contradicción sin resolver (E1 vs AC 4). Todos resueltos en R3. Además se incorporaron: effective_dir hybrid para look-ahead orgánico en desaceleración, consulta EventBus.is_combat_active() post-cinemática, criterios de autoría de CameraAnchors, y validación de camera_data. El GDD está listo para implementación.

Prior verdict resolved: Sí — R2 fue MAJOR REVISION NEEDED (2026-05-27); R3 es APROBADO.

---

## Review — 2026-05-27 — Verdict: MAJOR REVISION NEEDED → REVISIONS APPLIED 2026-05-27

**Scope signal:** L (Large)

**Specialists:** game-designer, systems-designer, qa-lead, godot-specialist, gameplay-programmer, creative-director

**Blocking items:** 7 → **0** (all resolved in revision pass)

**Summary:** Design intent is sound (cinematic camera for *Demons of Dravaryn*'s Pilar 1), but specification had ambiguities and formula contradictions. Revision pass completed:
- [A1] F4 exit formula specified (lerp continuous from current offset, Option B selected)
- [A2] Look-ahead philosophy chosen (sutil ~9.5px, asintótica). AC 2 y AC 4 rewritten with correct numbers.
- [A3] AC 4 convergence updated to 30-40 frames (asintótico, 95% en 34 frames) vs impossible 8-12 frames.
- [A4] `lerp_delta_clamp_max` mechanism specified: clamp output delta to 12px/frame max.
- [A5] ADR-002 updated: cinematic_started/cinematic_ended signals added to EventBus contract.
- [A6] Five untestable ACs rewritten (AC 1, AC 8, AC 9B, AC 10 now testable; AC 2, AC 4 fixed).
- [A7] Two new pillar-aligned ACs added (AC 16 feel-test, AC 17 rule-of-thirds composition).

**Prior verdict resolved:** First review — no prior state. Revision pass applied same session.

---

## Blocking Issues (Must Resolve Before Implementation)

### [A1] F4 exit formula not specified
- Two valid interpretations produce different camera behavior on CameraAnchor exit
- Programmer cannot implement without guessing
- Sources: [systems-designer #3 HIGH], [godot-specialist #1 HIGH], [gameplay-programmer #1 HIGH]

### [A2] AC 2 contradicts F2 — claimed ~16px vs formula gives ~9.5px
- Player fantasy (intended look-ahead magnitude) is unclear
- AC will fail on mathematically-correct implementation
- This is a design decision, not a math error
- Sources: [systems-designer #6 HIGH], [qa-lead BLOCKER AC 2]

### [A3] AC 4 convergence 8-12 frames is mathematically impossible
- F2 requires 34 frames to reverse look-ahead direction
- AC cannot pass under current formula parameters
- Sources: [gameplay-programmer #5 HIGH], [qa-lead BLOCKER AC 4]

### [A4] `lerp_delta_clamp_max` mechanism never specified
- Tuning knob described but no implementation formula provided
- Programmer doesn't know if clamp fires before/after lerp
- Sources: [godot-specialist #2 HIGH], [gameplay-programmer #1 HIGH]

### [A5] Cinematic signals missing from ADR-002 EventBus
- `cinematic_started` and `cinematic_ended` referenced but not defined in contract
- Will cause runtime null reference
- Sources: [godot-specialist #6 HIGH]

### [A6] Five Acceptance Criteria are unfit for validation
- AC 1 + AC 4: redundant assertions
- AC 2: threshold fails for all early convergence frames
- AC 8: black-frame detection unachievable by stated method
- AC 9B: "ignored/queued" ambiguous — both behaviors valid
- AC 10: no test setup; frame counter unspecified
- Sources: [qa-lead 5 BLOCKERs]

### [A7] ACs systematically avoid testing Player Fantasy
- All 15 ACs test pixels/frames; none tests "feels cinematic"
- For a game with Pilar 1 = Narrativa Cinematográfica, this is a category error
- Sources: [game-designer #10 MAJOR], [qa-lead pattern]

---

## Recommended Revisions (Important, Should Resolve)

### [B1] No vertical look-ahead — vertical traversal is blind spot
### [B2] 4-state machine missing DIALOGUE/NARRATIVE_EVENT state
### [B3] CINEMATIC post-combat race condition — re-emission timing unguaranteed
### [B4] "80% of zones no necesitan anchors" claim inconsistent with Pilar 1
### [B5] No screenshake/trauma system; no architecture to add later
### [B6] F2's 0.57s convergence may be too slow for combat responsiveness

---

## Specialist Consensus

All 5 specialists independently converged on same 5-6 structural problems from different domains. High confidence in findings. No significant disagreements — areas of apparent conflict (game-designer #7 vs gameplay-programmer #6) actually all point to [A2] above (unclear player fantasy).

---

## Creative Director Verdict

> "The design intent is correct. The vision is right. But the specification has too many ambiguities, contradictions, and gaps for a programmer to implement without making creative decisions on the designer's behalf. Given that this is *Demons of Dravaryn*'s cinematic backbone — the system that delivers Pilar 1 — those decisions cannot be delegated.
> 
> The 7 required fixes are achievable in a single focused work session (4-6 hours). After revision, this is likely a clean APPROVED on re-review."

---

## Recommended Next Steps

1. **Designer revision pass** — address A1-A7 (4-6 hours focused work)
2. **ADR-002 amendment** — add cinematic signals (1 hour)
3. **Re-review in new session** after fixes are complete

---

**Files to update for revision:**
- `/Users/abel/Desktop/UNIVERSIDAD/PROYECTOS/CC-GameStudio/design/gdd/camara.md`
- `/Users/abel/Desktop/UNIVERSIDAD/PROYECTOS/CC-GameStudio/docs/architecture/adr-001-manual-camera-smoothing.md`
- `/Users/abel/Desktop/UNIVERSIDAD/PROYECTOS/CC-GameStudio/docs/architecture/adr-002-eventbus-global-signals.md`

