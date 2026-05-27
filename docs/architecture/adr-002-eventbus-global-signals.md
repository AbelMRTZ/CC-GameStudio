# ADR-002: EventBus Autoload for Global Signal Distribution

**Status**: Accepted  
**Date**: 2026-05-27  
**Decider**: Abel + Technical Director  
**Affected Systems**: Camera (#9), Combat (#6), World State (#4), Exploration (#8), Audio (#5)

---

## Context

Multiple game systems need to broadcast state changes that other systems listen for, but direct references would create tight coupling:

- **Combat (#6)** needs to signal when combat starts/ends → **Camera (#9)** listens to adjust mode
- **Exploration (#8)** needs to signal zone transitions → **Camera (#9)** listens to freeze/thaw positioning
- **Audio (#5)** needs to signal music/sfx state changes → **World State (#4)**, **Camera (#9)** may listen
- **World State (#4)** emits state events → multiple systems depend on this

Approaches considered:

1. **Direct references**: Each system holds a reference to listeners
   - ❌ Creates circular dependencies (Combat references Camera, Camera references Combat data)
   - ❌ Hard to extend (adding a new listener requires modifying sender code)
   - ❌ Breaks separation of concerns

2. **EventBus Autoload (Event Dispatcher Pattern)**
   - ✅ Centralized signal hub
   - ✅ Senders and listeners are decoupled (don't reference each other)
   - ✅ Easy to add new listeners without modifying sender code
   - ✅ Godot native — uses Godot signals, not a custom event system

---

## Decision

**Create an `EventBus` Autoload (auto-loaded singleton) in the Godot project.**

```gdscript
# res://autoload/event_bus.gd
extends Node

# Combat signals (GDD #6)
signal combat_started
signal combat_ended

# Exploration signals (GDD #8)
signal zona_transition_started
signal zona_transition_ended

# Camera signals (GDD #9, GDD #17)
signal cinematic_started(camera_data: Dictionary)
signal cinematic_ended()

# Audio signals (GDD #5)
signal audio_music_changed(music_key: String)
signal audio_sfx_played(sfx_key: String)

# World state signals (GDD #4)
signal world_state_changed(state_key: String, value: Variant)

# Future signals can be added here
```

**Usage:**

Sender (e.g., Combat system in `_ready()`):
```gdscript
func start_combat():
    EventBus.combat_started.emit()
```

Listener (e.g., Camera in `_ready()`):
```gdscript
func _ready():
    EventBus.combat_started.connect(_on_combat_started)

func _on_combat_started():
    # Transition to FOLLOW_COMBAT mode
```

---

## Rationale

1. **Loose coupling**: Combat doesn't know or care who listens to `combat_started`. Camera doesn't import Combat code. Each system evolves independently.

2. **Godot-native**: Uses Godot's native `Signal` and `.emit()` — no custom event framework needed. Debuggable in the Godot debugger.

3. **Scalable**: New systems can hook into existing signals without modifying senders. Example: if Audio later needs to react to `combat_started`, it just connects a listener in its `_ready()`.

4. **Centralized visibility**: All inter-system signals are defined in one place, making the dependency graph explicit and reviewable.

5. **MVP scope**: EventBus is a single ~20-line script. No performance penalty — native Godot signals are optimized.

---

## Consequences

### Positive
- ✅ Zero coupling between systems (Combat ↔ Camera ↔ Audio)
- ✅ Signals are Godot-native, fully debuggable
- ✅ Easy to add new listeners (just `EventBus.signal_name.connect(callback)`)
- ✅ Signals are "fire-and-forget" — sender doesn't wait for responses
- ✅ All inter-system communication is explicit in EventBus code

### Negative
- ❌ Requires discipline: developers must remember to emit signals, not call listeners directly
- ❌ Signal names must be coordinated across systems (typos cause silent failures)
- ❌ If many signals are emitted per frame, could clutter logs (mitigated by organized signal naming)

### Mitigations
- Document all signals in EventBus code
- Use consistent naming: `[system]_[event]` (e.g., `combat_started`, `zona_transition_ended`)
- Add a debug mode that logs all emitted signals during development
- Code review: enforce that inter-system communication goes through EventBus, not direct references

---

## Related Decisions
- ADR-001 (Manual Camera Smoothing): Camera listens to `combat_started`/`combat_ended` to trigger F3 transitions
- GDD #9 (Camera): Listens to `combat_started`/`combat_ended`, `zona_transition_started`/`zona_transition_ended`, `cinematic_started`/`cinematic_ended` signals
- GDD #6 (Combat): Must emit `combat_started` and `combat_ended` signals. Post-cinematic: must re-emit `combat_started` if combat is still active when `cinematic_ended` fires.
- GDD #8 (Exploration): Must emit `zona_transition_started` and `zona_transition_ended`
- GDD #17 (Cinematics): Must emit `cinematic_started(camera_data)` and `cinematic_ended()` signals. Camera listens and transitions to CINEMATIC mode.
- GDD #5 (Audio): May emit audio-related signals in future

---

## Implementation Checklist
- [ ] Create `res://autoload/event_bus.gd` with initial signals (combat, exploration, camera/cinematics, audio, world state)
- [ ] Register EventBus as Autoload in Godot project settings (AutoLoad tab)
- [ ] Update GDD #6 (Combat) to specify signal emission in Detailed Rules + re-emission contract post-cinematic
- [ ] Update GDD #8 (Exploration) to specify signal emission in Detailed Rules
- [ ] Update GDD #9 (Camera) to reference EventBus signals for combat, exploration, cinematics (completed 2026-05-27)
- [ ] Update GDD #17 (Cinematics) to specify `cinematic_started`/`cinematic_ended` emission and camera_data dictionary schema
- [ ] Update GDD #5 (Audio) to reference EventBus for future audio signal integration
- [ ] Add unit tests: verify signals are emitted and listeners react correctly (test at least one signal chain: combat → camera transition)
- [ ] Document EventBus usage pattern in CLAUDE.md under "Architecture Patterns"
- [ ] Create Godot best practices guide: "Signal usage in multi-system games"
