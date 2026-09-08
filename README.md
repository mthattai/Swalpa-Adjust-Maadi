# Swalpa Adjust Maadi

> Dig up roads to cut off power and water from your opponents!

A browser-based, single-file board game about laying power and water pipes on a shared grid, connecting your houses to both supplies while your opponents try to cut you off. Dedicated to the residents of Bengaluru, whose roads are perpetually being dug up for cables, pipes, metro pillars, and flyovers.

Everything — game logic, rendering, the AI opponent, the in-app rules page (diagrams included), and a print-and-play PDF export — lives in one self-contained `index.html`. No build step, no server, no dependencies beyond a browser. (The PDF export lazy-loads `jsPDF` from a CDN only when you actually use it.)

**[Play it here](https://mthattai.github.io/Swalpa-Adjust-Maadi/)**

## Running it locally

Open `index.html` in a browser. That's it.

## Objective

Players are contractors laying power and water pipes on a board; one contractor's work often breaks another's. Every house has a power tank and a water tank, each holding 0–3 units. A house with **at least 2 units of power AND 2 units of water is SAFE**. The first player to get **three SAFE houses** wins.

## Modes

- **Player count: 2, 3, or 4.** Board size scales with player count: **2 players → 7×7, 3 players → 8×8, 4 players → 9×9.**
- **Human or Bot opponent**, toggled independently of player count — with Bot selected, every seat past Player 1 is AI-controlled.
- **Local hotseat** for human-vs-human play at any player count, all on one device.
- **Today's Game** — a daily puzzle seeded from the current date (India time), so everyone playing that day gets the same board. Always 2 players, human vs. Bot.
- Every player starts with **4 houses**, placed automatically during setup (not built during play).

## Board & pipes

- The board's top and bottom edges are the **Power Supply**; its left and right edges are the **Water Supply**.
- Every non-Park square holds a **Pipe tile**: a single power pipe and a single water pipe crossing it, each with two ends. There are **6 tile types** — two Straights and four Bends (one per rotation) — freely reusable any number of times by any player; there's no limited hand or deck to run out of.
- **Park tiles** are fixed at setup and never change; a pipe passes straight through a Park without turning.

## Turns and rounds

Play happens in rounds with a **rotating starting player** (P1 opens Round 1, P2 opens Round 2, and so on). Within a round, players take their turns in order — not simultaneously. On your turn you:

1. Pick a square and a Pipe tile (up to 5 tries before you have to commit), or just call out the square (e.g. `D5`) and the tile code (`S1`, `S2`, `B1`–`B4`) if playing with someone else driving the screen or a physical board.
2. Confirm — the old tile on that square is wiped and replaced.

**Off-limits squares:** Park squares, squares with another player's house, and any square already played this round.

Once every player has moved, the round resolves: power and water flow is traced for every house, tanks are updated, and a new round begins with the next starting player.

## Tracing power and water

For each house, start at one end of its power pipe and trace outward, moving only along connected power pipes. Hitting a water pipe or the Water Supply edge stops the trace; hitting a Park just passes straight through. If the trace reaches the Power Supply, power flows to the house. Water is traced the identical way, swapping colours.

- If **at least one end** of a house's power (or water) pipe reaches its Supply, that tank ticks **up** by 1 (capped at 3).
- If **neither end** reaches Supply, that tank ticks **down** by 1.
- Houses never vanish or get eliminated — a dormant, empty-tanked house just sits there, recoverable on a future round.

## Winning

The first player to reach **3 SAFE houses** at the end of a round wins immediately. If two or more players reach it in the very same round, the game is a draw. There's no round cap and no other draw condition — the game keeps going, however long it takes, until someone hits the target.

## The AI opponent

The Bot evaluates every (tile, square) combination in its visible options against a two-tier priority order:

1. **Don't hand anyone the game.** If a move would let an opponent reach 3 SAFE houses, it's ranked below any move that doesn't — even at the cost of the Bot's own progress. Among moves that are safe on this front, taking an outright win for itself is the next-highest priority.
2. **Build strength.** Failing a decisive moment either way, the Bot maximizes its own count of tank-target-met houses, then how many of its houses are connected on both sides right now (the leading indicator of future progress), then banked tank progress, then minimizes fully-disconnected houses — with board-wide territory as a final tiebreak.

## The Rules page

The in-app Rules page is written to match the mechanics above, and every diagram on it is drawn live by the game's own real rendering and board-generation code — never a static image — wrapped so generating them can never disturb an actual in-progress game's state.

## Print & Play

The **Save board** button exports the current board as a 2-page, landscape A4, black-and-white PDF designed for write-and-wipe play away from a screen:

- **Page 1:** the board shown twice side by side — a template copy with the current pipe layout, and a blank write-and-wipe copy — plus every player's score box (power/water tally per house, with a spot to record each move's square).
- **Page 2:** the full rules text in a two-column layout, with an icon key (Power/Water Supply, Houses, Park, and all six Pipe tile types) matching the exact stroke style used on the board itself.

## Copyright

Copyright 2026 Mukund Thattai

[@thattai.bsky.social](https://bsky.app/profile/thattai.bsky.social)
