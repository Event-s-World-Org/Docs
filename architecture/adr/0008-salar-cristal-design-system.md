# ADR-0008: Salar Cristal as the Final Visual Design System

* **Status:** Accepted
* **Date:** 28 Oct 2026
* **Supersedes:** decision 6 of [ADR-0007](0007-styling-and-components.md) (glassmorphism in moderation). The rest of ADR-0007 still applies: Tailwind CSS v4 + Angular CDK + own component library.

## Context

After ADR-0007 the team explored the visual identity in the style lab (`design/style-lab.html`). They compared:

* four colour palettes (Salar, Aguayo, Yungas, Neón);
* two full proposals built with the installed UI/UX skills (Taquilla, Altiplano);
* two intense-glass variants (Salar Cristal, Illimani).

All options remain comparable in the style lab (`design/style-lab.html`), with a live contrast check. The team chose **Salar Cristal** for all views, with the **Outfit** typeface and **soft corners**, in light and dark modes.

## Decision

1. **Visual style:** Salar Cristal, specified in [design/design-system.md](../../design/design-system.md), the single source of visual truth:
   * The Salar palette (sky blue `#0B63B6` / `#5AB0FF`, coral accent `#E5533B` / `#FF8A73`).
   * A sky backdrop with blue and coral reflections behind the whole app.
   * **Intense glassmorphism** on every panel, bar and control: 50 % white in light mode, 45 % night blue in dark mode, `blur(22px)`.
2. **Typography:** Outfit only.
   * Weights 400, 600 and 700. Titles are capped at 700, slightly lighter than the lab preview.
   * Minor Third scale over fixed 12 / 14 / 16 px tiers.
3. **Corners:** soft. 6 px for fields, 12 px for buttons, 18 px for panels, pills for chips and status badges.
4. **Not glass:** the primary button, status badges, toasts, event covers and payment vouchers.
5. **Accessibility guardrails** (they are part of the decision, not optional):
   * Every text/background pair passes WCAG AA in the **worst case** of the glass over the backdrop.
   * Glass becomes solid under `prefers-reduced-transparency` and when `backdrop-filter` is unsupported.
   * Touch targets are at least 44 px.
6. **Performance guardrails:**
   * Blur is applied to containers, never to each row or item.
   * At most about 6 blurred layers on screen on mobile.
   * Blurred elements are never animated.
   * Every release is tested on a low-end Android phone.

## Consequences

* ✅ A distinctive, modern identity tied to a Bolivian landmark, consistent across all views and both themes.
* ✅ Exact tokens already exist (lab token block plus design-system.md §8), ready for `styles.css`.
* ⚠️ This is a deliberate exception to the `anti-slop` skill, which discourages glass in product UI. It is acceptable only while the guardrails in points 5 and 6 hold.
* ⚠️ Glass costs GPU and battery on low-end phones, which are common in the region. Mitigations: the performance guardrails, plus the option to reduce blur to `blur(14px)` on a screen that drops frames.
* ⚠️ Contrast depends on the backdrop. Any change to colours, opacity or the backdrop requires recomputing the worst case in the lab.

## Alternatives Considered

| Alternative | Why it was not chosen |
|---|---|
| Glass in moderation (ADR-0007, decision 6) | The team preferred a more visible, distinctive glass identity. |
| Taquilla (ink and paper, colour only for status) | Clearest for payment review, but the team wanted a stronger brand colour. |
| Altiplano (sunset orange, wide poster type) | Festive, but the team preferred the calmer, trustworthy blue of Salar. |
| Illimani (frosted glass over a mountain landscape) | Close in spirit, but its borderless fields fail WCAG 1.4.11 and its landscape does not suit every type of event. |
