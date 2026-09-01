# BeFound CRM system

Working notes, audits and automation snapshots for the BeFound CRM — the
`Masterlist` Airtable base (`appqJeEPUeOPaQC3y`) and the Make scenarios that
feed it.

- `docs/` — audits and root-cause write-ups.
- `data/` — extracts pulled for a specific audit. Point-in-time, not a live mirror.
- `make/` — blueprint snapshots of the scenarios that write into the base.

## Current

- [`docs/appointment-date-gap.md`](docs/appointment-date-gap.md) — why 293
  `Booked Discovery` rows synced from GoHighLevel had no `Appointment Date`,
  how 287 of them were recovered from Acuity, and the ten rows still needing a
  human.
