# START HERE — Skitterlings

If you're from **Sky Wolf Studio** / **Lucid Winds** and you've come to take this and wire it into the website, this is your front door. Everything is in this repo — clone it, read the doc for your job, done.

```
git clone https://github.com/Stephenuffugus/skitterlings
cd skitterlings        # main branch = latest
```

## What it is
A **one-button endless runner**. Run automatically, tap **JUMP** to clear obstacles, journey through **100 themed worlds**, and collect from a deep **critter gacha** (~9.3M variants, 6 rarities incl. Mythic, no-dupe rolls, favourites). It's a **single static page** — `index.html` is the whole game. No build step.

## Live build
Once GitHub Pages is on for this repo, it's live at:
**https://stephenuffugus.github.io/skitterlings/**
(A Pages deploy workflow is included — see `.github/workflows/pages.yml`. If Pages isn't on yet: repo **Settings → Pages → Build and deployment → GitHub Actions**, then re-run the workflow, or just **Deploy from a branch → main → /root**.)

To run locally instead: `python3 -m http.server 8080` in the repo, open the port.

## Deploy the website
Publish the folder to any static host (GitHub Pages / Cloudflare Pages / Netlify / your CDN), served over **https**. Ship these:
`index.html` · `manifest.webmanifest` · `sw.js` · `icon.svg`.
It runs **fully standalone by default** (local coins + saves, no network). You opt it into the studio with launch params / one `configure()` call — **no source edits required**.

## Pick your job

| Your job | Read this | TL;DR |
|---|---|---|
| **Sunbeam economy** (Lucid Winds) | **`SUNBEAM-HANDOFF.md`** | Build 3 endpoints (`getBalance`/`addCoins`/`spendCoins`). Bearer auth, idempotency `txid`, return `{balance}`. Includes a **drop-in Firebase Functions impl** + checklist. |
| **Portal embed/launch** | **`INTEGRATION.md`** | iframe or page launch; `?uid=` identity; `postMessage` events + commands; `window.SKITTERLINGS` API. |
| **Engine / tuning** | **`HANDOFF.md`** | Architecture, section map, guardrails. Balance is in `CFG`; prices in `priceOf()`/`ROLL_COST`; odds in `RARITIES`. |

## Connecting the economy (one call, once your endpoints are live)
```js
gameWindow.SKITTERLINGS.configure({
  token: firebaseIdToken,
  endpoints: { getBalance, addCoins, spendCoins }
});
```
Identity arrives as `?uid=<firebaseUid>` (namespaces the save). The game emits `coins` / `game-over` / `run-start` / `world-unlocked` / `ready`, and accepts `configure` / `pause` / `resume` / `mute` / `grant-coins` / `to-menu`.

## Division of labor
- **Skitterlings (in-game economy):** what coins cost, what they buy, when they're earned. All coin changes route through one `Wallet` seam (`index.html` section 11).
- **Sunbeam (you):** authoritative shared balance, validation, idempotency, anti-cheat. Skitterlings is optimistic locally and reconciles to whatever your endpoints return.

Questions / tweaks (prices, jump feel, odds, contract changes) are quick edits — send them back and they'll get turned around fast.
