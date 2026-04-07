# Drifting — Launch Checklist

Tracks everything needed to submit to the App Store. In-app features are tracked in the design spec; this document covers the submission process only.

---

## App Store Connect Setup

- [ ] Create app in ASC — Bundle ID `com.vitorfreitas.drifting`, category: Music
- [ ] Create IAP product — ID `com.vitorfreitas.drifting.plus`, non-consumable, set price tier, add display name and description
- [ ] Fill out age rating questionnaire (expected: 4+)
- [ ] Fill out Privacy Nutrition Label — select "Data Not Collected" and complete the form
- [ ] Add `ITSAppUsesNonExemptEncryption = NO` to Info.plist (export compliance)

## App Metadata

- [ ] App name: "Drifting"
- [ ] Subtitle (≤30 chars) — e.g. "Radio discovery, visualized"
- [ ] Description
- [ ] Keywords (≤100 chars, comma-separated — critical for search ranking)
- [ ] Support URL
- [ ] Marketing URL (optional)
- [ ] What's New text (first release: can be brief)

## Screenshots

Required: at least iPhone 6.7" (15 Pro Max). Optional but recommended: 6.1", iPad.

- [ ] Capture 3–10 screenshots at 6.7" — suggested moments:
  - Visualization playing (show a vivid reactive state)
  - Info overlay open (station name, track, location, genre)
  - Favorites shelf
  - Settings — Plus section (shows feature list + price)
  - Widget on home screen (if you want to show it)
- [ ] Add optional text overlays / captions to screenshots (common practice, not required)

## Build & Testing

- [ ] Verify app icon renders correctly on a real device home screen
- [ ] TestFlight internal test build — install on real device
- [ ] Verify IAP flow end-to-end in StoreKit sandbox (price appears, purchase succeeds, Plus unlocks, restore works)
- [ ] Test on at least two device sizes (small iPhone + large iPhone)
- [ ] Verify background audio and lock screen controls work on device
- [ ] Verify widget appears and updates correctly on device

## Submission

- [ ] Upload build via Xcode → Organizer (or `xcodebuild archive`)
- [ ] Select build in ASC and attach to the version
- [ ] Add review notes: "This is a one-time tip IAP, not a subscription. The app plays live internet radio streams — an active internet connection is required to test."
- [ ] Submit for review

## Post-Launch

- [ ] Monitor reviews and crash reports in ASC
- [ ] Respond to initial user feedback
- [ ] Update design spec with any behaviour changes from review feedback
