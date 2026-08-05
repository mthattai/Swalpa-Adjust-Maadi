# Swalpa Adjust Maadi -- Technical Specification

Purpose of this document: a complete, self-contained record of every rule,
formula, and design choice underlying the game, sufficient to re-implement it
from scratch without needing to read the original source. This is a design
record for the author, not player-facing material. Visual/rendering details
(exact pixel sizes, colours used purely for decoration, canvas drawing code)
are intentionally omitted or kept brief, since those can be redesigned freely
without changing how the game plays.

All source line numbers refer to the shipped file
`index.html` at the time of writing.


## 1. Premise

The player is a contractor in a fictional Bangalore neighbourhood, competing
against one or more rival contractors (human or AI) to supply the most houses
with both power and water. Meanwhile, a random, unpredictable "Roadworks"
event periodically destroys infrastructure. The residents of the houses are
not represented as agents; they are simply whoever the current move's
scoring assigns a house to. The game is framed as dark comedy: "no matter who
wins, the residents lose."


## 2. Board and supply edges

- The board is an N x N grid, where N is 6, 7, or 8 depending on the chosen
  board size.
- Two independent supplies enter the board from its edges:
  - **Power ("red")** enters from the **top edge** (row 0) and the **bottom
    edge** (row N-1).
  - **Water ("blue")** enters from the **left edge** (column 0) and the
    **right edge** (column N-1).
- A space is **power-connected** if there exists an unbroken chain of tiles,
  starting from a top- or bottom-edge tile whose edge-facing-the-board-edge
  carries power, through adjacent tiles whose shared border both carry power
  on that specific side, ending at the space in question. Water-connectivity
  is defined identically, sourced from the left/right edges instead.
- This is computed as two independent breadth-first floods over the grid
  (`connectivity()`, source lines 465-501): one flood for power seeded from
  every tile on row 0/row N-1 whose N/S edge carries power, one flood for
  water seeded from every tile on column 0/column N-1 whose W/E edge carries
  water. A flood only propagates from tile A to an adjacent tile B if **both**
  A's edge facing B and B's edge facing A carry that colour.
- Connectivity is recomputed from scratch after every single card placement
  (there is no incremental/cached connectivity state).


## 3. Cards and their edges

Every tile on the board has one of four kinds: **Straight (S)**, **Bend
(B)**, **Park (P)**, or **Roadworks-rubble (X)**. Each kind, combined with a
rotation, defines which of its four sides (N/E/S/W) carry power ("R") and/or
water ("B" in the code, meaning "blue") -- see `edgesOf()` /
`straightEdges()` / `bendEdges()`, lines 381-398:

- **Straight, rotation 0 or 2** (both rotations give the identical
  configuration, since the check is `rot % 2`): N and S carry power; E and W
  carry water.
- **Straight, rotation 1 or 3**: N and S carry water; E and W carry power.
  (Only rotations 0 and 1 of Straight are ever actually used when building
  decks -- see Β§4/Β§5 -- since rotation 2/3 are redundant with 0/1 for this
  kind.)
- **Bend**, one of 4 distinct rotations, each routing power through one pair
  of adjacent sides and water through the other pair:
  - rot 0: power on N+E, water on S+W
  - rot 1: power on E+S, water on W+N
  - rot 2: power on S+W, water on N+E
  - rot 3: power on W+N, water on E+S
- **Park**: carries neither colour on any side (`blankEdges()`). A card can
  never be placed on a Park; it is permanent for the rest of the game unless
  destroyed by Roadworks (see Β§10).
- **Roadworks-rubble (X)**: also carries neither colour. Unlike a Park, a
  fresh card *can* be played on top of rubble in a later turn.

Every tile, connector or not, occupies exactly one board space at a time;
there is no concept of a tile carrying both a Straight and a Bend
simultaneously.


## 4. The setup deck (initial board)

At the start of every game (`buildSetupDeck()`, lines 402-420), the entire
board is filled in one pass, before any player has taken a turn:

- The number of Park spaces is fixed by board size: **6x6 -> 6 Parks, 7x7 ->
  8 Parks, 8x8 -> 10 Parks** (`parksConfig = {6:6, 7:8, 8:10}`). Verified
  directly: a 6x6 setup deck contains exactly 6 Parks out of 36 tiles, 7x7
  contains 8 out of 49, 8x8 contains 10 out of 64.
- The remaining `N*N - numParks` spaces are each independently filled with a
  uniformly random choice among exactly 6 connector orientations: Straight
  rot 0, Straight rot 1, Bend rot 0, Bend rot 1, Bend rot 2, Bend rot 3.
- All `N*N` tiles (the random connectors plus the Parks) are then combined
  into one list and Fisher-Yates shuffled, then placed onto the board
  row-major (row 0 left-to-right, then row 1, etc.).
- Immediately after the board is filled, connectivity is computed once, and
  **any space that already happens to be doubly-connected becomes a house
  immediately**, with no owner (owner stays `null`, rendered with a neutral
  roof colour). This is the only way an unowned house can exist; Roadworks
  never creates one directly (see Β§10).


## 5. The play deck

Once the board is set up, a **separate** deck of cards to be drawn and played
is built and shuffled (`buildAndShuffleDeck()`, lines 1030-1045). Its
composition scales with `effectiveN` (the number of players actually seated
at the table -- see Β§12 for exactly how this is set):

| Card | Count |
|---|---|
| Straight, rot 0 | 3 x effectiveN |
| Straight, rot 1 | 3 x effectiveN |
| Bend, rot 0 | 3 x effectiveN |
| Bend, rot 1 | 3 x effectiveN |
| Bend, rot 2 | 3 x effectiveN |
| Bend, rot 3 | 3 x effectiveN |
| Roadworks (X) | 2 x effectiveN |
| **Total** | **20 x effectiveN** |

This whole list is Fisher-Yates shuffled once, at the start of play, and
never reshuffled mid-game. Verified directly: effectiveN=2 produces a deck of
exactly 40 cards, effectiveN=3 produces 60, effectiveN=4 produces 80.

Every round consists of exactly one card played by each of the `effectiveN`
players in turn, for `TOTAL_ROUNDS = 20` rounds -- i.e. exactly `20 x
effectiveN` cards get played over a complete game. Since the deck is also
sized to exactly `20 x effectiveN` cards, **the deck is deliberately sized to
run out at the exact moment the 20th round's last card is played** -- the
game's fixed round limit and the deck's exhaustion are two ways of describing
the same underlying fact, not two independent constraints.

**Visible queue**: at any moment, the UI shows the next `VISIBLE_TILES = 8`
cards from the deck as a row of icons. The active/playable card is always the
one at the *end* of this row (`upcomingTiles[upcomingTiles.length-1]`).
Playing the active card pops it off the end and draws one new card onto the
front of the queue (`advanceTileQueue()`, lines 1062-1067) -- visually, this
reads as the whole row shifting over by one. In the last few turns of the
game, once the deck itself runs dry, the visible row simply shrinks below 8
cards rather than being padded with anything.


## 6. Turn structure and placement rules

A player may place the active card on any board space except:

- A Park (permanent, can never be played on).
- The exact space the *immediately preceding* move was played on (this is
  the only placement restriction tied to recency; older moves impose no
  restriction -- a card from several turns ago can be freely overwritten).

Any other space -- including a space that already holds an existing
connector card, or an existing house -- can be played on. Overwriting an
existing house's underlying tile is precisely the mechanism that can
disconnect it and potentially trigger a steal (Β§9).

**Two-step confirmation (human turns)**: clicking a valid space places the
card there as a *preview* (this already mutates the actual board state, so
connectivity/house-formation calculations immediately reflect it) and
highlights that space with a dashed border. Clicking the same space again
commits the move. Clicking a different valid space instead cancels the first
preview (restoring whatever was there) and previews the new space instead.

**Roadworks is not chosen by the player at all.** As soon as a Roadworks card
becomes the active card, the game already picks its target space and
executes the destruction (see Β§10) before the player does anything; the
player's only action is to tap the already-fixed, already-highlighted space
once to acknowledge it and move the turn along.


## 7. Houses: formation and ownership

Every single time any card is placed (`applyHouseAndRevenue()`, lines
974-1007, called immediately after every committed move, human or AI):

1. Connectivity is recomputed for the whole board.
2. **Steal check** (see Β§9) is evaluated first, but only at the exact space
   that was just played on.
3. **New-house formation**: the entire board is scanned. Every space that is
   now doubly-connected (power **and** water) and was not already a house
   becomes one, with its owner set to whichever player's turn this is. A
   single move can create more than one new house at once, if it happens to
   be the piece that completes several separate connections simultaneously
   -- all such houses are credited to the same mover.
4. **Scoring** (see Β§8) is then applied for the current player's entire set
   of connected, owned houses -- not just the one just played on or formed.

A house's roof colour is the visual indicator of its owner; a house formed
during initial setup has no owner (`null`) and is rendered with a neutral
colour, distinct from every player's colour, until someone claims it via
formation (impossible, since setup houses are already-formed) or theft.
Setup houses can only ever change hands via a steal, since the
new-house-formation branch above only fires for spaces that were *not*
already a house.


## 8. Scoring

On a player's own turn, immediately after their move resolves, they earn
**exactly one point for every house that is simultaneously (a) owned by
them and (b) connected to both power and water at that exact moment** --
regardless of whether that particular house was involved in this turn's
move at all. This is a full re-scan of the whole board every turn
(`applyHouseAndRevenue()`'s final loop), not an incremental "+1 for what
just happened."

Consequences that follow directly from this:

- A house that is only partially connected (power but not water, or vice
  versa) earns nothing, even if owned.
- A player's score does **not** change on an opponent's turn, even though
  their own houses may remain fully connected throughout that turn. Score is
  evaluated once per round, only on the scoring player's own turn.
- There is no cap on how many points a single turn can award; if a player
  owns ten connected houses, their turn awards ten points.


## 9. Stealing

The steal check (lines 983-994) runs immediately before the new-house/scoring
logic described above, and only ever examines the **exact space that was
just played on** -- never any other space on the board, regardless of what
else may have become disconnected as a side-effect of this move.

The condition, precisely:

1. The just-played space already has a house on it (from any earlier turn,
   including initial setup).
2. After this move, that house is **fully disconnected** -- neither power
   nor water reaches it any more. (A house retaining even one of the two
   connections is explicitly protected from stealing.)
3. Among that house's four edge-adjacent neighbours (N/S/E/W; diagonals do
   not count), **at least one** is itself a house whose owner matches the
   disconnected house's **own, pre-existing owner** -- not the current
   mover's colour. An unowned (setup) house's "owner" for this purpose is
   `null`, which is itself a valid matching category (i.e. an unowned,
   disconnected house next to another unowned house counts as a match).

If all three conditions hold, the house's owner is reassigned to the current
mover, regardless of who owned it before. If the mover already owned it, this
is a no-op with no visible feedback. If ownership actually changed hands, a
"Good steal!" popup is shown (unless a background simulation is running --
see Β§16 -- in which case this feedback is deliberately suppressed).

Because ownership only changes when there is at least one same-owner
neighbour, an isolated disconnected house with no matching neighbours simply
sits there, unowned by anyone able to score from it (it earns nothing per
Β§8), until either play reconnects it or Roadworks destroys it outright.


## 10. Roadworks

When a Roadworks card becomes the active card (`beginTurn()`, lines
1075-1089), the following happens immediately and automatically, before any
player acts:

1. A target space `(r, c)` is chosen uniformly at random over the whole
   board, via `rng()` (see Β§13 for exactly what source this draws from).
2. If that space is not already rubble, it is overwritten to rubble
   (`{kind:'X', rot:0}`), and if it held a house, that house and its
   ownership are cleared outright (`house[r][c] = false; owner[r][c] =
   null`) -- Roadworks destroys a house's *existence*, it does not merely
   disconnect it (so it can never itself trigger a steal; the house is
   simply gone).
3. This target space becomes the pending/highlighted space, exactly as if a
   player had clicked to preview a move there.
4. A popup is shown with two lines: a randomly-drawn excuse (see below) on
   the first line, and the fixed phrase "Swalpa adjust maadi!" on the
   second. This popup auto-hides after exactly **2200 ms**.
5. The acting player (human or AI) then only needs to tap/confirm the
   already-highlighted space to move the turn along; no location choice is
   involved.

A destroyed Park or house's space is not permanently blocked -- a later card
can be played on top of the rubble like any other non-Park space, and if
that new placement happens to newly connect both supplies there, a fresh
house can form and be claimed there, per Β§7.

**Excuse selection** is deliberately decoupled from the main game-affecting
RNG. At the start of every game (`freshGame()`, lines 433-438), all 20
entries of `ROADWORKS_EXCUSES` (line 815) are Fisher-Yates shuffled using
plain `Math.random()` directly -- never the seeded stream -- and then drawn
through sequentially (`shuffledExcuses[excuseDrawIndex++]`), guaranteeing no
excuse repeats within a single game (verified: the maximum possible number of
Roadworks draws in any game, 8, for a 4-player game, is comfortably under the
20 available excuses, so the pool never needs to reshuffle mid-game). Because
this shuffle is never seeded, replaying the identical Today's-game board on
the same day produces a different excuse sequence each time, even though
every other aspect of the board is identical -- a deliberate choice, since
this has no bearing on scoring or AI behaviour and keeps repeated play from
feeling identical in every last detail.


## 11. Winning

The game ends the instant the deck's cards run out at the end of round 20
(the two are the same event, per Β§5). Winner determination
(`finishGame()`, lines 1246-1263): whichever player(s) have the strictly
highest final score win; if more than one player ties for the highest score,
it is declared a tie. A popup announces "Player X WINS" (or "It's a tie!"),
paired with the fixed second line "Residents LOSE" -- this popup also
auto-hides after 2200 ms. If the game that just ended was Today's game
specifically, a second popup (the share-score popup, Β§17) is queued to
appear automatically the moment the first one finishes hiding.


## 12. Game modes, player counts, board sizes

Every launch button sets a specific combination of `numPlayers`,
`vsComputerOn`, `effectiveN`, and `N` (board size), and always calls
`freshGame()` immediately, discarding any game in progress:

| Button | Mode | numPlayers | vsComputerOn | effectiveN | N |
|---|---|---|---|---|---|
| Today's game / Dig up | Today's game | 1 | true | 2 | 6 (forced) |
| 1 player | Random | 1 | true | 2 | unchanged |
| 2 player | Random | 2 | false | 2 | unchanged |
| 3 player | Random | 3 | false | 3 | unchanged |
| 4 player | Random | 4 | false | 4 | unchanged |
| 6x6 / 7x7 / 8x8 board | Random | unchanged | unchanged | unchanged | 6/7/8 |
| Random game | Random | unchanged | unchanged | unchanged | unchanged |

`effectiveN` is the number of turns actually taken per round: for 1-player
mode this is 2 (the human plus the single AI opponent occupying "Player 2"),
otherwise it equals `numPlayers` directly, since 2/3/4-player modes are all
humans taking turns on the same device with no AI seat at all. Today's game
always forces exactly 1-player, 6x6 -- there is no Today's-game variant at
other player counts or board sizes.

In any mode with more than one human player (2/3/4-player), there is no AI
opponent; every seat is a human clicking on the same device in sequence.


## 13. The RNG architecture

Two random-number sources exist side by side:

- **`Math.random()`**: the browser's own, genuinely non-deterministic
  source. Used directly whenever a value should differ every single time,
  with no way to reproduce it (Fisher-Yates shuffling the excuse order at
  the start of every game, per Β§10).
- **`mulberry32(seed)`** (lines 783-791): a small, fast, deterministic
  pseudorandom generator. Given the same integer seed, it produces the
  identical sequence of floating-point values in `[0, 1)` every time it is
  freshly instantiated -- not cryptographically secure, and not intended to
  be; it only needs a reasonable distribution for shuffling a deck and
  picking Roadworks targets.

The seed used throughout Today's-game mode is derived from the calendar date
alone: `seedFromDateString(todaysIndiaDateString())`
(`todaysIndiaDateString()`, lines 793-797, reads today's date specifically in
the `Asia/Kolkata` timezone, independent of the player's own device
timezone, so every player worldwide sees the same seed on what India
considers "today"; `seedFromDateString()`, lines 799-805, is a simple
rolling-hash of that YYYY-MM-DD string into a 32-bit integer).

**The `rng()` dispatcher** (lines 810-812) is what almost all gameplay code
actually calls, rather than reaching for either source directly:

```
function rng(){
  return isTodaysGame && seededRandomFn ? seededRandomFn() : Math.random();
}
```

This single function is why the exact same setup-deck-building code, the
exact same Roadworks-targeting code, and (as of the AI-fairness change
described in Β§14) the exact same AI-move-selection code can be shared,
unmodified, between Today's-game mode (where they must be reproducible) and
Random-game mode (where they must not be) -- the *only* thing that differs
between the two modes is which global flag `rng()` is currently checking, not
any duplicated logic.

`seededRandomFn` itself is a single, module-level mutable slot holding
whichever `mulberry32` instance is "live" at the moment -- switching modes,
starting Today's game, or (temporarily, and very deliberately) running the
background rank-simulation batch (Β§16) all work by swapping out what this
slot currently points to.


## 14. Today's-game determinism and fairness

Today's game's core guarantee is: every player who launches it on the same
India calendar date sees the identical starting board and deck order,
because `enterTodaysGameMode()` (lines 1448-1451) always constructs a fresh
`mulberry32` instance from the same, date-derived seed, and `freshGame()`
then consumes that instance's sequence via `rng()` in exactly the same order
every time (setup deck, then play deck).

**AI fairness**, a deliberate later design decision: the live AI opponent
(Player 2, in Today's-game mode) also draws its move-selection randomness
(Β§15) through `rng()`, the same shared, seeded stream -- it is not given its
own, separately-seeded source. Since the human player's own move choices
never consume any RNG call at all (clicking a space is not a random act), the
*position* of the shared stream at the start of any given AI turn depends
only on how many prior RNG-consuming events have occurred (setup, and any
Roadworks/AI-turns so far) -- which is identical for any two players who have
made the identical sequence of moves up to that point, regardless of *where*
on the board they chose to play. The practical consequence, verified directly
by driving two independent playthroughs through an identical fixed sequence
of human moves: the AI's responses at every single turn come out
byte-for-byte identical between the two playthroughs. This is what makes
comparing scores across different players on the same Today's-game board
meaningful -- the opponent is not "luckier" for one player than another,
given identical play.

Switching to Random-game mode (`enterRandomGameMode()`, lines 1452-1455)
simply sets `isTodaysGame = false` and nulls out `seededRandomFn`; every one
of the same code paths above then falls through `rng()` to `Math.random()`
instead, with no special-casing required anywhere else in the codebase.


## 15. The AI

A single function, `aiChooseSquare(color, drawTile, randFn)` (lines
1127-1179), governs every AI-controlled move in the game -- there is no
separate logic for "the Today's-game opponent" versus "an AI used in a
background simulation" versus any hypothetical future AI seat; only the
`randFn` argument passed in differs by context (defaulting to the global
`rng()` if omitted).

**Restricted search.** The AI does not evaluate the entire board every turn.
Each call:

1. Builds the list of all currently-valid candidate spaces (same
   restrictions as a human's turn: not a Park, not the immediately-preceding
   move's space).
2. Fisher-Yates shuffles that list using `randFn`.
3. Takes only the first N of the shuffled list (or all of them, if fewer
   than N are actually valid), where N is looked up per board size from
   `AI_SUBSET_SIZE_CONFIG`:

   | Board size (N) | Subset size |
   |---|---|
   | 6x6 | 18 |
   | 7x7 | 25 |
   | 8x8 | 32 |

   This is always exactly N *distinct* spaces when at least N exist, never a
   repeat, since the shuffle operates on a list with no duplicate entries to
   begin with. (The 6x6 value was briefly changed to 12 during calibration,
   on the reasoning that a lower value would let novice players see a
   higher/better rank more easily; that made the AI too easy to beat, and 18
   was restored and is now frozen for 6x6. The per-board-size table itself
   was added after an initial oversight: a single fixed value of 18 had been
   applied uniformly to all three board sizes, which is a much smaller
   *fraction* of the board on 7x7/8x8 than on 6x6, disproportionately
   weakening the AI on larger boards with no such effect intended. Only
   6x6's own value, used by Today's game, needed to stay unchanged for
   consistency with the already-frozen calibration and the existing
   `todaysScoreDiffs` comparison batch, which is always built at N=6.)

**Scoring each candidate.** For each of the (up to N) candidate spaces, the
AI hypothetically places the current card there, recomputes connectivity,
and computes:

- `diff` = (number of doubly-connected houses this player would own) minus
  (number the opponent owns), counting a newly-completed, currently-unowned
  house as belonging to this player.
- `myTerr` = total count of spaces connected to *either* colour (not
  necessarily both) -- a rough measure of board presence/territory.
- `completesHouse` = 1 if this exact placement newly completes a house that
  did not already exist, else 0.
- `isSteal` = 1 if this exact placement would trigger a legal steal at that
  space per the Β§9 rule (checked with the same matching-neighbour logic)
  **and** the pre-existing owner at that space is not this player already
  (i.e. re-confirming a house the AI already owns never counts as a "steal"
  bonus, even if the matching-neighbour condition happens to hold), else 0.
- A combined value:
  `val = diff + AI_ALPHA * decay * completesHouse + AI_ALPHA * decay * isSteal + AI_BETA * myTerr`
  where `AI_ALPHA = 0.7`, `AI_BETA = 0.1`, and `decay = max(0, (TOTAL_ROUNDS -
  roundNum) / TOTAL_ROUNDS)` -- linearly shrinking from just under 1 on round
  1 to exactly 0 on round 20, so the house-completion and steal bonuses
  matter progressively less (relative to raw territory-difference) as the
  game nears its end.

The candidate with the strictly highest `val` is chosen (ties go to whichever
candidate was scanned first, i.e. earliest in the shuffled order -- there is
no random tie-break). The board is restored to its actual state after each
hypothetical placement is scored, so this evaluation has no side-effects
until the real move is finally committed.


## 16. The 99-simulation rank-comparison system

Today's-game mode additionally computes, once per calendar date, a fixed
reference distribution of 99 fully-simulated AI-vs-AI game outcomes, so that
a real player's own score can be expressed as a percentile-style rank rather
than a bare number.

**When this runs**: automatically at page load, before the player can click
anything (not on every launch of Today's game -- see below). The "Let's
start digging" button is disabled and reads "Loading..." until this
completes, then its label and enabled state flip. Switching modes away and
back to Today's game later, or reloading into Today's game via the in-game
button, does **not** re-run this batch; it only ever runs once per full page
load, since the result is seeded from the date and would only ever come out
identical anyway.

**Two independent RNG roles**, deliberately kept as two separate `mulberry32`
instances (`runTodaysRankSimulations()`, lines 1423-1446, and
`runOneRankSimulation()`, lines 1381-1415):

1. **Per-game board/deck/Roadworks/Player-2 stream**: a fresh `mulberry32`
   instance, re-seeded from the identical date-derived seed, created anew
   before *each* of the 99 games individually. This drives that game's setup
   deck, play deck, Roadworks targeting, and Player 2's own subset choice
   (18, the 6x6 lookup value from Β§15's table, since Today's game is always
   6x6), via the ordinary `rng()` dispatcher -- meaning every one of the 99
   games plays out on the literal, identical board a real human would face
   that day, against the identical-strength opponent.
2. **Continuous cross-batch stream for Player 1**: a single `mulberry32`
   instance, seeded once at the very start of the whole 99-game batch, then
   left running -- never re-seeded -- across all 99 games in sequence. This
   is passed explicitly as `aiChooseSquare`'s `randFn` argument for Player
   1's moves only. Because it is never reset, game 2 continues consuming
   this stream wherever game 1 left off, and so on; this is the **only**
   source of variation between the 99 games (if Player 1 also used the
   per-game-reseeded stream, all 99 games would be identical to each other,
   since identical seed + identical deterministic AI logic on both sides
   produces identical results every time -- this was an actual bug caught
   and fixed during development).

Because both streams are always initialised from the same, fixed,
date-derived seed and always consumed in the same order, **the entire batch
of 99 results is itself fully reproducible**: any two independent page loads
on the same calendar date produce byte-for-byte identical `todaysScoreDiffs`
arrays and, since the per-game board/deck/Roadworks stream is separately
re-seeded from that same value each time, an identical actual playable board
for the human as well (verified directly: two independent page loads on the
same date produced identical 99-value arrays and identical player boards).

**Side-effect suppression.** Because these 99 games are simulated using the
exact same shared game functions the live, visible game uses (rather than a
separate, parallel implementation), a global flag `simulationModeActive` is
set for the duration of the batch specifically to suppress the one visual
side-effect those shared functions would otherwise trigger -- the "Good
steal!" popup (Β§9) -- since a background simulation the player never sees
should never surface UI. (This was also an actual, caught-in-testing bug:
the first working version of this batch let that popup leak onto the
pre-launch screen.) All of the live game's actual board/score/deck state is
saved before the batch begins and restored byte-for-byte afterward, and a
fresh, entirely unconsumed seeded stream is created for the player's own
upcoming game once the batch finishes -- the batch's own RNG consumption
never bleeds into what the player subsequently plays.

**Rank formula** (`computeTodaysRank()`, lines 894-898): let `myDiff` be the
player's own final `revenue[1] - revenue[2]`. The rank is

```
rank = (number of the 99 simulated score-differences strictly greater than myDiff) + 1
```

so 1 is the best possible outcome (nothing in the reference set beat it) and
100 is the worst (all 99 beat it). Ties are deliberately resolved in the
player's favour: a simulated value *equal to* the player's own score does
**not** count against them (only strictly-greater values do), so a player
who ties with every single one of the 99 simulations receives rank 1, not
100.


## 17. Share message

Built by `buildShareText()` (lines 905-916), only ever shown after a
completed Today's-game session (never after a Random game). Exact template
(`{...}` marking the only variable parts):

```
I played 'Swalpa adjust maadi', the game where you dig up Bangalore roads for fun!

{woman-worker-emoji} (me) {revenue[1]} vs. {revenue[2]} (rival) {man-worker-emoji}
{day} {month} rank: {rank text}.

Can you beat my rank in today's game?
tinyurl.com/adjust-maadi
```

- The date shown omits the year, since it is irrelevant to the message's
  point.
- `{rank text}` is produced by `formatRank()` (lines 900-903): exactly
  "First of 100" if rank is 1, otherwise `"{rank}{ordinal suffix} highest of
  100"` (e.g. "48th highest of 100"). The ordinal suffix
  (`ordinalSuffix()`, lines 882-890) follows standard English rules,
  including the 11th/12th/13th exception (these always take "th" regardless
  of their last digit, unlike 21st/22nd/23rd etc.).
- All emoji in this template are written as explicit Unicode codepoints in
  the source, not typed directly, specifically to avoid a transcription/
  encoding-corruption failure mode that was hit more than once during
  development.


## 18. Timing constants (gameplay-relevant, not purely cosmetic)

- `TOTAL_ROUNDS = 20` -- the fixed game length.
- `STAGE_DELAY_MS = 800` -- used both as the delay before the AI's chosen
  move is actually placed after being decided (so the human can see which
  square got highlighted before it commits), and as the pacing delay before
  the turn advances to the next player after a move resolves.
- Any popup shown with `autoHide = true` (Roadworks excuse popup, steal
  popup, game-over popup) hides itself automatically after exactly **2200
  ms**, during which player input is fully blocked (`popupBlocking`) --
  gameplay does not silently continue underneath a visible popup.
- `NUM_RANK_SIMULATIONS = 99`.
- `AI_SUBSET_SIZE_CONFIG` -- see Β§15 for the full, per-board-size table (18 /
  25 / 32 for 6x6 / 7x7 / 8x8 respectively).


## 19. File and deployment structure

The entire game is a single, self-contained HTML file with inline CSS and
JavaScript -- no build step, no external JS/CSS libraries, no network
dependency for gameplay itself (Cloudflare Analytics is loaded but is
purely for traffic measurement and has no bearing on anything above). The
only other files that need to sit alongside it for full functionality are:

- `preview.png` -- the Open Graph / Twitter card image used when the game's
  link is shared.
- `favicon.ico`, `favicon-32x32.png`, `apple-touch-icon.png` -- browser/OS
  icons.
- `tutorial.html` -- a separate, self-contained (all its own screenshots
  embedded as base64 inline images, no external file references at all)
  player-facing walkthrough, linked from the main game via a "Tutorial"
  button that opens it in a new tab, with its own "Play" button that opens
  the game fresh in a new tab in turn. This tutorial file is explicitly
  *not* required for the game to function; it exists purely to help new
  players.

All of these live at the root of the same directory when deployed (currently
GitHub Pages, with the main game file named `index.html` there).
