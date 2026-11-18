# Plan: Curbside — Infra Web Hosting

**Epic**: curbside
**Sub-feature ID**: INF-3
**Repo**: `curbside-infrastructure`
**Track**: Web follow-up
**Status**: Ready
**Created**: 2026-11-16
**Depends on**: INF-1 (pool, IdPs, API). iOS MVP is a sequencing preference, not a technical dependency.

## Summary
Hosts the Angular desk companion at `curbside.xomware.com` and adds a web Cognito app client plus the frontend's deploy role. No backend changes.

## Approach
- `domgiordano/web-hosting` module at `v1.8.0`, the same one pickem and xomforms use: private S3 bucket behind OAC, CloudFront with the module's security-headers policy, ACM cert, alias record in the shared `xomware.com` zone, shared CloudFront WAF from `/xomware/shared/cloudfront-waf-acl-arn`.
- SPA fallback is the module default: 403 → `/index.html` with 200.
- New public client `curbside-web`: code flow, Apple + Google, `openid` scope, same token lifetimes and `custom:apple_at` write as `curbside-ios`. Callbacks `https://curbside.xomware.com/auth/callback` and `http://localhost:4200/auth/callback`; logout URLs are the two bare origins.
- The web client ID joins the JWT authorizer audience and is published at `/curbside/cognito/web-client-id`.
- `curbside-github-actions-deploy-frontend` role, assumable only from `Xomware/curbside-frontend` on `main`: sync the web bucket, invalidate the distribution, read the four build-config SSM parameters.

## Affected Files / Components
| File / Component | Change | Why |
|-----------------|--------|-----|
| `terraform/web_hosting.tf` | new | module call + shared WAF lookup |
| `terraform/oidc_deploy_frontend.tf` | new | frontend deploy role + outputs |
| `terraform/cognito.tf` | edit | web client |
| `terraform/api.tf` | edit | authorizer audience |
| `terraform/ssm.tf` | edit | web client ID parameter |

## Implementation Steps
- [ ] I9 — Web hosting + web Cognito client + authorizer audience + frontend deploy role (~130 lines, one PR)

## Plan corrections
- **SPA fallback is 403 only.** The module maps 403, not 404. That is enough: CloudFront's OAC has `s3:GetObject` without `s3:ListBucket`, so S3 answers a missing key with 403. Forking the module to add 404 buys nothing.
- **No new cert, bucket or distribution resources.** The house pattern is the shared module, not hand-written resources. The cert lands in the provider's region, which is `us-east-1` here; moving `var.aws_region` would break CloudFront.
- **CORS needed no change.** `var.cors_allow_origins` already defaults to the web origin and `localhost:4200`.
- **MapKit JS token needs no Terraform.** The epic resolved it as a static, domain-restricted token. `deploy-frontend-s3.yml` exports every secret in Infisical `/curbside` into the build env, so storing it there as `MAPKIT_JS_TOKEN` is the whole plumbing. Out of scope for this PR until Dom creates the Maps ID and key.
- **Build config comes from SSM, not Infisical.** The shared workflows keep Terraform-computed values in SSM and read them with the `ssm` input. The only new Infisical secret is `AWS_FRONTEND_ROLE_ARN`.
- **A separate deploy role**, not grants on the backend role as pickem does. Each repo gets its own role, so the frontend repo can't touch Lambdas.
- **PKCE is the client's job.** `aws_cognito_user_pool_client` has no argument to require PKCE, and Cognito only checks a verifier when the authorize request carried a challenge. The Angular app must always send `code_challenge` (S256).

## Out of Scope
- Angular code (WEB-1 onward)
- MapKit JS token: Infisical secret only, once Dom creates it
- Token-minting endpoint for MapKit JS (rejected in the epic; revisit only on abuse)

## Risks / Tradeoffs
- Geo restriction is the module default, US + CA. Matches the other Xomware sites; a traveller abroad gets a 403.
- The frontend build env holds every `/curbside` Infisical secret, including the Apple, Google and Anthropic keys. A build that bakes `process.env` wholesale into the bundle would leak them. The Angular build must read named variables only.
- Bucket name is the hostname `curbside.xomware.com`. S3 names are global; a squatter would fail the apply.

## Open Questions
- [x] MapKit JS: static origin-restricted token vs. token-minting endpoint? **Static** (epic, RESEARCH.md §1).
- [ ] MapKit JS: Dom creates the Maps ID + MapKit JS key restricted to `curbside.xomware.com` (and `localhost` for dev?), then stores it as `MAPKIT_JS_TOKEN` in Infisical `/curbside`.

## Skills / Agents to Use
- terraform, infra-standards
