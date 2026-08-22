# Swalpa Adjust Maadi — Handoff Document
Prepared: 2026-08-16. Updated: 2026-08-22 (v28 rules rework).

## What this is
"Swalpa Adjust Maadi" is a single-file HTML/JS/Canvas board game (Bangalore-roadworks
themed pipe-connection puzzle). The entire game — logic, rendering, audio, and the
pre-launch tutorial animation — lives in one self-contained HTML file with no external
JS/CSS dependencies (fonts are base64-embedded).

**Current live file: `swalpa_adjust_maadi_v28.html`.** This is the authoritative,
shipped version. `v25.html` is kept as the last stable pre-rework baseline (referenced
below for context). **`v26.html` and `v27.html`, if you find them lying around, are
abandoned exploratory branches — do not build on them.** See "Version history" below
for why.

## Files included in this handoff
- `swalpa_adjust_maadi_v28.html` — the full game, current shipped state
- `swalpa_adjust_maadi_v25.html` — last stable version before the house-rule rework,
  kept for reference/diffing only
- `tutorial.html` — standalone how-to-play tutorial page, linked from the game's
  Tutorial button (`window.open('tutorial.html', '_blank')`). Self-contained
  (base64-embedded images). Its "A house forms" step text was updated to match v28's
  rule (see below) — nothing else in this file has changed.
- `favicon.ico`, `favicon-32x32.png`, `apple-touch-icon.png` — icons referenced by
  the HTML `<head>`
- `preview.png` — the Open Graph / Twitter card image referenced by `og:image` /
  `twitter:image` meta tags (pulled from an absolute GitHub Pages URL at runtime;
  this local copy is for reference/backup only)
- `HANDOFF.md` — this file

## Version history
- **v25**: last stable baseline before the rules rework. Whole-board house creation
  (any red+blue-connected square became a house, every turn, regardless of what that
  turn's move actually did), setup-time grey/unowned houses, and a steal mechanic.
- **v26 — abandoned.** Explored removing grey houses, adding a house cap with a
  "vanishing houses" animation when a player exceeded it, and a transition-based
  house-creation rule. Accumulated enough layered patches (turn-start snapshot
  redesigned three times, animation sequencing bugs, a real crash on browser
  restart/mode-switch) that it was abandoned rather than continuing to patch it.
- **v27 — abandoned.** Rebuilt cleanly from v25 rather than patching v26 further.
  Re-implemented grey-house removal, a turn-start-flag-gated house-creation rule
  (with an exception for the directly-played square), the house cap with a
  deterministic spiral-based resolution order, and the AI-calls-real-function
  refactor (this last piece *did* carry forward into v28 — see below). Abandoned
  specifically because the house-cap "vanishing houses" mechanic was judged too hard
  for a player to follow in the moment, independent of whether its implementation
  bugs were fixed.
- **v28 — current, shipped.** Rebuilt from v25 (not from v26/v27). Carried forward the
  parts of the v27 housekeeping still judged correct — the AI-calls-real-function
  refactor and a launch-sequence consistency fix — then implemented a new, simpler
  house-creation rule that does **not** reintroduce any house cap or vanishing-houses
  concept at all. See below for the exact rule.

## Major rule changes in v28 (vs v25)

### 1. Grey (unowned) houses removed entirely
- `initFreshBoard()` no longer creates any houses from the initial board's
  connectivity. `house[][]` starts fully `false`, `owner[][]` fully `null`, for every
  board size and every game mode — random game, Today's Game, and the internal
  ranking simulations (`runOneRankSimulation()`) all share this one function, so
  there is no separate code path to keep in sync.
- No code path anywhere in the game can produce a house with no owner: `owner[r][c]`
  is always a real player number whenever `house[r][c]` is `true`.
- Stealing (unchanged mechanic, see `applyHouseAndRevenue()`'s steal-check block)
  still exists, but now only ever transfers a house between two real owners — there
  is no "grey" category for it to interact with any more. The steal-check logic
  itself needed no code change for this; only its own explanatory comment (which
  used to mention grey as a special case) was corrected.

### 2. Houses now form ONLY on the exact square played that turn
- **Old rule (v25 and earlier):** `applyHouseAndRevenue()` scanned the *whole board*
  every turn — any square that was currently red+blue connected and not yet a house
  became a new house, regardless of whether the current move had anything to do with
  it.
- **New rule (v28):** a house can only ever be newly created on `lastPlayedSquare` —
  the exact square this turn's card was placed on — and only if that square is
  red+blue connected after the placement. No other square, however connected, is
  ever eligible, no matter what else changed on the board that turn.
- This is intentionally simpler than an earlier, abandoned (v27) "genuinely-new-
  connection" design that needed a turn-start connectivity snapshot plus an
  exception for the played square. The v28 rule needs no snapshot at all, since it's
  scoped to a single square by construction — there's nothing to compare against.
- `render()`'s live preview mirrors this exactly, scoped to `pendingSquare` (the
  square currently being previewed) instead of scanning the whole board. Both the
  new-house preview and the steal preview now share one `isRelevantSquare` check
  (`pendingSquare`, or `lastPlayedSquare` during the brief post-commit reveal window
  via `awaitingStealReveal`) — see that variable's own comment in `render()` for why
  both the pendingSquare and the awaitingStealReveal-gated fallback are needed
  together, not just one or the other.
- The steal mechanic's own trigger condition is unchanged (the played square's
  existing house must be disconnected, with a same-owner neighbour) — it was never
  part of the whole-board scan to begin with, so removing that scan didn't touch it.

### 3. Pre-launch tutorial animation updated to match
The animation's "a house forms" frame (frame 6 of 8, 0-indexed as `FRAMES[6]`)
previously showed the new house appearing at bottom-right while a *different* square
(bottom-left) was the one flagged grey as "just played" — accurate under the old
whole-board rule, misleading under the new one.

Fixed:
- Bottom-left tile is now fixed in place from the first frame of this sequence
  (`FRAMES[4]`) onward — it no longer appears out of nowhere partway through; only
  its own connectivity display updates live as neighbouring tiles activate.
- The tile that used to sit permanently at bottom-right (the "Original", a straight
  tile) is now only placed there in `FRAMES[6]` itself — the exact move that both
  creates the house *and* is the square flagged as just-played. Before that frame,
  bottom-right holds that same tile's 90°-rotated version, which — given the fixed
  bottom-left and the existing top-right geometry — cannot connect either way, so it
  correctly shows no flow and no house until the Original is actually placed.
- `CARD_SPEC[6]` (the standalone active-card icon shown above the board) updated
  from the old bottom-left tile to `{kind:'S', rot:0}`, matching what's now actually
  being placed at that frame.
- Caption for that frame changed to: **"A card that connects power and water forms a
  house"**
- All animation captions (`CAPTIONS[4..7]`) had their trailing full stops removed for
  consistency with the rest.
- `drawFingerAt()` now sets its own `ctx.fillStyle` explicitly before drawing, rather
  than depending on whatever the previous draw call happened to leave behind. It was
  rendering invisibly in two of the four "tap to try" intro frames, because
  `drawHazardTapeBorder()`'s own `save()`/`restore()` correctly restored `fillStyle`
  back to `drawCornerCrop()`'s leftover board-texture pattern (very light, near-
  transparent) — the one frame where the finger *was* visible only worked by
  coincidence, because that frame's highlight function happened to leave a solid
  grey fill behind instead.
- Root logic for all of the above lives in `const FRAMES = [...]`, `const
  CAPTIONS = [...]`, and `const CARD_SPEC = [...]` — all three are 8-entry arrays,
  index-aligned to each other, near the end of the script.
- `tutorial.html`'s "A house forms" step text was updated separately (its own file,
  not part of the main game's inline animation) — same rule, described in prose
  rather than animated. No other tutorial content was touched.

## AI architecture: the AI now calls the real game logic directly

`aiChooseSquare()` used to contain its own hand-written reimplementation of house
creation and stealing, parallel to (and able to silently drift out of sync with)
`applyHouseAndRevenue()`. It now calls `applyHouseAndRevenue()` itself, once per
candidate square it evaluates:

- Everything that function touches (`house`, `owner`, `revenue`,
  `lastScoringSquares`, `lastScoringAmount`, `lastScoringPlayer`, `lastPlayedSquare`,
  `turn`) is saved before each candidate and fully restored after, so evaluating a
  hypothetical move never actually affects real game state.
- `simulationModeActive` is set for the whole evaluation loop, so
  `applyHouseAndRevenue()`'s own "Good steal!" popup/sound side effects (meant only
  for a real, committed move) don't fire while the AI is just considering options —
  the same mechanism Today's Game's own 99-game ranking simulation already relies on,
  for exactly this reason.
- `myScore` is read directly from `lastScoringAmount` after that call. `oppScore`,
  territory, and build/steal detection are simple *reads* of the resulting
  `house[][]`/`owner[][]` state (comparing before vs after the candidate), not
  separate re-derivations of the underlying rules.
- **Practical effect:** any future change to how houses form or change hands only
  has to be made once, inside `applyHouseAndRevenue()` (and, for preview purposes,
  the matching logic in `render()`) — the AI's evaluation picks it up automatically,
  with no separate AI-side update required. This was the whole point of the
  refactor, and it was verified directly rather than assumed: choose a square via
  `aiChooseSquare()`, then actually commit that exact move for real, and confirm the
  resulting revenue change matches what the AI's own evaluation used internally.
- **Deliberately unchanged:** opponent selection (`pickLeadingOpponent`), the AI's
  random-subset visibility sizing per board size (`AI_SUBSET_SIZE_CONFIG`), the
  structure of the 99-game Today's-Game ranking simulation, board/deck reset logic,
  and Today's-Game seed reproducibility. None of this was touched by the refactor.

## Launch-sequence cleanup: `resetForNewTurn()`

Today's Game's launch path (`freshGame()` → `showTodaysTargetPopup()`) renders the
board *before* `beginTurn()` ever runs, since it shows the "Target for today's game"
popup first and only calls `beginTurn()` once that popup closes. A normal (non-
Today's-Game) launch goes straight to `beginTurn()`.

This meant `beginTurn()`'s own turn-start resets (`pendingSquare`, `previewSavedTile`,
`triesRemaining`, `hintUsedThisTurn`) were **not yet applied** at the moment Today's
Game's early render happened — specifically, `triesRemaining` and `hintUsedThisTurn`
could still hold stale values left over from a previous game/session during the
Target-popup phase, even though the board itself was already correctly reset.

Fixed by extracting a single function:

```js
function resetForNewTurn(){
  pendingSquare = null;
  previewSavedTile = null;
  triesRemaining = 5;
  hintUsedThisTurn = false;
}
```

called by **both** `beginTurn()` and `showTodaysTargetPopup()`, before their
respective first `render()`. `awaitingAdvance` is deliberately *not* part of this
shared function — the two callers genuinely need different values for it
(`beginTurn()` sets it `false` immediately, to let the turn begin; `showTodaysTarget
Popup()` sets it `true`, to keep input blocked until the Target popup closes). **Any
future turn-start state that needs to stay consistent across both launch paths
should be added to this one function, not duplicated in both places** — that
duplication is exactly what caused this bug in the first place.

Verified directly: reproducibility (same seed + same move sequence → byte-identical
board/house/owner/revenue across two independent runs) was re-confirmed after every
rule change made in this session, including this one. One thing worth flagging for
whoever writes the next test harness for this file: the AI's own move-choice
shuffle (`aiChooseSquare()`'s default `randFn`, via `rng()`) is only seeded when
`isTodaysGame` is `true` — a test that reseeds `randomBoardRandomFn` but forgets to
set `isTodaysGame` will see genuinely random (non-reproducible) AI choices and can
misread that as a real bug. This cost real debugging time in this session before the
cause was found.

## Title screen aesthetic (v28)

The decorative row of rendered pipe/park tiles below the title
(`renderTitleTileRow()`, `<canvas id="titleTileRow">`) has been removed entirely —
the function, both of its call sites (inside `computeLayout()` and
`bootLaunchScreen()`), and the canvas element are all gone. Replaced with a plain
horizontal rule (`#titleDivider`, 70px wide, centred, using the same muted `--rule`
colour already used for borders elsewhere) between the title and the Kannada
subtitle. The Kannada subtitle (`#tagline`) is no longer italic and now uses the
same ink colour as the title (`var(--ink)`), rather than the softer, italic
secondary colour it used before.

## Key code landmarks (v28)
- `<head>`: meta/OG tags, favicon links, base64 `@font-face` (Teko)
- `initFreshBoard()` — shared by real play and `runOneRankSimulation()`; starts
  `house`/`owner` fully empty (see "Major rule changes" #1)
- `applyHouseAndRevenue()` — the single source of truth for house creation,
  stealing, and scoring; scoped to `lastPlayedSquare` for creation (see #2). Called
  directly by both the real commit flow and `aiChooseSquare()`'s own evaluation.
- `render()` — the live preview; its house/steal projection logic mirrors
  `applyHouseAndRevenue()`'s rule exactly, scoped to `pendingSquare` via the shared
  `isRelevantSquare` check
- `aiChooseSquare()` — see "AI architecture" above
- `resetForNewTurn()`, `beginTurn()`, `showTodaysTargetPopup()`, `freshGame()` — see
  "Launch-sequence cleanup" above
- `drawHazardTapeBorder()` / `getHazardTapePattern()` — hazard-tape rendering
- `drawCornerCrop(ctx, spec, houseAt, ownerColor, changedAt)` — renders a cropped
  board fragment for tutorial frames; `changedAt` is the greyed/highlighted square
- `const FRAMES = [...]` (8 entries), `const CAPTIONS = [...]` (8 matching strings),
  `const CARD_SPEC = [...]` (8 entries, `{tile, greyed}` or `null`) — the tutorial
  animation's frame data; see "Pre-launch tutorial animation updated to match" above
  for what changed and why
- `drawFingerAt(ctx, r, c)` — now sets its own `fillStyle`; see above for why that
  mattered
- `applyCardSpec(idx)` — shows/hides/draws the standalone active-card canvas per
  frame
- `window.startRulesAnim` / `window.resumeRulesAnim` — animation entry points,
  called from `bootLaunchScreen()`
- `simulationModeActive`, `runOneRankSimulation()` — the internal 99-game
  simulation batch that runs during boot (for daily-rank calculation); also the
  mechanism `aiChooseSquare()` now reuses to suppress popup/sound side effects
  during its own per-candidate evaluation

## Testing approach used in this session
Game-logic changes (the house-creation rule, the AI refactor, reproducibility) were
verified by extracting the relevant pure functions out of the HTML file into
isolated Node.js test harnesses (a small Python script pulls out named
function/const bodies by scanning for matching braces), then running many simulated
full games through them — checking for zero crashes, checking the new rule's
invariant directly (e.g. "walked every move of 15 full games and confirmed a house
was created only when the played square matched", not just "the final state looked
plausible"), and checking reproducibility by running the exact same seeded game
twice and diffing the full board/house/owner/revenue state, not just the final
score. `node --check` was run on the extracted `<script>` block after every edit.

Visual/animation changes (the tutorial frame fix, the finger-icon fix) were verified
by extracting the same drawing functions (`drawTileOn`, `drawHouseShape`,
`drawCornerCrop`, etc.) into a Node script using the `canvas` npm package
(`node-canvas`), rendering the actual frames to PNG images, and inspecting those
images directly — including, for the tutorial-frame fix, rendering the proposed new
frames *before* editing the HTML at all, so the design could be checked visually
first. The final render was done a second time reading the values directly back out
of the already-edited HTML file, specifically to catch any transcription mistake
between what was intended and what actually landed in the file, rather than trusting
the earlier, separate proof-of-concept render.

There is no Playwright/headless-browser test setup used in this particular session
(unlike the prior session referenced in "A note on the prior session" below) — real
interactive/DOM behaviour (actual click handling, actual popup timing, actual audio)
was reasoned through by reading the relevant code paths carefully rather than
executed end-to-end in a browser. If a future session has browser automation
available, re-verifying the full interactive flow end-to-end (not just the
extracted logic and rendering) would be a reasonable next step, especially for
anything touching `awaitingAdvance`/`awaitingStealReveal` timing or popup sequencing.

## Known state / no open issues
As of the last message in this session, all requested changes were implemented and
verified per the sections above. There is no known pending bug at handoff time — but
re-verify anything the user flags, since requirements can still shift, and note the
"Testing approach" caveat above about interactive/DOM behaviour not having been
exercised in a real browser this session.

## A note on the prior session (carried forward from the previous handoff)
An earlier session (before v25 was finalized) experienced a technical malfunction
(garbled/repetitive internal reasoning before some responses), which the user
correctly flagged multiple times. The actual code changes made during that session
were verified correct via direct testing, but the conversational flow around them
was degraded. If similar garbled-output behaviour recurs in a future session, stop
and flag it immediately rather than continuing.
