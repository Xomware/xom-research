# Curbside iOS — App Store privacy label, review notes, TestFlight

**Last updated**: 2026-11-17. Source for every answer is cited; anything not checked says so.

## What Dom has to do before the first TestFlight build

1. **Create the App Store Connect app record** for bundle ID `com.Xomware.Curbside` (name "Curbside", primary language English (U.S.), SKU e.g. `curbside-ios`). This makes the bundle ID permanent.
2. **Secrets**: none new. `testflight.yml` loads Infisical project `code`, env `prod`, folders `/shared` then `/curbside`. `/shared` already holds the four the workflow reads (checked 2026-11-17, names only):
   - `APPLE_TEAM_ID`
   - `ASC_API_ISSUER_ID`
   - `ASC_API_KEY_ID`
   - `ASC_API_KEY_P8` (raw PEM or base64; the workflow accepts both)

   The same key signs and uploads `xomfit-ios`. To give Curbside its own key instead, add those four names to `/curbside`; the later folder wins. The key needs the **Admin** role: cloud-managed distribution signing (`-allowProvisioningUpdates` with an API key) is refused for lower roles.
3. **Approve the first run** if GitHub holds it: new workflows that load secrets have been held for approval before. Claude doesn't approve these.
4. **Run it**: Actions → "TestFlight" → Run workflow. It is `workflow_dispatch` only until the first build has gone through; switching it to `push: main` is a one-line change after that.
5. **Publish the privacy policy** at `https://curbside.xomware.com/privacy` (linked from Settings). Draft below.
6. **TestFlight group**: create an internal group (Dom); internal testers need no beta review.

## App Store privacy label ("App Privacy" in App Store Connect)

**Tracking**: No. No analytics or ad SDK, no data shared with data brokers, nothing combined with third-party data (epic PLAN.md, "Privacy").

**Data collected** — every type below: *Linked to the user: Yes. Used for tracking: No. Purpose: App Functionality only.*

| ASC category → type | What it is in Curbside | Source |
|---|---|---|
| Contact Info → Name | The display name typed at onboarding (`POST /me`). | `lambdas/me_create` |
| User Content → Other User Content | Households, people, pets and notes the user writes about neighbours. Capture transcripts are sent for parsing (to the API, then Anthropic) and not stored by Curbside. | `lambdas/households_*`, `notes_create`, `capture_parse` docstring |
| Location → Precise Location | The place's center (from "Use my location" or a search) and each household's coordinate are stored. Live location for Near me stays on the device. | `lambdas/maps_create`, `NearMeModel` |
| Identifiers → User ID | Cognito `sub` per sign-in, Curbside `account_id`. | `lambdas/common/auth.py` |

**Not collected** (answer "No" for these):
- Contact Info → Email Address: Cognito requests `openid` only and maps nothing but `sub` (`curbside-infrastructure/terraform/cognito_idps.tf`, `cognito.tf`). Apple's relay email never reaches the backend.
- Audio Data: speech is turned into text on the device by `SpeechAnalyzer`; audio is never sent anywhere.
- Diagnostics, Usage Data, Contacts, Photos, Health, Financial, Browsing/Search History: none.
- Coarse Location: not separately; the stored coordinates are precise.

**Note on Anthropic**: capture transcripts plus the names of up to 9 nearby households go to the Anthropic API for one parse. Under Anthropic's commercial API terms, inputs may be retained for a limited period for trust and safety; they are not used for training. This is why "Other User Content" is declared even though Curbside itself stores no transcript. **Unverified**: the exact retention window for Dom's API org; check the Anthropic console's data-retention setting before submitting.

## Review notes (paste into "App Review Information → Notes")

> Curbside is a private address book for your street: who lives in which house, their kids and pets, and notes from conversations.
>
> Sign-in: "Sign in with Apple" and "Continue with Google" are both offered on the first screen (Guideline 4.8). Both run through our Amazon Cognito user pool in a web authentication sheet; please use your own Apple ID.
>
> To try it: after sign-in, pick any display name, then on "New place" search for any address (or allow location) and tap "Make the map". Tap a building outline, or long-press anywhere, then "+ ADD" to add a household. The red TALK sign opens capture: type or say a sentence such as "Talked to Dana at the blue house, her son Theo just turned 8", tap "Read it", review the proposed changes and save.
>
> Voice is transcribed on the device (SpeechAnalyzer); audio never leaves the phone. The text is sent to our server and parsed with Anthropic's Claude API, and is not stored.
>
> Account deletion (5.1.1(v)): Map menu (…) → Settings → Delete account. It asks you to sign in again (with Apple if Apple is linked), then deletes the account, every map it owns, and the Cognito users, and revokes the Sign in with Apple token. If revoking fails, the app shows Apple's page for stopping Sign in with Apple manually.
>
> Location is when-in-use only, to center the map and to list the houses you're walking past. There is no background mode, no tracking, no ads, and no analytics.

**Contact / demo account**: none needed; reviewers use their own Apple ID. If Review asks for one anyway, a Google test account would need creating; none exists today.

## Export compliance
`ITSAppUsesNonExemptEncryption = NO` is already set (`project.yml`). The app uses only HTTPS via the OS.

## Privacy policy (draft for curbside.xomware.com/privacy)

> **What Curbside stores.** Your display name; the places you map; and the households, people, pets and notes you add. Each sign-in you link is stored as an opaque ID. We don't receive your email address.
>
> **Who can read it.** You. Curbside's server reads your data to show it to you, to export it and to parse captures. The operator can technically read the database and does not, except to fix a problem you report. Data is encrypted at rest with a key whose every use is logged.
>
> **Capture.** Speech is turned into text on your phone. When you tap "Read it", the text and the names of up to nine nearby households are sent to Anthropic's API to propose changes. Curbside doesn't store the text. Anthropic may keep API inputs for a limited time for safety review and does not train on them.
>
> **Maps.** Building outlines come from OpenStreetMap via the Overpass API, which sees your place's coordinates. Apple Maps draws the map and looks up addresses.
>
> **No tracking.** No ads, no analytics, no data sold or shared for marketing.
>
> **Your controls.** Settings → Export downloads everything as JSON. Settings → Delete account removes it all, including your sign-ins, and revokes Sign in with Apple.

## TestFlight workflow (`curbside-ios/.github/workflows/testflight.yml`)
Archive (`Release`, cloud signing via the API key) → export (`app-store-connect`) → `altool --upload-app`. Version = `MARKETING_VERSION` from `project.yml`; build number = UTC timestamp + run id suffix, so reruns never collide (ITMS-90189). Not run yet: it can't succeed until step 1 above is done.
