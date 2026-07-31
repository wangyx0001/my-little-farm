# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-file, self-contained HTML5 canvas farm game (`index.html`) styled after Homa Games' "Farm Land": run a farmer around an island, harvest wheat/wood/eggs/berries/milk, sell them at a stand, and spend coins to unlock upgrades and a second island. It's a gift project for a child — no build tooling, no dependencies, no package.json. Everything (HTML, CSS, JS, and even the art) lives in `index.html` and is drawn procedurally on Canvas 2D; there are no external image/audio/font assets.

`apple-touch-icon.png` and `icon-512.png` are standalone copies of the home-screen icon also embedded as a base64 data URI directly in `index.html`'s `<link rel="apple-touch-icon">` — they exist for convenience if the folder is ever served as static files, but `index.html` does not reference them.

## Running / testing

There is no build step. To preview:

```
# open directly
start "C:\Users\wangy\Claude\Projects\Farm game\index.html"

# or serve it (useful for testing the "embedded in an iframe" code path)
python -m http.server 8123
```

Add `?reset` to the URL to wipe the saved farm (`localStorage['myLittleFarm.v1']`) and see the first-run name prompt again.

There are no automated tests, linter, or formatter configured for this project. Verify changes by opening the file in a browser and checking the console for errors, and by exercising the relevant mechanic (harvest a plot, sell, buy an upgrade, resize the window, reload to confirm the save round-trips).

## Deployment model — read before touching sizing/meta code

This file is deployed two ways simultaneously, and both must keep working:

1. **Standalone** — opened directly as a file, or added to an iOS home screen. `window.self === window.top`.
2. **Embedded** — pasted into a Claude "Artifact" and rendered inside an iframe whose *height the host derives from this page's own content height*. `window.self !== window.top`.

`EMBED` (near the top of the script) detects which mode is active and adds an `html.solo` class in standalone mode. `resize()` branches on `EMBED`:
- Standalone: sizes off `window.visualViewport` (excludes the iOS Safari toolbar so the game doesn't get clipped under it — plain `window.innerHeight` does not).
- Embedded: sizes off `document.documentElement.clientWidth`/`clientHeight` only — **never** the viewport — because reading the viewport there would feed the host's own output back in as input and the layout would never converge.

Everything is sized through one `#stage` div whose pixel `width`/`height` JS sets explicitly (not CSS `100%`, which is either wrong on iOS or circular when embedded). The `<canvas>` also needs an explicit CSS `width`/`height` set alongside its pixel `width`/`height` attributes, or the browser lays it out at the attribute size (`dpr`× too large) instead of the CSS size.

`resize()` is called directly on every resize/orientation/visualViewport event and additionally self-heals every 15 frames from the render loop (`frame()` near the bottom) — deliberately *not* coalesced through `requestAnimationFrame`, because rAF is throttled in hidden/offscreen iframes and a "resize already queued" flag would latch forever if that callback never fires, freezing the game at the wrong size.

If you change anything under the `resize()`/`EMBED`/`#stage` code, test at minimum: standalone in a normal browser tab, standalone with the URL bar hidden (mobile), and embedded in an iframe at a few widths — a regression here tends to only show up in one of the two deployment modes.

## Publishing

When this file is published as a Claude Artifact for sharing, publishing a new version does **not** automatically update what public link visitors see — Claude pins a public share to a specific version, and switching that pin to "Latest" is blocked while the artifact is shared publicly ("Change who has access first"). After republishing, the shared version must be manually re-pinned to the new version in the Artifact's Share dialog, or the public link keeps serving the old build.

## Code architecture

Single `<script>` block, organized top-to-bottom as: helpers → resize/EMBED → Audio → constants → state → save/load → world entities → input → HUD → update loop → render loop → boot. Section banners (`/* ---------------- Name ---------------- */`) mark these; use them to navigate rather than searching for individual functions.

**World layout is coordinate-based, not a grid.** `MAIN`, `FOREST`, `BRIDGE`, `CHICKEN_PEN`, `COW_PEN` are all `{x, y, w, h}` rects in one shared `WORLD` space (1700×1000). `isWalkable(x, y)` just checks whether a point falls inside one of those rects (bridge only counts once unlocked). Fixed-position content (`PLOT_POS`, `TREE_POS_MAIN`, `TREE_POS_FOREST`, `BUSH_POS`, `SELL_STAND`, `SELL_ZONE`) are plain coordinate constants inside those rects — there's no tilemap or spatial index. Adding a new resource node means picking coordinates by eye and adding an entry to the relevant `*_POS` array.

**Upgrades are data-driven via `ZONES`.** Each entry (`{id, x, y, r, cost, label, icon, vis}`) is a circle on the ground the player stands in to gradually pay off; `vis(unlocks)` gates when it appears (e.g. `pack2`/`cowpen`/`gold` require `u.bridge`). `S.un[id]` (unlocked) and `S.paid[id]` (progress) track state; `unlock(z)` in the update loop applies the one-time side effect for that specific id (spawn a plot/chicken/cow, show a floater) via an `if (z.id === '...')` chain — there's no separate effect-registration system, so a new upgrade needs both a `ZONES` entry and a branch in `unlock()`.

**Entities are flat arrays of plain objects**, not classes: `trees`, `plots`, `bushes`, `chickens`, `cows`, plus transient ones — `drops` (items sitting on the ground), `flyers` (items animating from source to backpack or from backpack to the sell stand), `floaters` (rising text), `parts`/`coinParts` (particle effects). All are mutated in place inside `update(dt)`.

**Rendering is immediate-mode with manual Y-sorting for depth**, redone every frame in `render()`: static ground/UI layers draw first, then movable entities (trees, bushes, animals, drops, fences, the player) are collected into one `drawables` array, sorted by their `y` anchor, and drawn back-to-front so higher-on-screen things are correctly occluded by lower ones. Truly foreground effects (flying items, particles, floating text, the joystick) draw after that sort, unaffected by depth. `draw*` functions never mutate state — they only read from the entity objects and `ctx`.

**Money/economy** is centralized in `PRICES` (sale value per resource type) and each `ZONES` entry's `cost`. `cap()` and `speed()` derive the player's carry capacity and move speed from which upgrades (`S.un.*`) are unlocked, rather than storing them as separate stateful fields — check these two functions before assuming a stat lives on `S.player`.

**Audio (`AU` object) is fully synthesized** via Web Audio oscillators/noise buffers — there are no audio files. `AU.init()` must run from a real user gesture (it does, from the name-entry button and canvas taps) both to unlock the `AudioContext` and, on iOS, to play a one-sample silent buffer that unlocks sound output entirely.

**Save data** is one JSON blob in `localStorage` under `myLittleFarm.v1` (name, coins, carried items, `un`/`paid` upgrade state, mute flag). `load()` also handles the `?reset` query param. Bump `SAVE_KEY` if the save shape changes incompatibly.
