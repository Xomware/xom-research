# Plan: curbside (working name)

**Status**: Ready
**Created**: 2026-11-16
**Last updated**: 2026-11-17

## Summary
A private address book for the people on your street. You tap a house on a map, or open a near-me sheet on the sidewalk, and see who lives there: adults, kids by birth year, pets, and your notes. New facts go in by voice or text, Claude Haiku parses them into a diff, and you confirm it. The iOS app ships first. An Angular desk companion follows on the same API with no backend changes. Success means Dom can stand on the sidewalk and get a kid's name in under 5 seconds, and log a conversation in under 15.

## Approach
This builds Option 1 from `BRAINSTORM.md`, plus its "Decision (2026-11-16)" and "self-published profile cards" sections:

- **Clients**: native SwiftUI + MapKit on iOS, Angular + MapKit JS on web.
- **Backend**: one shared stack. API Gateway HTTP API, Python Lambdas, one DynamoDB table and one Cognito pool.
- **Sign-in**: Apple and Google on both clients, federated through Cognito. Account linking is explicit and in-app, never matched by email (see Auth).
- **Footprints**: Overpass (OSM) per place, cached as GeoJSON in S3. Pins are the fallback. Overture is added only if the coverage spike says OSM is poor.
- **Data split**: every household entry has a `private` block (mine, never shared) and a `published` block (read-only, sourced from a connected user's card). MVP always returns `published: null`, but both clients render the split from day one, so profile cards add data without a migration.
- **Name**: Curbside (decided 2026-11-16). Every resource takes its name from `var.app_name = "curbside"`; repos are `curbside-*` (public, Xomware org); bundle ID `com.Xomware.Curbside`, matching the existing iOS apps. Renaming stateful AWS resources (pool, table, bucket) replaces them, and the bundle ID is permanent once an App Store Connect record exists.

### Precedent followed
- **Terraform**: one `terraform/` dir per infra repo with a file per concern (`main.tf`, `providers.tf`, `acm_api.tf`, `lambda_layers.tf`, `lambdas_<group>.tf` with a `for_each` locals map and `templates/lambda_stub.zip`), as in `xomify-infrastructure`. State goes in `xomware-terraform-state` under key `curbside/terraform.tfstate`. It runs through `Xomware/github-actions` `terraform.yml@v1`: plan on PR, apply on merge. Never run locally.
- **Cognito**: a standalone pool, following `today-in-sports-infrastructure/terraform/cognito.tf`: `deletion_protection`, `prevent_destroy`, Google IdP with the pinned endpoint fields, PKCE code flow, no client secret. It deliberately does not use the shared `xomware_users` pool. This app holds sensitive data about third parties and should be able to live or die alone. The shared pool's `users-presignup-link` email auto-link is exactly what we must not do here.
- **Handlers**: `lambdas/<name>/handler.py` with `@handle_errors(HANDLER)`, `success_response`, `require_fields` and the `lambdas/common` helpers, as in `xomify-backend/lambdas/ratings_track/handler.py`. Responses use `{ data, error, meta }`, and pydantic v2 validates at the boundary.
- **Secrets**: Infisical project `code`, folder `curbside`, with names matching TF vars in caps. Runtime secrets go Infisical → TF var → `aws_ssm_parameter` SecureString. Lambdas read SSM at cold start, and nothing is hand-set in SSM.

## Architecture

### Repos
| Repo | Contents |
|---|---|
| `curbside-infrastructure` | Terraform: KMS, DynamoDB, S3, Cognito, HTTP API, Lambdas (stub zips), layers, SSM, ACM, Route53 records in the shared `xomware.com` zone, and later web hosting |
| `curbside-backend` | Python Lambdas + `lambdas/common`, pytest + moto, deployed with `deploy-lambda-python.yml@v1` |
| `curbside-ios` | SwiftUI app, no third-party auth SDK |
| `curbside-frontend` | Angular desk companion (web phase) |

Repos are public in the Xomware org per the visibility convention. All fixtures must be synthetic: no real neighbor names in tests, recordings or screenshots.

### DynamoDB: single table
Table `${app_name}`, on-demand, PITR on, SSE with a customer-managed KMS key, TTL attribute `ttl`. Keys are `pk`/`sk` plus one GSI, `gsi1pk`/`gsi1sk`. IDs are ULIDs.

**Why single-table**: every access pattern is scoped to one map or one account, so a whole map loads in one partition Query. Household-plus-children writes, including capture apply, are a single `TransactWriteItems` against one resource. One table also means one IAM resource ARN, one backup config and one KMS grant. The accepted cost is that the table is harder to browse in the console, which is fine with about 10 entity types. Footprints stay out of DynamoDB: a 300 m GeoJSON can pass the 400 KB item limit, and it is read as a blob anyway.

| Entity | pk | sk | GSI1 | Notes |
|---|---|---|---|---|
| Identity → account | `IDENT#<cognito sub>` | `IDENT` | — | `account_id`, `provider`. Resolves every request |
| Account | `ACCT#<a>` | `PROFILE` | — | display name, created_at |
| Account's identities | `ACCT#<a>` | `IDENT#<sub>` | — | lists and unlinks sign-ins |
| Capture quota | `ACCT#<a>` | `QUOTA#capture#<yyyy-mm-dd>` | — | counter, `ttl` = +2 days |
| Map (place) | `MAP#<m>` | `META` | — | name, center, radius_m, footprint `{status, source, version, count}`, `version` |
| Membership | `MAP#<m>` | `MEMBER#<a>` | `ACCT#<a>` / `MAP#<m>` | `role` (`owner` in MVP; `editor` later for a spouse) |
| Household | `MAP#<m>` | `HH#<h>` | — | label, address, lat, lng, `building_id?`, `linked_account_id?` (null in MVP), last_talked_at, `version` |
| Person | `MAP#<m>` | `HH#<h>#PERSON#<p>` | — | name, role adult/kid, birth_year, birth_year_approx, birthday (mm-dd, optional), notes |
| Pet | `MAP#<m>` | `HH#<h>#PET#<x>` | — | name, species, notes |
| Note / log | `MAP#<m>` | `NOTE#<h>#<ulid>` | — | text, source (`voice`/`text`/`manual`), created_at |
| *Post-MVP* Card | `ACCT#<a>` | `CARD` | — | self-published fields, each tagged with a tier |
| *Post-MVP* Connection | `ACCT#<a>` | `CONN#<b>` | `ACCT#<b>` / `CONNOF#<a>` | tier that `a` granted `b` |
| *Post-MVP* Invite | `INVITE#<code>` | `INVITE` | — | `ttl` = +7 days |

Access patterns:
1. Resolve the caller: `GetItem IDENT#<sub>` → `account_id`, on every request. Not cached: an unlinked sub's access token stays valid for up to an hour, and this read is what cuts it off (B11).
2. List my maps: `Query GSI1 gsi1pk=ACCT#<a>, begins_with(gsi1sk, MAP#)`.
3. Authorize a map call: `GetItem MAP#<m> / MEMBER#<a>`, done in one common helper `require_member`.
4. Load a map (meta, households, people, pets): `Query pk=MAP#<m>`, `sk` between `HH#` and `HH#~`, plus `GetItem META`, paginated. Notes deliberately sort outside `HH#`, so they don't bloat map load.
5. Load one household: `Query begins_with(sk, HH#<h>)` + `Query begins_with(sk, NOTE#<h>#)`, newest first, limit 50.
6. Delete a household: both queries above → `BatchWriteItem` delete.
7. Delete or export an account: GSI1 → maps → per-map Query. Then the `ACCT#` partition, then the `IDENT#` items.
8. *Post-MVP* published fields for an entry: `GetItem ACCT#<linked>/CONN#<me>` (are we connected, and at what tier?) → `GetItem ACCT#<linked>/CARD`, filtered by tier at read time. Never copied into my partition.

Concurrency: the household and map items carry `version`, and writes use `ConditionExpression version = :expected`. Two clients editing the same household get a 409 and refetch.

Near-me needs no geo index. A map holds tens to a few hundred households, and the client already has them all in memory from pattern 4.

### API (HTTP API, Cognito JWT authorizer, base `https://api.curbside.xomware.com/v1`)
All responses use `{ data, error, meta }`. Every `/maps/{mapId}/...` route calls `require_member`.

| Method | Path | Lambda | MVP |
|---|---|---|---|
| GET | `/me` | `me_get` (404 `needs_onboarding` if the sub has no account) | yes |
| POST | `/me` | `me_create` (creates account + IDENT, idempotent per sub) | yes |
| POST | `/me/identities` | `me_link` (body: second provider's ID token) | yes |
| DELETE | `/me/identities/{sub}` | `me_unlink` (refuses to remove the last identity) | yes |
| GET | `/me/export` | `me_export` (JSON of everything the account owns) | yes |
| DELETE | `/me` | `me_delete` (requires fresh SIWA re-auth → Apple `/auth/revoke` with `custom:apple_at`, then hard delete + Cognito `AdminDeleteUser` for every linked sub) | yes |
| GET | `/maps` | `maps_list` | yes |
| POST | `/maps` | `maps_create` (async-invokes `footprints_fetch`) | yes |
| GET | `/maps/{mapId}` | `maps_get` (meta + households/people/pets) | yes |
| PATCH | `/maps/{mapId}` | `maps_update` | yes |
| DELETE | `/maps/{mapId}` | `maps_delete` | yes |
| GET | `/maps/{mapId}/footprints` | `footprints_get` (5-min presigned S3 URL + version; 202 while pending) | yes |
| POST | `/maps/{mapId}/footprints/refresh` | `footprints_refresh` | yes |
| POST | `/maps/{mapId}/households` | `households_create` | yes |
| GET | `/maps/{mapId}/households/{hhId}` | `households_get` (household + children + recent notes) | yes |
| PATCH | `/maps/{mapId}/households/{hhId}` | `households_update` | yes |
| DELETE | `/maps/{mapId}/households/{hhId}` | `households_delete` | yes |
| POST, PATCH, DELETE | `/maps/{mapId}/households/{hhId}/people[/{personId}]` | `people_*` | yes |
| POST, PATCH, DELETE | `/maps/{mapId}/households/{hhId}/pets[/{petId}]` | `pets_*` | yes |
| POST, DELETE | `/maps/{mapId}/households/{hhId}/notes[/{noteId}]` | `notes_*` | yes |
| POST | `/maps/{mapId}/capture/parse` | `capture_parse` | yes |
| POST | `/maps/{mapId}/capture/apply` | `capture_apply` | yes |
| GET, PUT | `/me/card` | `card_*` | post-MVP |
| POST | `/invites`, `/invites/{code}/accept` | `invites_*` | post-MVP |
| GET, PATCH, DELETE | `/connections[/{accountId}]` | `connections_*` | post-MVP |

Household response shape (fixed now, so the post-MVP card data is additive):
```json
{ "id": "...", "version": 3, "lat": 0, "lng": 0, "building_id": "osm:way/123",
  "private": { "label": "...", "address": "...", "people": [], "pets": [], "last_talked_at": null },
  "published": null }
```

### Auth flow
- **Pool**: `${app_name}-users`. No password sign-up: `supported_identity_providers = ["SignInWithApple", "Google"]` only, which removes the password-reset and email-sender surface. Prefix domain for MVP. The custom domain `auth.curbside.xomware.com` comes before public launch, because Apple's sheet shows the domain.
- **iOS**: `ASWebAuthenticationSession` → `/oauth2/authorize?identity_provider=SignInWithApple|Google&code_challenge=...` with a native-looking button per provider. This skips the hosted chooser. The redirect is `curbside://auth/callback` (custom scheme), followed by a PKCE code exchange at `/oauth2/token`. Tokens go in the Keychain, and refresh uses the refresh token (30 days). It is hand-rolled (~150 lines), with no Amplify.
- **Web** (follow-up): the same PKCE code flow, with its own public app client and callback `https://curbside.xomware.com/auth/callback` plus localhost.
- **API**: the HTTP API JWT authorizer validates the access token (issuer + audience = both client IDs). The handler resolves `sub` → `account_id` via `IDENT#`. Nothing downstream ever sees `sub` or email.
- **Account linking** (the private-relay problem): email is never used to match. Apple can hand out `xyz@privaterelay.appleid.com`, while Google returns the real address. Auto-linking on email also lets any IdP that asserts an unverified email take over an account.
  1. First sign-in with an unknown sub: `GET /me` → 404 `needs_onboarding`. The client shows a choice: "New here" (`POST /me`), or "I already use this app with another sign-in".
  2. Link path: the user signs in with the other provider first. Its access token authorizes the request. The client then runs a second PKCE flow with the new provider and sends that ID token to `POST /me/identities`. The server verifies the ID token against the pool JWKS (issuer, audience, `token_use=id`, `auth_time` within 10 min), then writes `IDENT#<sub2>` + `ACCT#<a>/IDENT#<sub2>` in one transaction, conditional on `IDENT#<sub2>` not existing.
  3. A sub that already owns an account with data cannot be linked. Merging accounts is out of scope. The user exports, deletes one account, then links.
  4. Settings → "Sign-in methods" lists the identities, can link another, and can unlink any but the last.
  - Using app-level identity mapping instead of Cognito `AdminLinkProviderForUser` keeps the two Cognito users separate. That avoids Cognito's rule that the linked IdP user must not already exist, which is exactly the case after someone has signed in on the other client.
- **App Store requirements**: Sign in with Apple is offered wherever Google is (4.8), in-app account deletion is required (5.1.1(v)), and Apple token revocation on deletion is an open question below.

### Footprint pipeline
1. `POST /maps {name, center, radius_m=300}` writes `META` with `footprint.status=pending` and async-invokes `footprints_fetch`. The response is 201.
2. `footprints_fetch` sends Overpass QL `[out:json][timeout:25];(way["building"](around:{r},{lat},{lng});relation["building"](around:{r},{lat},{lng}););out geom;` to `overpass-api.de`, with an identifying User-Agent, a 30s backoff on 429, and one retry against a second public instance. Overpass policy prefers self-hosted/paid instances for apps; accepted for MVP at one fetch per place (RESEARCH.md). The Lambda timeout is 60s.
3. It converts the result to a GeoJSON FeatureCollection. Feature `id` = `osm:way/<id>`, and properties are `{building, area_m2, accessory}`. `accessory=true` marks `garage|shed|carport|garages` tags or footprints under 30 m², which the client draws lighter and never prompts on. Coordinates are rounded to 6 decimals and the file is gzipped.
4. The result is written to `s3://${app_name}-footprints-<acct>/footprints/<mapId>/v<n>.geojson.gz`. The bucket is private, uses SSE-KMS and blocks all public access. Then `META.footprint` is updated to `{status: ready|empty|failed, source: "osm", version: n, count}`.
5. `GET /footprints` returns a presigned URL and the version. The client caches the file on disk by `(mapId, version)` and only re-downloads when the version changes. A refresh bumps `v<n>`.
6. With `empty` or `failed`, or for any house without a polygon, the client falls back to a long-press pin drop + `MKReverseGeocodingRequest` (`CLGeocoder` is deprecated in iOS 26). Households never require a `building_id`, because a footprint is a tap target, not a household (BRAINSTORM premise).
7. **Overture (conditional)**: if spike 0a shows OSM coverage is poor on Dom's street, PR B8b adds an Overture source. That is a DuckDB layer querying the buildings GeoParquet by bbox, with the same output contract and `source: "overture"`, and IDs from GERS. Time it in the spike before committing.
8. **Attribution**: show "© OpenStreetMap contributors" (ODbL) on the map. The served footprint GeoJSON is a Derivative Database; the public `curbside-backend` repo (`footprints_fetch`) satisfies ODbL §4.6. Household data keyed to building IDs is a Collective Database, outside share-alike (RESEARCH.md).

What the third party sees: Overpass gets a lat/lng + radius from an AWS IP, and no user identity.

### Haiku parsing contract
- **Endpoint**: `POST /maps/{mapId}/capture/parse`
- **Request**: `{ transcript: string (≤ 2000 chars), source: "voice"|"text", focus_household_id?: string, nearby_household_ids: string[] (≤ 8) }`. The transcript comes from on-device `SFSpeechRecognizer` (with `requiresOnDeviceRecognition = true` where supported). Web is text only.
- **Model input**: the system prompt (no prompt caching: below the minimum cacheable length, and negligible savings at Haiku 5.5 prices) plus a compact roster built server-side from the IDs above only: `[{hh_id, label, address_short, people:[{id,name,role,birth_year}], pets:[{id,name,species}]}]`, then the transcript. The rest of the map is never sent.
- **Output**: forced tool use with a JSON schema, validated by pydantic. Invalid output → 502 `parse_failed`, with no retry loop.
```json
{ "operations": [
    { "op": "create_household|update_household|add_person|update_person|add_pet|update_pet|add_note|touch_last_talked",
      "household_ref": "hh_id or new:1",
      "target_id": "person/pet id, update ops only",
      "fields": { "...": "only fields named in the transcript" },
      "evidence": "verbatim span from the transcript",
      "confidence": 0.0 } ],
  "unresolved": ["things mentioned that could not be placed"] }
```
- **Rules in the prompt**: never invent values. "About 6" → `birth_year` = current year − 6 with `birth_year_approx: true`. No last names unless spoken. Gossip and sensitive categories (health, religion, politics) are kept only as verbatim note text, never as structured fields.
- **Server checks**: drop ops whose IDs are not in the roster sent, and drop fields not in the entity schema. Return `{operations, unresolved, base_versions}`. Nothing is persisted, and the transcript is not stored or logged.
- **Apply**: the client shows the diff. The user unchecks or edits ops, then `POST /capture/apply {operations, base_versions}` → one `TransactWriteItems` with version conditions. Conflict → 409, and the client re-parses. The note text is saved only if the user keeps the `add_note` op.
- **Model and cost**: `claude-haiku-5-5`. About 1,500 input + ~400 output tokens. At $0.10/$0.50 per MTok that is about $0.0005 per dump, or about $0.05/month at 100 dumps (RESEARCH.md). The cap is 50 parses per account per day via the `QUOTA#` counter → 429. The API key goes Infisical → `TF_VAR_anthropic_api_key` → SSM SecureString `/curbside/anthropic/api-key`, billed against Dom's monthly API credits.
- **Eval**: 15 golden synthetic transcripts with expected ops, in `tests/fixtures/capture/`. A script runs them against the live model before merging any prompt change. Unit tests use recorded responses.

### Privacy: who can read what
| Reader | Can read | Cannot read |
|---|---|---|
| Me | Everything in maps I belong to, plus my card | Other users' maps |
| Co-members of a map (post-MVP spouse) | Everything in that map, private notes included. The map is the sharing unit | My other maps, my card tiers |
| A connected user B (post-MVP) | The fields of my card in the tier I granted B, live. Revoking hides them on the next read | Anything I wrote about B, whether I have an entry for B, my other connections |
| The server (Lambdas) | Plaintext of everything | — |
| Dom as operator | Technically everything, via AWS console/KMS | — |
| Anthropic | Transcript + roster subset for one parse | The rest of the map, identity |
| Overpass / Apple | Place lat/lng (Overpass), geocode lookups and map tiles (Apple) | Notes, names |

**Why the server reads plaintext**: Haiku parsing, the web client, export, and the card read path all need it. E2E would kill all four (BRAINSTORM premise). Mitigations, all in MVP:
- Customer-managed KMS key on the table and bucket, so access is auditable in CloudTrail and revocable.
- Per-Lambda IAM: footprints Lambdas get no table read beyond `MAP#*/META`, and `capture_parse` gets read-only access.
- The logger never logs request or response bodies, only IDs, status and latency. Enforced in `lambdas/common/logger.py` and covered by a test.
- No analytics SDK on either client.
- No enrichment of any kind (App Store 5.1.2).
- Face ID / passcode app lock on iOS.
- Export and hard delete from day one.
- The privacy policy says plainly that the operator can technically read the data and doesn't.

## Affected Files / Components
| File / Component | Change | Why |
|-----------------|--------|-----|
| `curbside-infrastructure/terraform/{main,providers,locals,kms}.tf` | new | state, tags (`environment`, `project`, `owner`), CMK |
| `.../terraform/dynamodb.tf`, `s3_footprints.tf` | new | table + GSI1 + PITR; private footprint bucket |
| `.../terraform/cognito.tf`, `cognito_idps.tf`, `ssm.tf` | new | pool, Apple + Google IdPs, iOS client (web client later) |
| `.../terraform/api.tf`, `acm_api.tf`, `lambda_layers.tf`, `lambdas_<group>.tf` | new | HTTP API, JWT authorizer, `api.` domain, Lambdas from stub zips |
| `.../terraform/web_hosting.tf` | new (web phase) | S3 + CloudFront + record for `curbside.xomware.com` |
| `.../.github/workflows/terraform.yml` | new | calls `Xomware/github-actions/.github/workflows/terraform.yml@v1` with Infisical |
| `curbside-backend/lambdas/common/*` | new | errors, responses, logger, dynamo, auth (`get_account_id`, `require_member`), models |
| `curbside-backend/lambdas/<name>/handler.py` | new, one per endpoint | house handler shape |
| `curbside-backend/.github/workflows/deploy.yml` | new | `deploy-lambda-python.yml@v1` |
| `curbside-ios/Curbside/{App,Services,Models,ViewModels,Views,Theme}` | new | SwiftUI app |
| `curbside-frontend/` | new (web phase) | Angular app |
| `docs/features/curbside/RESEARCH.md` (this repo) | new | spike 0a results |
| `docs/features/curbside/mockups/` (this repo) | new | step 0c |

## Implementation Steps

### Phase 0: gates before code (no PRs)
- [x] **0a. Verification spike** (desk research done 2026-11-16; OSM coverage on Dom's street still pending, RESEARCH.md §9) → `RESEARCH.md`:
  - Run the Overpass query above for Dom's actual street and two other neighborhood types (rowhouse, apartment). Record building counts and screenshot the polygons. Decide whether B8b (Overture) is MVP or not.
  - If Overture is in play, time a bbox DuckDB query over the public GeoParquet from a Lambda-sized box.
  - Confirm current prices and limits: MapKit JS daily free quota (recalled: 250k map views / 25k service calls per day), Cognito tier pricing for social federation (recalled: Lite free to 10k MAU), Haiku 4.5 $/MTok, and Overpass usage policy.
  - Confirm Anthropic API retention for inputs and whether zero-retention is available on Dom's account.
  - Confirm ODbL Produced Work treatment and attribution text.
- [x] **0b. Name**: Curbside, bundle ID `com.Xomware.Curbside` (decided 2026-11-16).
- [x] **0c. Themed mockups** — Curbside direction chosen 2026-11-16 (`mockups/curbside.html`): house-number plaques, street signs, yard-sign status, stop-sign record button, night-street dark mode., before any UI code. Three directions, survey/plat-map leading. Each shows four screens: map with footprints, household card, near-me sheet, capture diff confirm. Static HTML/PNG in `docs/features/curbside/mockups/`. SVG and text only, no emoji glyphs. Dom picks one, and S3 implements it.
  1. **Survey / plat map (lead)**: drafting-paper blue, lot lines and lot numbers on footprints, surveyor stamps for state ("MET", "NEW MOVE-IN", "LENT MOWER"), monospace legend block, north arrow.
  2. **Block party**: string-light accents, folding-table card layouts, warm evening palette.
  3. **Curbside / field notebook**: graph-paper notebook, hand-annotated callouts, rubber-band card stack.

### Phase 1: PR decomposition
Prefixes: I = infrastructure, B = backend, S = iOS, W = web. Line counts are hand-written logic only, and tests don't count against the budget. Every infra PR applies on merge via CI. Every Lambda an infra PR adds is a stub until its B PR deploys, so verify against the live endpoint after each B merge (stub-zip trap).

**iOS MVP**: I1–I8, B1–B12 (B8b only if 0a says so), S1–S10.

| # | PR | Repo | ~Lines | Depends on |
|---|---|---|---|---|
| I1 | Scaffold: backend/state key, providers, `locals.standard_tags`, CMK + alias, `terraform.yml` calling the shared workflow with Infisical | infra | 70 | — |
| I2 | DynamoDB table (pk/sk, GSI1, PITR, TTL, CMK) + private footprints bucket (SSE-KMS, public access blocked) | infra | 70 | I1 |
| I3 | Cognito pool (no password), SignInWithApple + Google IdPs (Google as `provider_type = "Google"`, map Apple `access_token` → `custom:apple_at` and verify it populates), iOS public client (PKCE, `curbside://auth/callback`), prefix domain, SSM outputs. Manual prereq: Apple Services ID + key, Google OAuth client → Infisical | infra | 160 | I1 |
| I4 | HTTP API + JWT authorizer + `api.curbside.xomware.com` (ACM + record) + common layer + Lambda `for_each` module pattern with per-Lambda roles, wiring `me_get`/`me_create` | infra | 190 | I2, I3 |
| B1 | Scaffold: `lambdas/common` (errors, responses, body-free logger + test, dynamo, `get_account_id`, `require_member`), pytest + moto, deploy workflow | backend | 180 | — |
| B2 | `me_get` + `me_create` (IDENT + ACCT transaction, idempotent) | backend | 100 | B1, I4 |
| I5 | Lambdas + routes: `maps_*`, `households_*`, `people_*`, `pets_*`, `notes_*` | infra | 80 | I4 |
| B3 | `maps_*` CRUD + membership items + pydantic models | backend | 170 | B2, I5 |
| B4 | `households_*` with the `private`/`published` response shape and version conditions | backend | 180 | B3 |
| B5 | `people_*` + `pets_*` child items | backend | 150 | B4 |
| B6 | `notes_*` + `last_talked_at` touch | backend | 80 | B4 |
| I6 | Lambdas `footprints_fetch/get/refresh`, async-invoke permission from `maps_create`, scoped S3 + `META` access | infra | 70 | I5 |
| B7 | `footprints_fetch`: Overpass (identifying UA, 429 backoff) → GeoJSON → S3 → `META` status (recorded-fixture tests) | backend | 180 | B3, I6 |
| B8 | `footprints_get` (presigned, 202 pending) + `footprints_refresh`; `maps_create` async-invokes fetch | backend | 70 | B7 |
| B8b | *Conditional*: Overture source behind the same contract + DuckDB layer (layer in infra) | backend+infra | 150 | B7, 0a |
| I7 | Lambdas `capture_parse/apply`, Anthropic key Infisical → SSM SecureString | infra | 50 | I5 |
| B9 | `capture_parse`: roster build, prompt, tool-use schema, validation, quota counter, eval script + golden fixtures | backend | 200 | B5, B6, I7 |
| B10 | `capture_apply`: ops → one `TransactWriteItems` with version checks | backend | 150 | B9 |
| I8 | Lambdas `me_link/unlink/export/delete` + `cognito-idp:AdminDeleteUser` scoped to the pool | infra | 50 | I4 |
| B11 | `me_link` (verify the second ID token via JWKS) + `me_unlink` | backend | 150 | B2, I8 |
| B12 | `me_export` + `me_delete` (SIWA `/auth/revoke`, all maps owned, partitions, Cognito users) | backend | 160 | B5, B6, I8 |
| S1 | Scaffold: XcodeGen project, app shell, test target, CI build/test | ios | 60 | — |
| S2 | `Config`, `APIClient` decoding `{data,error,meta}`, Keychain store (S2a), then auth: ASWebAuthenticationSession PKCE (Apple, Google), refresh, sign-out, onboarding (new vs. "I use another sign-in" → link) (S2b). Two stacked PRs, see `ios-foundation-auth/PLAN.md` | ios | 350 | S1, I3, B2, B11 |
| S3 | Theme foundation from the chosen mockup: palette, type, stamp + card components | ios | 150 | S1, 0c |
| S4 | Place setup (search / current location / radius) + muted MapKit base (`.muted`, POIs off) + household pins | ios | 200 | S3, B3 |
| S5 | Footprint overlays: download + disk cache by version, `MKPolygon` styling, tap hit-test, accessory styling, OSM attribution, pin fallback via long-press + `MKReverseGeocodingRequest` | ios | 200 | S4, B8 |
| S6 | Household card: view + create (address prefill), adults/kids/pets display, `published` section placeholder hidden when null | ios | 180 | S5, B4, B5 |
| S7 | Household editing: people, pets, notes, last-talked | ios | 180 | S6, B6 |
| S8 | Near-me sheet: CoreLocation + heading, nearest 5, left/right side of street | ios | 150 | S6 |
| S9 | Capture: on-device speech + text input → parse → diff-confirm UI → apply | ios | 200 | S7, B10 |
| S10 | Face ID lock, settings (sign-in methods, export share sheet, delete account, attribution, privacy policy link), App Store privacy label + review notes, TestFlight workflow | ios | 180 | S2, B11, B12, 0b |

**Web follow-ups**: no backend changes, infra only for hosting and the client.

| # | PR | Repo | ~Lines | Depends on |
|---|---|---|---|---|
| I9 | Web hosting (S3 + CloudFront + `curbside.xomware.com`), web Cognito app client + callback URLs, add the client ID to the authorizer audience, MapKit JS origin-restricted token in Infisical → build env | infra | 120 | iOS MVP |
| W1 | Angular scaffold + PKCE auth + onboarding/link + deploy via `deploy-frontend-s3.yml@v1` | frontend | 200 | I9 |
| W2 | Map view: MapKit JS, footprint polygons, theme from the chosen mockup | frontend | 200 | W1 |
| W3 | Household card view + edit (people, pets, notes) | frontend | 200 | W2 |
| W4 | Text capture → diff confirm → apply | frontend | 150 | W3 |
| W5 | Export download + account settings (sign-in methods, delete) | frontend | 100 | W1 |

**Post-MVP** (each gets its own `/plan`): profile cards + tiers + QR/invite connect; spouse shared map (`editor` role + invite); photos; ICS feed; lend/borrow; multiple places UI; friends map; holiday export.

## Out of Scope
- Profile cards, connections, invites, contact matching (model and response shape only)
- Shared household map UI, ICS feed, lend/borrow, multiple-places UI, friends map, holiday export
- Photos of any kind
- Account merging (two accounts that both hold data)
- Password sign-up, Android, widgets, Watch
- Any data enrichment (property records, social lookups)
- Overture, unless 0a pulls it in

## Risks / Tradeoffs
- **Overpass reliability**: the public instance can rate-limit or queue. It runs once per place, async, with one fallback instance and a manual refresh. Pins always work.
- **Footprint ≠ household**: duplexes, rowhouses and apartments share polygons. Households own lat/lng, and `building_id` is optional. Several households per polygon show as a list on tap.
- **Web-flow Sign in with Apple on iOS**: it is acceptable to App Review but less polished than the native sheet. Native SIWA would need a Cognito custom-auth challenge flow. Accepted for MVP; revisit if review pushes back.
- **App-level identity linking**: it adds one GetItem per request and a custom link flow. In exchange there is no email matching and no Cognito link-ordering trap. Accepted.
- **Server-readable data about non-consenting third parties**: accepted (see Privacy). Mitigated by CMK, scoped IAM, body-free logs, no analytics, Face ID and export/delete.
- **Lambda-per-endpoint count (~30)**: this matches house precedent, and `for_each` keeps the Terraform small. Cold starts are fine at personal scale.
- **Haiku hallucinating facts about real people**: nothing persists without per-op user confirmation, `evidence` spans are shown in the diff, and the eval set gates prompt changes.

## Open Questions
- [ ] Spike 0a results: OSM coverage on Dom's street (decides B8b), MapKit JS quota, Cognito federation pricing, Haiku price, Overpass policy, ODbL Produced Work.
- [ ] Anthropic retention for transcripts naming third parties. Is zero-retention available on Dom's account, and what does the privacy policy say?
- [x] Apple's account-deletion rule expects SIWA token revocation via Apple's REST API. Cognito holds the Apple tokens, not us. Is `AdminDeleteUser` enough for review, or do we need our own Apple client secret to call `/auth/revoke`? **Answer:** not enough. Map the Apple `access_token` to `custom:apple_at`, re-auth on delete, then call `/auth/revoke` with our own client secret, falling back to TN3194 manual revoke (RESEARCH.md §6).
- [ ] Custom auth domain (`auth.curbside.xomware.com`) before TestFlight, or before public launch only?
- [x] MapKit JS token: a static origin-restricted long-lived token (no backend change, current plan) vs. a token-minting endpoint (rotation, but a backend change)? **Answer:** static. Apple supports a domain-restricted, no-expiration dashboard token that can be revoked and reissued; add a minting endpoint only if abuse shows up (RESEARCH.md §1).
- [x] Final app name and bundle ID: Curbside, `com.Xomware.Curbside`.
- [ ] Default radius: is 300 m right, and should a place be allowed to grow later (a re-fetch at a larger radius)?
- [ ] Does Dom want spouse sharing soon enough that the `editor` role and invite should move ahead of profile cards post-MVP?

## Skills / Agents to Use
- **orchestrate**: this spans four repos and about 35 PRs, effectively an epic. Use it to split into per-repo goal sets (infra, backend, iOS MVP, web) rather than one `/goals` run.
- **research**: spike 0a, written to `RESEARCH.md`.
- **terraform** / **infra-standards**: I1–I9. Write the `.tf` as text and let CI plan. Never run locally.
- **lambda-handler** / **backend-standards** / **database**: B1–B12, including handler shape, single-table access helpers and pydantic models.
- **anthropic-api**: B9 tool-use schema, eval script.
- **ios-standards**: S1–S10.
- **angular-component** / **frontend-standards**: W1–W5.
- **testing**: moto-backed handler tests and capture golden fixtures.
- **github-batch-issues**: file one issue per PR row once Status is Ready.
- **review**: every PR. UI PRs (S*, W*) need "verified in app" per `verification.md`.
