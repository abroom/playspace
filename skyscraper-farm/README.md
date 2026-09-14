# Skyscraper Farm

A single-file browser puzzle game. Open `skyscraper-farm.html` directly in a
browser — no build step, no dependencies, no server required.

## Concept

Nine towers of 12 crates each get shuffled and dealt face-down (mostly), plus
a 10th empty tower and 2 empty "crates" (temporary holding slots) up top.
Reveal hidden crates, then stack matching animals until every tower is either
empty or full of one matching, fully-revealed animal.

## Terminology (matters for reading the code and copy)

- **Tower** — one of the 10 columns (`.tower` element, `dataset.kind === 'column'`).
- **Crate** — an individual animal tile, whether sitting in a tower or in one
  of the temporary holding slots (`.crate-tile` element).
- **Crate row / moving crates** — the small row of temporary holding slots up
  top (`.free-slot` elements, `dataset.kind === 'free'`). Internally called
  "free cells" in the JS (`freeCells` array), a naming holdover — don't let it
  confuse you, it's the same thing as "crates" in the UI copy.
- A crate only reveals when explicitly clicked/interacted with again after
  being exposed — there's no auto-reveal-on-expose.

## Rules quick reference

- Pick up a tower's revealed top run (all contiguous same-animal revealed
  crates), or reveal its topmost hidden crate.
- Drop onto an empty tower/crate, a matching revealed top, or a hidden top
  (burying it further). A run gets trimmed to fit if there isn't enough room.
- Crates (holding slots) always require an empty slot — no hot-swapping was
  intentionally cut after being tried.
- Deal types (chosen at "New farm"): **Basic** (only the top crate starts
  revealed), **More** (top 1–3), **Mixed** (top + 1–2 more at random depth).

## Scoring

`Cost = moves + (100 × crates used) + (10 × undos used)`. Lower is better.
Crates and undos are deliberately much more expensive than a plain move.

## Interaction model

Three input methods, all funneling into the same `attemptMove`/`handleClick`
logic:

- **Tap**: click once to pick up, click again to drop (or to reveal, if the
  target's top is hidden).
- **Drag**: press and drag past a small threshold (`DRAG_THRESHOLD`) to pick
  up and drop in one gesture — implemented with `pointerdown`/`pointermove`/
  `pointerup`, not the native HTML5 drag-and-drop API.
- **Keyboard**: number keys jump straight to a tower (`0` = tower 10), arrows
  or WASD move focus, Space/Enter act, Backspace undoes. Shift+arrow jumps 3
  towers, or — if something's picked up — straight to the next tower it could
  legally drop onto (`findEligibleTower`). The tower row and crate row each
  remember their own last-focused position independently (`lastTowerIdx` /
  `lastFreeIdx`), so switching rows and back returns you where you were.

Every render fully rebuilds the DOM (`render()` clears and redraws `#yard` and
`#free-row`), which drops native focus — `focusTarget` tracks where it should
land afterward and `render()` restores it at the end.

## Known intentional non-features

- No solvability guarantee on shuffle (plain Fisher-Yates via a CSPRNG,
  `crypto.getRandomValues`) — like Spider Solitaire, not every deal is
  guaranteed winnable.
- No animal-crate re-theming beyond the tower/crate/animal terminology above
  — the "Skyscraper Farm" name is flavor, not a full visual reskin.
