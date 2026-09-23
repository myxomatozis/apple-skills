# Sign in with Apple, revocation & App Store compliance reference (verified against iOS 27 docs)

## SignInWithAppleButton (SwiftUI)

```swift
SignInWithAppleButton(.signIn) { request in
    request.requestedScopes = [.fullName, .email]
} onCompletion: { result in
    // handle result
}
.signInWithAppleButtonStyle(.black) // .white / .whiteOutline also available
```

HIG requirements: the button must be prominent, at least as large as any other sign-in button on the screen, require no scrolling to reach, and use the system button (not a custom recreation).

## First-authorization gotcha

`fullName` and `email` are delivered **only on the first authorization**. Persist them immediately server-side — there is no later API to re-fetch them. Verify the `identityToken` JWT server-side; never trust a client-supplied email value.

## Credential state

- On launch/foreground, call `getCredentialState(forUserID:)` and handle all four states: `.authorized`, `.revoked` (sign the user out), `.notFound`, `.transferred`.
- Store the Apple user id in the Keychain so it survives across launches.

## Revocation notification

Observe `ASAuthorizationAppleIDProvider.credentialRevokedNotification` and log the user out when it fires — this is how the app learns the user revoked access from their Apple ID settings rather than in-app.

## Guideline 4.8 (verbatim requirements)

Any third-party/social login used for primary account setup requires offering a login option that:
- limits data collection to the user's name and email address,
- lets the user hide their email address,
- does not collect data for advertising purposes without the user's consent.

Sign in with Apple is the easiest way to satisfy this; a passkey or email-only option can also qualify.

## Korea server-to-server note

From January 1, 2026, Korea-based developers must register a server-to-server notification endpoint on their Services ID.

## Account deletion (Guideline 5.1.1(v), mandatory, release blocker)

- In-app full account deletion is mandatory.
- If the account uses Sign in with Apple, the app MUST revoke the credential server-side by calling `https://appleid.apple.com/auth/revoke`, using a token obtained from `/auth/token` via the authorization code — per Apple Technical Note TN3194.
- App Review behavior: reviewers test the full loop — sign in → delete account → sign in again — to confirm deletion actually revoked access.
- Design rules for the deletion flow: include a confirmation step, require re-authentication, and never force the user out to a phone call or website to complete deletion.

## Logout checklist

On logout, all of the following must happen:
- Delete Keychain session items.
- Revoke the refresh token server-side.
- Clear `URLCache` and `HTTPCookieStorage`.
- Reset any user-scoped in-memory state.
- If a non-ephemeral `ASWebAuthenticationSession` was used for OAuth, note that it leaves the IdP's SSO cookie behind — use an OIDC `end_session` call, or start the session ephemeral in the first place, to actually terminate SSO.

## ASWebAuthenticationSession (third-party OAuth)

- Use `ASWebAuthenticationSession` only — never `SFSafariViewController`, never `WKWebView` — with PKCE.
- iOS 17.4+: use the typed `ASWebAuthenticationSession.Callback` — `.https(host:path:)` (requires associated domains) or `.customScheme("myapp")` (guaranteed delivery to the initiating app; shows a one-time consent alert).
- `prefersEphemeralWebBrowserSession = true` skips the consent alert and cookies entirely, which kills IdP SSO. Choose this per product based on whether SSO continuity is wanted.
- Keep a strong reference to the session for its lifetime; supply a `presentationContextProvider`; handle the `.canceledLogin` error.

## Architecture (session/token ownership)

An `AuthSession`/`AuthManager` actor owns token state and exposes an observable `AuthState` enum (`unauthenticated` / `authenticated(User)` / `locked`) that drives the SwiftUI root view switch. At cold start, check both the Sign in with Apple credential state and Keychain validity before deciding the initial `AuthState`.
