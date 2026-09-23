# Swift Testing and UI testing reference

## Swift Testing core

- **`@Test` functions, `@Suite` structs.** `@Test` marks a test function; `@Suite` groups related tests in a struct (or class/actor). No `XCTestCase` subclassing.
- **`#expect`.** Non-throwing assertion macro; on failure it captures and reports the values of the sub-expressions that were evaluated, not just a pass/fail bit — write plain expressions (`#expect(result == expected)`) rather than helper-wrapped comparisons so the failure output stays informative.
- **`#require`.** Throwing-unwrap assertion macro: use it where a failed condition should stop the test immediately (e.g., unwrapping an optional that the rest of the test depends on), instead of `#expect` plus a manual early return.

## Parallelism and statelessness

- Swift Testing runs tests **in parallel by default**, with a **fresh suite instance created per test**. Design tests to be stateless — do not rely on one test's side effects being visible to another, and do not share mutable fixture state across tests via shared instance properties expecting sequential execution.
- `.serialized` forces sequential execution for a suite. Treat needing `.serialized` as a smell indicating hidden shared state or an ordering dependency — use it only as a migration crutch while un-coupling tests, not as a permanent fixture of new test code.

## Parameterized testing

- Prefer `@Test(arguments:)` over hand-written loops inside a test body — it produces one reported test case per argument, with per-case pass/fail and per-case failure output, instead of one opaque test that loops internally.
- Use `zip` to pair up multiple argument collections when a parameterized test needs more than one axis of input (e.g., zipping inputs with expected outputs) rather than reaching for the Cartesian product of two `arguments:` collections when that's not the intent.

## Traits

- `.enabled(if:)` / `.disabled()` — conditionally enable or unconditionally disable a test or suite.
- `.bug` — associate a test with a known bug/issue reference.
- `.tags` — categorize tests for selective running.
- `.timeLimit` — bound how long a test may run before being treated as failed.
- `confirmation(expectedCount:)` — the tool for testing async events: it hands you a confirming closure to call from an async callback/delegate/notification handler, and the test fails if the closure isn't called the expected number of times by the time the confirmation body completes. Use this instead of ad hoc `Task.sleep`-based waiting for async events to fire.
- `withKnownIssue` — wrap a test body that is expected to currently fail (a known, tracked issue) so it doesn't count as a new failure, while still flagging if it unexpectedly starts passing.

## Xcode 26 additions

- **Exit tests — `#expect(processExitsWith:)`.** Test code paths that are expected to terminate the process (e.g., `fatalError` paths, precondition failures) by running them in a subprocess and asserting on how that subprocess exits, instead of leaving `fatalError` paths untested because they'd crash the whole test run.
- **Attachments — `Attachment.record`.** Attach arbitrary data (images, logs, serialized state) to a test's result for inspection in the test report.
- **Raw-identifier test names.** Test function names can use raw identifiers (arbitrary strings as the function name), making test names read as natural-language descriptions instead of being constrained to valid Swift identifier characters.
- **Automatic runtime-issue detection.** Xcode 26's test runner automatically surfaces Thread Sanitizer (TSan) issues, memory leaks, and main-thread violations detected during a test run, without needing a separate manual TSan-enabled run to catch them.

## MainActor-isolation interplay

Modules that carry `SWIFT_DEFAULT_ACTOR_ISOLATION = MainActor` (see `concurrency.md`) also make their **tests** `@MainActor`-isolated by default, since the default isolation setting applies to the whole module, test targets included. Account for this when writing async test helpers or when a test intentionally needs to run off the main actor — it needs an explicit `nonisolated` or `@concurrent` marker to opt out of the module default, the same as any other declaration.

## MV testing guidance

Unit-test services, models, and state transformations — the things MV architecture (see `SKILL.md`) puts business logic into. Do not manufacture a view model whose only purpose is to give a test something to construct and call methods on; if logic genuinely needs isolated unit testing, it belongs in a service or model in the first place; extract it there rather than wrapping it in a view model for the test's sake.

## UI testing: XCUI record-replay-review workflow

- UI testing still runs on XCTest/XCUIAutomation (Swift Testing is for unit tests, not UI automation).
- Xcode 26 introduced a record–replay–review workflow (WWDC25 session 344): the recorder generates UI test code from a recorded interaction, replay runs that code across multiple locales/devices, and the test report includes videos/screenshots of the replay for review.
- Give every interactive element an accessibility identifier — this is what makes UI tests (recorded or hand-written) resilient to visual changes.
- Keep the UI test suite to a small number of high-level smoke tests covering critical flows. Previews and the Swift Testing unit suite carry the bulk of coverage; UI tests are not where thorough logic coverage belongs.

## iOS 27 baseline items relevant to testing

- **`@State` is now a macro.** `@Observable` objects held in `@State` are lazily initialized once per view lifetime (this behavior was backported to iOS 17+, so it applies regardless of deployment target). This is a breaking change in one specific pattern: a view can no longer both assign a default value to the `@State` property AND override it in `init` — that combination is now invalid. When writing or reviewing view initializers that both default and override `@State`-held `@Observable` objects, fix this before it ships; treat any code depending on the old dual-assignment pattern as tech debt to migrate.
- **`@ViewBuilder` → unified `@ContentBuilder`.** A unified builder replacing `@ViewBuilder`, with build-time improvements.
- **`alert(item:)` / `confirmationDialog(item:)`.** Item-driven alert/dialog presentation APIs — use these instead of the boolean/optional-binding alert and confirmation dialog APIs.
- **Xcode 27 "Migrate to Swift Testing" session.** Signals that XCTest-for-unit-testing is firmly legacy going forward — reinforces writing no new XCTest-based unit tests, only Swift Testing, consistent with the "Swift Testing core" rules above.
