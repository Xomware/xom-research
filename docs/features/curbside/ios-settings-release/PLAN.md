# Plan: Curbside — iOS Settings + Release

**Epic**: curbside
**Sub-feature ID**: IOS-5
**Repo**: `Xomware/curbside-ios` (base branch `main`)
**Track**: iOS MVP (closes it)
**Status**: Ready (S10a–S10e in review: PRs #34–#38, stacked on S9)
**Created**: 2026-11-16
**Last updated**: 2026-11-17
**Depends on**: IOS-1 (S2), BE-5 (B11, B12, live); 0b done. TestFlight in practice needs IOS-2 to IOS-4 merged.

## Summary
Face ID / passcode lock, settings (sign-in methods, export share sheet, delete account, attribution, privacy policy link), App Store privacy label and review notes, and the TestFlight workflow.

## Backend facts (from `curbside-backend/lambdas`, verified under moto)
- `GET /me` → `{account_id, display_name, created_at, identities: [{sub, provider: apple|google, created_at}]}`.
- `POST /me/identities {id_token}` links a second sign-in. The ID token must be fresh (auth_time ≤ 10 min). 409 `identity_in_use` when that sign-in already has its own account; 409 `already_linked`.
- `DELETE /me/identities/{sub}` → the account; 409 `last_identity`.
- `GET /me/export` → `{account, maps: [{map, households}]}`, notes included.
- `DELETE /me` → 403 `reauth_required` unless the access token's `auth_time` is ≤ 10 min old **and**, when Apple is linked, the session is an Apple one. Returns `{deleted, apple_revoke: revoked|manual|none, manual_revoke_url?}`.
- The original plan's route names (`GET/POST/DELETE /me/identities`) were close: listing is `GET /me`, not `GET /me/identities`.

## Decisions
- **App lock**: `LAContext.deviceOwnerAuthentication` (Face ID, falling back to the passcode). On by default, toggle in settings. Locks at launch and whenever the app returns from the background. While not active, a plain cover hides the map from the app switcher. A device with no passcode can't lock; the toggle says so. Demo mode never locks.
- **Linking a sign-in** runs a second PKCE flow for the other provider and posts its ID token. Those tokens are never stored: the session stays the current one. This also finishes S2b's unchecked "link path" item, from settings rather than onboarding.
- **Unlink** is offered only with two or more sign-ins.
- **Delete** runs a fresh sign-in first: Apple when Apple is linked, else the account's only provider. That replaces the stored session, then `DELETE /me`, then sign-out. `apple_revoke: manual` shows Apple's manual-revoke link before signing out.
- **Export** writes `GET /me/export` to a temporary `curbside-export.json` and opens the share sheet. The file is deleted when the sheet closes.
- **Privacy policy link**: `https://curbside.xomware.com/privacy`. That page does not exist yet; it ships with the web track (WEB-1) or by hand before review.
- **Attribution**: © OpenStreetMap contributors (ODbL, links to the copyright page), Overpass fonts (SIL OFL 1.1), Apple Maps.
- **Footprint refresh** (deferred here from IOS-2): not built. Nothing in MVP needs it yet; the map retries pending footprints on every load.
- **TestFlight**: `testflight.yml`, `workflow_dispatch` only until the App Store Connect record exists. Same shape as `xomfit-ios`: Infisical `/shared` + `/curbside`, archive with cloud signing via the ASC API key, export, `altool` upload. Build number from UTC time + run id.

## Implementation Steps
Stacked PRs. Line counts are hand-written logic.

- [x] **S10a — App lock** (~80, PR #34)
  - Tests: lock at launch, on return from background, toggle off, no passcode
- [x] **S10b — Settings screen** (~90, PR #35)
  - Settings sheet from the map header menu: app lock toggle, privacy policy, attribution, version, sign out
- [x] **S10c — Sign-in methods** (~160, PR #36, own PR at the coordinator's request)
  - List from `GET /me`; link the other provider (`POST /me/identities` with a fresh ID token, session kept); unlink (`DELETE /me/identities/{sub}`), hidden for the last one. Mirrors the web settings flow in `curbside-frontend`
  - Tests: link body, session tokens unchanged, unlink, `identity_in_use` / `already_linked` / `last_identity` messages
- [x] **S10d — Export + delete account** (~150, PR #37)
  - Tests: export file, delete with reauth (Apple when linked), `manual` shows the link, 403 message
- [x] **S10e — TestFlight workflow + release docs** (YAML, PR #38; docs in `RELEASE.md` here)

## Out of Scope
- Web settings (WEB-1)
- The privacy policy page itself (WEB-1)

## Risks / Tradeoffs
- Review can reject on SIWA token revocation. Covered by B12; when Cognito holds no Apple token the app shows the manual-revoke link (TN3194).
- `xcodebuild -allowProvisioningUpdates` with an API key needs an Admin-role key for cloud signing. The `/shared` key already signs `xomfit-ios`.

## Open Questions
- [x] Apple `/auth/revoke` on delete: needed for review? Yes; B12 does it.
- [ ] Anthropic retention wording in the privacy policy (draft in `RELEASE.md`).
- [ ] Custom auth domain before TestFlight? Not blocking; Apple's sheet shows the Cognito prefix domain.

## Plan corrections (2026-11-17)
- `POST /me` returns the profile without `identities`, so `Account.identities` is optional and the sign-ins section fetches `GET /me` itself.
- Delete checks the fresh sign-in's `sub` against the account's sign-ins before adopting it, so a wrong Apple ID can't replace the session with an orphan.
- Onboarding's "I already use Curbside with another sign-in" (curbside-ios#8) is still open; settings covers linking for signed-in users.
- One PR (~180) became five (~480 measured); sign-in methods split out on request.
- Footprint refresh moved out: see Decisions.
- Listing identities is `GET /me`.

## Skills / Agents to Use
- ios-standards; review with "verified in app"
