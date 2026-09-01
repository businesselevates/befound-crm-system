# Booked Discovery: rows synced from GHL have no Appointment Date

Audit date: 2026-09-01
Base: `Masterlist` (`appqJeEPUeOPaQC3y`) → table `Booked Discovery` (`tbl9RnE9oy5G29AN7`)
Scenario: **[BeFound] GHL Status → Airtable (Tracy)** (Make id `9550684`, webhook `4269673`)

## Summary

293 rows in `Booked Discovery` carry a `GHL Status` but no `Appointment Date`.

The date is **not being erased** — it was never written. The GHL sync only ever
maps two fields, and `Appointment Date` is not one of them. Existing dates are
safe: the update branch of the scenario does not touch that column at all.

## Numbers

| Bucket | Rows |
|---|---:|
| Total rows in `Booked Discovery` | 3,145 |
| `Appointment Date` set, no `GHL Status` | 1,639 |
| `Appointment Date` set **and** `GHL Status` set | 1,213 |
| **`GHL Status` set, `Appointment Date` empty** | **293** |

Of the 293:

- All 293 are linked to a lead in `Overview (Leads In)` and resolve to an email
  through the `Primary Email` lookup — so they are matchable against any
  external booking source by email.
- **292** belong to a lead that has *no other* `Booked Discovery` row. These
  bookings were simply never logged in Airtable before the sync existed.
- **1** (`recvhEihpY3S73xCI`, "Wen Ting") is a duplicate: the same lead later got
  two dated rows (created 2026-07-24 and 2026-08-01, i.e. *after* the sync row of
  2026-07-21). This one should be merged or deleted, not backfilled.

When they were created — the initial catch-up run plus live webhooks since:

| Created | Rows |
|---|---:|
| 2026-07-21 (first run / catch-up) | 270 |
| 2026-07-24 – 07-25 | 4 |
| 2026-08-07 – 08-08 | 10 |
| 2026-08-19 | 9 |

GHL status spread: No Show 149, Didn't Buy 66, Rescheduled 35, Cancelled 31,
"Didn't buy" 12.

## Root cause

`GHL Status` is a plain text field written by scenario 9550684. The scenario has
two branches:

1. **Record exists → update** (module 5). Writes `GHL Status` (`fldsvvnobT5os0cPn`)
   and `GHL Status Updated` (`fldDaPxwr7ZJxbox3`). Nothing else. An existing
   `Appointment Date` is left untouched. ✅
2. **No record → find lead → create** (module 7). Creates a brand new
   `Booked Discovery` row mapping only `Opportunity Name` (the link to the lead),
   `GHL Status` and `GHL Status Updated`. **`Appointment Date`
   (`fldc8iSqiEGVmP8qB`) is not in the mapper at all.** ❌

Branch 2 is where all 293 rows come from. The webhook payload the scenario reads
(`1.email`, `1.status`, `1.changed_at`, `1.customData.*`) carries no appointment
date, so there is nothing to map even if the field were added to the mapper.

Historically `Appointment Date` has always been filled in by hand — no scenario
in the account has ever written that column. The legacy Acuity watcher
(`[BeFound] Entries Logging`, id 6661810, off) logged appointments to Google
Sheets and only read from `Overview (Leads In)`.

## `GHL Status Updated` is not a usable substitute

Tested against the 1,213 rows that have both values, comparing
`GHL Status Updated` to the real `Appointment Date`:

| Gap | Share |
|---|---:|
| Same day | 36.8% |
| 1 day | 11.5% |
| 2–7 days | 14.1% |
| 8–30 days | 6.7% |
| 31–180 days | 5.8% |
| **> 180 days** | **21.5%** |
| Status timestamp *before* the appointment | 3.6% |

Median 1 day, but the 75th percentile is 118 days and the maximum is 945. Copying
`GHL Status Updated` into `Appointment Date` would put roughly a third of the
rows off by months. Do not do it.

## Where the real dates can come from

The date has to be re-imported from a system that actually holds the booking:

- **GoHighLevel** — the source of the status. There is no GHL connection in the
  Make account (the integration is a one-way webhook), so this needs either the
  GHL workflow extended to send the appointment date in its custom data, or a
  CSV export of appointments/opportunities matched by email.
- **Acuity Scheduling** — connected in Make (`14351824`, `10624867`) and already
  queried by email in `[BeFound] Skin Barrier Quiz → Booked Reconcile` (id
  9617664) via `acuity-scheduling:listAppointments`. Needs confirmation that
  Acuity holds bookings back to 2023, since the affected rows span 2023-10 to
  2026-08.

Either way the join key is the email on the `Primary Email` lookup, which every
one of the 293 rows has.

## Fix, in two parts

**Stop the bleeding (scenario 9550684).** Add `Appointment Date`
(`fldc8iSqiEGVmP8qB`) to the create branch, sourced from whichever system is
chosen above. In the update branch, only write it when the cell is currently
empty, so a hand-entered date is never overwritten — that requires adding
`Appointment Date` to module 2's returned fields and to module 3's aggregator so
the current value is readable.

**Backfill the 293.** Look each row up by email, write the date onto the record
IDs listed in `data/booked-discovery-missing-appointment-date.csv`, and handle
`recvhEihpY3S73xCI` as a duplicate rather than a backfill.

## Loose end worth fixing while in there

`GHL Status` is free text, and GHL is sending two spellings of the same status:
"Didn't Buy" (66) and "Didn't buy" (12). Any report grouping on this column
splits that stage in two. Normalising the value in the scenario — or converting
the column to a single select — closes it.

## Files

- `data/booked-discovery-missing-appointment-date.csv` — the 293 rows, with
  Airtable record ID, lead record ID, name, email, GHL status and timestamps,
  plus an empty `appointment_date` column to fill.
- `make/9550684-ghl-status-to-airtable.current.json` — the scenario as it is
  today, before any change.
