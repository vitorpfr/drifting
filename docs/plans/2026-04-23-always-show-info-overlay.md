# Always Show Info Overlay — Design Spec

**Date:** 2026-04-23
**Status:** Approved

## Summary

Add a free toggle in Settings → Playback that keeps the info overlay permanently visible on iOS, mirroring the existing Mac Catalyst behavior.

## Behavior

- Default: `false` (existing auto-dismiss behavior unchanged)
- When enabled: overlay stays visible at all times while a station is playing, regardless of the dismiss timer
- The dismiss timer logic in `showOverlay()` is untouched — it still fires, it just has no visible effect when the setting is on
- Mac Catalyst is unaffected (already always-on via `#if targetEnvironment(macCatalyst)`)

## Settings

- **Section:** Playback
- **Label:** Always Show Station Info
- **Subtitle:** Keep the station name, track, and controls visible at all times.
- **Type:** Toggle, free (no Plus required)

## Implementation

**AppState:** Add `@AppStorage("alwaysShowInfoOverlay") var alwaysShowInfoOverlay: Bool = false`

**MainView:** Change the iOS branch of the show logic:
```swift
// before
let show = showInfoOverlay
// after
let show = showInfoOverlay || appState.alwaysShowInfoOverlay
```

**SettingsView:** Add toggle in the Playback section, below Audio Only.
