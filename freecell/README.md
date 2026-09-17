# FreeCell

A single-file browser game. Open `freecell.html` directly in a browser —
no build step, no dependencies, no server required.

## Concept

All 52 cards deal face-up across 8 tableau columns (4 columns of 7, 4 of 6)
— there's no hidden information, so every FreeCell deal is theoretically
solvable with perfect play. 4 free cells each hold a single card as
temporary parking. Build tableau runs down in alternating colors — unlike
Klondike, *any* card can go on an empty column, not just a King. Build the
four foundations up by suit, Ace to King, to win.

## Interaction model

Click-based only, sharing the same pick-up/click-destination pattern as
Klondike:

- Click a face-up card to pick it up — either just that card, or the whole
  run above it if `idx..end` of its column is a valid alternating-color,
  descending sequence (`isValidRun`). Click a destination (a tableau column,
  a free cell, or a foundation) to drop it there. Click the same card again
  to put it back down; clicking a different valid card switches the
  selection instead of attempting a move.
- **Supermoves**: moving more than one card at once is only allowed onto a
  tableau column, and is capped by `maxSupermove()` — the classic FreeCell
  formula `(1 + empty free cells) × 2^(empty tableau columns)`, where an
  empty *destination* column doesn't count toward its own capacity (you're
  filling it, not routing through it). This simulates shuttling cards
  through empty cells/columns one at a time without actually animating each
  intermediate step.
- Cards can also move **from** a foundation back onto the tableau or a free
  cell, same as Klondike allows.
- **Undo** any move, any number of times — same whole-state-snapshot
  approach as Klondike (`snapshot()`/`pushHistory()`, `JSON.stringify` of
  `{freeCells, foundations, tableau, moves}`).
- **Give up** ends the current deal (with a confirmation first) and shows
  the same end-of-game screen a win does, just with a different heading.
- **Escape** puts down a card you're holding, if any; otherwise it starts a
  new game (prompting for confirmation first if the deal is underway), or
  closes whatever's open. **U** undoes. **Backspace** or **B** goes back to
  the Playspace index — prompting for confirmation first if the deal is
  underway (`gameUnderway()`: `gameState === 'playing' && moves > 0`),
  otherwise it leaves immediately.
- All custom keyboard shortcuts ignore Cmd/Ctrl/Alt combinations, so browser
  shortcuts still work normally.

Unlike Klondike, "New game" has no draw-mode picker to choose from — there's
only one way to deal FreeCell, so it goes straight from the confirmation (if
needed) to a fresh shuffle, the same way Hearts' "New game" does.

## End of hand

Winning (all 52 cards on the foundations) or giving up both open the same
"Game over" panel, just with a different heading and note. Either way it
shows **undos used** and **cards remaining** (0 on a genuine win).

## Mobile

Card size, tableau cascade offset, and the pile gap are all CSS custom
properties (`--card-w`, `--card-h`, `--cascade-offset`, `--pile-gap`),
redefined under `max-width: 560px` so all 8 columns reflow to a single set
of smaller numbers instead of needing per-element breakpoint rules.
Verified down to a 320px-wide viewport with no horizontal overflow — 8
columns are tighter than Klondike's 7, so the mobile card size here is
smaller. Hasn't had a multi-round mobile tuning pass, so further
breakpoints (landscape/short phones especially) may need follow-up if
issues turn up.

## Known intentional non-features

- No drag-and-drop — click/tap only, matching Klondike and Hearts'
  precedent for card games in this repo.
- No double-click/tap "auto-send to foundation" shortcut.
- No keyboard cursor for picking cards — only U (undo), Escape, and
  Backspace/B are wired up.
- No alternate rule variants (fewer free cells, 10-column deals, etc.) —
  just the standard 4-cell, 8-column game.
