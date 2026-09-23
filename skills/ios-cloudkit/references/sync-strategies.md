# Sync strategies reference

## Decision landscape: mirroring vs NSPersistentCloudKitContainer vs CKSyncEngine

Three ways to get CloudKit sync, in order of control-vs-effort tradeoff:

| Approach | Local store | Control | Sharing | Best for |
|---|---|---|---|---|
| `ModelConfiguration(cloudKitDatabase:)` | SwiftData | Low | No (private DB only) | Single-user, offline-first, low-to-mid data volume |
| `NSPersistentCloudKitContainer` | Core Data | Medium | Yes (CKShare support since iOS 15) | Need sharing/public DB but want a full-stack ORM |
| `CKSyncEngine` + your own store | Any (SwiftData, GRDB, SQLite, files) | High | Yes (you drive CKShare yourself) | Need sharing, public DB, fine-grained conflict/asset control, or want to avoid Core Data entirely |

**SwiftData + CloudKit mirroring** (`ModelConfiguration(cloudKitDatabase:)`): one-line opt-in, private database only. For an app already on SwiftData, this is the path of least resistance for private, per-user sync. It inherits Core Data's `NSPersistentCloudKitContainer` under the hood, so the same schema constraints apply — see `schema-sharing.md`.

**What changed WWDC25/26 for SwiftData**: WWDC25 added only model class inheritance — no new sync capabilities. WWDC26 added `@Query(sectionBy:)`, `@Attribute(.codable)`, `ResultsObserver`, and `HistoryObserver` — again **no shared/public database support**. The WWDC26 SwiftData Group Lab explicitly confirmed SwiftData "does not provide cloud syncing for public/shared data," cross-checked against two independent sources. If the app ever needs CloudKit sharing, SwiftData alone cannot do it through iOS 27 — the options are (a) a parallel `NSPersistentCloudKitContainer` stack for the shared subset, or (b) a hand-rolled `CKSyncEngine` module for that slice of data. See `schema-sharing.md` for the sharing architecture itself.

**NSPersistentCloudKitContainer**: the most mature path (since iOS 13, sharing since iOS 15). Full mirroring — Apple translates the `NSManagedObject` graph to `CKRecord`s automatically, and manages zones, subscriptions, and (for sharing) `CKShare` attachment to record hierarchies. Cost: it means maintaining Core Data as the actual store — either giving up `@Observable` ergonomics on `NSManagedObject` or hand-maintaining a parallel model layer. Reported real-world failure mode: it has proven too eager to remove local records for privacy reasons whenever it determines the user's iCloud account is unavailable — a data-loss footgun to design around (always keep independent local backups/export if this path is ever adopted).

**CKSyncEngine** (introduced WWDC23, iOS 17+): "bring your own local persistence" — CloudKit handles push scheduling, change tokens, retries, and operation batching; the app owns the local store and the mapping to/from `CKRecord`. No Core Data/SwiftData translation layer at all. Practitioner sentiment in 2026 is broadly positive on the API design, but with real production gotchas (below).

**Recommended default for a SwiftData app**: SwiftData + `ModelConfiguration(cloudKitDatabase:)` for private per-user sync (zero extra stack). `CKSyncEngine` is the escape hatch for any specific feature needing CloudKit sharing or public database — build that as an isolated module (its own store, its own sync loop) rather than retrofitting the whole app onto it. Do not adopt `NSPersistentCloudKitContainer` unless sharing is needed broadly across the existing SwiftData-modeled data, since that means running two persistence stacks or migrating off SwiftData entirely.

## CKSyncEngine in depth

Only relevant when building a CloudKit-sharing or public-database feature outside SwiftData's private-sync path.

**Core shape**: `CKSyncEngine` wraps one `CKDatabase` (private or shared) and periodically pushes/pulls changes, handling retries and respecting system conditions (battery, network, account state) automatically. Supply a `CKSyncEngineDelegate`. Multiple engines can coexist for different databases, but never run multiple engine instances against the *same* database — this causes conflicting operations (reported production issue).

**State serialization**: the engine keeps an internal `CKSyncEngine.State` that must be persisted across launches (e.g., to `UserDefaults` or a file) — access it via `engine.state`. On a `stateUpdate` event, grab `stateUpdate.stateSerialization` and write it to disk immediately; pass the last-saved serialization back in when constructing the engine on next launch (`CKSyncEngine.Configuration(database:stateSerialization:delegate:)`).

**Pending changes**: sync is driven by adding to engine state, not by calling send directly:
```swift
engine.state.add(pendingRecordZoneChanges: [.saveRecord(recordID), .deleteRecord(recordID)])
engine.state.add(pendingDatabaseChanges: [.saveZone(zone)])
```
Adding these triggers the engine's internal scheduling (subscriptions, batching, push registration) automatically.

**Event handling loop**: implement `handleEvent(_:syncEngine:)` and switch over `CKSyncEngine.Event` cases — at minimum handle: `stateUpdate` (persist token), `accountChange` (re-initialize zones on sign-in, tear down on sign-out), `fetchedDatabaseChanges`, `fetchedRecordZoneChanges` (apply remote edits/deletes to the local store), `sentDatabaseChanges`/`sentRecordZoneChanges` (confirm local pending state cleared), `willFetchChanges`/`didFetchChanges` (fetch lifecycle bracketing).

**Batching**: outgoing changes are grouped into `CKSyncEngine.RecordZoneChangeBatch` via the delegate method `nextRecordZoneChangeBatch(_:syncEngine:)` — return `nil` to signal no more changes for this send pass. Hard cap: **250 records (saves + deletes combined) per batch**; oversized batches fail with `.limitExceeded`. Use the batch initializer that takes `pendingChanges` plus a `recordProvider` closure to auto-build compliant batches rather than chunking manually.

**Conflict resolution pattern**: CKSyncEngine surfaces conflicts the same way manual `CKModifyRecordsOperation` does — via `CKError.serverRecordChanged` inside a partial failure. Standard pattern: read `ancestorRecord`/`clientRecord`/`serverRecord` off the error, merge field-by-field into `serverRecord` (it carries the current server change tag — save based on it, never a stale record), then re-add to pending changes for retry. Apple's own sample code and WWDC talk are thin on real merge examples — most nontrivial merge logic (per-field last-writer-wins vs. semantic merge) is left entirely to the app. See `accounts-errors.md` for the full conflict/merge caveats.

**Known production gotchas (2026 practitioner reports)**:
- **Deletion replay**: on a fresh/restored `CKSyncEngine` state, CloudKit can replay the *entire* history of already-deleted records before surfacing current data — one reported case took hours and forced an architectural redesign. Workaround: fetch/bootstrap current-state data manually *before* wiring up the engine's automatic fetch, rather than relying on the engine's cold-start replay.
- **Missing re-prioritization**: no built-in way to re-queue a specific failed item at high priority ahead of the rest of the pending queue.
- Sample code for conflict resolution is sparse; write and test merge logic thoroughly rather than assuming Apple's samples cover it.

**When CKSyncEngine beats mirroring**: (1) sharing/public DB is needed and two ORMs are unwanted; (2) explicit control over batching/timing is needed (e.g., large `CKAsset` uploads that shouldn't block record sync); (3) zero Core Data footprint is required; (4) a custom conflict-merge strategy beyond last-writer-wins is needed. It loses to mirroring on implementation effort (the entire local↔remote mapping and event loop is hand-written) and on documentation/example maturity (thinner than Core Data's decade of prior art).
