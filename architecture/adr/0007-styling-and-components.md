# ADR-0007: Tailwind CSS + Angular CDK + Own Component Library

* **Status:** Accepted. Decision 6 (glassmorphism in moderation) is superseded by [ADR-0008](0008-salar-cristal-design-system.md).
* **Date:** 26 Oct 2026
* **Replaces:** the Tailwind + Angular Material approach in requirements v0.3, NFR-01.1.

## Context

Requirements v0.3 asked for Tailwind CSS for layout and the glassmorphism style, plus Angular Material for complex components. When we designed the frontend (frontend.md) we reviewed what the wireframes actually need:

* **Simple components** (most of them): buttons, badges, cards, inputs, toggles, progress bars, KPI cards, filter chips, stepper.
* **Complex components:** date-time picker, dialogs, drawers and side navigation, data tables with selection and pagination, drag and drop (ORG-04), file upload with progress, toasts, image zoom (ORG-08) and charts (ORG-05, ADM-01).

The decision criteria, in order:

1. mobile first, with light public pages (also server-rendered, ADR-0006);
2. a look of our own;
3. dark mode;
4. accessibility;
5. speed for a small team;
6. long-term maintenance.

## Decision

1. **Tailwind CSS v4 is the only styling system.** There is no second theming system from a component library.
2. **Design tokens in three levels**, as CSS custom properties read by Tailwind's `@theme`:
   * *primitive* (palette);
   * *semantic* (`--color-primary`, `--color-surface`, `--color-status-pending`);
   * *component* (`--button-radius`).

   Dark mode redefines only the semantic tokens, under a `dark` class on `<html>`.
3. **Angular CDK** (maintained by the Angular team) provides behavior and accessibility: `Dialog`, `Overlay`, `Menu`, `A11y` (focus trap, live announcer), `DragDrop`, `Table`, `Clipboard` and `Layout` (breakpoints).
4. **Own component library in `shared/ui`**, mirroring the wireframe component library "00". Each component is standalone, uses Tailwind and the tokens, and gets a component test.
5. **Not built by us:**
   * **Date-time and simple selects:** native `<input type="datetime-local">` and `<select>`. Phones open the operating system's picker, the most usable and accessible option.
   * **Charts:** Chart.js, loaded only by the dashboard routes.
   * **Icons:** Lucide (`lucide-angular`), imported per icon.
6. **Glassmorphism in moderation:**
   * Applied only on decorative surfaces (header, cards over cover images, dialogs).
   * Forms, tables and long text use solid surfaces with WCAG AA contrast.
   * `@media (prefers-reduced-transparency: reduce)` and low-power fallbacks replace the blur with solid surfaces.
7. **Status colors are fixed tokens** (Pending, Approved, In observation, Rejected, event states). Status is always shown with text as well, never by color alone.

## Consequences

* ✅ The UI matches the wireframes exactly, with one styling system and no fight against a library's look.
* ✅ The lightest bundle for mobile public pages; component code is tree-shaken and SSR-friendly.
* ✅ No dependency on a third-party component vendor; the CDK is maintained by the Angular team.
* ⚠️ Initial cost of about 1 to 2 weeks to build the ~15 components of `shared/ui` before features move fast.
* ⚠️ Accessibility is our responsibility. The CDK covers focus, keyboard and screen-reader behavior for overlays; each component's tests must cover labels, roles and keyboard use.
* ⚠️ Native date-time inputs look slightly different in each browser on desktop. This is acceptable for the organizer panel; a custom picker can be built later on CDK `Overlay` if needed.

## Alternatives Considered

| Alternative | Why it was not chosen |
|---|---|
| Tailwind + Angular Material (requirements v0.3) | Two styling systems that can clash; Material's strong Google look fights the custom glassmorphism design; no file upload, charts or image zoom. |
| Tailwind + PrimeNG | Fastest coverage of complex components, but heavier on mobile, its own look to override, and dependency on a third-party vendor with a history of breaking major versions. A valid fallback if speed becomes the top priority. |
| Angular Material alone | A generic look and no utility-first layout. |
