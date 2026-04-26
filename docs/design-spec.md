# Drifting — Product Design Spec

**Date:** 2026-04-25
**Status:** Active — build 5 submitted for App Review
**Supersedes:** design-spec.md (v1, 2026-03-22), design-spec-v2.md (2026-04-03)

---

## Implementation status key

> ✅ Implemented — **⚠️ Partially implemented** — ❌ Not yet implemented

---

## 1. Concept & Principles ✅

Drifting is a passive music discovery app. The user opens it, music starts. There are no menus, no search, no playlists — just a full-screen audio visualization and a single interaction model: swipe to navigate.

**Guiding principles:**
- **Zero friction to music** — audio starts within 2 seconds of opening the app on a good connection; visualization begins immediately regardless of stream state
- **The visualization IS the UI** — a minimal always-visible overlay shows station info and controls at the edges; the center of the screen is unobstructed visualization
- **Every gesture feels intentional** — swipes are weighted with haptics and animation so actions feel satisfying, not accidental
- **The app gets smarter silently** — preference learning happens in the background, never interrupting the experience
- **Local-first** — no account, no login, no barrier. Preferences live on device.

**What Drifting is NOT:**
- A music player (no playlists, no on-demand)
- A radio directory (no browsing or searching)
- A social app — social features are intentionally deferred, not excluded from the long-term vision

---

## 2. Platform Strategy ✅

iPhone is the primary platform, built in native Swift. Mac is supported via Mac Catalyst (same binary, Universal Purchase) — this covers the desktop use case without a separate web app. Android is a potential future platform given its larger radio listener base, but deferred until iOS has traction.

| Platform | Tech | Priority |
|---|---|---|
| iPhone | Native Swift | Current — primary |
| iPad | Native Swift | Current — supported ✅ |
| Mac | Mac Catalyst (same binary) | Current — supported ✅ |
| Android | Native Kotlin | Future — large radio audience, feasible rewrite |
| Web | — | Not planned — Mac Catalyst covers the desktop use case |

**Haptics are iOS/iPadOS-only.** The Haptics section is hidden in Settings on Mac. The Mac app has no haptic feedback.

**Universal Purchase:** A single App Store listing covers iPhone, iPad, and Mac. Users who buy Plus on iOS get it on Mac too.

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
- On station selection, attempt to connect. Retry up to 2 times before advancing to the next candidate.
- Auto-advance after 3 failed retries is not counted as a recommendation signal.
- If a stream drops mid-session: listening time before the drop is discarded and no signal is recorded ✅. The station is deprioritized for the remainder of the session ✅.

**Stream reliability — saved stations:**
- When a user taps a saved station, retry up to 3 times. If the stream cannot be reached, an inline error indicator (⚠) appears on that saved station entry in Settings. ✅ Tapping the station again retries and clears the indicator.

**Handle connectivity issues ⚠️**
- "connecting" label appears during loading state (after wordmark fades out) ✅
- No-connection overlay (wifi.slash) shown when network is unavailable ✅
- Stall watchdog: AVPlayer stall detected via `timeControlStatus`; retries after 5s ✅
- Load timeout: 4s per attempt to catch hung connections ✅
- Analyzer watchdog: if visualization silent while playing, restarts analyzer stream every 3s ✅
- Analyzer/audio sync: frequency data gated on `playerState == .playing` ✅
- Tap to retry while idle ✅
- Auto-resume current station on network recovery ✅
- Network recovery does not auto-skip a playing/rebuffering station — only resumes if playback had fully stopped; AVPlayer handles `.waitingToPlayAtSpecifiedRate` on its own ✅

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
- Pre-buffer has a 10% chance of picking a saved station (matching the main selection rate), otherwise uses the full weighted taste profile for pre-selection ✅

---

## 5. Gesture System & Interaction Model ✅

The full-screen visualization is the only visible element by default. All interaction happens through gestures.

| Gesture | Action | Recommendation Signal |
|---|---|---|
| Swipe left | Next station | Negative — strength depends on listening duration (see Section 9) |
| Swipe right | Previous station (history stack) | Mild positive |
| Swipe down | Save / bookmark — station keeps playing | Strongest positive |
| Swipe up | Open favorites shelf (bottom sheet) | None |
| Tap | Start playback if idle; trigger visualization effect on hold (≥ 0.1s) | None |

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
- Drag to reorder rows; doing so auto-switches sort order to "Custom" (persisted across sessions) ✅
- Sort options: Saved (insertion order) · Custom (drag order) · A–Z · Country · Genre ✅
- Each row shows station name + "Country · Genre" subtitle ✅
- The current station is visually highlighted with a `speaker.wave.2.fill` icon ✅
- The mnemonic is intentional: swipe down to store, swipe up to recall ✅

**Info overlay:** ✅
- Always visible on all platforms (iOS, iPadOS, Mac) — no tap-to-show, no auto-dismiss.
- **Top row (right-aligned):** share button · AirPlay button · settings button ✅
  - AirPlay button hidden on Mac (not applicable) ✅
- **Station info block (bottom-left):** ✅
  - 32pt station logo tile (favicon loaded from Radio Browser; radial gradient fallback using `ColorIdentity`) · track title (if available, indented to align with station name) · station name · `heart.fill` if saved · `HQ` badge if bitrate ≥ 320kbps · `waveform` icon if playing · location (state + country) · genre (first tag, title-cased; always occupies a line to keep position stable when absent)
  - Genre and track title grow upward when present — favicon/name/country are always at a fixed position regardless of whether either is showing ✅
- **Bottom row (right-aligned):** search icon · filter icon · play/pause button ✅
  - Search and filter icons are hidden (opacity 0, non-interactive) for non-Plus users so play/pause never shifts position ✅
  - Filter icon uses a filled accent color tint when a filter is active ✅

**Tap — visualization interaction (hold):** ✅
- Hold anywhere on screen for ≥ 0.1s to trigger an interactive visualization effect.
- Hold duration (0.1s → 1.5s) controls effect intensity — quick hold fires a subtle effect, full charge fires a dramatic one.
- **Charging visual:** a ring contracts inward toward center as energy builds; a soft glow grows at the center. Each theme has its own charge animation that fits its visual language.
- **Release effect (v1 — shockwave):** a tight bright ring expands from center outward across the screen, with a softer echo ring behind it. Theme variants:
  - **Drift:** main ring + one echo ring
  - **Minimal:** single clean thin ring, no echo (stays sparse)
  - **Aurora:** radial brightness wave ripples across the light bands
  - **Neon:** sharp main ring + two echo rings (fits Neon's layered ring motif)
  - **Alchemy:** plasma wave expands with a color-blended echo
- **Haptic pattern:** charging pulses escalate in rate and intensity during the hold (slow/soft → rapid/strong); release fires a two-tap impact scaled by how long you held.
- Quick taps (< 0.1s) start playback if idle; otherwise no action (overlay is always visible). ✅

**Mac keyboard shortcuts & on-screen controls ✅**

On Mac Catalyst, swipe gestures are replaced by keyboard shortcuts and on-screen arrow buttons. These are Mac-only and do not affect iOS/iPadOS.

| Input | Action |
|---|---|
| Right arrow key | Next station |
| Left arrow key | Previous station |
| Down arrow key | Save station (or close favorites shelf if open) |
| Up arrow key | Open favorites shelf |
| Space | Play / Pause |

**On-screen edge-center arrows:** Four chevron buttons sit at the center of each screen edge (←·→·↑·↓). They are very dim (white, low opacity) and styled to match the rest of the UI. Clicking one performs the same action as its keyboard counterpart.

- Pressing a keyboard shortcut also triggers the corresponding on-screen button's press animation, giving visual feedback that the keystroke registered ✅
- On-screen buttons show a grey background flash + scale-down on click/keypress ✅
- Favorites shelf: arrow-down key/button closes the shelf when it is open (instead of saving) ✅
- Shortcuts are also accessible via the Mac menu bar under **Station** ✅

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
| Swipe up | Light tap — opening favorites shelf |
| Hold press (charging) | Escalating pulses — slow/soft at start, rapid/strong at full charge (1.5s) |
| Hold press release | Two-tap impact (rounded thump + sharp echo), both scaled to hold duration |

**Beat-synced haptics (off by default):**
- Pulses haptics with the bass groove using per-frame onset detection (frame-to-frame bass attack measurement) ✅
- Conservative thresholds (attack > 0.18, bass floor > 0.40, 400ms cooldown) — targets a "groove-following" pulse rather than per-kick precision; the analyzer runs on a separate HTTP connection so exact beat alignment is not achievable ✅
- Sharpness modulated by treble level: low treble → deep thud; high treble → sharp crack ✅
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
| Hold charge | Contracting ring + center glow build up while user holds; theme-specific variants | ✅ |
| Shockwave release | Ring expands from center outward on hold release; intensity scales with hold duration; theme-specific variants | ✅ |

**Visualization themes (Plus):**

All themes are selectable from Settings → Playback → Theme (accessible to all users; locked themes are dimmed with a lock icon). ✅

| Theme | Description | Status |
|---|---|---|
| Drift | Default. Ring + ripples + floating particles. Free. | ✅ |
| Minimal | Single clean ring, no particles or ripples. | ✅ |
| Aurora | Flowing horizontal light bands driven by bass/mid. | ✅ |
| Neon | Sharp electric ring with outer echo ring and dense particle field. | ✅ |
| Alchemy | Domain-warped plasma (WMP Alchemy-inspired) — fluid organic look. | ✅ |

Colors are station-specific and deterministic: derived from a hash of the station ID (`ColorIdentity`). ✅

**Tap interaction — v2 ideas (not yet implemented):**

v1 always fires the shockwave. In v2, each tap could randomly pick one of the following effects, keeping the hold mechanic the same:

| Effect | Description |
|---|---|
| Gravity pull | Particles collapse toward center on release, then spring outward — like a breath in and out |
| Beat amplification | One exaggerated beat pulse fires — as if you told the visualization "go harder on this one"; ties to the current bass envelope |
| Color explosion | The station's palette bursts outward from the release point as a saturated color bloom; especially striking on Aurora and Alchemy |
| Vortex | Particles spiral inward on hold, then unwind outward on release; intensity controls spin speed |

**Performance:**
- Targets native display refresh rate (120fps) by default ✅
- Throttles to 60fps when Battery Saving mode is active ✅
- Rendering pauses when app is backgrounded (iOS) or window is minimized/inactive (Mac) ✅

**Mac widescreen:**
- Particle grid is aspect-ratio-aware (`16 * aspectRatio` columns) so dots remain round on widescreen monitors; no distortion ✅

---

## 8. Background Playback ✅

Audio continues playing when the app is backgrounded or the device is locked.

- `AVAudioSession` configured with `.playback` category ✅
- Lock screen / Control Center controls (play/pause, skip, station name, current track) via `MPNowPlayingInfoCenter` and `MPRemoteCommandCenter` ✅
- Lock screen / Control Center artwork: `ColorIdentity` radial gradient rendered immediately; if the station has a favicon URL, it is fetched asynchronously and composited aspect-fill over the gradient ✅
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

1. **Cold start** (0–15 interactions) — music stations only (talk, news, sports, religion excluded); random selection with genre diversity: avoids picking a station with the same primary genre as the previous one ✅
2. **Learning phase** (16–75 interactions) — weighted random selection using accumulated taste profile scores ✅
3. **Mature profile** (76+ interactions) — stronger exploitation of preferences (score exponent amplified), same weighted random mechanism ✅

In all phases, 15% of picks are fully random (exploration) to surface genuinely new stations and prevent filter bubble. During cold start, exploration is also restricted to music stations. ✅

An additional 10% of picks inject a saved station from the user's favorites list, ensuring loved stations resurface naturally. A dedup window of 3 prevents the same favorite from repeating consecutively. ✅

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
- Audio Only toggle (free) — pauses the Metal renderer and stops the audio analyzer entirely; a static radial gradient in the station's color identity is shown instead ✅
  - Saves significant battery on long listening sessions ✅
  - Visualization resumes instantly when toggled off ✅
- Theme — opens a live-preview bottom sheet showing all five themes in a horizontal scrollable row; tapping a theme applies it immediately; locked themes are dimmed with a lock icon and show a "Available with Drifting Plus" hint; accessible to all users ✅

**Haptics (iOS only — hidden on Mac):**
- Command Haptics toggle (on by default) — controls gesture and button haptics ✅
- Beat Sync Haptics toggle (off by default) ✅
- Haptic intensity slider (visible only when Beat Sync is enabled) ✅

**Plus section (unlocked):**
- Listening Stats — total listening time, top genres/countries, top stations with country/genre subtitle; tapping a station plays it and dismisses settings ✅
- Station Filter (country + canonical genre picker, 14 genres) ✅
- Station Search (name search with direct play) ✅
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
  - · **hold** (mid centre)
  - ↓ **save station** (lower centre)
- Low-opacity white (~0.6) — visualization remains visible underneath ✅
- No buttons, no "Got it" prompt
- Fade-in 0.4s, fade-out 0.5s ✅

---

## 13. Monetisation ✅

Drifting's core experience must remain free and uninterrupted. Monetisation must be invisible during normal use.

**Guiding constraint:** if a monetisation mechanism causes a user to think about money while music is playing, it has failed.

---

### Drifting Plus — $4.99 one-time purchase ✅

A single non-consumable IAP surfaced in Settings only. No prompts, no banners, no subscription.

- Product ID: `com.vitorfreitas.drifting.plus` ✅
- Copy: *"Drifting is free. Plus helps keep it running and unlocks a few extras for people who want more."* ✅
- Price shown live from App Store Connect ✅
- After purchase: upgrade prompt replaced with feature list — never shown again ✅
- Restore purchase link available for users reinstalling the app ✅
- Privacy policy: https://vitorpfr.github.io/drifting/docs/privacy.html (required by App Store for apps with IAP)
- Purchase celebration: haptic + "Welcome to Plus ♥" toast ✅

**Feature split:**

| Feature | Free | Plus |
|---|---|---|
| Full catalog + recommendation engine | ✅ | ✅ |
| All gestures, saving, favorites | ✅ (up to 10 saves) | ✅ Unlimited saves |
| Background playback | ✅ | ✅ |
| All haptics | ✅ | ✅ |
| Default visualization (Drift theme) | ✅ | ✅ |
| Stream quality | 64kbps+ | ✅ 320kbps+ filter toggle (default off), HQ badge in overlay |
| Visualization themes | — | ✅ Drift (default), Minimal, Aurora, Neon, Alchemy |
| Home screen widget | — | ✅ now playing — station, track, HQ badge, play state, country/genre, station color gradient |
| Sleep timer | — | ✅ 15 / 30 / 60 min |
| Listening stats | — | ✅ total time, top genres/countries, top stations (tappable — plays station) |
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

### macOS ✅

Drifting runs on macOS via Mac Catalyst — the same binary as iOS. The Mac app is distributed through the same App Store listing using Universal Purchase, so iOS and Mac users share Plus unlocks automatically.

**Mac-specific behaviour:**
- Info overlay is always visible (no tap-to-show/auto-dismiss)
- Keyboard shortcuts via menu bar (Station menu) and direct key bindings
- On-screen edge-center chevron buttons replace swipe gestures
- Haptics section hidden in Settings (not applicable)
- Particle visualization is aspect-ratio-corrected for widescreen
- Renderer pauses when window is minimized or app resigns active (battery)
- Right-click to delete saved stations in Favorites (swipe-to-delete not available with a mouse)
- AirPlay button hidden (not applicable on Mac)

---

## 14. Home Screen Widget ✅

**Sizes:** small, medium. ✅

**Plus users:** station name, current track (if available), HQ badge, `waveform` icon when playing, country · genre detail line. Background is a linear gradient from the station's unique color (top-leading) to black (bottom-trailing). A decorative ring arc is partially visible at the bottom-right corner. Tapping opens the app. ✅

**Non-Plus users:** "drifting" wordmark + "Get Drifting Plus for widget access" upgrade prompt. ✅

Data is shared via App Group (`group.com.vitorfreitas.drifting`) UserDefaults. AppState writes on every station change, track change, pause/resume, and Plus purchase. Widget reloads on each write + hourly fallback. ✅

---

## 15. Future Considerations

### v1.1 — ship before driving external traffic

- **iPad info overlay layout** *(known bug)* — track/artist info not visible on iPad due to iPhone-specific positioning; needs layout fix for iPad screen geometry
- **Station blocking** — "never play this station" permanent filter; different from skipping — strong negative signal; add to a blocked IDs list and exclude from selection; will be one of the most common early requests (ad-heavy stations, jarring genres)
- **Persistent listening history** — currently session-only; keep last 20–30 stations across sessions so users don't lose stations they heard but didn't save
- **Station logos / artwork** ✅ — 32pt rounded tile in InfoOverlay (top-left of the station info block); loads favicon from Radio Browser `favicon` field (http/https only, validated at decode time); falls back to a `ColorIdentity` radial gradient; gradient shown immediately, favicon fades in once loaded. Same favicon also appears in lock screen / Control Center artwork (aspect-fill over gradient, fetched async after station change). Station search field autofocuses on open so keyboard appears immediately ✅
- **Saved stations limit** ✅ — free users can save up to 10 stations; Plus users get unlimited; when the limit is hit, a light haptic fires and a tappable "Save limit reached" toast appears (tapping opens Settings to the Plus section); grandfathering: on first launch of v1.1, if `favorites.json` exists on disk the user is an upgrade from v1.0 and `hasGrandfatheredUnlimitedSaves` is written as `true` (treated as Plus for save limits, invisibly); fresh installs get `false`

### v1.x — based on early user feedback

- **CarPlay support** — radio is fundamentally a car use case; requires a separate CarPlay scene but audio engine is already correct; unlocks a large natural audience
- **Optional genre seed on first launch** — single optional screen on first launch with 4–6 genre chips (dismissible); improves cold start dramatically without violating zero-friction; users who skip get current cold start, users who pick genres get a better day one
- **Time-of-day learning** — mellow mornings, energetic evenings; makes the app feel genuinely intelligent without extra user action
- **Lock screen widget** — iOS 16+ lock screen widget showing station name + waveform icon; low effort, widget infrastructure already exists
- **Tempo / energy level learning** — factor stream energy into recommendation weights; more nuanced than genre alone
- **v2 tap effects** — random effect selection on hold release: gravity pull, beat amplification, colour explosion, vortex (see Section 7 for full descriptions)
- **More visualization themes** — 1–2 new Plus themes per major update; gives existing Plus users ongoing value and free users a reason to upgrade
- **Mac — expanded keyboard shortcuts** — power-user shortcuts beyond the current navigation set: `S` to open station search, `F` to open favorites shelf and navigate it with arrow keys, `B` to toggle battery saving, number keys to jump to a saved station by position; makes the Mac version feel like a native app rather than a ported one

### Long term

- **Account + sync** — optional sign-up to sync taste profile and saved stations across devices; high effort, requires backend; defer until there's clear multi-device demand from users
- **Advanced haptic choreography** — multi-parameter patterns evolving with music structure (verse/chorus/drop detection)
- **Social features** — station sharing via deep link is already implemented ✅; anything beyond that (following friends, shared listening) is a different product; correctly deferred

### Separate platform track

- **Android** — native Kotlin; radio listener base is significantly larger on Android; visualization rewritten in GLSL/Vulkan, recommendation engine and UI are direct ports; target start ~2–3 months post-iOS launch once feedback is incorporated
