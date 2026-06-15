# SCOPE.md — Anomaly Log & Database Schema

## Anomaly Log

`expenses_export.csv` has 42 data rows. The importer detects **19 distinct
anomaly types**, auto-fixes the ones that have one obviously-correct
interpretation, and routes everything else to an admin **Review Queue**
(nothing is silently guessed or dropped, per Meera's request).

Result on this file: **30/42 rows auto-imported, 12/42 flagged for review.**

### Auto-fixed (logged in the Import Report, not blocking)

| # | Anomaly | Example row | Policy |
|---|---|---|---|
| 1 | Thousand-separator in amount | `"1,200"` (Electricity Feb) | Strip commas, parse as 1200.00 |
| 2 | Excess decimal precision | `899.995` (Cylinder refill) | Round to 2dp (899.99/900.00) — currency precision |
| 3 | Name case/whitespace mismatch | `priya`, `rohan ` | Auto-match to canonical member (case/whitespace-insensitive) |
| 4 | Non-standard date format | `Mar-14` (no year) | Parsed as `%b-%d`, year assumed 2026 (only year used in sheet) |
| 5 | Missing currency | Groceries DMart (15-03) | Default to INR (group's home currency), logged |
| 6 | USD currency | Goa villa booking, Beach shack lunch, Parasailing, Parasailing refund | Both USD original and INR-converted amounts stored (fixed rate, see Decisions) |
| 7 | Negative amount (refund) | Parasailing refund `-30 USD` | Imported as its own expense with a negative value; reduces each participant's share |
| 8 | Zero amount | Dinner order Swiggy (`₹0`) | Imported as-is — valid placeholder/cancelled-charge record |
| 9 | Guest participant (non-member) in split | "Dev's friend Kabir" (Parasailing) | Added as a one-off **guest** participant — owes their share, not a full group member |
| 10 | Guest payer (non-member paid) | "Dev" pays for Marina Bites, Goa villa, Parasailing, refund | Same as above — Dev gets a guest User record so he can hold a balance |
| 11 | Ambiguous date order | `04-05-2026` (Deep cleaning) — later than the next row's date | Resolved as DD-MM-YYYY (4th May), consistent with the rest of the sheet; surfaced in report as ambiguous |

### Flagged for Review Queue (require a human decision)

| # | Anomaly | Example row | Why it's flagged |
|---|---|---|---|
| 12 | Exact duplicate | "Dinner at Marina Bites" / "dinner - marina bites" (08-02), same amount | Same date + similar description + same amount — likely the same expense logged twice. Admin picks which row to keep. |
| 13 | Conflicting duplicate | "Dinner at Thalassa" (₹2400) vs "Thalassa dinner" (₹2450), same date | Same event, **different** amounts — admin must choose the correct figure. |
| 14 | Settlement logged as expense | "Rohan paid Aisha back" ₹5000, blank split_type, single recipient | Not a shared cost — proposed as a `Payment` (Rohan → Aisha), pending confirmation. |
| 15 | Transfer-like single-recipient row | "Sam deposit share" (split_type=equal, split_with=Aisha) | Looks like a one-to-one transfer logged via the expense form — admin confirms whether it should be a `Payment`. |
| 16 | Percentage split ≠ 100% | Pizza Friday & Weekend brunch (30+30+30+20 = 110%) | Can't silently rescale — admin corrects the percentages. |
| 17 | Fuzzy name match | "Priya S" (Groceries DMart, 18-02) | Similar to "Priya" (≈0.83 similarity) but not an exact match — could be a typo or a different person; admin confirms. |
| 18 | Missing `paid_by` | "House cleaning supplies" (22-02) | No payer at all — cannot create a valid expense. |
| 19 | Inactive member in split | Meera in "Groceries BigBasket" (02-04, after she left 31-03); Sam in "Housewarming drinks" & "Electricity Apr" (before he joined 15-04) | Directly addresses Sam's and Meera's concerns — a person shouldn't owe for expenses outside their membership window. Admin decides whether to remove them from the split. |
| — | `equal` split_type with stray `split_details` | "Furniture for common room" (18-04) | Ambiguous: ignore the details (truly equal) or treat as a `share` split using those numbers? Admin decides. |
| — | Unrecognized name in split_details | "Scooter rentals" referenced "Dev" in details before Dev was resolved | (Fixed in code — Dev is resolved as a guest before split calculation runs; kept here as a class of anomaly the importer can still surface if a name in `split_details` matches nobody.) |

Every flagged row stores the importer's **best-guess parse** (`proposed_data`)
so the admin reviews/edits rather than re-entering from scratch. Approving a
row creates the resulting `Expense`/`Payment` and links it back to the
`ImportRow` for traceability (the "trace this anomaly in code" requirement).

---

## Database Schema (relational, SQLite/Postgres)

### `accounts.User` (custom, extends Django's `AbstractUser`)
| Field | Type | Notes |
|---|---|---|
| id | PK | |
| username | varchar | |
| display_name | varchar | canonical name shown in UI |
| is_guest | bool | True for non-member participants added via import (e.g. Dev, Kabir) |
| ... | | standard Django auth fields (password, email, etc.) |

### `groups.Group`
| Field | Type | Notes |
|---|---|---|
| id | PK | |
| name | varchar | |
| created_by | FK → User | |
| created_at | datetime | |

### `groups.GroupMembership`
| Field | Type | Notes |
|---|---|---|
| id | PK | |
| group | FK → Group | |
| user | FK → User | |
| joined_date | date | |
| left_date | date, nullable | NULL = currently active |

This is what answers "was X a member on date Y?" — checked at import time
and whenever balances are calculated.

### `expenses.Expense`
| Field | Type | Notes |
|---|---|---|
| id | PK | |
| group | FK → Group | |
| description | varchar | |
| date | date | |
| paid_by | FK → User (PROTECT) | |
| amount | decimal(12,2) | original amount as entered |
| currency | enum (INR/USD) | |
| amount_inr | decimal(12,2) | used for all balance math |
| exchange_rate_used | decimal(8,4), nullable | only set for non-INR |
| split_type | enum (equal/unequal/percentage/share) | |
| notes | text | |
| import_row | FK → ImportRow, nullable | provenance |

### `expenses.ExpenseSplit`
| Field | Type | Notes |
|---|---|---|
| id | PK | |
| expense | FK → Expense | |
| user | FK → User | |
| share_amount | decimal(12,2) | this person's share, in INR |
| raw_split_value | varchar | e.g. "30%", "2 shares" — audit trail |

Unique together: `(expense, user)`.

### `expenses.Payment`
| Field | Type | Notes |
|---|---|---|
| id | PK | |
| group | FK → Group | |
| paid_by | FK → User (PROTECT) | |
| paid_to | FK → User (PROTECT) | |
| amount | decimal(12,2) | |
| currency | enum | |
| amount_inr | decimal(12,2) | |
| date | date | |
| notes | text | |
| import_row | FK → ImportRow, nullable | |

### `imports.ImportBatch`
| Field | Type | Notes |
|---|---|---|
| id | PK | |
| group | FK → Group | |
| file_name | varchar | |
| uploaded_by | FK → User, nullable | |
| uploaded_at | datetime | |
| total_rows / auto_imported_count / flagged_count | int | summary for the Import Report |

### `imports.ImportRow`
| Field | Type | Notes |
|---|---|---|
| id | PK | |
| batch | FK → ImportBatch | |
| row_number | int | 1-indexed, matches CSV row |
| raw_data | JSON | original row, untouched |
| status | enum (imported/pending/approved/rejected) | |
| anomalies | JSON list | `{type, description, action}` per issue |
| proposed_data | JSON, nullable | best-guess parse for pending rows |
| reviewed_by / reviewed_at / review_notes | | audit trail for Meera's approval requirement |

---

## Balance Calculation (no magic numbers)

For each user in a group:

```
net_balance =
    Σ amount_inr  (expenses they paid)
  − Σ share_amount (ExpenseSplit rows where they're the participant)
  + Σ amount_inr  (Payments they made)
  − Σ amount_inr  (Payments they received)
```

`Σ(net_balance)` across all users in a group is always **exactly 0** —
verified against the imported dataset. Every number is traceable back to
the `Expense` / `ExpenseSplit` / `Payment` rows that produced it via
`GET /api/groups/{id}/balances/{user_id}/`.

"Who pays whom" (Aisha's requirement) is produced by a greedy debt-
simplification algorithm: match the largest debtor against the largest
creditor repeatedly until all balances are zero, minimizing the number of
transactions.
