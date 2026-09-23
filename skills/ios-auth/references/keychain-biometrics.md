# Keychain & biometrics reference (verified against iOS 27 docs)

## Accessibility class decision table

| Item | kSecAttr class |
|---|---|
| Auth/refresh tokens | `kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly` — background refresh works; not backed up/synced |
| Foreground-only sensitive data | `kSecAttrAccessibleWhenUnlockedThisDeviceOnly` |
| Highest-sensitivity items | `kSecAttrAccessibleWhenPasscodeSetThisDeviceOnly` |

- Always prefer the `ThisDeviceOnly` variant for credentials.
- Never use `kSecAttrAccessibleAlways` — it is deprecated.

## Storing tokens

- `kSecClassGenericPassword`, with `kSecAttrService` + `kSecAttrAccount` identifying the item and the token bytes in `kSecValueData`.
- Wrapper: a small typed wrapper in your own code (actor or Sendable struct) over `SecItemAdd`/`SecItemCopyMatching`/`SecItemUpdate`/`SecItemDelete`. Third-party Keychain wrappers are unnecessary.
- Secure Enclave keys are for non-exportable signing only (attestation/DPoP) — not for storing the tokens themselves.
- Never store tokens in UserDefaults, files, or `@AppStorage`.

## Biometric-gated items (SecAccessControl)

```swift
SecAccessControlCreateWithFlags(
    nil,
    kSecAttrAccessibleWhenPasscodeSetThisDeviceOnly,
    .biometryCurrentSet,
    &error)
```

Set the result as `kSecAttrAccessControl` on the Keychain item. `kSecAttrAccessControl` is mutually exclusive with a plain `kSecAttrAccessible` value — use one or the other, not both.

### `.biometryCurrentSet` vs `.userPresence` trade-off

- `.biometryCurrentSet` is stronger than `.biometryAny`/`.userPresence` for high-value items: it is invalidated automatically when the enrolled biometric set changes (e.g. a new face/fingerprint is added), forcing re-authentication.
- `.userPresence` is the right choice when lockout recovery matters more than that invalidation guarantee.
- This access control is SEP-enforced (Secure Enclave Processor) — stronger than any LAContext-only gate, since an `if` statement guarding UI can be bypassed but a Keychain ACL cannot.

## Access groups

- Use a non-default access group only to share Keychain items between your own apps/extensions: `$(AppIdentifierPrefix)com.company.shared`.
- Otherwise, leave items in the default private access group.

## LAContext lifecycle

- Create a fresh `LAContext` per attempt — do not reuse a context across authentication attempts.
- Call `canEvaluatePolicy` first.
- Read `biometryType` (`.faceID` / `.touchID` / `.opticID` / `.none`) only *after* `canEvaluatePolicy` — never hardcode "Face ID" in UI copy.
- Evaluate in a `task`/`.onAppear` on the lock-screen view; re-lock the app when `scenePhase == .background`.

## Policy choice table

| Policy | Behavior | Notes |
|---|---|---|
| `.deviceOwnerAuthentication` | Biometrics with passcode fallback | Recommended default |
| `.deviceOwnerAuthenticationWithBiometrics` | Biometrics only, no passcode fallback | Must handle `.biometryLockout` explicitly |

- `localizedFallbackTitle` is customizable; setting it to `""` hides the fallback button entirely.
- `NSFaceIDUsageDescription` is mandatory in Info.plist — biometric prompts fail without it.

## LAError cases to handle

- `.userCancel`
- `.userFallback`
- `.biometryNotAvailable`
- `.biometryNotEnrolled`
- `.biometryLockout`

## Biometrics + Keychain, together

Biometric gating at the LAContext/UI level is UX-level protection only. Pair it with a Keychain access control (`.biometryCurrentSet`) so the token itself requires user presence to read — not just the screen in front of it. iOS 18+ also lets users lock any app system-wide, which is a separate, OS-level mechanism from either of the above.
