# ADR-0006: Hybrid Rendering: SSR for Public Pages, CSR for Panels

* **Status:** Accepted
* **Date:** 26 Oct 2026

## Context

The frontend is Angular 21 (NFR-01.1). Today it is a client-side single-page application (CSR): the server sends an almost empty HTML page and the browser builds everything with JavaScript.

That is a problem for the public pages:

* **Sharing:** events are shared mostly through WhatsApp and social networks. Their link previews read the Open Graph tags (title, image, description) from the raw HTML and **do not run JavaScript**. With CSR, every shared event shows an empty preview (FR-04.2).
* **Discovery:** search engines index server-rendered pages better (FR-04.1).
* **Mobile first:** participants open the link on phones, often on slow connections (NFR-01.2). Server-rendered HTML shows content sooner.

The private panels (organizer, Super Admin, participant pages) need none of this: they sit behind a login or a personal link and must never be indexed.

## Decision

Use Angular's **hybrid rendering** (`@angular/ssr` with per-route render modes in `app.routes.server.ts`):

| Render mode | Routes | Why |
|---|---|---|
| `Server` (SSR on each request) | `/` (PUB-01), `/e/:slug` (PUB-02) | Content changes constantly; Open Graph tags must be in the HTML. |
| `Prerender` (built once at build time) | `/legal/*` (PUB-07) | Static content. |
| `Client` (CSR) | Everything else: registration form, participant pages, auth, `/panel/**`, `/admin/**` | Interactive, private or behind login; no SEO value. |

* SSR pages call the public, anonymous API endpoints only. They never hold user sessions.
* The server sets the page title, description and Open Graph tags (event title, date, city, cover image from the public CDN). It never includes the payment QR (NFR-01.3).
* SSR pages use incremental hydration and `@defer`, so the browser downloads JavaScript only for the parts that become interactive.
* SSR responses for public pages may be cached for a short time (for example 60 s) at the CDN to absorb traffic peaks.

## Consequences

* ✅ Correct link previews in WhatsApp and social networks, plus better SEO for public events.
* ✅ Faster first paint on mobile.
* ✅ Panels keep the simplicity of a single-page application.
* ⚠️ The frontend needs a **Node.js runtime** in production for SSR. It is no longer only static files. This affects the hosting decision (D-02).
* ⚠️ Code that runs on the server cannot use browser-only APIs (`window`, `localStorage`). Those must be guarded or limited to client-only routes.

## Alternatives Considered

| Alternative | Why it was not chosen |
|---|---|
| CSR only | Empty link previews in WhatsApp: unacceptable for an event platform. |
| SSR for every route | Extra server load and complexity for private pages that gain nothing from it. |
| Prerender event pages at build time | Events are created all the time; a rebuild per event is not feasible. |
| Serve Open Graph tags from the .NET API for bots only | A hack: it duplicates rendering logic and needs bot detection. |
