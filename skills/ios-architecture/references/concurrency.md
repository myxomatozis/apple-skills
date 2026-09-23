# Swift 6.2 concurrency reference (Xcode 26 defaults)

## Xcode 26 defaults (new projects) — verbatim build settings

```
SWIFT_DEFAULT_ACTOR_ISOLATION = MainActor
SWIFT_APPROACHABLE_CONCURRENCY = YES
```

These are the two build settings Xcode 26 sets on new projects, and an app target should keep both. Do not remove or override them on the app target.

## SE-0466 and SE-0461 semantics

- **SE-0466 (default actor isolation).** With `SWIFT_DEFAULT_ACTOR_ISOLATION = MainActor`, everything in the target is implicitly `@MainActor` unless explicitly marked `nonisolated` or `@concurrent`. This is what makes the app "single-threaded by default" — Apple's recommendation for app targets and UI modules. Do not sprinkle explicit `@MainActor` annotations everywhere; the isolation is already implicit from the build setting.
- **SE-0461 (`nonisolated(nonsending)`).** A `nonisolated` async function now runs on the **caller's actor**, not on the global executor. This is a semantic change from pre-6.2 Swift and is what kills most spurious `Sendable` errors that used to force actor-hopping. The old intuition "`nonisolated` means it runs in the background" is wrong under Swift 6.2 — treat `nonisolated` as "no isolation requirement," not "escapes to a background executor."
- **`@concurrent`.** The explicit escape hatch to the global (background) executor. Mutually exclusive with `@MainActor` on the same declaration. Reserve it for work that has been measured to be heavy — large JSON decode, image resize — not as a default for "anything async."

## Escape hatches, in order of preference

Prefer the least escape necessary; reach for the next one only when the previous one doesn't fit.

1. **Pure helpers on Sendable types → `nonisolated`.** Free functions or methods with no shared mutable state and no actor affinity.
2. **`nonisolated static let shared`** for globally-shared, immutable singletons.
3. **Plain constants → `nonisolated`.** Compile-time or effectively-constant values don't need actor isolation.
4. **`Task {}` bodies capturing `self`: re-capture at the boundary.** When a `Task` closure captures `self` from an isolated context, re-capture explicitly at the task boundary rather than relying on implicit capture semantics across the isolation edge.
5. **KVO/framework callbacks → `MainActor.assumeIsolated`.** For callback APIs that are documented to always fire on the main thread but aren't statically known to the compiler to be `@MainActor`, assert isolation with `MainActor.assumeIsolated` instead of adding unsafe `@preconcurrency` escapes.
6. **Heavy sync work → `@concurrent func async`.** Convert a genuinely CPU-heavy synchronous function into an `async` function marked `@concurrent` so it runs off the main actor — only after measuring that it's actually heavy (large JSON decode, image resize are the canonical examples).

## Actor guidance

- Use `actor` only for services with **real mutable concurrent state**: caches, download managers, DB connections — things that are genuinely shared and mutated from multiple call sites concurrently.
- Most services are fine as an implicit-`@MainActor` class calling `async` APIs; reaching for `actor` by default is over-engineering under MainActor-by-default isolation.

## Package opt-in for non-UI SPM packages

Non-UI packages (networking, parsers, other logic with no UI dependency) do **not** get MainActor-default isolation — they stay `nonisolated` and isolate deliberately at their own boundaries. To opt a package's target into approachable-concurrency-style defaults where it genuinely wants them, use `SwiftSetting` in `Package.swift`:

```swift
swiftSettings: [
    .defaultIsolation(MainActor.self),
    .enableUpcomingFeature("NonisolatedNonsendingByDefault"),
]
```

Do not apply `.defaultIsolation(MainActor.self)` to non-UI packages by default — it is the opposite of the "stay nonisolated" guidance above and should only be used where a package target genuinely behaves like a UI module.

## SE-0475 — `Observations` AsyncSequence

`SE-0475` introduces `Observations`, an `AsyncSequence` that emits transactional snapshots when a set of observed `@Observable` properties change. This is the idiomatic way for **non-UI code** (services, sync engines, background coordinators) to react to `@Observable` state changes without needing SwiftUI's view-body observation tracking. UIKit/AppKit on OS 26 auto-track `@Observable` objects directly, but `Observations` is the tool for code that isn't a view body or a UIKit/AppKit auto-tracking context.

## Pitfalls

1. **Macros are blind to setting-based isolation.** `SWIFT_DEFAULT_ACTOR_ISOLATION = MainActor` is a build setting, not a source-level annotation, and some macros do not see it — this can produce isolation mismatches between macro-expanded code and the rest of the file. When a macro-generated declaration behaves as if it isn't MainActor-isolated, check whether the macro is the source of the mismatch before adding manual `@MainActor` annotations everywhere.
2. **Nested types and `deinit` sometimes need explicit markers.** Default isolation does not always propagate cleanly into nested types or into `deinit`; add explicit `nonisolated` or `@MainActor` markers on these when the compiler flags a mismatch rather than assuming the outer type's isolation always applies.
3. **Pervasive `nonisolated` in a module is a smell, not a pattern.** If most declarations in a module need explicit `nonisolated`, the module's default isolation is wrong for its actual workload — split the module (e.g., pull the non-UI logic into its own package with its own deliberate isolation) instead of annotating around the mismatch.
4. **6.2 changed `nonisolated` semantics — old advice is wrong.** Guidance written before Swift 6.2 that says "mark it `nonisolated` to push it to a background thread" no longer applies: under SE-0461, `nonisolated` async now runs on the caller's actor. Treat any pre-6.2 concurrency advice about `nonisolated` as suspect and re-verify it against SE-0461.
