# Income + Budgeting Dashboard — Spec

A single-page, static HTML dashboard (`index.html`) simulating a player's
income and monthly budget. Hostable on any static site host with no build
step.

## Core variables

| Variable | Start value | Notes |
|---|---|---|
| `level` | 1 | which expense set is loaded; see **Levels** below |
| `annual_pretax_income` | 15000 | player-editable via a number input; same default on every level |
| `income_tax` (rate) | 0% | hard-coded, drives `tax_amount` |
| `annual_posttax_income` | 15000 | = pretax − tax |
| `total_energy` | 7 | total energy pool |
| `available_energy` | 4 | = 7 − 3 (work, fixed) − energy spent on expenses |
| `annual_needs` | derived | sum of NEED expenses × 12 |
| `annual_wants` | derived | sum of WANT expenses × 12 |
| `annual_savings` | derived | posttax income − total annual expense |
| `total_monthly_expense` | derived | sum of all expense monthly costs |
| `total_annual_expense` | derived | `total_monthly_expense` × 12 |

Work (income) consumes a fixed 3 energy baked into the $15,000 figure and is
not player-adjustable in this MVP.

## Levels
A `<select id="level-select">` next to the "Budget Sheet" eyebrow (top-left
of the page) switches between **Level 0** and **Level 1** — two entirely
separate expense sets sharing one system:

- `EXPENSE_TEMPLATES` (`{ 1: EXPENSE_TEMPLATES_L1, 0: EXPENSE_TEMPLATES_L0 }`)
  and `PENDING_TEMPLATES` (`{ 1: [...4 keys...], 0: [] }`) are the
  read-only, per-level *source data* — never mutated directly.
- `expenses` and `pendingExpenses` are `let`-declared live copies:
  `loadLevel(levelNum)` deep-clones (`structuredClone`) that level's
  templates into them. Every other function in the file (render,
  changeLevel, applyEnergyDrop, the reset button, etc.) reads these two
  by name, so reassigning them on a level switch is automatically picked
  up everywhere with no re-wiring.
- `DEFAULT_STATE` holds every state field's shared starting value
  (income, tax, energy, jar, hearts — the "structure of the systems" that
  doesn't change between levels, since Level 0's CSV has no income/tax
  rows of its own). `state = { ...DEFAULT_STATE, level: 1 }` at load, and
  `loadLevel()` does `Object.assign(state, DEFAULT_STATE); state.level =
  levelNum;` — i.e. switching levels is a **full reset**: year, jar
  savings, hearts, income modifier, everything goes back to its default,
  not just the expense list.
- `loadLevel()` also clears `expenseRowRefs` and `#expense-list`'s
  innerHTML before re-rendering, since the previous level's expense-row
  DOM nodes are keyed by expense keys that may not even exist in the new
  level (Level 0 has no `rent`/`groceries`/etc. at all).
- Each template expense carries a `resetTarget` (`'min'` or `'max'`) that
  the "Reset to Lowest Costs" button reads generically — see below —
  instead of a hard-coded list of expense keys, so the button works
  correctly regardless of which level is loaded.
- `sliderWidthPx`'s reference max (previously a load-time constant) is
  now computed fresh from the *current* `expenses` array on every call,
  so Level 0's much smaller dollar amounts (max $30) scale their own
  sliders correctly instead of inheriting Level 1's $2,500 reference.

`LEVEL_STATE_OVERRIDES` layers level-specific values on top of
`DEFAULT_STATE` inside `loadLevel()` (`Object.assign(state,
DEFAULT_STATE, LEVEL_STATE_OVERRIDES[levelNum])`) for fields a level's
own spreadsheet redefines. Level 1's override is `{}` (uses the shared
defaults as-is); Level 0's is:

```js
{ annual_pretax_income: 600, total_energy: 3, work_energy: 2 }
```

— i.e. Level 0 starts at $600/yr pre-tax with only 1 of 3 total energy
available to spend (2 always go to work), shown as "1 / 3 available" on
the energy badge and 2 (not 3) work-diamonds next to Annual pre-tax income.

Level 0 is a much smaller-scale sheet — sourced from
`In-Game Budgeting L0.csv` — with only two WANT categories and no NEED
expenses, no hidden/yearly-reveal expenses (`PENDING_TEMPLATES[0]` is
empty), and no Groceries/Dining-style inverse link:

| Expense | Category | Default cost | Cost tiers | Max energy |
|---|---|---|---|---|
| Travel + Vacations + Experiences | WANT | $0 | 3 ($0 … −30) | 0 |
| Shopping | WANT | $0 | 2 ($0 … −20) | 1 |

## Level 1.5: spatial grid budgeting

A third `<option value="1.5">` on `#level-select` swaps to a completely
separate screen and a completely separate, self-contained system — it
shares no data with `EXPENSE_TEMPLATES`/`PENDING_TEMPLATES`/`state` above,
only the visual design tokens. Selecting it hides the masthead's
pie/needs-wants-savings row and the Income, Expenses, Interest Calculator,
Hearts, and Year sections; selecting Level 0 or 1 hides it and restores
those. `setLevel1_5Visible(active)` does the toggling; `loadLevel()`
branches on `levelNum === 1.5` before touching any Level 0/1 state.

Backpack-Hero-style spatial packing puzzle: expenses have a footprint in
grid cells and must be dragged onto a bounded tray instead of toggled in a
list. Savings is never placed directly — it's whatever grid space is left
unused.

**Grid**: three nested levels, big square ($1,000) → small square ($250)
→ tiny square ($62.50, `L15_CELL_VALUE`) — the tiny square is the actual
occupancy/snapping unit everywhere in code (`l15GridDims()`,
`l15BuildOccupancy`, drag candidates, etc. all work in tiny-square `col`/
`row`). `l15State.monthlyIncome` (flat $4,000 for now) determines grid
size — `l15BigSquareCount() = monthlyIncome / 1000` "big squares",
arranged via `l15BigGridDims()` (solved today only for the 4-big-square
case → 2×2 big squares; anything else falls back to a single row and
logs a warning). Each big square is *always* 2×2 small squares which are
*always* 2×2 tiny squares (that ratio doesn't depend on `l15BigGridDims`),
so `l15GridDims()` returns `{ cols: bigCols*4, rows: bigRows*4 }` — today
4 big squares → 8×8 = 64 total tiny cells, still worth $4,000 (64 × $62.50).
A non-multiple-of-$1000 income also just logs a console warning (assumed
not to happen yet). Occupancy (`l15BuildOccupancy`) is tracked as one flat
tiny-cell grid, not scoped per big/small square — an item's footprint can
straddle any boundary.

DOM mirrors this: `.l15-tray` > `.l15-big-square` (2×2 of) >
`.l15-small-square` (2×2 of) > `.l15-cell` (the tiny square,
`l15BuildGridDom`). Placed items are absolutely-positioned overlays (not
actual grid children, for simpler multi-cell sizing), so their pixel
position has to replicate the nested-grid math in JS:
`l15TinyOffset(i, cellPx)` decomposes a tiny-cell index into
big/small/tiny components and sums each level's own gap
(`L15_BIG_GAP`/`L15_SMALL_GAP`/`L15_TINY_GAP`, matching `.l15-tray`'s,
`.l15-big-square`'s, and `.l15-small-square`'s CSS `gap` respectively — no
level uses `padding`, so the math is pure nested-gap addition).
`l15PositionPlacedItem` and `l15PixelToCell` (the drag-candidate inverse
lookup) both go through this one function, so the two stay in sync.
`--l15-cell` is the tiny-square's pixel size (42px desktop / 32px mobile)
— since existing $250/$500/$1000 item footprints are exactly double their
old small-square-unit shapes while `--l15-cell` is exactly half its old
value, every existing item still renders at the identical pixel size it
did before tiny squares existed; the finer unit is purely additive,
opening the door to a future item priced in $62.50 increments that
doesn't fill a whole small square.

**Categories** (`L15_CATEGORIES`): each belt slot is a category (e.g.
"Groceries + Cooking @ Home") — `category.items` is the list of item
choices within it, transcribed directly from
`InGameBudgeting - Level1.5.csv` (one spreadsheet row per item; built via
the `l15Item(key, price, costs, hearts, note)` / `l15Category(key, name,
type, items)` helpers, not generated). Unlike the earlier flat $250/$500/
$1,000-per-category version, item counts and prices now vary per
category (2–5 items each, 30 total), shown side by side on the belt with
each item's own icon and price — the player drags whichever specific item
they want straight onto the grid. Single-instance is enforced per
*category*, not per item: placing any one item dims the whole group
(`l15IsCategoryPlaced`, applied to the `.l15-shelf-category` wrapper)
until it's dragged back out — you can't have two items of the same
category on the grid at once. `category.type` (`'NEED'`/`'WANT'`, from
the spreadsheet) is copied onto every item in it as `item.category`,
reserved for a future 50/30/20 breakdown pass; no UI reads it yet.

| Category | Type | Items |
|---|---|---|
| Rent + Utilities | NEED | 5 |
| Groceries + Cooking @ Home | NEED | 4 |
| Dining Out | WANT | 4 |
| Healthcare | NEED | 4 |
| Travel + Vacations + Experiences | WANT | 3 |
| Transit + Car Maintenance + Gas | NEED | 4 |
| Shopping | WANT | 4 |
| Trade School Education | NEED | 2 |

(The earlier "Pet" category from the flat-tier version isn't in the
spreadsheet and was dropped along with it.)

**Item shape/sprite** — no longer a fixed lookup, since spreadsheet prices
vary too widely and aren't always round (e.g. $190.50):
- `l15ItemCells(price) = Math.max(1, Math.round(price / 62.5))` — rounded
  rather than ceil'd, so a price a couple dollars off a clean multiple
  (e.g. $127 vs. a "true" $125) still lands on a clean cell count (2)
  instead of rounding up to 3; always at least 1 cell (covers the $0
  "on parents' healthcare" item).
- `l15ItemShape(item)` factors that cell count into the closest-to-square
  *exact* pair (`w * h === cells`, via a `while` loop shrinking `w` down
  from `floor(sqrt(cells))` until it divides evenly) — no wasted/
  overhanging grid space, since the grid's total capacity represents
  100% of income and a shape can't reserve more area than its price
  actually costs. A very expensive item (e.g. $2,500 rent → 40 cells →
  5×8) can legitimately dominate the grid; that's accurate to what it
  costs, not a bug.
- `l15ItemSprite(item)` — only items priced at exactly $62.50
  (bread1.png) or $127 (bread2.png) get a specific sprite; every other
  price renders as a plain `.l15-square-icon` div instead (per explicit
  instruction: "for any unspecified amounts, use a square shape").
  `l15IconHtml(item, styleAttr)` is the one place that branches on this —
  every icon-drawing call site (shelf, placed item, drag ghost) goes
  through it rather than checking `l15ItemSprite` itself.

`l15Money(n)` formats a dollar amount for display — spreadsheet prices
aren't always whole dollars, so this is used instead of a bare
`` `$${n.toLocaleString()}` `` (which would print "$190.5" instead of
"$190.50") everywhere a Level 1.5 dollar figure is shown.
`l15ItemByKey`/`l15CategoryForItemKey` resolve an item or its owning
category from a placed entry's `itemKey`.

**Drag-and-drop** (`startL15Drag`/`l15OnDragMove`/`l15OnDragUp`): the same
pointer-based pattern as the energy-diamond drag above (floating ghost,
`elementFromPoint`+`closest()` for drop-target resolution) adapted for a
multi-cell footprint — the ghost is sized to the item's own `w`×`h`, and
the drop location is resolved to a candidate top-left cell
(`l15CandidateFromPoint` → `l15PixelToCell`, which inverts the tray's
pixel-position formula by nearest-cell search since the big-square gap
makes it non-linear) rather than a single named target. A drop is valid
only if every covered cell is in-bounds and unoccupied
(`l15CanPlace`); cells preview green/red (`.drag-valid`/`.drag-invalid`)
during the drag. Dropping on the shelf or the trash icon (which swaps
`trash-closed.png` → `trash-open.png` on drag-over, and sits directly
below the tray — `.l15-tray-col` stacks them in one column — at
384×384px, well oversized relative to the tray, for an easy drop target)
removes the item;
dropping on an invalid tray location snaps it back and shows a red alert
(`l15ShowAlert`, `#l15-alert`) instead of allowing overflow — the grid can
never exceed 16/16 cells since it represents 100% of income. The alert
text depends on *why* the drop failed (`l15AnyValidPlacement` scans every
cell for whether the item fits anywhere else): "Only place expenses in a
free budget spot." if it collided with an occupied cell but room exists
elsewhere, or "There is not enough space in the budget for that!" if the
item can't fit anywhere at all given current occupancy.

Every dynamic (JS-built) asset path routes through one `l15AssetUri(file)`
helper rather than being constructed inline — the Artifact publish build
(`build_artifact.js`, scratchpad) swaps that single function's body for a
lookup into an embedded base64 map, since a published Artifact can't
fetch relative files. Route any new dynamic image reference through this
helper, not a hand-built path string, or the published copy will 404 for
just that one call site while everything else keeps working (this exact
bug shipped once for the trash icon, which broke the instant any drag
started because `l15SetTrashOpen` built its path with a ternary the old
per-call-site regex didn't catch).

**Tooltip**: hover or click a shelf or placed item to show its category
name, price, and (if present) the spreadsheet's note text in a small
floating bubble (`#l15-tooltip`) near the pointer.

**Belt scrolling**: the shelf is a single non-wrapping row
(`.l15-shelf { flex-wrap: nowrap; overflow-x: hidden }`) inside
`.l15-shelf-wrap`, with a left/right arrow button overlaid on each edge.
Hovering an arrow starts a `requestAnimationFrame` loop
(`l15StartShelfScroll`/`l15ScrollStep`) nudging `scrollLeft` every frame
until the pointer leaves; arrows hide via `l15UpdateShelfArrows` once
scrolled fully to that edge.

**Energy**: a separate resource from Level 0/1's — `L15_TOTAL_ENERGY = 4`,
all spendable (Level 1.5 has no "work" concept eating into it). Shown as
a diamond pool (`#l15-energy-pool`, reusing the `.energy-pool-cell`/
`.diamonds`/`.diamond` styling and the shared `renderDiamonds()` helper)
next to "Monthly Income" at the top of the screen. Each item's `costs`
array is indexed by energy spent (`costs[0]` = base price = 0 energy;
`costs[1]`, `costs[2]`, ... are progressively cheaper) — a shorter array
(Rent, Transit, Trade School Education items all have length 1) means
that item doesn't accept energy at all (`l15MaxEnergyForItem(item) =
item.costs.length - 1` is 0). Applying energy doesn't touch the item's
footprint or `price` (which still drives its grid shape/sprite/badge) —
only `l15CurrentCost(p)`, which is what actually counts toward Monthly
Cost.

Per the explicit request, energy is applied by dragging a diamond
*directly onto a placed item* (not a picker or a slider) — the same
pointer-based drag pattern as Level 0/1's energy-diamond system and this
level's own item drag (own module though, not a forced shared
abstraction: `l15StartEnergyDrag`/`l15OnEnergyDragMove`/
`l15OnEnergyDragUp`/`l15ApplyEnergyDrop`/`l15UpdateEnergyDropHighlight`),
reusing `renderDiamonds()` and `findGrabbableDiamond()` directly since
those two are already fully generic. Every placed item with
`maxEnergy > 0` shows its own small diamond row stuck to its bottom edge
(`.l15-item-diamonds`, filled up to `p.energy` — rebuilt in place each
render rather than the whole item, since only the diamonds actually
change); items with `maxEnergy === 0` show no diamonds at all. Drop
targets are marked `.l15-energy-drop-target` + `data-l15-energy-target`
on the pool and on every placed `.l15-item` (the *whole* item card, not
just its diamond row, so dropping anywhere on it counts — grabbing an
*existing* diamond back off an item is the tighter target, resolved via
`findGrabbableDiamond` inside the single pointerdown handler on `.l15-item`,
which branches between starting an energy-drag (near a filled diamond) or
the normal item-move drag (anywhere else) rather than using two competing
listeners). Pool ↔ item and item ↔ item moves both work, mirroring Level
0/1's `applyEnergyDrop` semantics exactly, just against `l15State.placed`
entries instead of `expenses`.

**Savings Calculator** (`.l15-sidebar`, titled via `.l15-hearts-title`):
Monthly Cost = sum of each placed item's *current* cost
(`l15CurrentCost(p)` — the energy-discounted figure, not the base
`price`; see Energy below) × 12 for annual. Monthly/Annual Savings =
`l15MonthlySavings()` (`(totalCells − occupiedCells) × 62.5`, × 12 for
annual) deliberately stays purely grid-space-based — the puzzle's core
mechanic since Level 1.5 shipped — and does *not* factor in energy
discounts, since energy reduces what an item costs without shrinking its
footprint. This means Cost + Savings can diverge from a strict
income-equals-cost-plus-savings identity once energy is in play; that gap
is real "found" savings from playing efficiently, intentionally not
reconciled away. Recomputed on every `renderLevel1_5()` call.

**Hearts**: each item carries its own `hearts` value straight from the
spreadsheet's HEARTS column (fractions like "1/4+"/"1/2-" pre-converted
to signed decimals when transcribed — e.g. `dining-2000` is `hearts: 1`,
`transit-127` is `hearts: -0.5`; most items are 0). A positive-hearts item
shows a ❤️ badge (`.l15-heart-badge`, absolutely positioned over the icon)
on the shelf, the drag ghost, and once placed — negative/zero-hearts
items show no badge but still count toward the total. A second card
(`.l15-hearts-box`, same `.l15-sidebar` styling) sits directly under the
cost/savings card on the right, mirroring Level 1's Hearts section:
`l15State.heartsLastYear` (starts at 5, matching Level 1's
`DEFAULT_STATE.hearts_last_year`) + `l15HeartsThisYear()` (sum of each
placed item's own `hearts` value) + `l15State.heartsModifier` (free-entry number
input, `#l15-hearts-modifier-input`) = `l15CurrentHeartsTotal()`. This is
a separate, Level-1.5-only heart total — it does not read or write
`state.hearts_last_year`/`heartsThisYear()` from Level 0/1, and resets to
5/0 in `l15ResetState()` on every level switch, same as the rest of
`l15State`.

**Interest Calculator**: a second card (`.l15-interest-box`) sits between
the Savings Calculator and Hearts cards. Same structure as Level 1's
Interest Calculator but relabeled and wired to the Savings Calculator
above it rather than a Next Year rollover (there is none — the Year
section stays hidden for Level 1.5):
- **Savings Last Year** (`l15State.savingsLastYear`, read-only display,
  starts at 0) — the Level-1.5 analog of Level 1's "Current Jar Savings".
- **Savings This Year** (`l15State.savingsThisYear`, number input) — the
  analog of "New Jar Savings", but with a twist: it *defaults* to the
  Savings Calculator's live Annual Savings rather than starting at 0.
  `l15State.savingsThisYear` starts `null`, meaning "not manually set";
  `l15SavingsThisYear()` returns `l15AnnualSavings()` while it's `null`,
  so the input visibly tracks Annual Savings as items are placed/removed.
  The instant the player types into the field (even to re-enter the same
  number), the `input` listener sets a concrete number and it stops
  tracking — from then on it holds exactly what the player typed,
  regardless of further grid changes, until reset.
- **Interest Rate** (`l15State.interestRate`, %, clamped ≥ 0 same as
  Level 1).
- **This year's interest** = `l15EarnedInterest()` =
  `(savingsLastYear + l15SavingsThisYear()) * (interestRate / 100)`.

All three (`savingsLastYear`, `savingsThisYear` back to `null`,
`interestRate`) reset in `l15ResetState()` on every level switch.

Art assets (`assets/budgeting-tab/`, copied from
`tinfa-card-godot-mvp/assets/art/budgeting-tab/`): `belt.png` (shelf
background), `tray.png` (grid background), `bread1/2/4.png` (item
sprites), `trash-closed.png`/`trash-open.png` (trash icon).

## Sections

### 1. Needs/Wants/Savings summary + pie chart
- Shows `needs%`, `wants%`, `savings%` of `annual_posttax_income`.
- Pie chart (conic-gradient, blue/purple/green) reflects those percentages.
- If `abs(total_annual_expense) > annual_posttax_income`, the pie is replaced
  with a red circle + "!" instead of rendering an invalid/overflowing chart.

### 2. Income
- Annual pre-tax income is a player-editable number input (min 0); tax
  amount (from the hard-coded rate) recomputes live as it changes.
- `income_modifier`: player-editable number input, starts at 0. Negative
  values are allowed and subtract from post-tax income.
- Annual post-tax income = `annual_pretax_income - tax_amount +
  income_modifier`, live-computed, cascading into the needs/wants/savings
  percentages, the pie chart, and the over-budget alert.
- `state.work_energy` static (non-draggable) filled ◆ diamonds sit next to
  the "Annual pre-tax income" label — `#work-diamonds`, rebuilt every
  render as `'<div class="diamond filled static"></div>'.repeat(work_energy)`
  — representing the energy always spent earning that income (3 on Level 1,
  2 on Level 0). Purely decorative: no drag listener, and never draws from
  or affects the draggable energy pool.

### 3. Expenses (the only player-editable section)
Each expense has:
- A **cost tier** slider (`<input type="range">`, discrete steps, one per
  spreadsheet tier) — dragging or using arrow keys moves through the tier
  ladder; the native `min="0"` means the player can never go below the
  cheapest tier. Each expense's track width is scaled to its own priciest
  tier relative to the priciest tier across all expenses (currently Rent's
  -$2,500 — this is derived from the data, not hard-coded, so it
  automatically rescales if the spreadsheet's priciest tier changes; 420px
  at the reference max, down to a 48px floor for a $0-range expense like
  Pet) — so e.g. Dining Out's bar reads visibly longer than Transit's.
  The track's actual rendered width is `min(that scaled px value, 100%)`,
  so it never overflows a narrow (mobile) viewport.
  Dragging is driven by custom `pointerdown`/`pointermove`/`pointerup`
  logic (not the browser's native track-following), because native range
  inputs stop tracking the instant the cursor leaves their bounds — the
  custom handler computes the tier from the pointer's X position relative
  to the track for as long as the button/touch is held, no matter how far
  outside the track the cursor strays, clamped to that track's own
  min/max either way. Keyboard interaction (arrow keys, etc.) still goes
  through the native default action.
- An **energy** ladder layered on the current cost tier: spending energy
  (◆ diamonds) further reduces that tier's cost, up to a per-expense max
  (0–2), bounded by `available_energy`. Energy is spent via drag-and-drop
  (see below), not a stepper. When energy actually lowers the
  price (i.e. the current tier's energy-0 cost differs from its cost at
  the current energy spend), the pre-energy price is shown struck through
  next to the new price, which turns green (`--accent`). Spending energy
  on a tier where all energy levels cost the same (e.g. Groceries' cheapest
  tier) shows no strikethrough, since there's no real discount to call out.
- A **notes** field surfaced as-is from the spreadsheet (flavor text and any
  "modifier" callouts — modifiers are noted only, not mechanically applied),
  shown as a sub-line directly under the expense name (there is no separate
  Notes column).

#### Energy spend: drag-and-drop
The "X / Y available" counter and a shared pool of draggable ◆ diamonds
(one per point of `total_energy - work_energy`, i.e. 4) sit directly above
the Energy column header — moved there from the header area at the top of
the page. It's styled as a filled gold badge (larger bold text, larger
diamonds) so it reads clearly rather than blending into the ledger.
There is no ⊖/⊕ stepper anywhere; energy is spent purely by dragging:
- **Pool → expense**: drag a filled pool diamond onto any expense's
  energy area to spend a point there (no-ops if that expense is already
  at its own max energy, or the pool is empty).
- **Expense → pool**: drag a filled diamond off an expense and drop it
  back on the pool to refund that point.
- **Expense → expense**: drag a filled diamond from one expense directly
  onto another to move the point in one gesture (no-ops if the target is
  already at its own max).
- Dropping anywhere that isn't a valid target (off-screen, on itself,
  etc.) cancels the drag with no change.
- The **drop target** for each expense is its entire Energy column cell
  (full column width × full row height, stretched via `align-self:
  stretch` + padding), not just the tight cluster of diamond icons —
  roughly a 30x larger hit area, so an imprecise drop anywhere in that
  row's energy area still lands correctly.
- The **grab** (pick-up) hit-area is also enlarged, separately from the
  drop area: `findGrabbableDiamond` treats a click within a radius equal
  to a diamond's own width (i.e. a grabbable field with double the
  diamond's diameter — 100% bigger) as grabbing that diamond, picking the
  nearest one if two diamonds' enlarged fields overlap. The pointerdown
  listener is on the whole row (expense rows) / whole pool row, not just
  the tight diamond cluster, so the enlarged radius has physical room to
  register clicks that land in an adjacent cell.

Implemented as custom `pointerdown`/`pointermove`/`pointerup` handling
with a floating drag-ghost diamond and `document.elementFromPoint` for
drop-target detection (`optimalEnergyAllocation` / the "Reset to Lowest
Costs" button still set `.energy` directly and are unaffected by this).

Row DOM nodes are built once per expense and updated in place on every
render rather than destroyed/recreated, so dragging a slider isn't
interrupted mid-gesture.

#### Locking: one year of editability, then frozen forever
Each expense tracks `revealedYear` (the year it first became visible —
1 for Groceries/Dining/Healthcare/Travel/Pet, since they're visible from
the start; whatever `state.year` was when a pending expense got revealed,
for Rent/Transit/Shopping/Education) and a `locked` flag (starts `false`).

On every Next Year click, **before** `year` increments, any expense with
`!locked && revealedYear === state.year` gets `locked = true` — i.e. an
expense is editable for exactly the one year it's shown in, then freezes
for the rest of the game at whatever cost tier/energy it had. This means
all five Year-1 expenses lock together on the very first Next Year click;
each later-revealed expense then gets its own single year before locking
in turn.

Locking only freezes the **cost tier** — energy stays fully movable on a
locked expense, in both directions, for the rest of the game. A locked
expense (`.expense-row.locked`):
- Shows a 🔒 lock icon in place of the cost slider (`display:none` on the
  `<input type="range">`, matching `title`/`aria-label` on the icon) — the
  frozen dollar value and hearts still display normally.
- Keeps the `energy-drop-target` class and remains a normal drag source/
  target: energy can still be dragged onto it from the pool or another
  expense, and dragged off it to the pool or another expense, exactly as
  if it weren't locked. Only `changeLevel` (the cost tier) checks
  `exp.locked` — the energy drag/drop path (`applyEnergyDrop`, the row's
  grab listener) does not.
- Is skipped by "Reset to Lowest Costs" *only* for its tier jump; the
  energy-reallocation half of that button still includes every visible
  expense (locked or not) against the full budget, same as before locking
  existed.
- Gets a subtly shaded row background (`--surface-2`) as a visual cue,
  independent of the lock icon.

Sourced from `In-Game Budgeting - Level 1.csv`:

| Expense | Category | Default cost | Cost tiers | Max energy |
|---|---|---|---|---|
| Rent + Utilities | NEED | -$200 | 5 (−200 … −2500) | 0 |
| Groceries + Cooking @ Home | NEED | -$400 | 4 (−50 … −400) | 2 |
| Dining Out | WANT | -$120 | 4 (−120 … −2000) | 1 (no discount on top tier) |
| Healthcare | NEED | $0 | 4 ($0 … −300) | 1 |
| Travel + Vacations + Experiences | WANT | -$300 | 3 (−100 … −1000) | 1 |
| Transit + Car + Gas | NEED | -$300 | 4 (−100 … −700) | 0 |
| Shopping | WANT | -$700 | 4 (−200 … −1000) | 1 |
| Education (Trade School) | NEED | -$400 | 2 (−250 … −400) | 1 |
| Pet | WANT | $0 | 1 (modifiers noted only) | 0 |

#### Yearly reveal: Rent, Transit, Shopping, Education
These four start **hidden** (not shown in the Expenses list, and excluded
from every total/percentage/energy calc) at Year 1. Each Next Year click
un-hides the next one, in this fixed order — Rent + Utilities, then
Transit + Car + Gas, then Shopping, then Education (Trade School) — until
all four are visible; further Next Year clicks then do nothing to the
expense list. A revealed expense keeps its normal default cost tier/energy
from that point on.

#### Hard-coded special behavior: Groceries ↔ Dining Out
These two move **inversely**: raising one's cost tier by N steps
automatically lowers the other's cost tier by N steps (clamped at each
one's own bounds — dragging a slider can jump multiple tiers in one
gesture, not just ±1), representing "cook more at home → eat out less" and
vice versa. Energy spent on either is unaffected by the link, but is
re-clamped to stay valid if a tier change reduces that expense's max
energy.

#### "Reset to Lowest Costs" button
Bottom-left of the Expenses block. Two steps, both bypassing the normal
slider/inverse-link interaction entirely, and both data-driven off the
current level's `expenses` rather than any hard-coded key list:
1. Every unlocked expense jumps to its own `resetTarget` tier — `'max'`
   (priciest) for Level 1's Groceries + Cooking @ Home, its half of the
   cook-more/eat-out-less pair; `'min'` (cheapest) for every other
   expense on every level.
2. With those tiers now fixed, spends the player's full available energy
   budget (`total_energy - work_energy`) across the visible expenses
   however saves the most total money — solved as a multiple-choice
   knapsack (`optimalEnergyAllocation`, a small memoized recursion) rather
   than greedily per-expense, so it correctly finds the globally best
   split even when marginal savings differ across expenses (e.g. it may
   spend 2 energy on one expense and skip another entirely if that beats
   spreading 1 energy across more expenses).

### 4. Totals
- `total_monthly_expense`, `total_annual_expense`, `total_annual_savings`
  (turns red if negative, i.e. overspending).

### 5. Interest Calculator
- `current_jar_savings` (starts 0, display-only — only the Next Year button
  changes it).
- `new_jar_savings`: player-editable number input (this year's contribution
  or withdrawal — negative values are allowed and reduce the effective jar
  total for this year's interest calc and the Next Year rollover).
- `jar_interest_rate`: player-editable number input; entering `3` means 3%
  (0.03) internally.
- `earned_interest` (this year's interest, live-computed, not stored):
  `(current_jar_savings + new_jar_savings) * jar_interest_rate`.

### 6. Hearts
Each expense's cost tier optionally carries a `hearts` value (fractional,
e.g. ±0.25/±0.5/±1) sourced from the spreadsheet's HEARTS column — most
tiers are 0. Only groceries, dining, healthcare, travel, transit, and rent
carry non-zero values on some tiers; shopping and education are 0
throughout in this revision. Healthcare additionally has a flat
`energyHeartsBonus` of +0.25 hearts applied whenever any energy is spent
on it (regardless of tier or how much energy, 1 or 2) — this is separate
from and additive with the tier's own `hearts` value. The current
(tier + energy bonus) total is shown per-row in a **Hearts** column in the
Expenses ledger (a "—" for 0, otherwise signed e.g. "+0.25"/"−0.5",
colored green/red).
- `hearts_last_year`: starts at 5, display-only to the player — updated only
  by Next Year (see below).
- `hearts_this_year` (live-computed): sum of `hearts` at the current tier
  of every **visible** expense (hidden/not-yet-revealed expenses are
  excluded, same as the cost totals).
- `hearts_modifier`: player-editable number input, starts at 0.
- `current_hearts_total` (live-computed):
  `hearts_last_year + hearts_this_year + hearts_modifier`.

### 7. Year / Next Year
- `year` counter, starts at 1, shown at the very bottom of the page next to
  a **Next Year** button.
- Clicking Next Year:
  - `current_jar_savings += new_jar_savings + earned_interest`
  - `hearts_last_year += hearts_this_year + hearts_modifier` (computed
    against this year's visible expenses and modifier, before either the
    modifier reset or that year's pending expense reveal below take
    effect).
  - `year += 1`
  - `new_jar_savings`, `income_modifier`, and `hearts_modifier` all reset
    to 0, with their inputs cleared to empty (ready for next year's
    values); `jar_interest_rate` is left as-is.
  - Does **not** touch `annual_pretax_income`, tax, or any expense tier/
    energy state — those are edited only through their own sections.

## Typography
Headings (`h1`, `h2`, and the over-budget "!" mark) use the hand-lettered
**Lazy Dog** typeface (sourced from
`tinfa-card-godot-mvp/assets/fonts/lazy_dog.ttf`, embedded as a base64
`@font-face` so the page stays a single self-contained file). Body copy and
all dollar/energy figures use IBM Plex Sans/Mono (via Google Fonts) for
legibility and tabular alignment.

## Status
Implemented in `index.html`. This spec documents the current build and is
the reference for any future changes.
