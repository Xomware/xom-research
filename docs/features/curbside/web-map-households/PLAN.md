# Plan: Curbside — Web Map + Households

**Epic**: curbside
**Sub-feature ID**: WEB-2
**Repo**: `curbside-frontend`
**Track**: Web follow-up
**Status**: Ready
**Created**: 2026-11-16
**Depends on**: WEB-1 (W1), gate 0c (theme), gate 0a (MapKit JS quota, attribution)

## Summary
MapKit JS map with footprint polygons in the chosen theme, and the household card with view and edit for people, pets and notes. The desk-side counterpart to IOS-2 and IOS-3.

## Approach
MapKit JS with the origin-restricted token from I9. Footprints from the same presigned URL and version contract. Same `private`/`published` rendering split and 409 handling as iOS.

## Affected Files / Components
| File / Component | Change | Why |
|-----------------|--------|-----|
| map component, footprint layer, theme | new | W2 |
| household card + edit | new | W3 |

## Implementation Steps
- [ ] W2 — Map view: MapKit JS, footprint polygons, theme (~200, needs W1)
- [ ] W3 — Household card view + edit (~200, needs W2)

## Out of Scope
- Near-me (iOS only), capture (WEB-3)

## Risks / Tradeoffs
- MapKit JS quota and token handling depend on 0a findings.

## Open Questions
- [ ] TBD

## Skills / Agents to Use
- angular-component, frontend-standards; review with "verified in app"

## Plan corrections (2026-11-17)

**Split.** W2 + W3 at ~400 lines became five stacked PRs on `main` (issue #10):

1. W2a — models, `MapStore` (first place, households, selection), footprint loader, geo helpers, `mapkitToken` config (~170)
2. W2b — map page: MapKit JS wrapper, plaque list, selection panel, OSM credit, "Map unavailable" (~200)
3. W3a — household card view at `/households/:id` (~140)
4. W3b — edit people and pets (~170)
5. W3c — notes, talked today, add a household at a picked building or dropped pin (~150)

**MapKit JS token is optional.** `environment.mapkitToken` comes from Infisical `/curbside` `MAPKIT_JS_TOKEN` via
`scripts/write-environment.mjs`, which reads that one extra name and allows it to be missing. Empty → the map area
says "Map unavailable" and the plaque list beside it still opens every card. Outlines, tap, long-press and
placing new houses need the map.

**Footprints are cached in memory, per tab.** iOS caches on disk by `(mapId, version)`. On the web a persistent
cache (Cache Storage, IndexedDB) would leave the outlines around the user's home on a shared computer after the
tab closes, which the `sessionStorage` token decision deliberately avoids. The PKCE redirect is a full page load,
so the memory cache also cannot leak across accounts. A reload costs one presigned GET.

**The footprints bucket needs CORS.** `curbside-infrastructure/terraform/s3_footprints.tf` has no
`aws_s3_bucket_cors_configuration`, so a browser `fetch` of the presigned URL is blocked. Until infra allows `GET`
from `https://curbside.xomware.com` and `http://localhost:4200`, the web map shows plaques without outlines (the
designed fallback). Same for MapKit's own failure modes: anything failing leaves plaques only.

**One place per account, no place setup on the web.** No map → "Set up your street in the iPhone app". Setup needs
search and location, which belong with MapKit's services once the token exists.

**Map behaviour mirrors iOS S4–S5:** muted standard map, POIs off; plaques as annotations; outlines as polygon
overlays (accessory lighter); single tap → our point-in-polygon → selected building in hazard yellow → its
households or "+ ADD"; long-press drops a pin; `mapkit.Geocoder` reverse lookup prefills the address.

**Card behaviour mirrors iOS S6–S7** (see `ios-households/PLAN.md` backend facts): PATCH responses omit
people/pets so the card keeps its own; notes set `last_talked_at` without a version bump; null clears; 409 reloads
the household and keeps the user's draft open.

**PRs (stacked, in merge order):** W2a #12, W2b #13, W3a #14, W3b #15, W3c #16 (Xomware/curbside-frontend).
