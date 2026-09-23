# Passkeys reference (verified against iOS 27 docs)

## Associated domains setup (prerequisite)

- Add the Associated Domains capability with `webcredentials:example.com`. The relying-party (RP) identifier used in code must match this domain exactly.
- Host `https://example.com/.well-known/apple-app-site-association` over HTTPS, with no redirects, served as `application/json`:
  ```json
  {"webcredentials": {"apps": ["<TeamID>.<bundle-id>"]}}
  ```
- The AASA file is fetched through Apple's CDN and cached — changes do not propagate instantly.
- **Dev mode caveat**: use `webcredentials:example.com?mode=developer` during development to bypass the CDN cache. Remove `?mode=developer` before submitting App Store builds — it must not ship.

## Registration

```swift
let provider = ASAuthorizationPlatformPublicKeyCredentialProvider(relyingPartyIdentifier: "example.com")
let request = provider.createCredentialRegistrationRequest(
    challenge: challengeFromServer,   // single-use, server-generated, >=16 bytes
    name: userName,
    userID: userIDFromServer)         // stable opaque user handle (Data)
```

SwiftUI: read `@Environment(\.authorizationController)`, then `try await authorizationController.performRequest(request)`. Send `rawAttestationObject`, `rawClientDataJSON`, and `credentialID` to the server.

- **Zeroed-AAGUID caveat**: consumer iCloud Keychain passkeys report a zeroed AAGUID. Don't depend on attestation to distinguish authenticator models or vendors.

## Assertion (sign-in)

`provider.createCredentialAssertionRequest(challenge:)`. Three presentation modes:

1. **AutoFill-assisted (recommended default)** — call `performAutoFillAssistedRequests()` when the sign-in screen appears. Suggestions surface on the QuickType bar for a field marked `textContentType(.username)`. Cancel the controller when leaving the screen. Use this as the default login-screen behavior.
2. **Modal on button tap** — `performRequests()`. Presents a sheet with cross-device QR fallback. Use when the user explicitly taps a "Sign in with passkey" button rather than relying on AutoFill.
3. **Local-only** — `performRequests(options: .preferImmediatelyAvailableCredentials)`. Fails fast with `.canceled` if no local credential exists. Caveat: does not extend to third-party credential managers (1Password etc.) — use only when a fast, no-fallback local check is specifically wanted.

- `ASAuthorizationAppleIDProvider` requests and passkey assertion requests can be mixed in one controller call, producing a single sheet.
- iOS 18+: the WebAuthn PRF extension (`ASAuthorizationPublicKeyCredentialPRFAssertionInput`) is available for deriving symmetric keys.

## New in iOS 26 (WWDC25) — five enhancements

### a) Account Creation API

`ASAuthorizationAccountCreationProvider` enables passwordless signup in one native sheet:

```swift
let provider = ASAuthorizationAccountCreationProvider()
let request = provider.createPlatformPublicKeyCredentialRegistrationRequest(
    acceptedContactIdentifiers: [.email, .phoneNumber],
    shouldRequestName: true,
    relyingPartyIdentifier: "example.com",
    challenge: try await fetchChallenge(),
    userID: try await fetchUserID())
let result = try await authorizationController.performRequest(request)
if case .passkeyAccountCreation(let account) = result { /* register on backend */ }
```

Error cases to handle:
- `.deviceNotConfiguredForPasskeyCreation` — fall back to traditional signup.
- `.canceled` — user dismissed the sheet.
- `.preferSignInWithApple` — route to SiwA instead, to avoid creating a duplicate account.

### b) Automatic passkey upgrades

After a successful password sign-in, fire a registration request with `requestStyle: .conditional` — no UI, silent creation, and it fails silently. Keep the password valid regardless of outcome; never surface a failure to the user.

### c) Signal APIs (`ASCredentialUpdater`, iOS 26)

Three calls, each fired after the corresponding server-side mutation:
- `reportPublicKeyCredentialUpdate(relyingPartyIdentifier:userHandle:newName:)` — after a username/email change.
- `reportAllAcceptedPublicKeyCredentials(...)` — after server-side passkey deletion.
- `reportUnusedPasswordCredential(domain:username:)` — after password removal.

### d) Passkey management endpoints

Serve `/.well-known/passkey-endpoints` as JSON:
```json
{"enroll": "https://…", "manage": "https://…"}
```

### e) Import/export

`ASCredentialExportManager` / `ASCredentialImportManager` implement FIDO CXP (Credential Exchange Protocol). No RP-side action is required to support this.

## Known regression

- **iOS 26.2**: a WKWebView `isUserVerifyingPlatformAuthenticatorAvailable()` bug. Be aware of it if a web-based fallback flow checks platform authenticator availability from within WKWebView on iOS 26.2.

## Recommended architecture (summary)

Passkey-first signup via the Account Creation API → SiwA fallback (honor `.preferSignInWithApple`) → password only as legacy, with `.conditional` auto-upgrade. AutoFill-assisted assertion on the login screen. Fire Signal APIs on account mutations (username/email change, passkey deletion, password removal).
