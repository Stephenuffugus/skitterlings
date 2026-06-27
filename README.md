# Skitterlings

> **Integrating this into the studio? Read [`START-HERE.md`](START-HERE.md).** Live build (once Pages is on): **https://stephenuffugus.github.io/skitterlings/**

A one-button endless runner. You **run automatically and tap JUMP** to hop obstacles — no ducking, just timing. Your run is a *journey* that cross-fades through **100 themed worlds** as you survive, dropping coins you spend rolling **hundreds of procedurally-drawn skitterlings** to collect and equip. Single-file vanilla PWA, zero build step, zero art assets.

## Files
- `index.html` — the whole game (engine, procedural skins, 100 worlds, renderers, shop, UI).
- `manifest.webmanifest` — installable PWA, **fullscreen** display, free orientation.
- `sw.js` — offline cache / service worker.
- `icon.svg` — app icon.
- `STORY.md` — the lore: skitterlings are sparks of **the Glimmer**, the light they chase but never catch.
- `HANDOFF.md` — architecture, guardrails, section map. **Read this first.**
- `INTEGRATION.md` — how to wire it into the Sky Wolf portal + the Lucid Winds / sunbeam economy (launch params, postMessage API, `Wallet` endpoints). **Hand this to the studio dev.**

## Run it (Codespace / browser — this is where to test controls)
The inline chat preview clips the bottom button. Test it for real:

1. Serve the folder over http (the service worker + install need a server, not `file://`):
   ```
   python3 -m http.server 8080
   ```
2. Open the forwarded port. On a phone, tap **⛶** (top-right) for true full screen, or use the browser's **Install app** — installed, it launches chromeless with the JUMP button clear of browser UI.

## Controls
- **JUMP** — the big button (hold = higher hop). Tapping the play area also jumps.
- Desktop: **Space / ↑ / W** to jump (hold for a higher hop). That's the whole game.

## What's in it
- **100 worlds** across 10 biome regions (Verdant Meadows → Celestial Spire), each a hand-tuned palette + parallax + obstacle/flyer style. Your run journeys forward through them; each world you reach unlocks. Buy ahead from the **Worlds** screen.
- **Hundreds of skitterlings** — a deterministic seed→creature engine (body × ears × tail × pattern × eyes × accessory × palette × rarity). 142 are named collectibles; the Vault rolls them by name, then keeps generating fresh ones.
- **18 parallax styles · 30 obstacle styles · 6 flyer styles**, all themed per world. Every obstacle is clearable with a single jump (no duck needed) — flyers are low gliders you hop.

## Quick tuning
All balance is in the `CFG` object in `index.html`:
- `stageLen` — score per world before the next cross-fade (240).
- `speed0` / `speedMax` / `speedRamp` — difficulty curve.
- `gravity` / `jumpV` / `jumpCut` — jump feel. `coyote` / `buffer` — input forgiveness.

Worlds live in the `THEMES` array, named characters in `ROSTER`. Adding a world = push one object referencing an existing `parallax`/`obStyle`/`flyStyle`. See `HANDOFF.md` for the economy/portal seam and the Lucid Winds / Sky Wolf Studio wiring.

## Deploy
Static host (GitHub Pages, Cloudflare Pages, Netlify) — publish the folder. It sits alongside the other studio games as its own page.
