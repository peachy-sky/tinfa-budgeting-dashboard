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
  amount (from the hard-coded rate) and post-tax income recompute live as
  it changes, cascading into the needs/wants/savings percentages, the pie
  chart, and the over-budget alert.

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

#### Hard-coded special behavior: Groceries ↔ Dining Out
These two move **inversely**: raising one's cost tier by one step
automatically lowers the other's cost tier by one step (clamped at each
one's own bounds), representing "cook more at home → eat out less" and vice
versa. Energy spent on either is unaffected by the link, but is re-clamped
to stay valid if a tier change reduces that expense's max energy.

### 4. Totals
- `total_monthly_expense`, `total_annual_expense`, `total_annual_savings`
  (turns red if negative, i.e. overspending).

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
