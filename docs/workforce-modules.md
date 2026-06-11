# Workforce modules

The workforce half of the platform replaced a separate scheduling/time-tracking app and a pile of spreadsheets. It covers the full loop: schedule staff, let them clock in, track hours, handle corrections and time off, and get pay-period timesheets to the accountant automatically.

## Scheduling

Managers work out of `/admin/schedule`, which offers week, month, and by-staff views.

- Batch shift creation and a shift-preset library for common patterns.
- An edit modal whose assignee dropdown is filtered to people who don't already have an overlapping shift, so you can't accidentally double-book someone.
- Soft-cancel (recoverable) and hard-delete (permanent) as distinct actions.
- A per-day staffing pill and a weekly-total column so coverage gaps are visible at a glance.

Staff get a read-only `/me/schedule` and a live `/me/hours` page showing today's worked time, a running week tally, and a way to file correction requests.

## Kiosk clock-in

`/kiosk/clock` is a public screen on a shared device — no login. Staff punch in and out with a PIN. The security model matters here because it's unauthenticated and on the shop floor:

- **Deterministic hashed lookup.** PINs are found through a SHA-256 lookup index rather than scanning, so a correct PIN resolves to exactly one person without storing the PIN in the clear.
- **Bcrypt verification** of the actual secret.
- **Lockout after repeated failures** to stop someone standing there guessing.
- **Device tagging** so punches are attributable to a station.

## Punches and corrections

Every punch links to its scheduled shift through a foreign key, so worked time can be compared against what was planned. When a punch is wrong (forgot to clock out, clocked in at the wrong station), staff file a correction and it lands in an approval queue at `/admin/punch-corrections` for a manager to accept or reject.

## Timesheets — automated dispatch

This is the piece that removed a recurring manual chore. A background scheduler:

- Wakes on a configurable interval but only acts on the **8th and 22nd** of each month (the pay-period boundaries), in the business's timezone.
- Builds a CSV for the just-closed period — staff, email, dates, clock in/out, hours, minutes, location.
- Emails it to every active accountant-role user plus any configured extra recipients, de-duplicated.
- Is **idempotent**: it records the last period it sent in config, so a restart or an overlapping tick never double-sends.
- Isolates errors per send, logs both successes and failures, and supports a manual "force this period" trigger for testing or recovery.

## Discipline / PIP workflows

The platform also tracks disciplinary actions and performance-improvement workflows, including auto-generated forms, so that side of people-management lives in the same system as scheduling and attendance rather than in scattered documents.

## Time off, availability, sick calls

Supporting surfaces round out the workforce domain: availability windows, time-off requests, and sick-call handling, all feeding back into what scheduling shows managers.

## Why this half could stand alone

Because the workforce domain has no dependencies on the inventory/Airtable code, it was later possible to export it into its own standalone repository as a focused product, without dragging along the integration machinery.
