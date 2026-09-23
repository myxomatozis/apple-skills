# Accounts, errors, environments reference

## CKAccountStatus

Four cases: `.available`, `.noAccount`, `.restricted`, `.couldNotDetermine`. Check via `try await container.accountStatus()`; treat any non-`.available` state as "sync disabled, operate local-only" in the UI, not as a hard error.

**Mid-session account changes**: register for `CKAccountChangedNotification` (`NotificationCenter`) to react to sign-in/out/switch while the app is running. **Caveat to design around**: if the user disables iCloud for this specific app in Settings, the system terminates the app outright and does **not** post `CKAccountChangedNotification` — the effect is only visible on next launch. Always re-check `accountStatus()` on foreground/launch; don't rely solely on the notification. Full sign-out of the iCloud account at the OS level does post the notification. Store `accountStatus` as an `@Observable`/published property gated behind a single owner (e.g., a CloudKit-status service) so UI reacts consistently.

## CKError retry/backoff taxonomy

CKError exposes roughly 36 distinct codes.

**Retryable (transient)**: `.requestRateLimited`, `.zoneBusy`, `.networkFailure`, `.networkUnavailable`, `.serverResponseLost`, `.serviceUnavailable`. For rate-limited/throttled responses, CloudKit supplies `CKErrorRetryAfterKey` in `userInfo` (surfaced as `CKError.retryAfterSeconds`) — **always honor this value rather than a fixed backoff**. On top of it, use exponential backoff with jitter for errors without an explicit retry hint, capped at a sane ceiling, and never hammer retries synchronously in a loop — aggressive retry after throttling risks a documented 24-hour reliability degradation per Apple's quota behavior.

**Non-retryable / needs-developer-action**: `.quotaExceeded` (user or app storage cap hit — surface to user, don't retry), `.invalidArguments`, `.unknownItem`, `.permissionFailure`, `.notAuthenticated` (route to sign-in prompt), `.badContainer`/`.badDatabase` (config bug, not transient).

## `serverRecordChanged` resolution (the core conflict case)

Triggered under `.savePolicy = .ifServerRecordUnchanged`: the error's `userInfo` carries `ancestorRecord`, `clientRecord`, `serverRecord`. Merge intended changes into `serverRecord` (it holds the current, valid change tag) and re-save that object — saving a stale `clientRecord` will just conflict again.

**Caveat**: `ancestorRecord` is sometimes populated with system fields only, not full data, unless the record uploaded was built via `CKRecord.encode(with:)` from a full archive or freshly fetched from CloudKit. Don't assume an always-complete 3-way merge is available — design merge logic to tolerate a partial ancestor. Last-write-wins is only acceptable for fields where the product accepts silent loss; anything else needs field-by-field merge logic, tested against fixture `CKRecord`s since Apple's own samples are thin here.

## Batch limits and rate limits

- **250 records (saves + deletes combined) per operation** — `CKSyncEngine` batches for you (see `sync-strategies.md`); manual `CKModifyRecordsOperation` must chunk manually.
- Documented aggregate figure: **40 requests/second up to ~400K active users**, scaling as **10 requests/second per additional 100K active users**; overage returns HTTP 503 "throttled" with a retry interval. Design implication: batch reads/writes, avoid firing a query on every UI event, and prefer push-delivered payload data (`desiredKeys`/`alertLocalizationArgs`) over a follow-up fetch after every remote-change notification.
- **Uncertain figures**: the paid overage tier (an additional 10 req/s reported at $100) and the public-database free-tier/pricing figures (250 MB asset storage, 2.5 MB database storage, 50 MB/day transfer, and per-GB overage rates) are last-known published numbers only — Apple's CloudKit pricing page has reportedly been pulled/inconsistent as of 2026. Treat these as directionally correct historical figures, not verified current pricing; check CloudKit Console's quota/usage dashboard and Apple's current developer pricing docs before any cost-sensitive design decision, especially before choosing the public database for anything beyond light metadata.

## Environments and CloudKit Console

Every container has independent **Development** and **Production** databases/schemas. Xcode debug builds and local device runs target Development by default; **TestFlight and App Store builds always target Production** — see `schema-sharing.md` for the deploy workflow. A common root cause of "works on my machine, broken for beta testers" bugs is simply forgetting the manual schema-deploy step.

**CloudKit Console** (console.cloudkit.com): inspect/edit schema, browse records per environment, manage indexes, view/deploy schema diffs. Notably, the private database's real user data is **not** browsable by the developer — a privacy boundary: only the owning device/account can read its own private-database records, even from the console. Public database records are visible/manageable, as intended.

## Testing: simulator and CI limitations

- The Simulator must be signed into a real iCloud account (Settings app inside Simulator), and the CloudKit-using test target needs a Host Application set for CloudKit APIs to function at all.
- Simulator CloudKit support has had recurring periods of reported breakage across Xcode versions — **do not treat Simulator as a reliable CI signal for CloudKit sync correctness**. Prefer real-device testing for sync-critical verification; keep CI coverage to logic that doesn't require a live CloudKit round-trip (mock the sync boundary — e.g., test `CKSyncEngineDelegate` batch-building/merge logic against fixture `CKRecord`s rather than a live container).
- No first-class headless/CI-friendly CloudKit test harness exists from Apple; automated end-to-end sync testing remains an open practitioner pain point, not something WWDC26 addressed.
- Push-notification-driven background sync in particular cannot be reliably triggered/observed in CI — test that path manually on-device.

## Privacy manifest and App Store review

- Data in the **private database** is protected — not even the developer can read another user's private records; a positive talking point for App Privacy "Data Not Collected/Not Linked" claims if private sync is the only data path.
- Data in the **public database** is, by default, readable by *anyone* with the container credentials, even unauthenticated — never put PII or sensitive user content there without explicit record-level access control and a clear privacy-policy disclosure; misusing the public database for what should be private data is a real rejection/compliance risk.
- **Privacy manifest**: CloudKit file-container access (reading timestamps/size/metadata of files inside the app's CloudKit container) falls under the Required Reason API category for file-timestamp/metadata APIs — declare reason code **`C617.1`** ("access timestamps, size, or other metadata of files inside the app container, app group container, or CloudKit container") in `PrivacyInfo.xcprivacy`'s `NSPrivacyAccessedAPITypes` if the code (directly or via a dependency) touches those file attributes. Enforced since May 1, 2024, and still enforced in 2026.
- Gate CloudKit sync behind an explicit user opt-in (toggle) and require iCloud sign-in before any data leaves the device — avoids silent background uploads that could be flagged in review or conflict with data-collection disclosures.
