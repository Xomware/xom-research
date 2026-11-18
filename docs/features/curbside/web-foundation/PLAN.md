# Plan: Curbside — Web Foundation + Account

**Epic**: curbside
**Sub-feature ID**: WEB-1
**Repo**: `curbside-frontend`
**Track**: Web follow-up
**Status**: Ready
**Created**: 2026-11-16
**Last updated**: 2026-11-17
**Depends on**: INF-3 (I9)

## Summary
Angular scaffold with PKCE sign-in, onboarding and link flow, deploy pipeline, plus export and account settings. Same API, no backend changes.

## Approach
Same Cognito PKCE code flow as iOS against the web client from I9. Deploy via `deploy-frontend-s3.yml@v1`. TypeScript strict, standalone components.

## Affected Files / Components
| File / Component | Change | Why |
|-----------------|--------|-----|
| `curbside-frontend/` scaffold, auth service, onboarding | new | W1 |
| settings + export | new | W5 |

## Implementation Steps
- [ ] W1 — Angular scaffold + PKCE auth + onboarding/link + deploy (~200, needs I9)
- [ ] W5 — Export download + account settings (sign-in methods, delete) (~100, needs W1)

## Out of Scope
- Map and households (WEB-2), capture (WEB-3)

## Risks / Tradeoffs
- Token storage in the browser: decide memory vs. sessionStorage for refresh tokens.

## Open Questions
- [x] Refresh token storage on web? **Answer:** `sessionStorage`, see Plan corrections.

## Skills / Agents to Use
- angular-component, frontend-standards

## Plan corrections (2026-11-17)

**Split.** W1 at ~200 lines did not fit scaffold + theme + deploy + auth + onboarding, and W5 grew with the
delete re-auth flow. Stacked PRs on `main`, one issue per W row:

1. W1a — bare `ng new` output (generated, Angular 22, Vitest), nothing hand-written
2. W1b — theme tokens + Overpass + sign styles, environment config, `.claude/CLAUDE.md`, deploy workflow (~120)
3. W1c — PKCE, Cognito token exchange, session store, API client with one refresh-and-retry (~200)
4. W1d — sign-in, `/auth/callback`, onboarding (new or link), route guards, signed-in home stub (~200)
5. W5a — settings: sign-in methods (list, link, unlink) and export download (~150)
6. W5b — delete account with the re-auth round trip and the Apple manual-revoke link (~100)

**Refresh token storage: `sessionStorage`.**
- Memory-only loses the session on every reload, and the PKCE flow is itself a full-page redirect.
- `localStorage` keeps a 30-day bearer token readable by any XSS for a month, on an app holding data about third parties.
- `sessionStorage` survives reloads and the redirects, dies with the tab, and stays per-tab.
- Sign-out also calls `/oauth2/revoke` so a copied refresh token stops working.
- Mitigation that belongs to I9: a CSP response-header policy on the distribution (`script-src 'self'`,
  `connect-src` limited to the API and the Cognito domain). No third-party scripts ship in the app.

**Link from onboarding without a second redirect.** The orphan sign-in's ID token is fresh (seconds old)
when the user picks "I already use Curbside with another sign-in". The client stashes it for the round trip,
signs in with the other provider, then posts the stashed token to `POST /me/identities`. Only if the
server answers `invalid_id_token` (older than 10 minutes) does it run a second PKCE flow for that sign-in.
The iOS app has not built this path yet: its onboarding only offers "Use a different sign-in" (sign out).

**Delete re-auth.** `DELETE /me` is tried first. On 403 `reauth_required` the user signs in again (Apple
when an Apple sign-in is linked) and lands back on settings with the confirm step open. Nothing deletes
without a click after the redirect.

**Config.** `src/environments/environment.ts` is committed with the prod values (web client
`1aqfq9g4nm92ppr2s0l1kqnc10` from I9) for local dev. The deploy rewrites it from SSM through the
workflow's `ssm` input (`/curbside/api/url`, `/curbside/cognito/domain`, `/curbside/cognito/web-client-id`)
via `scripts/write-environment.mjs`, which reads only those three names: the deploy env also holds every
Infisical `/curbside` secret. The redirect URI is `location.origin + /auth/callback`.

**Sign-out.** Local clear + `/oauth2/revoke`, then Cognito `/logout` with the bare origin as `logout_uri`.
After an account delete it signs out locally only: the logout reload would wipe the one-time result notice.

**From I9 (applied 2026-11-17):** bucket and alias `curbside.xomware.com`, `AWS_FRONTEND_ROLE_ARN` in
Infisical `/curbside`. Still to confirm: CloudFront maps 403/404 to `/index.html`, or a reload on
`/auth/callback` or `/settings` fails; a CSP response-header policy is still recommended.
