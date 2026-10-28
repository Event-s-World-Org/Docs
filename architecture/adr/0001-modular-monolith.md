# ADR-0001: Modular Monolith as the Architectural Style

* **Status:** Accepted
* **Date:** 26 Oct 2026

## Context

Event's World needs a backend architecture that a small team can build quickly and evolve for a long time. The requirements (v0.3) define these priorities:

1. **Data integrity.** The ledger is immutable and double-entry, capacity must never be oversold, and approving a payment must update the registration and the ledger together (NFR-03, FR-03.13).
2. **Simplicity and speed of delivery.** A small team of friends wants to ship the MVP fast.
3. **Evolvability.** Much is already planned after the MVP: co-administrators, teams, refunds, payment plans, platform fees and online events.
4. Security and privacy (vouchers, personal data, personal links).
5. Scalability for registration peaks (hundreds of people at once, not millions).
6. Low operating cost.

The stack is fixed: ASP.NET Core (.NET 10) with OOP and SOLID (NFR-02), PostgreSQL with EF Core (NFR-02.4), Angular 21 (NFR-01.1), Supabase Auth (FR-01.2) and Cloudflare R2 (NFR-01.3).

## Decision

The backend is a **modular monolith**: one ASP.NET Core application and one deployment, split into business modules with strict boundaries.

* **Modules:** Accounts, Events, Registrations, Payments & Ledger, Files, Notifications, Audit and Legal. Each module owns its own PostgreSQL schema (see `database.md`).
* **Inside each module:** layers in the Clean Architecture style: `Api → Application → Domain ← Infrastructure`. The domain has no dependency on frameworks. Controllers, services and repositories (NFR-02.3) live inside each module.
* **Communication between modules:**
  * Calls go through each module's public interface (a `Contracts` project).
  * A module never reads or writes another module's tables.
  * Side effects use in-process domain events. Anything that leaves the process, such as emails, goes through a transactional **outbox** table; there is no message broker in the MVP.
* **Atomicity:** operations that must be atomic across modules (approving a payment updates the registration and writes the ledger) share one database transaction. The application layer coordinates it as a unit of work over a single connection.
* **Scaling:** the API is stateless, so several instances can run behind a load balancer. Background work (outbox, scheduled jobs) runs as hosted services.

## Consequences

* ✅ ACID transactions stay simple because there is one database. This protects the ledger and capacity rules.
* ✅ There is one deployment and one codebase, a good fit for a small team and low cost.
* ✅ New features (refunds, co-admin, payment plans) fit inside existing modules or new ones without reworking the others.
* ✅ A module can be extracted into its own service later, because its boundaries and data are already isolated.
* ⚠️ Boundaries must be protected with discipline. We will enforce them with architecture tests (for example, NetArchTest) that fail the build when a module references another module's internals.
* ⚠️ All modules scale together. This is acceptable for the expected load.

## Alternatives Considered

| Alternative | Why it was not chosen |
|---|---|
| Layered monolith (single Controllers/Services/Repositories) | Simplest at first, but everything ends up depending on everything; post-MVP features would touch many places. |
| Microservices | High operational cost for a small team. Payment approval, ledger and capacity checks would become distributed transactions. |
| Serverless / Supabase-only (RLS + functions) | Contradicts the .NET/OOP requirements (NFR-02) and would scatter ledger logic across SQL and functions. |
| Event-driven with a message broker | Adds infrastructure and eventual consistency everywhere. We keep only in-process events plus an outbox. |
