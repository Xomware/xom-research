# Plan: Curbside — Web Capture

**Epic**: curbside
**Sub-feature ID**: WEB-3
**Repo**: `curbside-frontend`
**Track**: Web follow-up
**Status**: Ready
**Created**: 2026-11-16
**Depends on**: WEB-2 (W3), gate 0c (diff-confirm screen)

## Summary
Text-only capture on web: type a note, parse, confirm the diff, apply. Same endpoints and contract as iOS.

## Approach
`source: "text"`. Reuse the diff-confirm design from the chosen mockup. 409 → re-parse, 429 → quota message.

## Affected Files / Components
| File / Component | Change | Why |
|-----------------|--------|-----|
| capture component + diff confirm | new | W4 |

## Implementation Steps
- [ ] W4 — Text capture → diff confirm → apply (~150, needs W3)

## Out of Scope
- Voice on web

## Risks / Tradeoffs
- TBD

## Open Questions
- [ ] TBD

## Skills / Agents to Use
- angular-component, frontend-standards; review with "verified in app"

## Plan corrections (2026-11-17)

**Split.** W4 at ~150 became two stacked PRs on top of W3c (issue #11):

1. W4a — capture model: parse request, roster, dependant toggles, edits, placements, apply body, errors, summaries,
   evidence marks (~200)
2. W4b — capture page: type → review → place new houses → save, TALK on the map dock and selection panel (~200)

**Rules carried over from `ios-capture/PLAN.md`:**
- `source: "text"` always. Transcript lives in the page's model only; nothing in storage, gone on save, discard or
  leaving the page.
- Roster: focus = first household in the selected building; nearby = the 8 nearest to the selected spot, else the
  place centre. The web has no location fix: this is a desk companion.
- Apply sends checked ops only, every `create_household` first, each with `lat`/`lng` (+ `building_id`, address
  from the reverse lookup when the parse had none).
- Unchecking a new house unchecks its dependants; checking a dependant checks its house.
- Opened on an empty building or dropped pin, `new:1` starts placed there. Otherwise "Place it" opens the map in
  picking mode. Save stays disabled while a checked new house is unplaced. Without a MapKit token there is no
  picker: the row says so and the user unchecks the new house to save the rest.
- 409 → parse again and say so; 429 daily limit; 502 parse failed; 503 reading down; 413 too many changes.
- Text fields (label, address, name, species, birthday, notes, text) are editable on review; numbers and flags are
  fixed on the card. DETOUR lists unresolved phrases without "Pick a house".

**PRs (stacked on #16, in merge order):** W4a #17, W4b #18 (Xomware/curbside-frontend).
