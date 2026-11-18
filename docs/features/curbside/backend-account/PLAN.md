# Plan: Curbside — Backend Account

**Epic**: curbside
**Sub-feature ID**: BE-5
**Repo**: `curbside-backend`
**Track**: iOS MVP
**Status**: Ready
**Created**: 2026-11-16
**Last updated**: 2026-11-17
**Depends on**: B11 needs BE-1 (B2) + INF-2 (I8), both live; B12 also needs BE-2 (B5, B6)

## Summary
Explicit in-app identity linking (never by email), unlinking, full JSON export, and hard delete including every linked Cognito user. Linking gates iOS sign-in; export and delete gate App Store submission.

## Approach
Auth flow "Account linking" in the epic.

### B11 — `me_link`, `me_unlink`
- **`POST /me/identities {id_token}`**: the caller's access token picks the account. The ID token is verified with PyJWT against the pool JWKS (`https://cognito-idp.<region>.amazonaws.com/<USER_POOL_ID>/.well-known/jwks.json`, cached per container): RS256, issuer, `aud` in `CLIENT_IDS`, `token_use=id`, `exp`, `auth_time` within 10 min. Provider comes from `cognito:username`. Then one transaction of two Puts, `IDENT#<sub2>` conditional on `attribute_not_exists(pk)`, and `ACCT#<a>/IDENT#<sub2>`. Returns the new identity `{sub, provider, created_at}`, 201. Not the account: `get_account` Queries, and the role has GetItem + PutItem only.
  - Invalid token → 400 `invalid_id_token`. Same sub as the caller → 409 `already_linked`.
  - **Any** sub that already has an `IDENT#` item → 409 `identity_in_use`. This is stricter than the epic's "already owns data": telling an empty account from one with data needs a gsi1 `Query`, which `me_link`'s role doesn't have. The user deletes the other account (B12), then links.
  - `PyJWT[crypto]` joins `requirements.txt`. The layer build installs manylinux wheels, so `cryptography` ships fine.
- **`DELETE /me/identities/{sub}`**: Query the caller's `ACCT#<a>/IDENT#` items. Not one of them → 404 `identity_not_found`. Only one left → 409 `last_identity`. Else one transaction deletes `IDENT#<sub>` and `ACCT#<a>/IDENT#<sub>` (two Deletes; the role has no BatchWriteItem). Returns the account.
- **Account-id cache removed.** `get_account_id` reads `IDENT#<sub>` (consistent) on every request. Revoking Cognito tokens can't close the gap: the HTTP API JWT authorizer validates access tokens locally and keeps accepting one for up to an hour. With no cache, an unlinked or deleted sub gets 404 `needs_onboarding` on its next request in every container. Cost is one GetItem per request.

### B12 — `me_export`, `me_delete`
Access pattern 7.
- **`GET /me/export`**: `{account, maps:[{map, households}]}`. `account` as `GET /me`. Each map is one partition Query: `META` → map view, `HH#…` → households with people and pets, `NOTE#<h>#…` attached as `private.notes`. Quota and membership items stay out.
- **`DELETE /me`**:
  1. Re-auth: the caller's access-token `auth_time` must be within 10 min, else 403 `reauth_required`. If the account has an Apple identity, the caller must also be signed in with Apple. Apple's `/auth/revoke` returns 200 for a dead token too, so freshness is checked here.
  2. Apple revoke: `ListUsers` filter `sub = "<caller sub>"` returns the username and `custom:apple_at`. POST `appleid.apple.com/auth/revoke` with the Services ID, an ES256 client secret (key from SSM `/curbside/apple/private-key`, `kid` = key ID, `iss` = team ID, 5 min) and `token_type_hint=access_token`. Token missing or the call fails → TN3194 fallback: carry on and return `apple_revoke: "manual"` with `https://support.apple.com/en-us/102571`.
  3. Maps: gsi1 → each membership. Owner → `delete_map` (footprint objects, partition, `MEMBER#` last). Otherwise only the `MEMBER#` item.
  4. Cognito: for every linked sub, `ListUsers` by sub → `AdminDeleteUser`. A user already gone is skipped.
  5. `ACCT#<a>` partition, then the `IDENT#` items, the caller's `ACCT#/IDENT#` and `IDENT#` last. Every earlier step is idempotent, so a delete that dies part-way can be retried: the caller's `IDENT#` is the last thing removed, and the JWT authorizer keeps accepting the access token after `AdminDeleteUser`.
  - Returns `{deleted: true, apple_revoke: "revoked"|"manual"|"none", manual_revoke_url?}`.
- `AdminGetUser` (granted) isn't needed: `ListUsers` returns the attributes.

## Affected Files / Components
| File / Component | Change | Why |
|-----------------|--------|-----|
| `lambdas/common/auth.py` | drop `_accounts` cache | B11 |
| `lambdas/common/accounts.py` | `link_identity`, `unlink_identity` | B11 |
| `lambdas/common/id_tokens.py` | JWKS verification | B11 |
| `lambdas/me_link`, `lambdas/me_unlink` | new | B11 |
| `lambdas/me_export`, `lambdas/me_delete` | new | B12 |

## Implementation Steps
- [ ] B11 — `me_link` + `me_unlink` + cache removal (~90, needs B2, I8) — curbside-backend#14, stacked on #13
- [ ] B12 — `me_export` + `me_delete` (maps, partitions, Cognito users) (~160, needs B5, B6, I8) — curbside-backend#24

## Acceptance criteria (B11)
- [ ] A valid ID token for a new sub links it; `GET /me` from that sub returns the same account.
- [ ] Bad signature, wrong issuer/audience, access token in place of an ID token, expired, `auth_time` > 10 min → 400.
- [ ] Linking a sub with an existing `IDENT#` → 409, nothing written.
- [ ] Unlinking the last identity → 409; another account's sub → 404.
- [ ] After unlink, the removed sub resolves to `needs_onboarding` in the same warm container.
- [ ] Live: link a second provider and unlink it with real tokens.

## Acceptance criteria (B12)
- [ ] Export holds every map, household, person, pet and note the account owns, nothing from other accounts.
- [ ] Delete removes owned partitions, footprint objects, the `ACCT#` partition, every `IDENT#`, and every linked Cognito user.
- [ ] Stale `auth_time` → 403; Apple-linked account deleted from a Google sign-in → 403.
- [ ] Revoke failure or missing `custom:apple_at` → deletion completes with `apple_revoke: "manual"`.
- [ ] A retry after a mid-delete crash finishes the job.

## Out of Scope
- Account merging when both accounts hold data

## Risks / Tradeoffs
- **Unlink race**: two concurrent unlinks of different subs can both pass the "not the last" check and leave zero identities. Guarding it needs a counter (`UpdateItem`) or `ConditionCheckItem`, neither granted. Accepted: it needs two devices unlinking at once. The data stays intact but unreachable until the operator re-adds an `IDENT#` item.
- Partial failure in `me_delete`: ordered so a retry finishes it (see B12 step 5).
- Re-auth for Apple-linked accounts means the delete button runs a Sign in with Apple round first (S10). If Cognito never populates `custom:apple_at` (I3 unknown), every delete takes the TN3194 manual path.

## Open Questions
- [x] Apple account-deletion rule: answered in the epic (re-auth, `/auth/revoke` with our own client secret).

## Skills / Agents to Use
- lambda-handler, backend-standards, database, testing

- [ ] Sign in with Apple server-to-server notification endpoint (account-delete / email-forwarding events). Register its URL on the primary App ID `com.Xomware.Curbside`, left blank on 2026-11-17.
