---
name: ios-ui
description: Use when building or styling any SwiftUI screen, view, or component on iOS — layout, navigation, tab bars, toolbars, buttons, sheets, loading states, colors, tint, typography, dark mode, accessibility, or anything involving Liquid Glass or HIG compliance.
---

# iOS UI — SwiftUI + Liquid Glass (iOS 26+/27)

> iOS 27 APIs verified against June–July 2026 beta documentation; re-verify at GA.

## The layer model (the one rule that governs everything)
- Liquid Glass lives ONLY in the floating control/navigation layer (tab bars, toolbars, buttons, FABs). NEVER in the content layer (cards, list rows, backgrounds). Why: Apple's #1 documented first-year adoption mistake; distracts from content and tanks performance.
- Content-layer differentiation uses standard materials (.ultraThinMaterial….thickMaterial), never glassEffect.
- Never stack glass on glass. Never put custom backgrounds behind/inside bars, sheets, popovers — they break glass and the scroll edge effect.
- Forms (e.g. a login screen) are content layer: glass only on the primary actions.

## System first
- Use standard SwiftUI components; they adopt Liquid Glass automatically with the iOS 26 SDK. Custom glass is the exception, not the norm.
- Buttons: .buttonStyle(.glass) / .buttonStyle(.glassProminent). Never hand-roll glassmorphism with materials + shadows (breaks in dark mode and accessibility modes).
- Legibility under bars: scrollEdgeEffectStyle(_:for:) — never custom gradient/blur hacks.
- Multiple or morphing custom glass elements MUST share one GlassEffectContainer (each standalone effect costs ~3 offscreen textures).
- Glass variants: .regular for anything with meaningful text; .clear ONLY over rich media, with ~35% dark dim layer over bright content.
- Custom glass buttons need .contentShape(...) covering the whole glass shape (default hit area is label-only).
- Toolbar overflow and priority (iOS 27): use visibilityPriority(_:) and ToolbarOverflowMenu for secondary actions — never hand-roll an overflow menu. Tab bars may use Tab(role: .prominent) for one key action.
- Don't hard-code width assumptions — iPhone apps are resizable windows on iOS 27; layouts must survive narrow and wide sizes.
- Dim custom chrome when the window is inactive using the appearsActive environment value (system components do this themselves).

## Loading states
- Use the system `ProgressView` for determinate progress and for in-control waits (e.g. a button's in-flight spinner, which should keep the button's footprint).
- Prefer a loader plus a status line over a bare animation. Decorative motion carries no information, so hide it with `.accessibilityHidden(true)`; where the wait *is* the state, the sentence is the only accessible content.
- Avoid pulsing skeleton rows that show plausible placeholder values — a plausible-looking value on screen is a fabricated one.
- Custom loaders should handle Reduce Motion themselves (freeze on a static frame and stop redrawing), so callers never need their own check.
- Custom `Canvas` animations: derive phase from **absolute time**, not a per-view start date, or two instances on screen drift apart; and a `Canvas` **clips to its bounds**, so anything entering from outside the frame needs padding to arrive in.

## HIG tokens
- Typography: system text styles only (.largeTitle….caption); Dynamic Type must work; no Ultralight/Thin/Light weights; section headers use title-style capitalization (ALL CAPS is dead).
- Color: semantic system colors; custom colors ship light + dark + increased-contrast variants of each; prominence on glass via Glass.tint(_:) or .glassProminent, never painted backgrounds; never color as the only signal.
- **A tint is a surface, not a label.** Any control the system paints *white* content onto — a `Toggle`'s knob, a `swipeActions` button's title — needs a surface-role color as its tint, never a label/foreground color. Label colors invert with appearance, the system's white does not, so the control vanishes in exactly one appearance — and if the two colors share a light value, light-mode review won't catch it. When you must tint such a control, pair it with an explicit contrasting `.foregroundStyle` and check contrast in light, dark, and both increased-contrast appearances.
- **Tint is inherited, so "I set no tint" is not a neutral choice.** A `.tint` on a `TabView` (e.g. for tab bar glyphs) propagates into every tab's content *and* every sheet presented from it. A content-layer control that draws its tint gets that color unless it pins its own.
- Layout: content flows edge-to-edge UNDER bars; respect safe areas; don't hard-code control metrics; shapes concentric with container corners (ConcentricRectangle).
- Dark mode: no in-app appearance toggle; contrast ≥4.5:1 (7:1 preferred for small text).

## Accessibility (non-negotiable)
- Custom glass must be tested under Reduce Transparency, Increase Contrast, and Reduce Motion; honor accessibilityReduceTransparency / accessibilityReduceMotion in custom effects (fade, don't morph).
- Every icon-only toolbar/menu item gets an accessibility label. Don't mix text and icon items in one shared-background toolbar group.

## References
- references/glass-api.md — verified glass API signatures (iOS 26 core + iOS 27 additions: Glass, glassEffect, GlassEffectContainer, morphing, tab bar/toolbar/search APIs).
- references/hig-foundations.md — typography/color/layout/dark-mode details, app icons (Icon Composer).
- references/pitfalls.md — first-year mistakes checklist, known glitches with workarounds, framework quirks, deprecations.
