# Income + Budgeting Dashboard — Spec

A single-page, static HTML dashboard (`index.html`) simulating a player's
income and monthly budget. Hostable on any static site host with no build
step.

## Core variables

| Variable | Start value | Notes |
|---|---|---|
| `annual_pretax_income` | 15000 | player-editable via a number input |
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
- Three static (non-draggable) filled ◆ diamonds sit next to the "Annual
  pre-tax income" label, representing `work_energy` — the 3 energy always
  spent earning that income. Purely decorative: no drag listener, and
  never draws from or affects the draggable energy pool.

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
slider/inverse-link interaction entirely:
1. Jumps hard-coded expenses to hard-coded tiers — Groceries + Cooking @
   Home → its **priciest** tier (the only expense set to its max, not its
   min); Dining Out, Healthcare, Travel + Vacations + Experiences, Pet, and
   any other currently-revealed expense (Rent, Transit, Shopping,
   Education) → each expense's **cheapest** tier.
2. With those tiers now fixed, spends the player's full available energy
   budget (`total_energy - work_energy`) across the revealed expenses
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
