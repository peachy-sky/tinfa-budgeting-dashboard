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

**Grid**: `l15State.monthlyIncome` (flat $4,000 for now) determines grid
size — `l15BigSquareCount() = monthlyIncome / 1000` "big squares" of
$1,000 each, arranged via `l15BigGridDims()` (solved today only for the
4-big-square case → 2×2; anything else falls back to a single row and
logs a warning), each subdivided into a fixed 2×2 of $250 "small squares".
So $4,000/mo → 4 big squares → 16 total cells. A non-multiple-of-$1000
income also just logs a console warning (assumed not to happen yet).
Occupancy (`l15BuildOccupancy`) is tracked as one flat small-cell grid,
not scoped per big square — an item's footprint can straddle a
big-square boundary.

**Items** (`L15_SHELF_ITEMS`, single-instance — placing one dims it on the
shelf until dragged back out):

| Item | Price | Cells | Sprite |
|---|---|---|---|
| Dining Out | $250 | 1 (1×1) | bread1.png |
| Groceries | $1,000 | 4 (2×2) | bread4.png |
| Transit | $250 | 1 (1×1) | bread1.png |
| Phone Bill | $250 | 1 (1×1) | bread1.png |
| Healthcare | $250 | 1 (1×1) | bread1.png |
| Shopping | $250 | 1 (1×1) | bread1.png |
| Travel | $500 | 2 (2×1) | bread2.png |

`cells = Math.ceil(price / 250)`; shape comes from a fixed lookup
(`L15_ITEM_SHAPES`) keyed by cell count — a future price needing 3 cells
(e.g. $750) has no shape defined yet. Each item also carries a `category`
field, currently always `null` — reserved for a future Needs/Wants
(50/30/20) breakdown pass; no UI reads it yet.

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
`trash-closed.png` → `trash-open.png` on drag-over) removes the item;
dropping on an invalid tray location snaps it back (no-op) rather than
allowing overflow — the grid can never exceed 16/16 cells since it
represents 100% of income.

**Tooltip**: hover or click a shelf or placed item to show its name and
price in a small floating bubble (`#l15-tooltip`) near the pointer.

**Sidebar**: Monthly/Annual Cost = sum of placed item prices (× 12);
Monthly/Annual Savings = `(totalCells − occupiedCells) × 250` (× 12) —
recomputed on every `renderLevel1_5()` call, i.e. after every successful
place/remove/move.

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
