# Architecture

Architecture documentation for Event's World. The requirements it implements are in [requirements.md](../requirements.md).

## Documents

| Document | Content | Status |
|---|---|---|
| [database.md](database.md) | Schemas per module, data model, ledger invariants, concurrency, jobs and migrations | v0.1 |
| [backend.md](backend.md) | .NET solution layout, modules and contracts, request pipeline, API conventions, cross-cutting concerns, jobs and testing | v0.1 |
| [frontend.md](frontend.md) | Routes and render modes, layouts, folder structure, state, auth in the browser, forms, theming, testing | v0.1 |
| [../design/design-system.md](../design/design-system.md) | Final visual system Salar Cristal: tokens, glass levels, backdrop, typography, accessibility, performance, `styles.css` | v1.0 |

## Architecture Decision Records (ADR)

An ADR records one important decision: its context, the decision and its consequences. The content of an accepted ADR is never edited. When a later ADR replaces all or part of it, only its status line changes to point to the new one.

| ADR | Decision | Status |
|---|---|---|
| [0001](adr/0001-modular-monolith.md) | Modular monolith as the architectural style | Accepted |
| [0002](adr/0002-postgresql-on-supabase.md) | PostgreSQL hosted on Supabase, one schema per module | Accepted |
| [0003](adr/0003-backend-module-structure.md) | Backend module structure and in-process communication | Accepted |
| [0004](adr/0004-authentication-and-authorization.md) | Authentication and two-level authorization | Accepted |
| [0005](adr/0005-resend-email-provider.md) | Resend as the email provider | Accepted |
| [0006](adr/0006-hybrid-rendering.md) | Hybrid rendering: SSR for public pages, CSR for panels | Accepted |
| [0007](adr/0007-styling-and-components.md) | Tailwind CSS + Angular CDK + own component library (replaces Angular Material) | Accepted; decision 6 superseded by 0008 |
| [0008](adr/0008-salar-cristal-design-system.md) | Salar Cristal as the final visual design system (intense glass, Outfit, soft corners) | Accepted |

## Open Decisions

| # | Decision | Where |
|---|---|---|
| D-02 | Hosting for the API (always on) and the frontend SSR server (Node.js) | backend.md §8, frontend.md §12 |
