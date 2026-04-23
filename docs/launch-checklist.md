# Drifting — Launch Checklist

Tracks everything needed to submit to the App Store. In-app features are tracked in the design spec; this document covers the submission process only.

---

## App Store Connect Setup

- [x] Create app in ASC — app name "Drifting Radio", Bundle ID `com.vitorfreitas.drifting`, category: Music
- [x] Create IAP product — ID `com.vitorfreitas.drifting.plus`, non-consumable, **Tier 5 ($4.99)**, display name and description added
- [x] Fill out age rating questionnaire (4+)
- [x] Fill out Privacy Nutrition Label — Data Not Collected
- [x] Enable Mac availability (Universal Purchase via Mac Catalyst)
- [x] DSA compliance — selected "not a trader"
- [x] Tax and banking setup (W-8BEN filed, bank account added)

## Code

- [x] Add `ITSAppUsesNonExemptEncryption = NO` to Info.plist (export compliance)
- [x] Add `PrivacyInfo.xcprivacy` privacy manifest (UserDefaults declared with CA92.1 + 1C8F.1)
- [x] Set `CFBundleDisplayName` and `CFBundleName` to "Drifting" (fixes home screen label and Mac menu bar)

## App Metadata

- [ ] Subtitle (≤30 chars): "Radio discovery, visualized"
- [ ] Description (drafted — see submission plan)
- [ ] Keywords (97 chars): `internet radio,music discovery,radio stations,visualizer,world radio,radio player,streaming,genre`
- [ ] Support URL
- [ ] What's New text: "First release."
- [ ] Marketing URL (optional — skip for v1)

## Screenshots

Required: iPhone 6.7" (iPhone 17 Pro Max simulator). Optional: 6.1", iPad.

- [ ] Enable "Always Show Station Info" in Settings before capturing
- [ ] Unlock Plus via StoreKit sandbox before capturing screenshots 3 (Settings app → Developer → StoreKit)
- [ ] Screenshot 1 — Visualization playing with info overlay visible (default Drift theme, free experience)
- [ ] Screenshot 2 — Favorites shelf open (save 2–3 stations first, then swipe up)
- [ ] Screenshot 3 — Visualization with a Plus theme (Aurora or Neon — visually striking)
- [ ] Screenshot 4 — Settings Plus upgrade card (feature list + price visible)
- [ ] Screenshot 5 — Widget on home screen *(optional, real device)*
- [ ] Upload screenshots to ASC (iPhone 6.7" slot)
- [ ] Attach one screenshot to IAP review section in ASC

## Build & Testing

- [ ] Archive build in Xcode (Product → Archive, destination: Any iOS Device arm64)
- [ ] Upload to App Store Connect via Organizer
- [ ] Add yourself as TestFlight internal tester and install on real device
- [ ] Verify app icon renders correctly on home screen
- [ ] Verify IAP flow end-to-end in StoreKit sandbox (price appears, purchase succeeds, Plus unlocks, restore works)
- [ ] Test on at least two device sizes (small iPhone + large iPhone)
- [ ] Verify background audio and lock screen controls work
- [ ] Verify widget appears and updates correctly

## Submission

- [ ] Attach build to version 1.0 in ASC
- [ ] Enter all metadata (subtitle, description, keywords, support URL, What's New)
- [ ] Add review notes (see submission plan for exact text)
- [ ] Submit for review

## Post-Launch

- [ ] Monitor reviews and crash reports in ASC
- [ ] Respond to initial user feedback
- [ ] Update design spec with any behaviour changes from review feedback
