---
name: ios-architecture
description: Use when creating a new feature or screen, deciding project structure, state management, navigation, dependency injection, concurrency isolation, or writing tests for an iOS/Apple-platform app. Prescribes vanilla Apple MV architecture, Swift 6.2 MainActor-default concurrency, and Swift Testing.
---

# iOS Architecture — MV, Concurrency, Navigation, Testing (iOS 27+)

> iOS 27 APIs verified against June–July 2026 beta documentation; re-verify at GA.

## MV, not MVVM-by-default
- Views are pure expressions of state. Business logic lives in services/models, never in body.
- State tools in order: @State (view-local) → @Observable domain store (scoped by bounded context, NOT per screen) → @Environment (shared/services).
- BANNED: a view model that mirrors @State, wraps @Environment, duplicates @Query, or exists "to make views testable". Test services and state transformations instead.
- A view model IS justified for: multi-step async orchestration, state shared across one flow's screens, logic needing injected fakes. Then it's an @Observable class owned via @State. Never ObservableObject/@Published/@StateObject in new code.
- Split a big view into smaller views BEFORE inventing a view model.
- @State lazily initializes @Observable objects exactly once per view lifetime (iOS 27 @State macro). Don't both assign a default value and override it in init — that pattern is now a compile-time trap.

## @Observable wrapper table
| You… | Use |
|---|---|
| own/create the object | @State |
| only read a passed-in object | plain let |
| need $bindings into a non-owned object | @Bindable |
| share app-wide | @Environment + .environment() / @Entry |
- Non-observable deps stored inside an @Observable class: @ObservationIgnored.

## Concurrency: single-threaded by default, escape on purpose
- App target keeps MainActor default isolation + approachable concurrency. Don't annotate @MainActor everywhere — it's implicit.
- Escape hatches, in order of preference: nonisolated (pure helpers) → @concurrent (measured CPU-heavy work: big decode, image processing) → actor (real shared mutable state only: caches, token refresh, DB handles).
- nonisolated async now runs on the caller's actor (SE-0461) — old "nonisolated = background" intuition is wrong.
- Non-UI SPM packages do NOT get MainActor default; they stay nonisolated and isolate deliberately.

## Navigation
- One NavigationStack per tab; typed path: [Route] where Route is a Hashable (ideally Codable) enum. Never NavigationPath when you control destinations, never push views directly.
- An @Observable Router in the Environment owns per-tab paths + presented sheet/cover as an Identifiable enum; exposes push/pop/popToRoot/deepLink(url:).
- TabView: Tab builder syntax with values; Tab(role: .search) for search (exactly one). Deep links: URL → Route → path array, with a safe home-route fallback.
- Alerts and dialogs use the item-binding pattern: alert(item:) / confirmationDialog(item:) — no isPresented Bool dances.

## Structure & DI
- Now: single target, feature folders — App/ (entry, root scene, composition root), Features/<Feature>/, Models/, Services/, DesignSystem/.
- Later (only when builds slow or changes cascade): local SPM packages Core / Domain / Presentation, thin app target. Features depend on core, never the reverse.
- DI: @Entry-based Environment values holding services; a Dependencies composition root built in App.init pushes the same instances to Environment (non-view code uses the root directly). No DI frameworks.

## Testing
- Swift Testing for ALL unit tests: @Test, #expect, #require; parameterized via @Test(arguments:); tests are parallel and stateless (fresh suite instance per test) — .serialized is a smell.
- Test services, models, state transformations. Do not manufacture view models to have something to test.
- UI tests: XCUIAutomation, few and high-level (smoke of critical flows), accessibility identifiers everywhere.

## References
- references/concurrency.md — Swift 6.2 escape patterns, pitfalls, package settings.
- references/navigation-di.md — Router/deep-link patterns, folder template, @Entry DI, composition root.
- references/testing.md — Swift Testing conventions, traits, exit tests, XCUI record-replay.

## Related skills
- Visual/HIG decisions inside screens: ios-ui.
- API clients, persistence, repositories: ios-data.
- Auth state and session ownership: ios-auth.
