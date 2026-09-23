# Pitfalls, deprecations, and iOS 27 additions

## First-year adoption mistakes (eight, numbered)

1. **Keeping opaque custom backgrounds on toolbars/tab bars/sheets.** This is the most common migration bug — a leftover custom background silently breaks the glass rendering and the scroll edge effect. Remove all custom backgrounds behind or inside navigation elements.
2. **Sprinkling `glassEffect` across content-layer views.** This is Apple's #1 documented don't. Glass belongs only in the floating control/navigation layer — see SKILL.md's layer model.
3. **Using many standalone glass effects without a `GlassEffectContainer`.** Each standalone glass effect costs roughly 3 offscreen render textures; group related glass elements in one container to avoid the multiplied cost.
4. **Placing text over glass over busy imagery without `.regular` glass or a dimming layer.** This breaks legibility and crowds tap targets. Use `.regular` for anything with meaningful text, and add the ~35% dark dim layer over bright content when using `.clear`.
5. **Recreating glassmorphism with `.ultraThinMaterial` + shadows.** This hand-rolled approach breaks under dark mode and accessibility modes, and has no adaptivity to Reduce Transparency / Increase Contrast. Use `.buttonStyle(.glass)` / `.buttonStyle(.glassProminent)` instead.
6. **Hit-testing gap on custom glass buttons.** Custom glass buttons register taps only on the label by default, not the full glass shape. Fix with `.contentShape(...)` covering the entire glass shape.
7. **Known glitches — test per point release:**
   - `rotationEffect` applied to glass distorts the shape; workaround is to use `UIGlassEffect` from UIKit for rotated glass elements.
   - Menu morphing has glitches across iOS 26.0/26.1.
   - Menu placed inside a `GlassEffectContainer` broke morphing on 26.1.
8. **Changing everything at once.** Nielsen Norman Group and Slack's own staged rollout (spread over months) both argue against combining minimize-on-scroll + collapsing search + relocated navigation in a single release. Stage large navigation/chrome changes.

### Recommended patterns

- Prefer bottom navigation.
- Keep content flowing under glass chrome (see the layout section in `hig-foundations.md`).
- Prefer standard system controls over custom ones.
- Treat Liquid Glass as a layout/structure decision, not a reskin.
- Use plain (non-enclosed) SF Symbol variants inside glass, not circle/square-enclosed variants.
- Liquid Glass v2's refreshed visual tokens and adaptive contrast apply automatically on rebuild — keeping custom glass minimal lets the v2 restyle apply for free, with no extra work.
- The system now exposes a user-facing transparency/intensity slider; this is one more reason custom glass should stay minimal — Apple's system components already respond correctly to the slider, hand-rolled glass will not.
- iPad windows dim automatically when inactive; apply the same treatment to custom chrome via the `appearsActive` environment value (see SKILL.md's "System first" rules) — system components already do this themselves. (`appearsActive` itself is available back to iOS 18; the automatic system dimming behavior is new iOS 27 v2 styling.)
- Use `.reorderable()` / `.reorderContainer(for:)` for list reordering instead of hand-rolled drag gestures.
- Use `.swipeActionsContainer()` for swipeable row actions instead of a custom implementation.
- Prefer item-driven presentation: `alert(item:)` / `confirmationDialog(item:)`.

## Framework quirks (measured on device, not in Apple docs)

1. **`List` ignores scroll-target modifiers.** `.scrollTargetLayout()`, `.scrollTargetBehavior(.viewAligned)`, and `.scrollPosition(id:)` all have no effect inside a `List` on iOS 27.0. A row can't be made a scroll target without abandoning `List`, and `List` is what supplies `swipeActions`. Paged/snapping scroll and swipe-to-delete are mutually exclusive today: use a `ScrollView` if snapping matters more than row actions, otherwise ship free scrolling.
2. **`.accessibilityIdentifier` on a container overrides every descendant's.** A container carrying its own identifier makes `app.buttons["child.id"]`-style XCUI lookups fail to find anything, silently. Put identifiers on leaves, or query by label — never both on the same subtree.
3. **Content scrolls *under* a `safeAreaInset` region — correct, and a trap for bare text.** A bottom bar holding an unbacked `Label` let scrolled rows show straight through its glyphs; a primary button in the same bar hid the identical problem only because it happened to be opaque. Give the bar an explicit opaque background.
4. **`tabViewBottomAccessory` does not grow to fit its content.** It caps at roughly two lines in both placements, at every text size. Long copy must be capped with `.dynamicTypeSize(...)` or it truncates — treat that cap as a real accessibility trade-off to declare, not a free fix.
5. **A grouped `List` uppercases `Section` headers by default.** The absence of `.uppercased()` in source does not mean the rendered header is title case. `.textCase(nil)` is required to get the title-style capitalization this doc already calls for (see `hig-foundations.md`).
6. **A `TabView`-level `.tint` overrides `role: .destructive` on a swipe action.** Tinting the tab bar ink rendered a swipe-to-delete panel black instead of red. Set `.tint(.red)` on the destructive `Button` itself, not on the `TabView`.
7. **A modifier applied around a multi-row `ForEach` body silently kills `swipeActions` on every row.** No warning, no error — the actions just stop existing. Keep row builders bare; apply modifiers inside the row, not around it. Invisible in review, only caught by driving the UI.
8. **`@Observable` can't use `didSet` on its own stored properties.** Observation only fires when reads go through tracked storage. Back the property with a private stored value plus a computed accessor instead.
9. **`.labelsHidden()` can drop a control's label from the accessibility tree.** A visible title sitting next to the control does not guarantee VoiceOver announces what the control chooses between. Add an explicit `.accessibilityLabel`, which survives `.labelsHidden()`.

## Deprecated / discouraged

- Custom material hacks for chrome — `.ultraThinMaterial`/`UIBlurEffect` behind bars, custom toolbar backgrounds. Remove these; standard materials are acceptable for content-layer use only, never for chrome.
- Custom glassmorphism buttons — replace with `.buttonStyle(.glass)` / `.buttonStyle(.glassProminent)`.
- Custom scroll-fade gradients — replace with `scrollEdgeEffectStyle(_:for:)`.
- Hard-coded metrics, ALL-CAPS section headers, hard-coded colors, and in-app dark-mode toggles are all discouraged (see `hig-foundations.md`).
- Flat single-image app icons — replace with layered Icon Composer icons.
- `UIDesignRequiresCompatibility` — removed in Xcode 27, so it is no longer an opt-out; treat Liquid Glass as mandatory. The App Store is expected to require a modern-SDK build starting around April 2027.
- No Liquid Glass API deprecations: the iOS 26 glass APIs documented in `glass-api.md` remain a stable foundation under iOS 27 — nothing there has been removed.
- UIKit apps: scene lifecycle becomes mandatory (TN3187). Pure SwiftUI `App` lifecycle apps are unaffected.
