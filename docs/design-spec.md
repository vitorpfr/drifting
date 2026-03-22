# Drifting — Product Design Spec

**Date:** 2026-03-22
**Status:** Approved

---

## 1. Concept & Principles

Drifting is a passive music discovery app. The user opens it, music starts. There are no menus, no search, no playlists — just a full-screen audio visualization and a single interaction model: swipe to react.

**Guiding principles:**
- **Zero friction to music** — audio starts within 2 seconds of opening the app on a good connection; visualization begins immediately regardless of stream state
- **The visualization IS the UI** — no controls, labels, or chrome on screen by default
- **Every gesture feels intentional** — swipes are weighted with haptics and animation so actions feel satisfying, not accidental
- **The app gets smarter silently** — preference learning happens in the background, never interrupting the experience
- **Local-first** — no account, no login, no barrier. Preferences live on device.

**What Drifting is NOT in v1:**
- A music player (no playlists, no on-demand)
- A radio directory (no browsing or searching)
- A social app — social features (sharing, profiles, follows) are intentionally deferred, not excluded from the long-term vision

---

## 2. Platform Strategy

iOS is the primary platform, built in native Swift to deliver the best possible haptic and audio experience. The web app serves the secondary "work computer" use case. Android is deferred to a future version.

| Platform | Tech | Priority |
|---|---|---|
| iOS | Native Swift | v1 — primary |
| Web | React + Vite | v1 — secondary |
| Android | Native Kotlin | Future version |

**Shared logic** (recommendation engine, Radio Browser API integration, quality filtering) is implemented in Swift for iOS and TypeScript for web independently. The concepts and interfaces are aligned but each platform owns its implementation.

**Haptics are iOS-only in v1.** The web app has no haptic feedback. The web platform's Vibration API has insufficient iOS Safari support and is excluded from scope.

---

## 3. Radio Station Catalog

**Source:** Radio Browser API — open, community-maintained, ~30k stations worldwide.

**Quality filters applied at query time:**
- Minimum 128kbps bitrate
- Music-only stations (exclude news, talk, sports, religion tags)
- Active streams only (last-checked alive)
- Prefer stations with genre and language metadata (better for recommendation learning)
- All languages and regions included

**Stream reliability — discovery flow:**
- On station selection, attempt to connect. Retry up to 3 times before advancing to the next candidate.
- Auto-advance after 3 failed retries is not counted as a skip signal.
- If a stream drops mid-session (after audio has started): listening time accumulated before the drop counts toward signal calculation as normal; the station receives a reliability penalty and is deprioritized for selection for the remainder of the session.
- Signal weighting (Section 9) only applies once audio has successfully started playing. A swipe left on a buffering or failed stream carries no recommendation signal regardless of elapsed time.

**Stream reliability — saved stations:**
- When a user taps a saved station, retry up to 3 times. If the stream cannot be reached, show an inline error indicator on that saved station entry. Do not auto-advance to a random station — the user's intent was to play a specific saved station.

---

## 4. Stream Loading, Buffering & Transitions

Perceived loading time is a core UX concern. The app uses pre-buffering and carefully timed audio/visual fades to make every transition feel smooth and intentional — never abrupt.

**App launch:**
- The station catalog is pre-cached locally after first install and refreshed silently in the background — no API call on launch, station selection is instant
- The idle visualization (slow ambient pulse) plays immediately on launch, within ~0.3s of the app opening
- Stream buffering begins immediately in parallel with the UI rendering
- Target: audio begins within 2 seconds of launch on a good connection (WiFi / 4G)
- As the stream connects, audio fades in from silence to full volume over ~1.5s
- The visualization simultaneously transitions from idle pulse to fully reactive mode on the same curve as the audio ramp
- The effect: music and visuals emerge together, as if waking up — not appearing suddenly
- On slow connections, the idle visualization continues until buffering completes; the user always sees something happening immediately

**Between stations — skip (swipe left or down):**
Three-act transition:
1. **Exit** — on swipe, audio fades out over ~0.4s while the visualization sweeps in the swipe direction
2. **Loading** — idle pulse plays while the pre-buffered next stream takes over (ideally near-invisible due to pre-buffering)
3. **Enter** — new station's audio fades in from silence over ~1s while the visualization morphs into the new station's color identity and ramps to fully reactive

The audio volume and visualization energy always move together on the same curve. The visualization carries emotional continuity during the gap.

**Like (swipe right):**
No audio transition — music keeps playing uninterrupted. The intensity burst animation is the only feedback.

**Save (swipe up):**
No audio transition — music keeps playing uninterrupted. Subtle upward particle trail confirmation.

**Timing summary:**

| Moment | Audio | Visualization |
|---|---|---|
| Launch | Fade in over ~1.5s | Idle pulse → reactive, same curve |
| Skip (exit) | Fade out over ~0.4s | Sweeps in swipe direction |
| Skip (enter) | Fade in over ~1s | Morphs to new color identity, ramps up |
| Like / Save | No change | Brief burst / flash, then continues |

**Web — first interaction:**
- Browsers block audio autoplay without a prior user gesture
- A minimal full-screen dark canvas is shown with a subtle animated pulse and a "tap to start" prompt
- The first tap unlocks the Web Audio context and begins the launch sequence above
- Subsequent sessions: attempt autoplay on load, fall back to first-interaction screen only if blocked
- This screen is the only exception to the "no UI chrome" principle and must be as minimal as possible

**Pre-buffering:**
- While a station is playing, the app silently pre-selects and begins buffering the next candidate in the background
- On swipe, the transition begins immediately — the pre-buffered stream reduces or eliminates the loading act
- If pre-buffering has not completed (swipe happened quickly, or slow network), the idle pulse loading state plays until buffering finishes

---

## 5. Gesture System & Interaction Model

The full-screen visualization is the only visible element by default. All interaction happens through gestures.

| Gesture | Action | Recommendation Signal |
|---|---|---|
| Swipe left | Skip (dislike) | Negative — strength depends on listening duration (see Section 9) |
| Swipe right | Like — station keeps playing | Positive |
| Swipe up | Save / bookmark — station keeps playing | Strong positive |
| Swipe down | Next (neutral) — move on without signaling | No signal, regardless of listening duration |
| Tap | Show info overlay + settings access | None |

**Positive signals are not duration-modulated.** A like (swipe right) or save (swipe up) carries the same weight regardless of how long the user has been listening — the intent is unambiguous.

**Accidental swipe prevention:**
- Gestures require a minimum drag distance threshold before committing
- A visual "pull" indicator shows as the user drags, giving them a chance to cancel by releasing early

**Tap — info overlay:**
- Station name, country, and genre fade in for ~3 seconds then disappear
- A small, near-transparent settings icon appears alongside the info
- Tapping the settings icon opens the settings panel as a modal overlay (music keeps playing)

---

## 6. Haptics (iOS only)

Haptics are a core part of the emotional experience on iOS, implemented using Core Haptics for maximum expressiveness. The web app has no haptics in v1.

**Gesture haptics:**

| Gesture | Haptic |
|---|---|
| Swipe left | Heavy impact — the "no" feel |
| Swipe right | Medium + soft double pulse — satisfying "yes" |
| Swipe up | Light single tap — clean and precise |
| Swipe down | Lightest neutral confirmation |
| Tap | No haptic |

**Beat-synced haptics (v1 feature):**
- The app analyzes the audio in real time and pulses haptics in sync with the music beat using Core Haptics' precise timing API to minimize drift
- Enabled by default
- User-controllable via an intensity slider in settings
- Can be fully disabled in settings

---

## 7. Visualization System

The full-screen canvas is the entire UI. It reacts to the music in real time using frequency analysis.

**Technical approach:**
- Audio analyzed via AVAudioEngine + FFT on iOS; Web Audio API on web
- FFT extracts bass, mids, and highs to drive visual parameters
- Rendered at the display's native refresh rate (60Hz, 120Hz, etc.)

**Visual style:**
- Dark background (near-black, not pure black)
- Each station gets a seeded color identity — palette shifts per station so visuals feel unique to each stream
- Inspired by Windows Media Player — frequency bars, flowing waveforms, symmetry — but modernized with fluid particle systems, bloom/glow effects, and smooth interpolation
- Motion never feels mechanical

**Visual states:**

| State | Behavior |
|---|---|
| Playing | Full reactive visualization driven by audio frequencies |
| Loading / buffering | Slow idle pulse — calm and ambient, signals something is coming |
| Transitioning | Visualization morphs/dissolves as old station fades and new one loads |
| Like burst | Brief intensity spike — brighter, more energetic — then settles |
| Save flash | Subtle upward motion, then returns to normal |

**Performance:**
- Targets native display refresh rate by default
- Automatically drops to 60fps if: battery saver / low power mode is active, or device frame budget is consistently exceeded
- This throttling is automatic — no user-facing control
- Rendering pauses when app is backgrounded
- Fallback to simpler Canvas 2D renderer on lower-end devices (web)

---

## 8. Background Playback

Audio continues playing when the app is backgrounded or the device is locked on both platforms.

**iOS:**
- AVAudioSession configured with `.playback` category so audio continues uninterrupted in the background
- v1 includes minimal lock screen controls (pause/stop only) to satisfy Apple App Store requirements for background audio apps. Richer controls (skip, station info) are deferred to a future version.
- If a stream drops while backgrounded, the app silently retries (up to 3 times) and advances to the next pre-buffered candidate. The user may notice a brief silence but no action is required.

**Web:**
- The Web Audio API context is kept alive when the tab is backgrounded
- Browsers may throttle or suspend background tabs; the app detects suspension and resumes the stream when the tab returns to the foreground

---

## 9. Recommendation Engine

Drifting builds a taste profile silently using signals from gestures and listening duration.

**Signal weighting:**

Duration modifies the strength of a left-swipe (dislike) only. Positive signals (like, save) are not duration-modulated. Swipe down is always neutral. Signal weighting only applies once audio has successfully started playing (see Section 3).

| Condition | Signal |
|---|---|
| Swipe left after < 5s of audio | Strong negative |
| Swipe left between 5s–30s of audio | Moderate negative |
| Swipe left after > 30s of audio | Mild negative |
| Swipe down (any duration) | No signal |
| Listening > 60s without any gesture | Weak positive |
| Like (swipe right, any duration) | Positive |
| Save (swipe up, any duration) | Strongest positive |

**Interaction counting:**
- A gesture (any swipe) always counts as exactly one interaction
- A passive listen exceeding 60s without a gesture counts as one interaction
- If a gesture occurs at or after 60s, only the gesture is counted — the passive listen is superseded, not added

**What gets learned:**
- Genre / music style
- Language and region of station
- Tempo / energy level (where metadata allows)
- Time-of-day patterns (e.g. mellow mornings, energetic evenings)

**Station selection — three phases:**

1. **Cold start** (0–15 interactions) — random selection from quality-filtered catalog. Diversity is enforced by bucketing stations by genre and cycling through buckets, preventing consecutive plays of the same genre.
2. **Learning phase** (16–75 interactions) — 60% exploit known preferences, 40% explore new territory
3. **Mature profile** (76+ interactions) — weighted toward preferences, always keeping ~20% exploration to avoid filter bubble

**Local storage:**
- Taste profile stored as a weighted map on-device
- Interaction history kept for deduplication (avoid replaying recently skipped stations)
- Saved stations stored as a persistent separate list

**No backend in v1.** Everything runs client-side. No account required.

---

## 10. Settings

Settings open as a modal overlay (tap → info overlay → settings icon). The main screen and audio remain active behind the modal. Music always keeps playing.

The gesture context within settings is independent of the main screen — swipe gestures inside the modal are standard list interactions and do not trigger the main screen's recommendation gesture system.

**Haptics (iOS only):**
- Beat-synced haptics toggle (on by default)
- Haptic intensity slider (visible only when haptics are enabled)

**Saved Stations:**
- A scrollable list of bookmarked stations (added via swipe up on the main screen)
- Tapping a saved station plays it immediately and closes settings
- Removing a saved station: swipe left on the list entry — pure UI deletion, no recommendation signal
- If a saved station's stream is unavailable, it shows an inline error indicator

**Taste Profile:**
- Reset taste profile — clears all learned preferences and interaction history, returning to cold start

---

## 11. Future Considerations (Out of Scope for v1)

- **Account + sync** — optional sign-up to sync taste profile and saved stations across devices
- **Social features** — sharing stations, following friends, seeing what others are listening to
- **Android app** — native Kotlin, same product experience
- **Advanced haptic choreography** — multi-parameter haptic patterns that evolve with music structure (verse/chorus/drop detection) beyond simple beat-sync
- **Richer lock screen controls** — skip, station info, CarPlay support
- **Offline saved stations** — pre-buffer saved stations for low-connectivity environments
