# Navigation, TabView, structure, and DI reference

## Typed-path router pattern

- **One `NavigationStack` per tab/window.** Each tab (or window) owns its own stack — do not share one `NavigationStack` across tabs.
- **Typed path over `NavigationPath`.** Prefer `[Route]`, where `Route` is a `Hashable` (ideally also `Codable`) enum, over `NavigationPath` whenever you control the destinations. A typed array gives deep-link construction, inspection, and persistence "for free" — `NavigationPath` erases the type information you'd need for those.
- **`navigationDestination` registration.** Register destinations with `.navigationDestination(for: Route.self) { route in ... }`. Push `Route` values onto the path, never push views directly.
- **Path ownership.** An `@Observable` Router class, placed in the Environment, owns the paths. It exposes per-tab paths (so each tab's `NavigationStack(path:)` binds to its own slice) plus the currently presented sheet/cover state, modeled as an `Identifiable` enum (one enum case per possible sheet/cover, carrying whatever payload that presentation needs).
- **Router API surface.** The Router exposes `push`, `pop`, `popToRoot`, and `deepLink(url:)`. Screens call these methods rather than mutating a path array directly, keeping navigation logic centralized and testable.

## Deep-linking rules

- **Parse → Route.** A deep link URL is parsed into a `Route` value (or a short sequence of `Route` values) before it touches navigation state — never navigate directly off a raw URL string inside a view.
- **Build a path array.** The parsed `Route`(s) become the new path array (or are appended to it) for the relevant tab's Router state, driving `NavigationStack(path:)` to the right screen.
- **Safe home-route fallback.** If a deep link fails to parse into a valid `Route`, or points at a route that no longer exists, fall back to a safe home route rather than leaving the app in a partial or crashed navigation state.
- **Version the persisted-path storage key.** When a tab's path is persisted (e.g., for state restoration), version the storage key. A `Route` enum will evolve over the app's lifetime; an unversioned key can attempt to decode old, incompatible `Route` payloads on a later app version and crash or silently drop navigation state.

## iOS 26 tab APIs

- **Tab builder syntax (iOS 18+, still current on iOS 26).** `Tab("Home", systemImage: "house", value: .home) { HomeView() }` inside a `TabView(selection:)`. Use `.tabViewStyle(.sidebarAdaptable)` on iPad to let the tab bar adapt to a sidebar.
- **`Tab(role: .search)`.** The system search tab. Exactly one per `TabView` — a second `Tab(role: .search)` has undefined behavior at runtime, so never add more than one.
- **`.tabViewBottomAccessory { }`.** iOS 26 Music-style floating strip above the tab bar. It has documented visibility/crash quirks specifically when its content is rendered conditionally — do not swap the accessory's content in and out based on state inside the accessory closure without testing that transition; prefer keeping the accessory's presence stable and varying only its inner content deliberately, verifying on-device.
- **`.tabBarMinimizeBehavior(.onScrollDown)`.** Collapses the tab bar on downward scroll (iPhone). Other `TabBarMinimizeBehavior` values are `.automatic`, `.onScrollUp`, `.never`.
- **Per-tab stack preservation.** Each tab keeps its own `NavigationStack` and its own path in the Router; switching tabs preserves each tab's stack instead of resetting it.

## Folder template (now)

Single target, feature-foldered structure — this is the recommended starting point and stays until builds slow or changes start cascading across folders:

```
App/          — entry point, root scene, composition root (DI wiring)
Features/<Feature>/
Models/
Services/
DesignSystem/
```

- `App/` contains the `@main` entry, the root `Scene`, and the composition root that wires DI (see below).
- Each feature gets its own folder under `Features/`, named after the feature (e.g., `Features/Favorites/`).
- `Models/`, `Services/`, and `DesignSystem/` are shared across features.

## Modularization thresholds and package layout

- **When to modularize.** Move to local SPM packages only when builds are measurably slowing down or when changes in one area are cascading into unrelated areas — not preemptively.
- **Package layout.** Thin app target importing only `Presentation`:
  - `Core` — Models, DesignSystem, Logger, Networking, Storage, Utilities, Testing.
  - `Domain` — services, repositories. Depends on `Core`, never on `DesignSystem`.
  - `Presentation` — screens. Depends on `Domain` and `Core`.
- **Dependency direction.** Features depend on core (`Core`/`Domain`), never the reverse. `Core` must never import from `Domain` or `Presentation`.
- **CI note.** Package tests don't surface as workspace schemes automatically — account for this when wiring CI so package-level tests actually run.

## @Entry DI examples

- **Environment-based DI is the idiomatic default.** Define services and `@Observable` stores as `@Entry` extensions on `EnvironmentValues`:

```swift
extension EnvironmentValues {
    @Entry var favoritesService: FavoritesServicing = LiveFavoritesService()
}
```

- Inject the real implementation at the root (`App.init` / root `Scene`), and override with fakes/mocks in Previews and tests by setting `.environment(\.favoritesService, FakeFavoritesService())` on the relevant view.
- Environment values may also hold closures or protocol witnesses (the Point-Free "dependency as function values" style) — this is a respected alternative to protocol-typed services, use whichever reads more clearly for the given service.
- Constructor injection into an `@Observable` class is fine; mark non-observable injected dependencies `@ObservationIgnored` inside that class.
- No DI frameworks (no Factory, no swift-dependencies) — vanilla Environment-based DI is sufficient for most apps.

## Composition-root caveat

`Environment` is **view-scoped** — it only resolves inside the view hierarchy. Non-view code (services, background tasks, anything outside a `View`'s `body`) cannot read `@Environment`. For that code, build a `Dependencies` struct in `App.init` as the composition root, holding the same concrete instances that get pushed into the view hierarchy's Environment. Non-view code reads from the `Dependencies` composition root directly instead of trying to reach into `Environment`.

## Protocol-witness alternative note

Instead of protocol-typed services (`protocol FavoritesServicing { ... }` + a concrete conformer), Environment values can hold closures/protocol witnesses directly — a struct of function values instead of a protocol existential. This is the Point-Free-style approach referenced above; it is an accepted alternative to protocol types for services, not a replacement for the Environment-based DI pattern itself.
