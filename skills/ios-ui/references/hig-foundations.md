# HIG foundations reference

## Typography

- Typefaces are SF Pro and New York, both variable fonts with dynamic optical sizing.
- Body text defaults to 17pt; the minimum usable text size is 11pt.
- Avoid Ultralight, Thin, and Light font weights.
- Use system text styles (`.largeTitle` through `.caption`) rather than fixed point sizes — Dynamic Type scaling is then automatic.
- SF Symbols match SF font weights — prefer symbols over custom iconography, and align symbol weight with the adjacent text weight.
- Section headers use title-style capitalization. ALL CAPS section headers are no longer the HIG-correct pattern.

## Color

- Use semantic/dynamic system colors; never hard-code documented color values.
- Every custom color must ship a light variant, a dark variant, and an increased-contrast variant of each — this is required for correct Liquid Glass adaptivity, not optional polish.
- Add prominence on top of glass via `Glass.tint(_:)` or `.glassProminent`, never by painting a background behind the glass.
- The `.clear` glass variant is highly translucent and is used ONLY over rich media (photo/video); when using it over bright content, add a ~35% dark dimming layer so foreground content stays legible.
- Vibrant colors read well on top of materials.
- Never use color as the only signal for state or meaning (pair with icon/text/shape too).

## Layout

- Respect safe areas.
- Content flows edge-to-edge UNDER bars — glass is designed to have content scrolling beneath it; that's what drives the scroll edge effect. Do not inset content to avoid bars.
- Controls should be rounder and larger — prefer the extra-large control size where available.
- Shapes should stay concentric with their container's corners (`ConcentricRectangle` — see `glass-api.md`).
- Lists and forms get taller rows, more padding, and larger corner radii than pre-iOS-26 defaults.
- Don't hard-code control metrics (row height, corner radius, control size) — let the system compute them so they track OS-level changes.
- Sheets use a larger corner radius; half-sheets are inset from the screen edges.

## Dark mode

- There is no in-app appearance setting — respect the system-wide light/dark setting only; do not build a toggle.
- Maintain contrast of at least 4.5:1; 7:1 is preferred for small text.
- Test all custom UI with Increase Contrast and Reduce Transparency enabled, not just in default light/dark.

## Accessibility

- Honor `accessibilityReduceTransparency` and `accessibilityReduceMotion` in any custom effect: fade rather than use axis transitions, and don't animate blurs when Reduce Motion is on.
- Every icon-only toolbar item needs an accessibility label.
- Don't mix text-only and icon-only items within one shared-background toolbar group (this also degrades tap-target clarity — see `pitfalls.md`).

## App icons

- Icons are layered: a background layer plus foreground layer(s). The system applies specular highlights, refraction, and shadow automatically at render time — do not bake these effects into the artwork.
- Build icons with Icon Composer (ships with Xcode): define the background (solid or gradient fills are preferred over photographic backgrounds), group the layers, and annotate default/dark/clear/tinted variants.
- Keep layers square, unmasked, and centered. Prefer vector art. Avoid text and fine detail that won't survive the system's real-time rendering effects.
