# SwiftData reference

## Feature timeline

- **iOS 17** — SwiftData's debut. Early migration bugs, largely fixed by ~17.4.
- **iOS 18** — `#Index`, `#Unique`, a history API, and custom stores.
- **iOS 26 (WWDC25)** — model class inheritance (session 291). Notably **fixed**: `@ModelActor` mutations not reliably triggering `@Query` updates in the view layer, and Codable properties working inside predicates. On iOS 17/18 the bug is still present — and older guidance you find online may predate the fix.
- **iOS 27 (WWDC26, session 274)** — see "iOS 27 additions" below.

## Concurrency model

- The canonical background-write pattern is a `@ModelActor`, conventionally named something like `DataHandler` in app code. **All writes go through it.**
- `@Model` classes are **NOT `Sendable`**. Never pass a `@Model` instance across an actor boundary directly — pass its `PersistentIdentifier` instead, and re-fetch the model on the receiving side via `context.model(for:)`.
- `container.mainContext` is `@MainActor`-isolated, and it's what `@Query` in SwiftUI views reads from — UI reads go through `@Query` on `mainContext`, writes go through the `@ModelActor`.
- **Known quirk, still present in 2026**: a `ModelActor`'s executor inherits the isolation of the context it was *created* in — so a `ModelActor` created on the main thread can end up executing on the main thread despite being "an actor." For genuinely heavy work, create the handler inside `Task.detached` so its executor isn't inherited from a main-actor creation context. For light work, this quirk can be accepted rather than worked around — know it's there before debugging an unexpected main-thread stall.

## Performance and gaps (single-benchmark, directional — not a guarantee)

- Single-object CRUD is roughly at parity with Core Data.
- **No batch operations exist in SwiftData.** One published benchmark measured roughly **5x slower** for 10K-row inserts and roughly **7x slower** for 5K-row deletes compared to an equivalent batch operation elsewhere. This is one benchmark's numbers, directional only — but the underlying fact (no batch API) is not in question. For bulk workloads (10K+ rows), **flag the tradeoff explicitly as a decision** rather than silently looping single inserts/deletes.
- `#Predicate` lacks case-insensitive `contains` and regex support — plan filtering logic around this rather than discovering it mid-implementation.
- Autosave can persist temporary/in-progress objects earlier than expected — be deliberate about when autosave is enabled versus explicit `save()` calls for state you don't want persisted prematurely.
- **Core Data**: nothing new at WWDC25 or WWDC26. It's maintained, not evolving — a real but static baseline to compare against.

## Migrations

- Set up `VersionedSchema` + `SchemaMigrationPlan` **from day one**, even before the first migration is needed — retrofitting versioning onto a shipped schema is the hard path.
- Add a new schema version **only when actually shipping a model change** — don't create speculative versions "just in case."
- `.lightweight` migration stages handle additive changes (new optional properties, new types). `.custom(willMigrate:didMigrate:)` stages handle anything requiring derived values (renamed/restructured properties, computed backfills).
- **Pitfall: "Duplicate version checksums" crash.** Redundant/unnecessary schema versions — versions that don't actually change the schema's shape — can produce identical checksums between versions, which crashes the migration plan at runtime with a "Duplicate version checksums" error. Keep every `VersionedSchema` version meaningfully different from the last; don't bump a version number without a real shape change.
- Test migrations against fixture stores — a store file built at an old schema version, run through the migration plan, and asserted against the expected new-version shape — not just against a freshly created store.

## CloudKit

CloudKit sync, schema rules for mirrored SwiftData models, sharing, and CKError handling live in `ios-cloudkit` — see its `references/schema-sharing.md` before adding `ModelConfiguration(cloudKitDatabase:)` or any CloudKit-backed sync.

## Honest consensus (2026)

SwiftData is **production-ready for the mainstream case**: new SwiftUI-first iOS 27+ apps, small-to-medium data volumes (low tens of thousands of rows), private CloudKit sync, and no heavy relational querying.

Skeptics raise real, specific concerns: the threading quirks above, no batch operations, and the `#Predicate` gaps. Notably, Mattt (Tsai) has suggested GRDB for apps that need real query scale, and at least one team has reported migrating an app off SwiftData onto GRDB after hitting these limits in production.

**Rule of thumb for when to reach for something else:**
- **SwiftData** — the default for new iOS 27+ SwiftUI apps. Use it unless one of the cases below applies.
- **Core Data** — existing codebases already built on it, apps still supporting iOS 15/16, workloads needing 50K+ row batch operations, or schemas with 30+ entities where Core Data's tooling (batch requests, more mature migration tooling) matters more than SwiftData's SwiftUI-native ergonomics.
- **GRDB** — apps that need direct SQL control, full-text search (FTS), complex relational queries, or query performance at real scale, where `#Predicate`'s gaps and the lack of batch operations would become a recurring problem rather than an edge case.

Revisit this choice if the app's data volume or query complexity grows past "mainstream case" — this is a decision to reconsider deliberately, not a one-time pick that's permanent regardless of how the app evolves.

## iOS 27 additions (baseline)

These ship as of iOS 27 — gate with `@available(iOS 27, *)` if the app supports earlier versions.

- **`@Query(sectionBy:)`** — sectioned queries, for grouped list UI driven directly by `@Query` instead of a manual grouping pass over a flat result.
- **`@Attribute(.codable)`** — lets a `Codable`-conforming type be stored as a model attribute. **Not predicable or sortable** — don't use it for a property you need to filter or sort on in a `#Predicate`.
- **`ResultsObserver`** — a `@Query`-style observation API usable outside SwiftUI view bodies, for non-UI layers (services, sync engines) that need to react to store changes the way a view's `@Query` does.
- **`HistoryObserver`** — a transaction/change-feed API, the intended mechanism for custom sync engines to detect local changes that need pushing to a server (see `offline-caching.md` for the repository/sync-engine stack it plugs into).
