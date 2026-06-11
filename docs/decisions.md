# Engineering decisions

The tradeoffs behind the platform, and why I made them. Most of these come down to a simple reality: this runs a single business on modest infrastructure, so I optimized for safety and low operational cost over theoretical scale.

## Hourly sync instead of realtime

**Decision:** the Airtable sync is a plain in-process async loop on a roughly one-hour interval, not realtime and not a separate job queue.

**Why:** realtime sync would have meant standing up a worker queue (Celery/Redis) and making far more frequent API calls, which runs straight into Airtable's rate limits. For a single-business tool that's a lot of moving parts and cost for little benefit. I accepted that mirrored data can be up to an hour stale, and added an Airtable webhook receiver for the cases where fresher data actually matters. To make hourly staleness safe rather than dangerous, I paired it with the authoritative-table pattern so local edits are never silently overwritten between syncs.

## In-process schedulers instead of a worker fleet

**Decision:** background jobs (sync, timesheet dispatch, reconciliation) run as asyncio loops inside the API process's lifespan.

**Why:** no separate worker deployment to run, monitor, and pay for. The cost is that the jobs share the API's lifecycle, so I made each loop independently error-isolated (one failed tick never stops the next) and idempotent where it matters most — the timesheet dispatcher records the last period it sent so a restart can't double-email the accountant.

## Migrate table-by-table, not big-bang

**Decision:** move off Airtable one table at a time using per-table authoritative flags and a parallel-run period.

**Why:** the Airtable base was production-critical with 18 interconnected tables. A single cut-over would have been an all-or-nothing risk on live business data. The incremental pattern trades a longer migration timeline for the ability to verify each table, keep both systems consistent during transition, and roll back cleanly if drift showed up.

## Audit attribution baked into the data

**Decision:** the inventory change log records not just what changed, but the source (human vs. sync vs. import).

**Why:** the most common operational question was "why is this number different?" Without attribution, a manual correction and a sync overwrite look identical after the fact. Recording the source turned a guessing game into a one-line answer, and made it safe to let both humans and the sync write to the same rows.

## Security on the unauthenticated kiosk

**Decision:** the public PIN kiosk uses a deterministic SHA-256 lookup index, bcrypt verification, failure lockout, and device tagging.

**Why:** it's an unauthenticated screen on a shared device, so it's the softest target in the system. The hashed lookup means a correct PIN resolves to one person without ever storing PINs in the clear, and lockout plus device tagging limit both guessing and untraceable punches — without putting passwords on the shop floor.

## Keep the two domains decoupled

**Decision:** no cross-imports between the workforce code and the inventory/integration code.

**Why:** discipline up front kept the codebase navigable, and it paid off concretely later — the workforce half could be lifted out into its own standalone product repository without unpicking a tangle of shared dependencies.
