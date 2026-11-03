# ADR-0005: Resend as the Email Provider

* **Status:** Accepted
* **Date:** 26 Oct 2026

## Context

Event's World depends on email.

* **Participants without an account** reach their registration only through the personal link sent by email (FR-05.1, FR-07.1).
* **Supabase Auth** sends verification and password-reset emails (FR-01.4, FR-01.5). Its built-in email service is meant for testing and has very low sending limits, so production needs a custom SMTP provider.

The backend sends emails through the outbox (database.md §3.6, backend.md §5). The provider was left open as D-01 in backend.md.

## Decision

1. **Resend** is the email provider, for both:
   * **Application emails:** sent by the Notifications outbox worker through Resend's HTTP API.
   * **Supabase Auth emails:** sent through Resend's SMTP, configured in Supabase's "Custom SMTP" setting.
2. Emails are sent from a domain we own (for example `notificaciones@eventsworld.app`), with SPF, DKIM and DMARC records configured, so they do not land in spam.
3. The backend depends on an `IEmailSender` interface inside the Notifications module. Resend is one implementation of it, so changing provider only replaces that class.
4. Each outbox message ID is sent as the idempotency key when the API supports it, to limit duplicates when the worker retries.

## Consequences

* ✅ One provider for every email, with a free tier enough for the MVP.
* ✅ It integrates with Supabase Auth through SMTP and needs no extra code.
* ✅ The interface keeps us independent of the provider.
* ⚠️ We need our own domain and its DNS records before launch.
* ⚠️ Pricing and free-tier limits must be checked before launch and reviewed as volume grows.

## Alternatives Considered

| Alternative | Why it was not chosen |
|---|---|
| Supabase built-in email | For testing only; very low limits. |
| Amazon SES | Cheapest at volume, but requires AWS account setup and leaving SES sandbox mode; more operational work for the MVP. |
| Brevo / Postmark | Valid options; Resend was preferred for its simpler API and developer experience. |
