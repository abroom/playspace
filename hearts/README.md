# Hearts

A single-file browser game. Open `hearts.html` directly in a browser — no
build step, no dependencies, no server required.

## Concept

Classic 4-player Hearts: you vs. three CPU opponents (West, North, East).
Full 52-card deck, 13 cards each. Avoid taking hearts (1 point each) and the
Queen of Spades (13 points) — lowest score wins. Play continues hand after
hand until someone reaches 100; whoever has the lowest total at that point
wins.

## Each hand

1. **Pass 3 cards** — left, then right, then across, then no pass, repeating
   every 4 hands (`passDirectionFor`). Skipped entirely on "no pass" hands.
2. **Play 13 tricks** — whoever holds the 2 of clubs leads first. Follow the
   led suit if you can; if you can't, play anything. Highest card of the led
   suit wins the trick and leads the next one — no trump suit.
3. Rule details enforced by `legalPlays()`: hearts can't be led until one has
   already been played (`heartsBroken`) unless hearts are all you have left;
   no point cards on the very first trick unless you have nothing else.
4. **Shooting the moon**: take all 26 points (every heart + the Queen)
   yourself and you score 0 while everyone else takes 26 (`finishHand`).
   The CPUs don't deliberately attempt this — the rule just applies to
   whoever ends up with all of it, human included.

## CPU opponents

Heuristic, not a lookahead/optimal solver (`dangerScore`, `aiChooseCard`):

- **Passing**: pass away the 3 most "dangerous" cards — Queen of Spades
  first, then high spades, then high hearts.
- **Leading**: forced to the 2 of clubs on the first trick; otherwise leads
  its lowest-danger legal card.
- **Following**: if it can duck under the current winning card of the led
  suit, it does (to shed danger safely); if forced to win, it dumps its
  highest card of that suit since the damage is already taken.
- **Discarding** (void in the led suit): dumps its most dangerous card —
  this is the main way points actually move between players.

## Known bug found (and fixed) during testing

Playing the 4th card of a trick doesn't resolve it immediately — there's a
~1s pause (`resolvingTrick`) so the completed trick is visible before it's
swept. Early on, `turn` wasn't invalidated during that pause, so a fast
human click could sneak in an extra, uncounted play on an already-complete
trick, desyncing hand sizes from the trick count. Fixed by locking
`resolvingTrick = true` for that whole window and gating every play path
(the click handler, `legalPlays` rendering, and `maybeAdvanceCPU`) on it.
Caught via a scripted stress-test that auto-played random legal cards as
fast as possible — worth knowing if this class of bug ever resurfaces.

A second bug: `.trick-card { position: absolute; }` and `.card { position:
relative; ... }` are both single-class selectors of equal specificity, and
`.card` was defined *later* in the stylesheet — so it silently won on any
element with both classes, leaving trick cards in normal document flow
instead of pulled out and placed at their seat position. Symptom: the felt
visibly resized as cards were added to a trick, and cards landed wherever
flow put them rather than near the seat that played them. Fixed by making
the selector `.card.trick-card` so it wins on specificity regardless of
source order, rather than relying on rule ordering being kept just so.

## Mobile

- **Portrait** (`max-width: 700px`): an opponent's full hand used to render
  as one overlapping card-back per remaining card, which ate too much
  vertical space. Now it's a single back card with a count badge
  (`.fan-count`) instead of a fanned stack.
- **Landscape** (`max-height: 480px`, independent of width — a landscape
  phone is short, not necessarily narrow): West/North/East move into one row
  across the top of the felt instead of wrapping around it, and the hand
  merges directly into the felt (same background/edge, no gap) instead of
  sitting in a separate section below — otherwise the board and the hand
  never both fit on screen at once. Found a bug here too: shrinking the
  landscape card-back size without shrinking its overlap margin to match
  made the fan's negative margins accumulate faster than the (now smaller)
  card width, so it visually drifted off the edge of the felt. Same fix as
  portrait — single card + count badge — sidesteps the whole class of bug
  rather than tuning the margin. The trick area here is also barely taller
  than one card, so North's (top-anchored) and your (bottom-anchored) trick
  cards would sit almost exactly on top of each other; they're now offset
  sideways from center in opposite directions (`left: 62%` for yours, `38%`
  for North's) so both stay readable.

## Known intentional non-features

- No adjustable CPU difficulty or lookahead — heuristics only.
- No keyboard controls or drag-and-drop — click/tap only, unlike the other
  two games in this repo (could be added later if wanted).
- No mobile-specific layout pass yet beyond a basic `max-width: 700px`
  shrink of cards/table — not as tuned as Skyscraper Farm's mobile pass.
