# Swalpa Adjust Maadi -- Technical Specification

Purpose of this document: a complete, self-contained record of every rule,
formula, and design choice underlying the game, sufficient to re-implement it
from scratch without needing to read the original source. This is a design
record for the author, not player-facing material. Visual/rendering details
(exact pixel sizes, colours used purely for decoration, canvas drawing code)
are intentionally omitted or kept brief, since those can be redesigned freely
without changing how the game plays. Every number and rule below was checked
directly against the shipped implementation while writing this, not recalled
from memory.

All source line numbers refer to the shipped file
`swalpa_adjust_maadi_v24.html` at the time of writing.


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
  (`connectivity()`), one flood for power seeded from every tile on row 0/row
  N-1 whose N/S edge carries power, one flood for water seeded from every
  tile on column 0/column N-1 whose W/E edge carries water. A flood only
  propagates from tile A to an adjacent tile B if **both** A's edge facing B
  and B's edge facing A carry that colour.
- Connectivity is recomputed from scratch after every single card placement
  (there is no incremental/cached connectivity state).


## 3. Cards and their edges

Every tile on the board has one of four kinds: **Straight (S)**, **Bend
(B)**, **Park (P)**, or **Roadworks-rubble (X)**. Each kind, combined with a
rotation, defines which of its four sides (N/E/S/W) carry power ("R") and/or
water ("B" in the code, meaning "blue") -- see `edgesOf()` /
`straightEdges()` / `bendEdges()`:

- **Straight, rotation 0 or 2** (both rotations give the identical
  configuration, since the check is `rot % 2`): N and S carry power; E and W
  carry water. This is referred to elsewhere in this document (and in the
  in-game rules diagram) as "Straight A."
- **Straight, rotation 1 or 3**: N and S carry water; E and W carry power
  ("Straight B"). (Only rotations 0 and 1 of Straight are ever actually used
  when building decks -- see Β§4/Β§5 -- since rotation 2/3 are redundant with
  0/1 for this kind.)
- **Bend**, one of 4 distinct rotations, each routing power through one pair
  of adjacent sides and water through the other pair:
  - rot 0 ("Bend A"): power on N+E, water on S+W
  - rot 1 ("Bend B"): power on E+S, water on W+N
  - rot 2 ("Bend C"): power on S+W, water on N+E
  - rot 3 ("Bend D"): power on W+N, water on E+S
- **Park**: carries neither colour on any side (`blankEdges()`). A card can
  never be placed on a Park; it is permanent for the rest of the game unless
  destroyed by Roadworks (see Β§11). See Β§4 for the deliberate, non-uniform
  rule governing where Parks are placed at setup.
- **Roadworks-rubble (X)**: also carries neither colour. Unlike a Park, a
  fresh card *can* be played on top of rubble in a later turn.

Every tile, connector or not, occupies exactly one board space at a time;
there is no concept of a tile carrying both a Straight and a Bend
simultaneously.

"Straight A/B" and "Bend A/B/C/D" are the exact labels now used in the
in-game rules page's card-composition diagram (Β§5), generated directly from
the same `DECK_COMPOSITION` list described there -- not a separate naming
scheme invented for this document.


## 4. The setup deck (initial board)

At the start of every game (`initFreshBoard()`), the entire board is filled
in one pass, before any player has taken a turn. This now has three distinct
pieces, each a deliberate design choice, not simply "shuffle everything
together":

**4.1 Park count.** The number of Park spaces is fixed by board size: **6x6
-> 6 Parks, 7x7 -> 8 Parks, 8x8 -> 10 Parks** (`parksConfig = {6:6, 7:8,
8:10}`). Verified directly: a 6x6 setup deck contains exactly 6 Parks out of
36 tiles, 7x7 contains 8 out of 49, 8x8 contains 10 out of 64.

**4.2 Park placement rule** (`pickParkPositions()`, `isBadParkOffset()`).
Parks are *not* placed by uniform-random shuffle. Instead, positions are
chosen so that **no single board space is edge-adjacent to more than one
Park**.

The reasoning behind this rule, which motivated the whole feature: classify
every non-Park space by how many of its edge-adjacent neighbours (board
edges themselves always count as open, never as a blocking neighbour) are
Parks:

| Closed (Park) neighbours | What that space can do |
|---|---|
| 0 | Can host a house *and* channel both colours through |
| 1 | Can host a house *and* channel one colour through |
| 2 | Can host a house *or* channel one colour -- never both, whichever the placed card's orientation is chosen for |
| 3 or 4 | Genuinely useless for either purpose |

This is an exact classification (verified by brute-force enumeration over
every possible closed-side-count and every card orientation), and it depends
only on the *count* of Park-neighbours, never on which specific sides they
occupy. A space acquires a second closed side specifically when two Parks
are positioned such that they share a common neighbour -- which happens for
exactly three relative offsets between a pair of Parks: two apart in a
straight line, or diagonally adjacent. Every other spacing between two
Parks, including the two being directly *adjacent* to each other, is
completely safe and creates no degraded space at all.

`isBadParkOffset(p1, p2)` checks precisely this:

```
dr = abs(p1.row - p2.row), dc = abs(p1.col - p2.col)
bad if (dr==0 and dc==2) or (dr==2 and dc==0) or (dr==1 and dc==1)
```

`pickParkPositions(N, numParks)` places parks one at a time: shuffle every
board square, then for each Park in turn scan the shuffled list for the
first square that has no bad-offset relationship with any Park already
placed. If this process ever fails to place all `numParks` Parks (no valid
square remains for one of them), the entire attempt is discarded and
restarted from a fresh shuffle, up to 50 times; a final fallback (greedily
take whatever safe squares can be found, then fill any still-unplaced Parks
on the first available squares regardless of safety) exists purely as a
safety net and is not expected to trigger at the park counts this game
actually uses.

This is not a cosmetic distinction -- the true combinatorial ceiling (the
largest number of Parks that can *ever* satisfy this constraint on a given
board, found via extensive randomised search, not derived analytically) is
**12 for 6x6, 15 for 7x7, 19 for 8x8** -- comfortably above the 6/8/10
actually used, so a valid placement is found on the very first shuffled
attempt essentially every time in practice (verified: mean ~1.0-1.3 attempts
needed across 300 independent trials at the live park counts). Pushing park
counts much higher than today's values would eventually make placement
slow, then (beyond the ceiling) outright impossible -- see the design notes
in this section's own git history/chat log for the fuller derivation if this
is ever revisited. A secondary, deliberately avoided property of pushing
toward that ceiling: the *set* of valid maximal placements shrinks sharply as
density rises, and what remains is dominated by two efficient-but-repetitive
patterns (Parks paired up directly adjacent to each other, and Parks
clustered along the board's own edge) rather than the varied, spread-out
arrangements a moderate park count produces -- i.e. board "character" and
placement density trade off against each other under this specific rule,
which is part of why the park counts above were deliberately kept well
below the ceiling rather than pushed toward it.

**4.3 Connector fill.** Every non-Park space is filled independently, drawn
**with replacement** (i.e. the same orientation can and will recur many
times across different squares -- this is not a finite, shuffled pool the
way the play deck in Β§5 is), from a distribution weighted to match the *same
relative proportions* as the ongoing play deck's own composition (Β§5),
specifically excluding Roadworks (which is never part of the initial board).
Concretely, the draw is uniform over a list containing each connector
orientation repeated according to its `countPerPlayer` value from the shared
`DECK_COMPOSITION` list (Straight A x4, Straight B x2, each of the four Bends
x3 -- 18 entries total, so e.g. Straight A is drawn with probability 4/18).
Because this list is derived from the exact same `DECK_COMPOSITION` the play
deck itself is built from, a future change to that one list automatically,
correctly changes both the initial board's bias and the ongoing deck's
composition together -- there is no separate, second place that needs
updating to keep them in sync.

**4.4 Setup-house cutoff.** Immediately after the board is filled,
connectivity is computed once, and any space that already happens to be
doubly-connected becomes a house immediately, with no owner (owner stays
`null`, rendered with a neutral roof colour) -- this is the only way an
unowned house can exist; Roadworks never creates one directly (see Β§11). If
this produces **more than 4** such houses, the entire board (Park placement
and connector fill both) is discarded and regenerated from scratch, repeating
until a board with 4 or fewer setup houses is found, capped at 100 attempts
(a defensive limit only -- the underlying rejection rate is roughly 5-15%
depending on board size, so this virtually always resolves within one or two
attempts). This exists specifically to remove the thin tail of occasional
boards that would otherwise start with 5-11 houses already formed before a
single move is played.


## 5. The play deck

Once the board is set up, a **separate** deck of cards to be drawn and
played is built and shuffled (`buildAndShuffleDeck()`). Both this deck and
the setup fill's own weighting (Β§4.3) are derived from one shared list,
`DECK_COMPOSITION`, so there is exactly one place in the source that defines
"how many of each card exist" -- changing a `countPerPlayer` value there
changes both the live game's actual behaviour and the in-game rules page's
own displayed card-composition diagram together, since that diagram is
rendered directly from this same list rather than a separately maintained
description.

| Card | Count per player |
|---|---|
| Straight A (rot 0) | 4 x effectiveN |
| Straight B (rot 1) | 2 x effectiveN |
| Bend A (rot 0) | 3 x effectiveN |
| Bend B (rot 1) | 3 x effectiveN |
| Bend C (rot 2) | 3 x effectiveN |
| Bend D (rot 3) | 3 x effectiveN |
| Roadworks (X) | 2 x effectiveN |
| **Total** | **20 x effectiveN** |

(`effectiveN`, the number of players actually seated at the table, is set
per Β§13.) The Straight A/B split is deliberately uneven -- 4:2 rather than an
even 3:3 -- because "Straight A" is the orientation whose two connectors
align with each colour's actual supply direction (power runs top-to-bottom
through it, water runs left-to-right), while "Straight B" runs directly
against both colours' natural direction and was found, in play, to be
disproportionately awkward for genuinely extending either colour's reach
compared to how often it showed up. The four Bends are left untouched at an
even 3-each, since they don't show this same asymmetry between each other.

This whole list is Fisher-Yates shuffled once, at the start of play, and
never reshuffled mid-game. Verified directly: effectiveN=2 produces a deck of
exactly 40 cards, effectiveN=3 produces 60, effectiveN=4 produces 80 -- the
total size formula itself (20 x effectiveN) is unchanged by the 4:2 split,
since it only redistributes cards within the two Straight rows.

Every round consists of exactly one card played by each of the `effectiveN`
players in turn, for `TOTAL_ROUNDS = 20` rounds -- i.e. exactly `20 x
effectiveN` cards get played over a complete game. Since the deck is also
sized to exactly `20 x effectiveN` cards, **the deck is deliberately sized to
run out at the exact moment the 20th round's last card is played** -- the
game's fixed round limit and the deck's exhaustion are two ways of describing
the same underlying fact, not two independent constraints.

**Visible queue.** At any moment, the UI shows the next `VISIBLE_TILES = 8`
cards from the deck as a row of icons, each overlapping the next (left 2/3
of each card visible, right 1/3 covered by the card to its right) so the row
reads as a physical stack rather than 8 separate boxes. The active/playable
card is always the one at the *end* of this row
(`upcomingTiles[upcomingTiles.length-1]`). Playing the active card pops it
off the end and draws one new card onto the front of the queue
(`advanceTileQueue()`) -- visually, this reads as the whole row shifting over
by one.

The underlying `upcomingTiles` array does genuinely shrink once the deck
itself runs dry near the end of the game (nothing new gets unshifted onto
the front once `nextDeckCard()` starts returning null) -- but the on-screen
row never visually shrinks below its full 8-slot width and position:
`renderDrawArea()` always reserves the full 8-slot span, filling in from the
*left* with plain, blank placeholder cards for however many real cards are
currently missing (`numBlanks = VISIBLE_TILES - upcomingTiles.length`), so
the deck's location and width on screen stay constant throughout the whole
game, right up to the final card.


## 6. Turn structure and placement rules

A player may place the active card on any board space except:

- A Park (permanent, can never be played on).
- The exact space the *immediately preceding* move was played on (this is
  the only placement restriction tied to recency; older moves impose no
  restriction -- a card from several turns ago can be freely overwritten).

Any other space -- including a space that already holds an existing
connector card, or an existing house -- can be played on. Overwriting an
existing house's underlying tile is precisely the mechanism that can
disconnect it and potentially trigger a steal (Β§10).

**Two-step confirmation (human turns)**: clicking a valid space places the
card there as a *preview* (this already mutates the actual board state, so
connectivity/house-formation calculations immediately reflect it) and
highlights that space with a dashed border. Clicking the same space again
commits the move -- this always takes priority over every other check in the
click handler (e.g. it is checked before the "out of tries" condition below,
so confirming an already-previewed move never itself costs a try). Clicking
a different valid space instead cancels the first preview (restoring
whatever was there) and previews the new space instead, at the cost of one
try (see Β§7).

**Roadworks is not chosen by the player at all.** As soon as a Roadworks card
becomes the active card, the game already picks its target space and
executes the destruction (see Β§11) before the player does anything; the
player's only action is to tap the already-fixed, already-highlighted space
once to acknowledge it and move the turn along. Roadworks turns are entirely
outside the tries/hint system in Β§7 -- there is nothing to spend a try or a
hint on when the space isn't a choice at all.


## 7. Tries, hints, and Practice mode

None of this existed in earlier versions of the game; all three are closely
related, sharing much of the same state and guard logic.

**7.1 Tries.** Each turn (reset to 5 in `beginTurn()`), a player has 5 tries
-- previewing a *new*, different space (Β§6) costs one try; re-clicking the
already-previewed space to commit never costs a try, regardless of how many
tries remain, since the commit-check runs first, before the tries check, in
the click handler. Once tries reach 0, clicking any space other than the
currently-previewed one is blocked (`"You've run out of tries."`), but the
player can still commit whatever is currently previewed. Five dot indicators
next to the round counter show remaining tries directly (`triesDots`); a
dot is shown in its "used" state once its index is `>= triesRemaining`.
Tries have no bearing on Roadworks turns at all, since there is no space to
choose there in the first place (Β§6).

**7.2 Hints.** A "Hint" button, when clicked, calls the exact same AI
move-selection function real opponents use, `aiChooseSquare(turn,
currentDraw, Math.random)`, and previews whatever square it returns, via the
same preview mechanism a manual click would use. This deliberately uses
`Math.random` specifically, **never** the shared `rng()`/seeded stream --
even in Today's-game mode -- so that using a hint can never perturb the
position of the seeded stream the live AI opponent and Today's-game fairness
guarantee (Β§15) depend on; a hint is the human player's own action, and (like
excuse selection, Β§11) is deliberately kept outside the RNG stream that must
stay position-independent of human choices.

Using a hint (outside Practice mode) immediately spends the turn's one
allowed hint (`hintUsedThisTurn = true`, reset each turn by `beginTurn()`)
**and** sets `triesRemaining = 0` -- meaning outside Practice mode, taking a
hint is effectively a commitment to that square: with zero tries left, the
only legal action remaining that turn is to click the same, now-previewed
square again to confirm it. A running count of hints used this entire game,
`hintsUsedThisGame` (reset to 0 by `freshGame()`), is tracked purely for the
share message (Β§18): if greater than 0 when a Today's-game session ends, the
share text appends `(with N hint/hints)` after the rank.

The button's own guard conditions, checked in this order, each with its own
distinct message: game already over (silently does nothing); it's currently
the AI's turn (`"Please wait for Player 2."`); a popup is showing or a move
is still resolving (`"Give it a moment."`); the active card is Roadworks,
whose space is already fixed and not a choice at all
(`"Tap the highlighted roadworks card."`); a hint was already used this turn
and Practice mode is off (`"You've used up your hint."`). The button is also
visually greyed (a CSS class only, never the native `disabled` attribute --
every one of the above cases must remain clickable so its explanatory
message can still show) whenever any of these conditions currently holds.

**7.3 Practice mode.** A separate toggle button, `"Practice: on/off"`,
orthogonal to (but mutually exclusive with) Today's-game mode: turning
Practice on always forces Random-game mode (`enterRandomGameMode()`); launching
Today's game via either launch button explicitly turns Practice back off
first. Toggling Practice on while already in a Random game reuses the
existing board/deck seed (`freshGame(true)`, same "Reset board" mechanism as
Β§14); toggling it on from Today's-game mode instead generates a genuinely new
random board, since there is no sensible "existing Random-game seed" to
reuse in that case.

While Practice mode is on: tries never deplete and are never checked
(`!practiceMode` guards every place `triesRemaining` would otherwise matter,
Β§7.1) and hints have no per-turn limit (`practiceMode || !hintUsedThisTurn`
in every relevant check, Β§7.2) -- a player can take as many hints per turn as
they like, and can still freely preview a different square afterward, unlike
outside Practice mode. The five tries-dots are deliberately still rendered,
but always fully in their "used" (grey) visual state regardless of the
actual, functionally-irrelevant value of `triesRemaining` underneath --
showing a "5 of 5 remaining" state that would then never actually deplete
was considered more misleading than simply never suggesting a limit exists
in the first place.


## 8. Houses: formation and ownership

Every single time any card is placed (`applyHouseAndRevenue()`, called
immediately after every committed move, human or AI):

1. Connectivity is recomputed for the whole board.
2. **Steal check** (see Β§10) is evaluated first, but only at the exact space
   that was just played on.
3. **New-house formation**: the entire board is scanned. Every space that is
   now doubly-connected (power **and** water) and was not already a house
   becomes one, with its owner set to whichever player's turn this is. A
   single move can create more than one new house at once, if it happens to
   be the piece that completes several separate connections simultaneously
   -- all such houses are credited to the same mover.
4. **Scoring** (see Β§9) is then applied for the current player's entire set
   of connected, owned houses -- not just the one just played on or formed.

A house's roof colour is the visual indicator of its owner; a house formed
during initial setup has no owner (`null`) and is rendered with a neutral
colour, distinct from every player's colour, until someone claims it via
formation (impossible, since setup houses are already-formed) or theft.
Setup houses can only ever change hands via a steal, since the
new-house-formation branch above only fires for spaces that were *not*
already a house.


## 9. Scoring

On a player's own turn, immediately after their move resolves, they earn
points for every house that is simultaneously (a) owned by them and (b)
connected to both power and water at that exact moment -- regardless of
whether that particular house was involved in this turn's move at all. This
is a full re-scan of the whole board every turn (`applyHouseAndRevenue()`'s
final loop), not an incremental "+X for what just happened."

**Each qualifying house's own point value is its board-distance to the
nearest edge, plus 1**: `points = min(row, N-1-row, col, N-1-col) + 1`. A
house directly on the border (row 0, row N-1, column 0, or column N-1)
scores 1; each step further from every edge adds 1. The maximum possible
value is 3 on a 6x6 board and 4 on both 7x7 and 8x8 (a 7x7 board's single,
true centre space and an 8x8 board's four centre-most spaces both sit
exactly 3 steps from their nearest edge). This was originally a flat 1
point per house regardless of position; the change was deliberate, to
reward building toward the harder-to-reach centre rather than treating
every connected house as equally valuable.

Consequences that follow directly from this:

- A house that is only partially connected (power but not water, or vice
  versa) earns nothing, even if owned.
- A player's score does **not** change on an opponent's turn, even though
  their own houses may remain fully connected throughout that turn. Score is
  evaluated once per round, only on the scoring player's own turn.
- There is no cap on how many points a single turn can award; if a player
  owns ten connected houses, their turn awards the sum of all ten houses'
  own point values.
- The turn-end "+X" flash shown on the board (and the scorebox's own,
  matching "+X" for that player) now reflects each individual scoring
  square's own point value -- different squares scoring in the same turn
  can show different amounts, rather than every square always showing "+1."


## 10. Stealing

The steal check runs immediately before the new-house/scoring logic
described above, and only ever examines the **exact space that was just
played on** -- never any other space on the board, regardless of what else
may have become disconnected as a side-effect of this move.

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
see Β§17 -- in which case this feedback is deliberately suppressed).

Because ownership only changes when there is at least one same-owner
neighbour, an isolated disconnected house with no matching neighbours simply
sits there, unowned by anyone able to score from it (it earns nothing per
Β§9), until either play reconnects it or Roadworks destroys it outright.


## 11. Roadworks

When a Roadworks card becomes the active card (`beginTurn()`), the following
happens immediately and automatically, before any player acts:

1. A target space `(r, c)` is chosen uniformly at random over the whole
   board, via `boardRNG()` (see Β§14 for exactly what source this draws from).
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
   second. This popup auto-hides after `LONG_DELAY_MS` (see Β§19).
5. The acting player (human or AI) then only needs to tap/confirm the
   already-highlighted space to move the turn along; no location choice is
   involved, and (Β§7) no try or hint interaction applies to this turn at all.

A destroyed Park or house's space is not permanently blocked -- a later card
can be played on top of the rubble like any other non-Park space, and if
that new placement happens to newly connect both supplies there, a fresh
house can form and be claimed there, per Β§8.

**Excuse selection** is deliberately decoupled from the main game-affecting
RNG. At the start of every game (`freshGame()`), all entries of
`ROADWORKS_EXCUSES` are Fisher-Yates shuffled using plain `Math.random()`
directly -- never the seeded stream -- and then drawn through sequentially,
guaranteeing no excuse repeats within a single game (the pool is comfortably
larger than the maximum possible number of Roadworks draws in any one game,
so it never needs to reshuffle mid-game). Because this shuffle is never
seeded, replaying the identical Today's-game board on the same day produces
a different excuse sequence each time, even though every other aspect of the
board is identical -- a deliberate choice, since this has no bearing on
scoring or AI behaviour and keeps repeated play from feeling identical in
every last detail.


## 12. Winning

The game ends the instant the deck's cards run out at the end of round 20
(the two are the same event, per Β§5). Winner determination
(`finishGame()`): whichever player(s) have the strictly highest final score
win; if more than one player ties for the highest score, it is declared a
tie. A popup announces "Player X WINS" (or "It's a tie!"), paired with the
fixed second line "Residents LOSE" -- this popup also auto-hides after
`LONG_DELAY_MS` (Β§19). If the game that just ended was Today's game
specifically, a second popup (the share-score popup, Β§18) is queued to
appear automatically the moment the first one finishes hiding.


## 13. Game modes, player counts, board sizes

Every launch button sets a specific combination of `numPlayers`,
`vsComputerOn`, `effectiveN`, and `N` (board size), and always calls
`freshGame()` immediately, discarding any game in progress:

| Button | Mode | numPlayers | vsComputerOn | effectiveN | N |
|---|---|---|---|---|---|
| Today's game / Dig up | Today's game | 1 | true | 2 | 7 (forced) |
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
always forces exactly 1-player, 7x7 -- there is no Today's-game variant at
other player counts or board sizes. (7x7 was chosen over the originally-used
6x6 after direct comparison found it noticeably less prone to the small
board's edges and corners getting boxed in by Parks, without yet being so
large that a full game's fixed 20-round budget leaves much of the board
never touched by a player at all -- 8x8 was judged too big for that budget
and deliberately kept as a Random-game-only, more free-form "practice size"
rather than adopted for Today's game.)

Both Today's-game launch buttons also explicitly force Practice mode (Β§7.3)
off, since Practice and Today's-game are mutually exclusive.

In any mode with more than one human player (2/3/4-player), there is no AI
opponent; every seat is a human clicking on the same device in sequence.


## 14. The RNG architecture

Two random-number sources exist side by side:

- **`Math.random()`**: the browser's own, genuinely non-deterministic
  source. Used directly whenever a value should differ every single time,
  with no way to reproduce it (Fisher-Yates shuffling the excuse order at
  the start of every game, per Β§11; and, in Random-game mode, AI move
  choice -- see below).
- **`mulberry32(seed)`**: a small, fast, deterministic pseudorandom
  generator. Given the same integer seed, it produces the identical sequence
  of floating-point values in `[0, 1)` every time it is freshly instantiated
  -- not cryptographically secure, and not intended to be; it only needs a
  reasonable distribution for shuffling a deck and picking Roadworks targets.

The seed used throughout Today's-game mode is derived from the calendar date
alone: `seedFromDateString(todaysIndiaDateString())`
(`todaysIndiaDateString()` reads today's date specifically in the
`Asia/Kolkata` timezone, independent of the player's own device timezone, so
every player worldwide sees the same seed on what India considers "today";
`seedFromDateString()` is a simple rolling-hash of that YYYY-MM-DD string
into a 32-bit integer).

**Two separate dispatchers**, not one, exist over these sources -- this is a
deliberate split, not an accident of two similarly-named functions:

```
function rng(){
  return isTodaysGame && seededRandomFn ? seededRandomFn() : Math.random();
}
function boardRNG(){
  return isTodaysGame && seededRandomFn ? seededRandomFn() : randomBoardRandomFn();
}
```

`rng()` is what AI move-choice uses by default (`aiChooseSquare()`'s own
`randFn` parameter defaults to `rng`). `boardRNG()` is used everywhere else
gameplay-affecting randomness is needed: board setup (Β§4), deck shuffling
(Β§5), and Roadworks-target selection (Β§11).

In Today's-game mode the two are functionally identical, since both check
`isTodaysGame && seededRandomFn` first and Today's game only ever has the
one, date-derived seed active. They diverge specifically in Random-game
mode: `rng()` falls through to plain, unseeded `Math.random()` (so AI
behaviour in a Random game is never reproducible), while `boardRNG()` falls
through to `randomBoardRandomFn` -- a *separate* `mulberry32` instance, seeded
from a freshly-drawn random integer (`randomBoardSeed`) each time a genuinely
new Random game starts, but deliberately **not** re-drawn when
`freshGame(reuseSeed=true)` is called instead. This is exactly what "Reset
board" (`resetBoardBtn`, calling `freshGame(true)`) relies on: since
`randomBoardSeed` itself is remembered from the game currently in progress
and only a *fresh* `mulberry32` instance (not a fresh *seed*) is created,
reconstructing the identical board and deck order for a Random game while
leaving Roadworks' own excuse order and (per the divergence above) AI
behaviour genuinely different each time it's replayed. Practice mode's own
seed-reuse logic (Β§7.3) works via this exact same mechanism.

`seededRandomFn` itself is a single, module-level mutable slot holding
whichever `mulberry32` instance is "live" at the moment for Today's-game
purposes -- switching modes, starting Today's game, or (temporarily, and
very deliberately) running the background rank-simulation batch (Β§17) all
work by swapping out what this slot currently points to.


## 15. Today's-game determinism and fairness

Today's game's core guarantee is: every player who launches it on the same
India calendar date sees the identical starting board and deck order,
because `enterTodaysGameMode()` always constructs a fresh `mulberry32`
instance from the same, date-derived seed, and `freshGame()` then consumes
that instance's sequence via `boardRNG()` in exactly the same order every
time (setup deck, then play deck).

**AI fairness**, a deliberate design decision: the live AI opponent (Player
2, in Today's-game mode) also draws its move-selection randomness (Β§16)
through `rng()`, the same shared, seeded stream -- it is not given its own,
separately-seeded source. Since the human player's own move choices never
consume any RNG call at all (clicking a space is not a random act), the
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
given identical play. (Using a hint, Β§7.2, deliberately draws from
`Math.random` rather than `rng()` specifically to preserve this guarantee --
a human choosing to consult a hint must never itself shift the AI's later
behaviour.)

Switching to Random-game mode (`enterRandomGameMode()`) simply sets
`isTodaysGame = false`; every one of the same code paths above then falls
through `rng()`/`boardRNG()` to `Math.random()`/`randomBoardRandomFn`
respectively (Β§14), with no special-casing required anywhere else in the
codebase.


## 16. The AI

A single function, `aiChooseSquare(color, drawTile, randFn)`, governs every
AI-controlled move in the game -- there is no separate logic for "the
Today's-game opponent" versus "an AI used in a background simulation" versus
a hint-suggestion (Β§7.2) versus any hypothetical future AI seat; only the
`randFn` argument passed in differs by context (defaulting to the global
`rng()` if omitted).

**Opponent selection in multiplayer games.** Before anything else, the AI
needs a single "opponent" to compare itself against -- straightforward in a
2-player game, but 3/4-player games have more than one other player at the
table. `pickLeadingOpponent(color)` resolves this by finding whichever other
player is genuinely leading, using a three-level tie-break: highest revenue
so far; if tied, most houses currently owned (a distinct signal from
revenue, since it counts regardless of a house's current connectivity);
if still tied, whoever plays next soonest after the requesting player, in
upcoming turn order. In a 2-player game this trivially returns the only
other player -- there is nothing to compare, so this function is a strict,
backward-compatible superset of always picking a single fixed opponent, not
a special case bolted on separately for 3/4-player games. This same
resolved opponent is then used throughout the rest of the evaluation below.

**Visibility: the candidate subset.** The AI does not evaluate the entire
board every turn. The subset is chosen **first, from the whole board**, and
only afterward are illegal squares removed from within it:

1. Every one of the N x N board positions (Parks and the greyed-out square
   included) is Fisher-Yates shuffled using `randFn`.
2. The first `AI_SUBSET_SIZE_CONFIG[N]` squares of that shuffled list are
   taken (18/25/32 for 6x6/7x7/8x8 respectively -- the same per-board-size
   table as before, chosen so each board size gets roughly the same
   *fraction* of its own squares visible, rather than one fixed count that
   would under-represent a larger board).
3. **Only then** are Parks and the immediately-preceding move's square
   filtered out of that selection -- whatever remains is what actually gets
   evaluated.

This ordering is deliberate, and was a genuine, subtle bug in an earlier
version of this same mechanism: the original implementation filtered Parks
and the greyed-out square out *before* choosing the subset, which meant the
AI was actually seeing 18 of 29 legal squares on a 6x6 board (roughly 62%)
rather than the intended "about half the board." Selecting the subset first
means the AI genuinely sees close to `AI_SUBSET_SIZE_CONFIG[N] / (N*N)` of
the whole board (verified directly: mean surviving candidates of 14.94/36,
20.95/49, 26.86/64 across the three board sizes, matching the theoretical
expectation from each board's own Park count almost exactly) -- and, as a
direct consequence, the number of squares actually evaluated on any given
turn now legitimately varies (fewer if an unlucky draw happens to include
more Parks than usual), rather than always being a fixed count. This can
never reach zero: the subset size comfortably exceeds the total number of
excluded squares on every board size, so there are always real candidates
left to choose from.

**Evaluating each candidate.** For each surviving candidate space, the AI
hypothetically places the current card there, recomputes connectivity, and
computes:

- `diff` = (sum of the point values, per Β§9's same distance formula, of every
  doubly-connected house this player would own) minus (the same sum for the
  resolved opponent, from `pickLeadingOpponent` above), counting a
  newly-completed, currently-unowned house as belonging to this player.
- `combinedBuildStealValue` = the sum of the point values of every house
  this exact placement would newly build, plus the point value of a house
  it would steal (never more than one steal per move, since stealing only
  ever affects the exact space played on, per Β§10) -- both folded into one
  number rather than tracked separately.
- `myTerr` = total count of spaces connected to *either* colour (not
  necessarily both) -- a rough measure of board presence/territory,
  unchanged from earlier versions of this heuristic.

**The comparison is lexicographic, not additive.** Each candidate reduces to
a tuple `[diff, combinedBuildStealValue, myTerr]`, and candidates are
compared strictly in that order: `diff` is compared first and dominates
completely -- any candidate with a higher `diff` always wins, regardless of
the other two values. Only when `diff` is *exactly* tied does
`combinedBuildStealValue` get consulted to break the tie; only when both of
those are tied does `myTerr` get consulted. Ties that survive all three
still go to whichever candidate was scanned first in the shuffled order --
there is no random tie-break beyond that. The board is restored to its
actual state after each hypothetical placement is scored, so this
evaluation has no side-effects until the real move is finally committed.

This replaced an earlier, additive formula (`val = diff + AI_ALPHA * decay *
completesHouse + AI_ALPHA * decay * isSteal + AI_BETA * myTerr`, with fixed
weights `AI_ALPHA = 0.7`, `AI_BETA = 0.1`, and a round-based decay term)
after testing found the additive version's exact weighting was fragile --
performance swung depending on the chosen weights in ways that didn't
reliably improve on the original, and pushing the build/steal weight higher
made the AI progressively more myopic, since a single high-value
house-completion could swamp `diff` entirely at high enough weights. The
lexicographic version has no weights or decay term to calibrate at all:
multiplying a strict tie-breaking criterion by any constant can never change
which candidate wins the comparison, so none are needed. `diff` always
dominates by construction, which was found, across repeated head-to-head
testing against the additive version (properly paired -- identical starting
board, deck, and Roadworks sequence for both sides of every swapped-seat
pair), to perform statistically indistinguishably overall, while being
structurally simpler and free of any weight to accidentally get wrong.


## 17. The 99-simulation rank-comparison system

Today's-game mode additionally computes, once per calendar date, a fixed
reference distribution of 99 fully-simulated AI-vs-AI game outcomes, so that
a real player's own score can be expressed as a percentile-style rank rather
than a bare number.

Both `applyHouseAndRevenue()` (Β§9's scoring) and `aiChooseSquare()` (Β§16's
AI, including its points-based `diff`) are the exact same, single functions
used everywhere else in the game -- `runOneRankSimulation()` calls each by
name, not via some separately-maintained copy. This was verified directly,
not just assumed from the shared-architecture pattern: at the moment the
batch had just finished (before the player could click anything), the live
`applyHouseAndRevenue`/`aiChooseSquare` functions' own source was inspected
and confirmed to already contain the Β§9/Β§16 distance-based formula. So the
99-game reference batch, the pre-game target popup below, and the player's
own actual game have always used identical scoring and AI logic since these
were introduced -- there is no separate code path that could drift out of
sync.

**When this runs**: automatically at page load, before the player can click
anything (not on every launch of Today's game -- see below). The "Let's
start digging" button is disabled and reads "Loading..." until this
completes, then its label and enabled state flip. Switching modes away and
back to Today's game later, or reloading into Today's game via the in-game
button, does **not** re-run this batch; it only ever runs once per full page
load, since the result is seeded from the date and would only ever come out
identical anyway.

**Two independent RNG roles**, deliberately kept as two separate `mulberry32`
instances (`runTodaysRankSimulations()` and `runOneRankSimulation()`):

1. **Per-game board/deck/Roadworks/Player-2 stream**: a fresh `mulberry32`
   instance, re-seeded from the identical date-derived seed, created anew
   before *each* of the 99 games individually. This drives that game's setup
   deck, play deck, Roadworks targeting, and Player 2's own subset choice
   (25, the 7x7 lookup value from Β§16's table, since Today's game is always
   7x7), via the ordinary `rng()`/`boardRNG()` dispatchers -- meaning every
   one of the 99 games plays out on the literal, identical board a real
   human would face that day, against the identical-strength opponent.
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
steal!" popup (Β§10) -- since a background simulation the player never sees
should never surface UI. All of the live game's actual board/score/deck
state is saved before the batch begins and restored byte-for-byte afterward,
and a fresh, entirely unconsumed seeded stream is created for the player's
own upcoming game once the batch finishes -- the batch's own RNG consumption
never bleeds into what the player subsequently plays.

**Rank formula** (`computeTodaysRank()`): let `myDiff` be the player's own
final `revenue[1] - revenue[2]`. The rank is

```
rank = (number of the 99 simulated score-differences strictly greater than myDiff) + 1
```

so 1 is the best possible outcome (nothing in the reference set beat it) and
100 is the worst (all 99 beat it). Ties are deliberately resolved in the
player's favour: a simulated value *equal to* the player's own score does
**not** count against them (only strictly-greater values do), so a player
who ties with every single one of the 99 simulations receives rank 1, not
100.

**Pre-game target popup.** A feature layered on top of the same
`todaysScoreDiffs` array, entirely separate from the rank calculation above:
once per Today's-game launch, immediately after the board is rendered but
before the player's first turn actually begins (`showTodaysTargetPopup()`,
input blocked via `awaitingAdvance` for this entire window), a popup reads
"Target for today's game: Win by N point(s)", where N is the **median** of
the 99 simulated score-differences, floored at 1 (a median at or below zero
is never shown as a target, since "win by 0 or fewer points" isn't a
meaningful goal to hand the player). This popup auto-hides after
`LONG_DELAY_MS` (Β§19), after which the first turn begins as normal.


## 18. Share message

Built by `buildShareText()`, only ever shown after a completed Today's-game
session (never after a Random game). This entire template was substantially
reworked since it was first documented; nothing below should be assumed to
match any earlier description. Exact template (`{...}` marking the only
variable parts):

```
Swalpa Adjust Maadi!
{share result line}

{d Mon} rank: {rank text}{hints suffix}
Your turn! https://tinyurl.com/adjust-maadi
```

- `{share result line}` (`buildShareResultLine()`) is one of three fixed
  strings, each carrying its own trailing emoji, based on
  `revenue[1] - revenue[2]`: `"I won today's game by N point(s)! \u{1F60A}"`,
  `"I lost today's game by N point(s)! \u{1F61E}"`, or (a tie)
  `"Today's game was a draw! \u{1F610}"`. This is a deliberately separate
  function from the similar, in-app popup wording (`buildResultLine()`,
  which uses "You won/lost" rather than "I won/lost" and carries no emoji or
  "today's game" framing, since the player is already looking at the
  popup) -- not a shared function with a pronoun switch.
- The date shown omits the year, since it is irrelevant to the message's
  point.
- `{rank text}` (`formatRank()`) is simply `"#{rank} of 100"` -- e.g. `"#48
  of 100"`, or `"#1 of 100"` for the best possible result. (This replaced an
  earlier, ordinal-suffix-based wording along the lines of "48th highest of
  100"; the dedicated ordinal-suffix logic that wording needed no longer
  exists in the source at all.)
- `{hints suffix}` is appended only if `hintsUsedThisGame` (Β§7.2) is greater
  than zero for this game: `" (with N hint)"` or `" (with N hints)"` as
  appropriate; otherwise this suffix is omitted entirely, with no trace of
  it in the message.
- All emoji in this template are written as explicit Unicode codepoints in
  the source, not typed directly, specifically to avoid a transcription/
  encoding-corruption failure mode that was hit more than once during
  development.


## 19. Timing constants (gameplay-relevant, not purely cosmetic)

- `TOTAL_ROUNDS = 20` -- the fixed game length.
- `QUICK_DELAY_MS = 500` and `LONG_DELAY_MS = 1300` replace an earlier,
  single, unified pacing delay -- these now have distinct roles rather than
  one value doing double duty. `QUICK_DELAY_MS` paces the shorter,
  in-between steps of a turn's own resolution (e.g. the brief pause before
  the AI's chosen move is actually placed after being decided, so the human
  can see which square got highlighted before it commits, and the shorter
  gaps between the successive stages of a turn settling). `LONG_DELAY_MS` is
  used both as the default/actual auto-hide duration for every popup shown
  in the game (Roadworks excuse, steal, game-over, and the pre-game target
  popup, Β§17) and as the pause before a committed move's turn-advance
  actually happens. While any such popup is visible, player input is fully
  blocked (`popupBlocking`) -- gameplay does not silently continue
  underneath a visible popup.
- `NUM_RANK_SIMULATIONS = 99`.
- `AI_SUBSET_SIZE_CONFIG` -- see Β§16 for the full, per-board-size table (18 /
  25 / 32 for 6x6 / 7x7 / 8x8 respectively).


## 20. File and deployment structure

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
  players. Its own "Game modes" section states plainly that Today's game is
  "always one player against the computer," deliberately without
  committing to a specific board size in the player-facing text, so this
  file does not need editing again if the board size used for Today's game
  (Β§13) is ever revisited.

All of these live at the root of the same directory when deployed (currently
GitHub Pages, with the main game file named `index.html` there).
