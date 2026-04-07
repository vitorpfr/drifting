# Drifting — Product Design Spec

**Date:** 2026-04-07
**Status:** Active
**Supersedes:** design-spec.md (v1, 2026-03-22), design-spec-v2.md (2026-04-03)

---

## Implementation status key

> ✅ Implemented — **⚠️ Partially implemented** — ❌ Not yet implemented

---

## 1. Concept & Principles ✅

Drifting is a passive music discovery app. The user opens it, music starts. There are no menus, no search, no playlists — just a full-screen audio visualization and a single interaction model: swipe to navigate.

**Guiding principles:**
- **Zero friction to music** — audio starts within 2 seconds of opening the app on a good connection; visualization begins immediately regardless of stream state
- **The visualization IS the UI** — no controls, labels, or chrome on screen by default
- **Every gesture feels intentional** — swipes are weighted with haptics and animation so actions feel satisfying, not accidental
- **The app gets smarter silently** — preference learning happens in the background, never interrupting the experience
- **Local-first** — no account, no login, no barrier. Preferences live on device.

**What Drifting is NOT:**
- A music player (no playlists, no on-demand)
- A radio directory (no browsing or searching)
- A social app — social features are intentionally deferred, not excluded from the long-term vision

---

## 2. Platform Strategy ✅

iOS is the primary platform, built in native Swift to deliver the best possible haptic and audio experience. Web and Android are deferred to future versions.

| Platform | Tech | Priority |
|---|---|---|
| iOS | Native Swift | Current — primary |
| Web | React + Vite | Future |
| Android | Native Kotlin | Future |

**Haptics are iOS-only.** The web app has no haptic feedback.

---

## 3. Radio Station Catalog ✅

**Source:** Radio Browser API — open, community-maintained, ~30k stations worldwide.

**Quality filters applied at query time:**
- Minimum 64kbps bitrate
- Exclude stations with only non-music tags (news, talk, sports, religion); tagless stations and mixed-tag stations are allowed through
- Active streams only (last-checked alive)
- Prefer stations with genre and language metadata
- All languages and regions included

**Stream reliability — discovery flow:**
- On station selection, attempt to connect. Retry up to 3 times before advancing to the next candidate.
- Auto-advance after 3 failed retries is not counted as a recommendation signal.
- If a stream drops mid-session: listening time before the drop is discarded and no signal is recorded ✅. The station is deprioritized for the remainder of the session ✅.

**Stream reliability — saved stations:**
- When a user taps a saved station, retry up to 3 times. If the stream cannot be reached, an inline error indicator (⚠) appears on that saved station entry in Settings. ✅ Tapping the station again retries and clears the indicator.

**Handle connectivity issues ⚠️**
- "connecting" label appears during loading state (after wordmark fades out) ✅
- No-connection overlay (wifi.slash) shown when network is unavailable ✅
- Stall watchdog: AVPlayer stall detected via `timeControlStatus`; retries after 15s ✅
- Load timeout: 10s per attempt to catch hung connections ✅
- Analyzer watchdog: if visualization silent while playing, restarts analyzer stream every 3s ✅
- Analyzer/audio sync: frequency data gated on `playerState == .playing` ✅
- Tap to retry while idle ✅
- Auto-resume current station on network recovery ⚠️ — reconnection after wifi returns is partially working; edge cases remain

---

## 4. Stream Loading, Buffering & Transitions ✅

Perceived loading time is a core UX concern. The app uses pre-buffering and carefully timed audio/visual fades to make every transition feel smooth and intentional — never abrupt.

**App launch:**
- The station catalog is pre-cached locally after first install and refreshed silently in the background — no API call on launch, station selection is instant ✅
- The idle visualization (slow ambient pulse) plays immediately on launch, within ~0.3s ✅
- Stream buffering begins immediately in parallel with UI rendering ✅
- Target: audio begins within 2 seconds of launch on a good connection ✅
- Audio fades in from silence to full volume over ~1.5s ✅
- The visualization transitions from idle pulse to fully reactive mode on the same curve ✅

**Between stations — swipe left (next) or swipe right (previous):**

Three-act transition:
1. **Exit** — audio fades out over ~0.4s ✅ while the visualization sweeps in the swipe direction ✅
2. **Loading** — idle pulse plays while the pre-buffered next stream takes over ✅
3. **Enter** — new station's audio fades in over ~1s ✅ while the visualization morphs into the new station's color identity ✅

**Save (swipe down):**
No audio transition — music keeps playing uninterrupted. ✅ Full save burst animation (see Section 7). ✅

**Timing summary:**

| Moment | Audio | Visualization |
|---|---|---|
| Launch | Fade in over ~1.5s | Idle pulse → reactive, same curve |
| Next / Prev (exit) | Fade out over ~0.4s | Sweep in swipe direction ✅ |
| Next / Prev (enter) | Fade in over ~1s | Morph to new color identity |
| Save | No change | Radial burst, then continues |

**Pre-buffering:**
- While a station is playing, the app silently pre-selects and begins buffering the next candidate ✅
- On swipe left, the transition begins immediately using the pre-buffered stream ✅
- Swipe right plays back through a history stack; no pre-buffering needed ✅
- Pre-buffer has a 20% chance of picking a saved station, otherwise uses the full weighted taste profile for pre-selection ✅

---

## 5. Gesture System & Interaction Model ✅

The full-screen visualization is the only visible element by default. All interaction happens through gestures.

| Gesture | Action | Recommendation Signal |
|---|---|---|
| Swipe left | Next station | Negative — strength depends on listening duration (see Section 9) |
| Swipe right | Previous station (history stack) | Mild positive |
| Swipe down | Save / bookmark — station keeps playing | Strongest positive |
| Swipe up | Open favorites shelf (bottom sheet) | None |
| Tap | Show info overlay | None |

**History stack:**
- The app maintains an in-memory ordered list of stations played in the session ✅
- Swipe right walks back through the list; swipe left from the oldest entry advances to a new station ✅
- History is not persisted across sessions

**Accidental swipe prevention:**
- Gestures require a minimum drag distance threshold before committing ✅
- A visual pull indicator shows as the user drags, giving them a chance to cancel by releasing early ✅

**Save — already saved feedback:**
- If the user swipes down on a station already in their saved list, a light haptic fires and an "Already saved ♥" toast appears briefly — no duplicate is added ✅

**Favorites shelf (swipe up):**
- Swipe up from the main screen slides up a bottom sheet showing saved stations ✅
- Tapping a station plays it immediately and dismisses the sheet ✅
- Swipe-to-delete for management ✅
- The current station is visually highlighted with a `speaker.wave.2.fill` icon ✅
- Section header: "Saved stations" ✅
- The mnemonic is intentional: swipe down to store, swipe up to recall ✅

**Tap — info overlay:**
- Auto-dismisses after 3 seconds. Re-triggered on station change, track change, and on return from any sheet. ✅
- **Top row (right-aligned):** share button · settings button ✅
- **Bottom row (left to right):** search icon · filter icon · AirPlay button · play/pause button ✅
  - Search and filter icons are hidden (opacity 0, non-interactive) for non-Plus users so AirPlay and play/pause never shift position ✅
  - Filter icon uses a filled accent color tint when a filter is active ✅
- **Station info (bottom-left):**
  - Current track title (if available) ✅
  - Station name · `heart.fill` if saved · `HQ` badge if bitrate ≥ 320kbps · `waveform` icon if playing ✅
  - Location (state + country) ✅
  - Genre (first tag, title-cased) ✅

---

## 6. Haptics (iOS only) ✅

Haptics are implemented using Core Haptics for maximum expressiveness. Two independent controls exist: command haptics and beat sync haptics.

**Command haptics (on by default):**

| Gesture / Action | Haptic |
|---|---|
| Swipe left (next) | Heavy impact — decisive skip |
| Swipe right (previous) | Medium pulse — going back |
| Swipe down (save, new) | Triple escalating pulse — soft → medium → strong |
| Swipe down (already saved) | Single light tap |
| Button tap (play/pause, settings) | Light tap |
| Swipe up | Reserved |

**Beat-synced haptics (off by default):**
- The app pulses haptics in sync with the detected beat using Core Haptics' precise timing API ✅
- **Off by default** — user must enable in settings ✅
- User-controllable intensity slider (visible only when enabled) ✅
- Can be fully disabled ✅

**Battery saving:**
- Both command and beat-sync haptics are suppressed when Battery Saving mode is active (see Section 10) ✅

---

## 7. Visualization System ✅

The full-screen Metal canvas reacts to the music in real time using frequency analysis.

**Technical approach:**
- Audio analyzed via `AVAudioEngine` + FFT, extracting bass, mid, and treble energy bands ✅
- Beat detection via energy spike analysis ✅
- Rendered at the display's native refresh rate via `MTKView` ✅

**Visual style:**
- Dark background (near-black, not pure black) ✅
- Each station gets a seeded color identity — palette shifts per station so visuals feel unique ✅
- Fluid particle field, expanding ripple rings on beats, glow, breathing ring — music-reactive throughout ✅
- Speech detection suppresses bass/beat pulsation during vocal passages ✅

**Visual states:**

| State | Behavior | Status |
|---|---|---|
| Playing | Full reactive visualization driven by audio frequencies | ✅ |
| Loading / buffering | Slow idle pulse — calm and ambient | ✅ |
| Transitioning | Visualization morphs to new station color identity | ✅ |
| Skip direction sweep | Visualization sweeps in the swipe direction on skip | ✅ |
| Save burst | 8 particles explode radially outward + expanding ring, fades over ~1.5s | ✅ |
| Already-saved flash | Soft center glow (lower intensity than save burst, no particles) | ✅ |

**Visualization themes (Plus):**

All Plus themes (except Drift) are selectable from Settings → Plus → Theme picker. ✅

| Theme | Description | Status |
|---|---|---|
| Drift | Default. Ring + ripples + floating particles. Free. | ✅ |
| Minimal | Single clean ring, no particles or ripples. | ✅ |
| Aurora | Flowing horizontal light bands driven by bass/mid. | ✅ |
| Neon | Sharp electric ring with outer echo ring and dense particle field. | ✅ |
| Alchemy | Domain-warped plasma (WMP Alchemy-inspired) — fluid organic look. | ✅ |

Colors are station-specific and deterministic: derived from a hash of the station ID (`ColorIdentity`). ✅

**Performance:**
- Targets native display refresh rate (120fps) by default ✅
- Throttles to 60fps when Battery Saving mode is active ✅
- Rendering pauses when app is backgrounded ✅

---

## 8. Background Playback ✅

Audio continues playing when the app is backgrounded or the device is locked.

- `AVAudioSession` configured with `.playback` category ✅
- Lock screen controls (play/pause, skip, station name, current track) via `MPNowPlayingInfoCenter` and `MPRemoteCommandCenter` ✅
- If a stream drops while backgrounded, the app silently retries up to 3 times and advances to the next pre-buffered candidate ✅

---

## 9. Recommendation Engine ✅

Drifting builds a taste profile silently using signals from gestures and listening duration.

**Signal weighting:**

| Condition | Signal |
|---|---|
| Swipe left after < 5s of audio | Strong negative |
| Swipe left between 5s–30s of audio | Moderate negative |
| Swipe left after > 30s of audio | Mild negative |
| Listening > 60s without any gesture | Weak positive |
| Swipe right (any duration) | Mild positive |
| Swipe down / save (any duration) | Strongest positive |

Positive signals are not duration-modulated. Signal weighting only applies once audio has successfully started playing.

**Station selection — three phases:**

1. **Cold start** (0–15 interactions) — random selection with genre diversity: avoids picking a station with the same primary genre as the previous one ✅
2. **Learning phase** (16–75 interactions) — weighted random selection using accumulated taste profile scores ✅
3. **Mature profile** (76+ interactions) — stronger exploitation of preferences (score exponent amplified), same weighted random mechanism ✅

In all phases, 15% of picks are fully random (exploration) to surface genuinely new stations and prevent filter bubble. ✅

An additional 20% of picks inject a saved station from the user's favorites list, ensuring loved stations resurface naturally. ✅

**What gets learned:**
- Genre / music style (via station tags) ✅
- Country and language of station ✅
- ❌ Tempo / energy level — not yet implemented
- ❌ Time-of-day patterns — not yet implemented

**Local storage:**
- Taste profile stored as a weighted map on-device, persisted as JSON ✅
- Recently played stations tracked for deduplication ✅
- Saved stations stored as a separate persistent list ✅

No backend. Everything runs client-side. No account required.

---

## 10. Settings ✅

Settings open as a modal overlay (tap → info overlay → settings icon). Music always keeps playing.

**Playback:**
- Play station on launch — Picker with: Random · From Favorites · (Plus) specific saved station ✅
- Battery Saving toggle — limits visualization to 60fps and disables all haptics ✅
  - Automatically enabled when the device is in Low Power Mode; toggle is greyed out with an explanatory note ✅
  - Resets to off when Low Power Mode is disabled ✅

**Haptics (iOS only):**
- Command Haptics toggle (on by default) — controls gesture and button haptics ✅
- Beat Sync Haptics toggle (off by default) ✅
- Haptic intensity slider (visible only when Beat Sync is enabled) ✅

**Plus section (unlocked):**
- Listening Stats ✅
- Station Filter (country + canonical genre picker, 14 genres) ✅
- Station Search (name search with direct play) ✅
- Theme picker (Drift, Minimal, Aurora, Neon, Alchemy) ✅
- High Quality Only toggle (320kbps+, default off) ✅
- Sleep Timer picker (15 / 30 / 60 min) ✅

**Plus section (locked):**
- Feature pitch with upgrade button showing price + "once", restore purchase link ✅

**Reset section:**
- Reset Taste Profile (destructive, confirmation dialog) ✅

---

## 11. Launch Experience ✅

The launch experience must never show a blank black screen. Branding should feel like part of the visualization rather than a loading screen.

**Launch sequence:**
1. On app open, the idle visualization (slow ambient pulse) begins immediately within ~0.3s ✅
2. The app name **"drifting"** fades in centered on the canvas in a thin serif typeface at ~60% opacity ✅
3. The wordmark displays for a minimum of 1.5s — if audio connects faster, the fade-out is delayed ✅
4. Once the first station begins playing, the wordmark fades out over ~0.8s ✅
5. Audio fades in normally; the visualization transitions from idle pulse to reactive mode ✅

**Design constraints:**
- No loading spinner, progress bar, or percentage — the idle pulse IS the loading indicator
- The wordmark reads as ambient texture, not a traditional splash screen

---

## 12. First-Run Gesture Tutorial ✅

A minimal one-time overlay introduces the gestures without interrupting the experience.

**Behaviour:**
- Shown only on the very first launch, persisted via `UserDefaults` ✅
- Appears ~2s after audio first begins playing ✅
- Auto-dismisses after the user performs any swipe or tap gesture ✅
- Auto-dismisses after 8s if no gesture occurs ✅
- Never shown again after dismissal ✅

**Visual design:**
- Five hint labels arranged around the centre of the screen:
  - ← **next station**
  - → **previous station**
  - ↑ **favorites** (upper centre)
  - · **tap** (mid centre)
  - ↓ **save station** (lower centre)
- Low-opacity white (~0.6) — visualization remains visible underneath ✅
- No buttons, no "Got it" prompt
- Fade-in 0.4s, fade-out 0.5s ✅

---

## 13. Monetisation ✅

Drifting's core experience must remain free and uninterrupted. Monetisation must be invisible during normal use.

**Guiding constraint:** if a monetisation mechanism causes a user to think about money while music is playing, it has failed.

---

### Drifting Plus — one-time purchase ✅

A single non-consumable IAP surfaced in Settings only. No prompts, no banners, no subscription.

- Product ID: `com.vitorfreitas.drifting.plus` ✅
- Copy: *"Drifting is free. Plus helps keep it running and unlocks a few extras for people who want more."* ✅
- Price shown live from App Store Connect ✅
- After purchase: upgrade prompt replaced with feature list — never shown again ✅
- Restore purchase link available for users reinstalling the app ✅
- Privacy policy: https://vitorpfr.github.io/drifting/privacy.html (required by App Store for apps with IAP)
- Purchase celebration: haptic + "Welcome to Plus ♥" toast ✅

**Feature split:**

| Feature | Free | Plus |
|---|---|---|
| Full catalog + recommendation engine | ✅ | ✅ |
| All gestures, saving, favorites | ✅ | ✅ |
| Background playback | ✅ | ✅ |
| All haptics | ✅ | ✅ |
| Default visualization (Drift theme) | ✅ | ✅ |
| Stream quality | 64kbps+ | ✅ 320kbps+ filter toggle (default off), HQ badge in overlay |
| Visualization themes | — | ✅ Drift (default), Minimal, Aurora, Neon, Alchemy |
| Home screen widget | — | ✅ now playing — station, track, HQ badge, play state, country/genre, station color gradient |
| Sleep timer | — | ✅ 15 / 30 / 60 min |
| Listening stats | — | ✅ total time, top genres/countries |
| Station filter | — | ✅ country filter + curated canonical genre picker (14 genres) |
| Station search | — | ✅ name search with direct play |

**Deciding where future features land:**
- Improves the core experience (recommendation, transitions, stability, CarPlay) → free
- Adds something on top of an already-complete experience (new themes, new personalization tools) → Plus

**What to avoid:**
- Advertising of any kind
- Featuring stations in exchange for payment
- Locking swipe gestures, favorites, or any core interaction behind a paywall
- Prompts during playback or on launch

---

### macOS / iPad companion app (future)

A separate App Store listing (~$4.99–$7.99) serving the ambient background music use case at a desk. Separate product, not related to Plus.

---

## 14. Home Screen Widget ✅

**Sizes:** small, medium. ✅

**Plus users:** station name, current track (if available), HQ badge, `waveform` icon when playing, country · genre detail line. Background is a linear gradient from the station's unique color (top-leading) to black (bottom-trailing). A decorative ring arc is partially visible at the bottom-right corner. Tapping opens the app. ✅

**Non-Plus users:** "drifting" wordmark + "Get Drifting Plus for widget access" upgrade prompt. ✅

Data is shared via App Group (`group.com.vitorfreitas.drifting`) UserDefaults. AppState writes on every station change, track change, pause/resume, and Plus purchase. Widget reloads on each write + hourly fallback. ✅

---

## 15. Future Considerations

- **Tempo / energy level learning** — factor stream energy into recommendation weights
- **Time-of-day learning** — mellow mornings, energetic evenings
- **Account + sync** — optional sign-up to sync taste profile and saved stations across devices
- **Social features** — following friends (station sharing via deep link is implemented ✅)
- **Android app** — native Kotlin
- **CarPlay support**
- **Advanced haptic choreography** — multi-parameter patterns evolving with music structure (verse/chorus/drop detection)
- **Web app** — React + Vite secondary platform (or Mac desktop app)
