# DECISIONS.md — Decision Log

## 1. Tech stack: Django + DRF, React, SQLite/Postgres

**Options considered**: Node/Express + React, Flask/FastAPI + React, Django + DRF.

**Decision**: Django + DRF. Built-in admin panel, auth, and ORM cut
significant scaffolding time for a 2-day build, and Django's ORM makes the
relational schema (Group ↔ GroupMembership ↔ Expense ↔ ExpenseSplit ↔
Payment) straightforward to express and migrate.

## 2. Database: SQLite locally, Postgres in production

**Options considered**: SQLite everywhere (simplest, but Render's free-tier
filesystem is ephemeral — data would be wiped on redeploy); managed
Postgres (Neon/Supabase/Render Postgres).

**Decision**: SQLite for local dev (zero setup), Render's built-in free
Postgres for deployment, switched via a single `DATABASE_URL` env var
(`dj-database-url`). Keeps local dev frictionless while satisfying the
"relational DB only" requirement in production.

## 3. USD → INR conversion: fixed documented rate, store both amounts

**Options considered**: (a) fixed manual rate; (b) look up the historical
rate for each expense's date; (c) store original + converted using a
single rate.

**Decision**: (c). `USD_TO_INR_RATE = 83.00` is a constant in
`expenses/models.py`, applied to all USD rows in the CSV. Both `amount`
(original, USD) and `amount_inr` (converted) are stored on the `Expense`,
plus `exchange_rate_used` for traceability. This directly answers Priya's
complaint ("the sheet pretends a dollar is a rupee") without requiring a
live FX API call (avoids external dependency + non-determinism for a
2-day build). If a per-date historical rate is wanted later, it's a
one-line change to `import_csv` — the schema already supports a
per-expense rate.

## 4. Settlement-rows-as-expenses → Review Queue, not auto-converted

**Options considered**: (a) auto-convert to `Payment`; (b) flag for
manual review, admin confirms; (c) skip entirely.

**Decision**: (b). "Rohan paid Aisha back ₹5000" is unambiguously a
settlement (`split_type` blank, single recipient) — but Meera explicitly
asked to **approve anything the app deletes or changes**. Auto-converting
would be exactly that kind of silent change. The importer detects it,
proposes the conversion (Rohan to Aisha, ₹5000), and creates it only on
admin approval via the Review Queue.

## 5. Negative amounts (refunds): own expense, negative value

**Options considered**: (a) treat as a reversal of the original expense
(reduce its amount); (b) own expense with negative value, reducing each
participant's share; (c) flag and exclude from balance calc.

**Decision**: (b). "Parasailing refund -30 USD" is logged as its own
`Expense` row with `amount = -30`, `amount_inr = -2490`. Its
`ExpenseSplit` shares are negative too (each participant's share
*decreases*). This is simplest to implement, keeps the audit trail intact
(the refund is its own traceable row, not a silent edit to the original
Parasailing expense), and the balance math (sum paid minus sum owed)
handles negative values correctly without special-casing.

## 6. Name normalization: exact-after-trim/case auto-fixed; fuzzy goes to review

**Options considered**: (a) fuzzy-match everything automatically; (b) flag
all variants for review; (c) auto-fix only exact case/whitespace
mismatches, flag genuine fuzzy matches.

**Decision**: (c). "priya" -> Priya and "rohan " -> Rohan are unambiguous
(same string after trim+lowercase) — auto-fixed silently, logged.
"Priya S" is NOT an exact match (~0.83 similarity via `difflib`) — could be
a typo, a nickname, or a different person — flagged for review rather than
guessed.

## 7. Non-member participants ("Dev", "Kabir"): guest Users

**Options considered**: (a) exclude them, redistribute their share among
real members; (b) add as one-off guest participants who can hold a
balance but aren't full group members; (c) flag every such row for review.

**Decision**: (b). Dev appears across 5 rows as both payer and
participant — excluding him would misrepresent a large portion of the
March trip's costs. Guests get a `User` row with `is_guest=True` (so they
fit the existing FK structure for `paid_by`/`ExpenseSplit.user` without
schema changes) but no `GroupMembership` — they can't log in or be added
to future expenses through the UI's member picker.

## 8. Membership-window enforcement on split participants (Sam & Meera's asks)

**Decision**: At import time, every participant in `split_with` is checked
against `GroupMembership.is_active_on(expense_date)`. Rows where someone
is included outside their membership window (Meera in April groceries
after leaving in March; Sam in early-April expenses before joining
mid-April) are flagged — directly addressing both Sam's and Meera's
stated concerns. This check also runs whenever a new expense is created
via the API (not just CSV import).

**Known limitation**: the same check is not currently applied to
`paid_by` (e.g., Sam paying his deposit on 08-04, before his 15-04 join
date, is treated as a separate "transfer-like row" anomaly rather than a
membership-window violation, since paying a deposit ahead of moving in is
plausible). Documented rather than silently handled either way.

## 9. Percentage/unequal/share splits that don't balance -> review, not auto-rescale

**Decision**: A 110%-summing percentage split (Pizza Friday, Weekend
brunch) is not silently rescaled to 100% — that would change what each
person owes without anyone agreeing to it. Flagged for review; the admin
edits the percentages (or split type) before the expense is created.

## 10. Ambiguous date (04-05-2026) -> keep DD-MM-YYYY, but surface it

**Decision**: The sheet uses DD-MM-YYYY everywhere else (confirmed by
unambiguous dates like 08-03-2026 = 8th March, consistent with "trip
starts" context). 04-05-2026 parses validly as 4th May under that
convention. Rather than guess differently for one row, we keep the
sheet-wide convention — but detect that this row's date is later than
the next row's date (breaking chronological order) and surface that as an
"ambiguous, resolved as DD-MM" note in the Import Report, satisfying the
"surface it" requirement even for auto-resolved rows.

## 11. Exact & conflicting duplicates: flag the second occurrence

**Decision**: Both "Marina Bites" duplicates (same amount) and "Thalassa
dinner" (different amounts: 2400 vs 2450) are detected via same-date plus
description-token-overlap (>=50%). The first occurrence is left unflagged
(and auto-imports normally if otherwise clean); the second is flagged with
a description identifying which row it conflicts with, so the admin's
decision is "keep/discard row N" rather than "which of these two unlabeled
rows is real".

## 12. Rounding remainder on splits -> goes to the payer

**Decision**: When amount_inr / n (or percentage/share math) leaves a
paise-level remainder, it's added to the payer's share (or the first
participant if the payer isn't in the split). This guarantees that the
sum of ExpenseSplit.share_amount equals Expense.amount_inr exactly for
every expense — required for the group-wide balance sum to be exactly 0.
