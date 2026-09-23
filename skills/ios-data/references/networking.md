# Networking + Codable reference

## URLSession is the whole networking layer

- `data(for:)` / `data(from:)` are the standard request/response calls. `bytes(for:)` for streaming. `upload`/`download` for foreground transfers. Combine and delegate-based APIs are legacy for plain request-response — don't reach for them on new code.
- URLSession does not treat 4xx/5xx as errors — a request against a 404 or 500 endpoint completes successfully at the transport level. Always validate `HTTPURLResponse.statusCode` yourself, on every call site, before treating a response as success.
- Kick requests off from a `Task` tied to UI actions; use `withThrowingTaskGroup` for N-way parallel fan-out; reach for `Task.detached` rarely (it drops the caller's context and cancellation).
- URLSession requests auto-cancel when the surrounding `Task` is cancelled — pair loading calls with SwiftUI's `.task {}` modifier so navigation-away cancels in-flight requests for free, with no manual cancellation bookkeeping.

## Client shape

- **Consensus for 2026: a plain struct or `final class` marked `Sendable`, not an actor.** For stateless request/response work, an actor only serializes every request through its executor for no benefit — there's no shared mutable state to protect. Reserve `actor` for the cases with real mutable state: token cache/refresh coordination, request de-duplication, rate limiting.
- Structure: `Endpoint`/`Request<Response: Decodable>` value types describing method/path/body/headers, plus **one generic `send(_:)`** that takes a request value and returns `Response`. Every DTO is a `Sendable` struct.
- A dedicated networking SPM package should default to `nonisolated` — it does not get MainActor-default isolation (see `ios-architecture`'s concurrency reference for the package-settings mechanics).
- `async throws(APIError)` typed throws are a good fit right at the client boundary (the `send(_:)` call site) — don't push typed throws through generic middleware layers, where the concrete error type stops being useful and just adds friction.
- Token refresh is the canonical actor use case: an actor holds the in-flight refresh `Task`, so concurrent 401s across multiple requests all await the same refresh instead of firing N parallel refreshes. See `ios-auth` for the actor itself.

## Errors

Layer errors in this order, matching where they can originate:

1. `URLError` — transport failure (no connection, timeout, cancelled).
2. HTTP status — non-2xx `HTTPURLResponse.statusCode`, with the response body preserved (server error payloads are often meaningful — don't discard them).
3. `DecodingError` — wrapped with context (which type, which coding path) rather than surfaced raw.
4. Domain error — the app-meaningful translation of the above (e.g. "session expired," "item not found").

Collapse all four into **one `APIError` enum** with an `isRetryable` property, so call sites branch on one type instead of threading `URLError`/`DecodingError`/HTTP-status logic through the UI layer.

## Retry

- **Exponential backoff with full jitter**, driven by `Task.sleep` (which is cancellation-aware — a cancelled retry loop stops sleeping instead of firing a request nobody wants).
- Cap the number of attempts. Retry **idempotent requests only** (GET and safe retries — never blindly retry a POST that creates a resource).
- Respect a server's `Retry-After` header on `429` (rate limited) and `503` (unavailable) responses instead of computing a delay yourself when the server has told you one.
- Set `waitsForConnectivity = true` on the session configuration for the "no network yet, but one might appear" case, and use `NWPathMonitor` separately to drive offline UI state (banners, disabled actions) rather than inferring it from request failures alone.

## Alamofire — when it would actually earn its place

Alamofire is maintained and Swift-6-clean (5.10.x current; AF6 in development), but it is **not needed for a new app** — native URLSession plus the thin client shape above is the default. Reach for it only if the app specifically needs one of:

- `RequestInterceptor`-style cross-cutting request/response interception.
- Built-in auto-validation helpers.
- SSL pinning ergonomics beyond what `URLSessionDelegate` gives you directly.
- Multipart form construction helpers.
- Download resumption helpers.

Adopt it when one of these becomes a real requirement, not by default.

## Background transfers

- `URLSessionConfiguration.background(withIdentifier:)` is **delegate-based, not async/await**. The async/await surface (`data(for:)` etc.) does not survive process termination, so background transfers must go through `URLSessionDownloadDelegate`/`URLSessionTaskDelegate` callbacks instead.
- Use a **unique, stable session identifier** and recreate the session with that same identifier on relaunch — the system reconnects a relaunched background session to in-flight transfers by identifier.
- Implement `application(_:handleEventsForBackgroundURLSession:completionHandler:)` to be woken for background session events, and call the stored completion handler from `urlSessionDidFinishEvents(forBackgroundURLSession:)` once all delegate callbacks for that session have run — this is what tells the system the app is done processing and can be suspended again.
- Only `download`/`upload` tasks run in the background; **uploads must be from a file**, not from an in-memory `Data` buffer (the OS needs a file it can hand off to its own daemon after your process suspends).
- `isDiscretionary = true` defers a transfer to better conditions (charging, Wi-Fi) — use it for large, non-urgent transfers where timing doesn't matter to the user.
- Pair background URLSession transfers with `BGTaskScheduler` (`BGAppRefreshTask` for short periodic work, `BGProcessingTask` for longer maintenance work) when the goal is periodic sync rather than a single one-off transfer — the background session handles the transfer itself, BGTaskScheduler handles getting your code invoked periodically at all.

## Network framework redesign — not a URLSession replacement

WWDC25/iOS 26 introduced a Network framework redesign: `NetworkConnection`/`NetworkListener`/`NetworkBrowser` with async send/receive, Codable message framing, and Wi-Fi Aware peer-to-peer support. **This is for raw sockets and P2P connections, not HTTP.** URLSession remains the API for HTTP/HTTPS networking — do not route ordinary API calls through the new Network framework types.

WWDC26 confirmed no URLSession overhaul; gRPC Swift 2 got first-class guidance for gRPC services specifically. AsyncImage has request-based cache control via `AsyncImage(request:)` and `.asyncImageURLSession(_:)` (details: offline-caching.md).

## Codable safety patterns (full)

- **CodingKeys**: write them explicitly. `.convertFromSnakeCase` is fine for a small, stable API, but explicit `CodingKeys` is preferred at scale — it's faster (no runtime key transform pass) and grep-able (you can search for a wire key and land on the exact type that maps it).
- **Dates**: pick **one `dateDecodingStrategy` per API** and use it consistently. `.iso8601` does **not** parse fractional seconds — if the API emits `2026-08-07T12:00:00.123Z`, `.iso8601` fails on it. Use a `.custom` strategy backed by `ISO8601DateFormatter` with `.withFractionalSeconds`, or `Date.ISO8601FormatStyle` configured the same way. Never leave the default `.deferredToDate` in place — it decodes as a raw `TimeInterval` since a reference date, which almost never matches what a JSON API actually sends.
- **Partial-failure safety — one bad element must not kill a whole feed:**
  - A lossy-collection wrapper: decode each element with `try?` inside a `compactMap`, dropping only the elements that fail instead of failing the whole array decode. (Community patterns like `@LossyArray`/`@DefaultEmptyArray`, BetterCodable-style, implement exactly this.)
  - `decodeIfPresent` plus a default value for fields the server might omit, instead of a required key that throws when absent.
  - Open/extensible server enums: give them an `.unknown(String)` case so an unrecognized value the server adds later decodes into `.unknown` instead of throwing and killing the whole containing object.
  - **Log `DecodingError`'s `codingPath`** to telemetry wherever these safety patterns swallow a failure — silent lossiness (an element quietly dropped, a field quietly defaulted) hides real API drift from the team until a user notices missing data.
- **DTO vs. domain**: wire DTOs (`Sendable` structs mirroring the JSON shape) are a separate type from both SwiftData `@Model` classes and view state. Map explicitly between them at a clear boundary — never decode straight into a `@Model` type.
- **Large payloads**: decode inside a function marked `@concurrent` so the decode work runs off the main actor. Reserve this for payloads actually large enough to matter (measured, not assumed) — see `ios-architecture`'s concurrency reference for the general escape-hatch ordering.
