# First-Run Gesture Tutorial — Design Spec

**Created:** 2026-05-15  
**Status:** Approved  
**Web design spec:** §15

---

## What it is

A one-time overlay that teaches gesture navigation immediately after the first station starts playing. Five directional labels arranged in a cross pattern around screen center, adapting to the user's input method.

---

## Layout

```
           ↑  favorites


← next    •  hold    → prev


           ↓  save
```

Five absolutely-positioned text labels inside a fixed full-screen container. The container is centered; each label is offset from center using `position: absolute` + `transform: translate`.

**Typography:** system-ui, 12px, weight 300, letter-spacing 0.15em, `rgba(255,255,255,0.55)` — low-opacity so the visualization remains the dominant visual.

---

## Input-method adaptation

Detected once at mount via `navigator.maxTouchPoints > 0`.

| Position | Touch label | Desktop label |
|---|---|---|
| Left | `swipe ←  next` | `←  next` |
| Right | `swipe →  prev` | `→  prev` |
| Up | `swipe ↑  favorites` | `↑  favorites` |
| Down | `swipe ↓  save` | `↓  save` |
| Center | `hold` | `hold` |

The hold hint is a placeholder — same opacity as the others, no action description, shown regardless of input method.

---

## Behaviour

| Event | Action |
|---|---|
| `phase` becomes `'playing'` for the first time | Schedule show after 2s |
| 2s elapses (no dismiss yet) | Fade in labels |
| Any pointer gesture or keypress while visible | Dismiss immediately |
| 8s after appearing (no gesture) | Auto-dismiss |
| Dismiss (any cause) | Fade out; write `tutorialSeen=1` to localStorage |
| Mount with `tutorialSeen` already set | Never show |

Fade in/out: CSS `opacity` transition, 0.4s ease.

The tutorial never shows again once dismissed, regardless of reason. No way to re-trigger in the UI (dev can clear localStorage to reset).

---

## Architecture

### `useFirstRunTutorial.ts`

Manages timing and seen state. Returns `{ visible, dismiss }`.

```ts
function useFirstRunTutorial(phase: Phase): { visible: boolean; dismiss: () => void }
```

- On mount: reads `localStorage.getItem('tutorialSeen')`. If set, returns `{ visible: false, dismiss: noop }` permanently.
- When `phase === 'playing'` and not yet seen: starts a 2s timer. If the timer fires, sets `visible = true` and starts an 8s auto-dismiss timer.
- `dismiss()`: clears both timers, sets `visible = false`, writes to localStorage.
- Cleans up all timers on unmount.

### `GestureTutorial.tsx`

Renders the five labels. Props: `{ visible: boolean; onDismiss: () => void; isTouch: boolean }`.

- When `visible` is false, renders nothing (or `opacity: 0` with `pointerEvents: none` to allow CSS transition out).
- `isTouch` selects the label text set.
- `pointer-events: none` on the container so gestures pass through to the canvas.

### App.tsx changes

- Compute `isTouch` once: `const isTouch = useRef(navigator.maxTouchPoints > 0).current`
- Call `useFirstRunTutorial(phase)` → `{ tutorialVisible, dismissTutorial }`
- Pass `dismissTutorial` into `useNavigation` and `useSwipeGesture` so any gesture/keypress dismisses it
- Render `<GestureTutorial visible={tutorialVisible} onDismiss={dismissTutorial} isTouch={isTouch} />` above the canvas overlay

---

## What is not in scope

- Re-trigger UI (no settings toggle to show tutorial again)
- Per-gesture staged reveal (all five labels appear together)
- Animations beyond opacity fade
