# ADR-0003: Backend Module Structure and In-Process Communication

* **Status:** Accepted
* **Date:** 26 Oct 2026

## Context

ADR-0001 chose a modular monolith. We now need a concrete .NET solution layout that:

* makes module boundaries hard to break by accident;
* keeps the Clean Architecture layers (Api → Application → Domain ← Infrastructure) inside each module, satisfying NFR-02.2 and NFR-02.3;
* allows atomic operations across modules, such as approving a payment (database.md §4.2);
* stays small enough for a small team: we want to avoid 5 projects × 8 modules = 40 projects.

## Decision

1. **Two projects per module:**
   * `EventsWorld.Modules.<Module>` contains the module's layers as folders (`Domain/`, `Application/`, `Infrastructure/`, `Api/`). Every class is `internal` unless it must be public.
   * `EventsWorld.Modules.<Module>.Contracts` is the only public surface: interfaces the module offers to other modules, DTOs and integration events.
   * A module may reference other modules' `Contracts` projects, never their implementation projects. The compiler enforces this, because `internal` types are invisible outside the project.
2. **Shared projects:**
   * `EventsWorld.Shared.Kernel`: base types with no infrastructure (Entity, AggregateRoot, DomainEvent, Result/Error, Money, strongly typed IDs).
   * `EventsWorld.Shared.Infrastructure`: cross-cutting infrastructure (unit of work, outbox writer, audit writer, EF Core conventions, authentication helpers).
3. **One host:** `EventsWorld.Api` is the composition root. Each module exposes an `Add<Module>Module(IServiceCollection, IConfiguration)` extension, registered with the native DI container (NFR-02.3).
4. **Use cases, not a mediator library:**
   * Each use case is one Application class (for example `ApprovePaymentHandler`) behind a small in-house `ICommandHandler<TCommand, TResult>` / `IQueryHandler<TQuery, TResult>` interface.
   * We do **not** use MediatR or AutoMapper: both moved to commercial licensing in 2025, and we only need a fraction of what they offer. Mapping is explicit code.
5. **Communication between modules:**
   * **Synchronous queries and commands** go through `Contracts` interfaces, for example `IRegistrationsModule.MarkPaymentApproved(...)`.
   * **Domain events** stay inside a module and are dispatched before commit.
   * **Effects outside the process** (emails) always go through the outbox (database.md §3.6).
6. **One transaction for cross-module writes:** a scoped `IUnitOfWork` opens one PostgreSQL connection and transaction per request. Every module `DbContext` in that request enlists in it (`Database.UseTransaction`). Commit happens once, at the end of the use case.
7. **Boundaries are tested:** an architecture test project (NetArchTest) fails the build if:
   * a module references another module's implementation;
   * `Domain` depends on `Infrastructure` or ASP.NET Core;
   * a controller calls a repository directly.

## Consequences

* ✅ 16 module projects plus 3 shared ones: manageable, with boundaries enforced by the compiler and by tests.
* ✅ Layering stays visible inside each module through folders and architecture tests.
* ✅ Atomic cross-module operations remain simple because there is one database (ADR-0002).
* ✅ No commercial licenses are required.
* ⚠️ Layers inside a module are folders, not projects, so the compiler cannot stop `Domain` from referencing `Infrastructure`. The architecture tests cover this.
* ⚠️ A shared transaction couples modules at runtime. If a module is extracted later, its cross-module operations must switch to the outbox and eventual consistency.

## Alternatives Considered

| Alternative | Why it was not chosen |
|---|---|
| One project per layer per module (~40 projects) | Too heavy for a small team; slow builds and noisy solution. |
| Single project with folders per module | Boundaries rely only on discipline; nothing stops a module from using another module's classes. |
| MediatR + AutoMapper | Commercial licensing since 2025, and unnecessary indirection for our size. |
| Cross-module communication only through events | Would force eventual consistency on payment approval and capacity, against our first priority (integrity). |
