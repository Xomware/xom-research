# Plan: Curbside — Backend Footprints

**Epic**: curbside
**Sub-feature ID**: BE-3
**Repo**: `curbside-backend` (B8b also touches `curbside-infrastructure`)
**Track**: iOS MVP
**Status**: Ready
**Created**: 2026-11-16
**Last updated**: 2026-11-17
**Depends on**: BE-2 (B3, live), INF-2 (I6, applied), gate 0a (OSM coverage; decides B8b)

## Summary
Fetches building footprints from Overpass per place, stores them as versioned gzipped GeoJSON in S3, and serves them via presigned URL. Overture is added only if spike 0a says OSM coverage is poor.

## Approach
Footprint pipeline steps 1–7 in the epic, with the contract pinned below.

### B7 — `footprints_fetch` (async, no route)
- Payload `{map_id}`. Reads `META` projected to `pk, sk, center, radius_m, footprint` (the role's attribute list).
- Overpass QL from the epic, `[timeout:25]`, POSTed with `User-Agent: curbside/0.1 (+https://github.com/Xomware/curbside-backend)` and a 40 s client timeout.
- `overpass-api.de` first. On 429: sleep 30 s, then the second instance. On any other failure (5xx, timeout, connection, or a 200 whose `remark` reports a runtime error): the second instance at once. One retry total. The second instance is `overpass.private.coffee` (no rate limit per the wiki). 40 + 30 + 40 s stays under the 120 s Lambda timeout.
- GeoJSON FeatureCollection. Ways become Polygons; building relations become a Polygon (one outer ring plus inners) or a MultiPolygon of outer rings, with open member ways joined into rings. Feature `id` `osm:way/<id>` / `osm:relation/<id>`; properties `{building, area_m2, accessory}`. `accessory` = `garage|garages|shed|carport` or < 30 m². Coordinates rounded to 6 decimals.
- `n` = stored `footprint.version` + 1, so versions only grow and a client cache keyed by `(mapId, version)` is never stale.
  - Features → `PutObject footprints/<m>/v<n>.geojson.gz` (`Content-Encoding: gzip`, `application/geo+json`), then `footprint = {status: ready, source: osm, version: n, count}`.
  - No features → no object; `footprint = {status: empty, source: osm, version: n, count: 0}`.
  - Both instances failed → `status: failed`, version and count untouched. Logged, not raised: the SQS failure destination is for crashes only.
- Every `META` update: `ConditionExpression attribute_exists(pk)` (a map deleted mid-fetch must not be recreated as a stray item), `ReturnValues NONE`.
- A crash or timeout leaves `pending`; the user refreshes (B8).

### B8 — `footprints_get`, `footprints_refresh`, trigger
- `GET /maps/{mapId}/footprints`: `require_member`, then `META` projected to `pk, sk, footprint`. `pending` → 202 `{status}`. Otherwise 200 `{status, source, version, count, url}`, `url` a 5-minute presigned GET only when `ready`, else null.
- `POST /maps/{mapId}/footprints/refresh`: reads `footprint`, then writes it back whole as `pending` with `requested_at = now`, conditional on `attribute_exists(pk)` and `requested_at` absent or older than 5 min, then async-invokes fetch. Inside the cooldown → 429 `refresh_too_soon`. The cooldown also caps Overpass load (policy: < 100 queries/day) and frees a map stuck `pending` after a crash. 202 `{status: pending}`.
- `maps_create` sets `footprint.requested_at` and async-invokes fetch after the write. An invoke failure is logged and the 201 still returns: the map exists, and failing would make the client retry into a duplicate map. The user refreshes.
- `RateLimitError` (429) joins `errors.py`; B9 reuses it.

## Affected Files / Components
| File / Component | Change | Why |
|-----------------|--------|-----|
| `lambdas/common/overpass.py` | new | B7: query, fallback, GeoJSON |
| `lambdas/common/footprints.py` | put, presign, `start_fetch` | B7, B8 |
| `lambdas/footprints_fetch/handler.py` | new | B7 |
| `lambdas/footprints_get`, `lambdas/footprints_refresh` | new | B8 |
| `lambdas/common/maps.py`, `lambdas/maps_create/handler.py` | `requested_at`, async invoke | B8 |
| `tests/fixtures/overpass/` | recorded | public landmarks only (Independence Hall, the Pentagon) |
| Overture source + DuckDB layer | new, conditional | B8b |

Helpers live in `lambdas/common/`: the deploy zips each handler folder to its root, so a sibling module next to `handler.py` isn't importable as `lambdas.<name>.*` in tests and in Lambda alike.

## Implementation Steps
- [ ] B7 — `footprints_fetch` (~150, needs B3, I6) — curbside-backend#20
- [ ] B8 — `footprints_get` + `footprints_refresh` + `maps_create` trigger (~80, needs B7) — curbside-backend#21
- [ ] B8b — *Conditional on 0a*: Overture source behind the same contract (~150, needs B7). Out until Dom's coverage check. Split: DuckDB layer infra PR first, then the backend source.

## Acceptance criteria
- [ ] B7: recorded fixtures produce the expected features; 429 waits 30 s then uses the second instance; a 502 goes straight to it; both failing marks `failed` without touching version.
- [ ] B7: every `MAP#` read and update stays inside `pk sk center radius_m footprint`, `ReturnValues NONE`.
- [ ] B8: pending → 202; ready → presigned URL that downloads the object; non-member → 404.
- [ ] B8: refresh inside 5 min → 429, nothing invoked.
- [ ] Live: create a map at a public location, `GET /footprints` goes 202 → 200 with a working URL.

## Out of Scope
- Client rendering and pin fallback (IOS-2)
- Deleting a footprint object written by a fetch that raced a map delete (no `DeleteObject` on the fetch role; it's public OSM data)

## Risks / Tradeoffs
- Public Overpass instances rate-limit and queue; mitigated by async fetch, fallback instance, manual refresh, and pins. On 2026-11-17, `overpass-api.de` answered 502 "at capacity" twice before a 200, and `private.coffee` didn't answer from Dom's network. Verify the fallback from Lambda after B7 deploys.
- Recorded fixtures are from public landmarks, never Dom's street (public repo).

## Plan corrections (2026-11-17)
- `footprint` is always written whole (`SET #fp = :fp`). Nested paths would need placeholders for reserved names (`status`, `count`), and `dynamodb:Attributes` is documented for top-level names only, so a nested write risks AccessDenied that moto can't show.
- Epic step 2 says a 60 s Lambda timeout; I6 set 120 s to fit the 30 s backoff. 120 s is right.
- `overpass-api.de` answers 502 when at capacity, not only 429, so non-429 failures also fall back.
- `requested_at` inside `footprint` is new: it lets refresh rate-limit without a new attribute outside the footprints roles' list.

## Open Questions
- [ ] 0a: OSM coverage on Dom's street, rowhouse and apartment areas. Is B8b in MVP?
- [x] 0a: Overpass policy (UA + 429 backoff, accepted for MVP) and ODbL (Derivative Database, satisfied by public repo). See RESEARCH.md.

## Skills / Agents to Use
- lambda-handler, backend-standards, testing; terraform for the B8b layer
