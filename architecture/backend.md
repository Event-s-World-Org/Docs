# Backend Architecture

* **Version:** 0.1 · **Date:** 26 Oct 2026
* **Decisions:** [ADR-0001](adr/0001-modular-monolith.md) (modular monolith), [ADR-0003](adr/0003-backend-module-structure.md) (module structure) and [ADR-0004](adr/0004-authentication-and-authorization.md) (authentication and authorization).
* **Related:** [database.md](database.md), [requirements.md](../requirements.md) v0.3.
* **Stack:** ASP.NET Core Web API on **.NET 10**, C#, EF Core with Npgsql (NFR-02).

---

## 1. Solution Layout

```
Event-s-World-Backend/
├─ EventsWorld.slnx
├─ src/
│  ├─ Bootstrapper/
│  │  └─ EventsWorld.Api/                 ← host: Program.cs, middleware, module registration, hosted services
│  ├─ Shared/
│  │  ├─ EventsWorld.Shared.Kernel/        ← Entity, AggregateRoot, DomainEvent, Result/Error, Money, typed IDs
│  │  └─ EventsWorld.Shared.Infrastructure/← UnitOfWork, outbox and audit writers, EF conventions, auth, R2 client
│  └─ Modules/
│     ├─ Accounts/
│     │  ├─ EventsWorld.Modules.Accounts/            ← Domain/ Application/ Infrastructure/ Api/  (internal)
│     │  └─ EventsWorld.Modules.Accounts.Contracts/  ← public interfaces, DTOs, integration events
│     ├─ Events/            (same two projects)
│     ├─ Registrations/
│     ├─ Payments/
│     ├─ Files/
│     ├─ Notifications/
│     ├─ Audit/
│     └─ Legal/
└─ tests/
   ├─ EventsWorld.ArchitectureTests/       ← module boundaries and layering (NetArchTest)
   ├─ EventsWorld.UnitTests/               ← domain rules (ledger, capacity, state machines)
   └─ EventsWorld.IntegrationTests/        ← API + real PostgreSQL (Testcontainers)
```

**Inside a module project:**

```
EventsWorld.Modules.Payments/
├─ Domain/          PaymentSubmission, LedgerTransaction, LedgerEntry, Money rules, domain events
├─ Application/     ApprovePaymentHandler, ObservePaymentHandler, RevertApprovalHandler, queries
├─ Infrastructure/  PaymentsDbContext (schema "payments"), repositories, migrations, SQL for triggers
├─ Api/             PaymentsController (route group /api/v1/events/{eventId}/payments)
└─ PaymentsModule.cs  AddPaymentsModule(services, config): registers everything above
```

Dependency rule (checked by architecture tests): `Api → Application → Domain ← Infrastructure`. `Domain` knows nothing about EF Core, ASP.NET Core or other modules.

---

## 2. Modules

| Module | Responsibility | Public contract (examples) | Main screens |
|---|---|---|---|
| **Accounts** | Profiles, global role, suspension, account linking | `IAccountsModule.GetUser`, `IsSuspended` | AUT-01..04, ORG-12, ADM-03 |
| **Events** | Event lifecycle, form builder, payment QR, members | `IEventsModule.GetEventForRegistration`, `GetMemberRole` | ORG-01..04, ORG-11, PUB-01/02 |
| **Registrations** | Registrations, capacity, personal links, data edits | `IRegistrationsModule.MarkPaymentApproved`, `ReleaseCapacity`, `HasRegistrations` | PUB-03..06, PUB-08, ORG-06/07, ORG-09 |
| **Payments** | Voucher review, in-person payments, ledger | `IPaymentsModule.GetEventBalance`, `GetRegistrationBalance` | ORG-05, ORG-07, ORG-08, ORG-09 |
| **Files** | R2 uploads, signed URLs, retention | `IFilesModule.Upload`, `GetSignedUrl` | Covers, QR, vouchers |
| **Notifications** | Email templates, outbox worker | `INotificationsModule.Enqueue` (writes to outbox) | SYS-02 |
| **Audit** | Append-only audit log, admin queries | `IAuditLog.Record` | ADM-04 |
| **Legal** | Document versions, consents | `ILegalModule.RecordConsent`, `GetCurrentVersions` | PUB-07 |

The platform panel (ADM-01..04) is not a module: its endpoints live in each owning module under `/api/v1/admin/...`, protected by the `SuperAdmin` policy.

---

## 3. Request Pipeline

```
HTTP request
 → Exception handler (→ RFC 9457 ProblemDetails)
 → Correlation ID + structured logging scope
 → Rate limiter (per IP, NFR-04.2)
 → Authentication (Supabase JWT | Participant token)            ADR-0004
 → Authorization (global policy | [RequireEventRole])           ADR-0004
 → Controller (Api/)               maps HTTP ⇄ command/query
 → Input validation (FluentValidation)
 → Handler (Application/)          loads aggregates, calls domain, calls other modules' Contracts
 → Domain (Domain/)                enforces business rules, raises domain events
 → UnitOfWork.Commit              one transaction: module DbContexts + outbox + audit
 → Response DTO
```

### Example: approving a payment (FR-03.5, database.md §4.2)

```csharp
internal sealed class ApprovePaymentHandler(
    IPaymentSubmissionRepository submissions,
    ILedger ledger,
    IRegistrationsModule registrations,     // Registrations.Contracts
    INotificationsModule notifications,     // writes to the outbox
    IAuditLog audit,
    ICurrentUser currentUser,
    IUnitOfWork unitOfWork,
    TimeProvider clock) : ICommandHandler<ApprovePayment, Result>
{
    public async Task<Result> Handle(ApprovePayment cmd, CancellationToken ct)
    {
        var submission = await submissions.GetForUpdateAsync(cmd.SubmissionId, ct); // SELECT ... FOR UPDATE
        if (submission is null) return PaymentErrors.NotFound;

        var approval = submission.Approve(currentUser.Id, clock.GetUtcNow());       // domain rule: only pending/observed
        if (approval.IsFailure) return approval.Error;

        ledger.Record(LedgerTransaction.PaymentApproved(submission, currentUser.Id)); // balanced entries
        await registrations.MarkPaymentApproved(submission.RegistrationId, ct);      // same transaction
        notifications.Enqueue(new PaymentApprovedEmail(submission.RegistrationId));
        audit.Record(AuditAction.PaymentApproved, submission, cmd.Reason);

        await unitOfWork.CommitAsync(ct);                                            // all or nothing
        return Result.Success();
    }
}
```

---

## 4. API Conventions

| Topic | Convention |
|---|---|
| Style | REST over JSON, versioned under `/api/v1`. |
| Route groups | `/public/...` (anonymous), `/participant/...` (participant token), `/me/...` (current user), `/events/{eventId}/...` (event roles), `/admin/...` (Super Admin). |
| Errors | RFC 9457 ProblemDetails with a stable `code` (e.g. `registrations.capacity_full`) that the frontend translates to Spanish messages. |
| Validation | FluentValidation in the API layer for shape. Business rules live in the domain and return `Result`. |
| Pagination | `?page=&pageSize=` (max 100), with a `total` count. |
| Concurrency | Mutable resources return an `ETag` (from `xmin`). Updates send `If-Match`; a conflict returns 412. |
| Idempotency | `POST /public/events/{slug}/registrations` and in-person registration accept an `Idempotency-Key` header. A retry on a bad mobile connection returns the same registration instead of creating a duplicate (stored as `registrations.idempotency_key`). |
| Dates and money | ISO 8601 UTC timestamps plus the event's `timeZone`. Money is `{ "amount": "120.00", "currency": "BOB" }`, with the amount as a string. |
| Documentation | OpenAPI document (`Microsoft.AspNetCore.OpenApi`) with the Scalar UI in development. |

### Main endpoints per screen

| Screen | Endpoint |
|---|---|
| PUB-01 / PUB-02 | `GET /public/events?…`, `GET /public/events/{slug}` |
| PUB-03 | `POST /public/events/{slug}/registrations` (multipart: answers + voucher) |
| PUB-05 / PUB-06 | `POST /participant/sessions`, `GET /participant/registration`, `POST /participant/registration/vouchers`, `POST /public/registration-links` |
| PUB-08 | `GET /me/registrations` |
| ORG-01..04 | `GET /me/events`, `POST /events`, `PUT /events/{id}`, `PUT /events/{id}/payment-settings`, `PUT /events/{id}/form` |
| ORG-05 | `GET /events/{id}/dashboard` |
| ORG-06 / ORG-07 | `GET /events/{id}/registrations`, `GET /events/{id}/registrations/export.csv`, `PATCH /events/{id}/registrations/{rid}` |
| ORG-08 | `GET /events/{id}/payments?status=`, `POST /events/{id}/payments/{sid}/approve`, `…/observe`, `…/reject`, `…/revert` |
| ORG-09 | `POST /events/{id}/registrations/in-person` |
| ORG-11 | `POST /events/{id}/publish`, `…/close-registrations`, `…/reopen-registrations` |
| ADM-01..04 | `GET /admin/overview`, `GET /admin/events`, `POST /admin/events/{id}/suspend`, `GET /admin/users`, `POST /admin/users/{id}/suspend`, `GET /admin/audit` |

---

## 5. Cross-Cutting Concerns

| Concern | Implementation |
|---|---|
| **Rate limiting** (NFR-04.2) | ASP.NET Core's built-in rate limiter per IP on anonymous route groups. Per-email limits (registration, link recovery) are checked in the use case against PostgreSQL, so they hold across several API instances. |
| **Files** (NFR-01.3) | `Files` module with `AWSSDK.S3` against Cloudflare R2. Uploads are streamed through the API with a 5 MB request limit. The type is checked by magic bytes, then the SHA-256 is computed and the file stored with a random key. Reads use signed URLs valid for 5 minutes. |
| **Emails** | `Notifications` module: Razor or Scriban templates in Spanish, sent by the outbox worker through **Resend** behind an `IEmailSender` interface ([ADR-0005](adr/0005-resend-email-provider.md)). Supabase Auth emails (verification, password reset) use Resend's SMTP through Supabase's custom SMTP setting. |
| **Time** | `TimeProvider` is injected everywhere (testable). Local times are computed with the event's IANA time zone. |
| **Logging and tracing** | Serilog structured logs to the console, plus OpenTelemetry traces and metrics, ready to send to a hosted backend later (NFR-06, `[Next]`). Personal data and tokens are never logged. |
| **Health** | `/health/live` and `/health/ready` (PostgreSQL and R2 checks). |
| **Configuration and secrets** | `appsettings.json` holds non-secret defaults only. Connection strings, R2 keys, the participant signing key and the email API key come from environment variables, and from `dotnet user-secrets` in development. |
| **CORS** | Only the frontend origins (local development, staging, production). |
| **Security headers** | HSTS, `X-Content-Type-Options`, `Referrer-Policy: no-referrer` (NFR-04.1). |

---

## 6. Background Jobs

The jobs are `BackgroundService` classes in the host, each driven by a `PeriodicTimer` and guarded by a PostgreSQL advisory lock so that only one API instance runs it at a time. The schedule is in database.md §4.4: close registrations, final rejection after the deadline, expired QR, finish events, outbox and file retention.

* **Correctness never depends on a job.** Every use case checks deadlines and states itself; jobs only apply visible state changes and side effects.
* **Hosting requirement:** the API must be always on. If the chosen hosting sleeps when idle, the same jobs are exposed as a protected internal endpoint, called by an external cron (Supabase `pg_cron` or GitHub Actions).
* If a dashboard or manual retries become necessary, the jobs can move to Hangfire, which uses PostgreSQL storage.

---

## 7. Testing and Quality

| Level | Tooling | What it covers |
|---|---|---|
| Unit | xUnit | Domain rules: ledger balancing, reversals, capacity, state machines, editing locks (FR-02.5) |
| Integration | xUnit + `WebApplicationFactory` + Testcontainers (PostgreSQL) | Endpoints end to end, migrations, ledger triggers, concurrency (two approvals, last capacity spot) |
| Architecture | NetArchTest | Module boundaries, layering, anonymous endpoints list (ADR-0003, ADR-0004) |
| CI | GitHub Actions | Build, tests and the formatting check on every PR to `main` (conventions.md) |

### Main packages

* `Npgsql.EntityFrameworkCore.PostgreSQL`
* `EFCore.NamingConventions`
* `Microsoft.AspNetCore.Authentication.JwtBearer`
* `Microsoft.AspNetCore.OpenApi` and `Scalar.AspNetCore`
* `FluentValidation`
* `AWSSDK.S3`
* `Serilog.AspNetCore`
* `OpenTelemetry.Extensions.Hosting`
* Tests: `xunit`, `Testcontainers.PostgreSql`, `NetArchTest.Rules`

All of them have free (open-source) licenses.

---

## 8. Open Decisions

| # | Decision | Options | Recommendation |
|---|---|---|---|
| D-01 | Email provider | — | ✅ **Resend** ([ADR-0005](adr/0005-resend-email-provider.md)). |
| D-02 | Hosting for the API and the frontend's SSR server | Decided together with deployment | The API must be always on (§6), or use the external-cron fallback. The frontend needs a Node.js runtime for SSR (frontend.md §2). |

---

## 9. Starting From the Current Repository

The repository currently holds the ASP.NET Core template (`Event-s-World-Backend.csproj`, `Program.cs`, `WeatherForecast`). Implementation will:

1. Create `EventsWorld.slnx` with the layout of §1.
2. Turn the template into `EventsWorld.Api` (the namespace changes from `Event_s_World_Backend` to `EventsWorld.Api`).
3. Remove `WeatherForecast`.

These changes go through a feature branch and PR, following conventions.md.
