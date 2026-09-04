# Make blueprints

Snapshots of the Make scenarios that write into the `Masterlist` Airtable base
(`appqJeEPUeOPaQC3y`), kept in git so changes are reviewable.

| File | Scenario | ID | Schedule |
|---|---|---|---|
| `9550684-ghl-status-to-airtable.current.json` | [BeFound] GHL Status → Airtable (Tracy) | 9550684 | webhook |
| `9739026-backfill-appointment-date-from-acuity.json` | [BeFound] Appointment Date reconcile from Acuity (daily) | 9739026 | daily 02:00 SGT |

`9550684-...current.json` reflects the live scenario. Its only edit is the
`GHL Status` normalisation; the Appointment Date mapping was deliberately left
out — see `docs/appointment-date-gap.md` for why.

Re-export after editing anything in Make so these snapshots stay truthful.
