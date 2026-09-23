# Guided generation and tool calling reference (iOS 27+ baseline)

## @Generable macro

Marks a Swift `struct`/`enum` as a type the model can directly generate, synthesizing a compile-time `GenerationSchema` plus a `PartiallyGenerated` mirror type.

```swift
protocol Generable: ConvertibleFromGeneratedContent, ConvertibleToGeneratedContent,
                     InstructionsRepresentable, PromptRepresentable, SendableMetatype

@Generable(description: "Basic profile information about a cat")
struct CatProfile {
    var name: String
    @Guide(description: "The age of the cat", .range(0...20))
    var age: Int
    @Guide(description: "A one sentence profile about the cat's personality")
    var profile: String
}

let response = try await session.respond(to: "Generate a cute rescue cat", generating: CatProfile.self)
response.content   // CatProfile, already parsed — no manual JSON handling
```

- Supported property types: `Bool`, `Int`, `Float`, `Double`, `Decimal`, `String`, `Array`, nested `Generable` types, enums (including with associated values).
- Macro variants: `@Generable(description:)`, `@Generable(description:representNilExplicitlyInGeneratedContent:)`, and `@Generable(name:description:...)` for a custom schema name distinct from the Swift type name.
- Make every model output that feeds a data flow a `@Generable` type. Free-text string responses for structured extraction reintroduce the parse-and-pray failure class that guided generation exists to delete.

## @Guide macro

Attaches natural-language + programmatic constraints to a property:

```swift
@Guide(description: String)
@Guide(description: String, GenerationGuide)   // e.g. .range(1...10), .count(4), .maximumCount(4),
                                                //      .anyOf([...]), .constant(_), regex pattern guides
```

- Use `@Guide` selectively — every description/constraint is serialized into the prompt's JSON schema and consumes context budget; rely on clear property naming to skip descriptions where possible.
- Order matters: generation is sequential/token-by-token, so put prerequisite fields (e.g., `subject`) before fields that depend on them (e.g., `weeklyTopics`).
- Constrain enums/fixed choices via `Generable` enums or `.anyOf(_:)` rather than open strings — this is the framework's strongest anti-hallucination and safety lever (see `safety-performance.md`).
- **Reasoning-field pattern**: put a free-text `reasoningSteps: String` property *first* in the `Generable` struct so chain-of-thought lands there instead of polluting the structured answer field.

## GenerationSchema / DynamicGenerationSchema (runtime schemas)

For output shapes not known at compile time:

```swift
let menuSchema = DynamicGenerationSchema(
    name: "Menu",
    properties: [DynamicGenerationSchema.Property(
        name: "dailySoup",
        schema: DynamicGenerationSchema(name: "dailySoup", anyOf: ["Tomato", "Chicken Noodle", "Clam Chowder"])
    )]
)
let schema = try GenerationSchema(root: menuSchema, dependencies: [])
let response = try await session.respond(to: "…", schema: schema)   // -> GeneratedContent
let soup = try response.content.value(String.self, forProperty: "dailySoup")
```

Prefer compile-time `@Generable` types wherever the shape is known ahead of time (almost always); reach for `DynamicGenerationSchema` only when the schema itself must be constructed at runtime.

## Partial generation / streaming structured output

`streamResponse(generating:)` yields `Content.PartiallyGenerated` snapshots — a synthesized mirror type with every property `Optional` — one per incremental parse checkpoint, suitable for direct SwiftUI binding without manual JSON assembly:

```swift
let stream = try await session.streamResponse(generating: Recipe.self) { prompt }
for try await partial in stream {
    draft = partial   // Recipe.PartiallyGenerated — properties fill in generation order
}
```

Stream partially generated values into UI for perceived speed — never wait for the full structured response before showing anything to the user. Property declaration order in the `@Generable` type is the generation order; order fields so a value the user reads first (e.g., recipe title) is generated before values that take longer to settle (e.g., the full step list).

## Tool protocol

```swift
protocol Tool<Arguments, Output>: Sendable {
    associatedtype Arguments: ConvertibleFromGeneratedContent   // typically @Generable
    associatedtype Output: PromptRepresentable                  // typically String or a Generable type
    var name: String { get }
    var description: String { get }
    var parameters: GenerationSchema { get }                    // synthesized when Arguments is @Generable
    var includesSchemaInInstructions: Bool { get }
    func call(arguments: Arguments) async throws -> Output
}
```

```swift
struct FindContacts: Tool {
    let name = "findContacts"
    let description = "Finds a specific number of contacts"

    @Generable
    struct Arguments {
        @Guide(description: "The number of contacts to get", .range(1...10))
        let count: Int
    }

    func call(arguments: Arguments) async throws -> [String] {
        // fetch, return plain data — no need to hand-write JSON
        contacts.map { "\($0.givenName) \($0.familyName)" }
    }
}

let session = LanguageModelSession(tools: [FindContacts()], instructions: "…")
```

- The framework decides when/whether to call a tool based on `name` + `description` + the prompt; it also handles arbitrarily complex call graphs of parallel and serial tool calls automatically — do not hand-orchestrate multi-tool sequencing.
- Tool definitions (name/description/parameter schema) are injected into the model's context when `includesSchemaInInstructions` is true, so they count against the ~4K-token budget — keep tool surfaces small, single-purpose, with terse names and descriptions.
- Error handling: `call(arguments:)` is `async throws`; a thrown error surfaces to the caller as `LanguageModelSession.ToolCallError`, which wraps the underlying error — catch that at the `respond`/`streamResponse` call site alongside `LanguageModelError`. Tool implementations should return typed, recoverable errors so the model can adjust its next attempt rather than dead-ending the flow.
- This is the grounding mechanism: rather than having the model recall facts from memory (see `../SKILL.md` — small-model honesty), a `Tool` fetches real data, and the model's job is limited to drafting structured output over that provided data.
- System tools ship alongside your own: `BarcodeReaderTool`, `OCRTool` (both Vision-backed), and a local Spotlight-index-powered RAG search tool — drop-in `Tool` conformances for common capabilities, prefer them over reimplementing equivalent functionality.
