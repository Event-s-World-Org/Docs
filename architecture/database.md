# Database Design

* **Version:** 0.1 · **Date:** 26 Oct 2026
* **Decisions:** [ADR-0001](adr/0001-modular-monolith.md) (modular monolith) and [ADR-0002](adr/0002-postgresql-on-supabase.md) (PostgreSQL on Supabase, one schema per module).
* **Requirements:** [requirements.md](../requirements.md) v0.3.

---

## 1. Conventions

| Topic | Convention |
|---|---|
| Naming | `snake_case` tables and columns, plural table names (EF Core with `EFCore.NamingConventions`). |
| Primary keys | `uuid`, **UUID v7** (time-ordered), generated in .NET with `Guid.CreateVersion7()`. |
| Time | `timestamptz` (UTC). Each event stores its IANA time zone (`America/La_Paz`) to display local times (NFR-05.1). Plain dates use `date`. |
| Money | `numeric(12,2)` plus a `char(3)` ISO 4217 currency. Never floating point (NFR-05.2). |
| Enumerations | `text` with a `CHECK` constraint: readable and easy to extend in migrations. |
| Email | `citext` (case-insensitive comparison). |
| Dynamic data | `jsonb` for registration answers and field options (NFR-03.1). |
| Optimistic concurrency | PostgreSQL's `xmin` column as the EF Core concurrency token on mutable aggregates. |
| Audit columns | `created_at` and `updated_at` on mutable tables. Immutable tables only have `created_at` or `occurred_at`. |
| Deletion | No generic soft delete. State columns express the business lifecycle (statuses, `revoked_at`, `deleted_at` for files). |

---

## 2. Schemas and Ownership

| Schema | Module | Owns |
|---|---|---|
| `accounts` | Accounts | User profiles, global roles, suspension |
| `events` | Events | Events, organizers, payment QR, registration form, status history |
| `registrations` | Registrations | Registrations, answers, capacity counters, personal-link tokens, data changes |
| `payments` | Payments & Ledger | Payment submissions (vouchers and in-person payments) and the ledger |
| `files` | Files | Metadata of every object stored in Cloudflare R2 |
| `notifications` | Notifications | Outbox and email log |
| `audit` | Audit | Append-only audit log |
| `legal` | Legal | Terms/privacy versions and consents |
| `auth` | *(Supabase)* | Identities and credentials; read-only for us |

```mermaid
flowchart LR
  auth[(auth · Supabase)] --> accounts
  accounts --> events
  events --> registrations
  registrations --> payments
  events --> payments
  files -.used by.-> events & payments
  registrations & payments & events -.write.-> notifications & audit
  legal -.consents.-> accounts & registrations
```

---

## 3. Data Model per Module

### 3.1 `accounts`

```mermaid
erDiagram
  AUTH_USERS ||--|| USERS : "same id"
  USERS {
    uuid id PK "= auth.users.id"
    text first_names
    text last_names
    citext email "copy of auth email"
    date birth_date "18+ (FR-01.1)"
    text global_role "user | super_admin"
    text status "active | suspended"
    text suspension_reason
    text theme "light | dark | system"
    timestamptz created_at
    timestamptz updated_at
  }
```

* `id` references `auth.users(id)` with `ON DELETE CASCADE`. Deleting an account is blocked while the user owns events, because `events.events.owner_user_id` uses `ON DELETE RESTRICT` (FR-01.7).
* The email is copied from Supabase. The backend refreshes it after an email change (FR-01.7).

### 3.2 `events`

```mermaid
erDiagram
  EVENTS ||--o{ EVENT_MEMBERS : has
  EVENTS ||--o{ PAYMENT_QRS : has
  EVENTS ||--o{ FORM_FIELDS : defines
  EVENTS ||--o{ STATUS_HISTORY : logs
  CATEGORIES ||--o{ EVENTS : classifies
  EVENTS {
    uuid id PK
    uuid owner_user_id FK "accounts.users"
    text slug UK "/e/:slug"
    text title
    text description
    uuid cover_file_id FK "files.stored_files"
    uuid category_id FK
    char country_code "ISO 3166-1"
    text city
    text venue
    text time_zone "IANA"
    timestamptz starts_at
    timestamptz ends_at
    timestamptz registration_deadline
    int capacity
    smallint min_age "nullable"
    text organizer_contact "WhatsApp or email"
    text modality "in_person (online/hybrid: Next)"
    boolean is_free
    numeric price "nullable if free"
    char currency "nullable if free"
    text status "draft | published | registrations_closed | finished | suspended | cancelled"
    text registrations_paused_reason "null | qr_expired"
    timestamptz published_at
    timestamptz created_at
    timestamptz updated_at
  }
  EVENT_MEMBERS {
    uuid event_id PK, FK
    uuid user_id PK, FK "accounts.users"
    text role "organizer | co_admin (Next)"
    timestamptz created_at
  }
  PAYMENT_QRS {
    uuid id PK
    uuid event_id FK
    uuid file_id FK "private bucket"
    numeric amount "0 = open amount"
    date expires_on
    boolean is_current
    timestamptz created_at
  }
  FORM_FIELDS {
    uuid id PK
    uuid event_id FK
    text key "stable, used in answers"
    text label
    text type "8 types + system email / voucher"
    boolean is_required
    boolean is_system "locked fields"
    jsonb options "choices"
    int position
    boolean is_hidden
  }
  STATUS_HISTORY {
    uuid id PK
    uuid event_id FK
    text from_status
    text to_status
    text reason
    uuid actor_user_id
    timestamptz occurred_at
  }
  CATEGORIES {
    uuid id PK
    text name UK
  }
```

Rules enforced by the database:

* `CHECK (ends_at > starts_at AND registration_deadline <= starts_at AND capacity > 0)`.
* `CHECK (is_free OR (price > 0 AND currency IS NOT NULL))`.
* A unique partial index on `payment_qrs (event_id) WHERE is_current` allows one current QR per event. Replacing a QR keeps the old row for history.
* `UNIQUE (event_id, key)` on `form_fields`. System fields (`email`, `voucher`) have `is_system = true` and cannot be removed.
* The creator is also inserted in `event_members` with role `organizer`. Authorization always checks `event_members` (two-level RBAC, §2.1 of the requirements).

The locked fields after the first registration (FR-02.5) are enforced in the domain layer of the Events module. It asks the Registrations module whether the event has registrations.

### 3.3 `registrations`

```mermaid
erDiagram
  REGISTRATIONS ||--o{ ACCESS_TOKENS : "opened by"
  REGISTRATIONS ||--o{ DATA_CHANGES : "edited in"
  EVENT_CAPACITY ||--o{ REGISTRATIONS : limits
  REGISTRATIONS {
    uuid id PK
    uuid event_id FK "events.events"
    text code UK "EW-7F3K2"
    citext email
    timestamptz email_verified_at "first link open"
    uuid user_id FK "nullable, linked account"
    text full_name "denormalized for search"
    jsonb answers "field key -> value"
    text origin "online | in_person"
    text status "pending | in_observation | approved | rejected | confirmed"
    boolean counts_for_capacity
    uuid created_by_user_id "organizer, in-person"
    uuid idempotency_key UK "client retry key"
    timestamptz created_at
    timestamptz updated_at
  }
  EVENT_CAPACITY {
    uuid event_id PK "events.events"
    int capacity
    int taken "0 <= taken <= capacity"
  }
  ACCESS_TOKENS {
    uuid id PK
    uuid registration_id FK
    bytea token_hash UK "SHA-256 of a 256-bit token"
    timestamptz expires_at "event end + 30 days"
    timestamptz revoked_at
    timestamptz last_used_at
    timestamptz created_at
  }
  DATA_CHANGES {
    uuid id PK
    uuid registration_id FK
    uuid actor_user_id
    text reason
    jsonb changes "before / after"
    timestamptz occurred_at
  }
```

* `confirmed` is the status of registrations to free events.
* `counts_for_capacity` is true for `pending`, `in_observation`, `approved` and `confirmed`, and false for `rejected` (FR-03.13).
* Several registrations can share an email (FR-03.14), so `email` is indexed, not unique.
* `idempotency_key` (unique when present) stores the client's `Idempotency-Key` header. A retried submission on a bad mobile connection returns the existing registration instead of creating a duplicate (backend.md §4).
* Account linking (FR-05.3): `user_id` is set for every registration whose email equals the user's **verified** email. This happens on login and when a registration is created by a logged-in user.
* Indexes:
  * `(event_id, status)` and `(event_id, created_at)` for the participant list and dashboard;
  * `(email)` for link recovery and My registrations;
  * a GIN trigram index on `full_name` and `email` for search (`pg_trgm`).

### 3.4 `payments` (Payments & Ledger)

```mermaid
erDiagram
  PAYMENT_SUBMISSIONS ||--o| LEDGER_TRANSACTIONS : "approval creates"
  LEDGER_TRANSACTIONS ||--|{ LEDGER_ENTRIES : contains
  LEDGER_ACCOUNTS ||--o{ LEDGER_ENTRIES : receives
  LEDGER_TRANSACTIONS ||--o| LEDGER_TRANSACTIONS : "reversed by"
  PAYMENT_SUBMISSIONS {
    uuid id PK
    uuid registration_id FK
    int attempt_no "1 = first, 2+ = resubmission"
    text channel "online | in_person"
    text method "qr | cash"
    uuid voucher_file_id FK "online only"
    numeric expected_amount "QR amount or price"
    numeric cash_received "in-person cash"
    char currency
    text status "pending | in_observation | approved | rejected"
    text review_reason
    uuid reviewed_by_user_id
    timestamptz reviewed_at
    timestamptz submitted_at
  }
  LEDGER_ACCOUNTS {
    uuid id PK
    uuid event_id FK
    text type "organizer_cash | registration_revenue | refunds_payable (Next) | platform_fees (Future)"
    text method "qr | cash, only for organizer_cash"
    char currency
  }
  LEDGER_TRANSACTIONS {
    uuid id PK
    uuid event_id FK
    text type "payment_approved | reversal | refund_pending (Next) | refund_paid (Next)"
    uuid registration_id FK
    uuid payment_submission_id FK
    uuid reversed_transaction_id FK, UK
    text reason
    uuid actor_user_id
    timestamptz occurred_at
  }
  LEDGER_ENTRIES {
    uuid id PK
    uuid transaction_id FK
    uuid account_id FK
    uuid registration_id "dimension"
    numeric amount "signed, never 0"
    char currency
  }
```

**Ledger invariants (NFR-03), enforced in the database and not only in code:**

1. **Immutable:** a `BEFORE UPDATE OR DELETE` trigger on `ledger_transactions` and `ledger_entries` raises an error. In addition, `ew_app` has no `UPDATE` or `DELETE` grants on them.
2. **Balanced (double entry):** a `DEFERRABLE INITIALLY DEFERRED` constraint trigger checks at commit time that the entries of each transaction sum to 0.
3. **Consistent currency:** each entry's currency must match its account (trigger).
4. **One reversal per transaction:** `reversed_transaction_id` is unique.
5. **One approval per submission:** a unique partial index on `ledger_transactions (payment_submission_id) WHERE type = 'payment_approved'`.

**Balances (NFR-03.3)** are computed, never stored:

* The `payments.account_balances` view sums the entries of each account.
* "Real money" on the dashboard (FR-03.6) is the balance of the event's `organizer_cash` accounts.
* A registration's financial state is the sum of its entries (the `registration_id` dimension).
* If performance ever requires it, materialized views can be added.

**Example (FR-03.5, approving 120 BOB paid by QR):**

| Transaction | Account | Amount |
|---|---|---|
| `payment_approved` | organizer_cash (qr) | +120.00 |
| | registration_revenue | −120.00 |

Reverting that approval (FR-03.8) creates a `reversal` transaction with the opposite entries and `reversed_transaction_id` pointing to the original. Refunds (`[Next]`, FR-03.9) use their own types, as described in ORG-14.

### 3.5 `files`

```mermaid
erDiagram
  STORED_FILES {
    uuid id PK
    text bucket "public | private"
    text object_key UK
    text purpose "event_cover | payment_qr | payment_voucher | refund_proof (Next)"
    text content_type "checked by magic bytes"
    int size_bytes "max 5 MB"
    bytea sha256
    uuid uploaded_by_user_id
    uuid uploaded_by_registration_id
    date retain_until "vouchers: +5 years (FR-08.4)"
    timestamptz deleted_at
    timestamptz created_at
  }
```

* Event covers go to the public bucket. Payment QRs, vouchers and refund proofs go to the private bucket, served only through short-lived signed URLs (NFR-01.3).
* A daily job deletes expired private objects and sets `deleted_at`. The metadata and hash stay for audit. An R2 lifecycle rule on the `vouchers/` prefix acts as a safety net.

### 3.6 `notifications`, `audit` and `legal`

```mermaid
erDiagram
  OUTBOX ||--o{ EMAIL_LOG : produces
  OUTBOX {
    uuid id PK
    text type "registration_received, payment_approved, ..."
    jsonb payload
    int attempts
    text last_error
    timestamptz next_attempt_at
    timestamptz processed_at
    timestamptz created_at
  }
  EMAIL_LOG {
    uuid id PK
    uuid outbox_id FK
    citext to_email
    text template
    text provider_message_id
    text status
    timestamptz sent_at
  }
  AUDIT_LOG {
    uuid id PK
    timestamptz occurred_at
    text actor_type "user | participant | system"
    uuid actor_user_id
    text action "PAYMENT_APPROVED, ..."
    text entity_type
    uuid entity_id
    uuid event_id
    inet ip
    text reason
    jsonb before
    jsonb after
  }
  DOCUMENT_VERSIONS ||--o{ CONSENTS : accepted
  DOCUMENT_VERSIONS {
    uuid id PK
    text kind "terms | privacy"
    text version
    text content_sha256
    timestamptz published_at
  }
  CONSENTS {
    uuid id PK
    uuid document_version_id FK
    uuid user_id "nullable"
    uuid registration_id "nullable"
    inet ip
    timestamptz accepted_at
  }
```

* **Outbox:** domain operations insert a row in the same transaction as their change. A background worker sends the email and retries with backoff. Emails are never lost and are never sent for a transaction that was rolled back.
* **Audit log** (FR-06.4): append-only, with the same trigger and grants as the ledger. Indexes on `(entity_type, entity_id)`, `(actor_user_id, occurred_at)` and `(action, occurred_at)`.
* **Consents** (FR-08.2): exactly one of `user_id` or `registration_id` is set (`CHECK`).

---

## 4. Concurrency and Critical Operations

### 4.1 Capacity without overselling (FR-03.13)

Capacity is handled by one atomic statement, with no read-then-write race:

```sql
UPDATE registrations.event_capacity
SET    taken = taken + 1
WHERE  event_id = $1 AND taken < capacity
RETURNING taken;
```

* If no row is returned, the event is full and the registration is refused.
* Rejecting a registration (`taken - 1`) or reopening it runs in the same transaction as its status change.
* The Events module keeps `capacity` in sync through an in-process event when capacity changes. Capacity can never go below `taken` (FR-02.5).

### 4.2 Approving a payment (FR-03.5)

One database transaction, coordinated by the application layer (ADR-0001):

1. Lock the submission (`SELECT ... FOR UPDATE`) and check that it is still `pending` or `in_observation`.
2. Mark the submission `approved` and the registration `approved`.
3. Insert the `payment_approved` transaction and its two entries (the ledger triggers validate them).
4. Insert the `payment_approved` outbox email and the `PAYMENT_APPROVED` audit record.
5. Commit. Two reviewers approving at the same time cannot both succeed, thanks to the lock and the unique index from §3.4.

### 4.3 Registration codes

The code is `EW-` followed by 5 characters from a 31-character alphabet: A–Z and 2–9 without the ambiguous `0`, `O`, `1`, `I` and `L`. That gives 31⁵ ≈ 28.6 million combinations. It is generated randomly, so it reveals neither how many registrations exist nor anyone else's code. A unique constraint plus a retry on conflict guarantees uniqueness.

The code is an identifier, **not** a secret. Knowing a code does not give access to a registration; only the personal-link token does (§3.3).

### 4.4 Scheduled jobs (hosted services)

| Job | Frequency | Effect |
|---|---|---|
| Close registrations | every minute | Deadline reached → `registrations_closed` (FR-02.4) |
| Final rejection after the deadline | every minute | `in_observation` / `rejected` → final, frees capacity, sends email (FR-03.5) |
| Expired QR | hourly | Pauses paid registrations and notifies the organizer (FR-02.2) |
| Finish events | hourly | `ends_at` passed → `finished` |
| Outbox | every few seconds | Sends pending emails |
| File retention | daily | Deletes expired private objects (FR-08.4) |

When the API runs on several instances, each job takes a PostgreSQL advisory lock so it runs only once at a time.

---

## 5. Security in the Database

* Our schemas are not exposed through Supabase's Data API. `anon` and `authenticated` have no privileges on them (ADR-0002).
* `ew_app` has no DDL and no `UPDATE` or `DELETE` on the ledger and audit tables.
* Personal-link tokens are stored only as SHA-256 hashes (NFR-04.1). Payment QRs and vouchers are kept only as R2 object keys in the private bucket.
* Connection strings and R2 keys live in environment variables or a secrets manager, never in the repository.

---

## 6. Migrations

* Each module has its own EF Core `DbContext`, its own schema (`HasDefaultSchema`) and its own `__EFMigrationsHistory` table inside that schema.
* Triggers, grants, extensions (`citext`, `pg_trgm`) and views are added with raw SQL inside migrations.
* Migrations run in the deployment pipeline as `ew_migrator`, through a direct or session-mode connection, before the new API version starts.
* Initial seed data: categories, legal document versions and the first Super Admin (by email).

---

## 7. Expected Volume

Even an optimistic first year is small for PostgreSQL: about 1,000 events × 300 registrations ≈ 300,000 registrations, around 1 million ledger entries and a few GB of data. The indexes above cover all access paths in the wireframes. Partitioning and read replicas are not needed in the MVP.
