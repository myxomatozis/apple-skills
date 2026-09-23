# Schema and sharing reference

## Mirrored-model constraints (normative — the exact rule set every synced `@Model`/`NSManagedObject` must satisfy)

These apply whether the app uses SwiftData mirroring or `NSPersistentCloudKitContainer` (both ultimately produce a CloudKit schema via Core Data's model→`CKRecord` mapping); `CKSyncEngine` users design the `CKRecord` schema by hand and should still follow the CloudKit-native subset of these rules.

- **No `@Attribute(.unique)` / uniqueness constraints anywhere** — CloudKit cannot enforce uniqueness atomically across devices.
- **Every property must be optional or have a default value** — CloudKit must support partial-record sync.
- No Core Data "Undefined" attribute type.
- **All relationships must be optional** (even if defaulted to empty arrays) — no non-optional to-one/to-many.
- Relationships need an inverse defined (SwiftData does this automatically; Core Data requires manual setup).
- **Delete rule cannot be `.deny`** — cascade/nullify only.
- **No ordered relationships** — CloudKit doesn't preserve Core Data's "ordered" relationship flag; if order matters, store an explicit `Int` sort-index attribute.
- Mirroring targets the **private database only** — no shared/public database support (see "SwiftData sharing status" below).
- Once a schema is live in Production it is **add-only** — never delete, rename, or retype record fields (see "Add-only schema evolution" below).

**Framing**: within these constraints, SwiftData + CloudKit mirroring gives offline-first behavior for free — the local SwiftData store is the source of truth and stays fully functional with no account, while `ModelConfiguration(cloudKitDatabase:)` handles the private-database sync transparently. Violate any constraint above and mirroring either fails to set up or fails silently on some devices; there is no partial-compliance mode.

## CKAsset handling

Use `CKAsset` for any discrete binary blob (images, files) rather than inlining bytes into a record field — assets are stored/streamed separately and are owned by their record (deleting the record garbage-collects the asset server-side). SwiftData/Core Data map `Data`-typed "External Storage" attributes to `CKAsset` automatically once large enough. For CKSyncEngine-driven models, construct `CKAsset(fileURL:)` directly and manage local temp-file lifecycle yourself — asset uploads read from disk, not memory.

## References vs. denormalization

Avoid complex graphs of `CKReference`s — deep/wide reference networks cause pain on update/delete and query. Query support on references is limited: the `ANY %@ IN relatedRecords`-style filter caps at **250 objects** in the predicate. Prefer denormalizing small, frequently-read fields (e.g., a display name) onto the referencing record rather than joining — CloudKit has no server-side join. Reserve real references for ownership/parent-child structure, especially where `CKShare` hierarchy matters (see below).

## Indexes

CloudKit auto-creates indexes per field in Development; in Production, index types are independent — a field needing both filtering and sorting needs **two separate indexes** (Queryable ≠ Sortable). Indexes update asynchronously and are not guaranteed to be current immediately after write. Reported gotcha: indexes added to a record type in Production sometimes fail to cover records created *before* the index was added, with no dashboard signal of reindex status — treat newly-added production indexes as suspect until verified against real data.

## Dev → Production schema promotion workflow

The single most common "works in Xcode, broken in TestFlight/App Store" bug class:

1. Development environment allows automatic schema creation/modification as the app runs against it (default for Debug builds/local Xcode runs).
2. Production **prohibits automatic schema changes** — it must be explicitly promoted from CloudKit Console (console.cloudkit.com) → your container → Development → **Schema → Deploy Schema Changes** → review diff → confirm. Only the schema is copied, never records/data.
3. **TestFlight and App Store builds hit Production by default.** Submitting to the App Store does **not** deploy schema — that's a separate manual dashboard action to remember on every model change.
4. Repeat the deploy step every time a model change ships. There is no CI-triggered auto-deploy; this is a manual, human-gated step — belongs on a pre-release checklist.
5. To test Production behavior before general release, a build can point at Production via the CloudKit entitlement/environment override (temporarily) — revert before actual submission.

## Add-only schema evolution once live

Production is strictly additive:

- **Allowed**: add a new record type; add a new field to an existing record type.
- **Not allowed / dangerous**: renaming a field or record type (causes data loss — CloudKit treats it as a new field, old data orphaned); changing a field's type (breaks sync for existing records); deleting a field/record type that clients still reference.
- Practical rule: deprecate (stop writing to) unused fields rather than removing them; never rename — add a new field and migrate data in application code if needed.

## Sharing & collaboration architecture

**CKShare fundamentals**: `CKShare` is a special `CKRecord` subtype representing a share of either (a) an entire **record zone** (unbounded collection, no parent-child structure required), or (b) a **record hierarchy** rooted at a record with a `parent` reference set on descendants — the hierarchy form gives fine control over exactly which records a participant can see. Create the `CKShare`, set its `publicPermission`/per-participant permissions, save it (typically via `CKModifyRecordsOperation` alongside the root record); CloudKit mints a stable share URL.

**Participant management**: `CKFetchShareParticipantsOperation` looks up a potential participant (by email/phone/user record ID) before adding; `addParticipant(_:)`/`removeParticipant(_:)` on the share manage membership; the owner controls per-participant `permission` (`.readOnly`, `.readWrite`) and `role`.

**Presenting sharing UI**:
- **UIKit**: `UICloudSharingController` — standard system sheet for invite/manage/remove/permission-change flows, driven off the `CKShare` + container.
- **SwiftUI**: no native SwiftUI-first sharing view as of iOS 27 betas. Wrap `UICloudSharingController` in `UIViewControllerRepresentable` and present via `.sheet`/`.background(...)`. Apple's own sample project (`apple/sample-cloudkit-sharing`) does exactly this.
- **ShareLink**: a lighter-weight alternative for sending the share *link* only (not full participant management) — requires a custom `Transferable` implementing `CKShareTransferRepresentation` to hand CloudKit a `CKShare` through `ShareLink`. Gets "send invite" UX without the full management sheet; use `UICloudSharingController` when add/remove/permission management is needed in-app.
- **macOS/SwiftUI**: no direct SwiftUI equivalent either; use `NSSharingService(named: .cloudSharing)` from AppKit-bridged code.

**Share acceptance flow**: the recipient taps the share link/invite → the OS opens the app via `userDidAcceptCloudKitShareWith(...)` (UIKit `SceneDelegate` callback) or the SwiftUI `.onContinueUserActivity`/`CKShare.Metadata`-based entry point → fetch the shared zone into the CloudKit **shared database** (a third database alongside private/public) and surface those records in a "Shared" UI section, separate from the owner's private-database records.

**SwiftData sharing support status (through iOS 27): not supported.** SwiftData's CloudKit integration is private-database-only through iOS 27 betas — confirmed by the WWDC26 SwiftData Group Lab and cross-checked against two independent iOS-27-changelog writeups; no shared/public database capability shipped or was previewed for SwiftData. If the app needs collaborative/shared records, the options are: (a) `NSPersistentCloudKitContainer` for that subset of data (it has had `CKShare` support since iOS 15), or (b) hand-rolled `CKSyncEngine` against the shared database. There is no way to keep a sharing feature inside pure SwiftData today — do not attempt to bolt it on.
