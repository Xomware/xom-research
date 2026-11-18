# Plan: Curbside — iOS Capture

**Epic**: curbside
**Sub-feature ID**: IOS-4
**Repo**: `Xomware/curbside-ios` (base branch `main`)
**Track**: iOS MVP
**Status**: Ready (S9a–S9e in review: PRs #29–#33, stacked)
**Created**: 2026-11-16
**Last updated**: 2026-11-17
**Depends on**: IOS-3 (S7, merged), BE-4 (B9 `capture_parse`, B10 `capture_apply`, both live), gate 0c (capture confirm screen in `mockups/curbside.html`)

## Summary
On-device speech or typed text → parse → a diff the user can uncheck or edit → apply. This is the 15-second log-a-conversation metric.

## Backend facts the client must respect (from `curbside-backend/lambdas`, verified under moto)
- `POST /maps/{mapId}/capture/parse` `{transcript 1–2000, source voice|text, focus_household_id?, nearby_household_ids ≤ 8}` → `{operations, unresolved, base_versions}`. Each op: `op, household_ref, target_id, fields, evidence, confidence, needs_review, checked`.
- Only the households sent go to the model. Ops on anyone else come back as `unresolved` strings, which are short descriptions, not always verbatim spans.
- **The model can emit ops on `new:N` before that ref's `create_household`** (recorded `next_door_only` case). `capture_apply` rejects that order with 400, so the client sends every `create_household` first.
- `capture_apply` 400s an op on `new:N` whose `create_household` was left out, so unchecking a create unchecks its dependants, and checking a dependant checks its create.
- `create_household` fields from parse are `label` and `address` only. The client adds `lat`, `lng` and optional `building_id` (`HouseholdCreate` is strict; nothing else).
- Apply returns `[{op, id}]` only. The client refetches `GET /maps/{id}` afterwards rather than rebuilding households from ops.
- 429 `quota_exceeded` (50 a day), 502 `parse_failed`, 503 `model_unavailable` on parse; 409 `version_conflict` and 413 `too_many_operations` on apply.

## Decisions
- **Speech: `SpeechAnalyzer` + `SpeechTranscriber` (iOS 26), not `SFSpeechRecognizer`.** The new API has no server path at all, so on-device is a property of the API rather than a flag someone can forget. It handles long-form audio and the deployment target is already 26. Only microphone permission is asked.
- **No server fallback.** When `SpeechTranscriber` is unavailable (unsupported hardware or locale, simulator) or the mic is denied, the screen says so and the text field takes over. This answers the open question: transcripts never leave the device except as the parse request.
- **Transcripts are never persisted.** The text lives in the capture model only and is dropped on save, discard or dismiss. No `@SceneStorage`, no drafts.
- **Roster**: focus = the first household in the selected building, if any. Nearby = the 8 nearest households to the selected spot, else the user's location (one when-in-use fix), else the place center.
- **New households are placed by the user.** If capture opens from an empty building or a dropped pin, `new:1` starts placed there. Otherwise its row shows "Place it", which opens a map of the place: tap a building (or open ground) to set `lat`/`lng`/`building_id`. Save stays disabled while a checked new house is unplaced.
- **Entry point**: the TALK stop sign on the map dock, beside Near me, as in the mockup. The mockup's Index button is not part of this.
- **Editing**: tap a change to edit its text fields (name, label, address, species, notes, note text). Numbers and flags are not editable here; uncheck the change and fix it on the card instead.
- **DETOUR** lists unresolved items without the mockup's "Pick a house" button. Assigning a phrase to a house would need a second parse; the user can add it on the card.
- **429** reads as the daily limit. API Gateway throttling also answers 429; at one user's rate that is not worth a second message.

## Implementation Steps
Stacked PRs. Line counts are hand-written logic; tests, previews and fixtures excluded.

- [x] **S9a — Capture model** (~200, PR #29)
  - `CaptureOp` decoding parse output; `Patch.Value` gains decoding and `.double`
  - Roster, toggle with new-house dependants, placement, apply body (checked only, creates first, lat/lng added), 409 → re-parse, error messages; map refresh after apply
  - Tests: bodies are handler output under moto; the reordered apply body was run through `capture_apply`
- [x] **S9b — Rows** (~190, PR #30): `CaptureSummary` for every op kind, `evidenceMarks`, `OpRow`, `CheckboxStyle`
- [x] **S9c — Review screen + place picker** (~170, PR #31)
- [x] **S9d — Record and type, TALK** (~200, PR #32): `SpeechAnalyzer` recorder, input screen, TALK on the dock and on the selection panel (the only way to set a focus household), 3 s location timeout, demo fixtures, screenshots
- [x] **S9e — Edit a change** (~60, PR #33)

## Out of Scope
- Server parsing logic (BE-4)
- Capture from the household card

## Risks / Tradeoffs
- `SpeechTranscriber` needs a downloaded model per locale. First use downloads it (`AssetInventory`); offline first use falls back to typing.
- Devices without `SpeechTranscriber` support get typing only. `DictationTranscriber` would widen support; not worth the second path until someone hits it.
- The 15-second metric is measured by Dom on device, recorded in the S9c PR.

## Open Questions
- [x] Fallback when on-device recognition is unavailable? Typing, never server recognition.

## Plan corrections (2026-11-17)
- One PR (~200) became five: the measured total is ~820 lines.
- TALK also sits on the selection panel: the panel replaces the dock, so a dock-only TALK could never send a focus household.
- Added the apply-ordering, dependant-toggle and client-placement rules above; the original plan assumed apply took parse output as-is.
- `SFSpeechRecognizer` replaced by `SpeechAnalyzer` (reason above).

## Skills / Agents to Use
- ios-standards; every PR states "verified in app" per `verification.md`
