# Plan: Curbside — iOS Households + Near Me

**Epic**: curbside
**Sub-feature ID**: IOS-3
**Repo**: `Xomware/curbside-ios` (base branch `main`)
**Track**: iOS MVP
**Status**: Ready (S6–S8 in review: PRs #20–#26, stacked)
**Created**: 2026-11-16
**Last updated**: 2026-11-17
**Depends on**: IOS-2 (S5b), BE-2 B4/B5/B6 (live), gate 0c (household card + near-me screens in `mockups/curbside.html`)

## Summary
The household card (view, create, edit people/pets/notes/last-talked) and the near-me sheet that answers "who lives here" from the sidewalk. Near-me is the 5-second success metric.

## Backend facts the client must respect (from `curbside-backend/lambdas`)
- `GET /households/{id}` returns the household with `private.people`, `private.pets` and `private.notes` (newest 50, newest first). `GET /maps/{id}` omits notes.
- `PATCH /households/{id}` returns the household **without** `people`/`pets`; the client keeps its own copies.
- Every PATCH needs `version` plus at least one field. Clearing a nullable field means sending explicit `null` (`address`, `building_id`, `last_talked_at`; person `birth_year`, `birthday`, `notes`; pet `species`, `notes`).
- 409 `version_conflict` → refetch the household, keep the user's draft, say so.
- `POST /notes` sets `last_talked_at` to the note time **without** bumping the household `version`.
- Deletes return `{id}`. Optional child fields are absent, not null, when unset.
- `published` is always null in MVP.

## Decisions
- **Navigation**: the map is a `NavigationStack` root; the card is a pushed screen with a "Block" back button, as in the mockup. Create and edit forms are sheets.
- **Age**: `currentYear - birth_year`, shown as a speed-limit sign "AGE 8"; `birth_year_approx` shows "~5" and "Born ~2021, a guess".
- **Published section**: rendered only when `published` is non-null. MVP never shows it.
- **Near-me side of street**: relative bearing = bearing(user → household) − heading. Negative is LEFT, positive is RIGHT. `CLLocationUpdate` carries no heading, so near-me uses a delegate-backed `CLLocationManager`: compass `CLHeading` first, walking course second; with neither, rows show distance only.
- **`last_talked_at` writes** are ISO 8601 without fractional seconds (`2026-11-18T15:00:00Z`). The backend stores the string as sent, and `Date(_:strategy: .iso8601)` rejects fractions on read.
- **Note strip on near-me** shows only for households whose card has loaded notes; `GET /maps` omits notes by design.
- **Nearest 5** by straight-line distance from the in-memory map. No geo index.

## Implementation Steps
Stacked PRs. Line counts are hand-written logic; tests, previews and fixtures excluded.

- [x] **S6a — Household card**, two PRs: data model (~93, PR #20) and screen (~192, PR #21). One PR was 285 lines.
  - Tapping a plaque or a building's household opens the card: mailbox hero, label blade, address, "Talked n days ago"
  - Adults; kids with age signs; pets with species; notes (date + source); published section hidden while null
  - Loading, error-with-retry; edits elsewhere refresh the map pins
  - Tests: decoding with notes, ages (exact, approx, missing), published hidden/shown, section ordering
- [x] **S6b — Create household** (~100, PR #22)
  - "+ ADD" on a building or long-press pin → form prefilled with the reverse-geocoded address, building id and coordinate
  - `POST /households`, then opens the new card and adds the pin
  - Tests: request body (with and without `building_id`), label required, prefill
- [x] **S7a — Edit people and pets**, two PRs: people + `Patch` body (~185, PR #23), pets (~119, PR #24)
  - Add, edit, delete a person (name, adult/kid, birth year, approx, notes) and a pet (name, species, notes)
  - PATCH sends only changed fields, explicit null to clear; 409 refetches and keeps the draft
  - Tests: changed-fields body, null clears, 409 path, delete removes locally
- [x] **S7b — Notes and last talked** (~124, PR #25)
  - Add a note (source `manual`), delete a note, "Talked today" sets `last_talked_at`
  - Tests: note create updates `last_talked_at` without changing `version`; touch PATCH body
- [x] **S8 — Near-me sheet** (~218, PR #26; feed, model and sheet don't work apart)
  - "Near me" on the map dock opens a sheet: heading pill, the 5 nearest households as green exit signs with side + distance, the newest note as the yellow strip
  - Tap a row → card
  - Tests: nearest-5 ordering, left/right at 0/90/180/270 headings, no heading → distance only, empty map

**5-second check (S8 acceptance)**: from the map, "Near me" → sheet with rows rendered in under 1 s on device with the map already loaded (measured with `OSSignposter` around open → first rows). The remaining 4 s is the human reading the sign; Dom does one sidewalk walk and records the result in the PR.

## Out of Scope
- Capture (IOS-4)
- Birthday editing UI (`birthday` MM-DD stays API-only until capture needs it)

## Risks / Tradeoffs
- Heading is noisy at walking speed and absent in the simulator. Distance-only rows are the fallback, so the sheet is never empty because of a bad heading.
- Optimistic UI is not used for edits: a 409 needs the server's copy, and these are not social actions.

## Open Questions
- [ ] None blocking.

## Skills / Agents to Use
- ios-standards; every PR states "verified in app" per `verification.md`
