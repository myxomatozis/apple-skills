# Glass API reference (iOS 26.0 core + iOS 27 additions, verified)

Do not add or infer signatures beyond what is listed here.

## Core API surface

```swift
struct Glass  // .regular, .clear, .identity; .tint(_:), .interactive(_:)
// .identity renders NO glass (conditional/animated removal)

func glassEffect(_ glass: Glass = .regular, in shape: some Shape = DefaultGlassEffectShape()) -> some View
// default shape is Capsule. No isEnabled param (early-beta overload removed).
// Apply AFTER other appearance modifiers.

GlassEffectContainer(spacing: 40.0) { ... }
// spacing controls when nearby shapes blend/merge; container spacing > interior stack spacing => blend at rest.

func glassEffectID(_ id: (some Hashable & Sendable)?, in namespace: Namespace.ID)  // morphing, one container+namespace
func glassEffectUnion(id:namespace:)  // group views into ONE glass shape
GlassEffectTransition: .matchedGeometry (default), .materialize  // via .glassEffectTransition(_:)

.buttonStyle(.glass)  /  .buttonStyle(.glassProminent)   // preferred over hand-rolled glass buttons

func scrollEdgeEffectStyle(_ style: ScrollEdgeEffectStyle?, for edges: Edge.Set)
// .automatic, .soft, .hard — replaces custom gradient/blur header hacks

func backgroundExtensionEffect()  // hero content extends under sidebars/inspectors (mirror+blur)

func tabBarMinimizeBehavior(_ b: TabBarMinimizeBehavior)  // .automatic/.onScrollDown/.onScrollUp/.never (iPhone only)
Tab(role: .search)  // trailing separated search tab
func tabViewBottomAccessory(@ContentBuilder content:)  // floating accessory above tab bar

ToolbarSpacer  // .fixed splits toolbar items into separate glass groups
func searchToolbarBehavior(_ b: SearchToolbarBehavior)  // .minimize etc.

ConcentricRectangle  // shapes concentric with container corners
```

## Notes and semantics

- **Default shape.** `glassEffect(_:in:)` defaults to `DefaultGlassEffectShape()`, which renders as a Capsule when no explicit shape is supplied.
- **No `isEnabled` overload.** An early-beta overload of `glassEffect` that took an `isEnabled` parameter has been removed; do not target it.
- **Modifier ordering.** Apply `.glassEffect(...)` AFTER other appearance modifiers — applying it earlier in the chain produces incorrect visual results.
- **Container spacing and blending.** `GlassEffectContainer(spacing:)` controls when nearby glass shapes blend/merge at rest. If the container's `spacing` value is greater than the interior stack spacing between elements, those elements blend into one shape at rest.
- **Morphing.** Use `glassEffectID(_:in:)` to assign identity for morphing transitions between glass shapes — all morphing participants must share one `GlassEffectContainer` and one `Namespace.ID`.
- **Grouping without morphing.** `glassEffectUnion(id:namespace:)` groups multiple views into a single glass shape without an animated morph.
- **Transitions.** `GlassEffectTransition` offers `.matchedGeometry` (the default) and `.materialize`, set via `.glassEffectTransition(_:)`.
- **Buttons.** `.buttonStyle(.glass)` and `.buttonStyle(.glassProminent)` are preferred over hand-rolled glass buttons built from materials — see `pitfalls.md` for why the hand-rolled approach breaks.
- **Disabled `.glassProminent` desaturates the whole control, not just the modifier's own tint.** Measured on iOS 27.0, screenshotted in both appearances: the slab, the label text, and any nested `ProgressView` all collapse into the system's own washed grey — the enabled treatment is never what's on screen while disabled. A view extension can't see this state; put the disabled-aware styling in a `ViewModifier` reading `@Environment(\.isEnabled)`. Left unhandled, this ships as a near-invisible primary button (pale grey on pale grey, ~1:1 contrast).
- **A descendant's own `.tint` survives that wash.** Probed with `.tint(.red)` on a nested `ProgressView`: it rendered dimmed but clearly red even while the button around it was desaturated. So the color on nested content is load-bearing — point it at a *label*-oriented color, not a background-oriented one; a background color worsens both appearances (pale-on-pale in light, dark-on-mid-grey in dark).
- **Scroll edge legibility.** `scrollEdgeEffectStyle(_:for:)` (values `.automatic`, `.soft`, `.hard`) replaces custom gradient/blur header hacks used to keep content legible under bars.
- **Hero content under chrome.** `backgroundExtensionEffect()` extends hero content under sidebars/inspectors using a mirror-and-blur technique.
- **Tab bar behavior.** `tabBarMinimizeBehavior(_:)` accepts `.automatic`, `.onScrollDown`, `.onScrollUp`, `.never`, and applies to iPhone only.
- **Search tab.** `Tab(role: .search)` renders a trailing, visually separated search tab in a `TabView` — pinned to the trailing edge regardless of the tab's declaration order (measured on iOS 27.0). If a design draws search anywhere else, the role and the drawn order are mutually exclusive; don't reach for the role in that case.
- **Tab bar accessory.** `tabViewBottomAccessory(@ContentBuilder content:)` adds a floating accessory view above the tab bar.
- **Toolbar grouping.** `ToolbarSpacer` with `.fixed` splits toolbar items into separate glass groups (useful for separating icon-only groups from text groups — see the accessibility rule in SKILL.md against mixing text and icon items in one shared-background group).
- **Search toolbar behavior.** `searchToolbarBehavior(_:)` (e.g. `.minimize`) governs how a search field collapses/expands in the toolbar area.
- **Concentric shapes.** `ConcentricRectangle` produces shapes whose corners stay concentric with their container's corners — use it instead of hard-coded corner radii when nesting shapes inside bars, sheets, or cards near glass chrome.

## Search behavior changes (iOS 26)

- On iPhone, search fields move toward the bottom of the screen and slide up with the keyboard.
- `.searchable` used inside a `NavigationStack` renders in the toolbar area.

## iOS 27 toolbar & tab APIs (baseline)

These APIs require iOS 27; gate them with `if #available(iOS 27, *)` if you deploy earlier.

```swift
func visibilityPriority(_ priority: ToolbarItemVisibilityPriority) -> some ToolbarContent
struct ToolbarOverflowMenu<Content>
ToolbarItemPlacement.topBarPinnedTrailing
func toolbarMinimizeBehavior(_ b: ToolbarMinimizeBehavior)  // now also applies to navigation bars, not just tab bars
TabRole.prominent  // Tab(role: .prominent) — marks one key action in a tab bar
```

- **Overflow and priority.** `visibilityPriority(_:)` marks a toolbar item as safe to move into overflow when space is tight; `ToolbarOverflowMenu` renders that overflow — use both instead of hand-rolling an overflow menu.
- **Pinned trailing placement.** `ToolbarItemPlacement.topBarPinnedTrailing` keeps an item pinned to the trailing edge of the top bar regardless of overflow.
- **Minimize behavior extended to nav bars.** `toolbarMinimizeBehavior` (previously tab-bar-only, see `tabBarMinimizeBehavior` above) now also governs navigation bars.
- **Prominent tab.** `TabRole.prominent` marks one tab as the visually prominent, key action in a tab bar.

## Compatibility opt-out

- `UIDesignRequiresCompatibility` (Info.plist key) opted an app out of the Liquid Glass appearance on iOS 26. It is removed in Xcode 27, so apps built with it have no opt-out — treat Liquid Glass as mandatory. See `pitfalls.md`'s Deprecated/discouraged section for the App Store SDK-requirement timeline.
