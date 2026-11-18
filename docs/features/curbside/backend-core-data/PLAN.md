# Plan: Curbside — Backend Core Data

**Epic**: curbside
**Sub-feature ID**: BE-2
**Repo**: `curbside-backend`
**Track**: iOS MVP
**Status**: Ready
**Created**: 2026-11-16
**Last updated**: 2026-11-17
**Depends on**: BE-1 (B2, merged), INF-2 (I5, applied)

## Summary
CRUD for maps, households, people, pets and notes on the single table, with the fixed `private`/`published` household shape and optimistic version conditions. Everything the iOS map and household screens read and write.

## Approach
Access patterns 2–6 from the epic's DynamoDB section. Four stacked PRs, each branched off the previous one, merged in order.

- **Versioning**: every mutable item (map `META`, household, person, pet) carries `version`, starting at 1. PATCH bodies must send `version`; the update is `ConditionExpression #v = :expected` and bumps it. Mismatch → 409 `version_conflict`. A missing item → 404 (told apart with `ReturnValuesOnConditionCheckFailure=ALL_OLD`).
- **Child writes do not bump the household's version.** `people_create`/`pets_create` have no `UpdateItem` (I5 IAM), so a child carries its own `version`. B10 conditions each op on the item it touches.
- **Privacy split**: household items hold private fields only. Responses always return `published: null`; post-MVP card data stays on `ACCT#<linked>/CARD` and is never copied in. Write models use `extra="forbid"`, so a body can't set `published` or `linked_account_id`.
- **Path ids** (`mapId`, `hhId`, `personId`, `petId`, `noteId`) must match the ULID shape, else 404. Without that, `hhId=01` would prefix-match every household in `households_delete`.
- **`require_member` projects `pk, sk, role`**: footprints roles deny any `MAP#*` read without a projection (INF-2), and it is the shared membership check.
- **Pagination**: no client cursors in MVP. A map is tens to a few hundred households; the server follows `LastEvaluatedKey`. Notes return newest 50.
- **Field names** are snake_case. `META` uses `center {lat, lng}`, `radius_m`, `footprint`, which are the attribute names the footprints IAM allows.

## Affected Files / Components
| File / Component | Change | PR |
|-----------------|--------|----|
| `lambdas/common/dynamo.py` | `get_item` projection, `query_prefix` on gsi1 / whole partition, `batch_get`, `delete_keys`, `update_versioned`, `to_item` (float → Decimal) | B3 |
| `lambdas/common/errors.py` | `ConflictError` 409 | B3 |
| `lambdas/common/maps.py` | `META` + `MEMBER` items, map view | B3 |
| `lambdas/common/footprints.py` | delete `footprints/<mapId>/` objects | B3 |
| `lambdas/common/households.py` | household view assembly (people, pets), path-id check | B4 |
| `lambdas/common/models.py` | pydantic create/update models | B3–B6 |
| `lambdas/maps_{list,create,get,update,delete}` | new | B3 |
| `lambdas/households_{create,get,update,delete}` | new; `maps_get` gains `households` | B4 |
| `lambdas/{people,pets}_{create,update,delete}` | new | B5 |
| `lambdas/notes_{create,delete}` | new; `households_get` gains `notes` | B6 |

## Endpoints
| Route | Behavior |
|---|---|
| `GET /maps` | gsi1 `ACCT#<a>` / `MAP#` → `BatchGetItem` META → maps with caller's `role` |
| `POST /maps` | `{name, center, radius_m=300}` → one transaction: `META` (`footprint.status=pending`) + `MEMBER#<a>` (gsi1 `ACCT#<a>`/`MAP#<m>`, `role=owner`). 201 |
| `GET /maps/{mapId}` | `{map, households}` (households from B4) |
| `PATCH /maps/{mapId}` | `{version, name?, center?, radius_m?}` → updated map |
| `DELETE /maps/{mapId}` | S3 `footprints/<m>/` first, then every item in the partition, `MEMBER#` items last so a partial failure stays retryable. 200 `{id}` |
| `POST /maps/{m}/households` | `{label, lat, lng, address?, building_id?}` → 201 household |
| `GET /maps/{m}/households/{h}` | household + people + pets, plus `private.notes` newest 50 (B6) |
| `PATCH /maps/{m}/households/{h}` | `{version, label?, address?, lat?, lng?, building_id?, last_talked_at?}`; null clears an optional field. Response omits `people`/`pets` (role has no Query) |
| `DELETE /maps/{m}/households/{h}` | `HH#<h>*` + `NOTE#<h>#*` → `BatchWriteItem` |
| `POST /…/people`, `/…/pets` | `ConditionCheck` household exists + `Put` child, one transaction. 404 if household missing |
| `PATCH /…/people/{p}`, `/…/pets/{x}` | versioned update |
| `DELETE /…/people/{p}`, `/…/pets/{x}`, `/…/notes/{n}` | conditional delete, 404 if missing, 200 `{id}` |
| `POST /…/notes` | `{text ≤2000, source=manual}` → one transaction: `Put NOTE#<h>#<ulid>` + `Update` household `last_talked_at=now` (no version bump) |

## Implementation Steps
- [ ] B3 — maps CRUD + membership + common helpers (~200) — curbside-backend#10
- [ ] B4 — households + view assembly, `maps_get` households (~95) — #11
- [ ] B5 — people + pets (~115) — #12
- [ ] B6 — notes + `last_talked_at`, `households_get` notes (~40) — #13

PRs target `main` (INF-2 precedent); each includes its ancestors' diff until they merge.

## Acceptance criteria
- [ ] Every `/maps/{mapId}/…` handler calls `require_member`; a non-member gets the same 404 `map_not_found` as a missing map.
- [ ] Each handler uses only the table actions its I5 role grants (`TransactWriteItems` counted as its Put/Update/ConditionCheck/Delete parts).
- [ ] Stale `version` → 409 `version_conflict`; missing item → 404.
- [ ] `maps_delete` and `households_delete` are tested with > 25 items.
- [ ] Household responses always carry `published: null`.
- [ ] pytest + ruff clean; after each merge, each new route is invoked live with a real token.

## Out of Scope
- `maps_create` async-invoking `footprints_fetch` (B8)
- Refetching footprints when `maps_update` changes `center` or `radius_m` (B8: the client calls refresh)
- `editor` role and owner-only delete (post-MVP: today every member is an owner)
- Profile card / `published` population (post-MVP)

## Risks / Tradeoffs
- `update_versioned` uses `ReturnValues=ALL_NEW`. B8's `footprints_refresh` can't reuse it: its role only allows NONE / UPDATED_*.
- `maps_delete` on a large map runs paginated queries and batch deletes inside a 10 s timeout. Fine at hundreds of items; a retry finishes the job if it ever isn't.

## Open Questions
- [ ] Default radius 300 m, and can a place grow later? (Accepted 50–1000 m for now.)

## Skills / Agents to Use
- lambda-handler, backend-standards, database, python
