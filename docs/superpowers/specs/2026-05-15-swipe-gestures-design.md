# Swipe Gesture Recognition — Design Spec

**Date:** 2026-05-15
**Status:** Approved

---

## Overview

Add touch swipe gesture recognition to the web app so mobile users can navigate naturally without relying solely on the edge chevron buttons. Swipes are additive — chevrons remain the primary visible affordance and continue to work unchanged.

---

## Gesture Mapping

| Swipe direction | Action |
|---|---|
| Left | Next station |
| Right | Previous station |
| Up | Open saved shelf |
| Down | Save station |

Same actions as keyboard arrows and chevron buttons. Fires the same chevron flash animation as keyboard presses for visual confirmation.

---

## Implementation

### Hook: `useSwipeGesture`

A custom hook consistent with the existing `useNavigation` pattern for keyboard handling. Accepts the four action callbacks and a ref to the container element.

**Event flow:**

1. `pointerdown` — record start position (`clientX`, `clientY`); call `setPointerCapture` to track the pointer if it leaves the element; ignore if `event.isPrimary === false` (multi-touch / pinch-to-zoom)
2. `pointermove` — track current delta; once vertical intent is clear (`|deltaY| > |deltaX|` and `|deltaY| > 10px`), call `preventDefault()` to prevent page scroll
3. `pointerup` — if `Math.max(|deltaX|, |deltaY|) >= 50px`, determine direction by whichever axis is larger, fire the corresponding action; reset state

**Edge cases:**
- `pointercancel` — reset state without firing
- Left edge dead zone: ignore `pointerdown` starting within 20px of the left edge to avoid conflicting with iOS Safari's back-swipe navigation gesture
- Multi-touch guard: skip if `event.isPrimary === false`

### Attachment

Hook attaches listeners to the full-screen root container (the same element that already receives keyboard focus). No changes to `NavControls`, `useNavigation`, or the action callbacks in `App.tsx`.

---

## What does not change

- Chevron buttons remain visible and fully functional
- Keyboard navigation unchanged
- No tutorial or hint overlay — chevrons already teach the interaction model; swipes are discoverable

---

## Testing

- Unit tests for `useSwipeGesture`: threshold not met → no action; left edge dead zone → no action; multi-touch → no action; each of the four directions fires the correct callback
- Pointer event simulation via `@testing-library/user-event` or manual `dispatchEvent`
