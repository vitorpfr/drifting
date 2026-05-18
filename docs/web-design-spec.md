# Spectrale — Web Design Spec

**Created:** 2026-05-09  
**Status:** Live at spectrale.app — pre-release (deep links outstanding)  
**iOS counterpart:** `ios-design-spec.md`  
**Port plan:** `plans/2026-05-09-web-port-plan.md`

---

## 1. Why a web version

iOS submission rejected under Guideline 5.2.3. Appeal assessed at ~25–40% probability on first reply — plausible but draining. The web version sidesteps Apple's gatekeeping entirely and ships independently. The iOS app is parked, not abandoned.

All core work carries over: catalog source, direct streaming, recommendation engine, gesture model, catalog filtering. The web version is not a compromise — it is the same product on a different runtime.

---

## 2. What the web version is

Same product principles as iOS:

- **Zero friction to music** — audio starts on first tap; visualization begins immediately
- **The visualization IS the UI** — minimal overlay at edges, unobstructed center
- **Every gesture feels intentional** — swipes are the only navigation
- **The app gets smarter silently** — recommendation engine runs in IndexedDB, never interrupts
- **Local-first** — no account, no login, no backend

---

## 3. Tech stack

| Layer | Choice |
|---|---|
| Framework | React 19 + Vite 6 + TypeScript 5.7 |
| Visualization | Three.js r184 (WebGL) |
| Audio analysis | Web Audio API (AnalyserNode, FFT) |
| Persistence | IndexedDB (recommendation engine, saved stations) |
| Tests | Vitest + Testing Library + jsdom |
| Hosting | Cloudflare Pages |
| Domain | spectrale.app (live) |

No backend. All state on-device.

---

## 4. What differs from iOS

| Feature | iOS | Web |
|---|---|---|
| Haptics | Core Haptics, full expressiveness | Not implemented — browser vibration API too limited |
| CarPlay | Supported | Not applicable |
| Lock screen controls | MPNowPlayingInfoCenter (full) | Web Media Session API — can ship independently of PWA; deferred (see §18) |
| Background audio | Native, always | Requires PWA install on iOS Safari (16.4+); plain Safari pauses on lock |
| App Store | Submitted | Not applicable |
| Visualization | Metal shader (GPU) | WebGL / Three.js GLSL (GPU) — visually equivalent |
| Themes | 5 (Drift, Minimal, Aurora, Neon, Alchemy) | Drift only for now; others deferred |
| Plus / IAP | $4.99 one-time | Web is free; Plus deferred |

---

## 5. Catalog & CORS strategy

**Source:** radio-browser.info public API. Same quality filters as iOS (≥128kbps, music genre, active).

**CORS feasibility test (2026-05-09, 200-station sample):**
- 79% of stations returned CORS headers
- 73% are usable audio streams with CORS
- Geographically broad: DE, FR, NL, US, TR, PL, ES, IT, AT, RU

**Decision: Plan A — full visualization, CORS-filtered catalog**

Catalog entries carry a `corsFriendly` boolean (precomputed at build time). CORS-friendly stations get the audio-reactive WebGL renderer (AnalyserNode FFT); others get the ambient flow-field renderer. No label shown — the visual difference is the signal.

If a CORS-friendly station's analysis returns silence in-browser despite the flag, VisualizationManager falls back to ambient mode after a 2s grace period.

**Catalog build:** `npx tsx scripts/buildCatalog.ts` — outputs `public/catalog.json`. Run periodically to refresh CORS flags as hosting providers change.

**Current state:** `corsFriendly` values used as-is from catalog. CORS stations get beat-driven ripples via Web Audio AnalyserNode; non-CORS stations get the ambient 5s pulse timer.

---

## 6. Launch experience

Matches iOS spec §11:

1. Ambient visualization (flow-field shader) starts immediately — no blank frame
2. "spectrale" wordmark centered: Cormorant Garamond 300, `clamp(40px, 7vw, 72px)`, `rgba(255,255,255,0.6)`
3. "tap to begin" below: system-ui, 12px, uppercase, `rgba(255,255,255,0.2)`
4. Tap anywhere → "connecting…" replaces subtitle; cursor becomes `wait`
5. Station loads → wordmark fades out over 0.8s (CSS transition); burst animation fires
6. Station info appears bottom-left

**"tap to begin" rationale:** browsers block audio autoplay without a user gesture. The wordmark is the solution, not a workaround — it sets expectation and makes the mandatory gesture deliberate.

---

## 7. Gesture system

Matches iOS spec §5, adapted for touch + mouse on web.

| Gesture | Action | Signal |
|---|---|---|
| Swipe left | Next station | Negative (strength by listen duration) |
| Swipe right | Previous station (history stack) | Mild positive |
| Swipe down | Save / bookmark | Strongest positive |
| Swipe up | Open saved stations shelf | None |
| Tap | Start if idle | None |

- Minimum drag threshold before commit; visual pull indicator; cancel on early release
- History stack: in-memory ordered list of stations in session; not persisted
- "Already saved ♥" toast if station already in favorites (no duplicate added)

**Not implemented yet.** Scheduled for next iteration.

---

## 8. Station info overlay

Always visible while playing. No tap-to-show, no auto-dismiss.

**Bottom-left block:**
- Station name: system-ui, 16px, weight 500, `rgba(255,255,255,0.85)`
- Country: system-ui, 13px, weight 400, `rgba(255,255,255,0.45)` — normalized (see §9)
- Genre: system-ui, 13px, weight 400, `rgba(255,255,255,0.35)` — first tag, title-cased

**Top-right:** share · settings (both implemented)
**Bottom-right:** play/pause + volume slider (both implemented)

`pointer-events: none` on the info block so gestures pass through to the canvas.

---

## 9. Country name normalization

radio-browser.info uses verbose ISO 3166 title-cased names. Map applied in `App.tsx`:

| Raw | Display |
|---|---|
| The United States Of America | United States |
| The Russian Federation | Russia |
| The United Kingdom Of Great Britain And Northern Ireland | United Kingdom |
| The Netherlands | Netherlands |
| (others as needed) | (short form) |

---

## 10. Visualization system

Matches iOS spec §7, implemented in WebGL (Three.js).

**Renderers:**

| Mode | Trigger | Description |
|---|---|---|
| Ambient | Always (audio-reactive wired but never activated) | Flow-field FBM shader, station-seeded color, organic motion |
| Audio-reactive | Reserved for future — wired in VisualizationManager, never switched to | 8000-particle system driven by AnalyserNode FFT (bass/mid/treble) |

**Ambient renderer parameters (as shipped):**
- Time multiplier: `0.30` (fast, organic)
- Distortion scale: `r * 1.6` (turbulent pattern folding)
- Brightness: `pow(f, 1.2) * 1.55 + 0.12` — floor prevents fully-black areas, reduced peak avoids white bleaching
- Vignette: `smoothstep(0.5, 1.8, dist)` — gentle edge darkening, corners at ~30% brightness
- Station colors: full saturation + brightness (`hsbToRgb(hue, 1.0, 1.0)`)

**Visual effects (implemented):**

| Effect | Implementation |
|---|---|
| Burst | Full-screen GLSL shader — smoothstep ring + 8 radial particles, fires on station change and save |
| Color identity | UUID-seeded color per station (djb2 hash → full HSB wheel), 2s HSL lerp transition |
| Ripple rings | 4-slot expanding rings; CORS stations: beat-driven (onset threshold 0.05, cooldown 350ms); non-CORS: 5s autonomous timer |
| Tap charge | Two-layer center glow (tight core + soft halo) while holding, accumulates over 2s |
| Tap shockwave | Primary ring + echo ring on hold release (~0.7s each); intensity scales with hold duration (max at 2s); 150ms dead zone for taps |
| Volume control | Bottom-right slider (co-located with play/pause); persists across station transitions via `fadeIn(ms, targetVolume)` |

**GLSL shader patterns (established — use for all future effects):**

- Center + aspect-correct UV: `vec2 uv = vUv * 2.0 - 1.0; uv.x *= u_aspect;`
- Ring: `smoothstep(0.018, 0.0, abs(dist - radius))` — smooth edge IS the glow
- Ring radius scaled to diagonal: `length(vec2(u_aspect, 1.0)) * 1.1` — fills screen on any viewport
- Fade: `pow(1.0 - phase, 1.5)` — cubic, matches iOS feel
- Blending: `THREE.AdditiveBlending` — brightens underlying visualization
- Phase: when complete, set both `this.phase = -1` AND the uniform to `-1` in the same step — otherwise the shader runs as a full-screen pass every frame

**Performance rules:**

- Pixel ratio capped at `Math.min(window.devicePixelRatio, 1)` — retina would render 4× pixels for no perceptible gain on abstract visuals
- FBM ambient shader: 3 octaves (was 5 — 40% cheaper, visually indistinguishable)
- One `getDelta()` call per frame; track `elapsedMs` manually — `getElapsedTime()` calls `getDelta()` internally as a side effect

---

## 11. Audio

**Player:** `AudioPlayer` wraps `HTMLAudioElement` + `AudioContext` + `AnalyserNode`.

- `play(url, corsFriendly)` — if `corsFriendly`, routes audio through `AudioContext` for FFT analysis; otherwise plays directly via `audio.volume` without Web Audio graph
- `getFrequencyBuckets()` — returns `{ bass, mid, treble }` (0–1 each) for visualization
- `isSilent()` — returns true if all buckets below threshold; used by VisualizationManager silence fallback

**Critical invariant — AudioContext is only created for CORS streams.** Creating an AudioContext for non-CORS streams risks starting it in `suspended` state (iOS Safari / Chrome autoplay policy). Once `createMediaElementSource` is called, audio output is re-routed exclusively through the Web Audio graph — a suspended context produces complete silence. Non-CORS streams use `audio.volume` directly and are never connected to a graph.

**Implemented:**
- `fadeIn(durationMs, to = 1)` — 1s fade-in, respects current volume slider setting
- `fadeOut(durationMs)` — 400ms crossfade on skip
- Retry up to 3× with 4s load timeout per attempt
- Stall watchdog (5s) auto-advances to next station
- Auto-resume on network recovery and page visibility restore
- Pre-buffering next station (30s expiry, re-primed on pointer activity)
- `setVolume(v)` — live volume control; `desiredVolumeRef` in `useAudioLifecycle` threads it through all transitions
- Stale buffer reconnect — if the user was paused for >30s, `resume()` clears the src and calls `play()` fresh; on tab return after a long pause, `preconnect()` is called silently in the background so the buffer is ready before the user hits play
- External pause sync — on mobile, any pause that occurs while the app is backgrounded (e.g. lock screen controls) is treated as user-initiated on tab return, preventing unwanted auto-resume

**Mobile preload behavior — `prime()` uses `preconnect()` only on touch devices.** On desktop, the preloaded next-station player calls `play()` silently at volume 0 so the buffer is warm at navigation time. On mobile (`navigator.maxTouchPoints > 0`), only `audio.src` is set (`preconnect()`); `play()` is never called on the preloaded player. This prevents iOS Safari from leaking audio from the preloaded stream — gainNode silencing is unreliable on iOS for MediaElementAudioSourceNode. `play()` is called inside `goNext()` instead, which runs within the swipe gesture context where `AudioContext.resume()` is guaranteed to succeed.

---

## 12. Recommendation engine

Matches iOS spec §9. **Done.**

Runs entirely in the browser. Persists taste profile and saved stations in IndexedDB.

**Signal weighting:**

| Condition | Signal |
|---|---|
| Skip after <5s | Strong negative |
| Skip 5–30s | Moderate negative |
| Skip >30s | Mild negative |
| Dwell >60s without gesture | Weak positive |
| Swipe right | Mild positive |
| Save | Strongest positive |

**Selection phases:**

1. Cold start (0–15 interactions) — random, genre-diverse, music only
2. Learning (16–75) — weighted random from taste profile
3. Mature (76+) — stronger exploitation (amplified score exponent)

15% exploration (fully random) in all phases. 10% of picks inject a saved station.

---

## 13. Saved stations

Matches iOS spec §5 saved stations behavior. **Done.**

- Persisted in IndexedDB
- Save on swipe-down; burst animation triggers (already built)
- "Already saved ♥" toast if duplicate
- Save limit: cap at 25 for now (Plus not planned for web)
- Shelf: up-swipe bottom sheet; tap to re-play; swipe-to-delete
- Each row: station name + country · genre

---

## 14. PWA

Required for background audio on iOS Safari. **Not yet implemented.**

- `manifest.json`: name "Spectrale", display standalone, theme color dark, icons
- Service worker: cache shell assets; allow audio continuation in background
- Without PWA install on iOS Safari, audio pauses on screen lock

---

## 15. First-run gesture tutorial

Matches iOS spec §12. **Done.**

- Five hint labels arranged around screen center: ← next · → prev · ↑ favorites · ↓ save · · hold
- Low-opacity white (~0.6), visualization visible underneath
- Shown ~2s after first audio, auto-dismiss on any gesture or after 8s
- Never shown again (persisted in localStorage)

---

## 16. Connectivity handling

**Done.** See §17 status entry for full detail.

---

## 17. Implementation status summary

| Feature | Status |
|---|---|
| Launch experience | ✅ Done — on tap, wordmark disappears instantly; skeleton shimmer appears at bottom-left (station info placeholder) + real controls appear immediately (volume/settings functional, play/pause disabled until playing) |
| Burst animation | ✅ Done |
| Station info overlay | ✅ Done — favicon tile (genre gradient fallback, fades in on load), now-playing via ICY metadata (CORS stations only, polls every 30s), ♥ badge; name/country/favicon anchored; track title and genre float above/below without shifting |
| Country normalization | ✅ Done |
| Ambient visualization | ✅ Done |
| Audio-reactive visualization | ✅ Built, preserved in VisualizationManager — ambient-always for now; audio-reactive mode reserved for future exploration |
| Gesture model | ✅ Done — all four directions + keyboard (←→↑↓ Space) + on-screen chevrons + touch swipe recognition (`useSwipeGesture` hook: 50px H / 80px V thresholds, left-edge dead zone, scroll prevention, pointer capture, flash feedback); hold effects deferred |
| Any-key to start | ✅ Done — any keypress (excluding modifier-only combos) on the launch screen triggers playback |
| Pre-load first station | ✅ Done — on mount, catalog is fetched and first station stream is pre-connected (audio.src set, browser buffers stream); on first tap/keypress, play() reuses the pre-connected player without resetting the buffer |
| Next / prev station | ✅ Done — current audio/UI stays live during transition; "connecting…" shown in center |
| Audio fades | ✅ Done — 1s fade-in on station start; crossfade on navigation (old fades out 400ms while new fades in 1s, triggered only after new stream is ready to eliminate silence gaps) |
| Save + saved shelf | ✅ Done — swipe-down/ArrowDown/↓ saves; swipe-up/ArrowUp/↑ opens shelf; IndexedDB persistence (cap 25); toast feedback; burst on save; toast is dismissed when shelf opens (prevents z-index overlap) |
| Play / pause | ✅ Done — Space key; AudioPlayer.pause/resume |
| Recommendation engine | ✅ Done — pure functions, IDB schema (v2), `useRecommendation` hook, and App.tsx wiring all complete; signals recorded on skip/prev/save; dwell timer fires at 60s |
| Info overlay controls (settings) | ✅ Done — share button (copies URL to clipboard), settings gear (opens modal), play/pause toggle; modal has taste profile reset, audio-only toggle (pauses visualization), and clear saved stations |
| Donation jar (Ko-fi link in Settings) | ✅ Done — "Support Spectrale / Buy a coffee ☕" row at bottom of settings modal; links to ko-fi.com/spectrale |
| First-run tutorial | ✅ Done — `useFirstRunTutorial` hook (2s delay, 8s auto-dismiss, localStorage guard); `GestureTutorial` component; adapts labels for touch (swipe ← next) vs desktop (← prev / → next); hold hint included as placeholder; recommendation tagline "your taste profile builds as you listen"; dismissed by any goNext/goPrevious/save/shelf/playPause/share/audioOnly/settings action |
| Connectivity handling / retry | ✅ Done — full connectivity resilience in `useAudioLifecycle`: 3-attempt retry with 4s load timeout per attempt on initial load and navigation; stall watchdog (5s) auto-advances to next station on stream drop; amber "STREAM DROPPED" toast shows live state ("moving to next station…" while transitioning, "moved to next station" after, tap to dismiss); auto-resume on network recovery (reconnects if stream dropped, resumes if paused); page visibility recovery (resumes paused player on tab focus); all audio/navigation state consolidated in `useAudioLifecycle` |
| Renderer cross-fade (300ms) | ✅ Done — 300ms opacity cross-fade between ambient ↔ audio-reactive renderers; 2s HSL color transition between stations (shortest hue path, smoothstep easing); VisualizationManager owns all color state; station color derived from djb2 hash of station UUID (vivid, deterministic, full wheel coverage — replaces sparse genre map) |
| Genre hue bias on station colors | 🔭 Exploration — shift the hash hue toward a genre-appropriate range (e.g., jazz → warm, electronic → cool) while keeping per-station uniqueness; requires mapping common radio-browser tag variants |
| Ko-fi nudge | ✅ Done — fires after "Saved ♥" toast dismisses once 10 cumulative saves threshold is reached (tracked in `saves_count` localStorage); once per session + 7-day cooldown; auto-dismisses after 5s; copy: "Enjoying Spectrale? Support development ☕"; positioned above station info on mobile |
| Auth-required stream blocking | ✅ Done — fetch() HEAD probe before audio element load; 401 → treated as failed stream, retry logic skips to next station; probe skipped on mobile (touch devices) to preserve iOS Safari autoplay gesture context and avoid consuming the 4s load timeout budget |
| HQ quality badge | ✅ Done — "HQ" shown next to station name for ≥192kbps streams (~17% of catalog); replaces debug BEAT/PULSE badge |
| Hosting + domain | ✅ Done — Cloudflare Pages; spectrale.app live (2026-05-17); Ko-fi donation link operational (Stripe integrated) |
| Per-station deep link + share button | ✅ Done — share button copies `?station=<uuid>` when playing; deep link auto-plays that station on load; URL cleaned up after first station loads; falls back to random pick if UUID not in catalog; launch screen shows "tap to play [Station Name]" when opened via deep link; idle subtitle updated to "tap to discover radio in color" |
| PWA shell | ✅ Done — `manifest.json` (standalone display, dark theme, icons from existing 1024×1024 favicon); minimal service worker (network-first navigation, offline fallback to cached index); iOS meta tags (`apple-mobile-web-app-capable`, touch icon, status bar); registered in `main.tsx` on load |
| Tap shockwave / charge effects | ✅ Done — `ShockwaveEffect`: center glow charge, primary + echo ring on release; hold >150ms triggers; intensity 0→1 over 2s hold; `App.tsx` pointer handlers with 15px movement cancel |
| Ripple rings | ✅ Done — `RippleEffect`: 4-slot rings; CORS stations beat-driven (onset 0.05, cooldown 350ms, sentinel init); non-CORS 5s autonomous pulse |
| Volume slider | ✅ Done — bottom-right, co-located with play/pause; 44px touch target; persists across station transitions; hidden on touch/mobile (device volume controls suffice) |
| SEO foundations | ✅ Done — keyword-rich meta description and OG/Twitter tags; canonical link; JSON-LD WebApplication schema; robots.txt; sitemap.xml; noscript static content for crawlers |
| og:image (social preview card) | ❌ Not started — 1200×630 branded image for link previews on Twitter/Slack/iMessage; high impact for social sharing; requires design asset |
| Analytics instrumentation | ✅ Done — `analytics.ts` wrapper (`trackEvent`); CF Web Analytics beacon auto-injected via Cloudflare Pages dashboard (already enabled); 5 events: `playback_started` (once per session on first play), `playback_duration` (seconds on every navigation away), `station_skipped` (forward navigations), `station_saved`, `genre_seed_completed` (with genre count); console.log in dev, CF beacon API in prod |
| Ambient renderer improvements | ✅ Done — speed 0.30, turbulence r*1.6, brightness floor +0.12, vignette softened, station colors full saturation |
| Station pre-loading | ✅ Done — preload logic lives inside `useAudioLifecycle`; next station pre-loaded immediately after each station commit; 30s expiry to save bandwidth; re-primes on first pointer activity after expiry; `consume()` in `goNext` skips stream-wait for instant crossfade on forward navigation; desktop: pre-plays at volume 0 (buffer + AudioContext warm); mobile: preconnect() only — play() is deferred to goNext() inside the swipe gesture to prevent iOS audio leakage |
| Build SHA fingerprint | ✅ Done — `CF_PAGES_COMMIT_SHA` injected at build time via Vite `define`; shown in Settings modal footer (faint, selectable) and logged to browser console on startup; `'local'` fallback in dev |
| Search stations | ✅ Done — search icon in bottom-right controls; `S` key shortcut (desktop); multi-word search (splits on whitespace, all words must match); arrow keys navigate results, Enter plays; centered panel on desktop (480px), fullscreen on mobile |
| Filter stations | ✅ Done — filter icon in bottom-right controls; `F` key shortcut (desktop); genre chip picker + country dropdown; arrow keys navigate genre chips (Left/Right), Enter/Space toggles; centered panel on desktop (400px), fullscreen on mobile; active filter persists until cleared |
| MediaSession API | ✅ Done — `useMediaSession` hook sets `navigator.mediaSession.metadata` (title, artist = country · genre, artwork = favicon) on station change; registers play/pause/nexttrack/previoustrack handlers; updates `playbackState` on pause/resume; wired in App.tsx |
| Genre seed on first launch | ✅ Done — `GenreSeed` component shown on first visit (localStorage `genre_seed_shown` guard); 8 genre chips (Ambient/Chill · Classical · Electronic/Dance · Hip-Hop/R&B · Rock · Jazz · Latin · Pop); max 3 selections; × to skip; "Start" button to confirm; calls `seedGenres()` in `useRecommendation` which boosts tag scores +1.0 per genre tag; shown only on idle phase |
| Saved shelf improvements | ✅ Done — animated equalizer on playing station row; sort chips (Saved · A–Z · Country · Genre, persisted to localStorage); drag-to-reorder in Saved mode (HTML5 DnD desktop + touch); ArrowUp/Down opens/closes shelf; 1–0 keys play first 10 stations with visible number badges (desktop only) |
| Listening stats | ✅ Done — `totalListenSeconds` added to TasteProfile (zero-safe for existing profiles), accumulated in `onNavigatingAway`; `getStats()` in `useRecommendation` derives top genres + top countries from existing positive scores + total time + interaction count; "Your listening" section in Settings modal shows time / stations / top genres / top countries when `interactionCount > 0` |
| Sleep timer | ❌ Not started — 15 / 30 / 60 min picker in Settings; fades out and pauses after the selected duration; timer cancelled on manual pause |
| HQ-only filter toggle | ❌ Not started — Settings toggle to restrict catalog to ≥320kbps streams; currently only the HQ badge exists; ~5% of catalog qualifies |
| No-connection overlay | ✅ Done — `isOffline` state in `useAudioLifecycle` (init from `navigator.onLine`, updated by `online`/`offline` events); overlay shows "No connection / Will resume automatically" centered on screen; clears automatically on `online` event which also resumes playback |
| Persistent listening history | ❌ Not started — keep last 20–30 stations across sessions in IDB so users can revisit stations heard but not saved; in-memory session history already exists, just needs an IDB write on each commit and a read path on startup |
| Failing saved station indicator | ❌ Not started — ⚠ badge on saved station rows that failed to connect after 3 retries; clears automatically on successful retry |
| First-station quality boost | ✅ Done — `qualityBoostFilter` in `useAudioLifecycle` restricts the first 3 station picks per session to stations with `favicon_url + name + country + bitrate ≥ 128`; tracked via `qualityRemainingRef` (starts at 3, decrements on commit); applied in `prime` and `loadInitial`; falls back to unfiltered pool if no quality stations match |
| Locale-aware cold start | ✅ Done — `detectLocaleCountry()` in `utils.ts` maps device timezone (via `Intl.DateTimeFormat`) to a catalog country fragment (~30 IANA zones, prefix fallback for `America/` and `Australia/`); `localeBiasFilter()` narrows the pool to locale-matching stations for the first 3 picks, falls back to full pool if fewer than 10 local stations match; wired alongside `qualityBoostFilter` in `prime` and `loadInitial` |
| Audio-reactive visualization (activate) | 🔭 Ready to activate — code fully built in VisualizationManager; CORS stations (73% of catalog) can switch to the 8000-particle + beat-driven ripple mode; needs a UX decision: default-on for CORS stations, or opt-in via Settings toggle |
| Station blocking | ❌ Not started — "never play this station" action (long-press or button); adds station UUID to a blocked list in IDB; excluded from `pickNext` permanently; distinct from a skip signal |
| Visualization themes | ❌ Not started — Minimal (single clean ring, no particles or ripples), Aurora (flowing horizontal light bands driven by bass/mid), Neon (sharp electric ring + dense particle field), Alchemy (domain-warped plasma); iOS Metal implementations exist as reference for the GLSL ports; free on web (no Plus paywall) |

---

## 18. Pre-launch roadmap

Features to complete before posting to Reddit / driving external traffic. Ordered by priority.

### Must-have before posting

| Feature | Why |
|---|---|
| og:image | Reddit and social link previews are blank without it — single biggest conversion factor for a click |

### High value, ship before or shortly after posting

| Feature | Why |
|---|---|
| Audio-reactive visualization (activate) | Core differentiator — 73% of catalog supports it and the code is already written |
| Persistent listening history | "I heard something great but didn't save it" is a predictable early complaint |

### Post-launch, based on feedback

| Feature | Notes |
|---|---|
| Sleep timer | Common radio app request; straightforward to build |
| HQ-only filter toggle | Niche but vocal audience (audiophiles) |
| Failing saved station indicator | Reduces confusion when a saved station goes offline |
| Station blocking | Will be one of the most common early requests (ad-heavy stations, jarring genres) |
| Visualization themes | Significant work; adds visual variety and replayability |
