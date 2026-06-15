# AI_USAGE.md — AI Tooling Log

## Tool used

Claude (Anthropic, Sonnet 4.6) via the Claude.ai chat interface, used as
the primary development collaborator for backend scaffolding, the CSV
import/anomaly-detection pipeline, balance calculation, and API design.

## Key prompts (paraphrased, in order)

1. "Build a shared expenses app per this assignment PDF -- act as PM and
   developer. Help me decide stack." -> led to Django+DRF / React / SQLite
   dev+Postgres prod / Render+Vercel.
2. "Here's expenses_export.csv -- find the anomalies and propose handling
   policies before we build." -> produced the initial 19-anomaly list and
   policy table (refined further below).
3. Asked Claude to lock in policy decisions for: USD conversion, settlement
   rows, negative amounts, name normalization, non-member participants
   (Kabir), and the ambiguous date -- each presented as multiple-choice
   options so I could pick the policy rather than have Claude assume one.
4. "Scaffold the Django project: models for User/Group/Membership/Expense/
   Split/Payment/Import, then build the CSV importer with anomaly
   detection and a review queue."
5. "Build the balance calculation -- group balances, who-pays-whom, and a
   per-user itemized breakdown."
6. "Build the DRF API layer: serializers, views, urls for auth, groups,
   expenses, payments, balances."

## Cases where the AI produced something wrong (caught and corrected)

### 1. "Dev" misclassified as a typo instead of a recurring guest

What happened: the first version of the importer treated any `paid_by`
value that didn't exactly match a group member as a `fuzzy_name_match`
requiring review. "Dev" (paid for 4 expenses across the March trip) got
flagged this way every time -- which would have meant manually approving
the same "who is Dev?" decision 4 times.

How I caught it: ran the importer against the real CSV and printed the
flagged rows -- saw "Dev" repeatedly flagged as `fuzzy_name_match`
alongside genuinely ambiguous things like "Priya S".

What changed: added `find_close_match` (string similarity via `difflib`)
-- names with high similarity to an existing member (like "Priya S" vs
"Priya", ratio about 0.83) get flagged as a likely typo; names with low
similarity to every member (like "Dev") are instead auto-added as a
one-off guest user. This is `FUZZY_MATCH_THRESHOLD = 0.6` in
`imports/services/parsing.py`.

### 2. Guest participants (Dev, Kabir) weren't recognized inside split_details

What happened: for the "Scooter rentals" row, `split_with` includes "Dev"
(correctly converted to a guest participant), but `split_details`
("Aisha 1; Rohan 2; Priya 1; Dev 2") still failed to resolve "Dev" because
the share-calculation step only had access to the original group-member
lookup table, not the guest users created earlier in the same row's
processing -- so it was flagged as `unrecognized_name_in_split_details`
even though "Dev" had just been resolved two steps earlier in the same
function.

How I caught it: same test run -- saw the anomaly fire on a row where
every name in split_details should have been resolvable.

What changed: built a per-row lookup dict (`row_lookup`) that merges the
group-member lookup with every participant/payer resolved earlier in that
row (including newly-created guests), and passed that into
`compute_shares`/`calculate_splits` instead of the static group-wide
lookup.

### 3. The ambiguous date (04-05-2026) silently passed through with zero anomaly

What happened: "04-05-2026" parses as a perfectly valid date under the
sheet's DD-MM-YYYY convention (4th May 2026), so the date parser produced
no anomaly at all -- even though the assignment explicitly calls this row
out ("is this April 5 or May 4?") and requires every detected problem to
be surfaced, even ones that are auto-resolved.

How I caught it: cross-checked the 19-anomaly-type tally against a manual
review of the CSV and noticed this specific row wasn't producing any
anomaly in the importer's output.

What changed: added `_detect_ambiguous_dates`, which checks whether a
row's date is chronologically later than the very next row's date (a sign
the DD/MM fields might be swapped) and, if the day/month could plausibly
be transposed, adds an `ambiguous_date_order` anomaly -- auto-resolved per
the documented DD-MM-YYYY policy, but now visible in the Import Report as
required.

## Where I directed rather than accepted AI defaults

- Required every blocking-anomaly type to be enumerated explicitly
  (`BLOCKING_ANOMALY_TYPES`) rather than inferred ad hoc, so the
  review-vs-auto-import boundary is a single readable set, not scattered
  logic -- this is the line I'd be asked to walk through live.
- Required the split-calculation math to live in one shared module
  (`expenses/services/split_calculator.py`) used by both the CSV importer
  and API-driven expense creation, rather than two parallel implementations
  that could drift apart.
- Required a running balance-sum check (sum of all net balances equals 0)
  as a sanity test before moving on -- caught nothing on this run, but is
  the kind of check worth keeping in CI.

## Status note

At time of writing, the backend (models, import pipeline, balance logic,
DRF API for auth/groups/expenses/payments/balances) is complete and tested
against the real CSV. The Review Queue approve/reject endpoints, Import
Report endpoint, React frontend, and deployment are in progress -- this
file will be updated with further AI-usage notes as those are built.
