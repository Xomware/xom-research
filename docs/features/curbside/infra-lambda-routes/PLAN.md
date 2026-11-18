# Plan: Curbside — Infra Lambda Routes

**Epic**: curbside
**Sub-feature ID**: INF-2
**Repo**: `curbside-infrastructure`
**Track**: iOS MVP
**Status**: Ready
**Created**: 2026-11-16
**Last updated**: 2026-11-17
**Depends on**: INF-1 (I4, applied)

## Summary
Adds every remaining MVP Lambda (stub zips) and route on the HTTP API, each with its own role scoped to what its handler does. Lands ahead of the backend PRs that fill the stubs, so each backend sub-feature only deploys code.

## Approach
- One `lambdas_<group>.tf` per group, each a `for_each` locals map into `modules/api_lambda`, as `lambdas_me.tf` does.
- Function names are `curbside-<folder with - for _>`; the backend deploy workflow relies on that.
- Per-Lambda env beyond `TABLE_NAME` and per-Lambda extra policy documents come from small locals maps keyed by Lambda name.
- Runtime secrets go Infisical `/curbside` → `TF_VAR_*` → `aws_ssm_parameter` SecureString (default `aws/ssm` key, so roles need only `ssm:GetParameter`). Nothing is hand-set in SSM.
- `dynamodb:TransactWriteItems` has no IAM action; each item is authorized as Put/Update/Delete/ConditionCheckItem.

### Module changes
- `timeout` input, default 10 (I8). `capture_parse`, `me_export` and `me_delete` take 29 s, just under the HTTP API's 30 s integration limit. `footprints_fetch` takes 120 s.
- `route_key` becomes optional (I6). Null means no integration, route or API permission, for `footprints_fetch`. `moved` blocks keep every existing route in place.
- Outputs `function_name`, `function_arn`, `role_name` (I6).

### IAM per Lambda
| Lambda | Table actions | Other |
|---|---|---|
| `maps_list` | GetItem, Query (+ `index/gsi1`), BatchGetItem | |
| `maps_create` | GetItem, PutItem | `lambda:InvokeFunction` on `footprints_fetch` (I6) |
| `maps_get` | GetItem, Query | |
| `maps_update` | GetItem, UpdateItem | |
| `maps_delete` | GetItem, Query, DeleteItem, BatchWriteItem | S3 list/delete under `footprints/` |
| `households_create` | GetItem, PutItem | |
| `households_get` | GetItem, Query | |
| `households_update` | GetItem, UpdateItem | |
| `households_delete` | GetItem, Query, DeleteItem, BatchWriteItem | |
| `people_create`, `pets_create` | GetItem, PutItem, ConditionCheckItem | |
| `people_update`, `pets_update` | GetItem, UpdateItem | |
| `people_delete`, `pets_delete`, `notes_delete` | GetItem, DeleteItem | |
| `notes_create` | GetItem, PutItem, UpdateItem (`last_talked_at`) | |
| `footprints_fetch` | GetItem, UpdateItem on `MAP#*`, attributes `pk sk center radius_m footprint` | S3 PutObject `footprints/*`, SQS failure destination |
| `footprints_get` | GetItem on `IDENT#*`; GetItem on `MAP#*`, attributes `pk sk role footprint` | S3 GetObject `footprints/*` (presign) |
| `footprints_refresh` | as `footprints_get`, plus UpdateItem | `lambda:InvokeFunction` on `footprints_fetch` |
| `capture_parse` | GetItem, Query; UpdateItem on `ACCT#*`, attributes `pk sk counter ttl` | `ssm:GetParameter` on the Anthropic key |
| `capture_apply` | GetItem, Query, PutItem, UpdateItem, ConditionCheckItem | |
| `me_link` | GetItem, PutItem | env `USER_POOL_ID`, `CLIENT_IDS` |
| `me_unlink` | GetItem, Query, DeleteItem | |
| `me_export` | GetItem, Query (+ `index/gsi1`) | |
| `me_delete` | GetItem, Query (+ `index/gsi1`), DeleteItem, BatchWriteItem | `cognito-idp:ListUsers/AdminGetUser/AdminDeleteUser` on the pool; `ssm:GetParameter` on the Apple key; S3 list/delete under `footprints/` |

All table grants also carry the CMK via `kms:ViaService` DynamoDB. S3 grants carry it via S3.

**Footprints "META only" (answers the open question).** IAM has no sort-key condition, so `MAP#*/META` can't be expressed exactly. `dynamodb:LeadingKeys` limits footprints Lambdas to `MAP#*` (and `IDENT#*` for caller resolution). `dynamodb:Attributes` plus `dynamodb:Select = SPECIFIC_ATTRIBUTES` limits what they see there to the listed names. A read of a household, person, pet or note item yields keys at most, never names or notes. Backend contract (B7/B8): every `MAP#*` read in these Lambdas passes a `ProjectionExpression` within the list, `require_member` included. A read without one is denied. Updates use `ReturnValues` NONE or UPDATED_*.

**Capture quota.** `capture_parse` writes only the `QUOTA#` counter: UpdateItem limited to `ACCT#*` and attributes `pk sk counter ttl`. B9 must use those names.

### Secrets
- `me_delete`: SSM SecureString `/curbside/apple/private-key` from `var.apple_private_key` (already in Infisical for the Apple IdP). Services ID, team ID and key ID go in env; they aren't secret.
- `capture_parse`: Infisical `/curbside` `ANTHROPIC_API_KEY` (copied from `/pickem`) → `var.anthropic_api_key` → SSM SecureString `/curbside/anthropic/api-key`.

### Async invoke
`footprints_fetch` runs with `maximum_retry_attempts = 0` (a retry re-queries Overpass, and the handler already retries once) and an SQS on-failure destination `curbside-footprints-fetch-failures` (SSE, 14 days). The payload is `{map_id}` only.

## Implementation Steps
Merge order: I8 → I5 → I6 → I7. A linear stack, every PR based on `main`; until its ancestors merge, a PR's diff and plan include theirs.

| # | PR | ~Lines | Expected plan (own) |
|---|---|---|---|
| I8 | `lambdas_account.tf`: `me_link/unlink/export/delete`; module `timeout`; Apple key SSM; shared `footprints_delete` policy | ~110 | +29 |
| I5 | `lambdas_maps.tf`: 17 Lambdas (`maps_*` 5, `households_*` 4, `people_*` 3, `pets_*` 3, `notes_*` 2) | ~150 | +119 |
| I6 | `lambdas_footprints.tf`: `footprints_fetch/get/refresh`, failure queue, `maps_create` invoke grant; module optional route + outputs | ~150 | +21, 6 moved (me_* only; later ones are created at `[0]`) |
| I7 | `lambdas_capture.tf`: `capture_parse/apply`; Anthropic var + SSM | ~70 | +15 |

Every plan: creates only, 0 change, 0 destroy (I6 adds `moved`, no-op).

## Out of Scope
- Handler code (BE-2 to BE-5)
- Overture DuckDB layer (conditional, BE-3)
- Web hosting, web client, web client ID in `CLIENT_IDS` (INF-3 / I9)

## Risks / Tradeoffs
- Stub-zip trap: every route here returns stub output until its B PR deploys. Verify live after each B merge.
- IAM is tight per handler. A B PR whose handler needs another action gets AccessDenied live, and needs a one-line infra PR. moto doesn't enforce IAM, so tests won't catch it.
- The footprints attribute scoping makes B7/B8 depend on projection discipline (see above).
- `aws_lambda_function_event_invoke_config` checks the role can send to SQS at create time. IAM propagation lag can fail the first apply. Rerunning the apply fixes it.

## Open Questions
- [x] Can the `META`-only restriction be expressed in IAM? Partially: partition via `LeadingKeys`, content via `dynamodb:Attributes`. Sort key can't be scoped.

## Skills / Agents to Use
- terraform, infra-standards
