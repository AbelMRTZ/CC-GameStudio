# ADR-001: Manual Camera Smoothing vs Native Position Smoothing

**Status**: Accepted  
**Date**: 2026-05-27  
**Decider**: Abel + Creative Director  
**Affected Systems**: Camera (#9), Cinematics (#17)

---

## Context

The Godot `Camera2D` node provides a built-in `position_smoothing_enabled` feature that lerps the camera position automatically. However, this approach has a constraint: the smoothing speed is fixed per frame, and cannot be interpolated dynamically between exploration and combat modes without introducing snapping or discontinuities.

The Camera system needs to:
- Follow Edrick with smooth lerp in explorations (smoothing_speed = 6.0)
- Follow Edrick with faster lerp in combat (smoothing_speed = 9.0)
- Transition between these modes without visual jumps or snapping

Using the native `position_smoothing_enabled` would require either:
- A fixed smoothing speed (incompatible with dynamic mode transitions)
- Disabling/re-enabling the feature during transitions (causes snapping)
- Manually interpolating the speed parameter while the feature is active (redundant, conflicts with native)

---

## Decision

**Implement manual camera smoothing via `lerp()` in `_process(delta)`.** Disable `Camera2D.position_smoothing_enabled`.

```gdscript
# In Camera controller script, _process(delta):
var target_clamped = apply_zone_limits(target_position)  # F5
camera.position = camera.position.lerp(target_clamped, smoothing_speed * delta)
```

- `smoothing_speed` is a variable parameter: 6.0 (explore), 9.0 (combat), interpolated via F3
- Manual lerp gives full control over the interpolation curve and speed
- Allows smooth transitions without disabling/re-enabling the feature

---

## Rationale

1. **Dynamic smoothing speed**: F3 (Transición de Modo) can smoothly interpolate `smoothing_speed` from 6.0 to 9.0 over 0.4s. The manual lerp respects this changing parameter every frame.

2. **No snapping**: The lerp is continuous. Even if `smoothing_speed` changes mid-interpolation (e.g., rapid combat entry/exit), the camera position follows the current target without jumps.

3. **Predictable, debuggable**: Manual lerp is simple to trace in logs and profilers. The speed parameter is explicit in code, not hidden in engine internals.

4. **MVP scope**: For a 60 fps target, the math is trivial (1 lerp per frame), no performance penalty.

---

## Consequences

### Positive
- ✅ Smooth exploration↔combat transitions without snapping
- ✅ Camera never teleports or jitters during mode changes
- ✅ Full control over interpolation curve and speed
- ✅ Easy to debug: speed parameter is visible in code

### Negative
- ❌ Manual lerp is developer responsibility — must disable `position_smoothing_enabled` explicitly in project settings
- ❌ If `position_smoothing_enabled` is accidentally left on, will conflict (double-smoothing)

### Mitigations
- Add a code comment in the camera controller: `# NOTE: position_smoothing_enabled must be OFF in Camera2D node properties`
- Include in project CLAUDE.md as a known constraint
- Add a test assertion: detect if native smoothing is active and warn in logs

---

## Related Decisions
- GDD #9 (Camera): Rule R2, Formulas F2 and F3 depend on this decision
- ADR-002 (EventBus Global Signals): Camera receives `combat_started`/`combat_ended` signals that trigger F3 transitions

---

## Implementation Checklist
- [ ] Disable `position_smoothing_enabled` on all Camera2D instances
- [ ] Implement lerp in camera controller `_process(delta)`
- [ ] Add warning if native smoothing is detected at runtime
- [ ] Document constraint in CLAUDE.md
- [ ] Add unit tests for smoothing speed interpolation (F3)
