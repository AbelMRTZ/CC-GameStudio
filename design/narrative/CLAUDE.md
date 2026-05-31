# Narrative Directory

Story, world-building, and character documents for *Los Encadenados*.
These files are **narrative sources of truth** — GDDs and level designs reference them, not the other way around.

## Structure

```
design/narrative/
├── world/
│   ├── kingdoms/    # One file per kingdom: geography, cities, culture, politics, army, economy, religion
│   └── history/     # Historical events and world-level lore that predate the playable timeline
├── characters/      # One file per named character with a narrative role; minor NPCs grouped in secondary-npcs.md
└── acts/            # Playable story arcs broken down by arc → mission/bridge
```

## What goes where

### `world/kingdoms/`
Describes a kingdom **as it currently exists** at the start of the game.
One file per kingdom, named `[kingdom-slug].md` (e.g. `luxterra.md`).

Each file should cover, where applicable:
- General description and identity
- Geography and climate
- Capital and notable cities
- Population and culture
- Politics and ruling house
- Army and military doctrine
- Economy
- Religion and beliefs
- Active conflicts and recent history (summary — details go in `history/`)

### `world/history/`
Narrates **events that already happened** before the playable timeline begins.
One file per significant event or era, named `[event-slug].md` (e.g. `guerra-civil-luxiana.md`).

Also contains `world-overview.md` — global rules that apply across all kingdoms
(demon-human bonding, the 9 kingdoms structure, shared cosmology, etc.).

### `characters/`
One file per character with an independent narrative role.
Named `[character-slug].md` (e.g. `elian-lyndor.md`).

Each file should cover, where applicable:
- Age and physical description
- Background and origin
- Personality and motivations
- Relationships to other characters
- Demon bond (if any)
- Role in the story arc(s)
- Open questions / unknowns still to be defined

Minor NPCs without their own arc are grouped in `secondary-npcs.md`.

### `acts/`
The playable story timeline, broken down by act → arc → mission/bridge.
One file per act, named `acto-[N]-[kingdom-slug].md` (e.g. `acto-1-luxterra.md`).

Each file describes the narrative flow, key decisions, and emotional beats.
It is the direct input for GDD #16 (Narrative Events System) and for level design.

## Naming conventions

| Type | Pattern | Example |
|------|---------|---------|
| Kingdom | `[slug].md` | `luxterra.md` |
| Historical event | `[event-slug].md` | `guerra-civil-luxiana.md` |
| World overview | `world-overview.md` | — |
| Character (main) | `[firstname-lastname].md` | `elian-lyndor.md` |
| Character (minor NPCs) | `secondary-npcs.md` | — |
| Act | `acto-[N]-[kingdom-slug].md` | `acto-1-luxterra.md` |

## Relationship to GDDs

| Narrative doc | Consumed by |
|---------------|-------------|
| `acts/` | GDD #16 — Narrative Events System |
| `characters/` | GDD #16, level design briefs |
| `world/kingdoms/` | Level design briefs, art briefs |
| `world/history/` | GDD #16, writer reference |

When a GDD lists a narrative document as a dependency, that document must exist
and be marked reviewed before the GDD authoring session begins.
