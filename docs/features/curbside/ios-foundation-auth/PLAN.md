# Plan: Curbside — iOS Foundation + Auth

**Epic**: curbside
**Sub-feature ID**: IOS-1
**Repo**: `Xomware/curbside-ios` (base branch `main`)
**Track**: iOS MVP
**Status**: Ready
**Created**: 2026-11-16
**Last updated**: 2026-11-16
**Depends on**: S1 none; S2 needs INF-1 (I3), BE-1 (B2), BE-5 (B11)

## Summary
S1 scaffolds the SwiftUI app (`com.Xomware.Curbside`) with an XcodeGen project, a minimal app shell, a test target and a CI build/test workflow. S2 adds the networking and auth layer: `Config`, an `APIClient` that decodes `{ data, error, meta }`, a Keychain token store, then hand-rolled Apple + Google sign-in through Cognito with onboarding and the link-another-sign-in path.

## Decisions
- **Project generation**: XcodeGen `project.yml`, with the generated `Curbside.xcodeproj` committed, as in `xomper-ios`. `xcodegen generate` after any target or file-layout change.
- **Deployment target: iOS 26.0, iPhone only, portrait.** S5 uses `MKReverseGeocodingRequest`, which is iOS 26+ (`CLGeocoder` is deprecated there). iPhone-only also drops the iPad screenshot set at review time.
- **Swift 6** language mode, `SWIFT_STRICT_CONCURRENCY = complete`, `SWIFT_DEFAULT_ACTOR_ISOLATION` left at the default (explicit `@MainActor` on view models, as in xomper).
- **Tests**: Swift Testing (`@Test`, `#expect`) in a hosted `CurbsideTests` target. Precedent repos use XCTest; ios-standards prefers Swift Testing for new code.
- **Layout**: `Curbside/{App,Services,Models,ViewModels,Views,Theme,Resources}`. `Resources` holds `Info.plist`, `Assets.xcassets` and bundled fonts.
- **CI**: one `ci.yml`, `macos-26` runner, `build-for-testing` + `test-without-building` on an iPhone 17 simulator, no signing. Follows `xomify-ios/pr-checks.yml` minus SwiftLint, xcpretty and Codecov. TestFlight is S10.
- **Signing**: automatic, team `A3HMLDS2AL` (same as xomper). CI builds with `CODE_SIGNING_ALLOWED=NO`.
- **No URL type in Info.plist for the callback.** `ASWebAuthenticationSession` with `.customScheme("curbside")` catches `curbside://auth/callback` without a registered URL type. Registering it would let any other app's link open Curbside for nothing.

## Implementation Steps

### S1 — Scaffold, app shell, CI (~60 logic lines, no deps)
Scope is narrower than the epic row: `Config`, `APIClient` and the Keychain store move to S2. Nothing would call them before sign-in exists, and S1 stays one idea.

Acceptance criteria:
- [ ] `xcodegen generate` from a clean checkout reproduces the committed project with no diff
- [ ] Bundle ID `com.Xomware.Curbside`, display name "Curbside", iOS 26.0, iPhone only, portrait, Swift 6 strict concurrency
- [ ] `Curbside/{App,Services,Models,ViewModels,Views,Theme,Resources}` exist; the app launches to a root view in the simulator
- [ ] `CurbsideTests` target (Swift Testing) runs from the `Curbside` scheme
- [ ] `.github/workflows/ci.yml` builds and tests on PRs to `main` and pushes to `main`
- [ ] Builds with zero warnings

### S2 — Networking + auth (~350, needs S1, I3, B2, B11)
Over budget as one PR, so it lands as two stacked PRs under one issue:

**S2a — `Config`, `APIClient`, Keychain store (~150, needs S1 only)**
- [ ] `Config` reads API base URL, Cognito domain and iOS client ID from build settings (xcconfig), not hard-coded
- [ ] `APIClient` sends the bearer token, decodes `{ data, error, meta }`, maps `error` to a typed `APIError` (`needs_onboarding`, 401, 409, 429, other)
- [ ] One refresh-and-retry on 401, never a loop
- [ ] Keychain store saves, reads and deletes the token set, `kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly`
- [ ] Tests: envelope decoding (success, error, malformed), 401 retry once, Keychain round-trip

**S2b — PKCE sign-in, refresh, onboarding (~200, needs S2a, I3, B2, B11)**
- [ ] "Sign in with Apple" and "Continue with Google" open `ASWebAuthenticationSession` at `/oauth2/authorize?identity_provider=SignInWithApple|Google` with S256 PKCE and `state`
- [ ] Callback `curbside://auth/callback` → code exchange at `/oauth2/token`; `state` mismatch is rejected
- [ ] Tokens persist across relaunch; access token refreshes with the refresh token; refresh failure signs out
- [ ] `GET /me` 404 `needs_onboarding` shows "New here" (`POST /me`) vs. "I already use Curbside with another sign-in"
- [ ] Link path signs in with the other provider, then runs a second PKCE flow and posts its ID token to `POST /me/identities`
- [ ] Sign-out clears Keychain and returns to sign-in
- [ ] Tests: PKCE verifier/challenge, authorize URL, state check, onboarding branch in the view model
- [ ] Verified in app against the live pool (no Apple/Google SDKs)

## Out of Scope
- Theme (IOS-2 S3), settings / sign-in methods list (IOS-5 S10), TestFlight workflow (S10)

## Risks / Tradeoffs
- Web-flow SIWA is less polished than the native sheet; accepted for MVP.
- iOS 26 minimum excludes devices that can't run it. Fine for a personal app; revisit before a public launch only if S5 can fall back to `CLGeocoder`.

## Manual prerequisites (Dom)
- Register the App ID `com.Xomware.Curbside` in the developer portal (capabilities: none needed for S1; Sign in with Apple is web-flow via Cognito, so no SIWA entitlement).
- App Store Connect record before S10 (TestFlight), which makes the bundle ID permanent.

## Open Questions
- [ ] Custom auth domain before TestFlight? Apple's sheet shows the prefix domain until then.

## Skills / Agents to Use
- ios-standards
