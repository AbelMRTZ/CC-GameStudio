---
name: feedback-test-infrastructure
description: NPC dialogue unit tests require a WorldStateMock that injects npc_encounters, corruption_level, and reputation — do not test against live WorldState
metadata:
  type: feedback
---

Unit tests for the NPC & Dialogue system (GDD #15) must use a `WorldStateMock` that allows direct injection of `npc_encounters`, `corruption_level`, and `reputation` values — they must NOT depend on a live `WorldState` instance or file I/O.

**Why:** WorldState has save/load side effects and requires full game initialization. Unit tests for branch selection logic (R3) and formulas (F1, F2) are pure logic — they only need the relevant fields. Depending on live WorldState would make tests non-isolated and slow.

**How to apply:** When writing or reviewing unit tests in `tests/unit/npc_dialogue/`, check that each test constructs a mock/stub for WorldState rather than instantiating the real class. If a test requires loading a scene or reading a file, it is an integration test — move it to `tests/integration/npc_dialogue/`.

Related: [[project-gdd15-acs]]
