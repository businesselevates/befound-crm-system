# Booked Discovery: rows synced from GHL had no Appointment Date

Audit and fix, 2026-09-01
Base: `Masterlist` (`appqJeEPUeOPaQC3y`) → table `Booked Discovery` (`tbl9RnE9oy5G29AN7`)

## Summary

293 rows carried a `GHL Status` but no `Appointment Date`. The date was never
erased — it was never written. The GHL sync maps two fields and `Appointment
Date` is not one of them.

287 of the 293 have now been filled from Acuity Scheduling, matched by email.
The remaining 6 have no Acuity appointment under their address and need to be
filled by hand. A daily reconcile keeps new rows from reopening the gap.

## Root cause

`GHL Status` is written by Make scenario **[BeFound] GHL Status → Airtable
(Tracy)** (id `9550684`, webhook `4269673`). It has two branches:

1. **Record exists → update** (module 5) writes `GHL Status`
   (`fldsvvnobT5os0cPn`) and `GHL Status Updated` (`fldDaPxwr7ZJxbox3`), nothing
   else. An existing `Appointment Date` is never touched.
2. **No record → find lead → create** (module 7) creates a new row mapping only
   the `Opportunity Name` link, `GHL Status` and `GHL Status Updated`.
   **`Appointment Date` (`fldc8iSqiEGVmP8qB`) is not in the mapper**, and the
   webhook payload (`1.email`, `1.status`, `1.changed_at`, `1.customData.*`)
   carries no appointment date to map even if it were.

All 293 rows came from branch 2 — 270 in the first catch-up run on 2026-07-21,
the rest from live webhooks since. 292 of them belong to a lead with no other
`Booked Discovery` row: bookings that had never been logged in Airtable at all.

No scenario in the account has ever written `Appointment Date`. It has always
been typed in by hand, which is why gaps existed for the sync to land in.

## `GHL Status Updated` was ruled out as a substitute

Measured against the 1,213 rows that had both values: same day for 36.8%, but
off by more than 180 days for 21.5%, up to 945 days. Median 1 day, 75th
percentile 118. Copying it across would have put a fifth of the table off by
months.

## The fix

**Scenario `9739026` — [BeFound] Appointment Date reconcile from Acuity
(daily)**, 02:00 SGT.

1. Airtable search on `Booked Discovery`: `AND({GHL Status} != "",
   {Appointment Date} = BLANK())`.
2. Acuity `listAppointments` by the row's `Primary Email` lookup, from
   2023-01-01, canceled appointments included, earliest first, one result.
3. Airtable update writing `Appointment Date` only, and only when Acuity
   returned an appointment.

It never writes any other field and never touches a row that already has a
date, so it is safe to re-run. Rows with no Acuity match are left empty rather
than guessed at.

### Backfill result, run 2026-09-01

| | Rows |
|---|---:|
| Had `GHL Status`, no `Appointment Date` | 293 |
| Filled from Acuity | 287 |
| No Acuity match, left empty | 6 |
| Flagged for review (see below) | 4 |

Verified after the run: table still holds 3,145 rows, and **no row that already
had an `Appointment Date` had it changed or cleared**.

Sanity check on the 287 written dates, comparing each to its `GHL Status
Updated` timestamp:

| Gap | Share |
|---|---:|
| Same day | 20.9% |
| 1–7 days | 29.3% |
| 8–30 days | 29.6% |
| 31–180 days | 13.9% |
| Appointment after the status change, within 60 days (cancelled ahead of time) | 4.9% |
| **Implausible (< −60 or > 180 days)** | **1.4%** |

Median 5 days, 95th percentile 63. For comparison, the 1,213 hand-entered rows
show 21.5% beyond 180 days — the backfilled dates sit tighter to their status
change than the existing manual data does.

### Rows that still need a human

Six with no Acuity match — mostly 2023–2024, before the current booking setup,
or booked under a different address:

| Record | Lead | Email | GHL status |
|---|---|---|---|
| `recbQSIsT8iVEQZIf` | Vera | kongmingwai_vera@live.com | Cancelled |
| `recm4f7TkPF8kjxJx` | Nessa (Lim) | nessalim27@gmail.com | No Show |
| `recks1MLoQ2GW7E96` | ChloeLim | chloe@xjgroup.com.sg | Cancelled |
| `recfRFUVtM98M2ocg` | Wang Li | sararoof2003@yahoo.com.sg | Cancelled |
| `rec3DNferexQY373H` | Weishan (Teo) | weishanno@gmail.com | No Show |
| `recH5VlUu950QyLvQ` | Norliza Suhaimi | lizacine@gmail.com | Didn't Buy |

Four where the gap to the status change is implausible. These leads have
several Acuity bookings and the reconcile takes the earliest, so the wrong one
may have been picked — worth eyeballing:

| Record | Lead | Date written | Status changed | Gap |
|---|---|---|---|---|
| `recqEemWRWYSQf3E5` | MelissaChua | 2026-07-18 | 2024-06-25 | −753 d |
| `recJvQCaAjBerh6TP` | Mummhd | 2026-03-06 | 2025-10-07 | −150 d |
| `recucfQJMm44LdD7G` | faiqah | 2025-10-31 | 2026-06-27 | +239 d |
| `reczKY1sBVIhAs98C` | Shida | 2025-11-22 | 2026-06-27 | +217 d |

All ten are marked in the `review` column of
`data/booked-discovery-missing-appointment-date.csv`.

## Why scenario 9550684 was left alone

Adding the Acuity lookup to its create branch would put a search module in
front of the record creation. A Make search that returns nothing halts the rest
of the branch, so any lead without an Acuity appointment would stop being
recorded in `Booked Discovery` at all — trading a missing date for a missing
row. The daily reconcile closes the same gap within a day without that risk,
and also catches the case where the appointment is booked after the status
webhook fires.

If you would rather have the date land on the row the moment it is created,
the safe shape is a third router branch (create the row, then look up Acuity
and update it) rather than a lookup inline in the existing branch.

## GHL Status normalisation

`GHL Status` is a free-text field and GHL was sending two spellings of the same
stage — "Didn't Buy" (481) and "Didn't buy" (12) — which split that stage in two
in any grouped report. The other three values were already consistent.

Canonical set, Title Case: **Cancelled**, **Didn't Buy**, **No Show**,
**Rescheduled**. All apostrophes are straight (U+0027).

Fixed on both sides:

- The 12 rows spelled "Didn't buy" were updated in place. The column now holds
  exactly four distinct values — Cancelled 520, Didn't Buy 493, No Show 455,
  Rescheduled 38 — across the same 1,506 rows as before.
- Scenario `9550684` now normalises on write. Both its status mappings (the
  update branch and the create branch) pass the incoming value through:

  ```
  {{switch(lower(trim(ifempty(1.status; 1.customData.status)));
     "cancelled"; "Cancelled"; "canceled"; "Cancelled";
     "didn't buy"; "Didn't Buy"; "didn’t buy"; "Didn't Buy"; "didnt buy"; "Didn't Buy";
     "no show"; "No Show"; "no-show"; "No Show"; "noshow"; "No Show";
     "rescheduled"; "Rescheduled";
     trim(ifempty(1.status; 1.customData.status)))}}
  ```

  It folds case, trims whitespace, and absorbs the obvious variants (US
  "canceled", a curly apostrophe, a missing apostrophe, "no-show"). The last
  argument is the fallback: anything GHL sends that is not in the list is
  written through as-is rather than blanked, so a stage added later still lands
  in the column — just unnormalised, where it is visible and easy to fold in.

Nothing else in the scenario changed.

## Files

- `data/booked-discovery-missing-appointment-date.csv` — all 293 rows with the
  date written, the gap to the status change, and a `review` flag on the ten
  that need a human.
- `make/9739026-backfill-appointment-date-from-acuity.json` — the reconcile
  scenario.
- `make/9550684-ghl-status-to-airtable.current.json` — the GHL sync as it
  stands, with the status normalisation applied.
