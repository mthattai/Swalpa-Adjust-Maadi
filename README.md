# Swalpa Adjust Maadi

A browser-based, single-file board game about laying power and water pipes on a shared grid. Players are contractors connecting houses to both power and water; houses whose supply is disrupted eventually vanish; the last player with any houses standing wins.

Everything — game logic, rendering, AI, and the in-app rules page — lives in one self-contained HTML file. No build step, no server, no dependencies beyond a browser (an optional PDF-export feature lazy-loads `jsPDF` from a CDN only if the person clicks "Save board").

## Running it

Open the HTML file in a browser. That's it.

## Modes

- **1 player** — human (Player 1) vs. a built-in AI opponent (Player 2).
- **2 / 3 / 4 player** — local hotseat. Each player's turn is private: the board reverts to a neutral view between turns so other players at the table can't see what was chosen until everyone has moved and the round resolves together.
- **Today's Game** — a 1P game seeded from the current date, so everyone playing on a given day gets the same board and deck order (useful for daily-puzzle-style sharing).
- **Practice mode** — unlimited card/square tries per turn instead of the normal cap of 5.

Board size is selectable: 6×6, 7×7, or 8×8.

## Core rules

- The board's top/bottom edges supply power (red); left/right edges supply water (blue).
- Each round, every player privately picks one pipe card and one square, then all reveal and resolve together.
- If two or more players pick the same square, none of their cards are placed — the square becomes Roadworks (rubble, carries nothing) instead, and stays open next round.
- A square whose neighbours already supply both colours is a Site. Playing a card there that completes both connections turns it into a house, claimed by whoever placed it.
- Each house has two reservoirs (power, water), capacity 3, starting full. Each round, a still-connected reservoir rises by 1 (capped at 3); a disconnected one falls by 1. Either reservoir hitting 0 removes the house.
- A player is eliminated the moment they have no houses left. Last player standing wins. Simultaneous elimination, or reaching the round cap with 2+ players still up, is a draw.
- **Collision Champion:** every collision a player is part of adds one mark against them. Reaching 7 marks is an instant win, checked before the elimination check each round. Every player who crosses the threshold in the same round wins together as co-champions — this matters especially in 2P, where any collision necessarily involves both players at once, so a collision-triggered ending there is always a shared win, never one-sided.

## Board & connectivity model

Each pipe card carries a power line and a water line simultaneously, along different paths:

- **Straight** cards connect two opposite edges with one colour and the other two opposite edges with the other colour (`straightEdges(rot)`).
- **Bend** cards connect two adjacent edges with one colour, the other two adjacent edges with the other colour (`bendEdges(rot)`).

`connectivity()` does a two-pass flood fill from the board's own supply edges (red from N/S, blue from E/W) outward through matching-colour edges of adjacent tiles, producing `redC[r][c]`/`blueC[r][c]` grids. A square is a genuine house candidate only if both are true there and it's not on the outer ring (`onOuterRing`).

Deck composition (fixed per player, dealt once, no drawing/discarding — a player plays straight from this fixed hand across all 36 rounds):

| Card | Count/player |
|---|---|
| Straight A (rot 0) | 8 |
| Straight B (rot 1) | 4 |
| Bend A–D (rot 0–3) | 6 each |

Park squares carry no connectivity at all and can't be built on.

## Setup algorithm

The visible game always starts mid-way through a randomly generated board — there's no "empty board, round 1" state. This is done via a **hidden pre-game simulation**, invisible to players, before the first real round begins:

1. Scatter Parks across interior squares (count depends on board size: 4/6/8 for 6×6/7×7/8×8), with no two Parks adjacent.
2. Fill every other square with a random Straight/Bend tile, weighted by the same relative proportions as the real deck (see table above).
3. Run a flat loop of `SETUP_MOVES_CONFIG[N]` simulated AI moves — 48/56/64 for 6×6/7×7/8×8 respectively, always `8×N`, regardless of how many real players this game session actually has. This number isn't arbitrary: it's exactly what a full 4-player game's setup always computed under the game's original formula, now applied unconditionally so the generated board doesn't depend on player count.
4. Gate on `REQUIRED_CONNECTED_FOR_HOUSES = 8`: the generation is rejection-sampled (re-attempted from scratch) until the board has at least 8 genuinely connected squares (own-tile-based, stricter than the "site" definition below). If it isn't reached, retry — in practice this converges quickly and doesn't materially affect load time.
5. Two houses per active colour are then placed on the resulting board and claimed. For a 4P *board* played with fewer real players, the extra colours' pre-assigned houses are simply left as ordinary board squares — never materialized, never claimed by anyone.

Player-visible play begins immediately after this — round 1 is the first round a player actually sees, on an already-substantially-built board.

## The AI heuristic

`aiChooseCardAndSquare(color, randFn)` evaluates every (card, square) combination from a shared random subset of squares — `AI_SUBSET_SIZE_CONFIG = {6:18, 7:25, 8:32}` squares, picked once per turn (not reshuffled per card type), so what the AI "can see" this turn is one consistent view regardless of which card is under consideration. Off-limits squares, Parks, and other players' houses are excluded from the candidate list. On Roadworks turns the AI isn't consulted at all — that resolves automatically elsewhere.

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

## Key constants

| Constant | Value |
|---|---|
| `RESERVOIR_MAX` | 3 |
| `GRACE_PERIOD_ROUNDS` | 0 (knockout is live from round 1) |
| `TOTAL_ROUNDS` | 36 |
| `COLLISION_CHAMPION_THRESHOLD` | 7 |
| `SETUP_MOVES_CONFIG` | `{6:48, 7:56, 8:64}` |
| `REQUIRED_CONNECTED_FOR_HOUSES` | 8 |
| `AI_SUBSET_SIZE_CONFIG` | `{6:18, 7:25, 8:32}` |
| Parks per board size | `{6:4, 7:6, 8:8}` |
