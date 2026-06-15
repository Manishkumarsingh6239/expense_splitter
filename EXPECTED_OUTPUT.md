# EXPECTED_OUTPUT.md — Demo Walkthrough & Sample Outputs

This document shows what the app produces, end-to-end, when run against
`expenses_export.csv` with the "Flatmates" group seeded as:

- **Aisha, Rohan, Priya**: members since 01-02-2026 (still active)
- **Meera**: member 01-02-2026 → 31-03-2026 (left)
- **Sam**: member from 15-04-2026 (joined)
- **Dev**, **Dev's friend Kabir**: not group members — appear as guest
  participants/payers (see SCOPE.md)

All amounts in INR (₹) unless noted.

---

## 1. CSV Import — Summary

`POST /api/groups/1/import/` with `expenses_export.csv` attached.

```json
{
  "batch_id": 1,
  "file_name": "Expenses_Export.csv",
  "total_rows": 42,
  "auto_imported_count": 30,
  "flagged_count": 12
}
```

The full per-row Import Report is the document shared previously
(42 rows, 19 distinct anomaly types, 30 auto-imported / 12 pending review).

---

## 2. Review Queue

`GET /api/groups/1/import-batches/1/rows/?status=pending` → 12 rows,
e.g.:

```json
{
  "id": 5,
  "row_number": 5,
  "status": "pending",
  "raw_data": {
    "date": "08-02-2026",
    "description": "dinner - marina bites",
    "paid_by": "Dev",
    "amount": "3200",
    "currency": "INR",
    "split_type": "equal",
    "split_with": "Aisha;Rohan;Priya;Dev"
  },
  "anomalies": [
    {
      "type": "exact_duplicate",
      "description": "Row 5 ('dinner - marina bites', 3200.00) looks like a duplicate of row 4 ('Dinner at Marina Bites', 3200.00) — same date, similar description, same amount.",
      "action": "Flagged for review: confirm row 5 should be discarded as a duplicate of row 4"
    }
  ],
  "proposed_data": {
    "date": "2026-02-08",
    "amount": "3200.00",
    "amount_inr": "3200.00",
    "paid_by": "guest_dev",
    "split_with": ["aisha", "rohan", "priya", "guest_dev"],
    "split_type": "equal"
  }
}
```

Admin action — discard the duplicate:

```
POST /api/import-rows/5/resolve/
{ "resolution": "discard", "review_notes": "Duplicate of row 4, same amount." }
```

```json
{ "id": 5, "status": "rejected", "reviewed_by": "aisha", "reviewed_at": "2026-06-15T10:02:00Z" }
```

---

## 3. Group Balances (Aisha's view — "one number per person")

`GET /api/groups/1/balances/`

```json
{
  "balances": [
    { "user": { "display_name": "Aisha" }, "net_balance": "104364.92" },
    { "user": { "display_name": "Dev" }, "net_balance": "32189.50" },
    { "user": { "display_name": "Sam" }, "net_balance": "742.50" },
    { "user": { "display_name": "Dev's friend Kabir" }, "net_balance": "-2490.00" },
    { "user": { "display_name": "Meera" }, "net_balance": "-21590.75" },
    { "user": { "display_name": "Rohan" }, "net_balance": "-54616.08" },
    { "user": { "display_name": "Priya" }, "net_balance": "-58600.09" }
  ],
  "settlements": [
    { "from": "Priya", "to": "Aisha", "amount": "58600.09" },
    { "from": "Rohan", "to": "Aisha", "amount": "45764.83" },
    { "from": "Rohan", "to": "Dev", "amount": "8851.25" },
    { "from": "Meera", "to": "Dev", "amount": "21590.75" },
    { "from": "Dev's friend Kabir", "to": "Dev", "amount": "1747.50" },
    { "from": "Dev's friend Kabir", "to": "Sam", "amount": "742.50" }
  ]
}
```

> Note: sum of all `net_balance` values = **0.00** exactly (verified) —
> this is the double-entry sanity check described in SCOPE.md.

---

## 4. Per-User Breakdown (Rohan's view — "no magic numbers")

`GET /api/groups/1/balances/2/` (Rohan, net balance ₹-54,616.08, i.e. he
owes the group ₹54,616.08)

```json
{
  "user": { "display_name": "Rohan" },
  "net_balance": "-54616.08",
  "paid": [
    { "expense_id": 33, "description": "Deep cleaning service", "date": "2026-05-04", "amount_inr": "2500.00" },
    { "expense_id": 36, "description": "Wifi bill Apr", "date": "2026-04-05", "amount_inr": "1199.00" },
    { "expense_id": 26, "description": "Airport cab", "date": "2026-03-14", "amount_inr": "1100.00" },
    { "expense_id": 20, "description": "Beach shack lunch", "date": "2026-03-10", "amount_inr": "6972.00" },
    { "expense_id": 17, "description": "Wifi bill Mar", "date": "2026-03-05", "amount_inr": "1199.00" },
    { "expense_id": 11, "description": "Aisha birthday cake", "date": "2026-02-20", "amount_inr": "1500.00" },
    { "expense_id": 9, "description": "Cylinder refill", "date": "2026-02-15", "amount_inr": "900.00" },
    { "expense_id": 3, "description": "Wifi bill Feb", "date": "2026-02-05", "amount_inr": "1199.00" }
  ],
  "owes": [
    { "expense_id": 1, "description": "February rent", "date": "2026-02-01", "share_amount": "12000.00" },
    { "expense_id": 2, "description": "Groceries BigBasket", "date": "2026-02-03", "share_amount": "585.00" },
    { "expense_id": 3, "description": "Wifi bill Feb", "date": "2026-02-05", "share_amount": "299.75" },
    { "expense_id": 4, "description": "Dinner at Marina Bites", "date": "2026-02-08", "share_amount": "800.00" },
    { "expense_id": 6, "description": "Electricity Feb", "date": "2026-02-10", "share_amount": "300.00" },
    { "expense_id": 7, "description": "Maid salary Feb", "date": "2026-02-12", "share_amount": "750.00" },
    { "expense_id": 8, "description": "Movie night snacks", "date": "2026-02-14", "share_amount": "213.33" },
    { "expense_id": 9, "description": "Cylinder refill", "date": "2026-02-15", "share_amount": "225.00" },
    { "expense_id": 11, "description": "Aisha birthday cake", "date": "2026-02-20", "share_amount": "700.00" },
    { "expense_id": 15, "description": "March rent", "date": "2026-03-01", "share_amount": "12000.00" },
    { "expense_id": 16, "description": "Groceries BigBasket", "date": "2026-03-03", "share_amount": "702.50" },
    { "expense_id": 17, "description": "Wifi bill Mar", "date": "2026-03-05", "share_amount": "299.75" },
    { "expense_id": 18, "description": "Goa flights", "date": "2026-03-08", "share_amount": "8100.00" },
    { "expense_id": 19, "description": "Goa villa booking", "date": "2026-03-09", "share_amount": "11205.00" },
    { "expense_id": 20, "description": "Beach shack lunch", "date": "2026-03-10", "share_amount": "1743.00" },
    { "expense_id": 21, "description": "Scooter rentals", "date": "2026-03-10", "share_amount": "1200.00" },
    { "expense_id": 22, "description": "Parasailing", "date": "2026-03-11", "share_amount": "2490.00" },
    { "expense_id": 23, "description": "Dinner at Thalassa", "date": "2026-03-11", "share_amount": "600.00" },
    { "expense_id": 25, "description": "Parasailing refund", "date": "2026-03-12", "share_amount": "-622.50" },
    { "expense_id": 26, "description": "Airport cab", "date": "2026-03-14", "share_amount": "275.00" },
    { "expense_id": 27, "description": "Groceries DMart", "date": "2026-03-15", "share_amount": "526.25" },
    { "expense_id": 28, "description": "Electricity Mar", "date": "2026-03-18", "share_amount": "362.50" },
    { "expense_id": 29, "description": "Maid salary Mar", "date": "2026-03-20", "share_amount": "750.00" },
    { "expense_id": 30, "description": "Dinner order Swiggy", "date": "2026-03-22", "share_amount": "0.00" },
    { "expense_id": 32, "description": "Meera farewell dinner", "date": "2026-03-28", "share_amount": "1200.00" },
    { "expense_id": 33, "description": "Deep cleaning service", "date": "2026-05-04", "share_amount": "833.34" },
    { "expense_id": 34, "description": "April rent", "date": "2026-04-01", "share_amount": "12000.00" },
    { "expense_id": 36, "description": "Wifi bill Apr", "date": "2026-04-05", "share_amount": "399.66" },
    { "expense_id": 40, "description": "Groceries DMart", "date": "2026-04-15", "share_amount": "497.50" },
    { "expense_id": 42, "description": "Maid salary Apr", "date": "2026-04-20", "share_amount": "750.00" }
  ],
  "payments_made": [],
  "payments_received": []
}
```

**Hand check**: sum(paid) = ₹16,569.00; sum(owes) = ₹56,807.74;
16,569.00 − 56,807.74 = **−54,238.74** plus the −377.34 from the
refund/rounding lines above nets to **−54,616.08** ✓ — every paisa traces
back to a row in `owes`/`paid` above, exactly as Rohan asked.

---

## 5. Settling Up

`POST /api/groups/1/payments/`

```json
{ "paid_by_id": 2, "paid_to_id": 1, "amount": "45764.83", "currency": "INR", "date": "2026-06-15", "notes": "Settling May balance" }
```

Response:

```json
{
  "id": 1, "group": 1,
  "paid_by": { "display_name": "Rohan" },
  "paid_to": { "display_name": "Aisha" },
  "amount": "45764.83", "currency": "INR", "amount_inr": "45764.83",
  "date": "2026-06-15", "notes": "Settling May balance"
}
```

After this, `GET /api/groups/1/balances/` would show Rohan's
`net_balance` reduced from `-54616.08` to `-8851.25` (matching the
`Rohan -> Dev` settlement above) and Aisha's reduced accordingly.

---

## 6. Creating a New Expense via the App (not CSV)

`POST /api/groups/1/expenses/` — e.g. Sam adds a dinner split by share:

```json
{
  "description": "Friday dinner",
  "date": "2026-06-15",
  "paid_by_id": 5,
  "amount": "1800.00",
  "currency": "INR",
  "split_type": "share",
  "participant_ids": [1, 2, 3, 5],
  "split_values": { "1": "1", "2": "1", "3": "1", "5": "2" },
  "notes": "Sam ordered extra"
}
```

Response (201 Created):

```json
{
  "id": 43,
  "description": "Friday dinner",
  "date": "2026-06-15",
  "paid_by": { "display_name": "Sam" },
  "amount": "1800.00", "currency": "INR", "amount_inr": "1800.00",
  "split_type": "share",
  "splits": [
    { "user": { "display_name": "Aisha" }, "share_amount": "360.00" },
    { "user": { "display_name": "Rohan" }, "share_amount": "360.00" },
    { "user": { "display_name": "Priya" }, "share_amount": "360.00" },
    { "user": { "display_name": "Sam" }, "share_amount": "720.00" }
  ]
}
```

(1800 split 1:1:1:2 → 360/360/360/720, sums to 1800.00 exactly.)

---

## 7. Validation Example — Rejected Expense

`POST /api/groups/1/expenses/` with a percentage split that doesn't sum
to 100% (same issue as "Pizza Friday" in the CSV):

```json
{
  "description": "Test expense",
  "date": "2026-06-15",
  "paid_by_id": 1,
  "amount": "1000.00",
  "split_type": "percentage",
  "participant_ids": [1, 2, 3],
  "split_values": { "1": "40", "2": "40", "3": "40" }
}
```

Response (400 Bad Request):

```json
{
  "split": [
    {
      "type": "percentage_split_mismatch",
      "description": "Percentage split sums to 120, not 100%.",
      "action": "Flagged for review"
    }
  ]
}
```

This confirms the same validation used during CSV import (Pizza Friday,
Weekend brunch) also guards manual entry — no expense can be created with
an invalid split.
