---
name: ios-auth
description: Use when implementing login, signup, sign-out, session or token handling, Keychain storage, biometrics (Face ID), account deletion, or OAuth in an iOS/Apple-platform app. Covers passkeys, Sign in with Apple, and App Store auth compliance.
---

# iOS Auth — Identity, Sessions, Keychain

> iOS 27 APIs verified against June–July 2026 beta documentation; re-verify at GA.

## Recommended flow order
1. Signup: passkey-first via ASAuthorizationAccountCreationProvider (one native sheet). Handle .deviceNotConfiguredForPasskeyCreation (fall back to traditional signup) and .preferSignInWithApple (route to SiwA — prevents duplicate accounts).
2. Sign in with Apple as the standing alternative (SignInWithAppleButton, at least as large as any other login button, visible without scrolling).
3. Passwords are legacy-only: after any successful password sign-in, fire a passkey registration with requestStyle: .conditional (silent; failures are silent by design — never surface them; keep the password valid).
4. Login screen: AutoFill-assisted passkey assertion (performAutoFillAssistedRequests()) active on appear, username field marked .textContentType(.username); cancel the controller on disappear.

## Tokens & Keychain (hard rules)
- Tokens live in the Keychain only. Never UserDefaults, files, or @AppStorage.
- Refresh token: kSecClassGenericPassword + kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly (background refresh works; no backup/sync). Always the ThisDeviceOnly variants.
- Access token: in memory, owned by an actor; serialize refresh through that single actor (concurrent 401s await one refresh).
- High-value secrets: SecAccessControl with .biometryCurrentSet (SEP-enforced; invalidated when biometric enrollment changes). This is stronger than any LAContext-only gate — an if-statement can be bypassed, a Keychain ACL cannot.
- Wrapper: small typed wrapper in your own code (actor, or Sendable struct) over SecItemAdd/CopyMatching/Update/Delete. No third-party Keychain dependency.

## Biometrics
- Fresh LAContext per attempt; canEvaluatePolicy first; read biometryType after it — never hardcode "Face ID" in copy.
- Default policy: .deviceOwnerAuthentication (biometrics + passcode fallback). NSFaceIDUsageDescription is mandatory in Info.plist.
- Re-lock on scenePhase == .background.

## Sign in with Apple hygiene
- fullName/email arrive ONLY on first authorization — persist them immediately.
- On launch/foreground: getCredentialState(forUserID:); observe credentialRevokedNotification → sign out.
- Verify the identityToken JWT server-side; never trust client-supplied email.

## App Store compliance gates (release blockers)
- Guideline 4.8: offering any third-party login requires a privacy-equivalent option (SiwA or passkey/email-only qualifies).
- Guideline 5.1.1(v): in-app full account deletion is mandatory; with SiwA it MUST revoke via appleid.apple.com/auth/revoke server-side. Reviewers test sign-in → delete → sign-in again.
- Logout must delete Keychain session items, revoke server-side, and clear URLCache/HTTPCookieStorage.

## OAuth (third-party IdPs)
- ASWebAuthenticationSession only (never WKWebView), with PKCE; typed Callback (.https preferred, .customScheme acceptable); decide prefersEphemeralWebBrowserSession per product (ephemeral = no consent alert, no SSO).

## References
- references/passkeys.md — registration/assertion code, Account Creation API, conditional upgrades, Signal APIs, associated-domains setup.
- references/keychain-biometrics.md — accessibility class decision table, SecAccessControl patterns, LAContext error handling.
- references/apple-signin-compliance.md — SiwA implementation, revocation, account deletion flow, 4.8/5.1.1 details.

## Related concerns
- Login screen visuals: keep the form in the content layer (no Liquid Glass wrap); glass belongs on the primary action only. System passkey/SiwA sheets are already Liquid Glass.
- Auth state architecture: an actor owns tokens and exposes an observable `AuthState` enum that drives the root view (see references/apple-signin-compliance.md).
