# Solitaire

A single-file browser game. Open `solitaire.html` directly in a browser —
no build step, no dependencies, no server required.

## Concept

Classic Klondike. Seven tableau columns are dealt 1–7 cards deep, only the
top of each face-up. Build tableau runs down in alternating colors (an empty
column only accepts a King), and build the four foundations up by suit from
Ace to King. Clear all 52 cards onto the foundations to win.

## Draw modes

Chosen from a "Choose a draw" dialog opened by the New game button (same
pattern as the other games' size/deal pickers):

- **Draw 1** — the stock turns one card at a time into the waste.
- **Draw 3** — the stock turns three at a time; only the top (most recently
  turned) waste card is ever playable, the other two are just a visual fan.

When the stock runs out, click it to flip the waste back over and keep
drawing — there's no limit on redeals.

## Interaction model

Click-based only (no drag-and-drop or keyboard card navigation — see
**Known intentional non-features**):

- Click a face-up card to pick it up — either just that card, or the whole
  run above it if `idx..end` of its column is a valid alternating-color,
  descending sequence (`isValidRun`). Click a destination pile (a tableau
  column, or a foundation) to drop it there. Click the same card again to
  put it back down; clicking a different valid card switches the selection
  instead of attempting a move.
- Cards can also move **from** a foundation back onto the tableau (classic
  Klondike allows this), not just onto foundations.
- Click the stock to draw, or to recycle the waste once it's empty.
- **Undo** any move, any number of times — each one increments the tally but
  is never itself undoable. Uses the same whole-state-snapshot approach as
  Skyscraper Farm's undo (`snapshot()`/`pushHistory()`, `JSON.stringify` of
  `{stock, waste, foundations, tableau, moves}`), which is simpler and far
  less bug-prone here than hand-writing an inverse for every move type.
- **Give up** ends the current hand (with a confirmation first) and shows
  the same end-of-game screen a win does, just with a different heading.
- **Escape** puts down a card you're holding, if any; otherwise it opens the
  New game menu (prompting for confirmation first if the hand is underway),
  or closes whatever's open. **U** undoes. **Backspace** or **B** goes back
  to the Playspace index — prompting for confirmation first if the hand is
  underway (`gameUnderway()`: `gameState === 'playing' && moves > 0`),
  otherwise it leaves immediately.
- All custom keyboard shortcuts ignore Cmd/Ctrl/Alt combinations, so browser
  shortcuts still work normally.

## End of hand

Winning (all 52 cards on the foundations) or giving up both open the same
"Game over" panel, just with a different heading and note. Either way it
shows **undos used** and **cards remaining** (0 on a genuine win) — the two
numbers asked for regardless of how the hand ended.

## Mobile

Card size, the tableau cascade offset, and the waste fan offset are all CSS
custom properties (`--card-w`, `--card-h`, `--cascade-offset`,
`--fan-offset`), redefined under `max-width: 560px` so the whole board
(all 7 columns) reflows to a single set of smaller numbers instead of
needing per-element breakpoint rules. Verified down to a 375px-wide
viewport with no horizontal overflow; hasn't had the same multi-round
mobile tuning pass the other games eventually got, so further breakpoints
(landscape/short phones especially) may need follow-up if issues turn up.

## Known intentional non-features

- No drag-and-drop — click/tap only, matching Hearts' precedent for card
  games in this repo (unlike Skyscraper Farm's drag-and-drop, which is a
  block-sorting game, not a card game).
- No double-click/tap "auto-send to foundation" shortcut — every foundation
  move is still just as reachable via the normal pick-up/click-destination
  flow, this just skips the extra convenience of auto-detecting it.
- No keyboard cursor for picking cards (unlike Hearts' arrow-key hand
  cursor) — only U (undo), Escape, and Backspace/B are wired up.
- No redeal limit and no scoring system (Vegas or standard) — redeals are
  unlimited and the only counters are moves, undos, and cards remaining.
