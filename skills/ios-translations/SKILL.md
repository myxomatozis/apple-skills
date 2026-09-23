---
name: ios-translations
description: Use when adding, moving, or changing any user-facing string in an iOS/Apple-platform app — String Catalogs, catalogue keys, Localizable.xcstrings, LocalizedStringResource, plurals, accessibility labels, or checking whether copy is translatable before shipping to a new market.
---

# iOS Translations — String Catalogs, translation-ready

Goal: adding a language is a **data change with no code change**. Every rule below exists because breaking it makes that false silently — in English everything still renders, and the defect only appears once a second language exists.

## One door for catalogue strings

If your strings live in a Swift package, route them through one helper:

```swift
func catalogueCopy(_ key: StaticString, _ fallback: String.LocalizationValue) -> LocalizedStringResource
```

- Keep it **package-internal**: the app target can only *read* catalogue strings, never declare them. Copy types (`SettingsCopy`, `OnboardingCopy`, …) live in the package; the app reads them.
- **Two bundle spellings, and the asymmetry is real.** `LocalizedStringResource` takes `bundle: .atURL(Bundle.module.bundleURL)` — what the helper writes for you. `String(localized:)` takes `Bundle?`, i.e. `bundle: .module`, so **a function returning `String` cannot use the helper** and writes `.module` at its own call site. Omitting `bundle:` searches the *app* bundle, which never receives the package's catalogue: correct in English, silently untranslated forever.
- `key` is `StaticString` because a key is a compile-time constant by construction.
- **`.module` means the bundle of the target you are in.** From the package's sources it carries the catalogue; **from its test target it does not**. An experiment run inside the test target reports a silent fallback the real call site never has. Verify catalogue behaviour from a source file or a standalone package, never from the test target.

## Key naming and the translator's view

- `<domain>.<member>`, lowerCamelCase, dotted (`onboarding.welcomeTitle`, `settings.signOutButton`).
- **The catalogue `comment` is the only thing a translation vendor sees.** Write it for someone with no access to the repo: name the screen, the trigger condition, and what each `%@` is. No ticket numbers, internal IDs or session bookkeeping.
- **Expand abbreviations a translator must localise.** A single-letter label or clipped unit means nothing until the comment spells it out; other languages abbreviate differently.
- **One key per user-visible surface.** Split when two surfaces have different constraints (a tab label has a width budget, a nav title does not). Share when it is one surface in two modalities with identical text (a visible row and its VoiceOver label). Point rather than duplicate when it is literally the same surface. Identical English under two keys is fine and often right; identical English under two keys *for the same surface* is drift waiting to happen.

## Numbers — declare every one, and thread the locale

A number interpolated into a **localised** value silently takes the locale's grouping separator: `1200` → `1,200` in en_US, `1.200` in de_DE. That is only correct if it is the reader's actual locale, so a function that renders a number takes its own `locale:` rather than trusting the format style's default.

- **Declare every number.** A visible figure takes `.formatted(.number.locale(locale))` (or `.formatted(.percent.locale(locale)…)` — the sign's position is locale-dependent: `"61%"` in English, `"61 %"` with a no-break space in French, `"%61"` in Turkish, so a hardcoded `%` is wrong the same way a hardcoded separator is). A spoken figure takes bare `String(...)` — never a format style, never a `locale:` parameter — so VoiceOver never reads a separator as "comma". A raw `Int` interpolated directly is never correct **except** at a plural key, where the rule selects on the numeric value.
- **The parameter is `locale: Locale = .autoupdatingCurrent`, last position, on every visible function.** The default keeps existing call sites unchanged; making it explicit is what lets a test prove the separator flips. `.autoupdatingCurrent`, not `.current` — a snapshot goes stale when the reader changes region without relaunching.
- **Thread it to every half the function has, including transitively.** A function often formats the number *and* resolves the sentence around it (`String(localized:…, locale:)`). Threading to only one is a defect wearing a fix's clothes. If a function receives an already-formatted argument and passes it to another catalogue-resolving function, that callee needs the locale too. A static check that flags a locale-taking function calling another locale-taking function without passing `locale:` is worth having; one hop by member name catches most cases, but a multi-hop chain or a callee reached through a variable stays a review item.
- **Spoken functions take no `locale:` parameter at all**, not even for the sentence around a spoken number, while the app ships one language — a parameter that can never be exercised is one not to add.
- **`Text(verbatim: "\(count) items")` is not a fix.** It dodges the separator and **freezes the English unit word permanently** — and a stray-literal guard waves it through. Catalogue the template, format only the number.
- **Pin the numbering system to `"latn"` only for a value the app parses back** (e.g. a text field round-tripping a typed number — grouping and decimal separators still follow the locale; only digit shapes are pinned). Display-only readouts use the locale's own numbering system.
- **A `locale:` on a function that formats nothing is enforceable only by a static check.** When the function receives an already-formatted value, dropping `locale: locale` changes nothing observable while the catalogue is English-only, so no test can catch it. Write the two-locale test anyway (it pins that the caller's separator survives the template), but say in its doc comment what it does **not** cover.

## Plurals

- Native `variations.plural` with `one`/`other` and `%lld`. **Never** `count == 1 ? … : …` in Swift — that hardcodes two grammatical categories; Arabic has six, Polish four, Russian three.
- **When the sentence never prints the count** (only "it"/"them" varies), a bare `variations.plural` is *illegal* — `xcstringstool` refuses a plural variant that never references the number. **Use a `substitutions` entry**: `"value": "%#@count@"` plus a `substitutions.count` block with `argNum: 1`, `formatSpecifier: "lld"`, and the whole sentence in each variant using `%2$@` for the other argument. Two top-level `.one`/`.other` keys is the shape the tool's error message suggests — **and it is wrong**, for the same reason the Swift ternary is.
- **The `defaultValue` interpolation declares the argument list and its types.** For a substitution key the count must be interpolated **first** even though it is never printed:

  ```swift
  String(localized: "permissions.missingReads",
         defaultValue: "\(count)The system didn't share your \(names), so the app asks for it on the next screens.",
         bundle: .module)
  ```

  Interpolate only `names` and argument 1 becomes a `String`, the plural rule has no number to select on, and **it silently falls back to `defaultValue`** with no error. It looks like the `substitutions` form "not working".
- **Verify plurals by compiling, not by reading JSON.** `xcstringstool compile` the catalogue and `plutil -p` the generated `.stringsdict`; a real plural has `NSStringFormatSpecTypeKey = NSStringPluralRuleType`. A key that compiled into `.strings` instead is not a plural.

## The four traps

1. **`Text(verbatim:)` on catalogue-backed copy** — freezes English. Correct only for genuine *data* (a user-entered name, a provider name) or an already-resolved `String`.
2. **`LocalizedStringResource(stringLiteral:)` around resolved text** — turns finished text back into a lookup key.
3. **A bare literal reaching a `LocalizedStringResource`/`LocalizedStringKey` parameter** from package code — English-as-key against the app bundle, which never gets the package's catalogue. `Text`, `Label`, `Button`, `.navigationTitle`, `.alert`, `LabeledContent`, `TextField`, and every custom view taking `LocalizedStringKey` are all this trap.
4. **A `String` overload beside a `LocalizedStringResource` one under the same name** — a bare literal binds to `String` and skips the catalogue. Name the `String` variant distinctly (e.g. `sectionHeaderVerbatim(_:)`).

**Gratuitous resolution is a defect.** `Text`/`Label`/`Button`/`.navigationTitle`/`.alert` all take a `LocalizedStringResource` — widen a `String`-typed helper, don't feed it resolved strings. The exception: a member that composes sentences at runtime, or is public API pinned by tests, may legitimately return `String`.

## Sweeping a folder — pattern-greps are not enough

**Run the pattern set, then read every remaining `"` in non-DEBUG code and classify it** (copy / data / identifier / log / SF Symbol / URL / comment), and report the counts.

A `private var`/`func` returning `String` with plain Swift interpolation scores **zero** on `Text(`, `Label(`, `Button(`, `TextField(`, `String(localized: "` and every accessibility-modifier grep — that shape hides whole VoiceOver values and unconverted views. Also grep `String(localized: "` on its own: English-as-key has no `Text(` anywhere near it.

**Exclude:** `#if DEBUG` regions and `#Preview` blocks (check the guard's real line range), logger calls, `accessibilityIdentifier` values, SF Symbol names, `UserDefaults`/`@AppStorage` keys, URL schemes.

## Never catalogue

- **Prompts and schema descriptions sent to an on-device model** (e.g. Foundation Models `@Guide(description:)`, instruction constants) — never rendered, and translating them changes what the model returns. Error messages from the same code *are* copy. Use a **line-level allowlist, never a folder rule** — a folder exclusion leaves the real copy there permanently English.
- **Legal text that must stay in its original language** (e.g. licence attributions). Mark it with an allowlist token; a guard must key on the **token's position** (its own line, or the line above the literal), never on a match count.
- Proper nouns and licence names are data. Borderline cases (a licence *status* like "Public domain") are a product call, not yours.

## Tests

- Every `*Copy` type has a `*CopyTests` file pinning its English. **Read-back assertions are weak**: `defaultValue` equals the catalogue value, so `String(localized: X) == "…"` passes whether the lookup succeeds *or silently falls back*.
- **The load-bearing tests assert interpolation, not lookup**: pin every number-formatting function above 1000. Two shapes:
  - **A visible function names the expected grouped form per locale**, `@Test(arguments:)` over at least `en_US` and `de_DE` (add `tr_TR` wherever a sign's position, not just its separator, moves — e.g. percentages). Name it for what it asserts (`…RendersGroupedDigitsAboveOneThousand`) — never `…RendersBareDigits…` on a function that groups; that name pins the bug as if it were the fix.
  - **A spoken function keeps a bare-digit assertion** and a `…RendersBareDigitsAboveOneThousand` name — a spoken figure never groups.
  Add one for every new function that takes a number, and mutate the fix (revert the format-style/locale change, confirm the test fails, restore) before trusting it.
- **Keep one sentinel string whose source default and catalogue value differ on purpose** — the only way to tell "catalogue wired" from "falling back to defaults".
- **A key typo is otherwise invisible.** The durable fix is a static check: parse `Localizable.xcstrings` and cross-check every key literal in source. It must handle **all three** construction forms — your helper, `String(localized:)`, and raw `LocalizedStringResource(...)` — or it reports false orphans.

## Verifying

- Run the full test suite **and** compile the app target — a package-only test run never builds the app. If catalogue guards live outside the Swift suite (scripts, linters), run those too; a copy change can break a check that only they read.
- **UI tests match copy by rendered label.** Changing those words breaks them silently — grep the UI tests before changing visible text.
- **Lowercasing a resolved noun to fit it mid-sentence is a defect**, invisible in English: `"Common at \(title.lowercased())"`. German capitalises every noun regardless of position, and a translator cannot undo a fold Swift applies *after* the lookup. Author one whole sentence per case instead. No English-only test catches it; a pseudolanguage run can.
- Copy is **byte-identical** through a conversion unless the change is declared. Copy values out of the source; never retype them — em dashes, curly apostrophes and `›` all bite.

## Red flags

- Writing `count == 1` near a catalogue key
- A bare `Int` inside a string literal
- `Text(verbatim:)` on anything with English words in it
- A sweep that only ran greps
- `bundle:` omitted, or `.module` used from a test target
- Citing a **line number** in a comment — cite a symbol or a token
- Writing "N call sites" or "the only X" in a comment or commit message without running the command first
