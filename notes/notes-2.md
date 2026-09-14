# Notes 2 — Combat design decisions

Date: 2026-07-29
Follows: [notes-1.md](notes-1.md) (engine choice, project roadmap, UI approach)

Scope: the design decisions that are load-bearing on architecture. Everything not listed
here is deliberately deferred until the prototype is playable — abilities, classes,
enemy types, story, all balancing.

---

## Decisions (locked)

### Action economy

- **2 actions per unit per turn.**
- **Movement is an action.** A move action lets the unit travel up to its movement range.
- **Move + move is allowed** — effectively a "dash" for double distance at the cost of
  the whole turn.
- **Partial movement does not refund.** Move 2 of 4 tiles and the action is spent.
  Simpler to reason about and to display than tracking leftover movement.
- **`max_actions` is a stat, not a constant.** Artifacts and abilities may grant a third
  action. Specifics unknown; the model must absorb it without special-casing.

### Ability restrictions

Per-ability, data-driven. Each ability declares its own cost and limits:

- `action_cost` — may be 0, 1, or 2 (a heavy ability can consume the whole turn)
- `uses_per_turn` — default 1, `-1` for unlimited
- `cooldown_turns` — multi-turn cooldowns expected; specifics unknown

**Cooldown ticking:** decrement at the start of the unit's *own* turn, not per round.
Consequence — a high-Speed unit acting more often also recovers abilities faster. That
is intended: Speed becomes a genuinely valuable stat rather than just turn priority.

### Double attack is permitted

A unit that doesn't move may attack twice (subject to the ability's own
`uses_per_turn`). **The counterbalance is positional risk**: staying put means staying
exposed if you're out of cover or out of position as the battle evolves.

> **This makes cover a core system, not an add-on.** See Derived requirements.

### Turn order

**Individual initiative queue.** One shared queue interleaving both sides, ordered by a
Speed stat. Not team-based phases.

- **Initiative is deterministic** — derived from Speed with a stable tiebreak, no roll.
  Randomness belongs in combat rolls where it creates drama, not in turn order where it
  just feels arbitrary.
- **The queue must support mid-battle insertion** — wave encounters spawn units into an
  in-progress battle. Build for insertion from the start; retrofitting is painful.

### Determinism

**Probabilistic.** Hit chance percentages plus damage ranges.

- Requires a **seeded RNG owned by `BattleState`** — not global `randi()`. Needed for
  replays, for save/load mid-battle, and for AI lookahead that doesn't burn the real
  random stream.
- Damage preview UI shows hit chance *and* damage range, not a single number.

### Battle scale

- **Player side: 3-8 units.** Variable per encounter.
- **Enemy side: variable by encounter type.** Three shapes wanted:
  - single large unit (boss) — occupies multiple tiles
  - waves of smaller units — spawn mid-battle
  - even fights — comparable numbers

---

## Derived requirements

These follow from the decisions above and are not optional.

### 1. Cover system — in the vertical slice

Double-attack is only balanced if position carries real risk. Without cover, standing
still and attacking twice is strictly optimal and movement becomes dead weight.

- Tiles carry a cover value (none / half / full)
- Hit chance computed from attacker position, target position, and **which direction the
  cover faces** — directional, not a flat defense bonus
- Directional is more work, but it's what makes "reposition or double-tap?" a real
  decision

**Caveat:** the step-6 placeholder AI (walk toward nearest, attack if adjacent) will not
exercise cover at all. Expect the system to look pointless until the AI can advance,
flank, and punish static units. Don't conclude it's broken.

### 2. Enemy AI must apply positional pressure

The whole loop depends on enemies making a static unit regret it. Placeholder AI is fine
for steps 1-7, but "AI that pressures position" is the first real milestone after the
slice.

### 3. Multi-tile unit footprints — design now, build later

A 2x2 boss touches flood fill, adjacency, targeting, cover evaluation, and pathing.

**Approach:** put a `footprint` field on units immediately, use only 1x1 in the slice.
The grid code gets written with the concept present, so enabling large units later is a
feature rather than a refactor.

### 4. Queue lookahead UI is load-bearing

Initiative queue + double attack + percentage rolls compounds variance considerably — a
fast enemy can act twice before you respond, and both attacks roll. Players need to see
the next several turns to plan around it.

In team-based rounds a turn tracker is a nicety. Here it's essential.

With 3-8 player units plus variable enemies, the queue can hold 20+ entries — the
display needs truncation ("next 8, +12 more") rather than showing everything.

### 5. Pannable camera — from the start

At 32px tiles and 3x scale on 1080p, roughly **20 tiles wide by 11 tall** are visible.
Maps up to ~18x10 fit with no scrolling — fine for 3v3, tight for 8v8 plus a wave.

Given variable encounter sizes, build camera panning in immediately. Trivial in Godot,
and definitely needed.

### 6. Pre-battle deployment screen

3-8 player units implies choosing a squad and placing it before the fight. Noted as a UI
requirement; not built yet.

---

## Data model

Authored as Godot `Resource` subclasses — inspector-editable `.tres` files, diffable in
git, no serialization code. See notes-1.md "Data layer".

```gdscript
class_name UnitStats extends Resource
@export var display_name: String
@export var max_hp: int = 20
@export var move_range: int = 4          # tiles per single move action
@export var base_actions: int = 2        # artifacts may raise this
@export var speed: int = 10              # initiative order, deterministic
@export var aim: int = 75                # base hit chance contribution
@export var footprint: Vector2i = Vector2i.ONE   # 2x2 for large units
@export var abilities: Array[Ability]
```

```gdscript
class_name Ability extends Resource
@export var display_name: String
@export var action_cost: int = 1         # 0, 1, or 2
@export var uses_per_turn: int = 1       # -1 = unlimited
@export var cooldown_turns: int = 0      # ticks on the unit's own turn
@export var range: int = 1
@export var damage_min: int = 4
@export var damage_max: int = 6
```

Runtime state per unit — lives in `BattleState`, **not** on nodes (see notes-1.md
"Architecture: separate logic from nodes"):

```
actions_remaining: int
uses_this_turn:    Dictionary   # ability -> count used
cooldowns:         Dictionary   # ability -> turns remaining
grid_position:     Vector2i
```

`BattleState` also owns:

```
rng:            RandomNumberGenerator   # seeded, not global
initiative:     queue supporting mid-battle insertion
grid:           tile cover data + occupancy (footprint-aware)
```

---

## Revised combat build order

Supersedes the 7-step list in notes-1.md. Steps 4 and 5 are heavier than originally
scoped; cover moves earlier because it's core.

1. **Grid** — `TileMapLayer` + `local_to_map()` / `map_to_local()`. Click a tile, print coords.
2. **Units on the grid** — occupancy model with `footprint` (1x1 only for now). Click to select, highlight.
3. **Movement** — BFS flood fill to `move_range`, highlight reachable tiles, click to move. Spends an action.
4. **Initiative queue** — deterministic order from Speed, stable tiebreak, insertion-capable. Action pips per unit.
5. **Cover data** — tiles carry cover level; directional evaluation against attacker position.
6. **Attack** — hit chance from aim + cover + range; damage roll from seeded RNG; damage preview UI before commit.
7. **Ability framework** — `action_cost`, `uses_per_turn`, `cooldown_turns` enforced.
8. **Placeholder AI** — move toward nearest, attack if in range. Known not to exercise cover.
9. **Win/lose** — one side eliminated.

Then, in rough order: AI with positional pressure → wave spawning → multi-tile units →
pre-battle deployment.

---

## UI additions

Beyond the list in notes-1.md:

- **Action pips** per unit — `●●` / `●○` / `○○`, on the unit and in the info panel.
  Required rather than decorative, since `max_actions` varies with artifacts.
- **Initiative queue strip** — next N turns, truncated, both sides visually distinct.
- **Damage preview** — hit chance % *and* damage range, plus the cover modifier that
  produced it. Players need to see *why* the number is what it is.
- **Cover indicators** on tiles, directional.
- **Cooldown display** on ability buttons — turns remaining.

---

## Still open (deliberately)

- Whether progress persists across battles, or the game is run-based / roguelike.
  Affects `GameState` and saves; not the combat core. Safe to defer past the slice.
- Specific abilities, classes, artifacts.
- Enemy roster and encounter design.
- All numbers — HP, damage, aim, cover modifiers, Speed values.
- Terrain effects beyond cover (elevation, hazards, destructibility).
- Whether facing/flanking matters beyond cover direction.
- Story, setting, characters.
