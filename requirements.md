# Event's World · Software Requirements Specification

**Version:** 0.5 (draft for review) · **Date:** 28 Oct 2026 · v0.3: open questions and proposals resolved · v0.4: NFR-01.1 styling (ADR-0007), NFR-01.3 private payment QR · v0.5: final visual style Salar Cristal (NFR-01.1, NFR-01.2, ADR-0008)
**Changes since v0.1:** see [requirements-review.md](requirements-review.md). **Screens:** see `wireframes.excalidraw` (page IDs such as PUB-03 or ORG-08 are frame names there).

---

# 1. Introduction

### 1.1 Purpose

This document defines the functional and non-functional requirements for "Event's World", a web platform for the efficient management of events.

### 1.2 Product Scope

"Event's World" is a multi-user platform for events of any size: a yoga class, a recreational activity, an organization's camp or a charity or competitive race. Organizers publish events and manage registration, payment verification and logistics. Participants discover events and register **without needing an account**.

### 1.3 Key Constraint: Payments Happen Outside the Platform

Participants pay the organizer directly, by scanning the organizer's bank QR code or in cash. Event's World **never holds or moves money**: it collects payment vouchers, lets the organizer verify them, and records every confirmed payment in an immutable ledger.

### 1.4 Release Scope Tags

Every requirement carries one tag:

| Tag | Meaning | Wireframe label |
|---|---|---|
| `[MVP]` | Included in the first release | Existente / Nuevo |
| `[Next]` | Planned right after the MVP; may have a sketch | Próximamente |
| `[v2]` | Later version (payment plans) | v2 |
All open questions (Q-01..Q-10) and proposals (P-01..P-12) were resolved on 24 Oct 2026. Each decision is listed in the review document, §4 and §5, and is cited here by its code.

### 1.5 Glossary

* **Event:** anything an organizer publishes for people to register to.
* **Organizer:** the user who created an event (event owner).
* **Participant:** a person registered to an event, with or without an account.
* **Registration:** a participant's enrollment in one event, identified by a **registration code** (e.g. `EW-7F3K2`).
* **Personal link:** a secret link, sent by email, that gives access to one registration without logging in.
* **Voucher:** a screenshot or PDF of the payment the participant made to the organizer's QR.
* **Ledger / Transaction / Entry / Reversal:** see NFR-03.

---

# 2. Overall Description

### 2.1 User Roles and Permissions

Access is managed with Role-Based Access Control (RBAC) at **two levels**: a global role on the account, and a role on each event.

**Global roles**

1. **Visitor (no account):** browses public events, registers to events, and reaches their registrations through the personal link.
2. **User (account):** everything a Visitor can do, plus "My registrations" (PUB-08) and the ability to create events. Creating an event makes the user its Organizer.
3. **Super Admin (core team):** platform-wide administration (Module 6). Can view, suspend and audit events and users, and change roles. **Cannot** approve payments, see passwords or act as another user.

**Event roles**

4. **Organizer (event owner):** full control over their event, its logistics and its registration form. Has exclusive access to the **Event Balance** (financial metrics).
5. **Co-Administrator** `[Next]`: invited per event (COA-01..03). Supports logistics, payment review and team organization. **Strict restriction:** no access to the Event Balance.

### 2.2 Assumptions and Constraints

* Identity is managed by Supabase Auth (FR-01.2).
* The interface is in Spanish for the MVP (NFR-01.4).
* Each event uses a single currency (NFR-05.2).
* There is no payment gateway (see §1.3).

---

# 3. Functional Requirements (FR)

## Module 1: Authentication, Accounts and Profile

* **FR-01.1 (Accounts)** `[MVP]` · AUT-01: Anyone may create an account. An account is **mandatory only to create and manage events**; registering to an event never requires one.
  * Sign-up asks for First name(s), Last name(s), Email, Password and Date of birth.
  * Users must be **18 or older**, validated in the backend.
  * Password policy: at least 8 characters, with one uppercase letter and one number.
  * The user must accept the terms and privacy policy (FR-08.2).
* **FR-01.2 (Identity and Access Management)** `[MVP]`: Authentication and authorization use **Supabase Auth**, which handles registration, secure credential storage and JWT issuance.
* **FR-01.3 (Security)** `[MVP]`: All communication uses the JWTs issued by Supabase. The backend checks the global role and, for every event-scoped endpoint, the user's event role. Unauthorized access returns 401/403 pages (SYS-01).
* **FR-01.4 (Email Verification)** `[MVP]` (P-01) · AUT-02: Users must verify their email before publishing events. Drafts are allowed without verification. The verification link expires after 24 h.
* **FR-01.5 (Password Recovery)** `[MVP]` (P-02) · AUT-04: Users can reset their password through a single-use link that expires after 1 h. Resetting closes all other sessions.
* **FR-01.6 (Login)** `[MVP]` · AUT-03:
  * The error message is generic and never reveals whether an email exists.
  * The account is temporarily locked after 5 failed attempts.
  * After login the user goes to the page that requested it (returnUrl). Otherwise: Super Admin → ADM-01; users with events → ORG-01; others → PUB-08.
* **FR-01.7 (Profile)** `[MVP]` · ORG-12:
  * The user can edit their names and change their email (re-verification required) and password.
  * "Sign out of all devices" is available.
  * Date of birth is not editable.
  * Theme preference: light, dark or system.
  * Account deletion is allowed only without published events or registrants; financial records are retained (NFR-03).
  * Language selection: `[Next]`.
* **FR-01.8 (Social Login)** `[MVP]` (P-03) · AUT-01, AUT-03: Sign-in with Google through Supabase.

---

## Module 2: Event Creation and Configuration (Organizer)

* **FR-02.1 (Event Data)** `[MVP]` · ORG-02:
  * Required: title, description, cover image (16:9, JPG/PNG, max 5 MB), category, country, city, venue, time zone (pre-filled from the city), start and end date-time, registration deadline, maximum attendees (capacity), and a **public organizer contact** (WhatsApp number or email) shown on the event page (P-07).
  * Optional: minimum age.
  * Validations: end > start; registration deadline ≤ start; capacity ≥ 1.
  * A draft can be saved at any step.
  * Only **in-person** events in the MVP. Online and hybrid events are `[Next]` (Q-04).
* **FR-02.2 (Price and Payment Configuration)** `[MVP]` · ORG-03:
  * A "Free event" toggle hides the payment QR, the amount and the voucher.
  * Paid events require: ticket price, currency, the organizer's **payment QR image**, the **QR expiration date**, and the **amount the QR charges**. If the QR has no amount, it is entered as 0.
  * The QR amount is shown as the "expected amount" during payment review (FR-03.5).
  * Non-blocking warnings (the organizer can still save):
    * The QR amount is different from the ticket price.
    * The QR amount is 0 or empty (an open-amount QR).
    * The QR expires before the registration deadline (a confirmation pop-up appears).
  * If the QR expires while registrations are open, **paid registrations are paused automatically** until the organizer uploads a new QR. The organizer is notified by email and by an alert in My events (FR-02.7), and the public page shows "Registrations paused" (Q-06).
  * Payment plans (partial or monthly payments) are `[v2]` (V2-01).
* **FR-02.3 (Registration Form Builder)** `[MVP]` · ORG-04:
  * Default fields: First name(s), Last name(s), Email and Phone. All can be removed or made optional except Email.
  * **Email** is always required and locked.
  * **Payment voucher upload** is always required and locked for paid events (not shown for free events).
  * The organizer adds custom fields with a label, a type and a required flag; select-type fields have editable options.
  * Fields can be reordered by dragging. A mobile preview is available.
  * Once the event has registrants, fields cannot be deleted, only hidden.
  * Supported custom field types: short text, long text, number, single choice, multiple choice, date, phone and checkbox. File upload fields (e.g. a medical certificate) are `[Next]` (Q-10).
* **FR-02.4 (Event Lifecycle)** `[MVP]` · ORG-11:
  * States: **Draft → Published → Registrations closed → Finished.**
  * Registrations close automatically at the deadline or when capacity is reached, or manually by the organizer.
  * They can be reopened if there is capacity left and the deadline is in the future.
  * "Finished" is set automatically when the event ends.
  * Paid registrations are paused while the payment QR is expired (FR-02.2).
  * An event can be deleted only while it is a Draft with no registrants.
  * **Suspended** (set by the Super Admin, FR-06.2) hides the event and blocks registrations without deleting anything (P-09).
  * **Cancelled** is `[Next]` (FR-03.9).
* **FR-02.5 (Editing Restrictions)** `[MVP]` (Q-02): Event data can be edited while the event is Draft or Published. Once at least one registration exists:
  * **Locked:** dates (start, end and registration deadline), country, city, venue and time zone; price, currency and QR amount.
  * Capacity can only be reduced down to the current number of registrations.
  * Form fields cannot be deleted, only hidden (FR-02.3).
  * **Always editable:** title, description, cover image, category, minimum age and organizer contact.
* **FR-02.6 (Public Link and Visibility)** `[MVP]` · ORG-05, ORG-11: Each event has a public URL (`/e/:slug`) that can be copied or downloaded as a QR image for posters. Hiding the event from public listing (accessible only by link) is `[Next]` (P-04).
* **FR-02.7 (My Events)** `[MVP]` · ORG-01:
  * The organizer's events appear as cards that can be filtered by state and sorted by event date.
  * Cards show alerts: pending vouchers, a QR about to expire, an expired QR (registrations paused), capacity full.

---

## Module 3: Registrations, Payments and Event Logistics

* **FR-03.1 (Team Splitter)** `[Next]` · ORG-10: Tools to distribute participants into groups, by number of teams or people per team. Distribution can be random or balanced by a form field, and participants can be moved between teams by drag and drop. Teams can be exported.
* **FR-03.2 (Filtering by Payment State)** `[MVP]` · ORG-06:
  * The participant list can be filtered by payment state (Pending, Approved, In observation, Rejected) and by origin (online form or in person), and searched by name, email or code.
  * The Team Splitter (`[Next]`) uses the same filters.
  * Filtering by Complete/Partial is `[v2]` (V2-04).
* **FR-03.3 (Online Registration and Payment)** `[MVP]` · PUB-03, PUB-04:
  * Guests register through the dynamic form defined in FR-02.3.
  * The email must be typed twice and match, ignoring case and spaces. If that email already has registrations in the event, the form warns the participant but allows the registration (FR-03.14).
  * For paid events the form shows the total to pay, the organizer's QR with its validity date, and a **mandatory voucher upload** (JPG, PNG or PDF, max 5 MB, validated in the client and again in the server).
  * The participant must accept the privacy policy.
  * On submission:
    * The registration is created as **Pending** (or **Confirmed** for free events) and receives a registration code.
    * A confirmation page is shown. "Add to calendar" (.ics) is `[Next]` (P-08).
    * An email with the personal link (FR-05.1) is sent.
  * There is no CAPTCHA; abuse is limited by NFR-04.2.
  * Choosing a payment plan is `[v2]` (V2-02).
* **FR-03.4 (In-Person Registration and Payment)** `[MVP]` · ORG-09: The organizer registers an attendee in person from a mobile device by filling in the same form. Email is mandatory so every participant receives their personal link; the organizer reads it back aloud to confirm it. The organizer then selects the payment method:
  * **QR:** the organizer's device shows the QR; the organizer presses "Confirm payment received" after seeing the credit in their bank.
  * **Cash:** the organizer enters the cash received; the system shows the change to return and records the full ticket price.
  * The registration is created as **Approved** immediately, and a ledger transaction records the method and the organizer who collected the payment.
  * Partial in-person payments that start a payment plan are `[v2]` (V2-05).
* **FR-03.5 (Payment Review)** `[MVP]` · ORG-08:
  * A review page lists remote vouchers with counts per state (Pending, In observation, Approved, Rejected).
  * The selected voucher is shown as a large, legible image (NFR-01.3), together with:
    * the expected amount (the QR amount; for an open-amount QR, the ticket price with a "QR libre" label);
    * the payment method;
    * whether it is a first submission or a resubmission.
  * Actions:
    * **Approve:** creates a ledger transaction (NFR-03).
    * **Observe:** the reason is mandatory; the participant can upload a new voucher.
    * **Reject:** the reason is mandatory.
  * Every action emails the participant (FR-07.1).
  * Suggested keyboard shortcuts: A, O, R and arrow keys.
  * A participant can resubmit a voucher until the registration deadline. After the deadline, registrations still In observation or Rejected become **Rejected for good**, free their spot and the participant is notified (Q-07). Pending vouchers are always reviewed by the organizer.
* **FR-03.6 (Event Dashboard and Financial Metrics)** `[MVP]` · ORG-05:
  * Shows registrations against capacity, vouchers awaiting review, payment-state distribution, registrations per day, recent activity, and the public link with share options.
  * Financial metrics:
    * **Real money:** the balance of confirmed ledger entries for the event.
    * **Estimated money:** active registrations × ticket price. In `[v2]` it includes amounts committed in payment plans.
  * Financial cards are visible only to the Organizer.
* **FR-03.7 (Edit Participant Data)** `[MVP]` · ORG-07:
  * The **Organizer** can correct a participant's data, with clear warnings.
  * Every change requires a reason, is kept in the registration's change history and in the audit log (FR-06.4), and is notified to the participant.
  * Only the organizer can change a participant's email. Changing it resends the personal link.
  * Price and payment state cannot be edited here.
* **FR-03.8 (Revert an Approval)** `[MVP]` · ORG-07: Only to **correct an approval made by mistake**. A reason is mandatory. The system creates a reversal transaction linked to the original (NFR-03.4), the registration returns to Pending, and the action is audited. Reversal is never used for refunds.
* **FR-03.9 (Cancellations, Annulments and Refunds)** `[Next]` · ORG-14, ORG-11, ORG-07:
  * The organizer can annul a registration (with or without a refund, according to their policy) or cancel the event.
  * Refunds are paid outside the platform; the organizer records them with a proof of payment.
  * Cancelling an event creates a pending refund for each approved payment and rejects vouchers not yet approved.
  * New registration states: *Annulled · refund pending → Refunded*, and *Annulled without refund*.
  * Ledger treatment: NFR-03.4.
* **FR-03.10 (Manual Payment After Rejection)** `[MVP]` · ORG-07: The organizer can record an in-person payment (QR or cash) for a registration whose voucher was rejected.
* **FR-03.11 (Check-in)** `[Next]` (P-06) · ORG-13: On the event day, the organizer scans a participant's entry QR, or searches by name or code, to record attendance. Only approved registrations can enter. It works with poor connectivity.
* **FR-03.12 (Export)** `[MVP]` (P-05) · ORG-06: Export the participant list, including custom fields, to CSV.
* **FR-03.13 (Capacity)** `[MVP]` (Q-01): Registrations that are **Pending, In observation or Approved** count toward capacity; **Rejected** ones free their spot. Capacity is checked in the server with a lock so concurrent registrations cannot exceed it. In-person registrations count the same way.
* **FR-03.14 (Several Registrations per Email)** `[MVP]` (Q-05): One email may hold several registrations in the same event (e.g. a parent registering their children). Each registration has its own code and personal link.

---

## Module 4: Public Discovery

* **FR-04.1 (Explore Events)** `[MVP]` · PUB-01:
  * Lists only Published events and events with closed registrations that are still upcoming.
  * Supports search, filters (category, city, date, free) kept in the URL, and incremental loading.
* **FR-04.2 (Event Detail)** `[MVP]` · PUB-02:
  * A shareable public page with Open Graph metadata, the event information and a map link.
  * The registration button reflects the event state: available, capacity full, registrations closed, registrations paused (expired QR) or cancelled.
  * Shows the organizer's public contact (WhatsApp or email) defined in FR-02.1 (P-07).
* **FR-04.3 (Public Navigation)** `[MVP]` · PUB-09: On mobile, a ☰ menu gives access to all public options, with and without a session. On desktop the same links appear in the header.

---

## Module 5: Participant Access Without an Account

* **FR-05.1 (Personal Link)** `[MVP]` · PUB-05:
  * Each registration is stored on the server whether or not the participant has an account.
  * The confirmation email and every status email include a personal link that opens the registration page without logging in.
  * The page shows the payment status, the submission history and the participant's data (read-only). It allows uploading a new voucher when the payment is In observation or Rejected.
  * The link is valid until 30 days after the event ends. Security: NFR-04.1.
* **FR-05.2 (Link Recovery)** `[MVP]` · PUB-06:
  * A participant enters their email and receives links to their active registrations.
  * The response is identical whether or not the email exists, and requests are rate-limited (NFR-04.2).
* **FR-05.3 (My Registrations)** `[MVP]` · PUB-08:
  * Users with an account see every registration made with their **verified** email, linked automatically, split into upcoming and past.
  * Past registrations remain as account history indefinitely.
* **FR-05.4 (Email Corrections)** `[MVP]`: Participants cannot change their email after registering. If a correction is needed, the organizer applies it (FR-03.7).
* **FR-05.5 (Entry QR)** `[Next]` (P-06): Approved registrations show an entry QR in PUB-05 and in the approval email, used by check-in (FR-03.11).

---

## Module 6: Platform Administration (Super Admin)

* **FR-06.1 (Platform Panel)** `[MVP]` · ADM-01:
  * Shows platform metrics: organizers, published events, recent registrations and vouchers unreviewed for more than 48 h.
  * Shows a list of events that need attention, based on simple rules (P-10): vouchers unreviewed for more than 48 h, an expired QR with registrations open, and a high rejection rate (30 % or more).
  * Alerts from user reports are `[Next]`.
* **FR-06.2 (Events)** `[MVP]` · ADM-02:
  * Lists all events with filters, a public view and a read-only dashboard.
  * The Super Admin can suspend and reactivate an event, with a mandatory reason and notice to the organizer (P-09). Suspension hides the event and blocks registrations without deleting anything.
* **FR-06.3 (Users)** `[MVP]` · ADM-03:
  * Lists users with their role, events and state.
  * Actions:
    * Suspend an account (blocks login and unpublishes the user's active events; reason mandatory).
    * Change a user's role.
    * Send a password-reset link.
  * The Super Admin cannot see passwords or act as the user.
* **FR-06.4 (Audit Log)** `[MVP]` · ADM-04:
  * An append-only log of every sensitive action: payment approvals and reversals, in-person payments, participant data edits, event state changes, suspensions, role changes and failed logins.
  * Each record stores the actor, timestamp, IP, the entity affected, the reason, and before/after values.
  * The log can be filtered and exported.

---

## Module 7: Notifications

* **FR-07.1 (Transactional Emails)** `[MVP]` · SYS-02:
  * The system sends emails to participants and organizers for:
    * registration received (with code and personal link);
    * payment approved, observed (with reason) or rejected (with reason);
    * registration rejected for good after the registration deadline (FR-03.5);
    * participant data edited by the organizer;
    * payment QR expired and paid registrations paused (to the organizer, FR-02.2);
    * event suspended or reactivated by the Super Admin (to the organizer);
    * personal link recovery;
    * account verification and password reset.
  * Every participant email includes the personal link.
* **FR-07.2 (Organizer Notification Settings)** `[Next]` · ORG-11:
  * Email when a new voucher arrives.
  * Daily registration summary.
* **FR-07.3 (QR Expiration Reminder)** `[Next]`: Notify the organizer 3 days before their payment QR expires.

---

## Module 8: Legal and Privacy

* **FR-08.1 (Terms and Privacy Pages)** `[MVP]` · PUB-07:
  * Public pages for terms of use and the privacy policy, linked from the registration form, sign-up, the public footer and the public menu.
  * The final legal text is drafted separately, ideally reviewed by a legal advisor, and must cover:
    * the service scope;
    * payments made outside the platform;
    * refunds and cancellations (organizer's responsibility);
    * personal data processed, its purpose and who can see it;
    * service providers;
    * retention periods;
    * user rights and contact;
    * the data protection rules of Bolivia and of each country of operation.
* **FR-08.2 (Consent Versioning)** `[MVP]`: The system stores which version of the terms and privacy policy each person accepted, and when (at registration or sign-up). Users are asked to accept again when the documents change.
* **FR-08.3 (Data Deletion Requests)** `[Next]` (P-11): On request, personal data is anonymized; ledger records are kept.
* **FR-08.4 (Data Retention)** `[MVP]` (Q-08): Payment vouchers and payment records are kept for **5 years** (provisional, to be confirmed with the final legal text). Registration history stays in the user's account indefinitely (FR-05.3). Personal links expire 30 days after the event ends (FR-05.1).

---

# 4. Non-Functional Requirements (NFR)

## NFR-01: User Interface, Experience, and Content Delivery (UI/UX & CDN)

* **NFR-01.1 (Aesthetics, Styling and Components):** The frontend will be developed in **Angular** using **TypeScript**, with a single styling system and its own component library ([ADR-0007](architecture/adr/0007-styling-and-components.md)):

  * **Tailwind CSS** is the only styling system: layout, grid (Mobile First), typography, colors and every component's look. It reads the design tokens (CSS variables) that define the light and dark themes.
  * **Angular CDK** provides behavior and accessibility for complex components without imposing any look: dialogs and overlays, menus and drawers, focus management, drag and drop, data tables and clipboard.
  * **Own component library** (`shared/ui`), built from the wireframe component library "00": buttons, inputs, badges, cards, stepper, dialogs, drawer, table, file drop zone, toasts and others.
  * **Native inputs** for date-time and simple selects, because mobile operating systems provide the best pickers. Charts use a small library (Chart.js) loaded only on dashboards. Icons use Lucide.
  * **Visual style: Salar Cristal** ([ADR-0008](architecture/adr/0008-salar-cristal-design-system.md); full specification in [design/design-system.md](design/design-system.md)), applied to **every view** in both light and dark modes:
    * The Salar palette: sky blue primary, with a coral accent used sparingly.
    * A fixed sky backdrop with blue and coral reflections behind the whole app.
    * **Intense glassmorphism** on panels, bars and controls: translucent surfaces with strong background blur, light borders and a top highlight.
    * Typeface **Outfit** (weights 400 / 600 / 700) and **soft corners** (6 / 12 / 18 px, pills for chips and status badges).
    * The primary button, status badges, toasts, event covers and payment vouchers are never glass.
  * **Glass guardrails:**
    * Every text passes WCAG AA in the worst case of the glass over the backdrop.
    * Glass turns into solid surfaces when the user prefers reduced transparency or the browser lacks `backdrop-filter`.
    * Blur is applied to containers, not to individual rows; at most about 6 blurred layers are visible at once on mobile.
    * Every release is tested on a low-end Android phone.

* **NFR-01.2 (Responsive Design):** Mandatory "Mobile First" approach, assuming heavy mobile usage, with native Dark Mode support. Light and dark themes are both part of the Salar Cristal system (design-system.md). Every view has a theme toggle (☾), and the choice follows the system preference by default.

* **NFR-01.3 (CDN & Media Storage):** **Cloudflare R2** will be used for user-uploaded media, split into two buckets:
  * **Public bucket:** event covers and other public assets, served through the CDN.
  * **Private bucket:** payment vouchers, refund proofs and **the organizers' payment QR images**. They are **never** public: the backend authorizes each request and returns a signed URL valid for a few minutes, served with `Cache-Control: private, no-store`.
  * The payment QR contains bank data. It is shown only inside the registration form (PUB-03) and the in-person payment flow (ORG-09), never in public listings or Open Graph previews.
  * Uploads go through the backend, which validates type (by magic bytes) and size before storing the file. Expired files are deleted according to FR-08.4. See `architecture/database.md` §3.5.

* **NFR-01.4 (Language):** The MVP interface is in Spanish. The frontend is built ready for translation; language selection is `[Next]`.

---

## NFR-02: Performance and Architecture (Backend)

* **NFR-02.1 (Structural Framework):** The backend will be built using **C#** and **.NET (ASP.NET Core Web API)**.

* **NFR-02.2 (Object-Oriented Paradigm - OOP):** Backend development will follow SOLID principles. Classes, Interfaces, and Inheritance will be used.

* **NFR-02.3 (Native Dependency Injection):** The native .NET Dependency Injection container (`IServiceCollection`) will be exclusively used to decouple Controllers, Services, and Repositories.

* **NFR-02.4 (ORM & Data Access):** **Entity Framework Core (EF Core)** with the `Npgsql` provider will be used for database access and migrations to PostgreSQL, supporting the Ledger pattern and JSONB operations.

---

## NFR-03: Data Modeling and Ledger Pattern (PostgreSQL)

* **NFR-03.1 (Relational Database):** **PostgreSQL** will be the primary database engine. `JSONB` columns will be used to store dynamic user responses for "Extra Fields."

* **NFR-03.2 (Strict Ledger Pattern Implementation):** The use of static and mutable `balance` fields (e.g., a simple `UPDATE balance = balance - X`) is strictly prohibited. The model uses double entry: each transaction's entries sum to zero.

  * `Accounts`: financial accounts **per event**:
    * *Organizer cash* (money the organizer received, by method: QR or cash);
    * *Registration revenue*;
    * *Refunds payable* `[Next]`.

    Participants have no balance or account. Every entry carries the `registration_id`, so a registration's financial state is computed from its entries.

    There is **no platform account in the MVP**: Event's World charges no fees. The schema must allow adding platform accounts later, for per-registration fees and/or subscription plans (Q-03).
  * `Transactions`: **immutable** logical grouping of an operation (e.g., "Payment approved for EW-7F3K2"), with a type, actor, timestamp and reason.
  * `Entries`: **immutable** individual movements within a transaction, recording positive and negative amounts.

* **NFR-03.3 (Balance Calculation):** The available balance for any account will be computed dynamically (or via materialized views) by summing all confirmed `Entries`.

* **NFR-03.4 (Ledger Reversals and Refunds):** Modifying or deleting existing entries is strictly prohibited.
  * **Reversal** `[MVP]` is used **only to correct mistakes** (FR-03.8). It is a new transaction with inverse entries, linked to the original through `reversed_transaction_id`.
  * **Refunds and cancellations** `[Next]` are real economic events (money came in and later went out), so they use their own transaction types and are never reversals:
    * `REFUND_PENDING` when a registration is annulled or the event is cancelled;
    * `REFUND_PAID`, with proof, when the organizer returns the money.

    This keeps the organizer's cash consistent with their bank statement.
  * All transaction types are defined from the start, even if the MVP only uses `PAYMENT_APPROVED` and `REVERSAL`. The full table is in ORG-14 of the wireframes.

---

## NFR-04: Security and Anti-Abuse `[MVP]`

* **NFR-04.1 (Personal Link Tokens):** Opaque random tokens of at least 256 bits, stored only as a hash. They can be revoked and have an expiration date (FR-05.1). When the link is opened, the token is exchanged for a short-lived session limited to that registration, and the token is removed from the URL (`Referrer-Policy: no-referrer`).
* **NFR-04.2 (Rate Limiting Instead of CAPTCHA):** Public forms (registration, link recovery, sign-up, login and password reset) have no CAPTCHA in the MVP. They are protected by per-IP and per-email limits and by server-side validation rules (for example, 3 link-recovery emails per hour per email). CAPTCHA will be reconsidered if abuse appears.
* **NFR-04.3 (File Validation):** Uploaded files are validated in the server by type, size and actual content, not only by extension.
* **NFR-04.4 (No User Enumeration):** Login, password reset and link recovery return the same response whether or not the email exists.
* **NFR-04.5 (Auditability):** Every sensitive action is written to the audit log (FR-06.4).

---

## NFR-05: Time and Money `[MVP]`

* **NFR-05.1 (Time Zones):** Date-times are stored in UTC together with the event's time zone, and are always displayed in the event's local time, including registration deadlines (Q-09).
* **NFR-05.2 (Money):** Amounts are stored as `numeric(12,2)` with an ISO 4217 currency code, never as floating point. Each event has one currency.

---

## NFR-06: Quality `[Next]` (P-12)

* **NFR-06.1 (Accessibility):** WCAG 2.1 AA for public pages and forms.
* **NFR-06.2 (Observability):** Structured logs and error tracking in the frontend and the backend.
* **NFR-06.3 (Backups):** Automated daily database backups with point-in-time recovery.

---

# 5. Release Roadmap

| Release | Content |
|---|---|
| **MVP** | Modules 1–8 tagged `[MVP]`: accounts (with email verification, password recovery and Google login), in-person event creation (single payment or free), online and in-person registration, payment review, capacity control, CSV export, dashboard, participant data editing, approval reversal, public discovery, personal link access, Super Admin (with alerts and event suspension), transactional emails, terms, privacy and data retention |
| **Next** | Co-administrators (COA-01..03), Team Splitter (FR-03.1), cancellations, annulments and refunds (FR-03.9), check-in and entry QR (FR-03.11, FR-05.5), online and hybrid events, events hidden from listing, file upload fields, add to calendar, organizer notification settings (FR-07.2/07.3), language selection, data deletion requests (FR-08.3), quality NFRs (NFR-06) |
| **v2** | Payment plans: plan configuration, partial payments in online and in-person registration, Complete/Partial filters, estimated money including plans (V2-01..05) |
| **Future** | Platform fees per registration and/or subscription plans (Q-03) |
