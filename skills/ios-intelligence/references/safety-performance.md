# Safety, small-model behavior, performance, and testing reference (iOS 27+ baseline)

## What the ~3B on-device model is good/bad at

**Good**: summarization, entity extraction, text understanding/refinement, short dialog, generating short creative content, classification against a small fixed label set (via `Generable` enums), basic tool orchestration. The current-generation on-device model shows a large quality jump over the prior generation, including on image-understanding tasks.

**Bad**: general world knowledge (explicitly not designed as a general chatbot/knowledge base — no reliable long-tail facts), multi-step/complex reasoning, math, long/nested conditional logic embedded in a single prompt, verbose or ambiguous instructions.

**Rule of thumb**: if a task needs facts the model wasn't trained on, ground it with a `Tool` (fetch real data) rather than trusting recall; if it needs deep reasoning, either restructure as short deterministic steps in Swift (not in the prompt) or escalate to `PrivateCloudComputeLanguageModel` / a server LLM. E.g. never ask the model for a product's price or specs from memory — have it draft structure over tool-provided data.

## Prompt engineering specifics for small models

- **1–3 paragraphs max**, direct imperative verbs ("List", "Create"), no hedging/politeness filler, single well-defined goal per prompt — combining unrelated asks measurably degrades quality.
- **Assign a role/persona** ("You are an expert travel assistant…") — the model responds well to explicit framing and holds a consistent tone if the prompt is written in that voice.
- **Step-by-step plans beat implicit reasoning** — write out numbered steps for the model to follow rather than expecting it to derive a plan; if a single request still fails, split it across multiple `LanguageModelSession`s (accepts higher latency for reliability).
- **Few-shot with 2–15 simple examples** — long/complex examples cause the model to repeat/hallucinate details from the examples themselves; keep exemplars terse.
- **Push conditionals into Swift, not the prompt** — build the instructions string with `switch`/`if` in code rather than asking the model to evaluate many `IF` branches; this both improves reliability and shrinks the context footprint.
- **Reasoning-field pattern** — see `guided-generation-tools.md`.
- **Emphasis**: `MUST`/`ALWAYS`/`AVOID` in caps strengthens instruction-following, but over-repeating/over-emphasizing makes prompts brittle — re-evaluate after each tweak rather than piling on emphasis.
- **Temperature**: near-0 (`.greedy` sampling) for deterministic/guardrail-compliant structured output; ~0.7–0.8 for creative freeform text. `GenerationOptions(samplingMode:temperature:maximumResponseTokens:)` — use `maximumResponseTokens` sparingly, hard truncation can produce malformed/ungrammatical output rather than a clean stop.

## Guardrails: violation vs. refusal

- Two layers: (1) the model itself is trained to handle sensitive topics carefully, on-device and on PCC; (2) `Guardrails` independently check both input and output for self-harm, violence, adult content, etc.
- Violations throw `LanguageModelError.guardrailViolation` (string responses) or, for guided generation, the model may instead throw `GenerationError.refusal(_:_:)` with an async-fetchable `.explanation` — surface that explanation to the user rather than guessing.
- **Unconfirmed — re-verify at GA**: the refusal case is confirmed only under its 26.x `GenerationError` spelling; whether it moves to `LanguageModelError` alongside the rest of the iOS 27 rename (see `foundation-models-api.md`) or keeps a separate `GenerationError` home is not yet settled in the live doc JSON — re-check once Xcode 27 ships out of beta.
- String responses can also come back as an in-band **refusal message** ("Sorry, I can't help with…") without throwing — you generally can't detect this programmatically, so design the UI to display whatever text comes back gracefully.
- Both a thrown `guardrailViolation`/`refusal` and an in-band refusal message are normal control flow, not exceptional failures: catch/detect them and land on the manual (non-AI) fallback with honest copy. Never retry-loop against the filter; adjust the prompt instead.

## Input/output boundary hierarchy

Safest → least safe: fixed-choice enum picked in Swift → `Generable` enum constrained output → freeform string with a deny-list check → fully open freeform string. Prefer the narrowest boundary the feature can tolerate.

- Never interpolate raw user input into `Instructions` (prompt-injection risk); wrap it inside app-authored `Prompt` framing text instead (see `foundation-models-api.md`).
- **Permissive guardrail mode** for legitimate reasoning-about-sensitive-material use cases (e.g., flagging profanity in logged text): `SystemLanguageModel(guardrails: .permissiveContentTransformations)` — string generation only; guided generation still uses default guardrails, and the base model's own safety layer can still refuse.
- Do a proactive per-feature risk assessment (harm × severity × mitigation table), maintain a safety test suite (nonsense input, sensitive topics, controversial topics, vague input; log prompt+response+whether guardrails fired), and re-run the full safety suite whenever Apple ships a model or guardrail update (guardrails can update out-of-cycle with the OS). Report unhandled gaps via Feedback Assistant; retrieve `LanguageModelFeedback` transcripts for structured bug reports.

## Localization / language support

- The on-device model is multilingual; supported locales track Apple Intelligence's supported-locale list generally. **Unconfirmed**: the specific four-group locale breakdown (English variants / PFIGSCJK / Nordic+Turkish+Vietnamese / Arabic+Finnish+Indonesian+Hebrew+Hindi+Malay+Polish+Russian+Thai+Ukrainian) was reconstructed from secondary sources — treat the runtime API below as the source of truth, not any hardcoded list.

```swift
SystemLanguageModel.default.supportsLocale()                 // defaults to Locale.current, fuzzy-matches (en-AU ~ en-NZ)
SystemLanguageModel.default.supportedLanguages: Set<Locale.Language>
```

- Unsupported locale throws `LanguageModelError.unsupportedLanguageOrLocale(_:)` — disable the AI feature for that user/locale and fall back to a non-AI path.
- All `@Generable` type/property names and descriptions must themselves be in a supported language if you want the model to honor them well.
- Guardrails are only guaranteed active for supported languages/locales — a short unsupported-language phrase mixed into otherwise-supported-language text is a real gap; be conservative with fully open input in multilingual flows.
- To force a specific output language, state it explicitly and emphatically in `Instructions` (e.g., `"You MUST respond in U.S. English."`), and pass locale context (`"The person's locale is \(locale.identifier)."`) rather than relying on implicit inference.

## Performance: prewarm(), latency, memory

- Call `session.prewarm(promptPrefix:)` as early as UX allows (screen appear, before the user finishes composing) — it eagerly loads model resources and can cache a known prompt prefix, reducing time-to-first-token for the eventual real call.
- Session/tool/schema size directly trades off against latency and reliability: fewer tools, shorter descriptions, and smaller `Generable` graphs both reduce time-to-first-token and reduce the chance of the model getting confused by an overloaded context.
- Profile with the Instruments **Foundation Models template** — inspect prompts, warm-up latency, session memory footprint, and Neural Engine utilization before shipping, not just in ad hoc manual testing. An enhanced agentic-flow debugging variant is available for tool-calling flows.

## #Playground and Instruments

- Add `#Playground { … }` at the top of a standalone Swift file to get a live, interactive canvas (like SwiftUI Previews) for iterating on prompts against the real on-device model without a full app run cycle. Playgrounds can access types defined elsewhere in your target, so you can loop a candidate prompt over real sample data and eyeball outputs quickly. This is the fastest inner loop for prompt iteration — use it before writing any tests.
- The Instruments Foundation Models template gives latency breakdown and control-flow traces for agentic/tool-calling flows; use it to validate `prewarm()` actually helps and to watch session memory / Neural Engine utilization before shipping.

## Unit testing nondeterministic output — three-ring model

Exact-match assertions are wrong for a probabilistic system; structure tests in three tiers:

1. **Deterministic floor** — assert shape, not words: every enum case the schema can emit maps to a known Swift case (catches enum drift), required fields non-empty, array counts within `@Guide` bounds, enum-constrained fields are valid members. Runs against live model output but only checks structural contracts.
2. **Behavior around the seam** — extract the model call behind a small protocol (e.g., `protocol TicketExtracting: Sendable { func extract(from:) async throws(TicketExtractionError) -> TicketDraft }`), then unit test all the surrounding app logic (error mapping, caching, retries, UI states) against a mock/stub — fully deterministic, runs in CI/Simulator/offline, no model involved.
3. **Evals for output quality** — accept a quality-rate floor rather than 100% correctness (e.g., "80% of N trials ground the answer in real data") and track the rate over time as a regression metric.

Gate live-model test rings behind an explicit env var (e.g. `RUN_LIVE_AI_TESTS=1`) plus an availability check — availability alone isn't sufficient for CI gating since Simulator/CI hardware may not support Apple Intelligence at all. Run fixtures across all availability states (available / disabled / downloading), not just the happy path.

## Evaluations framework (iOS 27+)

A Swift framework purpose-built to quantify how prompt/schema/instruction changes move accuracy on a fixed eval set, giving a statistical read on regressions/improvements instead of eyeballing a few Playground runs, paired with a "Hill-climbing" systematic prompt-optimization workflow. Treat this as the eventual replacement for ad hoc "ring 3" eval harnesses.

## On-device vs. PCC vs. server — decision table

| Need | On-device (`SystemLanguageModel`) | PCC (`PrivateCloudComputeLanguageModel`) | Server LLM API (own backend / Anthropic / OpenAI) |
|---|---|---|---|
| Works fully offline / airplane mode | required | needs network | needs network |
| Zero per-request cost, no infra | yes | free under 2M downloads (2026 pricing), else quota-limited | pay per token always |
| Context > ~4K tokens (26.x) | no | yes — 32K token window (**unconfirmed**, verify at GA) | yes, provider-dependent, often larger |
| Deep multi-step reasoning / hard math / code generation | weak | "advanced reasoning," configurable levels | frontier models strongest here |
| Broad world-knowledge Q&A | not designed for this | better, still bounded by training cutoff | best, especially with retrieval/search tools |
| Strict data-never-leaves-device requirement (health, sensitive PII) | only option that guarantees this | strong, independently verifiable privacy guarantees, but still off-device | generally not acceptable |
| Need a specific frontier model's behavior/brand (Claude, Gemini) | no | no (unless Apple's own PCC model suffices) | yes, via first-party SDK, or via `LanguageModel`-conforming Anthropic/Google Swift packages once shipped (**unconfirmed** — see `foundation-models-api.md`) |
| Available on devices without Apple Intelligence eligibility | no | no — same eligibility gate applies | works on any device with network |
| Latency-critical, must respond in real time with no network round trip | yes | no — network round trip | no — network round trip |

Default to on-device for anything privacy-sensitive, offline-capable, or "small" (summarize/extract/classify/short-gen); reach for PCC/server only once a feature demonstrably needs a bigger context window, harder reasoning, or broader knowledge than the 3B model can deliver — and always keep an on-device or non-AI fallback path, since Foundation Models availability is never guaranteed.

## Privacy and disclosure

- **No network requirement**: on-device Foundation Models calls make zero network requests and involve no telemetry leaving the device — this is the core privacy pitch. Don't undermine it by logging prompts/responses to an analytics backend without disclosure.
- **Privacy manifest**: because Foundation Models is a local, on-device system API, it does not need to be declared as a data-collection mechanism in the Privacy Manifest the way a third-party analytics/ad SDK would. This does not extend to PCC or third-party `LanguageModel` providers — those leave the device and must be disclosed like any other network service/API in the privacy nutrition label and manifest.
- **App Store review**: if the app shares any user data with third-party AI systems (bringing your own cloud LLM, or a third-party `LanguageModel` provider), disclose this in-app (visible UI, not just a link to a privacy policy) and obtain clear permission before transmission. Apps that generate user-visible AI content are expected to explain how the feature works, mark AI-generated content as such, provide a way to report inappropriate output, and hold rights to any data feeding the model. Don't over-claim capability the ~3B on-device model doesn't reliably deliver — that is a stated review risk, not just a UX quality issue.
- **User-facing AI disclosure norms**: make it clear when a UI surface is AI-generated (visual treatment, label, or both), give users an easy regenerate/retry action, avoid silently discarding a refusal (show something rather than a blank state), and give users a path to opt out of an AI feature where feasible. **Unconfirmed**: the exact current wording of Apple's HIG "Generative AI" page could not be fully verified in the source research — confirm live wording before finalizing disclosure copy.
