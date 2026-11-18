# Curbside — Brand and Logo Brief

The logo should look like it was bolted to a pole at the end of your street. Everything in the app is a road sign: street-name blades, house-number plaques, yard signs, a stop sign for delete. The logo is that world's sign for the app itself.

## The idea in one line

**A street sign that remembers who lives there.** The sign system says *place*. The plaque on a house says *people*. The logo should hold both.

## Palette

The tokens are the same in the app and on the web (`curbside-ios/Curbside/Theme/Palette.swift`, `curbside-frontend/src/styles.scss`).

| Role | Light | Dark | Use in the logo |
|---|---|---|---|
| **Sign green** | `#0D6B3C` | `#0F6E40` | Primary. The street-name blade. |
| **Sign white** | `#F5F5EE` | `#F2F3EA` | Lettering, inset border, number plaque. Warm, not pure white. |
| Sign ink | `#151716` | `#151716` | Numbers on white plaques. |
| Concrete (background) | `#EBE8E1` | — | Sun-bleached sidewalk. A light backdrop. |
| Asphalt (background) | — | `#161918` | Night street. A dark backdrop. |
| Warning yellow | `#F2C12A` | `#F2C12A` | Accent only: a "NEW" diamond. |
| Work-zone orange | `#E9711A` | `#F08232` | Accent only. |
| Stop red | `#C4302A` | `#C4302A` | Delete only. Keep it out of the logo. |
| Info blue | `#1F4FA3` | `#3A6FD2` | Notes sign. Accent only. |
| School-zone lime | `#C7DD33` | `#C7DD33` | Kids pentagon. Accent only. |

The logo is **green + warm white**, with at most one accent. Don't make it a rainbow of every sign colour.

## Type

**Overpass** (free, Google Fonts, SIL OFL) is the open-source cut of the US highway sign face (Highway Gothic). The app uses:

- **Overpass ExtraBold**: street names, headings
- **Overpass Black**: small caps labels ("ADULTS", "NEW")
- **Overpass Mono Bold**: house numbers

A wordmark should be set in Overpass ExtraBold or Black, title case ("Curbside"), the way a street blade is lettered. Not all caps: real street blades in the US are mixed case now, and mixed case reads faster.

## The sign vocabulary you can borrow from

- **Street-name blade:** a green rectangle with rounded corners (about 8 px at app size), a thin white rule inset about 2 px from the edge, white lettering. A hard drop shadow down and right (no blur), like a sign standing proud of its post.
- **House-number plaque:** a small white tab with a dark mono number, fused to the left end of a green blade ("11 | PUMPERNICKEL"). This is the app's signature element, and you see it on every house on the map.
- **Yard signs:** a yellow diamond (warning, "NEW"), an orange diamond (work zone), a yellow pennant (event), and a white panel (a settled fact).
- **Stop octagon:** red, used only for delete.
- **The sign post:** a grey pole. Many signs in the mockup imply one.

Reference: `xom-research/docs/features/curbside/mockups/curbside.html` (open it in a browser). It's the design you picked.

## Logo directions

Pick one, or mix them:

1. **The corner blade.** Two street blades crossed at an angle on a post, like an intersection sign. One reads "Curbside", the other is blank or carries a house number. Strongest "this is a street" read. The hard part is fitting it into a square icon: crop to the crossing point.
2. **The plaque.** The house-number plaque alone: a white number tab fused to a short green blade. Use a number with meaning (your own house number, or "1"), or a single "C" on the tab. Most ownable, because it's the thing users tap all day. Works at 29 px.
3. **The blade C.** A green blade with the white inset rule and a single heavy Overpass "C". This is today's placeholder favicon. Clean and legible, but the least distinctive.
4. **Curb + pin.** A curb profile (the step from sidewalk to street) as a horizontal line, with a plaque or a tiny house sitting on it. More illustrative. Risk: it starts to look like a generic maps or real-estate app.

My pick is **2, the plaque**. It's literal to the theme, it's the app's own signature shape, and it survives the tiny sizes. Use **1** for the App Store marketing art and the web hero, where there's room.

## Don'ts

- No map pin or teardrop. Every maps and real-estate app uses one.
- No house-with-roof outline as the main mark; it reads "realtor".
- No emoji, no gradients that fight the flat sign paint, no glossy 3D.
- No red in the logo; red means delete in the app.
- Don't copy a real city's sign layout or a real street name. Signs are a common language; a specific city's design is someone else's.

## Deliverables

| Asset | Spec | Where it goes |
|---|---|---|
| **iOS app icon** | 1024×1024 PNG, square, **no transparency**, no rounded corners (iOS masks it). Keep important content inside the centre ~80%. | `curbside-ios/Curbside/Resources/Assets.xcassets/AppIcon.appiconset` (empty now; App Store upload fails without it) |
| iOS dark icon | 1024×1024 PNG, artwork on a **transparent** background (iOS supplies the dark backdrop) | same set, "dark" appearance |
| iOS tinted icon | 1024×1024 PNG, greyscale artwork on transparent (iOS tints it) | same set, "tinted" appearance |
| Or one **Icon Composer** `.icon` file | Xcode 26's Icon Composer, layered, for Liquid Glass | replaces the three PNGs |
| **Web favicon** | SVG, square viewBox, readable at 16 px; one simple shape | `curbside-frontend/public/favicon.svg` (placeholder there now) |
| Web touch icon | 180×180 PNG, no transparency | `curbside-frontend/public/apple-touch-icon.png` |
| Web/social card | 1200×630 PNG | link previews for curbside.xomware.com |
| Wordmark | SVG, "Curbside" in Overpass ExtraBold on a green blade, plus a version with no blade | sign-in screens (web and iOS), App Store screenshots |

Check every mark at **29 px** (smallest iOS settings icon) and **16 px** (browser tab). If the shape stops reading there, simplify it.

## Hand-off

Drop the files in a folder (for example `~/Desktop/curbside-logos/`) and tell me the path. Pasted images don't reach me as files. I'll wire the icon sets, favicon and wordmark into both apps, and ship them.
