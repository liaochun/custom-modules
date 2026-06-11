# Inventory & production modules

The operations half of the platform manages the physical business: what's in stock, what's being made, what needs buying, and how all of that reconciles against the orders flowing in from retail and wholesale channels. It's also the side that talks to Airtable and Shopify.

## Master Items — the editable grid

The catalog hub is an Airtable-style editable grid for Master Items. It behaves the way the team already expected from Airtable, but on top of the platform's own database:

- **In-place editing** — click a cell, type, and it saves on blur or Enter. No separate save step.
- **Optimistic updates** with automatic rollback and an error surfaced if the write fails.
- **Inline selects** for status and type fields.

### The per-field change log

The most useful addition is a full audit trail. A dedicated change-log table records **every field change** with:

- the old value and the new value,
- who made the change,
- and the **source** — a human dashboard edit (`user`), an Airtable sync overwrite (`sync`), or a reserved import path (`csv`).

That source attribution solved a real, recurring problem: when a number looked wrong, nobody could tell whether someone had mistyped it or whether a sync had quietly overwritten a local correction. Now there's a per-row history drawer and a global, source-filterable changes feed that answers the question directly.

## Recipes / BOM and production

- **Recipes / bill-of-materials** define what goes into each product.
- **Production runs** track what's actually being made, against **production rates** the system can tune over time.
- The ops dashboard surfaces **production-capacity bottlenecks** so managers can see where throughput is constrained.

## Purchasing and replenishment

- **Purchase orders** and a **replenishment** engine that produces a buy list.
- An **inventory ledger** and a recommendation service back the replenishment suggestions.
- **Transfers** track stock moving between locations, including what's in flight.

## Integrations

- **Airtable** — 18 mirrored tables kept in sync, with a two-way layer covered in detail in [airtable-migration.md](./airtable-migration.md).
- **Shopify** — inventory sync plus a webhook receiver for fulfillment events, so order activity reflects in the platform without manual entry.

## The operations dashboard

`/admin/inventory/dashboard` is a single 17-section operations view: headline stats, a revenue snapshot with trend and top SKUs/customers/channels, today's shifts, an "action required" list, transfers in flight, open orders, production-capacity bottlenecks, the buy list, and at-risk customers. It auto-refreshes about once a minute, has jump navigation between sections, and shows banners when data is stale or a sync error has occurred.
