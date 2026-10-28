# ADR-0004: Authentication and Two-Level Authorization

* **Status:** Accepted
* **Date:** 26 Oct 2026

## Context

The requirements (v0.3) define three kinds of callers:

1. **Users with an account,** authenticated by Supabase Auth (FR-01.2). They may be organizers of some events, co-administrators of others (`[Next]`), or Super Admins.
2. **Participants without an account,** who reach one registration through the personal link (FR-05.1, NFR-04.1).
3. **Anonymous visitors** browsing public pages (Module 4).

Roles exist at two levels: a global role on the account and a role per event (requirements §2.1, `events.event_members`). Suspended users and suspended events must be blocked (FR-06.2, FR-06.3).

## Decision

1. **User authentication:**
   * The API validates Supabase access tokens (JWT) with the standard ASP.NET Core JWT bearer handler.
   * Tokens are validated against the project's **JWKS endpoint** (`/auth/v1/.well-known/jwks.json`, asymmetric signing keys), so no shared JWT secret lives in the backend.
   * `sub` is the user ID, which equals `accounts.users.id`.
2. **Global roles come from our database, not from JWT claims:**
   * A request-scoped `ICurrentUser` loads `global_role` and `status` from `accounts.users`, cached for a few seconds.
   * Suspending a user or changing a role takes effect immediately, without waiting for the token to expire.
   * Suspended users receive 403 on every authenticated endpoint.
3. **Event-level authorization is resource-based:**
   * Event-scoped endpoints declare the role they need, for example `[RequireEventRole(EventRole.Organizer)]`.
   * An authorization handler reads the `eventId` from the route and checks `events.event_members` through `IEventsModule`.
   * The Event Balance (financial metrics) requires `Organizer`; co-admins will be denied (`[Next]`).
   * Super Admins get read-only access to event dashboards (FR-06.2), never write access to payments.
4. **Participants without an account:**
   * `POST /api/v1/participant/sessions` exchanges the personal-link token (hashed lookup in `registrations.access_tokens`) for a **short-lived participant token** (30 minutes) issued and signed by our backend.
   * This is a separate authentication scheme (`Participant`). Its claims are `registration_id` and `scope = registration`.
   * Participant endpoints can only see and act on that one registration.
5. **Anonymous endpoints** are limited to the public catalog, event detail, registration creation, link recovery and participant session exchange. They are protected by rate limiting (NFR-04.2).
6. **Deny by default:** every endpoint requires authentication unless it is explicitly marked `[AllowAnonymous]`. Architecture tests list the anonymous endpoints, so any new one shows up in code review.

## Consequences

* ✅ No secret is shared with Supabase; key rotation is handled by JWKS.
* ✅ Role changes and suspensions apply instantly.
* ✅ Co-administrators (`[Next]`) only add rows to `event_members` and a new `EventRole` value.
* ✅ The participant token never grants more than one registration, and the personal-link secret leaves the URL right after the exchange.
* ⚠️ Checking roles in the database adds one cached query per request. This is acceptable at our scale.
* ⚠️ We sign our own participant tokens, so the backend manages one signing key, stored as a secret and rotatable.

## Alternatives Considered

| Alternative | Why it was not chosen |
|---|---|
| Roles as custom JWT claims (Supabase Auth Hook) | Changes would only apply when the token refreshes; a suspended user could keep acting until then. |
| Supabase RLS policies as the authorization layer | Business authorization belongs in .NET (ADR-0001); our schemas are not exposed to the Data API (ADR-0002). |
| Anonymous Supabase sign-in for participants | Would create auth users for every registration and complicate account linking (FR-05.3). |
| Using the personal-link token directly on every request | Keeps the long-lived secret in URLs, logs and browser history (NFR-04.1). |
