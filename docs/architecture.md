# Architecture

A high-level map of how the platform is put together. No source code is included — this describes the shape of the system.

## The big picture

```
                 ┌─────────────────────────────────────────┐
                 │            Next.js 14 (App Router)        │
   Managers ───▶ │  /admin/*   schedule, inventory, users…   │
   Staff    ───▶ │  /me/*      schedule, hours, corrections  │
   Kiosk    ───▶ │  /kiosk/*   PIN-only clock-in (public)    │
                 └───────────────────┬───────────────────────┘
                                     │  JSON over HTTPS
                 ┌───────────────────▼───────────────────────┐
                 │              FastAPI (Python 3.12)         │
                 │  ~34 route modules, capability-gated       │
                 │  auth · workforce · inventory · webhooks   │
                 └───┬──────────────┬───────────────┬─────────┘
                     │              │               │
            ┌────────▼──┐   ┌───────▼──────┐  ┌─────▼─────────┐
            │ PostgreSQL │   │ Background    │  │ Integrations  │
            │ (system of │   │ schedulers    │  │ Airtable ⇄    │
            │  record)   │   │ (asyncio)     │  │ Shopify →     │
            └────────────┘   └──────────────┘  └───────────────┘
```

## Backend layout

The FastAPI app follows a conventional, domain-organized layout:

- **`api/`** — route modules (one file per feature area) plus shared auth dependencies. Routes are thin; they validate input, check capabilities, and delegate to services.
- **`core/`** — configuration and JWT/auth helpers.
- **`db/`** — SQLAlchemy session management.
- **`models/`** — ORM models.
- **`schemas/`** — Pydantic request/response schemas.
- **`services/`** — where the real work happens: Google OAuth, email, the full Airtable client and sync layer, webhook dispatch, kiosk auth, punch aggregation, production, replenishment, and the background schedulers.

Roughly **34 route modules** are registered on a single FastAPI app, and around **40 service modules** sit behind them. Schema changes are managed with Alembic migrations.

## Frontend layout

The Next.js App Router app is split by audience:

- **`/admin/*`** — manager surfaces: schedule, integrations, inventory dashboards, users, per-staff pages, roles, kiosk activity, punch corrections, timesheets, QR codes.
- **`/me/*`** — staff surfaces: read-only schedule and a live hours page with correction requests.
- **`/kiosk/*`** — a public, PIN-only clock-in screen.
- **`/login`** — Google OAuth entry.

Shared components include capability gates, auth gates, a header with a live Airtable sync indicator, dashboard cards, data grids, calendars, and drawers.

## Auth and authorization

- **Sign-in** is Google SSO restricted to the company domain. The first bootstrap email is auto-promoted to super admin on first sign-in.
- **External users** (contractors, freelancers on any domain) join through an invite-only flow.
- **Authorization** is a four-role hierarchy with per-user capability overrides. The same capability checks gate both API routes and frontend navigation, so a user never sees a surface they can't use.

## Background work

Several jobs run as in-process asyncio loops inside the API's lifespan rather than as separate worker infrastructure:

- **Airtable sync** keeps the mirrored tables current (see [airtable-migration.md](./airtable-migration.md)).
- **Timesheet dispatch** emails pay-period CSVs to the accountant on a fixed cadence.
- **Reconciliation and sales schedulers** run periodic maintenance.

Each loop is independently error-isolated: a failure in one tick is logged and the next tick still runs. The decision to keep these in-process rather than reach for Celery/Redis workers is covered in [decisions.md](./decisions.md).

## Two decoupled domains

The workforce side (scheduling, punches, timesheets, discipline) and the inventory/operations side (Airtable, Shopify, production, purchasing) are architecturally independent — there are no cross-domain imports between them. That separation is what made it possible to later carve the workforce half out into its own standalone repository without untangling a web of dependencies.
