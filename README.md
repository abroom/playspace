# Playspace

A personal scratch space for small side projects and misc notes.

## Contents

- **[`skyscraper-farm/`](skyscraper-farm/)** — a single-file browser puzzle
  game (tower/crate sorting). See its own README for details.
- **[`minesweeper/`](minesweeper/)** — a single-file classic Minesweeper.
  See its own README for details.
- **[`hearts/`](hearts/)** — a single-file 4-player Hearts vs. 3 CPU
  opponents. See its own README for details.
- **[`klondike/`](klondike/)** — a single-file classic Klondike solitaire.
  See its own README for details.
- **`notes/`** — miscellaneous design notes.

`index.html` is the landing page linking to all four. Its game list is
keyboard-navigable: **↑/↓** or **W/S** move between cards (Enter opens the
focused one, native `<a>` behavior), and **1**/**2**/**3**/**4** jump
straight to a game. Each game itself has **Backspace**/**B** to come back
here.
