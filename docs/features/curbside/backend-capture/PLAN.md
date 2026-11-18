# Plan: Curbside — Backend Capture

**Epic**: curbside
**Sub-feature ID**: BE-4
**Repo**: `curbside-backend`
**Track**: iOS MVP
**Status**: Ready
**Created**: 2026-11-16
**Last updated**: 2026-11-17
**Depends on**: BE-2 (B5, B6, live), INF-2 (I7, applied)

## Summary
Voice/text capture: Haiku parses a transcript plus a small roster into a list of proposed operations, the client confirms them, and apply writes them in one transaction. Nothing persists without per-op confirmation.

## Approach
Haiku parsing contract in the epic, pinned below.

### B9 — `capture_parse`
- **Request** as the epic: `{transcript ≤ 2000, source, focus_household_id?, nearby_household_ids ≤ 8}`, ids ULIDs.
- **Order**: `require_member` → roster → quota → model. A bad id never spends quota.
- **Roster** from the sent ids only: one `Query begins_with(HH#<h>)` each. Ids with no household are dropped. Entry `{hh_id, label, address_short, people:[{id,name,role,birth_year}], pets:[{id,name,species}]}`; `address_short` is the address up to its first comma.
- **Quota**: `UpdateItem ACCT#<a> / QUOTA#capture#<yyyy-mm-dd>` `SET counter = if_not_exists(counter, 0) + 1, ttl = +2 days`, condition `attribute_not_exists(counter) OR counter < 50`, `ReturnValues NONE`. Only `pk sk counter ttl` appear (I7 IAM). Fail → 429 `quota_exceeded`. The count isn't refunded when the model call fails.
- **Model**: `claude-haiku-5-5`, `max_tokens` 2048, forced `tool_choice` on one tool, no `cache_control`, 25 s timeout. Stdlib `urllib`, not the SDK: one POST, and the layer stays small. Key from SSM `/curbside/anthropic/api-key` (`ANTHROPIC_API_KEY_PARAM`), cached per container. The system prompt carries today's date so "about 6" resolves.
- **Tool schema**: the epic's op shape plus `needs_review: bool`, set when the household is placed only by relative position ("next door", "across the street"). `fields` lists every entity field name, so the model can't invent one.
- **Server checks**:
  - Output fails pydantic → 502 `parse_failed`. API error or timeout → 503 `model_unavailable`. No retry loop.
  - `household_ref` must be a roster id or a `new:<n>` created by a `create_household` op in the same output. `target_id` must be a person/pet of that roster household. Else the op is dropped and its evidence goes to `unresolved`.
  - Fields are validated against the entity models (`PersonCreate`, `PersonUpdate`, ...); an op that fails moves to `unresolved`. `create_household` fields are `label`, `address?`; the client adds `lat`/`lng` before apply.
- **Response** `{operations, unresolved, base_versions}`. Each op gains `checked`, false when `needs_review`; Dom accepted that default in the mockups. `base_versions` maps each roster household, person and pet id to its `version`.
- Transcript never stored or logged.
- **Eval**: `scripts/eval_capture.py` runs the 15 fixtures in `tests/fixtures/capture/` against the live model and scores expected ops (op, ref, target, field subset). Not in CI. Unit tests use recorded responses from one eval run.

### B10 — `capture_apply`
- **Request** `{operations, base_versions, source: voice|text}`; ops as parse returns them, client-edited and filtered to the checked ones.
- **One transaction item per DynamoDB item**: a transaction can't touch an item twice, so ops on the same item merge.
  - Existing household: `update_household` fields + `last_talked_at` (from `touch_last_talked` or `add_note`, as B6) → one `Update`. A fields change is conditioned on `version = base_versions[h]` and bumps it; a touch only is conditioned on `attribute_exists(pk)` and doesn't bump. Child-only → `ConditionCheck attribute_exists(pk)`.
  - `create_household` → `Put` (`HouseholdCreate`, so `lat`/`lng` required), `attribute_not_exists(pk)`. `new:<n>` resolves to its new id.
  - `add_person`/`add_pet` → `Put`, version 1. `add_note` → `Put` with `source`.
  - `update_person`/`update_pet` → `Update` conditioned on `version = base_versions[target]`, bumped (per-item versioning from B5).
- Fields re-validated with the B4/B5 models. Update op without a base version, or a `new:<n>` with no `create_household` → 400.
- More than 100 transaction items → 413 `too_many_operations`, nothing written.
- Any `ConditionalCheckFailed` → 409 `version_conflict`; the client re-parses.
- Response: one `{op, id}` per op, the id of the entity written.
- IAM (I7): GetItem, Query, PutItem, UpdateItem, ConditionCheckItem. No DeleteItem needed.

## Affected Files / Components
| File / Component | Change | Why |
|-----------------|--------|-----|
| `lambdas/common/capture.py` | prompt, tool schema, roster, checks | B9 |
| `lambdas/common/ssm.py` | cached SecureString read | B9, reused by B12 |
| `lambdas/capture_parse/handler.py` | new | B9 |
| `lambdas/common/capture_apply.py`, `lambdas/capture_apply/handler.py` | new | B10 |
| `tests/fixtures/capture/` (15 synthetic), `scripts/eval_capture.py` | new | prompt gate |

## Implementation Steps
- [ ] B9 — `capture_parse` (~200 + eval script, needs B5, B6, I7) — curbside-backend#22. Eval 15/15 on three live runs, after two prompt rules: a newborn's birth year, and people off the user's street go to `unresolved`
- [ ] B10 — `capture_apply` (~150, needs B9) — curbside-backend#23

## Acceptance criteria
- [ ] B9: unknown ids dropped; invented field names dropped; bad output → 502; 51st parse of the day → 429; transcript absent from logs.
- [ ] B9: a "next door" op comes back `needs_review: true, checked: false`.
- [ ] B9: eval run recorded in the PR; failures explained.
- [ ] B10: mixed ops (new household + person, person update, note) land in one transaction; a stale version → 409 and nothing written; > 100 items → 413.
- [ ] Live: parse and apply one synthetic dump on a test map.

## Out of Scope
- Speech recognition (on-device, IOS-4)
- Retry loops on bad model output
- Delete ops (capture never deletes)

## Risks / Tradeoffs
- Haiku inventing facts about real people: mitigated by evidence spans, per-op confirm, the `needs_review` default and the eval gate.
- Quota counts attempts, not successes. A run of 503s eats into the 50/day.

## Plan corrections (2026-11-17)
- Strict tool use (`strict: true`) isn't used: the server re-validates every field anyway, and the model was seen emitting a `notes` field on `create_household` that the server drops.
- "Cached system prompt" above the epic is gone: no prompt caching (RESEARCH.md §4).
- The 100-item cap is decided: 413, checked after merging ops into items.
- `create_household` from a transcript has no location; the client supplies `lat`/`lng` at apply.

## Open Questions
- [x] Anthropic retention: not retained by default for Haiku (RESEARCH.md §10).
- [x] Haiku price: Haiku 5.5, $0.10/$0.50 per MTok (RESEARCH.md §4).

## Skills / Agents to Use
- anthropic-api, lambda-handler, backend-standards, testing
