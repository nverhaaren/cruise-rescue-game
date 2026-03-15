# Pichu Rescue Cruise — Structure Summary

A foundation document describing the current codebase for future development.

---

## Overall Summary

The entire game is a **single self-contained HTML file** (`pichu-rescue-cruise.html`) with no dependencies beyond Google Fonts. It uses vanilla HTML, CSS, and JavaScript — no frameworks, no build step, no external scripts.

The file is organized in three sections:
1. **`<style>`** — all CSS, including layout, animations, and component styles (~400 lines)
2. **`<body>`** — static HTML shell with four screen `<div>`s and a persistent header
3. **`<script>`** — all game logic: constants, state, helpers, rendering functions, and event wiring (~400 lines)

The HTML body is mostly a skeleton; nearly all visible content is injected by JavaScript at runtime.

---

## Gameplay Loop

The game follows a repeating four-phase cycle:

```
[1. Cruise] → [2. Summary] → [3. Care] → [4. Upgrade] → back to [1. Cruise]
```

**Phase 1 — Cruise (Set Sail)**
- Player clicks "Set Sail!" to begin a rescue voyage.
- `buildRoute()` generates 3–6 stops (scaling with cruise number), each with a 70% chance of containing a Pichu.
- A `setInterval` loop (1.2s per step) moves the ship across the route, visiting each stop.
- At each stop: if a Pichu is present, `makePichu()` creates one, adds it to `state.pichus`, increments `state.totalRescued`, and awards +1 star.
- When all stops are visited, `finishCruise()` fires.

**Phase 2 — Summary**
- A brief interstitial screen showing the rescued Pichus and a narrative summary.
- Player clicks "Visit Upgrade Shop" to continue — but this actually navigates to the Care screen first.

**Phase 3 — Care (Aboard the Ship)**
- Player selects a Pichu from the roster, then chooses an activity.
- Each activity modifies the Pichu's stats (HP, Happy, Strength) and awards +1 star.
- Player clicks "End Cruise & Dock" when done.
  - Bonus stars are awarded: +3 per Pichu with average stats ≥ 70, +1 per Pichu with average ≥ 50.
  - All Pichus are cleared ("graduated"), cruise number increments, and the game moves to the Upgrade screen.

**Phase 4 — Upgrade Shop**
- Player spends stars on permanent ship upgrades that unlock new activities or enhance existing ones.
- Player clicks "Start Next Cruise!" to return to Phase 1.

---

## State and Action Representation

### State

All mutable game data lives in a single top-level `state` object:

```js
const state = {
  stars: 5,             // currency; earned via rescues, care, and end-of-cruise bonuses
  cruiseNum: 1,         // increments at the end of each cruise
  totalRescued: 0,      // lifetime Pichu count
  pichus: [],           // Pichu objects currently aboard the ship
  selectedPichu: null,  // which Pichu is active in the care screen
  upgrades: {},         // map of upgrade id → true for owned upgrades
  sailStep: 0,          // index of the current stop being visited
  stops: [],            // route stop objects for the current cruise
  cruiseLog: [],        // array of narrative log strings shown during sailing
};
```

There is no persistence — state resets on page refresh.

**Pichu objects** are plain JS objects created by `makePichu()`:
```js
{ id, name, hp, happy, strength, emoji }
```
Stats are integers 0–100, clamped by activity effects via `Math.min(100, ...)`.

**Stop objects** are created by `buildRoute()`:
```js
{ pos, hasPichu, visited, pichu }
```
`pos` is a percentage from the left edge of the route track (drawn from `STOP_POSITIONS = [8, 22, 38, 55, 72, 88]`).

**Upgrade and activity definitions** are static arrays (`UPGRADES_DEF`, `ACTIVITIES_DEF`) defined as constants — they never change at runtime. Upgrade ownership is recorded in `state.upgrades`; activity availability is derived from `state.upgrades` at render time.

### Actions

There is no action queue or dispatcher. Actions are direct imperative function calls triggered by DOM event listeners.

**Event wiring** is done at the bottom of the script with simple `element.onclick = ...` assignments for the four main buttons:
- `#btn-sail` → `startSail()`
- `#btn-end-cruise` → inline handler (awards bonuses, clears pichus, increments cruise)
- `#btn-go-upgrade` → navigates to care screen, calls `renderCareScreen()`
- `#btn-next-cruise` → navigates to cruise screen

**Dynamic elements** (Pichu cards, activity buttons, upgrade cards) get their `onclick` handlers set during each render pass inside `renderPichuList()`, `renderActivities()`, and `renderUpgradeScreen()`.

The core action handler for care activities is `doActivity(act, event)`:
1. Guards: requires a selected Pichu
2. Captures before-state
3. Calls `act.effect(pichu)` — a plain function that mutates the Pichu's stats
4. Computes gain, shows float-up text and a toast message
5. Awards +1 star, calls `updateHeader()` and `renderPichuList()`

### Interface / Rendering

**Screen management:** Four `<div class="screen">` elements are toggled with a single `.active` class via `showScreen(id)`. Only the active screen is visible (`display: flex` vs `display: none`).

**The header** (`#header`) is persistent across all screens and is updated after every state-changing action via `updateHeader()`, which writes directly to three `<span>` elements.

**Rendering is manual and pull-based:** There is no reactive data layer. After any state mutation, the relevant render function(s) must be called explicitly. The render functions (`renderPichuList`, `renderActivities`, `renderRoute`, `renderUpgradeScreen`, `renderLog`) each clear their container's `innerHTML` and rebuild it from scratch.

**Feedback mechanisms:**
- `showToast(msg)` — a fixed-position notification that slides in and auto-dismisses after 2s
- `floatText(txt, color, x, y)` — a transient DOM element that animates upward from the clicked button's position, used for stat gains
- `setMsg(screen, txt)` — writes to the `#msg-box` element within the active screen
