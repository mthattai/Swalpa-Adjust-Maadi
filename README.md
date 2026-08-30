# Swalpa Adjust Maadi

A browser-based, single-file board game about laying power and water pipes on a shared grid. Players are contractors connecting houses to both power and water; houses whose supply is disrupted eventually vanish; the last player with any houses standing wins.

Everything — game logic, rendering, AI, and the in-app rules page (including its diagrams, drawn live by the game's own rendering code, never embedded images) — lives in one self-contained HTML file. No build step, no server, no dependencies beyond a browser (an optional PDF-export feature lazy-loads `jsPDF` from a CDN only if the person clicks "Save board").

## Running it

Open the HTML file in a browser. That's it.

## Modes

- **Player count: 2, 3, or 4.** Board size is fixed by player count, not selectable: **2 players always play on a 6×6 board; 3 or 4 players always play on 7×7.** (8×8 was previously a selectable third size; it has been dropped entirely.)
- **Human or Bot opponent**, toggled independently of player count — with Bot selected, every seat past Player 1 is AI-controlled.
- **Local hotseat** for human vs. human play at any player count. Each player's turn is private: the board reverts to a neutral view between turns so other players at the table can't see what was chosen until everyone has moved and the round resolves together.
- **Today's Game** — a daily-puzzle mode seeded from the current date, so everyone playing on a given day gets the same board and deck order. Always exactly 2 players (human vs. Bot), always the 6×6 board.
- Tries per turn (how many card/square combinations you can try before confirming) is a fixed constant, **5**, for every mode — there is no adjustable or "unlimited tries" mode.

## Core rules

- The board's top/bottom edges supply power (red); left/right edges supply water (blue).
- Each round, every player privately picks one pipe card and one square, then all reveal and resolve together.
- If two or more players pick the same square, none of their cards are placed — the square becomes a **Collision** square (shown as a warning-triangle icon; internally still called "Roadworks" in the code, though that term is not player-visible anywhere) instead, and stays open next round.
- A square whose neighbours already supply both colours is a Site. Playing a card there that completes both connections turns it into a house, claimed by whoever placed it.
- Each house has two reservoirs (power, water), capacity 3, starting full. Each round, a still-connected reservoir rises by 1 (capped at 3); a disconnected one falls by 1. Either reservoir hitting 0 removes the house.
- A player is eliminated the moment they have no houses left. Last player standing wins. Simultaneous elimination, or reaching the round cap (36 rounds) with 2+ players still up, is a draw.
- There is currently only one win condition — knockout (elimination). A previous "Collision Champion" secondary win condition (instant win for accumulating collisions) existed in an earlier version of this game but has been removed; there is no trace of it left in the shipped logic.

## Board & connectivity model

Each pipe card carries a power line and a water line simultaneously, along different paths:

- **Straight** cards connect two opposite edges with one colour and the other two opposite edges with the other colour (`straightEdges(rot)`).
- **Bend** cards connect two adjacent edges with one colour, the other two adjacent edges with the other colour (`bendEdges(rot)`).

`connectivity()` does a two-pass flood fill from the board's own supply edges (red from N/S, blue from E/W) outward through matching-colour edges of adjacent tiles, producing `redC[r][c]`/`blueC[r][c]` grids. A square is a genuine house candidate only if both are true there and it's not on the outer ring (`onOuterRing`).

Deck composition (fixed per player, dealt once, no drawing/discarding — a player plays straight from this fixed hand across all 36 rounds), **6 distinct types, equal quantities**:

| Card | Count/player |
|---|---|
| Straight A (rot 0) | 6 |
| Straight B (rot 1) | 6 |
| Bend A (rot 0) | 6 |
| Bend B (rot 1) | 6 |
| Bend C (rot 2) | 6 |
| Bend D (rot 3) | 6 |

36 cards per player in total, matching the 36-round game length exactly. Both the setup board's initial random fill and the player's own dealt hand draw from these same equal proportions — verified directly (2000-draw and 300-game Monte Carlo checks of `buildSetupDeck()`), not just assumed from the constant's shape. (Note: even though the *dealing* probabilities are exactly equal, the board's *accepted* initial layout after setup's rejection-sampling is not — Straight A ends up structurally overrepresented by roughly 2.4× relative to Straight B, because its power/water axes happen to align with the board's fixed supply-entry geometry in a way that makes rejection-sampling accept boards with more of it more often. This is a property of the setup acceptance criterion, not the deal itself.)

Park squares carry no connectivity at all and can't be built on.

## Setup algorithm

The visible game always starts mid-way through a randomly generated board — there's no "empty board, round 1" state. This is done via a **hidden pre-game simulation**, invisible to players, before the first real round begins:

1. Scatter Parks across interior squares (count depends on board size: 4 for 6×6, 6 for 7×7), with no two Parks adjacent.
2. Fill every other square with a random Straight/Bend tile, weighted by the same (equal) proportions as the real deck.
3. Run a flat loop of `SETUP_MOVES_CONFIG[N]` simulated AI moves — 48/56 for 6×6/7×7 respectively, always `8×N`, regardless of how many real players this game session actually has (this figure is exactly what a full 4-player game's setup always computed under the game's original formula, applied unconditionally so the generated board doesn't depend on player count — a deliberate physical-game-mirroring design choice, not an approximation).
4. Gate on a required-connected-squares threshold before accepting the board (rejection-sampled — re-attempted from scratch until met): **8** for 3P/4P games (7×7), or **2 × player count** (= 4) for 2P games (6×6) — the smaller threshold for 2P reflects that a 6×6, 2-player game only ever needs 2 houses total, so demanding the same 8-square bar as a 4-colour board was both unnecessarily strict there and a needless drag on setup attempts.
5. Houses are then placed on the resulting board and claimed — 2 per active player colour, starting at full reservoirs.

Player-visible play begins immediately after this — round 1 is the first round a player actually sees, on an already-substantially-built board.

## The AI heuristic

`aiChooseCardAndSquare(color, randFn)` evaluates every (card, square) combination from a shared random subset of squares — `AI_SUBSET_SIZE_CONFIG = {6:18, 7:25}` squares, picked once per turn (not reshuffled per card type), so what the AI "can see" this turn is one consistent view regardless of which card is under consideration. Off-limits squares, Parks, and other players' houses are excluded from the candidate list. On a Collision-square turn the AI isn't consulted at all — that resolves automatically elsewhere.

Each candidate is scored by a priority tuple:

1. **Stay-alive filter** — any candidate leaving the mover with zero houses is excluded, unless that would exclude every candidate (the AI still has to move something).
2. **Knockout count** — how many opponents this exact move would eliminate outright. Checked first among scored terms: a kill shot can end the game right now, and the opportunity may not persist to next turn.
3. **Lead score** — `myG − max(oppG)`, where `G` approximates a player's own houses' expected remaining lifetime as a population under shared, limited risk (see below). The opponent max is recomputed fresh per candidate — not a single leader fixed for the whole turn, since different candidate moves can affect different opponents differently.
4. **AllSites**, then **territory** — board-wide tiebreaks when everything above is equal.

Among the resulting ranked candidates, the AI picks randomly between the top 2 (50/50), as a hedge against an opponent's own evaluation converging on the exact same single best square and turning a good move into a wasted collision.

### The G (lifetime) score

`computeLifetimeH(player, vanishSet, redC, blueC)`:

```
G = houseCount × untouchedCount + Σ min(power, water)
```

summed over that player's surviving houses, where a house is "untouched" if **both** its power and water are connected right now (read from live `redC`/`blueC`, not reservoir level — a house can be momentarily disconnected while still full, or connected while nearly empty). The model behind this: an untouched house is still waiting on whatever risk comes next, since neither reservoir will drain on the next tick if nothing changes; a house with one side already cut is instead on a bounded, mostly self-determined countdown (its lower reservoir hits zero within `RESERVOIR_MAX` more ticks regardless of anything else on the board). The `houseCount × untouchedCount` term dominates — more untouched houses in a bigger population means longer before shared risk catches up with all of them — and the summed-minimum term is a secondary correction for however much running room remains, mattering most once few or no houses are untouched.

## Rules page

The in-app rules page (`#rulesPage`) covers Components, Objective, Game Phases, The Board, The Deck, Playing a Round, Playing on a Device, Power and Water Supplies, Building Houses, Demolishing Houses, and Winning the Game. Its diagrams (deck quantities, power/water flow before/after, choosing-a-square/confirming-a-move, a collision, site→house, and a three-house reservoir-tick illustration) are not images — they're drawn live, once per session, by directly calling the game's own real drawing and board-generation functions (`drawTileOn()`, `connectivity()`, `buildSetupDeck()`, `runHiddenSetup()`, `drawHouseShape()`, etc.), wrapped so the diagram-generation process never disturbs an actual in-progress game's state (`withScratchBoardState()` snapshots and restores every global these functions touch around each diagram).

## Physical/print components

A "Save board" button exports the current board as a single-page, monochrome, print-friendly PDF schematic (dotted lines for power routing, dashed for water, grey wash for sites, a numbered house icon per player, real park-tree outlines) — built with `jsPDF`, loaded lazily from a CDN only when this button is used.

The physical game's shipped component quantities (cards and tokens, sized to match the printed board's own grid squares) were determined via extensive Monte Carlo simulation of real gameplay, run in Node against the actual extracted game script (never a reimplementation):

| Component | Quantity |
|---|---|
| Park cards | 6 |
| Pipe cards | 100 (25 × Straight A, 15 × each other type) |
| Collision cards | 10 |
| House tokens | 40 (10 × each of 4 players) |
| Site tokens | 15 |

Verified across 250-replicate simulation runs at every player count with zero breaches of any of these limits.

## Key constants

| Constant | Value |
|---|---|
| `RESERVOIR_MAX` | 3 |
| `GRACE_PERIOD_ROUNDS` | 0 (knockout is live from round 1) |
| `TOTAL_ROUNDS` | 36 |
| `SETUP_MOVES_CONFIG` | `{6:48, 7:56}` |
| `REQUIRED_CONNECTED_FOR_HOUSES` | 8 (3P/4P); 2P uses `2 × player count` instead |
| `AI_SUBSET_SIZE_CONFIG` | `{6:18, 7:25}` |
| Parks per board size | `{6:4, 7:6}` |
| Tries per turn | 5 (fixed, not adjustable) |
