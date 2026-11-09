# Event's World Design System · Salar Cristal

* **Version:** 1.0 · **Date:** Oct 28, 2026 · **Status:** Final decision
* **Decision:** [ADR-0008](../architecture/adr/0008-salar-cristal-design-system.md). Retains the technical foundation of [ADR-0007](../architecture/adr/0007-styling-and-components.md): Tailwind CSS v4 + Angular CDK + custom components.
* **Requirements:** NFR-01.1 and NFR-01.2 from [requirements.md](../requirements.md).
* **Visual reference:** "Salar Cristal" option from the [style lab](style-lab.html), with Outfit typography and soft corners.

This document serves as the **single source of truth** for colors, typography, surfaces, and visual rules across **all views** of the application: public, attendee, authentication, organizer, and Super Admin, in both light and dark mode.

---

## 1. Concept

**Salar Cristal** is inspired by the Salar de Uyuni: an expanse of blue sky reflected on the ground and the coral light of dusk.

* **Surfaces:** The entire interface floats on **translucent glass panels** over a sky backdrop with blue and coral highlights.
* **Brand:** Blue conveys confidence in payments, while coral appears exclusively as a selective accent.
* **Balance:** Glass is prominent and visible, but all text is verified against the **worst-case** contrast scenario (§6).

---

## 2. Colors

### 2.1 Semantic Tokens

| Token | Light | Dark | Usage |
|---|---|---|---|
| `bg` | `#E6F0FB` | `#0A121C` | Base color of the sky backdrop (§4) |
| `surface` | `#FFFFFF` | `#111C29` | Solid surface; foundation for glass and "reduce transparency" mode |
| `surface-2` | `#EBF1F7` | `#182638` | Secondary solid surface (neutral states) |
| `border` | `#D3DEE9` | `#28394F` | Subtle border on solid surfaces |
| `control-border` | `#5A6B80` | `#7F95B0` | Border for input fields and controls (≥ 3:1) |
| `text` | `#0E1B2A` | `#EAF2FA` | Primary text |
| `muted` | `#3E5367` | `#9AB0C6` | Secondary text |
| `primary` | `#0B63B6` | `#5AB0FF` | Primary button, links, focus, active selection |
| `primary-hover` | `#09528F` | `#84C4FF` | Hover and pressed states |
| `on-primary` | `#FFFFFF` | `#04192E` | Text on `primary` |
| `accent` | `#E5533B` | `#FF8A73` | Selective coral accent: highlighted chart data, avatar. **Never** for buttons, backgrounds, or text |
| `chart` | `#0B63B6` | `#5AB0FF` | Chart bars and series |

### 2.2 Statuses (Payments and Events)

These are fixed: color meaning never changes across screens. They are always presented with **text and a colored dot**, never through color alone (WCAG 1.4.1). Badges are **solid**, not glass.

| Status | Light (text / background) | Dark (text / background) |
|---|---|---|
| Pending · Registration closed | `#8A3B0A` / `#FEF3C7` | `#FCD34D` / `#3A2A08` |
| Approved · Published | `#166534` / `#DCFCE7` | `#86EFAC` / `#0B3B22` |
| Under review | `#5B21B6` / `#EDE9FE` | `#C4B5FD` / `#2E1A5E` |
| Rejected · Suspended · Cancelled | `#991B1B` / `#FEE2E2` | `#FCA5A5` / `#4A1515` |
| Completed (info) | `#1E40AF` / `#DBEAFE` | `#93C5FD` / `#172554` |
| Unpaid · Draft (neutral) | `muted` / `surface-2` | `muted` / `surface-2` |

---

## 3. Glass

### 3.1 Levels

| Level | Where | Light | Dark |
|---|---|---|---|
| **Panel** | Cards, tables, tiles, sidebar, detail sheet, fields, chips, secondary buttons | `rgba(255, 255, 255, .50)` | `rgba(17, 28, 41, .45)` |
| **Bar** | Top bar, sticky action bars (registration, actions), mobile bottom bar | Same as panel | Same as panel |
| **Overlay** | Modals, drawers (☰ menu), popup menus | `rgba(255, 255, 255, .72)` + dimmed backdrop `rgba(10, 18, 28, .35)` | `rgba(17, 28, 41, .72)` + dimmed backdrop `rgba(0, 0, 0, .55)` |

### 3.2 Common Properties

| Property | Light | Dark |
|---|---|---|
| Blur | `blur(22px) saturate(180%)` | `blur(22px) saturate(160%)` |
| Border (1 px) | `rgba(255, 255, 255, .72)` | `rgba(255, 255, 255, .16)` |
| Top highlight | `inset 0 1px 0 rgba(255, 255, 255, .85)` | `inset 0 1px 0 rgba(255, 255, 255, .14)` |
| Shadow | `0 10px 30px rgba(14, 27, 42, .14)` | `0 10px 30px rgba(0, 0, 0, .45)` |

### 3.3 What is NOT Glass

* **Primary button:** Solid `primary`, with a shadow cast in its own color (`0 6px 18px` of `primary` at 35%).
* **Status badges:** Solid (§2.2).
* **Event covers and receipts:** Images; never blurred.
* **Long-form text** (terms and privacy, PUB-07): Uses an overlay-level panel due to its higher opacity.

---

## 4. Sky Backdrop

Placed behind the entire application, fixed in place (does not scroll with content), allowing the glass to reveal motion during scrolling.

```css
/* Light */
--backdrop:
  radial-gradient(42% 30% at 10% 6%,  rgba(11, 99, 182, .45), transparent 70%),
  radial-gradient(40% 30% at 94% 22%, rgba(229, 83, 59, .42), transparent 72%),
  radial-gradient(50% 34% at 26% 64%, rgba(90, 176, 255, .55), transparent 72%),
  radial-gradient(40% 28% at 86% 92%, rgba(229, 83, 59, .30), transparent 70%),
  linear-gradient(180deg, #E6F0FB, #F6EEEC);

/* Dark */
--backdrop:
  radial-gradient(42% 30% at 10% 6%,  rgba(90, 176, 255, .45), transparent 70%),
  radial-gradient(40% 30% at 94% 22%, rgba(255, 138, 115, .30), transparent 72%),
  radial-gradient(50% 34% at 26% 64%, rgba(11, 99, 182, .55), transparent 72%),
  radial-gradient(40% 28% at 86% 92%, rgba(255, 138, 115, .22), transparent 70%),
  linear-gradient(180deg, #08172A, #0A121C);
```

* Rendered on a `body::before` pseudo-element with `position: fixed`, as `background-attachment: fixed` is unreliable in iOS Safari.
* Color gradients must not overlap beyond design specifications: worst-case contrast in §6 was calculated against the single most intense gradient spot.

---

## 5. Typography, Spacing, and Shapes

### 5.1 Typography

* **Single font family:** **Outfit** (Google Fonts), with fallback stack `'Outfit', system-ui, -apple-system, 'Segoe UI', sans-serif`. Only weights 400, 600, and 700 are loaded, using `display=swap`.
* **Weights:**

  | Weight | Usage |
  |---|---|
  | 400 | Body text |
  | 600 | Labels, buttons, names, table headers |
  | 700 | Headings and metrics |

  No font weight exceeds 700. The style lab displayed some heavier headings; in implementation they are capped at 700, adhering to the typography skill guideline.
* **Minor Third Scale (1.2):**

| Role | Size | Weight | Line Height | Examples |
|---|---|---|---|---|
| Meta | 12 px (0.75 rem) | 400 | 1.4 | Timestamps, IDs under names, helper text |
| Secondary | 14 px (0.875 rem) | 400 / 600 | 1.5 | Table cells, field labels, badges |
| Body | 16 px (1 rem) | 400 | 1.55 | Descriptions, input field values, buttons |
| Subtitle | 19.2 px (1.2 rem) | 600 | 1.25 | Price in bar, card titles |
| Page title | 23 px (1.44 rem) | 700 | 1.2 | "Payment review", KPIs |
| Event title | 27.6 px (1.728 rem) | 700 | 1.15 | PUB-02 |
| Display | 33.2 px (2.074 rem) | 700 | 1.1 | Home (PUB-01), desktop |

  No text falls below 12 px. Numerical values use `font-variant-numeric: tabular-nums`.

### 5.2 Spacing

* **4/8 px rhythm:** `4, 8, 12, 16, 24, 32, 48`.
* 16 px lateral page margins on mobile.
* 12–16 px gap between panels.

### 5.3 Soft Corners

| Token | Value | Usage |
|---|---|---|
| `radius-sm` | 6 px | Fields, compact badges |
| `radius-md` | 12 px | Buttons, tiles, square chips |
| `radius-lg` | 18 px | Panels, cards, detail sheets, modals |
| `radius-pill` | 999 px | Filter chips and status badges |

### 5.4 Focus, Icons, and Motion

* **Focus:** 2 px ring in `primary` with a 2 px separation from the element (`outline-offset`), clearly visible on both glass and solid surfaces.
* **Icons:** Lucide, 20 px (or 24 px in the mobile navigation bar), with a 1.75 stroke. No decorative icons inside form fields; only the magnifying glass in search.
* **Motion:** Color and opacity transitions only, restricted to 150 ms. **Never** animate an element with a blur filter. With `prefers-reduced-motion`, all transitions become instantaneous.

---

## 6. Accessibility and Contrast

Contrast over glass depends on underlying elements, so every pair is verified against the **worst-case scenario**: glass positioned over the darkest backdrop patch (in light mode) or the brightest patch (in dark mode).

| Pair | Light | Dark | Required |
|---|---|---|---|
| Text on glass (worst case) | 11.85 | 9.77 | 4.5 |
| Secondary text on glass (worst case) | 5.43 | 4.94 | 4.5 |
| Field border on glass (worst case) | 3.72 | 3.59 | 3 |
| Large heading directly on backdrop (worst case) | 7.68 | 6.48 | 3 |
| Primary button | 6.04 | 7.68 | 4.5 |
| Statuses (§2.2) | 6.49–7.57 | 7.85–9.62 | 4.5 |

**Rules:**

1. All standard-sized text must be placed on glass or on a solid surface. Only large headings may sit directly on the background.
2. Under `prefers-reduced-transparency: reduce`, and on browsers without `backdrop-filter` support (`@supports not`), all glass falls back to solid `surface` with a `border` stroke.
3. Interactive touch targets measure at least **44 × 44 px** on touch devices (`pointer: coarse`).
4. Any adjustment to color, opacity, or the background requires recalculating the worst-case scenario in the style lab.

---

## 7. Performance (Crucial for Mid- and Low-End Mobile Devices)

* **Apply blur to containers, not individual elements:** The table is a glass panel; individual rows are not.
* **Limit:** At most **6 blurred layers** visible simultaneously on mobile.
* **No animated resize or reposition operations** on glass panels, and never apply permanent `will-change`.
* **Mandatory release testing:** Prior to every release, test on a low-end Android device using Chrome. If scroll performance drops below ~50 fps, blur must be reduced to `blur(14px)` on that screen.

---

## 8. Implementation in Angular (Tailwind v4)

`src/styles.css` (frontend.md §7):

```css
@import "tailwindcss";
@custom-variant dark (&:where(.dark, .dark *));

:root {
  --ew-bg: #E6F0FB; --ew-surface: #FFFFFF; --ew-surface-2: #EBF1F7; --ew-border: #D3DEE9; --ew-control-border: #5A6B80;
  --ew-text: #0E1B2A; --ew-muted: #3E5367; --ew-primary: #0B63B6; --ew-primary-hover: #09528F; --ew-on-primary: #FFFFFF; --ew-accent: #E5533B;
  --ew-glass: rgba(255, 255, 255, .50); --ew-glass-overlay: rgba(255, 255, 255, .72);
  --ew-glass-border: rgba(255, 255, 255, .72); --ew-glass-highlight: rgba(255, 255, 255, .85);
  --ew-glass-shadow: 0 10px 30px rgba(14, 27, 42, .14); --ew-blur: blur(22px) saturate(180%);
  /* + --ew-backdrop (§4) and statuses (§2.2) */
}
.dark {
  --ew-bg: #0A121C; --ew-surface: #111C29; --ew-surface-2: #182638; --ew-border: #28394F; --ew-control-border: #7F95B0;
  --ew-text: #EAF2FA; --ew-muted: #9AB0C6; --ew-primary: #5AB0FF; --ew-primary-hover: #84C4FF; --ew-on-primary: #04192E; --ew-accent: #FF8A73;
  --ew-glass: rgba(17, 28, 41, .45); --ew-glass-overlay: rgba(17, 28, 41, .72);
  --ew-glass-border: rgba(255, 255, 255, .16); --ew-glass-highlight: rgba(255, 255, 255, .14);
  --ew-glass-shadow: 0 10px 30px rgba(0, 0, 0, .45); --ew-blur: blur(22px) saturate(160%);
}

@theme inline {
  --font-sans: 'Outfit', system-ui, -apple-system, 'Segoe UI', sans-serif;
  --radius-sm: 6px; --radius-md: 12px; --radius-lg: 18px;
  --color-bg: var(--ew-bg); --color-surface: var(--ew-surface); --color-surface-2: var(--ew-surface-2);
  --color-border: var(--ew-border); --color-control: var(--ew-control-border);
  --color-text: var(--ew-text); --color-muted: var(--ew-muted);
  --color-primary: var(--ew-primary); --color-primary-hover: var(--ew-primary-hover); --color-on-primary: var(--ew-on-primary);
  --color-accent: var(--ew-accent);
}

@utility glass {
  background: var(--ew-glass); border: 1px solid var(--ew-glass-border);
  backdrop-filter: var(--ew-blur); -webkit-backdrop-filter: var(--ew-blur);
  box-shadow: inset 0 1px 0 var(--ew-glass-highlight), var(--ew-glass-shadow);
}
@utility glass-overlay {
  background: var(--ew-glass-overlay); border: 1px solid var(--ew-glass-border);
  backdrop-filter: var(--ew-blur); -webkit-backdrop-filter: var(--ew-blur);
}
@media (prefers-reduced-transparency: reduce) {
  .glass, .glass-overlay { background: var(--ew-surface); backdrop-filter: none; -webkit-backdrop-filter: none; border-color: var(--ew-border); }
}
@supports not (backdrop-filter: blur(1px)) {
  .glass, .glass-overlay { background: var(--ew-surface); border-color: var(--ew-border); }
}

body { font-family: var(--font-sans); color: var(--ew-text); background: var(--ew-bg); }
body::before { content: ""; position: fixed; inset: 0; z-index: -1; background: var(--ew-backdrop); }
```

The "Tokens of selected combination" section from the style lab (Salar Cristal option) generates the exact `:root` and `.dark` values.

---

## 9. Component Breakdown (`shared/ui`)

| Component | Surface | Details |
|---|---|---|
| Top bar and mobile bottom bar | `glass` (bar) | Fixed; glass bottom or top border |
| Panel, card, KPI, tile | `glass` | `radius-lg` (tiles: `radius-md`) |
| Table and list | Single `glass` for the entire block | Dividers with `glass-border`. Selected row: `primary` at 14% and a 3 px lateral indicator in `primary` |
| Text field and native select | `glass` | Border `control-border`; focus with `primary` ring |
| Primary button | Solid `primary` | `radius-md`, weight 600, colored shadow |
| Secondary button | `glass` | Font color `text` |
| Approve, Observe, Reject | Solid status color | Never glass |
| Filter chip | `glass`; active item is solid `primary` | `radius-pill` |
| Status badge | Solid (§2.2) | `radius-pill`, colored dot, and text |
| Modal and drawer (☰) | `glass-overlay` + dimmed backdrop | Trapped focus; Escape closes (CDK `Dialog`) |
| Toast | Solid `text` on `bg` | No glass, ensuring immediate readability at all times |
| Event cover | Image | Detail sheet overlaps by 28 px |
| Receipt (ORG-08) | Image on solid `surface` | No blur; descriptive alt text mandatory |

---

## 10. Exceptions to UI/UX Skills

This design system intentionally diverges from a rule in `vectorlab-ux-skills:anti-slop`: glass in product UI (S3, S4, and S15). This is an explicit **brand decision** made by the team, recorded in ADR-0008. It satisfies the conditions outlined by the same skill when using glass:

* a visible boundary;
* an opaque fallback;
* sufficient contrast, validated against the worst-case scenario.

All other skill rules remain active: a single interactive accent, 3 weights, 12/14/16 px font sizes, and unambiguous action verbs on buttons.