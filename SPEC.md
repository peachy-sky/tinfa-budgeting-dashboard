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

### 3. Expenses (the only player-editable section)
Each expense has:
- A **cost tier** ladder, stepped with ⊖/⊕ (manual increase/decrease of
  monthly cost).
- An **energy** ladder layered on the current cost tier: spending energy
  (◆ diamonds, ⊖/⊕) further reduces that tier's cost, up to a per-expense max
  (0–2), bounded by `available_energy`.
- A **notes** field surfaced as-is from the spreadsheet (flavor text and any
  "modifier" callouts — modifiers are noted only, not mechanically applied).

Sourced from `In-Game Budgeting - Level 1.csv`:

| Expense | Category | Default cost | Cost tiers | Max energy |
|---|---|---|---|---|
| Rent + Utilities | NEED | -$200 | 8 (−200 … −4000) | 0 |
| Groceries + Cooking @ Home | NEED | -$50 | 4 (−50 … −400) | 2 |
| Dining Out | WANT | -$2,300 | 4 (−120 … −2300) | 1 |
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
These two move **inversely**: raising one's cost tier by one step
automatically lowers the other's cost tier by one step (clamped at each
one's own bounds), representing "cook more at home → eat out less" and vice
versa. Energy spent on either is unaffected by the link, but is re-clamped
to stay valid if a tier change reduces that expense's max energy.

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
throughout in this revision. The current tier's value is shown per-row in
a **Hearts** column in the Expenses ledger (a "—" for 0, otherwise signed
e.g. "+0.25"/"−0.5", colored green/red).
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
    against this year's visible expenses, before that year's pending
    expense reveal below takes effect); `hearts_modifier` is left as-is.
  - `year += 1`
  - `new_jar_savings` resets to 0 and its input is cleared to empty (ready
    for next year's contribution); `jar_interest_rate` is left as-is.
  - Does **not** touch income, tax, or any expense tier/energy state —
    expenses and income are edited only through their own sections.

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
