# Airtable migration

The business ran on a large, mature Airtable base — 18 linked tables, 1,400+ fields, modeling inventory, production, transactions, and multi-channel orders. The platform's long-term goal is to become the system of record without forcing a risky, all-at-once switch. This is how that migration works.

## Starting point: a complex base

The Airtable base is a hub-and-spoke design. Master Items and Variants sit at the center with the most fields and the broadest relationships, linking out to ingredients, products, BOM, production runs, transactions, transfers, orders, and both retail and wholesale customer tables. There's even a Sync Errors table — the base was production-critical and already had its own error tracking.

Migrating something this connected in one move would be reckless. So the design is incremental and reversible at every step.

## Two-way sync

A sync layer mirrors the Airtable tables into Postgres. It runs as an in-process async loop that wakes on a configurable interval (about an hour by default) and syncs the tables **in dependency order**, so foreign-key relationships resolve correctly — master items before variants, variants before transactions, and so on.

Design properties that keep it safe:

- **Per-table error isolation** — one table failing doesn't halt the rest; each failure is logged on its own.
- **A startup stagger** before the first sync so the API and migrations finish booting first.
- **Graceful shutdown** so a restart never leaves a sync half-done.
- **Audit events** written for each run, so the UI can show "last synced" timestamps and a header status dot.

## The cut-over pattern: authoritative tables

The heart of the migration is a per-table **authoritative** flag, stored in an app-config table in Postgres. Rather than a single binary "use Airtable / use Postgres" switch, each table moves independently:

1. **Parallel run.** For a few weeks the platform reads from Airtable but also pushes its own changes back, keeping both sides current.
2. **Verify drift.** A drift-detection service compares the two so you can confirm the local copy is trustworthy.
3. **Lock in Airtable.** Once confident, the table is locked down on the Airtable side.
4. **Mark authoritative.** The table is flagged authoritative in Postgres. From then on, the sync **stops pulling** that table (so it can't overwrite local edits) but **keeps pushing** to Airtable so anything still reading from Airtable stays current.

This lets the business migrate one table at a time — suppliers, ingredients, retailers, BOM, wholesale pricing, production runs — instead of betting everything on one cut-over day.

## Operability

- **A runtime toggle** flips auto-sync on or off from the dashboard, stored in config and effective on the next tick — no service restart, with a fall-back to an environment variable for backward compatibility.
- **Defensive config.** Unknown table keys are rejected and malformed payloads are tolerated gracefully, so a typo in configuration can never silently lock a table or crash the UI.
- **Near-realtime option.** An Airtable webhook receiver exists to push changes in closer to realtime, reducing reliance on the polling interval where it matters.

## Why polling instead of realtime everywhere

The default hourly loop is a deliberate cost/latency tradeoff — covered in [decisions.md](./decisions.md).
