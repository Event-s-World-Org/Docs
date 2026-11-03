# ADR-0002: PostgreSQL Hosted on Supabase, One Schema per Module

* **Status:** Accepted
* **Date:** 26 Oct 2026

## Context

PostgreSQL is the primary database (NFR-03.1), and Supabase Auth already manages identity (FR-01.2). Every Supabase project includes a PostgreSQL database in which Auth stores its users (`auth.users`). The modular monolith (ADR-0001) needs each module's data to be isolated.

## Decision

1. **Hosting:** the application database is the PostgreSQL instance of the same Supabase project used for Auth.
2. **One schema per module:** `accounts`, `events`, `registrations`, `payments`, `files`, `notifications`, `audit` and `legal`.
   * The `public` schema stays empty.
   * Supabase-managed schemas (`auth`, `storage`, and others) are never modified by our migrations.
3. **No automatic Data API on our data:**
   * Our schemas are **not** added to the "Exposed schemas" of Supabase's Data API (PostgREST).
   * All privileges on them are revoked from the `anon` and `authenticated` roles.
   * Only our backend reads or writes application data.
4. **Database roles:**
   * `ew_migrator` owns the schemas and runs migrations.
   * `ew_app` is used by the API: data access only (DML), no DDL, and no `UPDATE` or `DELETE` on the ledger and audit tables.
5. **Connections:**
   * The API connects through Supabase's connection pooler in transaction mode, with Npgsql configured for pooling (`No Reset On Close=true`, no automatic prepared statements).
   * Migrations use a direct or session-mode connection.
6. **Migrations:** EF Core, with one `DbContext` per module and a migrations history table inside each module's schema. Triggers, grants and invariants that EF Core cannot express are written as raw SQL inside migrations.
7. **Cross-module foreign keys are allowed.** Integrity is the top priority and there is a single database. Code must still never query or write another module's tables (ADR-0001). If a module is extracted in the future, its foreign keys are dropped.

## Consequences

* ✅ There is one platform for Auth and data, with a free tier for the MVP and managed backups.
* ✅ Users can be referenced directly through `auth.users(id)`.
* ✅ Schemas make module ownership visible and allow per-module grants.
* ⚠️ We depend on Supabase as a provider. This is mitigated because it is standard PostgreSQL: the database can be moved with `pg_dump`.
* ⚠️ The plan must be checked before launch: limits, inactivity pausing on free projects, backup retention, and point-in-time recovery as an add-on.
* ⚠️ Exposing a schema in the Data API by mistake would publish data. A deployment checklist item and a CI test must verify the `anon` and `authenticated` grants.

## Alternatives Considered

| Alternative | Why it was not chosen |
|---|---|
| Separate PostgreSQL provider (Neon, Azure, AWS RDS) | Two providers to pay for and operate, and users could only be referenced by ID without a foreign key. |
| A single schema for everything | Module ownership would not be visible and per-module permissions would be impossible. |
| Supabase Data API with RLS as the main access path | Business rules (ledger, capacity) belong in the .NET domain (ADR-0001), not in table policies. |
