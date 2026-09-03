# Swalpa Adjust Maadi

A browser-based, single-file board game about laying power and water pipes on a shared grid. Players are contractors connecting houses to both power and water; houses whose supply is disrupted eventually get knocked out; the last player with any houses standing wins.

Everything — game logic, rendering, AI, the in-app rules page, and the print/export pipeline — lives in one self-contained HTML file, `swalpa_adjust_maadi_v39.html`. No build step, no server, no dependencies beyond a browser (the "Save Board" export lazy-loads `jsPDF` from a CDN only when it's actually used).

> This document describes the game as currently shipped. It supersedes the earlier `README_v39.md`/`HANDOFF_v39.md` pair — enough has changed (house-building disabled, the rules text rewritten, the PDF export replaced entirely, an elimination bug fixed) that a delta-on-a-delta stopped being the clearer format. The base game mechanics not touched by any of that (the connectivity model, the AI's G-score formula, the setup algorithm) are unchanged from those documents and are restated here for a single source of truth.

## Running it

Open the HTML file in a browser. That's it.

## Modes

- **Player count: 2, 3, or 4.** Board size is fixed by player count, not selectable: **2 players always play on a 6×6 board; 3 or 4 players always play on 7×7.**
- **Human or Bot opponent**, toggled independently of player count — with Bot selected, every seat past Player 1 is AI-controlled.
- **Local hotseat** for human vs. human play at any player count. Each player's turn is private: the board reverts to a neutral view between turns so other players at the table can't see what was chosen until everyone has moved and the round resolves together.
- **Today's Game** — a daily-puzzle mode seeded from the current date, so everyone playing on a given day gets the same board and deck order. Always exactly 2 players (human vs. Bot), always the 6×6 board.
- Tries per turn (how many card/square combinations you can try before confirming) is a fixed constant, **5**, for every mode — no adjustable or "unlimited tries" mode.

## Core rules

- The board's top/bottom edges supply power; left/right edges supply water.
- Each round, every player privately picks one pipe tile and one square, then all reveal and resolve together — genuinely simultaneous, not turn-order-resolved (see *AI heuristic* below for why this matters to how the game is tested).
- If two or more players pick the same square, none of their tiles are placed — the square becomes a **Collision/Roadworks** square instead (pipes struck out), and stays open next round.
- **Every player starts with all 3 of their houses already on the board at setup.** Houses are no longer built during play — connecting a Site to both supplies is purely cosmetic now (see *House-building* below).
- Each house has two reservoirs (power, water), capacity 3, starting full. Each round, a still-connected reservoir refills to full; a disconnected one falls by 1. Either reservoir hitting 0 knocks the house out.
- A player is eliminated the moment they have no houses left. Last player standing wins. Simultaneous elimination is a draw.
- **There is no round cap.** Elimination (knockout) is the only way a game can end — a previous 36-round cap-triggers-a-draw rule has been removed entirely.

## House-building (disabled, not removed)

As of this version, houses can no longer be built mid-game — every player's full house budget (`HMAX = 3`) is handed out at setup, matching the number of houses actually placed, so the in-game "build a house" token pool (`houseTokensRemaining`) is always zero for everyone from the first round on.

This was done as a soft, reversible disable rather than a deletion:

- `HMAX = 3` (was 4) — the one change that actually stops building, since every trigger point for it (`selectHouseIcon()`, the bot's own `allHousesFragile()` check) already required a nonzero token count before doing anything.
- `HOUSE_BUILDING_UI_ENABLED = false` — hides the house-build icon that used to sit as a 7th slot in the hand row (both the live player's own hand and the shared multiplayer reveal deck).
- `SHOW_SITE_HIGHLIGHT = false` — hides the "Site" highlight on the live board (a square with full power+water connectivity but no house), since it no longer leads anywhere for a player to see. **Site tracking itself is untouched** — `hasFullEdgeSupply()` and the AI's own site-counting heuristics are unaffected; only the two *visual* callers (the live board's own render loop, and the legend row's "Site" icon) were gated off.

Flipping all three flags back is enough to fully restore the feature; none of the underlying mechanics were deleted.

## Board & connectivity model

Each pipe tile carries a power line and a water line simultaneously, along different paths:

- **Straight** tiles connect two opposite edges with one colour and the other two opposite edges with the other colour (`straightEdges(rot)`).
- **Bend** tiles connect two adjacent edges with one colour, the other two adjacent edges with the other colour (`bendEdges(rot)`).

`connectivity()` does a two-pass flood fill from the board's own supply edges outward through matching-colour edges of adjacent tiles, producing `redC[r][c]`/`blueC[r][c]` grids. A square is a genuine house candidate only if both are true there and it's not on the outer ring.

Deck composition, **6 distinct types, equal weighting**:

| Tile | Weight |
|---|---|
| Straight A (rot 0) | 6 |
| Straight B (rot 1) | 6 |
| Bend A (rot 0) | 6 |
| Bend B (rot 1) | 6 |
| Bend C (rot 2) | 6 |
| Bend D (rot 3) | 6 |

Card-count limits were removed some time ago — a player may play any tile type any number of times, for as long as the game goes on. The weights above now only govern the initial board fill's random draw, not a hand.

Park squares carry no connectivity at all and can't be built on.

## Setup algorithm

The visible game always starts mid-way through a randomly generated board — there's no "empty board, round 1" state:

1. Scatter Parks across interior squares (4 for 6×6, 6 for 7×7), no two adjacent.
2. Fill every other square with a random Straight/Bend tile, weighted by the deck proportions above.
3. Run `SETUP_MOVES_CONFIG[N]` simulated AI moves — 48/56 for 6×6/7×7 — to give the board a starting network, as if several rounds had already been played.
4. Gate on a required-connected-squares threshold before accepting the board (rejection-sampled): **12** for 3P/4P (7×7 — enough for all 4 colours' worth of 3-house slots on the shared template, regardless of `effectiveN`), or **3 × player count** for 2P (6×6).
5. Houses are placed on the resulting connected squares and claimed — **3 per active player colour**, starting at full reservoirs.

Player-visible play begins immediately after this.

## The AI heuristic

`aiChooseCardAndSquare(color, randFn, cardsOnly)` evaluates every (tile, square) combination from a shared random subset of squares — `AI_SUBSET_SIZE_CONFIG = {6:18, 7:25}` squares, picked once per turn, so what the AI "can see" this turn is one consistent view regardless of which tile is under consideration. Off-limits squares, Parks, and other players' houses are excluded from the candidate list.

Each candidate is scored by a priority tuple:

1. **Stay-alive filter** — any candidate leaving the mover with zero houses is excluded, unless that would exclude every candidate.
2. **Knockout count** — how many opponents this exact move would eliminate outright.
3. **Lead score** — `myG − max(oppG)` (see the G/lifetime score below), recomputed fresh per candidate.
4. **AllSites**, then **territory** — board-wide tiebreaks when everything above is equal.

Among the resulting ranked candidates, the AI picks randomly between the top 2, as a hedge against an opponent's evaluation converging on the exact same square and turning a good move into a wasted collision.

House-building candidates are still evaluated in the same ranked pool as everything else, gated on `houseTokensRemaining[color] > 0` — which, per the *House-building* section above, is currently always false, so this branch is presently dead code, not deleted code.

### The G (lifetime) score

`computeLifetimeH(player, vanishSet, redC, blueC)`:

```
G = houseCount × untouchedCount + Σ min(power, water)
```

summed over that player's surviving houses, where a house is "untouched" if **both** its power and water are connected right now. An untouched house is still waiting on whatever risk comes next; a house with one side already cut is instead on a bounded, self-determined countdown. The `houseCount × untouchedCount` term dominates; the summed-minimum term is a secondary correction that matters most once few or no houses are untouched.

### A note on testing this fairly

Moves are chosen **simultaneously** — no player's AI evaluation ever sees another player's choice for the same round. This means a correctly-built simulation of the game should show **no seat/turn-order advantage** at all between otherwise-identical players. If a self-play simulation harness ever shows one player winning noticeably more than another, treat that as a prompt to check the harness for an information leak (e.g. accidentally mutating shared board state before every player has been asked for their move) before concluding it's a real effect — at a 100-game sample size, seat-win-rate differences of 10-20 percentage points are well within ordinary sampling noise for this game and will often vanish at a few hundred games instead.

## Rules page

The in-app rules page (`#rulesPage`) covers, in order: Components, Dedication, Objective, The Board, Power and Water Supplies, The Deck, Playing a Round, Collisions, Knocking Out Houses, Winning the Game, Playing on a Write & Wipe Sheet, Playing on a Board, and Playing on a Device.

Its diagrams (pipe flow before/after, choosing-a-square/confirming-a-move, a collision, and a three-house reservoir-tick illustration) are not images — they're drawn live, once per session, by directly calling the game's own real drawing and board-generation functions (`drawTileOn()`, `connectivity()`, `buildSetupDeck()`, `runHiddenSetup()`, `drawHouseShape()`, etc.), wrapped so diagram generation never disturbs an actual in-progress game's state (`withScratchBoardState()`).

Two diagrams that existed in earlier versions — the empty board-setup placeholder, and a "site becomes a house" before/after pair — were removed along with the rules sections they illustrated. The site diagram's underlying board-state computation is still shared with the reservoir/knockout diagram that follows it (that diagram's `houseA` is the same square the old site diagram illustrated), so removing it took care to only drop the now-orphaned canvas draws, not the shared setup logic.

## The Write & Wipe sheet (PDF export)

"Save Board" no longer exports a simplified single-page schematic. It now generates the **same physical print-and-play sheet** this game's own design process produced separately (originally built as a Python/reportlab pipeline: `generate_playsheet.py` / `shade_options.py` / `choice_sheet.py` / `combine_a4.py`), ported line-for-line into `jsPDF` so it can be generated client-side from whatever board is currently in front of the player, rather than shipping only as a fixed pre-printed template.

The exported A4 sheet includes, top to bottom:

- The title, in Teko Bold with a hollow-outline treatment on "Write & Wipe".
- Two copies of the current board side by side — left shows Power supply + pipes only, right shows Water supply + pipes only — rendered in the same grey-background / white-negative-space-channel style as the board's own on-screen look, including park trees and numbered house icons, matching the live game's `board`/`house`/`owner`/`N`/`effectiveN` exactly (not a fixed seed).
- A score table per player (2, 3, or 4 depending on the live game's own player count), horizontally centred between the board's coordinate labels and the legend below.
- A legend (Power Supply, Water Supply, Park, Pipe, Houses).
- A rotated "choice card" grid below a cut line — one column per player, six tile types each, laid out so cutting along the dashed lines gives each player their own private "choose a tile and square" card, matching the *Playing on a Write & Wipe Sheet* rules section above.
- A copyright line, rotated to read bottom-to-top in the board sheet's own margin.

### How the choice-card rotation actually works

The choice-card section isn't built as a separate rotated sub-page (the way the original Python/reportlab pipeline did it, via a real PDF-level page transform) — it's computed directly in final coordinates. For a point at native `(x, y)` in the same coordinate convention the original design used:

```
finalX (from the sheet's left edge) = y
finalY (from the sheet's top edge)  = A4_H − x
```

This was verified empirically with a probe shape before being trusted (a previous, very similar transform in the Python pipeline had a sign error that took a dedicated empirical test to catch — this version was checked the same way from the start). Axis-aligned rectangles, circles, and lines transform through this trivially; the two shapes that aren't rotation-invariant (the bolt and drop icons) are rotated via a small local-vector helper (`rotateOffset`), and text uses jsPDF's own `angle: 90` option — which, note, does **not** combine correctly with `align:'center'`/`baseline:'middle'` in the jsPDF version this project loads (2.5.1); the CHOOSE/SQUARE and player-number labels are centred by manually computing the anchor offset instead (half the text's own width and cap-height), not by relying on those alignment options.

The bolt and drop icon shapes used throughout the export (board supply icons, the choice-card tags, the legend) are the exact path data from the game's own live icon row (`drawLightningBoltLegend`/`drawWaterDropLegend`), not a separate approximation — recentred from their original canvas-coordinate form but otherwise unchanged, so the exported sheet's icons match what's already on screen.

## Known-fixed bugs worth knowing about

**Eliminated players could still be handed a turn.** `eliminated[]` used to only be consulted at the end of a round (for standings and the win check) — nothing in the turn-sequencing itself skipped an eliminated seat. Fixed in three places that all shared the same gap: the round-start turn assignment (was hardcoded to player 1), the hotseat mid-round handoff (was a bare `player + 1`), and the bot-move computation loop (had no `eliminated[p]` check at all). A `nextActivePlayer(p)` helper now backs all three. A related structural gap was found and fixed alongside it: in Bot mode, if the human (player 1) is eliminated while bots remain, there was previously no remaining trigger point for the bots to keep playing at all, since every bot move was only ever computed as a side effect of player 1's own commit — `playAllBotsAndReplay()` now covers that case directly.

## Key constants

| Constant | Value |
|---|---|
| `RESERVOIR_MAX` | 3 |
| `HMAX` | 3 (matches the 3 houses given at setup — see *House-building*) |
| `HOUSE_BUILDING_UI_ENABLED` | `false` |
| `SHOW_SITE_HIGHLIGHT` | `false` |
| `GRACE_PERIOD_ROUNDS` | 0 (knockout is live from round 1) |
| Round cap | none — removed entirely; knockout is the only way a game ends |
| `SETUP_MOVES_CONFIG` | `{6:48, 7:56}` |
| `REQUIRED_CONNECTED_FOR_HOUSES` | 12 (3P/4P); 2P uses `3 × player count` instead |
| `AI_SUBSET_SIZE_CONFIG` | `{6:18, 7:25}` |
| Parks per board size | `{6:4, 7:6}` |
| Tries per turn | 5 (fixed, not adjustable) |
| `PERIODIC_ROADWORKS_ENABLED` | `false` (the shared periodic Roadworks event exists in code, dormant) |
