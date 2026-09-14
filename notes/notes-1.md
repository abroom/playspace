# Notes 1 — Engine choice, project roadmap, UI approach

Date: 2026-07-29

## The game

- 2D pixel art. Characters at 32x32 or 64x64.
- Turn-based grid combat.
- Overworld is a set of static scenes with dialogue options. Travel by clicking a
  location and instantly entering it (new background, new scene). No free-roaming
  overworld movement.

Implication: no physics, no real-time movement, no camera work, no netcode. The
technical demands are low. The hard parts are combat design, data authoring, and UI.

---

## Decision: Godot 4

### Why it fits

1. **Crisp pixel rendering** — three project settings and 32x32 sprites look correct
   at any window size:
   - `Rendering > Textures > Default Texture Filter = Nearest`
   - `Display > Window > Stretch Mode = canvas_items`
   - `Display > Window > Stretch Aspect = keep`
2. **Grid math is free** — `TileMapLayer` gives `local_to_map()` / `map_to_local()`
   for tile <-> world coordinate conversion. The combat logic itself (turn order,
   movement range, targeting) is code we'd write in any engine, so no engine wins there.
3. **Scene swapping matches the overworld design exactly** — each location is a
   `.tscn` with a background `TextureRect` plus clickable hotspots.
   `change_scene_to_file()` and done. A more simulation-oriented engine would fight this.
4. **Data authoring via custom `Resource` classes** — the highest-leverage feature for
   an RPG. See "Data layer" below.
5. **Dialogue** — the Dialogue Manager addon (Nathan Hoad) handles branching,
   conditions, and variable state. Half the game is dialogue; not writing this from
   scratch matters.
   https://github.com/nathanhoad/godot_dialogue_manager

Also: MIT licensed, no royalties or seat fees ever, ~150MB editor, exports to desktop
and web.

### Native 2D matters

Unity and Unreal are 3D engines with 2D layered on — you work in world units with an
orthographic camera. Godot has a *separate* native 2D renderer, so you work in real
pixel coordinates. `position = Vector2(64, 32)` means 64px right, 32px down. Removes a
whole category of blurry-sprite / wrong-scale / half-unit-offset problems.

### Known limitations (none of which hit this project hard)

- **Console porting needs a third party** (W4 Games, Pineapple Works) — platform SDKs
  are under NDA and can't ship in an MIT codebase. Costs money and time.
- Smaller asset store than Unity; more gets written by hand.
- 3D is real but behind Unity/Unreal. Irrelevant here.
- **UI system is powerful but unintuitive.** This is the one that does affect us —
  see the UI section.

---

## Alternatives considered

### Unity
Viable. Would win if console ports (especially Switch) are a firm goal — certification
is a documented path there, unlike Godot. Also the right call if we already had deep
C# / Unity experience. Neither applies.

### GameMaker
Fast for 2D, excellent built-in sprite editor. But its strengths (rapid 2D action
prototyping) don't help a turn-based tactics game, and its weaknesses (structured data,
complex UI, larger codebases) hit us directly. GML is also engine-locked — skills don't
transfer out.

Shipped in GML: Undertale, Hyper Light Drifter, Katana ZERO, Nuclear Throne,
Hotline Miami. Clearly capable; the objection is fit and skill portability, not quality.

### Web (Phaser / PixiJS / plain canvas)
Genuinely reasonable — for a turn-based game with static scenes you barely need an
engine, and browser distribution is a real advantage for demos. Would pick this if
TypeScript were the preferred language.

### Unreal — rejected
Bad fit for this art direction, and it fights us on the thing we care most about.

- **The renderer is adversarial to pixel art.** Every frame goes through tonemapping,
  colour grading, bloom, and TAA by default. The tonemapper means the hex value picked
  in Aseprite is *not* what reaches the screen. TAA smears sprite edges on movement.
  Getting 1:1 pixel output is a project-settings excavation, not a checkbox.
- **Paper2D is neglected** — semi-maintained for years; Epic's attention is on Nanite,
  Lumen, MetaHumans. Serious 2D work leans on PaperZD, a community plugin.
- **Weight for nothing** — 50-100GB+ editor install, hundreds of MB for a tiny 2D
  build, slow C++ compiles, shader compilation stutter. We'd use almost none of its
  strengths.

Concedes: UMG is better than Godot's Control nodes, and DataTables handle CSV-imported
stat data well.

**Honest counterexample:** Octopath Traveler and Triangle Strategy are both UE4, and
Triangle Strategy is literally turn-based grid tactics. But HD-2D isn't pixel-perfect
2D — it's sprites in lit 3D dioramas drenched in depth of field, bloom, and tilt-shift,
i.e. exactly the post-processing Unreal exists for. Square Enix wanted the renderer and
had large teams with existing Unreal pipelines.

**So the real decision point:** if we ever want HD-2D (sprites in 3D environments),
Unreal becomes defensible. For flat 2D with hand-drawn backgrounds and crisp 32x32
sprites, it isn't.

---

## Getting started

### Install

```bash
brew install --cask godot          # standard build — GDScript, not .NET/Mono
```

Take the **standard build**. GDScript's iteration speed beats language familiarity
while learning the engine, and every tutorial uses it.

Sprite tool, pick one:
- **Aseprite** ($20, Steam/itch) — industry default, worth it
- **Pixelorama** (free, itself built in Godot) — genuinely good
- **LibreSprite** (free fork of old Aseprite) — fine, less polished

### Strategic call: build combat first, not the overworld

The instinct will be to start with the overworld because it's easier — static
backgrounds, hotspots, dialogue trees. **Don't.**

The overworld is a visual novel. It's low-risk; it will work. Turn-based grid combat is
where the game is either fun or it isn't, and that can't be known from a design doc. If
the combat turns out flat, better to find out in week two than after building forty
dialogue scenes around it.

So: one ugly combat encounter, playable end to end, before anything else. Placeholder
coloured squares instead of sprites. Get it fun, *then* make it pretty, *then* build the
world around it.

### Architecture: separate logic from nodes

The one thing to get right upfront.

The temptation is to make each unit a `Node2D` holding its own HP, position, and
abilities, with units talking to each other directly. That works until enemy AI needs to
evaluate "what happens if I move here and attack that" — and it can't, because the only
representation of game state is the live scene tree.

Instead: a plain `BattleState` class holding grid, unit stats, and turn order as data.
The node tree *renders* that state and *sends commands* to it.

That boundary gets us:
- AI that can look ahead by copying state and testing moves
- Logic unit-testable without launching the game
- Replays and undo, nearly free
- Animation decoupled from rules (attack resolves instantly in logic; visual takes 400ms)

Mildly annoying in week one, a gift for the following year.

### Combat build order

Each step playable before moving on:

1. **Grid** — `TileMapLayer` + `local_to_map()` / `map_to_local()`. Click a tile, print coords.
2. **Units on the grid** — two squares. Click to select, show highlight.
3. **Movement** — BFS flood fill from unit out to `move_range`, highlight reachable tiles, click to move.
4. **Turn manager** — alternate player/enemy, one action per unit, end-turn button.
5. **One attack** — target adjacent unit, subtract HP, remove at zero.
6. **Dumbest possible AI** — step toward nearest player unit, attack if adjacent. Enough to start.
7. **Win/lose** — one side has no units left.

That's a real tactics game. Abilities, terrain, elevation, status effects, and facing are
all iteration on a working core.

### Then the world layer

- `location.tscn` template: background `TextureRect` + clickable `Area2D` hotspots + dialogue trigger
- Dialogue Manager addon
- `GameState` autoload singleton for flags and party roster that survive scene changes

### Data layer

Custom `Resource` classes give inspector-editable `.tres` files:

```gdscript
class_name UnitStats extends Resource
@export var display_name: String
@export var max_hp: int = 20
@export var move_range: int = 4
@export var abilities: Array[Ability]
```

Every unit becomes a small text file — editable through a real UI, diffable in git, zero
serialization code written by us. For a game accumulating hundreds of units, abilities,
and items, this is the biggest win in the engine.

### Learning

- Official docs are unusually good: https://docs.godotengine.org — do
  **"Your first 2D game"** (Dodge the Creeps) before touching our own project. ~2 hours,
  teaches the node/scene model properly.
- GDQuest free material is the best next step.
- **Filter tutorials for "Godot 4."** Godot 4.0 (March 2023) was a large rewrite of 3.x:
  Vulkan renderer, GDScript 2.0 (typed, `@export`, lambdas, `await`), reworked TileMap
  (again in 4.3, to `TileMapLayer`). Anything from 2022 or earlier has API drift.

---

## UI

### Why it feels hard

One insight fixes most of the confusion: **containers position their children, and you
don't.**

If a `Control`'s parent is a container (`VBoxContainer`, `HBoxContainer`,
`GridContainer`, `MarginContainer`), its position and size are computed, not ours to
set. Dragging it in the editor does nothing — that's where everyone's first hour goes.
Control it indirectly via **size flags** (fill / expand / shrink) and **custom minimum
size**.

Anchors are the *other* system — they only apply when the parent is a plain `Control` or
`CanvasLayer`. They pin edges to a fraction of the parent so things survive window
resizing.

- **Containers** for anything list-like or stacked — ability menus, stat rows, inventories
- **Anchors** for top-level placement — "this panel sits bottom-left"

All UI goes under a **`CanvasLayer`**, not in the world. It then ignores the camera
entirely.

### Resolution decision — make this before authoring any art

| Approach | Sprites | UI & text | Feel |
|---|---|---|---|
| **A. Low-res viewport** — base 384x216, stretch `viewport` | Authored 1:1, whole screen integer-scaled | Chunky, tiny, cramped | Purest retro (Celeste, Owlboy) |
| **B. Native-res canvas** — base 1280x720+, stretch `canvas_items` | Authored 1:1, displayed at 3-4x node scale | Crisp, high-res, roomy | Modern pixel (Into the Breach) |
| **C. Hybrid** — world in a `SubViewport`, UI on top | Low-res viewport, integer-scaled | Native res, sharp text | Best of both, more setup |

**Going with B. C is the upgrade path.**

Reason is specific to tactics RPGs: we need to show a *lot* of information — unit stats,
ability descriptions, damage previews, turn order, terrain effects, dialogue. At 384x216
there's room for about six words, and every low-res tactics game fights this constantly.
B keeps a readable interface without losing crisp sprites, since nearest-neighbour at
integer scale is still pixel-perfect.

Gotcha with B: animating movement can land a unit on a fractional pixel and shimmer.
Fix by snapping tween targets to multiples of the scale factor, or enabling
`snap_2d_transforms_to_pixel` in project settings.

### Pixel fonts

A pixel font is a bitmap pretending to be a font — it only looks right at its **native
size or exact integer multiples**. At 13px when drawn for 8px it becomes mush.

In `FontFile` import settings: **disable antialiasing, disable hinting, subpixel
positioning off.** Only use sizes like 8 / 16 / 24 / 32. Set once in a `Theme` and never
think about it again.

### Two nodes to know immediately

- **`NinePatchRect`** — takes a small texture (e.g. 24x24 with a decorative border) and
  stretches only the middle, keeping corners and edges pixel-intact. This is how one
  tiny asset becomes a dialogue box or stat panel at any size. Non-negotiable for pixel
  art UI.
- **`Theme`** resources — define fonts, colours, and panel styles once, apply to the
  whole tree. The alternative is font overrides on 200 individual labels, then being
  asked to change the colour.

### What the game needs

Combat:
- Unit info panel (hovered/selected: portrait, HP, stats)
- Ability bar + ability tooltips
- **Damage / hit-chance preview on target** — the single most important UI in a tactics
  game; players make every decision from it
- Turn order tracker
- End turn button, floating damage numbers

Overworld:
- Dialogue box with speaker name and choice list (Dialogue Manager covers most of this)
- Location navigation hotspots
- Party / roster and inventory screens

### When to build it

**Functional-ugly immediately, styled late.** During combat steps 3-6 a plain `Label`
printing `Knight HP 14/20 | Move 4 | AP 2` is enough — and genuinely necessary to test
anything. Styled panels, NinePatch frames, and transitions come after the mechanics
settle, because restyling is cheap and rebuilding is not.

---

## Next action

Scaffold the project:
- git repo + Godot `.gitignore`
- pixel-perfect render settings (approach B)
- folder structure
- `Theme` resource with pixel font settings pre-configured
- combat scene through build step 3 (grid, selection, movement range highlighting) with
  placeholder squares
- debug combat HUD wired to `BattleState`, unstyled
