# Frontend Architecture

* **Version:** 0.1 · **Date:** 26 Oct 2026
* **Decisions:** [ADR-0006](adr/0006-hybrid-rendering.md) (hybrid rendering), [ADR-0007](adr/0007-styling-and-components.md) (styling and components), [ADR-0008](adr/0008-salar-cristal-design-system.md) (Salar Cristal visual system) and [ADR-0004](adr/0004-authentication-and-authorization.md) (authentication).
* **Related:** [backend.md](backend.md), [requirements.md](../requirements.md) v0.3, `wireframes.excalidraw` (screen IDs).
* **Stack:** Angular 21 (standalone components, signals, **zoneless** change detection), TypeScript, Tailwind CSS + Angular CDK + own component library (NFR-01.1, ADR-0007), Vitest.

---

## 1. Principles

1. **Mobile first** (NFR-01.2). Every screen is designed at 390 px first; desktop layouts are an enhancement.
2. **Signals for state, zoneless change detection.** The project already runs without `zone.js`. Components use `signal`, `computed` and `input()` / `output()`.
3. **Lazy loading by feature.** Each area (public, participant, auth, organizer, admin) is a lazily loaded route tree, so a participant on a phone never downloads the organizer panel.
4. **Hybrid rendering** (ADR-0006): SSR for public pages, CSR for the rest.
5. **The backend is the source of truth.** Guards and hidden buttons improve the experience, but every permission is enforced again by the API (ADR-0004).

---

## 2. Routes, Layouts and Rendering

| Route | Screen | Layout | Render | Access |
|---|---|---|---|---|
| `/` | PUB-01 Explore events | Public | **Server** | Anonymous |
| `/e/:slug` | PUB-02 Event detail | Public | **Server** | Anonymous |
| `/e/:slug/inscripcion` | PUB-03 Registration form | Public | Client | Anonymous |
| `/e/:slug/inscripcion/listo` | PUB-04 Registration sent | Public | Client | Anonymous |
| `/mi-inscripcion/recuperar` | PUB-06 Recover link | Public | Client | Anonymous |
| `/mi-inscripcion/:token` | PUB-05 My registration | Participant | Client | Personal link → participant token |
| `/legal/terminos`, `/legal/privacidad` | PUB-07 Terms and privacy | Public | **Prerender** | Anonymous |
| `/mis-inscripciones` | PUB-08 My registrations | Public | Client | Logged in |
| `/registro`, `/registro/verificar` | AUT-01, AUT-02 | Auth | Client | Anonymous |
| `/login`, `/recuperar` | AUT-03, AUT-04 | Auth | Client | Anonymous |
| `/panel/eventos` | ORG-01 My events | Organizer shell | Client | Logged in |
| `/panel/eventos/nuevo`, `/panel/eventos/:id/editar?paso=1..3` | ORG-02..04 Event wizard | Organizer shell | Client | Organizer of the event |
| `/panel/eventos/:id` | ORG-05 Dashboard | Organizer shell | Client | Organizer |
| `/panel/eventos/:id/participantes[/:pid]` | ORG-06, ORG-07 | Organizer shell | Client | Organizer |
| `/panel/eventos/:id/pagos` | ORG-08 Payment review | Organizer shell | Client | Organizer |
| `/panel/eventos/:id/presencial` | ORG-09 In-person registration | Organizer shell (mobile) | Client | Organizer |
| `/panel/eventos/:id/configuracion` | ORG-11 Settings | Organizer shell | Client | Organizer |
| `/panel/perfil` | ORG-12 Profile | Organizer shell | Client | Logged in |
| `/admin`, `/admin/eventos`, `/admin/usuarios`, `/admin/auditoria` | ADM-01..04 | Admin shell | Client | Super Admin |
| `**` | SYS-01 Not found / errors | Public | Client | Anonymous |

* `[Next]` routes (ORG-10 Teams, ORG-13 Check-in, co-admin pages) exist behind **feature flags** read from the runtime configuration. In the MVP they are disabled, so their menu items show "próximamente" (as in the wireframes).
* `/mi-inscripcion/recuperar` is declared before `/mi-inscripcion/:token`, so "recuperar" is never read as a token.

### Layouts (shells)

| Layout | Content | Wireframes |
|---|---|---|
| **Public** | Header with logo, ☾ toggle and "Ingresar" or avatar. ☰ opens the public menu drawer. On desktop the links sit in the header. | PUB-01, PUB-09 |
| **Participant** | Minimal header with back arrow and ☾; no navigation menu. | PUB-05 |
| **Auth** | Centered card (440 px on desktop, full width on mobile). | AUT-01..04 |
| **Organizer shell** | Desktop: sidebar ("Mis eventos" + event-scoped items + "Perfil · Cerrar sesión") and an **event selector**. Mobile: bottom bar Dashboard · Pagos · Participantes · Más. | ORG-01..12 |
| **Admin shell** | Sidebar Panel · Eventos · Usuarios · Auditoría. | ADM-01..04 |

---

## 3. Folder Structure

```
src/
├─ main.ts · main.server.ts · server.ts      ← browser and SSR entry points (@angular/ssr)
├─ app/
│  ├─ app.config.ts · app.config.server.ts
│  ├─ app.routes.ts · app.routes.server.ts   ← client routes and per-route render modes
│  ├─ core/                                  ← singletons, loaded once
│  │  ├─ api/            generated API client + HTTP interceptors (auth, errors, idempotency)
│  │  ├─ auth/           Supabase session, participant session, guards
│  │  ├─ config/         runtime config (API URL, Supabase URL + anon key, feature flags)
│  │  ├─ errors/         ProblemDetails `code` → Spanish message, global error handler
│  │  ├─ theme/          light / dark / system, persisted (FR-01.7, NFR-01.2)
│  │  └─ seo/            title, meta and Open Graph service (SSR pages)
│  ├─ shared/
│  │  ├─ ui/             components from the wireframe library "00": button, badge (payment and
│  │  │                  event states), event card, file drop zone, toggle, stepper, KPI card, empty state
│  │  ├─ forms/          dynamic form renderer (8 field types + email confirm + voucher)
│  │  └─ pipes/          money, event-local date, relative time
│  ├─ layouts/           public · participant · auth · organizer-shell · admin-shell
│  └─ features/
│     ├─ public/         explore, event-detail, registration, legal, public-menu          PUB-01..04, 07, 09
│     ├─ participant/    my-registration, recover-link, my-registrations                  PUB-05, 06, 08
│     ├─ auth/           sign-up, verify-email, login, reset-password                     AUT-01..04
│     ├─ organizer/      my-events, event-wizard, dashboard, participants, payments,
│     │                  in-person, settings, profile                                     ORG-01..12
│     └─ admin/          overview, events, users, audit                                   ADM-01..04
└─ styles.css            Tailwind v4 (@theme inline), Salar Cristal tokens (light/dark), glass utilities, sky backdrop
```

Each feature folder contains its routes file, pages (route components), feature components and a **feature store** (§4). Features never import from each other; shared code moves to `shared/` or `core/`. ESLint boundary rules enforce this.

---

## 4. State and Data Access

* **Generated API client:** TypeScript types and Angular `HttpClient` services are generated from the backend's OpenAPI document (for example with **orval**). Requests and responses always match the API, and a breaking change fails the frontend build.
* **Feature stores:** an `@Injectable()` service per feature, provided in its route. It holds `signal`s (data, loading, error) and exposes `computed` views and methods that call the API. There is no NgRx in the MVP; `@ngrx/signals` can be adopted later if a feature grows complex.
* **Server state first:** data is fetched when a route opens and refreshed after each mutation. There is no global cache in the MVP. The payment review (ORG-08) refreshes its counts after every action.
* **Interceptors:**
  * `authInterceptor` adds the Supabase access token, or the participant token on `/participant/...`.
  * `errorInterceptor` turns ProblemDetails into typed errors and handles 401 (go to login, with a return URL) and 403 (SYS-01).
  * `idempotencyInterceptor` adds an `Idempotency-Key` to registration requests (backend.md §4).

---

## 5. Authentication in the Browser

| Caller | How |
|---|---|
| **User with an account** | `@supabase/supabase-js` handles sign-up, email verification, login (email + Google), password reset and token refresh. The API receives the Supabase access token (ADR-0004). Supabase only stores the session in the browser, so authenticated routes are always client-rendered. |
| **Participant (personal link)** | `/mi-inscripcion/:token` sends the token to `POST /participant/sessions`, keeps the 30-minute participant token **in memory and `sessionStorage`**, then replaces the URL so the token disappears from the address bar and history. The page sets `Referrer-Policy: no-referrer` (NFR-04.1). |
| **Guards** | `authGuard` (logged in), `superAdminGuard`, `eventRoleGuard` (asks the API for the user's role in `:id`). They only shape navigation; the API enforces every rule. |

---

## 6. Forms

* **Typed Reactive Forms** (stable API). Angular's Signal Forms are still experimental in v21; we will reconsider them later.
* **One dynamic form renderer** builds the registration form from the event's form definition (FR-02.3): the 8 field types, the email confirmation (FR-03.3) and the voucher upload. It is reused in three places:
  1. the public registration form (PUB-03);
  2. in-person registration (ORG-09);
  3. the mobile preview inside the form builder (ORG-04).
* **Validation mirrors the backend:** required fields, email match ignoring case and spaces, file type and size (5 MB) before upload. Server errors map back to their fields.
* **Uploads** show progress (`HttpClient` progress events) and can be retried without losing the form data.
* **Non-blocking warnings** (QR amount ≠ price, open-amount QR, QR expiring before the deadline: FR-02.2) are shown as dismissible alerts, never as validation errors.

---

## 7. UI, Theming and Language

* **Styling and components** ([ADR-0007](adr/0007-styling-and-components.md)):
  * **Tailwind CSS v4** is the only styling system.
  * The **Angular CDK** adds behavior and accessibility (dialog, overlay, menu, focus trap, drag and drop, table, clipboard).
  * `shared/ui` holds our own components, built from the wireframe library "00".
  * Native `datetime-local` and `select` inputs give phones their system pickers.
  * Chart.js is lazy-loaded only on dashboards, and icons come from Lucide.
* **Visual system: Salar Cristal** ([ADR-0008](adr/0008-salar-cristal-design-system.md)). The single source is [design/design-system.md](../design/design-system.md), which holds:
  * the tokens;
  * the glass levels;
  * the backdrop;
  * the typography;
  * a ready-made `styles.css` (§8);
  * the component-by-component mapping (§9).
* **Design tokens** are CSS custom properties (`--ew-*`), redefined for dark mode under `.dark`. Tailwind v4 maps them with `@theme inline`, so utilities such as `bg-surface` or `text-muted` follow the active theme.
* **Glass:**
  * Two Tailwind utilities, `glass` (panels, bars, controls) and `glass-overlay` (modals and drawers), sit over a fixed sky backdrop painted on `body::before`.
  * Glass turns solid under `prefers-reduced-transparency` and when `backdrop-filter` is unsupported.
  * The primary button, status badges, toasts, covers and vouchers stay solid.
* **Typography and corners:**
  * Outfit (400 / 600 / 700) on a Minor Third scale over 12 / 14 / 16 px.
  * Soft corners: 6 / 12 / 18 px, and pills for chips and badges.
* **Dark mode:**
  * A `dark` class on `<html>`; the default follows `prefers-color-scheme`.
  * The choice is stored in `localStorage`, and in the profile for logged-in users (FR-01.7).
  * A tiny inline script applies it before first paint, so there is no light flash on SSR pages.
* **Status colors** follow the wireframe library "00": Pending (yellow), Approved (green), In observation (purple), Rejected (red), and the event states.
* **Language:**
  * The MVP is Spanish only (NFR-01.4).
  * Templates use Angular's built-in `i18n` markers and `$localize` from the start, so adding languages later (`[Next]`) does not require touching every component.
  * Locale `es-BO` for numbers and currency. Dates always show in the **event's time zone** (NFR-05.1) through a shared pipe.

---

## 8. Performance and Accessibility

* Lazy routes per feature, `@defer` for below-the-fold content, and incremental hydration on SSR pages.
* `NgOptimizedImage` for event covers served by the public CDN (sizes and lazy loading).
* The bundle budgets already in `angular.json` stay enforced in CI.
* **Glass performance** (design-system.md §7):
  * Blur goes on containers, never on each list row.
  * At most about 6 blurred layers are on screen at once on mobile.
  * Blurred elements are never animated.
  * Each release is checked on a low-end Android. A screen that drops frames reduces its blur to `blur(14px)`.
* The CDK's accessibility primitives plus semantic HTML give the baseline. Every `shared/ui` component's tests cover labels, roles and keyboard use. A full WCAG 2.1 AA audit is `[Next]` (NFR-06).

---

## 9. Security

* No secrets in the frontend. The Supabase anon key is public by design and safe, because our schemas are not exposed to the Data API (ADR-0002).
* A Content Security Policy is set by the SSR server: scripts only from our origin, plus the hash of the inline theme script (§7), generated by Angular's `autoCsp` build option; images from our CDN and signed R2 URLs.
* Payment QRs and vouchers are shown only from short-lived signed URLs and never cached (NFR-01.3).
* No personal data in URLs except the personal-link token, which is removed right after the exchange (§5).

---

## 10. Testing and Tooling

| Level | Tooling | Covers |
|---|---|---|
| Unit / component | Vitest (already configured) + Angular Testing Library | Stores, pipes, the dynamic form renderer, guards |
| End to end | Playwright, with mobile and desktop viewports | Critical flows: register and pay, review a payment, in-person registration, personal link |
| Static checks | ESLint (angular-eslint, boundary rules) + Prettier (already configured) | Code style, feature boundaries |
| CI | GitHub Actions | Lint, tests, build and bundle budgets on every PR to `main` |

---

## 11. Starting From the Current Repository

The repository currently holds the Angular 21 starter (zoneless, Vitest, empty routes). Implementation will:

1. Add SSR with `ng add @angular/ssr` and define `app.routes.server.ts` (§2).
2. Add Tailwind CSS v4, the Angular CDK and Lucide; set up the design tokens and theme (§7); build the `shared/ui` components (about 1 to 2 weeks).
3. Add `@supabase/supabase-js`, the API client generator and ESLint.
4. Create the folder structure (§3) and the layouts (§2).

These changes go through a feature branch and PR, following conventions.md.

---

## 12. Open Decisions

| # | Decision | Note |
|---|---|---|
| D-02 | Hosting for the frontend SSR server and the API | SSR needs a Node.js runtime (ADR-0006); the API must be always on (backend.md §6). To be decided together with deployment. |
