# Research: Curbside gate 0a verification spike

**Date**: 2026-11-16
**Decision it informs**: `PLAN.md` Phase 0 gate 0a. It settles model choice, auth/deletion design, the footprint licensing posture, and the MapKit JS token approach before I1/B1 start
**Status**: Complete for desk research. OSM coverage on Dom's street is **unknown** until Dom runs the check in section 9. The Overture DuckDB timing was not run because DuckDB isn't installed and the spike wasn't needed yet.

Every fact cites a URL, and all sources were checked 2026-11-16. Anything marked **unknown** could not be confirmed from a primary source. Anything marked *inference* is my reading, not stated by the source.

## Question
Do the plan's recalled prices, limits, licences and App Store rules hold as of 2026-11-16, and do any of them change the design?

## Assumptions the plan got wrong

1. **The model is stale.** `claude-haiku-5-5` shipped 2026-10-07 at $0.10/$0.50 per MTok, about 8x cheaper than Haiku 4.5. Haiku 4.5's minimum cacheable prompt is 4,096 tokens, so the plan's ~1k-token "cached" system prompt would never have been cached on 4.5.
2. **The ODbL framing is wrong.** Serving footprint GeoJSON to app users is conveying a database (Derivative Database), not a Produced Work. The obligations are easy to meet but different (section 7).
3. **`AdminDeleteUser` alone doesn't meet Apple's revocation guidance.** There is a workable route: map Apple's access token through Cognito and call `/auth/revoke`, with Apple's own documented manual fallback (section 6).
4. **`CLGeocoder` is deprecated in iOS 26.** S5 should use MapKit's `MKReverseGeocodingRequest` (section 2).
5. **Overpass policy steers app backends away from the public instance.** Curbside's volume is tiny, but the plan should record that as a known policy tension (section 5).

Confirmed as written: MapKit JS quotas, the Cognito 10k free MAU for social sign-in, and a static origin-restricted MapKit JS token being an officially supported option.

---

## 1. MapKit JS: pricing, quotas, token types

| Fact | Source |
|---|---|
| Free daily limit of **250,000 map views and 25,000 service calls** per Apple Developer Program membership. Snapshots are 25,000 unique requests/day. Above that, "contact Apple". No paid price list is published | https://developer.apple.com/maps/web/ |
| The 25,000 service calls/day are **shared per team between Maps Server API and MapKit JS**. Over quota returns HTTP 429 | https://developer.apple.com/documentation/applemapsserverapi (doc JSON) |
| **Token type A, dashboard token**: created in the developer account under Maps → Tokens, token type "MapKit JS". Restriction type is **Domain** or **None**. Domain tokens have a validity of **No Expiration, 30, 90 or 180 days, or a custom date**. Revocable from the dashboard | https://developer.apple.com/documentation/mapkitjs/creating-a-maps-token ; https://developer.apple.com/help/account/service-configurations/maps-tokens |
| **Token type B, self-signed JWT**: signed with your Maps private key, `scope` = `mapkit_js`. An `origin` claim is **required** when scope includes `mapkit_js`. Apple recommends one scope per token | https://developer.apple.com/documentation/applemapsserverapi/creating-and-using-tokens-with-maps-server-api |
| Creating and revoking dashboard tokens needs the Account Holder or Admin role | https://developer.apple.com/help/account/service-configurations/maps-tokens |

**Settles the Open Question.** A domain-restricted, no-expiration dashboard token is a first-class Apple option, so I9's "static token in Infisical → build env" works with no backend change.
- Tradeoff (*inference*): the token ships in the JS bundle. The domain restriction is enforced through the browser `Origin`, which a non-browser client can forge. The worst case is someone burning the shared 25k/day service-call quota, which iOS doesn't use (see 2). Mitigation: revoke and reissue from the dashboard. The minting endpoint can wait until abuse shows up.

## 2. MapKit on iOS: cost and limits

| Fact | Source |
|---|---|
| Apple publishes **no quota or price for native MapKit**. The developer Maps page lists none | https://developer.apple.com/maps/ |
| An Apple DTS engineer on the forums says native MapKit has no cost beyond Developer Program membership. This is a forum answer, not licence text, so it is unverified as a contractual guarantee | https://developer.apple.com/forums/thread/654501 ; https://developer.apple.com/forums/thread/743284 |
| Geocoding is **rate-limited per app**. Exceeding the rate returns a network error, and Apple's rule of thumb is at most one geocoding request per user action | https://developer.apple.com/documentation/corelocation/clgeocoder |
| **`CLGeocoder` is deprecated as of iOS 26.0 ("Use MapKit")**. The replacement is `MKReverseGeocodingRequest`, introduced in iOS 26.0 | https://developer.apple.com/documentation/corelocation/clgeocoder ; https://developer.apple.com/documentation/mapkit/mkreversegeocodingrequest |

Plan impact: S5 and S6 should use `MKReverseGeocodingRequest` instead of `CLGeocoder`, provided the deployment target is iOS 26+. If the target is lower, the deployment target is **unknown** and S1 must decide it. Reverse-geocode only on the long-press pin drop, never in a loop.

## 3. Amazon Cognito: tiers and federated MAU pricing

| Fact | Source |
|---|---|
| Three tiers: **Lite, Essentials, Plus**. **Essentials is the default** for new pools when `UserPoolTier` is unset | https://aws.amazon.com/cognito/pricing/ ; https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-sign-in-feature-plans.html |
| Users who sign in "directly … or through a **social identity provider**" get a **10,000 MAU/month free tier** per account or org on Lite or Essentials. Plus has no free tier | https://aws.amazon.com/cognito/pricing/ |
| Above the free tier: Essentials **$0.015/MAU**, Plus **$0.020/MAU**, Lite tiered (the page's example uses $0.0055, then $0.0046) | https://aws.amazon.com/cognito/pricing/ |
| Users federated through **SAML or OIDC** IdPs are billed separately: **50 MAU free**, then $0.015 | https://aws.amazon.com/cognito/pricing/ |
| Social, SAML and OIDC sign-in plus the classic hosted UI are on **all tiers, Lite included**. Managed login, access-token customization, passkeys and email MFA need Essentials | https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-sign-in-feature-plans.html |

Plan impact:
- Apple and Google are social IdPs, so Curbside sits in the 10k free bucket. Precedent `today-in-sports-infrastructure/terraform/cognito.tf` configures Google as `provider_type = "Google"` (line 190), which is correct. Keep it that way. Re-declaring Google as a generic `OIDC` provider would move it to the 50-MAU free bucket (*inference* from the pricing split).
- The plan's flow skips the hosted chooser, so Lite is enough. Either tier costs $0 at personal scale. Set `user_pool_tier` explicitly in I3 instead of inheriting the Essentials default.

## 4. Claude Haiku: model ID, price, cost per voice dump

| Fact | Source |
|---|---|
| **`claude-haiku-5-5`**, released **2026-10-07**, is the current Haiku. Retirement is not sooner than 2027-10-07 | https://platform.claude.com/docs/en/models/haiku-5-5/overview |
| Haiku 5.5 prices for prompts ≤100k tokens: **$0.10 input, $0.50 output, $0.125 5-min cache write, $0.20 1-h cache write, $0.01 cache read** per MTok | https://platform.claude.com/docs/en/about-claude/pricing |
| Haiku 4.5 (`claude-haiku-4-5-20251001`) prices: **$1 input, $5 output, $1.25 5-min write, $0.10 cache read** per MTok | https://platform.claude.com/docs/en/about-claude/pricing ; https://platform.claude.com/docs/en/models/overview |
| Haiku 5.5 uses the newer tokenizer, so the same text is **~30% more tokens** than on Haiku 4.5 | https://platform.claude.com/docs/en/models/haiku-5-5/overview |
| Minimum cacheable prompt: **512 tokens on Haiku 5.5, 4,096 on Haiku 4.5**. Shorter prompts silently don't cache. The default TTL is 5 min | https://platform.claude.com/docs/en/build-with-claude/prompt-caching |
| Forced tool use (`tool_choice` any/tool) adds **406 system tokens on Haiku 5.5** and **588 on Haiku 4.5** | https://platform.claude.com/docs/en/about-claude/pricing |
| Haiku 5.5 has adaptive thinking on by default. Thinking tokens bill as output. **Forced tool use is supported, and "the response starts with the tool call and has no `thinking` block"**. `thinking: {type: "disabled"}` is accepted at effort `high` or below | https://platform.claude.com/docs/en/build-with-claude/thinking |
| Structured outputs and strict tool use are supported on both Haiku 4.5 and 5.5 | https://platform.claude.com/docs/en/build-with-claude/structured-outputs |

**Cost per voice dump.** Assumes 300 transcript tokens + 1,000 system-prompt tokens in and 400 out, measured on the Haiku 4.5 tokenizer. Haiku 5.5 is scaled ×1.3, and no caching is applied.

| | Haiku 4.5 | Haiku 5.5 |
|---|---|---|
| Brief as stated (1,300 in / 400 out) | $0.00130 + $0.00200 = **$0.0033** | 1,690 × $0.10 + 520 × $0.50 per M = **$0.00043** |
| Plus forced-tool overhead (588 / 406 tokens) | **$0.0039** | **$0.00047** |
| 100 dumps/month | ~$0.39 | ~$0.05 |
| Quota ceiling, 50/day × 30 | ~$5.85 | ~$0.70 |

The tool JSON schema's own token count is **unknown** and not included. Measure it with the token-counting endpoint in B9.

Caching isn't worth it on either model.
- Haiku 4.5 can't cache a ~1k prompt at all.
- On Haiku 5.5 a cache write costs 1.25× base. Dumps that are minutes or days apart mostly miss the 5-min TTL, so most calls would pay the write premium.
- The best-case saving is ~$0.00013 per dump (*inference* from the prices above).

Recommendation: use `claude-haiku-5-5` with a forced tool and no `cache_control`, and run the 15-transcript eval to confirm quality before B9 merges. The plan's cost line is about 7× too high, and its caching note was wrong for 4.5.

## 5. Overpass API usage policy

| Fact | Source |
|---|---|
| Safe one-off use on `overpass-api.de` is < 10,000 queries/day **and** < 1 GB/day **and** < 10 min total processing. **Regular use: divide by 100**, i.e. < 100 queries and < 10 MB per day. "If you have an app or website, then the usage counts towards the sum of requests made by all your users" | https://wiki.openstreetmap.org/wiki/Overpass_API (raw wiki text) |
| Send a `User-Agent` or `Referer` that uniquely identifies the app. No parallel scripts. **"Commercial use should use self-hosted or paid Overpass servers."** On 429/406, pause 30 s. The main server is "overloaded … do not expect high reliability" | https://wiki.openstreetmap.org/wiki/Overpass_API |
| Rate limiting is slot-based per IP. Requests queue up to 15 s, then get 429 (rate) or 504 (resources). The operator lists "setting up an app for more than just OSM mappers and relying on the public instances as backend" as problematic | https://dev.overpass-api.de/overpass-doc/en/preface/commons.html |
| Live status from a no-location `/api/status` call on 2026-11-16 showed "Rate limit: 2" slots per IP | https://overpass-api.de/api/status |
| Alternatives: private.coffee (no rate limit, notify for large projects), dresel.systems North America (4 parallel per IP, asks for an identifying UA), and paid options (Geofabrik, Overpass.ch free to 10k/month) | https://wiki.openstreetmap.org/wiki/Overpass_API |

Plan impact (*inference*): one ~300 m query per place creation or refresh is orders of magnitude under the regular-use line. That holds for Dom alone and for a few hundred users. Two things to write down:
- **Commercial status is unknown.** The policy is explicit that apps backed by the public instance are discouraged. If Curbside ever charges money or grows past personal scale, move to a paid instance or a self-hosted extract.
- B7 should send `User-Agent: curbside/<version> (+https://curbside.xomware.com)`, honour 429 with a 30 s backoff instead of an immediate fallback hit, and pick the fallback instance by region (dresel.systems NA for US addresses).

## 6. Sign in with Apple account deletion and token revocation

| Fact | Source |
|---|---|
| Apps with account creation must offer in-app deletion that removes the account and associated data. Deactivation alone is insufficient. No support-ticket-only flows | https://developer.apple.com/support/offering-account-deletion-in-your-app/ |
| "Apps that support Sign in with Apple **should use the Sign in with Apple REST API to revoke user tokens**" | https://developer.apple.com/support/offering-account-deletion-in-your-app/ |
| `/auth/revoke` needs `client_id` (App ID or Services ID used at authorization), a `client_secret` JWT signed with the SIWA private key, a **refresh token or access token**, and `token_type_hint`. It returns 200 even if the token was already invalid | https://developer.apple.com/documentation/signinwithapplerestapi/revoke-tokens |
| Apple's token response example shows `expires_in: 3600` for the access token | https://developer.apple.com/documentation/signinwithapplerestapi/generate-and-validate-tokens |
| **TN3194**: `/auth/revoke` is "the only way to programmatically invalidate user tokens". **"If you don't have the user's refresh token, access token, or authorization code, you must still fulfill the user's account deletion request"**. In that case: (1) delete the user's data, (2) direct the user to revoke manually at https://support.apple.com/en-us/102571, (3) handle the credential-revoked notification | https://developer.apple.com/documentation/technotes/tn3194-handling-account-deletions-and-revoking-tokens-for-sign-in-with-apple |
| Cognito can **map an IdP's access token into a custom user attribute**. "Social providers present an access token". The attribute must be mutable, writable by the app client, and ≤ 2,048 chars | https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-specifying-attribute-mapping.html |
| Cognito does not appear to revoke Apple tokens on user deletion. An AWS re:Post thread shows users hitting exactly this gap with no AWS-side fix | https://repost.aws/questions/QU1Dh6o7ASS5SNJvLYlG_SqA/revoke-cognito-federated-login-apple-token |

**Settles the Open Question.** `AdminDeleteUser` alone doesn't follow Apple's guidance. We do need our own client secret, but we already hold its inputs: the Cognito Apple IdP config takes the same team ID, key ID and `.p8`. Recommended shape (*inference*, built from the facts above):
1. **I3**: add a mutable custom attribute `custom:apple_at` (2,048 chars) and map the Apple IdP's `access_token` → `custom:apple_at`. Make it writable by the iOS client but **left out of the client's read attributes**, so it never appears in ID tokens.
2. **S10 delete flow**: Apple access tokens live about an hour, so the delete button first runs a fresh "Sign in with Apple to confirm" PKCE round. That makes Cognito rewrite the attribute with a live token.
3. **B12 `me_delete`**: `AdminGetUser` → read `custom:apple_at` → POST `appleid.apple.com/auth/revoke` with the Services ID and an ES256 client-secret JWT (key from SSM) → delete data → `AdminDeleteUser`.
4. **Fallback** if the token is missing or revoke fails: finish the deletion anyway and show TN3194's manual-revoke instructions with the support.apple.com link. Apple documents this as meeting the requirement.

**Unknown, test in I3:** whether Cognito actually populates `access_token` for the `SignInWithApple` provider type. The doc says "social providers" generically, and Apple's access token length isn't documented. If it doesn't populate, steps 1–3 drop out and the fallback becomes the path. That still complies per TN3194, with a worse UX.

## 7. ODbL obligations for serving OSM building footprints

| Fact | Source |
|---|---|
| **Derivative Database** = "a database based upon the Database, and includes any translation, adaptation, arrangement, modification", including extracting a substantial part into a new database | https://opendatacommons.org/licenses/odbl/1-0/ |
| **Produced Work** = "a work (such as an image, audiovisual material, text, or sounds)" resulting from using the database | https://opendatacommons.org/licenses/odbl/1-0/ |
| **Publicly** = to persons other than you or entities under your control | https://opendatacommons.org/licenses/odbl/1-0/ |
| OSMF test: output "intended for the extraction of the original data" is a database, not a Produced Work. Usually Produced Works: PNG/JPG/PDF/SVG, printed maps | https://osmfoundation.org/wiki/Licence/Community_Guidelines/Produced_Work_-_Guideline |
| §4.3: publicly using a Produced Work requires a notice that it came from the database under ODbL. §4.4: a publicly used Derivative Database must stay ODbL. **§4.6: if you publicly use a Derivative Database, offer it, or "a file containing all of the alterations … or the method of making the alterations"**, free over the internet | https://opendatacommons.org/licenses/odbl/1-0/ |
| **Collective Database guideline**: non-OSM data that references OSM elements "by a database key or any other method of identifying a specific OSM … element" stays independent. Example: "Your phone numbers are not subject to share-alike" | https://osmfoundation.org/wiki/Licence/Community_Guidelines/Collective_Database_Guideline_Guideline |
| **Attribution for apps**: credit "OpenStreetMap" with a link to openstreetmap.org/copyright. "© OpenStreetMap contributors" is acceptable. Place it in a map corner or adjacent to the map. It may collapse after interaction or 5 s if an "(i)"/About path stays reachable. Other logos (e.g. Apple's) must not imply the OSM data isn't from OSM | https://osmfoundation.org/wiki/Licence/Attribution_Guidelines |

How this applies to Curbside (*inference*, not legal advice):
- **The per-map GeoJSON is a Derivative Database.** It is OSM geometry filtered, coordinate-rounded and enriched with `area_m2`/`accessory`, then sent to the user's device as data. It is not a Produced Work, so the plan's line 132 framing is wrong.
- **§4.6 is met by the public repo.** `curbside-backend` is public per the visibility convention, and `footprints_fetch` *is* "the method of making the alterations". Link to it from the in-app "Data licences" screen. Keep the GeoJSON itself under ODbL and never claim other terms for it.
- **Households, people, notes and pets are a separate non-OSM dataset** that references OSM ways by `building_id`. That is a Collective Database, so share-alike does not reach them. Household lat/lng and address come from Apple geocoding, not OSM, which keeps the layers cleanly apart.
- **The rendered map on screen is a Produced Work.** It needs "© OpenStreetMap contributors" linked to /copyright, near the map, alongside Apple's own legal label. S5 and W2 already include it.

## 8. Overture Maps buildings

| Fact | Source |
|---|---|
| Buildings theme licence: **ODbL**. Sources include OSM (ODbL), Microsoft Global ML Building Footprints (ODbL), Esri Community Maps (CC BY 4.0), Google Open Buildings (CC BY 4.0) and others. Optional citation: "Overture Maps Foundation, overturemaps.org" | https://docs.overturemaps.org/attribution/ |
| DuckDB ≥ 1.1.0 with `spatial` + `httpfs` and `SET s3_region='us-west-2'`, reading `s3://overturemaps-us-west-2/release/<release>/theme=buildings/type=building/*` with `hive_partitioning=1` and filtering on `bbox.xmin/ymin` | https://docs.overturemaps.org/getting-data/duckdb/ |
| Monthly releases. Latest is `2026-10-21.0`, next `2026-11-18.0`. **Public releases are kept on S3 for at most 60 days**, then removed | https://docs.overturemaps.org/release-calendar/ |

Plan impact, if B8b happens:
- Do not hard-code a release path. It disappears within two months, so resolve the latest release at fetch time.
- Overture's buildings are still ODbL, so section 7 applies unchanged, with Overture plus each upstream credited.
- **Lambda query latency for a ~300 m bbox: unknown.** Not timed. DuckDB isn't installed locally, and a laptop timing wouldn't represent Lambda in us-east-1 reading a us-west-2 bucket anyway.
- **GERS ID stability across releases: unknown.** The release calendar doesn't cover it.

## 9. OSM coverage check (Dom runs this, no location sent by research)

No real address was sent anywhere during this research. The command below was tested only against a public landmark. Fill in the two `# REPLACE:` lines and run:

```bash
LAT=   # REPLACE: latitude of the middle of your street (Apple Maps → drop pin → copy coordinates)
LON=   # REPLACE: longitude
R=300

curl -s -A "curbside-coverage-check/0.1" \
  --data-urlencode "data=[out:json][timeout:25];(way[\"building\"](around:$R,$LAT,$LON);relation[\"building\"](around:$R,$LAT,$LON););out tags;" \
  https://overpass-api.de/api/interpreter |
python3 -c '
import json, sys, collections
els = json.load(sys.stdin)["elements"]
kinds = collections.Counter(e["tags"].get("building", "?") for e in els)
accessory = sum(kinds[k] for k in ("garage", "garages", "shed", "carport"))
addr = sum("addr:housenumber" in e["tags"] for e in els)
print(f"buildings within 300 m: {len(els)}")
print(f"  accessory (garage/shed/carport): {accessory}")
print(f"  with a house number tag: {addr}")
print("  by building=*:", ", ".join(f"{k} {n}" for k, n in kinds.most_common(8)))
'
```

What to look for:
- **Count against reality.** In a 300 m radius of detached suburban housing, expect roughly 60–150 primary dwellings (*inference*, depends on lot size). Count the houses on your own block from Apple Maps satellite, then check that the polygons cover them. To eyeball the polygons, paste the same query with `out geom;` into https://overpass-turbo.eu.
- **Pass:** ≥ 90% of primary dwellings on the blocks you care about have a polygon. Missing garages and sheds are fine. `building=yes` dominating is fine. OSM imports often don't tag the subtype, and the app doesn't need it.
- **Fail → B8b into MVP:** the count is clearly below the visible houses, or whole blocks are empty. US suburbs without a building import often look like this.
- **House-number count is informational only.** Households take addresses from Apple reverse geocoding, so low `addr:housenumber` coverage doesn't matter.
- Run the two other neighbourhood types (rowhouse, apartment) the same way. Expect fewer polygons than households there. That is the known "footprint ≠ household" case, handled by the list-on-tap.

## 10. Anthropic API data retention

| Fact | Source |
|---|---|
| Privacy center (commercial API): "we automatically delete inputs and outputs on our backend **within 30 days** of receipt or generation". Exceptions: ZDR agreements, Usage Policy enforcement (flagged content **up to 2 years**, classifier scores up to 7 years), legal requirements, and feedback data (5 years) | https://privacy.claude.com/en/articles/7996866-how-long-do-you-store-my-organization-s-data |
| Platform docs: "Conversation content (your prompts and Claude's outputs) **is not retained by default**; the exception is Covered Models, which require 30-day retention". Covered Models are the Fable and Mythos lines. **Haiku is not one.** "Retained data is never used for model training without your express permission" | https://platform.claude.com/docs/en/manage-claude/api-and-data-retention |
| ZDR: "To request ZDR for your organization, contact the Anthropic sales team". Enabled **per organization**, "subject to Anthropic's approval". It covers the Messages API. Prompt caching and structured outputs/strict tools are eligible, and the JSON schema itself is cached up to 24 h. The Batch API is **not** eligible (29-day retention) | https://platform.claude.com/docs/en/manage-claude/api-and-data-retention ; https://privacy.claude.com/en/articles/8956058-i-have-a-zero-data-retention-agreement-with-anthropic-what-products-does-it-apply-to |
| A third-party blog claims a 7-day API retention since 2025-09-14. **No Anthropic page confirms it**, so it is unverified | https://char.com/blog/anthropic-data-retention-policy |

The two Anthropic pages differ: the privacy center says ≤ 30 days, the platform docs say not retained by default for non-Covered models. The privacy policy should state the conservative version: "Anthropic processes the transcript to parse it and deletes it within 30 days; it is not used for training". Link both pages.

**Unknown:**
- Whether ZDR is available to Dom's account. It is sales-gated per organization with approval, and nothing published says individual or self-serve orgs qualify or don't.
- Which terms govern Dom's API key. Per memory, the key runs on Max-plan monthly credits since 2026-10-07. If that key belongs to a Commercial Terms org, the above applies. If it is consumer-governed, the consumer retention and training settings apply instead (https://privacy.claude.com/en/articles/10023548-how-long-do-you-store-my-data, not reviewed here). Check Console → Settings → Privacy → Data retention, and confirm the org is under Commercial Terms, before any real neighbour data goes through it.

Don't use the Batch API for capture. It isn't ZDR-eligible and retains data for 29 days.

---

## Recommendation
- Switch B9 to `claude-haiku-5-5` with forced tool use and no prompt caching. Expect ~$0.0005 per dump. The eval set gates it.
- Keep the static domain-restricted MapKit JS token for I9.
- Keep Cognito as planned, with an explicit `user_pool_tier` and Google as `provider_type = "Google"`.
- I3 adds the `custom:apple_at` mapping, and B12 calls `/auth/revoke` with the TN3194 manual fallback. Test the mapping first.
- Replace the plan's "Produced Work" line with the section 7 posture: GeoJSON is an ODbL Derivative Database, satisfied by the public `footprints_fetch` source and a "Data licences" screen. Annotations are a Collective Database.
- Use `MKReverseGeocodingRequest` instead of `CLGeocoder`.
- B8b stays conditional on Dom's run of section 9.

## Open Questions
- [ ] OSM coverage on Dom's street, plus the rowhouse and apartment samples (Dom runs section 9). Decides B8b.
- [ ] Does Cognito populate `access_token` for the `SignInWithApple` IdP into a mapped custom attribute? Test in I3.
- [ ] Is ZDR obtainable for Dom's org, and is the API key under Commercial Terms or consumer terms?
- [ ] Tool-schema token count for B9. Measure with the token-counting endpoint.
- [ ] iOS deployment target. Decides whether `MKReverseGeocodingRequest` (iOS 26+) is usable without a fallback.
- [ ] If B8b happens: DuckDB bbox latency from Lambda, and GERS ID stability across monthly releases.
