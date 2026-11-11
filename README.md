# Playspace

Playspace is a collection of classic and casual browser games, built via
agentic engineering, mainly for personal use. Play them at
<https://abroom.github.io/playspace/>.

## Contents

- **[`freecell/`](freecell/)** — a single-file classic FreeCell. See its
  own README for details.
- **[`hearts/`](hearts/)** — a single-file 4-player Hearts vs. 3 CPU
  opponents. See its own README for details.
- **[`klondike/`](klondike/)** — a single-file classic Klondike solitaire.
  See its own README for details.
- **[`minesweeper/`](minesweeper/)** — a single-file classic Minesweeper.
  See its own README for details.
- **[`skyscraper-farm/`](skyscraper-farm/)** — a single-file browser puzzle
  game (tower/crate sorting). See its own README for details.

`index.html` is the landing page linking to all five, listed alphabetically.
Its game list is keyboard-navigable: **↑/↓** or **W/S** move between cards
(Enter opens the focused one, native `<a>` behavior), and **1**–**5** jump
straight to a game (by that alphabetical position). Each game itself has
**Backspace**/**B** to come back here.

The black square at the top right of `index.html` blanks the screen: it
covers the page in black and goes browser-fullscreen where supported. Any
click or key brings the page back.
