# Minesweeper

A single-file browser game. Open `minesweeper.html` directly in a browser —
no build step, no dependencies, no server required.

## Concept

Classic Minesweeper: reveal every square that isn't a mine. A revealed square
shows a number if any of its 8 neighbors are mines; reveal a mine and it's
game over. The first square you reveal is always safe — mines are only
placed after that click, excluding it and its neighbors.

## Board sizes

Chosen via the Small/Medium/Big buttons (matches classic Beginner/
Intermediate/Expert dimensions):

| Size   | Grid   | Mines |
|--------|--------|-------|
| Small  | 9×9    | 10    |
| Medium | 16×16  | 40    |
| Big    | 30×16  | 99    |

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
- **R** or the smiley face button both start a new game (same size). If a
  game is actually in progress (`gameState === 'playing'`), either one
  prompts for confirmation first, so a stray press/click can't wipe out
  progress — before the first reveal, or once the game's already won/lost,
  there's nothing to lose so it just restarts immediately.

## On loss

All mines reveal. A mine you'd correctly flagged beforehand shows a green
background (`.correct-flag`) instead of blending in with the rest — everything
else unflagged just shows the plain 💣 styling, and the one you actually
clicked shows red (`.exploded`).

## Known intentional non-features

- No chording (clicking a satisfied number to auto-reveal its remaining
  neighbors) — not implemented, kept out of scope.
- No incorrect-flag marking (an X over a wrongly-flagged non-mine) on loss —
  only the correct-flag highlight was asked for.
