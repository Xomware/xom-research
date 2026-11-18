# Plan: Curbside — Infra Foundation

**Epic**: curbside
**Sub-feature ID**: INF-1
**Repo**: `Xomware/curbside-infrastructure` (public, base `main`)
**Track**: iOS MVP
**Status**: Ready
**Created**: 2026-11-16
**Last updated**: 2026-11-16
**Depends on**: bootstrap prereqs below (all manual, Dom)

## Summary
Stands up the Terraform repo, state, CMK, the single DynamoDB table, the private footprints bucket, the Cognito pool with Apple + Google IdPs, and the HTTP API with its JWT authorizer, `api.curbside.xomware.com`, common layer and the Lambda `for_each` pattern wired for `me_get`/`me_create`. Everything later in the epic hangs off this.

## Approach
- Layout and style follow `uncle-mikes-pickem/infrastructure/terraform` (newest stack): flat `terraform/`, file per concern, tags via provider `default_tags`, no per-resource `merge()`.
- State: S3 `xomware-terraform-state`, key `curbside/terraform.tfstate`, lock table `xomware-terraform-locks`.
- CI: `.github/workflows/terraform.yml` is a shim over `Xomware/github-actions/.github/workflows/terraform.yml@v1` with `app: curbside`. That loads Infisical `code` → `/curbside`, exports every secret as `TF_VAR_<lowercase>`, plans on PRs (any base, so stacked PRs plan), applies the saved plan on push to `main`.
- Cognito follows `today-in-sports-infrastructure/terraform/cognito.tf` (deletion protection, `prevent_destroy`, Google pinned endpoint fields), minus passwords and email.
- All names from `var.app_name = "curbside"`. Never run Terraform locally.

## Bootstrap prereqs (manual, before any PR can plan)
1. **Plan/apply roles** in `xomware-infrastructure`, copying `oidc_pickem_terraform.tf`: `curbside-github-actions-terraform-{plan,apply}`. Subjects `repo:Xomware/curbside-infrastructure` and `repo:Xomware@263047999/curbside-infrastructure@1460745853`; apply trusted from `ref:refs/heads/main` only.
2. **Infisical** `code` / `prod` / `/curbside`: `AWS_TERRAFORM_PLAN_ROLE_ARN`, `AWS_TERRAFORM_APPLY_ROLE_ARN`.
3. **Before I3**: Apple Services ID + SIWA key, Google OAuth web client, stored as `APPLE_SERVICES_ID`, `APPLE_TEAM_ID`, `APPLE_KEY_ID`, `APPLE_PRIVATE_KEY`, `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`. Both providers' redirect/return URL is `https://xomware-curbside.auth.us-east-1.amazoncognito.com/oauth2/idpresponse`; Apple also needs `xomware-curbside.auth.us-east-1.amazoncognito.com` under the Services ID's domains.

## PRs

### I1 — Scaffold (~60 lines, no deps)
- `terraform/main.tf`: `required_version >= 1.5.0`, AWS `~> 5.0`, S3 backend above.
- `terraform/providers.tf`: provider with `default_tags = local.standard_tags`; `aws_caller_identity`.
- `terraform/variables.tf`: `aws_region`, `app_name`, `environment`, `owner`.
- `terraform/locals.tf`: `standard_tags = {source, project, environment, owner}`, `account_id`.
- `terraform/kms.tf`: one CMK `alias/curbside` (rotation on, 30-day window, `prevent_destroy`). Shared by table and bucket so there is one key to audit and revoke.
- `.github/workflows/terraform.yml`, `.gitignore`.

Acceptance:
- CI plan on the PR shows exactly 2 adds (key, alias), 0 change, 0 destroy.
- After merge the apply run succeeds; `aws kms describe-key --key-id alias/curbside` shows `KeyRotationEnabled` and tags `project=curbside`, `environment`, `owner`.

### I2 — Table + footprints bucket (~70 lines, needs I1)
- `terraform/dynamodb.tf`: table `curbside`, on-demand, `pk`/`sk` (S), GSI `gsi1` on `gsi1pk`/`gsi1sk` (ALL), TTL `ttl`, PITR on, SSE with the I1 CMK, `deletion_protection_enabled`, `prevent_destroy`.
- `terraform/s3_footprints.tf`: bucket `curbside-footprints-<account_id>`, all public access blocked, SSE-KMS with the CMK and bucket key on. No versioning (objects are a regenerable cache keyed by `v<n>`).

Acceptance:
- Plan: 4 adds, 0 change, 0 destroy.
- After apply: `describe-table` shows PITR, TTL on `ttl`, `SSEType=KMS` with the CMK, GSI `gsi1`; `get-public-access-block` all true; `get-bucket-encryption` shows `aws:kms` + the CMK.

### I3 — Cognito (~150 lines, needs I1; Apple/Google secrets in Infisical)
- `terraform/cognito.tf`:
  - Pool `curbside-users`, `user_pool_tier = "LITE"` (social federation is on all tiers, 10k MAU free), deletion protection + `prevent_destroy`, case-insensitive usernames, `allow_admin_create_user_only = true` (kills the public `SignUp` API; federated sign-in still creates users), account recovery `admin_only`.
  - Custom attribute `apple_at` (string, mutable, max 2048). Cognito never lets you remove or change a custom attribute, so this is permanent.
  - Prefix domain `xomware-curbside`. A bare `curbside` prefix is likely taken globally, and a failed apply would come after the Apple and Google consoles were already pointed at it.
- `terraform/cognito_idps.tf`:
  - `SignInWithApple`: `client_id` = Services ID, `team_id`, `key_id`, `private_key`, scope `name`; mapping `username = sub`, `custom:apple_at = access_token`.
  - `Google` with `provider_type = "Google"` (keeps it in the social 10k MAU bucket), scope `openid`, pinned endpoint fields from the precedent; mapping `username = sub`.
  - No email or name mapped. The app never matches on email, and the display name is typed at onboarding.
- iOS client `curbside-ios`: no secret, `code` flow, scope `openid`, callback `curbside://auth/callback`, IdPs `SignInWithApple` + `Google` only (no `COGNITO`), `ALLOW_REFRESH_TOKEN_AUTH` only, access/ID 60 min, refresh 30 days, `read_attributes` left unset, which Cognito defaults to standard attributes only, so `custom:apple_at` never appears in an ID token, `write_attributes = ["custom:apple_at"]` so the IdP mapping can write it.
- `terraform/ssm.tf`: String params `/curbside/cognito/user-pool-id`, `/curbside/cognito/ios-client-id`, `/curbside/cognito/domain`.
- Vars `apple_services_id`, `apple_team_id`, `apple_key_id`, `apple_private_key` (sensitive), `google_client_id`, `google_client_secret` (sensitive). Required, no defaults: a missing secret fails the plan instead of applying a broken IdP.

Acceptance:
- Plan fails until the six Infisical secrets exist; with them, 8 adds (pool, domain, 2 IdPs, client, 3 params), 0 change, 0 destroy.
- After apply, a second plan on `main` shows no changes. A diff on the Apple IdP's `provider_details` means Cognito fills in fields we didn't pin; pin them the way the Google ones are.
- Manual check: open `https://xomware-curbside.auth.us-east-1.amazoncognito.com/oauth2/authorize?identity_provider=SignInWithApple&response_type=code&client_id=<ios client>&redirect_uri=curbside://auth/callback&scope=openid` in Safari and sign in. Then `aws cognito-idp admin-get-user` on the new `signinwithapple_…` user shows `custom:apple_at`. If it's absent, record that in RESEARCH.md §6 and B12 falls back to TN3194 manual revoke.
- Same URL with `identity_provider=Google` creates a `google_…` user.

### I4 — HTTP API + first Lambdas (~190 lines, needs I2 and I3 applied)
- `terraform/api.tf`: HTTP API `curbside`, JWT authorizer (issuer `https://cognito-idp.us-east-1.amazonaws.com/<pool>`, audience = iOS client ID), stage `v1` with auto-deploy and access logs (no bodies).
- `terraform/acm_api.tf`: ACM cert + DNS validation for `api.curbside.xomware.com` in the shared `xomware.com` zone, API domain + mapping + alias A record.
- `terraform/lambda_layers.tf`: `curbside-common` layer from a stub zip, ignore `filename`/`source_code_hash` changes (backend deploy owns code).
- `terraform/lambdas_me.tf`: `for_each` map `{me_get = GET /me, me_create = POST /me}` → function from `templates/lambda_stub.zip`, own role, log group (30 days), route + integration + invoke permission. Explicit timeout and memory. Per-Lambda policy: `dynamodb:GetItem/PutItem/TransactWriteItems` on the table only; KMS decrypt/encrypt on the CMK.
- SSM `/curbside/api/url`.

Acceptance:
- Plan shows only adds.
- After apply: `curl https://api.curbside.xomware.com/v1/me` without a token returns 401 from the authorizer; with a real access token it reaches the stub.
- Each Lambda role's policy names only the table ARN (+ `/index/*` only where a handler queries GSI1) and the CMK.

## Out of Scope
- Lambda code (BE-1 onward), remaining Lambdas/routes (I5–I8), web client and hosting (I9).
- Apple SIWA key in SSM for `/auth/revoke`: lands with I8, alongside `me_delete`.
- Custom auth domain `auth.curbside.xomware.com` (before public launch, not TestFlight).

## Risks / Tradeoffs
- Stateful names (pool, table, bucket, Cognito domain, custom attributes) are permanent in practice.
- Apple IdP endpoint fields are unpinned on first apply; there is no Apple precedent in the org. Watch the second plan.
- Cognito does not let a client require PKCE. The iOS client must always send `code_challenge`; the server won't enforce it.
- Stub-zip trap: `me_*` are stubs until B2 deploys. Verify the live endpoint after B2.
- Public repo: PR plan comments expose the account ID and ARNs (same as other Xomware stacks). Secrets stay redacted because their variables are `sensitive`.

## Open Questions
- [x] Custom auth domain timing: before public launch, not in INF-1.
- [x] Cognito federation pricing: social IdPs are in the 10k free MAU bucket on Lite (RESEARCH.md §3).
- [ ] Does Cognito populate `access_token` into `custom:apple_at` for `SignInWithApple`? Answered by I3's manual check.

## Skills / Agents to Use
- terraform, infra-standards
