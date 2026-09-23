# Offline-first & caching reference

## The stack

```
View (@Query or ResultsObserver-backed VM)
  → Repository
    → SwiftData store ↔ SyncEngine (actor) ↔ API client
```

**UI never renders network responses directly.** The network layer only ever writes into the local store; views only ever read from the store (via `@Query` on `mainContext`, or a `ResultsObserver`-backed view model outside SwiftUI). This is the rule that makes "working offline" not a special case — the view's data source is always local, so there is nothing different to do when the network is unavailable.

## Repository responsibilities

- **Read**: fetch from the local SwiftData store first (this is what the view actually sees), and separately trigger a background refresh against the API. The refresh's job is to update the store — the view picks up the update through its existing `@Query`, not through a value the repository hands back from the refresh call.
- **Write**: write to the local store immediately (so the UI reflects the change with no network round-trip latency), tag the write with a `syncState` flag (`pending`/`synced`/`failed`), and enqueue the corresponding push to the server. The write is "done" from the UI's perspective the moment it lands locally; sync is a background concern layered on top.

## SyncEngine actor duties

- An `actor`, because sync state (in-flight pulls/pushes, retry bookkeeping) is genuinely shared mutable state — this is one of the legitimate actor use cases, not a stateless client that would be over-engineered as an actor.
- **Pull**: fetch server state and reconcile it into the SwiftData store.
- **Push**: send locally-originated writes (the ones tagged `pending`) to the server, and flip their `syncState` to `synced` or `failed` based on the result.
- **Conflict policy**: the default is **last-write-wins (LWW)**, commonly keyed off an `updatedAt` timestamp comparison between the local and server versions. Don't build a more elaborate merge/CRDT strategy unless the product actually has a documented need for it — LWW is the deliberate, sensible default here, not a stand-in for something more sophisticated later.
- **Retry**: pair failed pushes with `BGTaskScheduler` so a failed sync gets retried on the next opportunity (app foreground, background refresh window) rather than being silently dropped or requiring the user to notice and retry manually.
- `HistoryObserver` (iOS 27 baseline) is the change-feed primitive for detecting local changes that need pushing; pair it with the `syncState` flag on each write (pending/synced/failed) so the SyncEngine can query cleanly for what still needs to go out.

## URLCache configuration

- URLCache and SwiftData are **two distinct tiers with two distinct jobs** — don't blur them:
  - **URLCache**: for server-owned, re-fetchable responses and images. It understands HTTP caching semantics natively (`Cache-Control`, `ETag`) — the server tells it how long a response is good for and how to revalidate, and URLCache honors that automatically.
  - **SwiftData**: for user-owned data, and for anything the app needs to query while offline (filter, sort, join across entities). URLCache has no query capability — it's a blob cache keyed by request, not a queryable store.
- **Never build a JSON-blob cache inside SwiftData** — storing raw response bodies as blobs in a `@Model` property defeats the point of both tiers: it gets none of URLCache's HTTP-semantics handling, and none of SwiftData's actual query capability over structured data.
- Configure URLCache per session (memory/disk capacity sized to the app's actual response volume) rather than relying on undocumented system defaults for a cache the app depends on for offline behavior.

## AsyncImage caching

- `AsyncImage` has real HTTP caching: it uses the **shared URLCache** and honors server headers (`Cache-Control`/`ETag`) — the same semantics as the rest of the app's networking, with no extra configuration needed.
- `AsyncImage(request:)` and `.asyncImageURLSession(_:)` give direct, request-based control over the URLRequest and URLSession `AsyncImage` uses, instead of relying on the shared cache implicitly — reach for these when a screen needs cache-policy control beyond the default.
- **Kingfisher/Nuke**: still the right choice for genuinely heavy image feeds (large scrolling grids, aggressive prefetching, disk-cache tuning beyond what URLCache offers) — adopt when such a feed exists, not by default. The baseline `AsyncImage(request:)` control reduces how often a third-party image library is needed.

## SwiftData + CloudKit: offline-first for free

Pure-CloudKit offline-first behavior (private database, no custom server) is owned by `ios-cloudkit` — see its `references/schema-sharing.md` for the constraints and setup. The Repository/SyncEngine stack above is what to build instead when the app needs a custom server backend.
