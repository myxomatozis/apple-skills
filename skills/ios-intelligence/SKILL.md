---
name: ios-intelligence
description: Use when implementing Apple Intelligence or on-device AI on iOS/Apple platforms — Foundation Models, SystemLanguageModel, LanguageModelSession, prompting, guided generation (@Generable/@Guide), tool calling, image understanding, Private Cloud Compute, or any LLM-powered feature.
---

# iOS Intelligence — Foundation Models (iOS 27+)

> iOS 27 APIs verified against June–July 2026 beta documentation; re-verify at GA. Items marked "unconfirmed" are not settled facts.

## Availability is a state machine, not a boolean
- Gate every AI feature on SystemLanguageModel availability and handle each unavailable reason distinctly: device not eligible (explain, don't tease), Apple Intelligence disabled (deep-link to Settings), model downloading (progress + retry). Never crash or dead-end. Why: assuming availability is the most common failure of Foundation Models apps.
- Keep a manual (non-AI) path for every AI flow, even if the app is built around AI.

## Session discipline
- One LanguageModelSession per logical flow. Instructions carry identity, rules, and output constraints; prompts carry user content — never mix them. Why: the instructions/prompt split is the framework's prompt-injection boundary.
- On-device context is small (~4K tokens). Budget with contextSize / tokenCount(for:); on context-exceeded errors, summarize state into a fresh session. Design flows that fit — no long chat histories.
- prewarm() before expected use; generation is async — stream into UI, never block the main thread.

## Output is guided generation, never string parsing
- Every model output feeding a data flow is a @Generable type with @Guide annotations (ranges, descriptions, enum constraints). Why: structured output deletes the parse-and-pray failure class.
- Stream partially generated values into UI for perceived speed.
- Image input: prompt with the image `Attachment` plus a @Generable result type. Treat output as a draft the user confirms; don't auto-commit model output.

## Small-model honesty
- The ~3B on-device model is good at summarization, extraction, classification, and structured drafting over PROVIDED content; bad at world knowledge, math, and code. Never ask it to recall facts from memory — ground it with a data source (a Tool, or a lookup done in Swift).
- Tools (Tool protocol, @Generable arguments) are how the model reaches data: keep tools fast, deterministic, and error-tolerant — return typed errors the model can recover from.

## Chain stages; never let the model drive a tool loop
- **One session that recognises, searches and composes rarely fits in 4K.** A single session with a lookup tool tends to loop on the tool — repeating the same query for data that isn't there — until the context overflows.
- **Split into stages instead.** Extract (no tools) → resolve in plain Swift → judge/compose (no tools). Each stage gets its own context window, and missing data becomes an `if` in Swift rather than repeated retries. Typically both more reliable and faster.
- **Prompt-side control of a tool loop does not work reliably**: lowering `.maximumCount`, a call budget inside the tool, a "never repeat a search" rule, or a tool replying "not found, move on" don't stop it. **Shrinking tool payloads can *increase* call count** — thinner candidate lists leave the model unable to settle. Don't "optimise" a tool by returning less.
- **Overflow does not always arrive as `contextSizeExceeded`.** It can surface as an opaque `GenerativeError`/`inferenceFailed`, so classification that only matches `LanguageModelError` buckets it as unrecognised.
- Budget before running: `contextSize` and `tokenCount(for:)` price instructions, tools, schema and prompt separately — they live on `SystemLanguageModel`, **not** on the session. The fixed floor is rarely where the budget goes; tool round-trips are.
- **Exemplars in a `@Guide` leak into tool queries.** A concrete example value in a guide description gets echoed into tool arguments when the input resembles it. Teach the format with neutral examples; never name something the model might be looking at.
- Ask heuristic questions rather than enumerating open-ended cases in code: e.g. tell the extraction stage to use your data source's vocabulary (regional synonyms) instead of maintaining a synonym table that can never be complete.
- Measure prompt changes on device. Since the Xcode 27 toolchain the Simulator serves generation (text and vision) on a Mac with Apple Intelligence enabled — a fine smoke surface — but its timings are the Mac's, and simulator-vs-device agreement is not established.

## Guardrails and refusals
- guardrailViolation and refusals are normal control flow: catch them and land on the manual fallback with honest copy. Never retry-loop against the filter; adjust prompts instead.

## Escalation ladder
| Need | Use |
|---|---|
| Drafting, extraction, classification over provided content | On-device SystemLanguageModel (default) |
| Larger context or quality, still private | Private Cloud Compute model (specifics unconfirmed — verify at GA) |
| Beyond Apple models | Server LLM behind the same app-level protocol |

## Testing
- Iterate prompts in #Playground. Unit tests assert @Generable structure and value ranges, never exact strings. Run fixtures across availability states (available / disabled / downloading).

## References
- references/foundation-models-api.md — sessions, availability states, context management, error taxonomy (26.x GenerationError vs 27 LanguageModelError), version notes.
- references/guided-generation-tools.md — @Generable/@Guide, GenerationSchema, streaming partials, Tool protocol patterns.
- references/safety-performance.md — guardrails, prewarm/latency/memory, testing and evals, on-device vs PCC vs server detail.
