# Plan: Curbside — Backend Foundation

**Epic**: curbside
**Sub-feature ID**: BE-1
**Repo**: `curbside-backend`
**Track**: iOS MVP
**Status**: Ready
**Created**: 2026-11-16
**Last updated**: 2026-11-16
**Depends on**: B1 none; B2 needs INF-1 (I4) and the backend deploy role (see Dependencies on infra)

## Summary
B1 scaffolds `lambdas/common`, pytest + moto, ruff, PR CI and the deploy workflow, with no handlers. B2 ships `me_get` + `me_create` so a Cognito sub resolves to an account. B2 is the first live handler and the first check against the stub-zip trap.

## Approach
Handler shape from `xomify-backend/lambdas/ratings_track/handler.py`: `lambdas/<name>/handler.py`, `@handle_errors(HANDLER)`, `success_response`, typed errors, all table access through `lambdas/common`. Deliberate departures from xomify:

- **Envelope**: every response is `{ "data", "error", "meta" }`. Errors are `error: {code, message, details?}` with `data: null`. xomify returns bare bodies and `{error: {...}}`.
- **No CORS module**: HTTP API `cors_configuration` (I4/I9) sets CORS headers, and API Gateway overrides integration CORS headers when it is configured. xomify's `cors.py` exists because it is a REST API.
- **Body-free logs**: `handle_errors` logs one line per request with `handler`, `route`, `status`, `error_code` and `latency_ms`. It never logs the event, body, headers, query or exception message (a message can carry user input). On a 500 it logs the exception type and the stack frames only. xomify's `log_error_context` logs a masked body, which is not acceptable here.
- **No `handler=`/`function=` kwargs on errors**: the decorator already logs the handler, and the traceback names the function.
- **Structured logging via Lambda's JSON log format**, not structlog. Stdlib `logging` with `extra={...}` comes out as JSON fields when the function has `logging_config { log_format = "JSON" }`. That needs no dependency, but **I4 must set it**.
- **Pydantic v2 at the boundary**: `parse_body(event, Model)`. A `pydantic.ValidationError` becomes 400 `invalid_request` with `details: [{loc, msg}]`.

Event format is HTTP API payload v2.0. The caller's sub is `requestContext.authorizer.jwt.claims.sub`. Nothing downstream sees `sub` beyond `auth.py`.

## Affected Files / Components

### B1
| File | Change | Why |
|---|---|---|
| `lambdas/__init__.py`, `lambdas/common/__init__.py` | new | package roots; the layer ships `python/lambdas/common` |
| `lambdas/common/errors.py` | new | `CurbsideError(message, code)` with `status`; `ValidationError` 400, `AuthError` 401, `NotFoundError` 404 |
| `lambdas/common/logger.py` | new | the `curbside` logger, level from `LOG_LEVEL` |
| `lambdas/common/responses.py` | new | `success_response`, `error_response`, `parse_body`, `handle_errors`; Decimal-safe JSON |
| `lambdas/common/dynamo.py` | new | cached `table()` from `TABLE_NAME`, `ident_key`, `member_key`, `get_item` (consistent read) |
| `lambdas/common/auth.py` | new | `get_sub`, `get_account_id` (`IDENT#<sub>` cached per warm container), `require_member` (`MAP#/MEMBER#` GetItem, 404 if absent) |
| `tests/conftest.py` | new | env, moto table with pk/sk + GSI1, HTTP API v2 event factory, cache resets |
| `tests/test_responses.py`, `tests/test_logging.py`, `tests/test_auth.py` | new | envelope + error mapping, body-free logs, auth helpers |
| `requirements.txt`, `requirements-dev.txt`, `pyproject.toml`, `.python-version`, `.gitignore` | new | pinned runtime deps, pytest/moto/ruff, ruff + pytest config, Python 3.12 |
| `.github/workflows/ci.yml` | new | PRs: `ruff check`, `ruff format --check`, `pytest` |
| `.github/workflows/deploy.yml` | new | push to main + manual: `Xomware/github-actions/.github/workflows/deploy-lambda-python.yml@v1`, `app: curbside` |
| `README.md` | edit | setup, test, lint commands |

### B2
| File | Change | Why |
|---|---|---|
| `lambdas/common/dynamo.py` | edit | `acct_key`, `acct_ident_key` |
| `lambdas/common/models.py` | new | `AccountCreate` (display name), account response model |
| `lambdas/me_get/handler.py` | new | `GET /me`: `get_account_id` → `ACCT#/PROFILE` → account + identities. 404 `needs_onboarding` from `get_account_id` |
| `lambdas/me_create/handler.py` | new | `POST /me`: one `TransactWriteItems` of `IDENT#<sub>` (condition `attribute_not_exists(pk)`), `ACCT#<a>/PROFILE`, `ACCT#<a>/IDENT#<sub>`. On a cancelled condition it re-reads `IDENT#<sub>` and returns the existing account with 200, so it is idempotent per sub |
| `tests/test_me_get.py`, `tests/test_me_create.py` | new | moto-backed |

## Acceptance criteria

### B1
- [ ] `pytest` passes locally in a repo-local `.venv` on Python 3.12, and in `ci.yml` on the PR.
- [ ] `ruff check` and `ruff format --check` are clean.
- [ ] Any handler wrapped in `@handle_errors` returns `{data, error, meta}`. `CurbsideError` subclasses map to their status and code. A pydantic failure maps to 400 `invalid_request` with `loc`s. Malformed JSON maps to 400. Any other exception maps to 500 `internal_error` with a fixed message and no stack trace.
- [ ] A test puts a sentinel string in the request body, the headers, a query param and an exception message. It asserts the sentinel appears in no log record on the success, 4xx and 500 paths.
- [ ] Missing JWT claims → 401 `unauthorized`. Unknown sub → 404 `needs_onboarding`. Known sub → its `account_id`, and a second call in the same container makes no DynamoDB read.
- [ ] `require_member` returns the member item or raises 404 `map_not_found`. It never reveals whether the map exists.
- [ ] DynamoDB `Decimal` serializes to `int` or `float`.
- [ ] `deploy.yml` calls the shared workflow with `app: curbside`, `test-command` installing `requirements-dev.txt`, and `contents: read` + `id-token: write`.
- [ ] Logic ≤ ~200 lines, tests excluded.

### B2
- [ ] `GET /me` for an unknown sub → 404 `needs_onboarding`. For a known sub → 200 with the account and its identities.
- [ ] `POST /me` creates IDENT + ACCT in one transaction. A second `POST /me` from the same sub returns the same `account_id` with 200 and writes nothing new.
- [ ] After the deploy, both live endpoints are invoked with a real Cognito token and return the envelope, not stub output.

## Dependencies on infra (not in INF-1's stub as written)
- **Backend deploy role.** The shared workflow assumes the role in Infisical `/curbside` → `AWS_BACKEND_ROLE_ARN`. Nothing in I1–I4 creates it. xomify keeps it in `xomify-infrastructure/terraform/oidc.tf` (`github_actions["backend"]`). Curbside needs the same: OIDC trust on `repo:Xomware/curbside-backend:*`, `lambda:PublishLayerVersion`/`ListLayerVersions` on `curbside-shared-packages`, and `lambda:UpdateFunctionCode`/`UpdateFunctionConfiguration`/`GetFunction*` on `curbside-*`. The ARN goes into Infisical. Add it to I1 or I4.
- **Names the code assumes**: function `curbside-<name with _ as ->`, layer `curbside-shared-packages`, env `TABLE_NAME`, Lambda `logging_config.log_format = "JSON"`, HTTP API payload format 2.0.

## Implementation Steps
- [ ] B1 — Scaffold above (~180 logic lines, no deps)
- [ ] B2 — `me_get` + `me_create` (~100, needs B1, I4, deploy role)

## Out of Scope
- Linking, export, delete (BE-5)
- Any map data (BE-2)
- ULID generation (lands with B2, the first writer)

## Risks / Tradeoffs
- Public repo: all fixtures are synthetic, with no real names.
- **The `IDENT#` cache outlives unlink and delete.** `get_account_id` caches sub → account per warm container. After `me_unlink` or `me_delete`, a container that already cached the sub keeps resolving it until it recycles, and the JWT authorizer keeps accepting that sub's access token until it expires (up to 1 h). **Resolved in B11** by removing the cache; token revocation can't help because the HTTP API authorizer validates access tokens locally. Original options were revoking the sub's Cognito tokens (`AdminUserGlobalSignOut` / `AdminDeleteUser`) and keeping the access-token TTL short, or by dropping the cache for write routes.
- **The first deploy run on main fails.** B1 changes `lambdas/common`, so merging it triggers a layer publish. That fails until the deploy role exists. With the role in place, a layer-only push with zero handler folders still fails in the shared workflow: an empty target list reaches `xargs`, which runs `deploy` once with no arguments. B2 adds handler folders, which removes the second case for Curbside. The shared workflow should still guard empty input (`xargs -r`).

## Open Questions
- [ ] Does I4 own the backend deploy role, or I1?

## Skills / Agents to Use
- lambda-handler, backend-standards, python, database, testing
