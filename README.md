# custom-modules

A showcase of an **internal operations platform** — a self-hosted system I designed and built to run a real consumer-products business. It replaced a sprawling Airtable base and a separate workforce app with one platform that owns scheduling, time tracking, inventory, production, and a gradual migration off Airtable into a proper database.

> **This repository is a portfolio showcase, not the product.**
> It documents the architecture, modules, and engineering decisions behind the platform. It intentionally contains **no runnable source code, configuration, secrets, or data**. You cannot clone and deploy this — that is by design. See [`LICENSE`](./LICENSE) and the notice at the bottom of this page.

---

## What it is

The platform is a single internal dashboard that several different kinds of users sign into:

- **Managers** schedule staff, review timesheets, approve punch corrections, and run inventory/production.
- **Staff** see their own schedule and hours, and request corrections.
- **A public kiosk** lets staff clock in and out with a PIN, no login required.

Built primarily with AI tooling (Claude Code), the system is a real production application that runs the business day to day — it is not a demo or a toy project.

## Stack

| Layer | Technology |
|---|---|
| Backend | FastAPI (Python 3.12), SQLAlchemy, Alembic |
| Frontend | Next.js 14 (App Router, TypeScript), Tailwind |
| Database | PostgreSQL |
| Cache / queue | Redis |
| Auth | Google SSO (domain-restricted) + invite flow |
| Email | Resend |
| Integrations | Airtable (two-way sync), Shopify (webhooks) |
| Runtime | Docker Compose locally; managed cloud in production |

## Scale of the system

- **~34 API route modules** spanning workforce, inventory, production, and integrations
- **~40 backend service modules**, including a full Airtable two-way sync layer
- **18 mirrored Airtable tables** (1,400+ fields) being migrated table-by-table into Postgres
- **A four-role hierarchy** with per-user capability overrides gating every surface

## The modules

The platform splits cleanly into two decoupled domains plus shared infrastructure. Each has its own deep-dive doc:

| Area | What it covers | Doc |
|---|---|---|
| Architecture | How the whole thing fits together, request/auth/sync flow | [docs/architecture.md](./docs/architecture.md) |
| Workforce | Scheduling, kiosk clock-in, punches, timesheets, discipline | [docs/workforce-modules.md](./docs/workforce-modules.md) |
| Inventory & production | Master items, recipes/BOM, production, purchase orders, replenishment | [docs/inventory-modules.md](./docs/inventory-modules.md) |
| Airtable migration | The two-way sync and the safe, table-by-table cut-over to Postgres | [docs/airtable-migration.md](./docs/airtable-migration.md) |
| Engineering decisions | The tradeoffs I made and why (cost, latency, safety) | [docs/decisions.md](./docs/decisions.md) |

## Highlights

A few things I'm particularly happy with:

- **A safe migration off Airtable.** Instead of a risky big-bang switch, I built per-table "authoritative" flags so each table can move to Postgres independently while the platform keeps pushing changes back to Airtable during a parallel-run period. ([details](./docs/airtable-migration.md))
- **A per-field audit log.** Every cell edit in the inventory grid records the old and new value, who changed it, and whether it came from a human or a sync — which finally answered the recurring question of "why did this number change?". ([details](./docs/inventory-modules.md))
- **Hands-off timesheet dispatch.** A scheduler builds CSV timesheets and emails them to the accountant twice a month, and is idempotent so a server restart never double-sends. ([details](./docs/workforce-modules.md))
- **A PIN kiosk that's actually secure.** Deterministic hashed lookup, bcrypt verification, lockout after repeated failures, and device tagging — no passwords on the shop floor. ([details](./docs/workforce-modules.md))

## Screenshots

Screenshots and diagrams live in [`assets/`](./assets/). (Add exports here — see `assets/README.md`.)

---

## Notice — look, don't run

This repo exists to **describe** the work, not to distribute it.

- No application source code, environment files, credentials, API keys, or business data are included.
- Nothing here is deployable, and that is intentional.
- The platform, its code, and its design are proprietary. See [`LICENSE`](./LICENSE) — **all rights reserved**.

If you're a recruiter or hiring manager and want a closer look, I'm happy to walk through the live system, share a sanitized code sample, or give a screen-recorded demo on request.
