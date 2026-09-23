---
name: ios-cloudkit
description: Use when implementing iCloud sync, CloudKit, CKSyncEngine, sharing or collaboration, sync conflict handling, iCloud account state, schema design for synced models, or CloudKit schema deployment in an iOS/Apple-platform app.
---

# CloudKit — Sync, Schema, Accounts (iOS 27+)

> iOS 27 APIs verified against June–July 2026 beta documentation; re-verify at GA.

## Choosing the sync stack
| Need | Use |
|---|---|
| Private-data sync of the app's SwiftData store | SwiftData + CloudKit mirroring: ModelConfiguration(cloudKitDatabase:) |
| Sharing/collaboration, public database, or fine-grained sync control | Isolated CKSyncEngine module |
| Coexistence with a Core Data stack | NSPersistentCloudKitContainer |
- SwiftData sharing is UNSUPPORTED through iOS 27. Never bolt sharing onto the SwiftData stack — build it as an isolated CKSyncEngine (or NSPersistentCloudKitContainer) module with its own zone. Why: confirmed unsupported at WWDC26; teams that assumed otherwise shipped rewrites.
- Mirroring targets the private database only. Public-database features are their own module and their own design.

## Schema rules for mirrored models
- No @Attribute(.unique). Every property optional or defaulted. All relationships optional. No .deny delete rules.
- Once deployed to Production the schema is add-only: never delete, rename, or retype record fields.
- Binary payloads ride in CKAsset, never in record fields.

## Multi-configuration containers (mirrored + local-only, side by side)
- To keep sensitive data (e.g. personal health information, App Store Guideline 5.1.3(ii)) out of
  iCloud while still SwiftData-backed, add a second `cloudKitDatabase: .none` `ModelConfiguration`
  inside the same `ModelContainer` alongside the mirrored one.
- **Both configurations must be explicitly named**, or `ModelContainer` throws building the two
  together (an unnamed configuration defaults `name == "default"`, and two same-named configurations
  are indistinguishable to it).
- Reopening a store written under this pattern needs a matching `name` *and* a matching-breadth
  `for:` schema, or Core Data reads it as a migration.

## Accounts
- Check CKAccountStatus at launch and observe the account-changed notification; keep the app fully usable signed-out (local store is the source of truth; sync resumes on sign-in).
- Re-check status on foregrounding: disabling the app in iCloud settings does NOT fire the change notification. Why: documented field gotcha — apps missed the state change entirely.

## Errors
- Classify CKError: retryable (network failures, service unavailable, rate limited — honor retryAfterSeconds, exponential backoff) vs terminal (quota exceeded → user-facing message, not authenticated → account flow).
- serverRecordChanged is a conflict: three-way merge (client, server, common ancestor); last-write-wins only for fields where the product accepts silent loss.
- Batch limit: ≤250 records per operation (CKSyncEngine batches for you; manual CKModifyRecordsOperation must chunk).

## Deployment & testing
- Never ship against the Development environment. Promote schema Dev → Production in CloudKit Console before release; verify with a TestFlight build on Production.
- Simulator sync is unreliable — test sync flows on devices.
- Privacy manifest: CloudKit access declares approved reason code C617.1; no tracking domains.

## References
- references/sync-strategies.md — decision landscape detail; CKSyncEngine internals (state serialization, pending changes, event loop, batching, production gotchas).
- references/schema-sharing.md — schema and CKAsset rules; sharing architecture (CKShare, zones, sharing UI); SwiftData sharing status.
- references/accounts-errors.md — CKAccountStatus handling, CKError taxonomy, environments and promotion workflow, testing limits, review considerations.

## Related skills
- Local persistence, repositories, and the sync-engine stack shape: ios-data (this skill owns everything CloudKit-specific).
- Actor design for sync engines: ios-architecture.
