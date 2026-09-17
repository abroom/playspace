# Klondike

A single-file browser game. Open `klondike.html` directly in a browser.
No build step, no dependencies, no server required.

## Concept

The classic solitaire. Seven tableau columns are dealt 1–7 cards deep, only the
top of each face-up. Build tableau runs down in alternating colors (an empty
column only accepts a King), and build the four foundations up by suit from
Ace to King. Clear all 52 cards onto the foundations to win.

## Draw modes

Chosen from a "Choose a draw" dialog opened by the New game button (same
pattern as the other games' size/deal pickers):

- **Draw 1**: the stock turns one card at a time into the waste.
- **Draw 3**: the stock turns three at a time; only the top (most recently
  turned) waste card is ever playable, the other two are just a visual fan.

When the stock runs out, click it to flip the waste back over and keep
drawing. There's no limit on redeals.

## Interaction model

Both click/tap and drag-and-drop work, sharing one underlying model:

- Click (or press and drag past `DRAG_THRESHOLD`) a face-up card to pick it
  up, either just that card, or the whole run above it if `idx..end` of its
  column is a valid alternating-color, descending sequence (`isValidRun`).
  Click/drop a destination pile (a tableau column, or a foundation) to move
  it there. Click the same card again to put it back down; clicking a
  different valid card switches the selection instead of attempting a move.
- Dragging follows the exact `pointerdown`/`pointermove`/`pointerup`
  pattern Skyscraper Farm pioneered: a floating "ghost" (a full multi-card
  stack for a run) tracks the pointer, and the pile under it gets a
  green/red outline showing whether the drop would be legal
  (`updateDragHover`/`canMove`). Cleanup isn't just on `pointerup`;
  `pointercancel`, the tab going hidden, and window `blur` all abort the
  drag too (see Skyscraper Farm's README for why that matters on mobile),
  and starting a new drag sweeps any stray `.drag-ghost` left over as a
  last-resort safety net.
- Cards can also move **from** a foundation back onto the tableau (classic
  Klondike allows this), not just onto foundations.
- Click the stock to draw, or to recycle the waste once it's empty.
- **Undo** any move, any number of times. Each one increments the tally but
  is never itself undoable. Uses the same whole-state-snapshot approach as
  Skyscraper Farm's undo (`snapshot()`/`pushHistory()`, `JSON.stringify` of
  `{stock, waste, foundations, tableau, moves}`), which is simpler and far
  less bug-prone here than hand-writing an inverse for every move type.
- **Give up** ends the current hand (with a confirmation first) and shows
  the same end-of-game screen a win does, just with a different heading.
- **Escape** puts down a card you're holding, if any; otherwise it opens the
  New game menu (prompting for confirmation first if the hand is underway),
  or closes whatever's open. **U** undoes. **Backspace** or **B** goes back
  to the Playspace index, prompting for confirmation first if the hand is
  underway (`gameUnderway()`: `gameState === 'playing' && moves > 0`),
  otherwise it leaves immediately.
- All custom keyboard shortcuts ignore Cmd/Ctrl/Alt combinations, so browser
  shortcuts still work normally.

## Keyboard cursor

A cursor lives in one of two rows: the stock/waste/foundations strip, or
the tableau, each remembering its own last position independently when you
switch away and back (`TOP_ROW` array + `topRowIdx`/`tableauCol`, the same
"return to where you were" convention Skyscraper Farm uses for its tower and
crate rows). **&larr;**/**&rarr;** move within the current row; **&uarr;**
switches to the top row, **&darr;** back to the tableau. Landing on a
tableau column always puts the cursor at the start of the maximal valid run
sitting at its top (`topRunStart()`), so **Space** (which does exactly what
a click at the cursor would do, via the same `handlePileClick()`) always
grabs (or drops) the biggest run that's actually there.

## End of hand

Winning (all 52 cards on the foundations) or giving up both open the same
"Game over" panel, just with a different heading and note. Either way it
shows **undos used** and **cards remaining** (0 on a genuine win), the two
numbers asked for regardless of how the hand ended.

## Responsive board (mobile and desktop)

Card size, the tableau cascade offset, and the waste fan offset are all CSS
custom properties (`--card-w`, `--card-h`, `--cascade-offset`,
`--fan-offset`) computed from `.table`'s own width via a container query
(`container-type: inline-size` on `.table`, `cqw`-based `clamp()` math on
its two direct children `.row-top`/`.tableau`; has to live one level down,
since an element can't be its own container-query reference). This means
the board fills however wide the felt actually is on any screen size, from
a small phone up to a wide desktop window, instead of sitting at one small
fixed card size with a lot of unused felt around it (which is what a
separate `max-width: 560px` mobile override used to do here). Verified from
a 320px-wide viewport up through a ~1450px-wide desktop window with no
horizontal overflow at either end.

## Double-click

Double-clicking a single card (`tryAutoMove()`) sends it wherever it best
fits: its own foundation first, or failing that any other legal tableau
spot. Doesn't apply to a card that's part of a bigger run, only ever a
single card. The two clicks that make up the double-click already ran
through the normal tap handling by the time `dblclick` fires (pick the card
up, then put it back down since it's the same card again), so this is a
separate, independent action layered on top rather than a special case of
the tap logic.

## Drop-anywhere foundations

Dropping a card on *any* of the four foundation slots redirects it to the
one matching its own suit (`resolveDestination()`), rather than requiring
the exact slot. Foundations are only ever visually distinguished by their
(possibly empty) contents, so there's no reason to demand pixel-perfect
targeting for something that only has one correct destination anyway. This
runs for every input method (click, drag, keyboard Space) since it's inside
`attemptMove()` itself; the drag hover highlight (`updateDragHover()`) redoes
the same resolution before checking `canMove()`, so the green/red preview
matches what actually happens on drop.

## Early win recognition

Once the stock and waste are both empty and every tableau card is face up
(`checkEarlyWin()`, checked at the end of every `render()`), the game is
recognized as won immediately, no matter how many cards are still actually
sitting in the tableau rather than on the foundations. This is the same
assumption every digital Klondike relies on: with nothing hidden and
nothing left to draw, the rest is guaranteed to be just moving cards home
in order, with no real decisions left to make. Nothing gets auto-moved for
you; the board is left exactly as it is, and you're free to keep placing
cards by hand (the win is already recorded either way, so `gameState` is
`'won'` and the summary screen shows 0 cards remaining regardless of what's
actually still sitting in the tableau at that moment).

## Known intentional non-features

- No redeal limit and no scoring system (Vegas or standard). Redeals are
  unlimited and the only counters are moves, undos, and cards remaining.
