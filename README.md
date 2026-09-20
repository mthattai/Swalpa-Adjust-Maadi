# Swalpa Adjust Maadi

A board game about being a ruthless contractor in Bengaluru — connect your own houses to power and water, and don't worry too much about whose pipes you dig up along the way.

**Current build:** `swalpa_adjust_maadi_v47-42.html` — a single, self-contained HTML file. Open it in any modern browser; no install, no server, no build step.

---

## 1. What the game is

2–4 players compete on a shared grid of pipe tiles. Power flows in from the top and bottom edges, water from the left and right. Each player owns four houses; on your turn you move to a square, rotate or flip its pipe tile, and see how that ripples through the board's power and water flow. A house needs a steady supply of *both* resources to stay safe — starve it of either one and it's abandoned, tanks reset to empty.

From round 3 onward, a die roll adds a bonus move before your standard one:

| Roll | Bonus | What happens |
|---|---|---|
| 1, 2 | No-bonus | Nothing extra — just your standard move |
| 3 | Jump 🚀 | Move your token to *any* square, no row/column constraint |
| 4 | 2x 🎁 | An extra standard move (tanks update only after your real turn) |
| 5, 6 | Dig 🚧 | Relocate the dug-up tile and bulldozer anywhere on the board |

First player to get all four houses safe *at the same time* wins. Everyone else gets one last move with a jump bonus once that happens, so a photo finish is possible.

Play against other humans (pass-and-play), against the built-in bot, or print the reference PDF (below) to play with a physical board and real dice.

## 2. Running it

Just open the HTML file. Everything — game logic, rendering, sound, the PDF exporter — is in that one file, aside from [jsPDF](https://github.com/parallax/jsPDF), which loads lazily from a CDN the first time you actually export a PDF (see `PDF_EXPORT_README.md`).

## 3. What's in this build

- **In-app Rules page** (the "?" button) — walks through the full rule set, including bonus moves and the updated endgame, with inline diagrams.
- **PDF export** (the save/printer icon, `generateBoardPDF()`) — a three-page reference document, board-agnostic (not a snapshot of the currently-loaded game): a page of cut-apart board templates for 2P/3P/4P games, a printed rules page, and a page combining a full graphic bonus-moves icon key with a tank-update cheat sheet. This replaced an older single-page "write-and-wipe" live-board snapshot entirely — **`PDF_EXPORT_README.md` still describes that older, now-superseded version** (`generateWriteAndWipeSheet()`, no longer present in the script at all), not this one; treat it as historical background on the PDF pipeline's own earlier shape, not as documentation of the current export.
- **Human or bot opponents**, any mix, 2–4 players.
- **Auto/Hint button** — the same AI that plays bot turns can also suggest or play your own move.

## 4. If you're picking this project up

This repo accumulates a lot of documentation as it goes — start here, then go deeper as needed:

- **`MASTER_HANDOFF.md`** — the authoritative technical reference: turn-type system, AI heuristic, lock/position model, UI/animation conventions, seeding gotchas for simulation, and a running log of what's been verified and how. Read this before making any nontrivial change.
- **`SIMULATION_METHODOLOGY.md`** — how to write headless tests against the real game logic (no rendering, real per-turn driver), and the seeding trap that makes `enterReproducibleTestMode()` easy to misuse for "many distinct games" testing.
- **`PDF_EXPORT_README.md`**, **`tank_cheatsheet_README.md`**, **`STAMP_TILES_README.md`** — deep dives on specific export/print features.
- **`high_speed_simulation.js`** — a ready-to-run driver for aggregate stats (win rates, turn-type distribution, game length) across many genuinely distinct random games.

**One thing worth knowing up front, since it's caused real mistakes before:** the PDF's own printed die-face groupings (what a player rolling a physical die reads) are deliberately decoupled from the digital game's internal random-number wiring (`rollTurnType()`). The app never shows a die face on screen — only the resulting bonus type — so the two don't need to match, and shouldn't be "fixed" into matching without that being an explicit, deliberate instruction. See `MASTER_HANDOFF.md` §9.2 for the full story.

## 5. Assembling a new build

The working script (`game_script_table.js`) and the HTML shell (`swalpa_adjust_maadi_v43-32.html`) are separate files, recombined on every ship:

```bash
head -n <script-tag-line> swalpa_adjust_maadi_v43-32.html > OUTPUT.html
cat game_script_table.js >> OUTPUT.html
echo "</script>" >> OUTPUT.html
tail -n +<line-after-close-script-tag> swalpa_adjust_maadi_v43-32.html >> OUTPUT.html
```

The shell's own `<script>`/`</script>` line numbers shift whenever its HTML (not the script) changes, so re-check them (`grep -n "^<script>$\|^</script>$"`) rather than reuse a cached line number. Always `node --check` the extracted script before shipping.
