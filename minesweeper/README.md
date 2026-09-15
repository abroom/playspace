# Minesweeper

A single-file browser game. Open `minesweeper.html` directly in a browser —
no build step, no dependencies, no server required.

## Concept

Classic Minesweeper: reveal every square that isn't a mine. A revealed square
shows a number if any of its 8 neighbors are mines; reveal a mine and it's
game over. The first square you reveal is always safe — mines are only
placed after that click, excluding it and its neighbors.

## Board sizes

Chosen from a "Choose a size" dialog opened by the New game button (matches
classic Beginner/Intermediate/Expert dimensions):

| Size   | Grid   | Mines |
|--------|--------|-------|
| Small  | 9×9    | 10    |
| Medium | 16×16  | 40    |
| Big    | 30×16  | 99    |

Small and Medium use a fixed cell size. On non-mobile screens (`min-width:
561px` and `min-height: 481px`), Big instead fills the width of the play area
with fluid `1fr` grid columns and `aspect-ratio: 1/1` cells (via a
container-query `cqw` font-size, so the numbers/flags scale with it) — at a
fixed 22px per cell it looked lost on a wide desktop screen. Below that
breakpoint it falls back to the fixed size and scrolls horizontally instead.

## Interaction model

- **Click** a hidden square to reveal it (flood-fills connected zero-count
  squares automatically).
- **Right-click** a hidden square to cycle it: blank → 🚩 flagged →
  ❓ unsure → blank.
- **Keyboard**: arrows or WASD move a single cursor around the grid
  (`cursor` variable, not native per-cell focus — there can be hundreds of
  cells on the Big board), Space reveals the cursor's square, F cycles its
  flag state.
- Mouse and keyboard share the same cursor position, so switching between
  them mid-game stays consistent.
- **R** and the smiley face both restart with the *same* size currently being
  played. If a game is actually in progress (`gameState === 'playing'`), they
  prompt for confirmation first; otherwise there's nothing to lose, so it
  restarts immediately.
- The **New game** button is separate: it always opens the "Choose a size"
  dialog directly, with no confirmation step — deliberately picking a size
  there is the friction, even if a game's already in progress.
- **Escape** doubles as a menu key: it opens the size-picker dialog if nothing
  else is open, or dismisses whatever dialog is currently open. While it's
  open, **1**/**2**/**3** pick Small/Medium/Big directly. Opening the dialog
  this way also pauses the game (see **Pause** below) if it wasn't paused
  already, via a `pausedByMenu` flag; closing the dialog by any route (Escape,
  Cancel, the backdrop) resumes it again in that case, but leaves a pause you
  set yourself with **P** alone.
- All custom keyboard shortcuts ignore Cmd/Ctrl/Alt combinations, so browser
  shortcuts like Cmd+R (reload) or Cmd+F (find) still work normally.
- **H** is a hint (see below), **P** pauses, and once the game is lost, **C**
  continues and **V** reveals.

## On loss

Only the mine you actually clicked reveals (red, `.exploded`) — the rest of
the field stays hidden until you choose to see it (see **Reveal** below).

## Hint, Reveal, and Continue

Status-bar squares sit alongside the mine counter and timer: the timer is now
white, a yellow one tallies hints used, a blue one tallies continues used
(`hint-count`/`continue-count`, kept in sync by `updateStatsDisplay()`). The
💡 button between them is the hint control.

- **Hint** (💡 or **H**) reveals a random hidden, unflagged, non-mine square
  for you — before mines are placed, any hidden square qualifies, since
  `reveal()` guarantees the first click is always safe. Disabled once the
  game is won, lost, or paused.
- On loss, the banner offers two buttons:
  - **Reveal** (or **V**) shows every mine (a correctly-flagged one green,
    `.correct-flag`; a square you flagged that *wasn't* a mine, yellow,
    `.false-flag`) and disables itself once used — `boardRevealed` gates the
    false-flag styling so it only appears after this button is pressed.
  - **Continue** (or **C**) turns the mine you hit into a flag and un-reveals
    every other mine (whether or not Reveal had been used), putting the board
    back to exactly how it looked the instant before you hit it, then resumes
    the timer and play. You can hit another mine later and continue again.
- Hints and continues are tallied per game (`hintsUsed`, `continuesUsed`) and
  shown on whichever banner appears, in addition to the always-visible status
  bar counters — reset on **New game**/restart.

## Pause

The ⏱️ button (or **P**) toggles `paused`: it stops the timer, blurs the
board behind a `backdrop-filter` overlay (`#board-overlay`), and blocks every
other shortcut except P itself, so there's no way to sneak a peek or a move
in while paused. Disabled once the game is won or lost, since there's
nothing left to pause.

## Known intentional non-features

- No chording (clicking a satisfied number to auto-reveal its remaining
  neighbors) — not implemented, kept out of scope.
