---
kind: error_handling
name: Defensive Browser-API Error Handling in Single-Page Visualization
category: error_handling
scope:
    - '**'
source_files:
    - Alg_Virt_WhLL.html
---

## Overview

The repository is a single-file interactive visualization (`Alg_Virt_WhLL.html`) — no server, no framework, no build step. Error handling is therefore entirely client-side JavaScript inside one `<script>` block, using plain `try/catch`, `console.warn`, and early-return guards rather than any error type system, middleware, or centralized error handler.

## Patterns Observed

### 1. Defensive null-checks (early returns)
Most functions that touch the DOM or shared state begin with a guard:
- `updateDerivedQuality`: `const box = document.getElementById('derivedQuality'); if (!box) return;`
- `rebuildRingSpeedControls`: `const sel = document.getElementById('ringSpeedSelect'); if (!sel) return;`
- `updateRingSpeedDisplay`: `if (!sel || !valueEl) return;`
- `drawCogwheel`: `const layers = Object.keys(layerMap).sort(...); if (layers.length === 0) return;`
- `updateMathOutput`: `const node = valueAlgebra[componentName]; if (!node) { document.getElementById('mathOutput').textContent = 'ERROR: Component not found.'; return; }`
- `getOrderedDependencies` / `resolveDeps`: `if (!node) return [];`

This is the dominant pattern: instead of throwing, functions silently bail out when required DOM nodes or data entries are absent.

### 2. Try/catch around browser APIs that may be unavailable
Two locations wrap `localStorage` access in `try/catch` with the comment `/* storage unavailable */`:
```js
try { localStorage.setItem('avw-theme', currentTheme); } catch (err) { /* storage unavailable */ }
```
and
```js
try { saved = localStorage.getItem('avw-theme'); } catch (err) { /* storage unavailable */ }
```
The caught errors are swallowed — no logging, no fallback UI beyond defaulting to the `'spaceship'` theme.

### 3. `console.warn` for rejected promises
Fullscreen API calls use `.catch(console.warn(...))`:
```js
panel.requestFullscreen().catch(err => console.warn('Fullscreen failed:', err));
```
The legacy `webkitRequestFullscreen` path has no `.catch`, so its rejection would propagate as an unhandled promise rejection.

### 4. No custom error types, sentinel values, or error propagation
There are no `Error` subclasses, no error codes, no result/option-like wrapper, and no central error dispatcher. Errors either:
- Do not surface (swallowed by `try/catch`),
- Are logged via `console.warn`,
- Or cause a silent no-op via early return.

### 5. No `throw` statements anywhere in the script
A grep for `throw` across the file finds none. The only user-visible failure text is the hard-coded string `'ERROR: Component not found.'` rendered into the telemetry readout when `valueAlgebra[componentName]` is falsy.

### 6. No `panic`/`recover` equivalent
As a browser script there is no Go-style panic/recover. There is no `try/catch` wrapping entire initialization blocks either — initialization runs top-down without a global error boundary.

### 7. Input validation via clamping, not error signaling
User-driven numeric inputs are constrained in-place rather than reported as errors:
- Ring speed: `Math.max(-RING_SPEED_MAX, Math.min(RING_SPEED_MAX, ...))` where `RING_SPEED_MAX = 3.0`
- Zoom: `zoomFactor = Math.min(zoomFactor * 1.15, 2.0)` / `Math.max(zoomFactor * 0.85, 0.4)`
- Stack rotation: `stackRotX = Math.max(5, Math.min(85, stackRotX - dy * 0.5))`
- Fill level: `Math.max(0, Math.min(1, pointerFill))`

These clampers prevent invalid state but do not produce error messages.

### 8. Theme lookup defaults safely
```js
function getThemePalette() { return THEMES[currentTheme] || THEMES.spaceship; }
function applyTheme(themeId) {
    if (!THEMES[themeId]) themeId = 'spaceship';
    ...
}
```
Unknown themes fall back to `'spaceship'` rather than throwing.

## Key Files

- `Alg_Virt_WhLL.html` — the sole source file; all error-handling logic lives here.

## Conventions & Constraints

- **Observed convention**: Functions that depend on DOM elements perform an explicit existence check and return early if the element is missing, rather than propagating an error upward.
- **Observed convention**: Asynchronous browser APIs (`requestFullscreen`) reject with a warning logged to the console; synchronous APIs (`localStorage`) are wrapped in `try/catch` and silently ignored on failure.
- **Constraint enforced by code shape**: Because the project is a single HTML file with no module system, there is no possibility of cross-module error propagation — all error handling is local to the script scope.
- **No documented rule** was found stating that errors must be handled in a particular way; the conventions above are descriptive of what the code does, not prescribed by documentation.