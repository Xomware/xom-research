# Plan: Curbside — iOS Map + Footprints

**Epic**: curbside
**Sub-feature ID**: IOS-2
**Repo**: `Xomware/curbside-ios` (base branch `main`)
**Track**: iOS MVP
**Status**: Ready (S4–S5 in review: PRs #15–#19, stacked)
**Created**: 2026-11-16
**Last updated**: 2026-11-17
**Depends on**: S3 (merged), S2a/S2b (merged), BE-2 B3 (live), BE-3 B8 (in progress; S5 codes against the contract below)

## Summary
Place setup, a muted MapKit map with households as plaque pins, then OSM building footprints as tap targets with a long-press pin fallback. S3 (theme) shipped as `Curbside/Theme` + `Curbside/Views/Signs`.

## Decisions
- **One place per account in MVP.** The map screen opens the first map from `GET /maps`; no maps routes to place setup. The multiple-places UI stays post-MVP.
- **SwiftUI `Map`, no `UIViewRepresentable`.** `.standard(emphasis: .muted, pointsOfInterest: .excludingAll)`, households as `Annotation`s, footprints as `MapPolygon(MKPolygon)`. Taps go through `MapReader` → coordinate → our own point-in-polygon, so hit-testing is a pure function with tests.
- **Plaque pins cap at `DynamicTypeSize.large`.** S3's AX2 gallery showed plaques covering neighbouring houses. Card and sheet text still scales fully.
- **Location**: when-in-use only, via `CLServiceSession(authorization: .whenInUse)` + `CLLocationUpdate.liveUpdates()`. No always-on, no background mode. Purpose string: "Curbside uses your location to center the map on your street and to show which houses you're walking past."
- **Search**: one text field, `MKLocalSearch`, first result. Covers users who deny location.
- **Radius**: 150 / 300 / 500 m picker, default 300 (backend accepts 50–1000).
- **Debug demo mode** (`-demo` launch argument, Debug builds only): signed in, every API call answered from synthetic fixtures. It exists so screenshots and UI walkthroughs need no real account; Release builds exclude `Curbside/Demo`.
- **Geocoding**: `MKReverseGeocodingRequest` / `MKMapItem.address` (iOS 26). `CLGeocoder` and `MKPlacemark` are deprecated there.

## Footprint contract (matches `curbside-backend` `footprints_get`, commit 0d2e04b)
`GET /v1/maps/{mapId}/footprints`
- 202 `{data: {status: "pending"}}`: no version, no url
- 200 `{data: {status: "ready", source, version, count, url: "<presigned, 5 min>"}}`
- 200 `{data: {status: "empty"|"failed", source, version, count, url: null}}`

The S3 object is gzipped GeoJSON with `ContentEncoding="gzip"` and `ContentType="application/geo+json"` (B7), so URLSession inflates it. The client also inflates raw gzip bytes if the header is ever missing. Feature `id` = `osm:way/<id>`; properties `{building, area_m2, accessory}`.

Client behaviour: when `place.footprint` (from `GET /maps`) is `ready` and `Caches/footprints/<mapId>/v<n>.geojson` exists, skip the API. Otherwise fetch, write the decompressed file, delete older versions. Pending, empty, failed or any error leaves pins only; the next map load tries again.

## Implementation Steps
Stacked PRs. Line counts are hand-written logic; tests, previews and fixtures excluded.

- [x] **S4-0 — Debug demo mode + screenshot scheme** (~40, PR #15). Split from S4a, which came to 231 lines.
  - `-demo <scenario>` launch argument; Release excludes `Curbside/Demo`; `Screenshots` scheme writes PNGs, appearance set per run with `simctl ui`
- [x] **S4a — Map screen with plaque pins** (~194, PR #16)
  - `Place` and `Household` models decoding `maps_get` exactly (`private` → `mine`, `published`, `version`)
  - Muted map, plaque per household (house number from the address), street-sign header, loading / error-with-retry / empty states
  - Plaque pins capped at `.large`; past 400 m camera altitude they shrink to number tabs, because real houses sit closer than a name blade is wide
  - Tests: decoding against handler output under moto, `GET /maps` → `GET /maps/{id}`, no-map routing, error state
- [x] **S4b — Place setup** (~154, PR #17)
  - Current location (when-in-use) or address search, radius picker, map preview with radius circle, name prefilled from the street
  - `POST /v1/maps` then opens the map; 400 shows an inline error
  - Tests: request body, name validation, create → map routing, denied location keeps search usable
- [x] **S5a — Footprint overlays** (~94, PR #18)
  - Fetch per contract, gzip handling, disk cache by `(mapId, version)`, `MKGeoJSONDecoder` → polygons
  - Roof styling from the mockup; accessory buildings lighter; "© OpenStreetMap contributors" whenever outlines are drawn
  - Tests: cache hit skips download, version bump replaces file, pending/empty/failed leave no polygons, gzip and plain bodies both parse
- [x] **S5b — Tap and long-press** (~147, PR #19)
  - Tap → point-in-polygon (accessory skipped) → selected building in hazard yellow → its households, or "+ ADD"
  - Several households in one polygon list together
  - Long-press anywhere (no polygon needed) → reverse geocode → "+ ADD" at that point
  - Tests: hit-test (inside, outside, hole, accessory), building → households grouping, reverse-geocode address passed through

## Out of Scope
- Household card and create form (IOS-3 S6)
- Footprint refresh button (`POST /footprints/refresh`) — settings, S10

## Risks / Tradeoffs
- `MapPolygon` per building in SwiftUI: fine at a few hundred polygons. If a 500 m place gets slow, move overlays to `MKMapView`.
- Long-press via `LongPressGesture.sequenced(before: DragGesture(minimumDistance: 0))` fires on finger lift, not at 0.5 s. Accepted; it is the only way to get a location from a SwiftUI long-press on `Map`.
- Screenshots use the debug demo mode with synthetic households over public OSM geometry, never Dom's street.

## Open Questions
- [x] Which mockup direction (0c)? Curbside, chosen 2026-11-16.
- [x] Does 0a change attribution text? No: "© OpenStreetMap contributors" (RESEARCH.md).

## Skills / Agents to Use
- ios-standards; every PR states "verified in app" per `verification.md`
