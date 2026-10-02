# Swalpa Adjust Maadi — Master Handoff v5 (final stress ruleset shipped, danger-tuple AI, grace-period popups)

This document **replaces v4 in full** — not an addendum, a fresh statement
of where the shipped game stands today. v4 described random-seed board
setup and frontier-based house-siting; both are unchanged and carried
forward by reference (§§2, 4, 6–11 below). **New since v4**: the entire
stress/win-condition subsystem was redesigned from scratch and shipped
(§3), the AI gained a restored "danger tuple" Tier between iWin and the
PT/frontier tiers (§5.1b), two new grace-period popups were added (§3.4),
and the UI's stress tick/pop system was reworked (§3.5). The file itself
also moved — see §1.

## 1. File location and verification workflow

**Single self-contained file**: `/home/claude/dev/swalpa_dev_v29_final_stress_rules.html`
— this is the current shipped game, superseding
`swalpa_dev_v27_stress_tuning.html` (v27 is untouched, left as a historical
reference; v28 was an abandoned intermediate, never finished). v29 started
as a fresh copy of v27 and had the entire stress redesign (§3) and danger
tuple (§5.1b) built on top of it this session.

**Verification workflow used this session** (ad hoc, not yet captured as
permanent sync files the way v4 described for v27): extract the inline
`<script>` and `<body>` blocks straight from the shipped HTML into
scratch files before each check, rather than maintaining a permanently
synced `variants/current_v29.js` / `body_only_v29.html` pair:
```js
const html = fs.readFileSync('swalpa_dev_v29_final_stress_rules.html','utf8');
const scripts = [...html.matchAll(/<script>([\s\S]*?)<\/script>/g)].map(m=>m[1]);
const main = scripts.reduce((a,b) => a.length>b.length?a:b); // the real game script is the largest one
const body = html.match(/<body[^>]*>([\s\S]*?)<\/body>/)[1];
```
Then, before trusting any change:
1. `node --check` on the extracted script — confirms it still parses.
2. A headless jsdom smoke test — `JSDOM` with `runScripts:'outside-only'`,
   a stubbed canvas context, `window.setTimeout` made synchronous — driving
   real games end-to-end through the exact real dispatch chain
   (`decideBotMove()` → `applyCardChoice()` → `applySquarePreview()` →
   `commitDecidedMove()`, looped until `gameOver`). Every change in §3 and
   §5.1b below was verified this way (typically 20–40 headless games per
   change, sometimes targeted to force a specific condition — e.g. forcing
   `capTriggered` to exercise the danger-tuple's round-boundary logic).
3. For anything touching the AI's move choice, re-run after the change and
   confirm `0` errors/crashes across the smoke batch — no fixed regression
   suite exists yet beyond this.

**Setting up a reproducible/pinned board** (for fairness testing, unchanged
mechanism from v4 §1, still present in v29): `isTodaysGame = true;
reproducibleTestSeed = <any integer>;` then `freshGame()` — pins a fully
deterministic board+deck. Critically, **`rng()` itself also reads
`seededRandomFn` whenever `isTodaysGame` is true** (not just `boardRNG()`),
so AI tie-breaking becomes deterministic too unless the global `rng`
identifier is explicitly reassigned to `function(){ return Math.random(); }`
right after `freshGame()` returns — this is what lets a board stay pinned
while genuine tie-break variance is preserved across repeated trials on
it. This is the exact method used for the board-fairness run in §12.

**Verification is headless-only**, same caveat as v4: everything here has
been checked via direct jsdom function calls, never in an actual browser.

## 2–4, 6–11: carried forward unchanged from v4

Turn structure (Tile/Roadworks only, Jump fully removed), houses/reservoirs
fixed at pre-game setup (`HMAX=4`, `RESERVOIR_MAX=3`), the VACATE
("abandoned house") mechanic (kept, after explicit reconsideration),
house-siting by frontier (`computeSiteFrontier`/`chooseHouseBuildSquare`),
the "Expanding frontier" UI message investigation, the dead-code removal
inventory, and the open items list from v4 — **none of this changed this
session**. See v4's own §§2, 4, 6–11 for the full write-up; it all still
applies to v29 exactly as described there (v29 is a direct descendant of
the same `v27` file those sections describe).

## 3. Win condition — stress tokens (COMPLETELY REDESIGNED this session)

The entire v4 §3 description (`STARTING_STRESS=12`, stress-only-decreases,
`DISCARD_PER_FULL_HOUSE_BY_MODE`, no cap trigger) is **gone**. The full
design derivation, tuning process, and rejected alternatives are written
up in the project doc `claude/STRESS_RULESET_BEST_SO_FAR.md` — this
section covers only the **shipped mechanics and their implementation**,
not the "why."

### 3.1 Core numbers (all in v29, grep-verified)
```js
const STARTING_STRESS = 0;
const STRESS_CAP = 12;          // doubles as the end-game trigger threshold
const STRESS_PER_TURN = 1;      // accrual, every player, every turn
const ROADWORKS_STRESS_COST = 0;
const DISCARD_PER_QUALIFYING_HOUSE = 2;  // flat, regardless of tier or player count
const FULL_HOUSE_THRESHOLD = 3;  // == the "eq3" tier's threshold
const GE2_QUALIFY_THRESHOLD = 2;
const GE1_QUALIFY_THRESHOLD = 1;
const QUALIFY_THRESHOLD_BY_TIER = { eq3: FULL_HOUSE_THRESHOLD, ge2: GE2_QUALIFY_THRESHOLD, ge1: GE1_QUALIFY_THRESHOLD };
const INSTANT_WIN_ENDGAME = true;
```
Stress rises by `STRESS_PER_TURN` every turn (floored/capped via
`Math.min(STRESS_CAP, ...)`), then is discarded by
`DISCARD_PER_QUALIFYING_HOUSE` per house that currently qualifies under
whichever tier is active, floored at 0 (`Math.max(0, ...)`).

### 3.2 The global qualifying-house ratchet
A single **game-wide** (not per-player) state decides which houses count
as "qualifying" for a discard:
```js
let activeTier = 'eq3';       // 'eq3' -> 'ge2' -> 'ge1', one-way
let anyDiscardedYet = false;  // the instant this flips true, activeTier LOCKS forever
function updateActiveTierForRoundStart(){
  if (anyDiscardedYet) return;
  if (roundNum > 8) activeTier = 'ge1';       // 2Y=8
  else if (roundNum > 4) activeTier = 'ge2';  // Y=4
}
```
Called once per round, in `beginRound()`, before any turn that round
resolves. A qualifying house needs **both** reservoirs at/above
`QUALIFY_THRESHOLD_BY_TIER[activeTier]` (`isQualifyingHouseAtThreshold`/
`qualifyingHouseCountAtThreshold`/`qualifyingHouseSquaresAtThreshold`).
The moment any player's qualifying-house count is found nonzero at the
real discard site (`revealScoringStage()`), `anyDiscardedYet` flips true
and the tier locks permanently — see `STRESS_RULESET_BEST_SO_FAR.md` for
why this has to be global and one-way (fixes an "instant-win" exploit
discovered during tuning).

### 3.3 Win/draw resolution — two distinct endings
- **Mid-game instant win** (`INSTANT_WIN_ENDGAME=true`): the instant any
  player's `stressTokens[p]` hits exactly 0 (checked in
  `afterTurnRevealComplete()`), the game ends immediately —
  `gameWinner=turn; winType='target'; finishGame();` — no grace turns for
  anyone else, no popup (see §3.4 for why the popup moved elsewhere).
- **End-game cap** (`finishCapEndgame()`): the instant any player's stress
  reaches `STRESS_CAP`, `capTriggered` flips true, but the game does
  **not** end there — the current round is allowed to finish normally
  (every remaining seat in `roundPlayOrder` still gets its turn), and
  `advanceToNextPlayerOrRound()` calls `finishCapEndgame()` at that
  round's own boundary instead of starting a new one. Winner = lowest
  final stress; an exact tie for lowest is a draw (`gameWinner=null;
  winType=null`). **Pitfall found and fixed this session**: a stats
  harness must check `capTriggered && gameWinner!==null/===null`, not
  `winType==='stress_cap'`, to classify draws correctly —
  `finishCapEndgame()` sets `winType=null` on a draw, not `'stress_cap'`.

### 3.4 Grace-period popups (restored/moved this session)
The *old* pre-stress-redesign grace-period mechanism (announce, then give
every other seat one final turn before resolving) is superseded by
`INSTANT_WIN_ENDGAME`'s synchronous resolution — that old code path
(`showEventPopup('Player X has cleared all stress tokens!', ...)` +
`endgamePlayersRemaining` queue) is still physically present in
`afterTurnRevealComplete()` but is **dead, unreachable code**, gated
behind `if (INSTANT_WIN_ENDGAME)` returning first. Left in place
deliberately as a historical fallback in case `INSTANT_WIN_ENDGAME` is
ever flipped back — not yet cleaned up.

Two genuinely new popups were added in its place, per direct instruction:
- **Cap-triggered announcement**, in `afterTurnRevealComplete()` at the
  `stressTokens[turn] >= STRESS_CAP` check: fires once (guarded by the
  same `!capTriggered` that sets the flag), announces, then pauses
  (`showEventPopup` + `scheduleAfterPopupOrDelay(continueAfterEndgameCheck, LONG_DELAY_MS)`)
  before the round's remaining turns play out:
  > "Player X is stressed out!" / "Game ends this round…"
- **Ratchet-loosening announcement**, in `beginRound()`: captures
  `activeTier` before calling `updateActiveTierForRoundStart()`, and if it
  actually changed, announces the new (lower) threshold before that
  round's first turn begins:
  > "Relax!" / "Minimum tank level is now {2 or 1}"

  Never fires when the tier was already locked or simply hasn't crossed
  its next Y/2Y boundary — verified via a 40-game headless batch, firing
  "now 2" at round 5 and "now 1" at round 9 exactly when reached, silent
  every other round.

### 3.5 UI — stress ticks and pops (reworked this session)
Two separate, coexisting tick systems:
1. **Board, per-house "-2" tick** (`lastDiscardingHouses`, populated in
   `revealScoringStage()` via `qualifyingHouseSquaresAtThreshold(mover,
   activeThreshold)`): every house that qualified for a discard this turn
   shows a flat `"-2"` pop, drawn in the main `render()` per-square loop.
   This **replaced** an older, vestigial board pop that used to show
   "+1"/"-1" for a house crossing the `eq3` threshold in either direction
   — unrelated to the actual discard mechanism, removed outright.
2. **Scorebox pop sequence** (per player, in the score-row render loop):
   `"+1"` at turn start (`turnStartStressPopActive`/`turnStartStressPopPlayer`,
   set by `showTurnStartStressPop()`), then `"-x"` at reveal
   (`lastStressDelta[mover]`), then settles to the real floored
   `stressTokens[p]`. **Correction made mid-session**: `lastStressDelta`
   must be computed as the *raw, unfloored* delta
   (`extraAccrual - discard`, computed before the `Math.max(0, ...)`
   floor) — not derived from `stressTokens[mover] - stressBeforeThisBlock`,
   which reads the already-floored post-discard value and under-reports
   the discard whenever it would have taken the player negative. An
   earlier edit this session accidentally removed this whole pop sequence
   while adding (1) above; it was restored once the mistake was caught —
   both tick systems are meant to coexist, not replace each other.
3. **Qualifying-house indicator icon**: in `renderModeRow()`, alongside the
   Tile/Roadworks button slots, a small canvas redraws the house-
   battery-gauge shape (`drawHouseShape`) at `power=water=
   QUALIFY_THRESHOLD_BY_TIER[activeTier]` — icon only, no text label (one
   was added then explicitly removed per direct instruction: "I didn't
   tell you to put text there"). Rebuilt fresh every `renderModeRow()`
   call since that function clears its container (`area.innerHTML=''`)
   at the top.

## 5. The AI heuristic — Tier 1 (iWin) → **Tier 1.5 (danger tuple, NEW)** → Tier 2 (PT/frontier)

§5.2–5.5 of v4 (the PT tuple, leading-opponent/leadFlag, ranking,
Stage A/B/C control flow) are **unchanged** — see v4 for the full
write-up. What's new is a tier inserted between Tier 1 and Tier 2:

### 5.1b Tier 1.5 — the danger tuple (restored, redesigned for stress tokens)

A pre-stress-token "danger tuple" existed in an earlier version of the
game and was deleted as dead code during the stress-token AI redesign
(it read a hardcoded, already-drifted "full house" bar, orphaned once its
only callers were removed). This session **restored the concept from
scratch**, redesigned around the actual stress-token mechanics, through
several rounds of direct correction:

- **What it measures**: for each opponent, project their stress forward
  by one real turn — their own `STRESS_PER_TURN` accrual, then a discard
  off whatever houses qualify under the correct tier (see tier-selection
  below), against the **candidate's own resulting hypothetical
  connectivity** (`cand.hypConn`) — a tile move can change an opponent's
  pipe connectivity too, not just the mover's own.
  ```js
  const oppQualifyingHouses = virtualQualifyingHouseCountAtThreshold(p, cand.hypConn, oppThreshold);
  const oppStress = projectedStress(stressTokens[p], STRESS_PER_TURN, oppQualifyingHouses);
  ```
  **Bug found and fixed mid-session**: the accrual argument must be
  `STRESS_PER_TURN`, not `0` — an opponent's stored `stressTokens[p]` is
  from the *end of their last turn*, so their own next turn's `+1` hasn't
  happened yet and has to be added explicitly (unlike the mover, whose
  `stressTokens[color]` already has this turn's `+1` baked in by
  `resetForNewTurn()` before `decideBotMove()` runs).
- **`cand.dangerFlag = 1`** iff any opponent's projected stress hits 0 —
  i.e. they could mid-game-win on their very next move regardless of what
  they do with it.
- **Which tier an opponent is judged against** — the subtle part, fixed in
  two further passes:
  - An opponent who still has a turn coming up **later this round**
    (`roundPlayOrder` slot after the mover's) is judged against today's
    live `activeTier` — the ratchet can't move mid-round.
  - An opponent who **already moved this round** won't move again until
    next round — judged against `tierAssumingNoFurtherDiscards(roundNum+1)`,
    the *worst case* (loosest possible) tier by then, per direct
    instruction ("assume worst case... opponents playing beyond the
    boundary have the new (reduced) threshold") — a looser tier only ever
    *widens* qualification, never narrows, so this is the cautious
    direction. `tierAssumingNoFurtherDiscards(round)` is a pure,
    read-only sibling of `updateActiveTierForRoundStart()`.
  - **Exception, found by direct question-asking rather than by a bug
    report**: if `capTriggered` is *already* true, an opponent who already
    moved this round gets **no further turn at all this game** —
    `advanceToNextPlayerOrRound()` calls `finishCapEndgame()` at this
    round's boundary instead of starting round N+1. Projecting a
    hypothetical next-round turn for such an opponent would score a turn
    that can never happen. Fixed via a `hasFutureTurn` guard: when false,
    the opponent's `oppProjStress` is reported as their real, locked-in
    `stressTokens[p]` as-is (what `finishCapEndgame()` will actually
    compare), and they can never set `dangerFlag`.
- **How it's used** (`pickFromPool()` in `decideBotMove()`): after the
  existing `iWin`-filter (Tier 1) produces `finalPool`, a new filter runs
  — `finalPool.filter(x => !x.dangerFlag)`, falling back to the full pool
  only if *every* candidate is dangerous — **before** the existing
  `leadFlag`/PT ranking (Tier 2) runs on whatever survives. Skipped
  entirely when the mover already has a winning candidate (no opponent
  "next turn" matters if the game just ended).

All of the above was verified via targeted headless smoke tests — forcing
multi-round games, forcing `capTriggered`, and instrumenting
`evaluateCandidatePosition` to confirm each branch (round-boundary tier
switch, cap-triggered skip) actually fires in practice, not just compiles.

## 12. Fresh stats against the ACTUAL shipped engine (this session, not the design-time CSV)

Everything in `STRESS_RULESET_BEST_SO_FAR.md`'s stats table was computed
by a **standalone re-simulation of the ruleset** against a pre-generated
CSV of raw reservoir counts (`master_dataset_v3.csv`) — fast, but it
never ran the real `decideBotMove()`, so it can't reflect anything the AI
itself does (including the new danger tuple). This session additionally
ran the **real engine**, full AI included, headless:

- **Win-type distribution** (n=50/mode, one run): 2P 70/26/4 (mid/end/draw
  %), 3P 42/40/20, 4P 32/44/26. Zero unresolved/stuck games across 150.
  Noisy at this sample size (~±14pt) — a full n=200/mode run was scoped
  but not completed this session (~7min runtime estimated, deferred).
- **Per-seat win-rate, 10 pinned boards × 10 trials/board** (the
  board-fairness methodology from v4 §1, extended): aggregate seat-win%
  came out much closer to even than an earlier n=50-*random*-boards
  sample had suggested — **4P in particular came out essentially dead
  even (21/19/18/21%)**, contradicting that earlier sample's apparent
  30% seat-1 edge. **Conclusion: most of the apparent turn-order skew in
  small random-board samples is board-sampling noise, not a structural
  bias in `decideBotMove()`** (which has no seat-specific logic anywhere
  in it — turn order is the only axis that could produce such a skew, and
  it washes out once enough distinct boards are sampled).
- **But per-board variance is real and sometimes large**: individual
  pinned boards showed genuine, large skews under repeated trials with
  only tie-break randomness varying (e.g. one 2P board went 9–1 in favor
  of seat 1 across 10 trials; two 3P boards shut seat 1 out entirely,
  0/10). This is a board-geometry effect (pipe layout / starting-position
  interaction), not a turn-order or AI bug — flagged as worth a closer
  look (e.g. dumping the actual tile layout for an outlier seed) if board
  fairness becomes a priority, but not investigated further this session.

## 13. Open items (new, in addition to v4's unchanged list)

- **No permanent sync files for v29** (`variants/current_v29.js` /
  `body_only_v29.html` equivalents) — every verification this session
  re-extracted from the HTML ad hoc into scratch files. Fine for one
  session's own iteration, but the next session should either set these
  up properly (per v4 §1's original discipline) or keep doing ad hoc
  extraction consistently — just don't let a stale synced copy silently
  diverge from the shipped HTML the way v4 §1 warned about for v27.
- **Dead grace-period code** in `afterTurnRevealComplete()` (the old
  `endgamePlayersRemaining`-queue branch, unreachable while
  `INSTANT_WIN_ENDGAME=true`) — not cleaned up, left as a documented
  historical fallback (§3.4).
- **Full n=200/mode win-type stats on the shipped engine** — only
  n=50/mode was run (§12); a tighter read was scoped but deferred for
  time.
- **3P draw-rate / board-outlier investigation** (§12's last bullet) —
  specific boards with large seat skew were found but not dug into.
- **Rules page**: explicitly deferred to a future session/chat per direct
  instruction ("Tomorrow, we make the rules page in another chat") — not
  started.
