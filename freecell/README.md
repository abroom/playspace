# FreeCell

A single-file browser game. Open `freecell.html` directly in a browser.
No build step, no dependencies, no server required.

## Concept

All 52 cards deal face-up across 8 tableau columns (4 columns of 7, 4 of 6).
There's no hidden information, so every FreeCell deal is theoretically
solvable with perfect play. 4 free cells each hold a single card as
temporary parking. Build tableau runs down in alternating colors; unlike
Klondike, *any* card can go on an empty column, not just a King. Build the
four foundations up by suit, Ace to King, to win.

## Interaction model

Both click/tap and drag-and-drop work, sharing the same
pick-up/attempt-a-move pattern as Klondike (see its README for the full
drag/ghost/cleanup writeup, identical implementation here):

- Click (or press and drag) a face-up card to pick it up, either just that
  card, or the whole run above it if `idx..end` of its column is a valid
  alternating-color, descending sequence (`isValidRun`). Click/drop a
  destination (a tableau column, a free cell, or a foundation) to move it
  there. Click the same card again to put it back down; clicking a
  different valid card switches the selection instead of attempting a move.
- **Supermoves**: moving more than one card at once is only allowed onto a
  tableau column, and is capped by `maxSupermove()`, the classic FreeCell
  formula `(1 + empty free cells) × 2^(empty tableau columns)`, where an
  empty *destination* column doesn't count toward its own capacity (you're
  filling it, not routing through it). This simulates shuttling cards
  through empty cells/columns one at a time without actually animating each
  intermediate step.
- Cards can also move **from** a foundation back onto the tableau or a free
  cell, same as Klondike allows.
- **Undo** any move, any number of times. Same whole-state-snapshot
  approach as Klondike (`snapshot()`/`pushHistory()`, `JSON.stringify` of
  `{freeCells, foundations, tableau, moves}`).
- **Give up** ends the current deal (with a confirmation first) and shows
  the same end-of-game screen a win does, just with a different heading.
- **Escape** puts down a card you're holding, if any; otherwise it starts a
  new game (prompting for confirmation first if the deal is underway), or
  closes whatever's open. **U** undoes. **Backspace** or **B** goes back to
  the Playspace index, prompting for confirmation first if the deal is
  underway (`gameUnderway()`: `gameState === 'playing' && moves > 0`),
  otherwise it leaves immediately.
- All custom keyboard shortcuts ignore Cmd/Ctrl/Alt combinations, so browser
  shortcuts still work normally.

Unlike Klondike, "New game" has no draw-mode picker to choose from. There's
only one way to deal FreeCell, so it goes straight from the confirmation (if
needed) to a fresh shuffle, the same way Hearts' "New game" does.

## Keyboard cursor

Identical mechanism to Klondike (see its README for the full writeup): a
cursor lives in either the free cells/foundations row or the tableau, each
remembering its own last position; **&larr;**/**&rarr;** move within the
current row, **&uarr;**/**&darr;** switch rows, and **Space** does exactly
what a click at the cursor would do, including triggering a supermove if
the cursor's sitting on a multi-card run and there's room for it.

## End of hand

Winning (all 52 cards on the foundations) or giving up both open the same
"Game over" panel, just with a different heading and note. Either way it
shows **undos used** and **cards remaining** (0 on a genuine win).

## Responsive board (mobile and desktop)

Card size, tableau cascade offset, and the pile gap are all CSS custom
properties (`--card-w`, `--card-h`, `--cascade-offset`, `--pile-gap`)
computed from `.table`'s own width via a container query (same technique as
Klondike: `container-type: inline-size` on `.table`, `cqw`-based `clamp()`
math on its direct children `.row-top`/`.tableau`). The board fills however
wide the felt actually is at any screen size instead of sitting at one
small fixed card size. Verified from a 320px-wide viewport (8 columns are
tighter than Klondike's 7, so the floor here is a touch smaller) up through
a ~1450px-wide desktop window with no horizontal overflow at either end.

## Double-click

Double-clicking a single card (`tryAutoMove()`) sends it wherever it best
fits, in order: its own foundation, then any other legal tableau spot, then
an empty free cell as a last resort. Doesn't apply to a card that's part of
a bigger run, only ever a single card. Identical implementation to
Klondike (see its README for the full writeup).

## Drop-anywhere foundations and free cells

Dropping a card on *any* of the four foundation slots redirects it to the
one matching its own suit, and dropping on an occupied free cell redirects
to any other open one (`resolveDestination()`). Free cells are otherwise
interchangeable, so which one you happened to drop on isn't meaningful.
Identical mechanism to Klondike (see its README for the full writeup,
including why the drag hover highlight redoes the same resolution).

## Early win recognition

Every FreeCell card is visible from the first deal, so unlike Klondike
there's no "nothing hidden" signal to key off of. Instead, `checkEarlyWin()`
(checked at the end of every `render()`) runs `wouldGreedyClear()`: a
simulation on a scratch copy of the board that repeatedly sends any
tableau or free-cell card that's exactly next-in-sequence for its suit,
without touching the real game state. If that clears the whole tableau,
the position is recognized as won immediately, however many cards are
still actually sitting off the foundations. Nothing gets auto-moved for
you; the board is left exactly as it is, and you're free to keep placing
cards by hand (the win is already recorded either way, so `gameState` is
`'won'` and the summary screen shows 0 cards remaining regardless of what's
actually still on the board at that moment).

## Known intentional non-features

- No alternate rule variants (fewer free cells, 10-column deals, etc.).
  Just the standard 4-cell, 8-column game.
