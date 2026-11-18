# Brainstorm: curbside

Status: Brainstorm. Date: 2026-11-16. No RESEARCH.md exists, so the map and data facts below should be verified in `/plan` (flagged inline).

## Premise check

- **The map is the hook, but it doesn't keep people using the app.** People keep using it for the 5-second sidewalk lookup, the voice dump, and the nudges. A gorgeous map with 3 houses filled in is a dead app. Data entry decides whether this lives or dies, so voice capture belongs in MVP and footprints can degrade to pins.
- **A footprint is not a household.** Footprints include garages and sheds. A rowhouse block or duplex is one polygon with several households, and an apartment building holds dozens. Treat the footprint as a tap target only. The household record owns its address and lat/lng, with an optional `building_id`. Keying data on footprints breaks on day one in any non-suburban place.
- **The proposed MVP is too big.** Map + footprints + cards + near-me + voice + ICS + shared household is 4-6 weeks of evenings. Cut ICS and sharing from the first cut (see Recommendation). Sharing is a data-model decision now and a feature later.
- **Google sign-in alone will get rejected on iOS.** App Store guideline 4.8 requires an equivalent privacy-preserving login (Sign in with Apple) whenever a third-party social login is offered. Cognito federates both. Plan for Apple + Google.
- **Encryption at rest doesn't buy much by itself.** DynamoDB and S3 already encrypt at rest by default. The real question is whether the *server* can read the notes. Server-side Haiku parsing, server-rendered ICS feeds and server-side sharing all require plaintext on the server. True E2E rules out all three unless they move on-device. Decide this explicitly.

## Phase 1 — Explore

Platform
- Native SwiftUI only (MapKit, CoreLocation, Speech, Contacts, EventKit, Face ID)
- Angular PWA only (Mapbox GL JS)
- iOS first, read-only Angular web view later for desk editing
- Local-first iOS on SwiftData + CloudKit with no AWS backend at all
- iOS home-screen widget: "nearest house" without opening the app
- Apple Watch complication for the sidewalk lookup (later, maybe never)

Map and house-tap
- Reverse-geocode a tapped point (CLGeocoder, free) and drop a pin. Cheapest, works everywhere, looks generic.
- Overture Maps buildings theme (GeoParquet on public S3, merges OSM + Microsoft ML footprints, stable GERS IDs). Extract once per place via a DuckDB Lambda, cache GeoJSON in S3.
- Microsoft US Building Footprints (~130M polygons, per-state GeoJSON, ODbL). Same data Overture mostly includes, without the stable IDs.
- OSM Overpass API for `building=*` in a bbox. Free, one HTTP call. Coverage is good in many US suburbs and patchy in others.
- Mapbox Streets building layer + `queryRenderedFeatures`. Tappable out of the box, but feature IDs aren't stable and polygons get clipped at tile edges.
- Google Maps: buildings render but aren't queryable. No footprint API for this use.
- MapKit 3D buildings: render at high zoom but aren't selectable (`selectableMapFeatures` covers POIs, territories and physical features, not buildings).
- Skip the basemap entirely: draw a stylized "plat map" of just the street (footprints + road centerlines) in SwiftUI Canvas or as MapKit overlays on a muted base. This is the bold literal theme.
- Manual "draw my block" mode: user drags rectangles. Fallback where footprint data is missing.

Data and privacy
- Server-readable DynamoDB (default AWS encryption) + Face ID app lock
- Client-side encryption of free-text fields only; structured fields (name, birthday) stay readable so ICS still works
- Full E2E with keys in iCloud Keychain. Kills server Haiku, ICS and web.
- CloudKit private DB (Apple-managed, E2E if the user has Advanced Data Protection on)
- Photos stored in S3 under per-user prefixes with presigned URLs, never public
- "Panic delete" per household and full account export (JSON) from day one
- No face recognition, no data enrichment of any kind. Exa is already dropped, and the same reasoning rules out auto-pulling property records or social profiles (App Store 5.1.2 bars compiling personal data not provided by the user).

Sharing
- Single user only
- Shared "map" entity with members: spouse is a second Cognito user with edit rights
- CloudKit `CKShare` (free, iPhone-only for both people)
- Export/import a map file (low-tech sharing)
- Friends as entries (address book), never as collaborating users. Collaborating friends is a social network and out of scope.

Calendar
- Subscribable ICS feed per map from a Lambda at an unguessable, revocable URL. Works in Google + Apple and survives sharing.
- EventKit writes directly to the device calendar. Zero backend. Syncs to Google if the user's iOS default calendar is a Google account. Spouse gets duplicates.
- Google Calendar API writes. Needs OAuth scopes and verification. Most work for the least gain.
- In-app reminders + local notifications only, no calendar at all for MVP

Capture
- Voice dump → on-device `SFSpeechRecognizer` transcript → Haiku Lambda → structured JSON diff → user confirms
- Text dump through the same pipeline (typing on the sidewalk is real too)
- Share-sheet extension: paste a text thread snippet in and parse it
- Contacts import for friends (CNContactStore), mapped to households by address

Theme and name ideas
- "Nosy neighbor": blinds, binoculars, curb-level view. Funny, but slightly on the nose given the privacy angle.
- Subdivision survey / plat map: lot lines, lot numbers, drafting-paper blue, surveyor stamps for "met", "lent mower", "new move-in". Bold and literal.
- Block party: string lights, folding tables, cooler as the lend/borrow icon
- Names: Over the Fence, Curbside, Plat, Block Book, Xomneighbor

## Phase 2 — Converge

## Option 1: Native iOS on the Xomware AWS stack

**What**: SwiftUI + MapKit app with Overture-derived footprints drawn as overlays, backed by the standard three-repo setup (`curbside-ios`, `curbside-backend`, `curbside-infrastructure`).

**How it works**: Adding a place sends its center to a Lambda, which pulls building polygons within ~300m from Overture (DuckDB over the public GeoParquet, bbox-filtered) or Overpass as the cheap first pass. The Lambda caches the GeoJSON in S3 and returns it. The app renders the polygons as `MKPolygon` overlays on a muted MapKit base (`emphasisStyle: .muted`, POIs filtered out), filled by closeness. Tapping hit-tests the polygons. If a polygon has no household yet, the app reverse-geocodes it with CLGeocoder for an address prefill. Data lives in a single DynamoDB table keyed by map (`MAP#id` → `HOUSEHOLD#`, `PERSON#`, `NOTE#`, `ITEM#`, `LOAN#`), with a `USER#id → MAP#id` membership item so sharing is a data change, not a migration. Voice goes on-device to text, then to a Haiku Lambda that returns a proposed diff the user confirms. Auth is Cognito with Apple + Google federation.

**Pros**:
- Best fit for sidewalk use: CoreLocation + heading for "near me" (show the 3-5 nearest, with the side of the street inferred from heading), Face ID lock, widget later
- MapKit is free with no usage caps, and so is CLGeocoder (Apple rate-limits it, which is fine at this scale)
- Matches the xomify-ios / xomfit-ios precedent and the existing Terraform, Infisical and Cognito patterns
- Server-side data enables the ICS feed, spouse sharing and a web view later without re-architecting
- Cost is roughly zero: Lambda + DynamoDB on-demand at personal scale is pennies, and Haiku 4.5 at ~800 tokens per dump is about $0.002 each

**Cons / Risks**:
- Most moving parts: three repos, Cognito with two IdPs, a footprint pipeline, and App Store review
- MapKit can't be restyled into a truly illustrated basemap. The bold look has to come from your own overlays plus a muted base, or a full `MKTileOverlay` replacement, which means hosting tiles.
- Footprint coverage varies. Verify Overture/Overpass coverage for Dom's actual street before committing. Keep the manual rectangle/pin fallback in scope.
- ODbL attribution is required for Overture/OSM/Microsoft-derived polygons. Annotations keyed to building IDs are very likely a "produced work", not a derivative database, but confirm in `/plan`.
- Server-readable sensitive data about third parties. Mitigate with Face ID lock, per-user IAM-scoped access, no analytics on note content, and a one-tap export/delete.
- A DuckDB Lambda layer is big, and Overture bbox queries over S3 can take tens of seconds. Fine for a one-time per-place job, but time it before the plan assumes it.

**Best if**: Dom wants spouse sharing across accounts, Google sign-in, and a future web view, and is willing to pay the setup cost once.

## Option 2: Local-first iOS on SwiftData + CloudKit

**What**: The same SwiftUI/MapKit app with no Xomware backend for user data. SwiftData syncs through the user's iCloud, sharing uses `CKShare`, and calendar goes through EventKit.

**How it works**: All records live on the device and in the user's private CloudKit database. The spouse joins via a CloudKit share. Footprints still come from a stateless Lambda (or are fetched straight from Overpass on device). Haiku parsing goes through one stateless proxy Lambda that holds the API key and stores nothing. Birthdays are written to the device calendar with EventKit. There is no Cognito, because the Apple ID is the identity.

**Pros**:
- Strongest privacy story: Dom never holds third-party data, and with Advanced Data Protection the notes are E2E encrypted. That matters given the non-consenting-subjects problem.
- Least infra: one or two stateless Lambdas, no DynamoDB, no auth stack. Effectively one repo plus a small backend.
- Offline by default, which matters on a sidewalk with one bar of signal
- CloudKit is free at this scale

**Cons / Risks**:
- Drops Google sign-in and any web or Android access. Both people must be on iPhone.
- SwiftData + CloudKit sharing is notoriously fiddly: `CKShare` with SwiftData still needs Core Data bridging, debugging is painful, and Dom has no precedent for it
- An ICS feed isn't possible without a server copy of the data. EventKit duplicates events per device or user.
- Breaks from the org's AWS defaults, so the Infisical, Terraform and Cognito muscle memory doesn't transfer
- Haiku still sees the transcript, so the E2E claim stops at storage

**Best if**: Dom and his spouse are both on iPhone, web access isn't wanted, and privacy is the headline feature.

## Option 3: Angular PWA + Mapbox GL JS on AWS

**What**: A web app on the Angular three-repo template, using Mapbox's built-in building layer for tappable houses.

**How it works**: Mapbox GL JS renders a fully custom style (an illustrated plat-map look is achievable in Mapbox Studio), and `queryRenderedFeatures` on the building layer gives tap targets without a footprint pipeline. Backend and data model are the same as Option 1. Voice uses the Web Speech API where available, falling back to recording audio and transcribing server-side.

**Pros**:
- Fastest route to tappable buildings and the most control over the illustrated look
- Reuses the Angular template, S3/CloudFront hosting and deploy workflows Dom already has
- Works on any device for the spouse, and the desktop for bulk editing
- Mapbox free tier is 50k web map loads/month. Personal use won't approach it. Verify current terms in `/plan`.

**Cons / Risks**:
- Worst at the core use case: no Face ID, flaky and slow geolocation prompts on iOS Safari, Web Speech API support on iOS is unreliable, and no widget
- Mapbox building features lack stable IDs and get clipped at tile seams, so households still need their own lat/lng key (fine given the premise above) and the tap-to-household match is approximate
- Mapbox temporary geocoding terms forbid storing results. Use Apple/Overture/OSM for addresses or pay for permanent geocoding.
- Google Maps is no better here. Its $200 monthly credit was replaced by per-SKU free caps in 2025, and its buildings aren't queryable.

**Best if**: The spouse isn't on iPhone, or Dom decides the desktop "curate my neighborhood" session matters more than the sidewalk lookup.

## Phase 3 — Recommendation

**Option 1: native iOS on the Xomware AWS stack.** The defining moment is standing on the sidewalk trying to remember a kid's name. That needs native location, heading, Face ID and on-device speech. Option 3 can't deliver it well. Option 2 delivers it but gives up Google sign-in and cross-platform sharing, and takes on CloudKit sharing complexity Dom has no precedent for, to solve a privacy problem that a Face ID lock, export/delete and "never enrich" mostly address.

It depends on two things to confirm before `/plan`:
1. **Is the spouse on iPhone, and is a web view wanted within a year?** If iPhone and no, Option 2 becomes a real contender, and privacy-first is a stronger App Store story.
2. **Does Overture/Overpass have clean footprints for Dom's street?** Check this before building the map. If coverage is bad, the MVP is pins + reverse-geocode with footprints as a later enhancement, and Option 1 still holds.

Proposed MVP, cut down from the original list:
1. Place setup + footprint fetch (Overpass first, Overture if coverage disappoints) + plat-map overlay styling
2. Household cards: adults, kids (birth year), pets, notes, last-talked
3. Near-me sheet (nearest 3-5 by distance + heading)
4. Voice/text dump → Haiku diff → confirm
5. Data model with `MAP#` membership from day one, but no sharing UI

Deferred: shared household UI, ICS feed (the URL carries names + birthdays to Google's fetchers, so it needs the revocable-token design done properly), lend/borrow nudges, item search, multiple places, friends map, holiday card export, and the Angular web view.

Next gate before code: themed mockups. The plat-map/survey direction is the strongest literal theme. Produce 2-3 visual directions first, per Dom's preference.

## Open questions for `/plan`

- Exact Mapbox/MapKit JS/Google free-tier numbers as of today. Only relevant if a web view lands.
- Anthropic API data retention for transcripts that name third parties. Confirm the current policy and state it in the privacy policy.
- Does Cognito's hosted UI give acceptable UX for Apple + Google on iOS, or does the app need native Sign in with Apple + token exchange?
- Photo storage: S3 presigned URLs vs keeping face photos on device only (stronger privacy, but no spouse sync).
- App Store privacy label and review notes explaining the user-entered-only data model.

## Decision (2026-11-16)

- Native iOS app AND Angular web app, one shared backend (API Gateway + Lambda + DynamoDB + one Cognito user pool).
- Sign in with Apple and Google on both clients, federated through Cognito.
- Open: account linking when the same person uses Apple on one client and Google on the other (Apple private-relay email breaks email matching).
- Lean: MapKit JS on web so both clients share the map look and quota.
- Lean: ship iOS MVP first; keep the API client-agnostic so web follows without backend changes.

## Idea: self-published profile cards (2026-11-16)

- Each user maintains their own card (name, household, birthdays, pets, address, socials, gift ideas) and picks which fields to share.
- Connected friends see the shared fields live on their entry for you; revoking removes them immediately.
- Private notes, items and logs about someone are never shared, in either direction.
- Connect via QR / invite link for v1. Contact-list matching later, if at all (uploads non-users' numbers; hashed phones are brute-forceable).
- Data model from day one: an entry = self-published fields (read-only, from their card) + my private fields. Never merge the two.
- Sharing tiers (e.g. close friends vs neighbors) instead of per-person field toggles.
