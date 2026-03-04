# Race Clock Fade In/Out — Implementation Plan

Machine-readable plan for implementing race clock visibility (invisible → fade in when clock runs → fade out when all skaters have results) across all `broadcast_race_overlay*` views.

---

## 1. Scope

### Views to modify (each has `template.html` and `styles.css`)

| View folder |
|-------------|
| `broadcast_race_overlay` |
| `broadcast_race_overlay_simple` |
| `broadcast_race_overlay_splits` |
| `broadcast_race_overlay_splits_bell` |
| `broadcast_race_overlay_affiliation` |
| `broadcast_race_overlay_relay` |

Optional (if same behavior desired in tests): `*/test.html`, `*/standalone_test.html`.

---

## 2. Behavior specification

| Phase | Condition | Action |
|-------|-----------|--------|
| **Initial** | Page load / race not started | Clock invisible (`opacity: 0`). |
| **Fade in** | First time `data.currentTime > 0` and not yet faded out | Add class so clock becomes visible (fade in via CSS transition). |
| **Fade out** | "All skaters have result" (see 2.1) | Remove visible class so clock fades to invisible. |
| **Reset** | `data.status === 'NotStarted'` or `data.status === 0` | Reset state; ensure clock is invisible so next race can fade in again. |

### 2.1 "All skaters have result"

- **Preferred (simple):** `data.status === 'Finished' || data.status === 3`
- **Alternative (strict):** `activeRacers.length > 0 && activeRacers.every(r => hasResult(r))` where `hasResult(r)` is defined per API (e.g. `r.cumulativeSplitTime != null` or `r.finished`).

Use the simple rule unless the API can report Finished before all skater times are in.

---

## 3. CSS changes (per view `styles.css`)

Apply to the existing `.race-clock` rule block (and any `@media` overrides that touch `.race-clock`).

### 3.1 Default state (invisible)

- Set: `opacity: 0`
- Optional: `visibility: hidden` and/or `pointer-events: none` if layout/click-through is a concern.

### 3.2 Visible state (new rule)

- Selector: `.race-clock.visible`
- Set: `opacity: 1`
- If you added `visibility`/`pointer-events`, restore them here.

### 3.3 Transition

- On `.race-clock`: ensure `transition` includes `opacity` (e.g. `transition: opacity 0.3s ease` or keep existing `transition: all 0.2s ease-in-out`).

---

## 4. JavaScript logic (per view `template.html`)

### 4.1 State variables (script scope, once per view)

```js
let raceClockFadedIn = false;
let raceClockFadedOut = false;
```

### 4.2 Where to run logic

Inside the existing `updateRaceData` callback (the function passed to `WebLynx.updateRaceData`), after the line that sets `race-clock` text content, e.g. after:

`document.getElementById('race-clock').textContent = WebLynx.formatRaceTime(data.currentTime);`

### 4.3 Logic (pseudocode)

```
clockEl = document.getElementById('race-clock')

// Reset when race not started
if (data.status === 'NotStarted' || data.status === 0) {
  raceClockFadedIn = false
  raceClockFadedOut = false
  clockEl.classList.remove('visible')
  return  // or continue to rest of callback
}

// Fade out when all skaters have result
if (data.status === 'Finished' || data.status === 3) {
  raceClockFadedOut = true
  clockEl.classList.remove('visible')
  // continue to rest of callback (update text, etc.)
}

// Fade in when clock is running and not yet faded out
if (!raceClockFadedOut && data.currentTime > 0) {
  raceClockFadedIn = true
  clockEl.classList.add('visible')
}
```

### 4.4 DOM

- No structural change. Same element `id="race-clock"`; only add/remove class `visible`.

---

## 5. Edge cases (implementer should handle)

| Case | Handling |
|------|----------|
| Same overlay, multiple races in sequence | Reset on `NotStarted` / `0` so next race starts with clock invisible. |
| Relay or other race types | Same rules: fade in on `currentTime > 0`, fade out on Finished (or equivalent status). |
| Bell lap / laps-to-go (splits_bell) | Only change race clock visibility; do not tie bell-lap container to this logic unless specified. |

---

## 6. Implementation checklist (per view)

- [ ] **CSS:** `.race-clock` default `opacity: 0`; add `.race-clock.visible { opacity: 1 }`; ensure `transition` on `.race-clock` for opacity.
- [ ] **JS:** Add state variables `raceClockFadedIn`, `raceClockFadedOut`.
- [ ] **JS:** In `updateRaceData` callback, after updating race-clock text: add reset branch (NotStarted → clear state and remove `visible`), fade-out branch (Finished → set fadedOut, remove `visible`), fade-in branch (currentTime > 0 and !fadedOut → add `visible`).
- [ ] **JS:** Use same element `#race-clock`; no new nodes.

Repeat for: `broadcast_race_overlay`, `broadcast_race_overlay_simple`, `broadcast_race_overlay_splits`, `broadcast_race_overlay_splits_bell`, `broadcast_race_overlay_affiliation`, `broadcast_race_overlay_relay`.
