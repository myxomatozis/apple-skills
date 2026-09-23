# Foundation Models core API reference (iOS 27+ baseline)

## SystemLanguageModel

`final class SystemLanguageModel` — the on-device Apple Foundation Model (~3B params) that powers Apple Intelligence. Conforms to `Observable`, `Sendable`, and (iOS 27+) `LanguageModel`.

```swift
static var `default`: SystemLanguageModel   // base general-purpose model
convenience init(useCase: SystemLanguageModel.UseCase, guardrails: SystemLanguageModel.Guardrails)
// e.g. SystemLanguageModel(useCase: .contentTagging)
```

## Availability enum — every unavailable reason

```swift
var availability: SystemLanguageModel.Availability   // .available | .unavailable(UnavailableReason)
var isAvailable: Bool                                  // convenience bool

enum UnavailableReason {
    case deviceNotEligible   // no A17 Pro+/M-series NPU — Apple Intelligence not supported on this hardware
    case appleIntelligenceNotEnabled   // Apple Intelligence toggle off in Settings
    case modelNotReady       // assets still downloading, or other system readiness issue
    // additional unknown/future reasons possible — always add a default case
}
```

```swift
struct GenerativeView: View {
    private var model = SystemLanguageModel.default
    var body: some View {
        switch model.availability {
        case .available:
            // show AI UI
        case .unavailable(.deviceNotEligible):
            // hide feature entirely — no path to enable
        case .unavailable(.modelNotReady):
            // downloading — show progress state, retry later
        case .unavailable(let other):
            // unknown reason — hide gracefully
        }
    }
}
```

Treat each reason distinctly, never as one generic "unavailable" bucket:
- `appleIntelligenceNotEnabled` is user-recoverable — deep-link to Settings.
- `modelNotReady` is temporary — poll/retry, show progress.
- `deviceNotEligible` is permanent for that device — hide the feature, don't tease it.

## LanguageModelSession

`final class LanguageModelSession` — one conversational context; interactions recorded in a `Transcript`. Conforms to `Observable`, `Sendable`.

```swift
convenience init(model: some LanguageModel = SystemLanguageModel.default,
                  tools: some Collection<some Tool> = [],
                  instructions: String)
convenience init(model: some LanguageModel, tools: some Collection<some Tool>, transcript: Transcript) // rehydrate
```

- One `LanguageModelSession` = one context; reuse across turns of the same conversation, create a fresh one per unrelated task.
- `var isResponding: Bool`, `var transcript: Transcript` (full history, replayable), `var usage: LanguageModelSession.Usage` (accumulated token usage).
- `func prewarm(promptPrefix: Prompt? = nil)` — eager resource load + optional prompt-prefix cache warm. Call as early as plausible (screen `onAppear`, before the user finishes typing) to hide model/asset load latency; verify with Instruments that it actually reduces time-to-first-token for your prompt shape.

## Instructions vs. prompt separation

- `Instructions` (session-level, set once at init) define persistent behavior/persona/safety rules and **take priority over prompts** — the model is trained to weight instructions above user prompts, making them the correct place for safety/steering rules.
- `Prompt` (per-call) is the specific request/user input.
- **Never interpolate unverified user input directly into `Instructions`** — that's a prompt-injection vector; only verified/programmatic app state belongs there. User input belongs in the `Prompt`, ideally wrapped in app-authored framing text rather than passed raw.

## Prompt/respond APIs

```swift
func respond(to prompt: Prompt, options: GenerationOptions = .init())
    async throws -> LanguageModelSession.Response<String>

func respond<Content: Generable>(to prompt: Prompt, generating: Content.Type,
                                  includeSchemaInPrompt: Bool = true, options: GenerationOptions = .init())
    async throws -> LanguageModelSession.Response<Content>

func respond(to prompt: Prompt, schema: GenerationSchema, includeSchemaInPrompt: Bool = true, ...)
    async throws -> LanguageModelSession.Response<GeneratedContent>   // dynamic/runtime schema
```

Trailing-closure `PromptBuilder` variant is idiomatic:

```swift
let response = try await session.respond {
    "Compare these two images by using three bullet points:"
    Attachment(imageOne)
    Attachment(imageTwo, orientation: .right)      // multimodal prompting
}
```

### Streaming

```swift
func streamResponse(to prompt: Prompt, options: GenerationOptions = .init())
    rethrows -> sending LanguageModelSession.ResponseStream<String>

func streamResponse<Content: Generable>(generating: Content.Type, includeSchemaInPrompt: Bool = true, ...)
    rethrows -> sending LanguageModelSession.ResponseStream<Content>
```

For structured `Generable` streaming, each element is `Content.PartiallyGenerated` — a synthesized mirror type with every property optional — **not raw text deltas**. Bind directly to SwiftUI state:

```swift
let stream = try await session.streamResponse(generating: Itinerary.self) { prompt }
for try await partial in stream {
    itinerary = partial   // Itinerary.PartiallyGenerated — properties fill in generation order
}
```

Property declaration order in the `@Generable` type is the generation order — order fields so dependent content (e.g., a "reasoning" field) comes before content that depends on it.

## Multimodal image input

```swift
let response = try await session.respond {
    "What animal is this?"
    Attachment(uiImage)          // UIImage / NSImage / CGImage / CIImage / CVPixelBuffer / file URL
}
```

Handles arbitrary sizes/aspect ratios automatically (scaling, color conversion); larger images consume more tokens. `.label("barcode-image")` helps the model disambiguate multiple attachments. For photo-to-structured-data flows, prompt with the photo `Attachment` plus a `@Generable` result type.

## Context window limits & overflow

- On-device context window ≈ **4,096 tokens total** — covers instructions + full transcript (all prior prompts/responses) + current prompt + injected tool/schema definitions, cumulative across the session's lifetime, not just the current turn.
- `model.contextSize` (Int, tokens) and `session.tokenCount(for:)` let you inspect budget/cost programmatically instead of parsing error strings.
- **Recovery pattern**: catch the overflow error, summarize the transcript so far (e.g., ask the model itself for a compact recap), and start a new `LanguageModelSession(transcript:)` seeded with the condensed history — full re-derivation of context is not possible, transcripts must be actively managed. See TN3193 "Managing the on-device foundation model's context window."
- Tool/`Generable` schema definitions (name, description, JSON-schema-shaped parameters) count against the same budget — keep tool names/descriptions terse, prefer short property names, use `@Guide(.maximumCount(_:))` on arrays, and skip `@Guide` descriptions where the property name alone is unambiguous.

### Version note: context-overflow error rename (historical, 26.x → 27)

- **iOS 26.x**: overflow threw `LanguageModelSession.GenerationError.exceededContextWindowSize(_:)`. The associated `Context` includes a human-readable message reporting the actual token count (e.g., "Content contains 9056 tokens, which exceeds the maximum allowed context size of 4096") — this was the only practical way to estimate tokens on 26.0 since no public tokenizer was exposed. `contextSize`/`tokenCount(for:)` were added at 26.4.
- **iOS 27**: the whole `GenerationError` enum is deprecated in favor of `LanguageModelError` (and `SystemLanguageModel.Error` / `LanguageModelSession.Error`); the overflow case is now `LanguageModelError.contextSizeExceeded(_:)`. When targeting iOS 27+, write against `LanguageModelError.contextSizeExceeded(_:)` — the `GenerationError` name matters only for older deployment targets and possibly the refusal case, and is kept here so old sample code and Apple docs snippets are recognizable — see `safety-performance.md` for the still-unconfirmed refusal-case exception.
- **Unconfirmed — re-verify when Xcode 27 ships**: the precise Xcode-version cutover behavior for this rename — apps built with Xcode 26 reportedly keep compiling against `GenerationError` until rebuilt with Xcode 27 — is confirmed only as deprecated-as-of-27.0 in the live doc JSON; the Xcode-cutover claim itself should be re-checked against release notes once Xcode 27 ships out of beta, not treated as settled.

## PrivateCloudComputeLanguageModel and third-party providers

```swift
let session = LanguageModelSession(model: PrivateCloudComputeLanguageModel())
let response = try await session.respond(to: "…", contextOptions: ContextOptions(reasoningLevel: .light))
response.usage.input.totalTokenCount
response.usage.output.reasoningTokenCount
```

- Server-side Apple model on Private Cloud Compute. **Unconfirmed**: the widely-cited 32,000-token context window figure is sourced from a secondary WWDC-2026 summary, not Apple's primary doc JSON — confirm the exact number before stating it as fact in code, comments or UI copy.
- Configurable reasoning levels; no developer account/API-key setup — entitlement-gated (`com.apple.developer.private-cloud-compute`), request access at developer.apple.com/private-cloud-compute/.
- Privacy: no prompt storage, architecture independently verifiable.
- Pricing: free for developers with <2M first-time App Store downloads; higher daily limits for iCloud+ subscribers.

```swift
protocol LanguageModel: Sendable {
    var capabilities: LanguageModelCapabilities { get }
    var executorConfiguration: Executor.Configuration { get }
    associatedtype Executor: LanguageModelExecutor
}
protocol LanguageModelExecutor: Sendable {
    init(configuration: Configuration) throws
    func prewarm(model: Model, transcript: Transcript)
    func respond(to: LanguageModelExecutorGenerationRequest, model: Model,
                 streamingInto: LanguageModelExecutorGenerationChannel) async throws
}
```

- `SystemLanguageModel` and `PrivateCloudComputeLanguageModel` both conform to `LanguageModel`. **Unconfirmed**: Anthropic and Google announced at WWDC 2026 that they are publishing conforming Swift packages so `Claude`/`Gemini` models would plug into the same `LanguageModelSession` API surface unchanged — actual package availability/names are not confirmed shipped as of the source research (Aug 2026). Verify at integration time before writing code against them; do not assume package names.
- Executors are cached/reused by configuration hash to preserve KV-cache and session state across calls.
- Auth guidance for any such provider: no plain API-key string params; use OAuth/token-provider flows, store tokens in Keychain, consider App Attest for device attestation on your own backend.

## Open sourcing (unconfirmed)

Apple stated at WWDC 2026 that the Foundation Models framework core is going open source "later this summer" (2026), alongside `CoreAILanguageModel` (Neural Engine) and `MLXLanguageModel` (Mac GPU via MLX), both conforming to `LanguageModel`. **Unconfirmed**: verify the actual repository and license exist before depending on or referencing it.
