# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

"Constellation · Focus" — a single-page, zero-build, static PWA Pomodoro timer. The user focuses, earns stars, and grows a shared constellation that persists in `localStorage`. Deployed to Vercel as static files.

The entire app lives in **`index.html`** (~3,500 lines: inline `<style>` + inline `<script>` IIFE). There is no bundler, no framework, no package manager, no test suite, and no `package.json`. Treat `index.html` as the codebase.

## Running / "build" / deploy

- **Local dev (Windows):** open `index.html` directly in a browser, or run `Open Focus App.bat`. There is no dev server — edits to `index.html` are visible on browser refresh.
- **Deploy:** Vercel serves the directory as-is. `.vercelignore` excludes `*.bat`. Pushing to `main` triggers deploy.
- **No tests, no lint, no typecheck** exist. Verify changes by loading the page and exercising the affected flows in the browser (timer, place-stars modal, settings, tasks, heatmap).

### Service worker cache busting

`sw.js` uses a constant cache name (`grove-v3` at time of writing) and `cache-first` for everything in `ASSETS`. **When you change `index.html`, `manifest.webmanifest`, or `og.svg`, bump the `CACHE` constant**, otherwise returning users will keep seeing the old cached page until they hard-reload. The `activate` handler deletes old caches by name.

### Design system

The app commits to a "celestial observatory" aesthetic — purple-navy with rare champagne-gold (`--gold`, `--gold-soft`) reserved for the current Pomodoro dot, achievement chrome, and the place-stars credit counter. **Don't spray gold across the UI** — its scarcity is the point.

Typography (loaded from Google Fonts in the `<head>`):
- `--font-display` Fraunces (variable, used italic via `font-variation-settings:"opsz" 144` for the timer, headings, stat numerals)
- `--font-serif` Instrument Serif (italic body emphasis: empty states, hints, footer)
- `--font-body` Instrument Sans (general UI)
- `--font-mono` JetBrains Mono (numerics, kbd, coordinate labels)

Card titles are styled as small-caps eyebrows (`text-transform:uppercase; letter-spacing:.24em; font-size:10.5px`). Numerals use italic Fraunces with `font-variant-numeric:tabular-nums`. The constellation rendering (`renderSkySVG`) emits its own SVG `<defs>` for `starCore`, `softGlow`, `bigGlow`, `nebulaBlur` — IDs are intentionally shared between the multiple SVGs (skySvg, treeSvg, placeSvg); each SVG resolves `url(#…)` against its own subtree first, so duplicates are safe.

## Architecture

### State and persistence

- All state is held in one in-memory object `state`, persisted to `localStorage` under the key **`grove.v2`** via `save()` after every mutation.
- `defaults` (index.html:1574) is the source of truth for shape. `load()` merges stored data over `structuredClone(defaults)` so new fields appear automatically for existing users.
- **Schema migrations live inside `load()`** (index.html:1604). Example: legacy `style: 'tree:*' | 'crystals:*'` is force-rewritten to `'stars:auto'`; `settings.species` / `settings.dayNight` are deleted. When you remove or rename a setting, add a migration here — do not assume stored data matches `defaults`.
- Keys to know in `state`: `settings`, `stats`, `forest` (legacy session log, still used for daily/heatmap/achievements), `sky` (`{stars, edges}` — the constellation graph), `pendingStars`, `tasks`, `activeTaskId`, `pomodorosThisCycle`, `achievements`, `daily` (date → minutes), `soundMix`.

### Timer model

The timer is driven by **wall-clock timestamps**, not interval counting, so it survives tab-hidden / sleep:

- `start()` sets `endAt = Date.now() + remaining`; `tick()` recomputes `remaining = endAt - Date.now()` each frame.
- `visibilitychange` handler resyncs `remaining` from `endAt` on tab-resume and calls `complete()` if already elapsed.
- Pausing tracks `pausedAt` + `pausedTotal`. `wiltFraction()` uses `pausedTotal` minus `settings.pauseGrace` seconds to decide whether the session "wilts" (reduces stars earned).
- `mode` is `'focus' | 'short' | 'long'`. `setMode()` / `nextMode()` advance the Pomodoro cycle using `state.pomodorosThisCycle` against `settings.cycle`.

### Constellation rendering

- `state.sky.stars` is an array of `{id, x, y, palette, type, ts}`; `state.sky.edges` connects star ids. Coordinates are in a virtual `SKY_W × SKY_H` = `1000 × 600` viewBox, scaled responsively by SVG.
- `renderSkySVG(sky, opts)` (index.html:1825) is the single SVG renderer used in both the main scene and the place-stars modal. `opts.highlightLastN` highlights freshly placed stars; `opts.pulseAt` adds a preview pulse.
- New stars connect to their nearest neighbor (`nearestStarTo`), producing the constellation edges deterministically.
- Background palette is driven by `settings.bgFocus` / `bgBreak` / `bgLong` looked up in `BG_PALETTES` (index.html:1564) and applied via CSS custom properties on `.scene-bg`. `'auto'` lets the time-of-day logic in `applyDayNight()` pick.

### DOM access pattern

- All DOM lookups are cached once at script start in the `els` object (index.html:1647). Add new elements there rather than calling `document.querySelector` ad-hoc, and use the existing `$` / `$$` helpers for one-offs.
- The script is wrapped in an IIFE — there are no module exports, and nothing is on `window`. New code goes inside the IIFE.

### Keyboard shortcuts

Bound globally in the `keydown` handler (index.html:3398). Shortcuts are suppressed while typing in `INPUT/SELECT/TEXTAREA` or while any modal is open (`anyModalOpen()`). Current bindings: Space (start/pause), R (reset), S (skip), F (focus mode), D (distraction), ? (help), 1/2/3 (mode), T (focus task input).

## Conventions worth following

- **Mutate `state` then call `save()` and the relevant `render*()` function.** There is no reactivity layer. The render functions are cheap and idempotent — re-render liberally rather than trying to patch the DOM by hand.
- **Don't introduce a build step or dependencies** unless explicitly asked. The "open the .html file" workflow is intentional and the Vercel deploy depends on it.
- **Be conservative with `defaults` changes.** Removing a key requires a migration in `load()`; renaming a key without one will silently drop user data on next save.
