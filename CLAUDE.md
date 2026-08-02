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

# or serve it over HTTP (e.g. to test on a phone on the same network)
python -m http.server 8123
```

Add `?reset` to the URL to wipe the saved farm (`localStorage['myLittleFarm.v1']`) and see the first-run name prompt again.

There are no automated tests, linter, or formatter configured for this project. Verify changes by opening the file in a browser and checking the console for errors, and by exercising the relevant mechanic (harvest a plot, sell, buy an upgrade, resize the window, reload to confirm the save round-trips).

## Deployment model — read before touching sizing/meta code

This file runs as a **standalone** page: opened directly as a file, served over HTTP, or added to an iOS home screen. (It used to also support an embedded/iframe "Claude Artifact" mode; that path — the `EMBED` detection and its `resize()` branch — has been removed. Don't reintroduce iframe/host-driven sizing without restoring it deliberately.)

`resize()` sizes off `window.visualViewport` (which excludes the iOS Safari toolbar so the game doesn't get clipped under it — plain `window.innerHeight` does not), falling back to `window.innerWidth`/`innerHeight`.

Everything is sized through one `#stage` div whose pixel `width`/`height` JS sets explicitly (not CSS `100%`, which is wrong on iOS because of the toolbar). The `<canvas>` also needs an explicit CSS `width`/`height` set alongside its pixel `width`/`height` attributes, or the browser lays it out at the attribute size (`dpr`× too large) instead of the CSS size.

`resize()` is called directly on every resize/orientation/visualViewport event and additionally self-heals every 15 frames from the render loop (`frame()` near the bottom) — deliberately *not* coalesced through `requestAnimationFrame`, because a "resize already queued" flag would latch forever if that callback were ever throttled, freezing the game at the wrong size.

If you change anything under the `resize()`/`#stage` code, test at minimum: standalone in a normal browser tab, and on mobile with the URL bar hidden — the iOS-toolbar case is where regressions here tend to show up.

## Publishing

The live public deployment is **GitHub Pages**: `git push origin master` to `https://github.com/wangyx0001/my-little-farm` auto-deploys to `https://wangyx0001.github.io/my-little-farm/` within a minute or two, no extra steps. This deploys straight from what's committed — it's the source of truth for the public URL.

## Code architecture

Single `<script>` block, organized top-to-bottom as: helpers → resize → Audio → constants → state → save/load → world entities → input → HUD → update loop → render loop → boot. Section banners (`/* ---------------- Name ---------------- */`) mark these; use them to navigate rather than searching for individual functions.

**World layout is coordinate-based, not a grid.** `MAIN`, `FOREST`, `BRIDGE`, `DOCK`, `CHICKEN_PEN`, `COW_PEN` are all `{x, y, w, h}` rects in one shared `WORLD` space (1700×1000). `isWalkable(x, y)` (via `walkableRects()`) checks whether a point falls inside one of the walkable rects — `MAIN`, `FOREST`, `DOCK`, plus `BRIDGE` only once unlocked — after shrinking each by a 20px margin. Because that margin is applied per-rect, **adjacent walkable rects must physically overlap by more than the margin or a dead band forms at the seam** (this is why `BRIDGE` and `DOCK` extend a little onto the islands they connect to). Fixed-position content (`PLOT_POS`, `TREE_POS_MAIN`, `TREE_POS_FOREST`, `BUSH_POS`, `SELL_STAND`, `SELL_ZONE`, `FISH_SPOT`) are plain coordinate constants inside those rects — there's no tilemap or spatial index. Adding a new resource node means picking coordinates by eye and adding an entry to the relevant `*_POS` array.

**Upgrades are data-driven via `ZONES`.** Each entry (`{id, x, y, r, cost, label, icon, vis}`) is a circle on the ground the player stands in to gradually pay off; `vis(unlocks)` gates when it appears (e.g. `pack2`/`cowpen`/`gold` require `u.bridge`). `S.un[id]` (unlocked) and `S.paid[id]` (progress) track state; `unlock(z)` in the update loop applies the one-time side effect for that specific id (spawn a plot/chicken/cow, show a floater) via an `if (z.id === '...')` chain — there's no separate effect-registration system, so a new upgrade needs both a `ZONES` entry and a branch in `unlock()`.

**Entities are flat arrays of plain objects**, not classes: `trees`, `plots`, `bushes`, `chickens`, `cows`, `sheeps` (unlocked with the `sheep` upgrade; give `wool`, mirror the cow system exactly), plus transient ones — `drops` (items sitting on the ground), `flyers` (items animating from source to backpack or from backpack to the sell stand), `floaters` (rising text), `parts`/`coinParts` (particle effects). All are mutated in place inside `update(dt)`. Two singleton entities live outside the arrays: `cat`, a cosmetic pet that eases toward a point behind the player each frame (and warps to catch up past ~320px, e.g. across the bridge), and `fishing`, the state for the dock mini-activity (auto-casts a bobber when the player stands still on `FISH_SPOT` and drops a `fish` straight into the basket via `collectItem`).

**Day/night is purely cosmetic.** `dayT` (0..1) advances every frame over `DAY_LEN` seconds and is never saved (resets to morning on load). `nightAmount()` derives a 0→1 darkness (long day, shorter night, smooth dusk/dawn ramps) that `render()` uses to lerp the sky gradient, fade the water sparkles, draw the sun/moon/stars (`drawSky`), and lay a screen-space dark overlay punched with warm `'lighter'` lantern pools at fixed world anchors. It does **not** affect gameplay (crops/animals ignore it) — keep it that way so a young player is never blocked by nightfall.

**Rendering is immediate-mode with manual Y-sorting for depth**, redone every frame in `render()`: static ground/UI layers draw first, then movable entities (trees, bushes, animals, drops, fences, the player) are collected into one `drawables` array, sorted by their `y` anchor, and drawn back-to-front so higher-on-screen things are correctly occluded by lower ones. Truly foreground effects (flying items, particles, floating text, the joystick) draw after that sort, unaffected by depth. `draw*` functions never mutate state — they only read from the entity objects and `ctx`.

**Money/economy** is centralized in `PRICES` (sale value per resource type — `wood`/`wheat`/`berry`/`egg`/`fish`/`wool`/`milk`) and each `ZONES` entry's `cost`. `cap()` and `speed()` derive the player's carry capacity and move speed from which upgrades (`S.un.*`) are unlocked, rather than storing them as separate stateful fields — check these two functions before assuming a stat lives on `S.player`. Upgrades come in two tiers (the second — `berry3`/`boots2`/`sheep`/`pack3`/`diamond`/`fountain` — exists to keep coins meaningful late-game); total unlock cost far exceeds quick income by design, so don't "rebalance" by only tweaking prices. A separate coin sink is the **dress-up shop** (`🎨` HUD button → `#shopOverlay`): `S.cos` holds the chosen/owned farmer-dress and cat-fur palette indices (`DRESS_COLORS`/`CAT_COLORS`); `drawPlayer`/`drawCat` read `S.cos` for their colors (cat's dark/belly shades are derived from the base via `mixHex`).

**Audio (`AU` object) is fully synthesized** via Web Audio oscillators/noise buffers — there are no audio files. `AU.init()` must run from a real user gesture (it does, from the name-entry button and canvas taps) both to unlock the `AudioContext` and, on iOS, to play a one-sample silent buffer that unlocks sound output entirely.

**Save data** is one JSON blob in `localStorage` under `myLittleFarm.v1` (name, coins, carried items, `un`/`paid` upgrade state, `cos` cosmetics, mute flag). `load()` tolerates older saves missing newer keys (new `un`/`cos` fields fall back to their defaults) and handles the `?reset` query param. Bump `SAVE_KEY` if the save shape changes incompatibly.
