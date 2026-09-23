---
name: ios-data
description: Use when making API calls, designing endpoints or DTOs, decoding JSON, storing or querying data, caching, sync, offline behavior, or background transfers in an iOS/Apple-platform app. Prescribes native URLSession, Codable safety, and SwiftData rules.
---

# iOS Data — Networking, Codable, SwiftData

> Targets iOS 27+ (`@Query(sectionBy:)`, `@Attribute(.codable)`, `ResultsObserver`, `HistoryObserver` need availability gating on earlier targets). Verified against June–July 2026 beta docs; re-verify at GA.

## Networking
- Native URLSession async/await. No Alamofire. Client is a plain Sendable struct/final class — NOT an actor (an actor serializes all requests for no benefit).
- The only networking actor: token refresh/dedup — concurrent 401s await one refresh Task (see ios-auth).
- Shape: Endpoint/Request<Response: Decodable> value types + one generic send(_:) → Response. DTOs are Sendable structs.
- Always validate HTTPURLResponse.statusCode yourself — URLSession does not fail on 4xx/5xx. Keep the error body for server error payloads.
- One APIError enum layered transport → HTTP → decoding → domain, with isRetryable.
- Retry: exponential backoff + jitter via Task.sleep (cancellation-aware), capped attempts, idempotent requests only, respect Retry-After on 429/503. waitsForConnectivity for "no network yet".
- Load in .task {} so navigation-away cancels requests for free.
- Background transfers: URLSessionConfiguration.background is delegate-based (async APIs don't survive termination); uploads from file only; stable session identifier recreated on relaunch.

## Codable
- Explicit CodingKeys (grep-able, no key-transform pass). One dateDecodingStrategy per API; .iso8601 does NOT parse fractional seconds — use ISO8601FormatStyle/.withFractionalSeconds custom strategy. Never leave .deferredToDate.
- One bad element must not kill a feed: lossy-collection wrapper (try?-decode elements, compactMap), decodeIfPresent + defaults, server enums with an .unknown(String) case. Log DecodingError codingPath to telemetry — silent lossiness hides API drift.
- Wire DTOs are separate from SwiftData @Model classes and view state. Map explicitly.
- Very large payloads: decode inside a @concurrent function (off the main actor).

## SwiftData
- VersionedSchema + SchemaMigrationPlan from day one; new schema version only when shipping model changes. .lightweight for additive, .custom for derived values. Migration tests against fixture stores.
- All writes go through a @ModelActor "DataHandler"; UI reads via @Query on mainContext.
- @Model is NOT Sendable — pass PersistentIdentifier across actor boundaries, re-fetch via context.model(for:).
- Known quirk: a ModelActor can run on the main thread (executor inherits creation context) — create handlers inside Task.detached for heavy work.
- No batch operations exist — for bulk workloads (10K+ rows) flag the decision; don't loop-insert silently. #Predicate lacks case-insensitive contains and regex.
- Grouped lists use @Query(sectionBy:) — don't group in view code.
- Non-view layers observe the store with ResultsObserver (queries + observation outside SwiftUI) and consume change feeds with HistoryObserver — the intended building blocks for sync engines.
- @Attribute(.codable) persists arbitrary Codable types but cannot be used in predicates or sorting — never put queryable fields inside one.
- CloudKit sync, schema rules for mirrored models, sharing, and CKError handling: ios-cloudkit owns all of it — follow that skill before touching sync.

## Caching & offline
- Two tiers: URLCache for server-owned re-fetchable responses/images (honors Cache-Control/ETag); SwiftData for user-owned data and anything queried offline. Never build a JSON-blob cache inside SwiftData.
- If the product needs offline-first: View → Repository → SwiftData store ↔ SyncEngine (actor) ↔ API client. UI never renders network responses directly; the network only writes into the store. Writes carry a syncState flag (pending/synced/failed).
- AsyncImage has real HTTP caching (honors server headers) plus AsyncImage(request:) and .asyncImageURLSession(_:) for cache-policy control; third-party image loaders (Kingfisher/Nuke) only for unusually heavy image feeds.

## References
- references/networking.md — client code shape, error/retry patterns, background transfer rules.
- references/swiftdata.md — migrations how-to, @ModelActor patterns, caveats, iOS 27 additions (baseline). CloudKit rules live in ios-cloudkit.
- references/offline-caching.md — repository/sync-engine stack, URLCache configuration, image loading.

## Related skills
- Concurrency defaults and package isolation: ios-architecture.
- Token storage and refresh actor: ios-auth.
- iCloud sync and CloudKit: ios-cloudkit.
