# Skitterlings — Integration Guide (for the Sky Wolf Studio dev team)

This is everything needed to wire **Skitterlings** into the **two destinations**:

1. **Sky Wolf Studio portal** — the hub/launcher that lists the studio's games.
2. **Lucid Winds** — the game that owns the shared **sunbeam** coin economy that Skitterlings should funnel into.

Skitterlings is a **single static page** (`index.html` + 4 support files). It ships **standalone-by-default**: with zero configuration it runs fully local (own coins, own saves, no network). You opt it into the studio by passing launch params and/or pointing it at a coin-balance endpoint. **No source edits are required** to integrate — everything is configured from outside via launch params, `postMessage`, or a tiny config object.

The integration surface lives in one place: **section 11 (`INTEGRATION`) at the bottom of `index.html`'s `<script>`**. The three objects you'll touch are `SK_CFG` (config), `Wallet` (the single economy seam), and `Bridge` (the postMessage bridge). All confirmed working by an automated harness.

---

## 0. What to deploy

Publish this folder to any static host (GitHub Pages / Cloudflare Pages / Netlify / studio CDN). No build step.

```
index.html              ← the entire game
manifest.webmanifest    ← PWA (fullscreen, installable)
sw.js                   ← offline service worker (network-first for the page)
icon.svg                ← app icon
README.md / HANDOFF.md / INTEGRATION.md   ← docs (not served to players)
```

Serve over **https** (the service worker + PWA install require it). That's the URL the portal links to / embeds — call it `GAME_URL` below.

---

## 1. Identity & saves (read this first)

- Saves are stored in `localStorage` under `skitterlings_save_v1`.
- **Pass `?uid=<playerId>`** and the save key becomes `skitterlings_save_v1:<playerId>` automatically, giving each signed-in player their **own** save on shared devices. Use your existing Firebase Auth uid.
- Saves are **per-browser** by default. For cross-device, either (a) point Skitterlings at the coin endpoints below (coins sync; cosmetics stay local), or (b) mirror the `skitterlings_save_v1:<uid>` blob to Firestore on your side (it's plain JSON). The game already keeps local-first; treat your store as the sync layer.

`uid` **must** arrive as a launch param because it namespaces the save *before* load. Everything else can be set later via `configure()` or `postMessage`.

---

## 2. Launch parameters (`GAME_URL?…`)

| Param | Example | Effect |
|---|---|---|
| `uid` | `?uid=fb_abc123` | Namespaces the save per player. Use your auth uid. |
| `embed` | `&embed=1` | Marks the session as embedded (also auto-detected inside an iframe). Enables `postMessage` emits. |
| `origin` | `&origin=https://portal.skywolf.studio` | Locks outbound/inbound `postMessage` to this host origin. **Set in production.** |
| `return` | `&return=https://portal.skywolf.studio/hub` | Where `SKITTERLINGS.goBack()` sends the player. |
| `world` | `&world=candy-frosting-dawn` | Deep-link a starting world (unlocks + selects it). IDs are in the `THEMES` array. |
| `coins` | `&coins=500` | Grants coins on load. **Testing only — do not pass in production** (client-trusted; see Security). |

Example portal launch URL:
```
https://games.skywolf.studio/skitterlings/?uid=fb_abc123&embed=1&origin=https://portal.skywolf.studio&return=https://portal.skywolf.studio/hub
```

---

## 3. Destination A — the Sky Wolf Studio portal

The portal just needs to **launch** the game and (optionally) **listen** for events. Two patterns:

### Option A1 — Embed in an iframe (recommended for an in-portal play surface)
```html
<iframe
  id="skitterlings"
  src="https://games.skywolf.studio/skitterlings/?uid=FB_UID&embed=1&origin=https://portal.skywolf.studio&return=https://portal.skywolf.studio/hub"
  allow="fullscreen; autoplay"
  style="border:0;width:100%;height:100%"></iframe>

<script>
  const FRAME = document.getElementById("skitterlings").contentWindow;
  const GAME_ORIGIN = "https://games.skywolf.studio";

  // listen to the game
  window.addEventListener("message", (e) => {
    if (e.origin !== GAME_ORIGIN) return;
    const m = e.data;
    if (!m || m.source !== "skitterlings") return;
    switch (m.type) {
      case "ready":         /* game booted */ break;
      case "run-start":     /* m.data.world */ break;
      case "game-over":     /* {score,best,coins,world} → update portal profile, leaderboard */ break;
      case "coins":         /* {balance,delta,reason} → mirror sunbeam balance in portal UI */ break;
      case "world-unlocked":/* {id,count} */ break;
      case "navigate-back": /* player tapped Back → route in your SPA */ break;
    }
  });

  // command the game
  const send = (type, data) => FRAME.postMessage({ target: "skitterlings", type, data }, GAME_ORIGIN);
  // send("pause"); send("resume"); send("mute", {muted:true});
  // send("grant-coins", {amount: 250});   // e.g. a daily reward from the portal
  // send("configure", {endpoints: {...}}); // push economy endpoints post-load
</script>
```

### Option A2 — Launch as its own page/tab (simplest)
Add a tile/card in the portal that links to `GAME_URL?uid=…&return=…`. No iframe, no message bus. The game reads `uid`/`return`/`world`; the player taps the in-game menu and (if you wire it) `SKITTERLINGS.goBack()` returns them via `return`. Cross-game economy still works through the endpoints in §4.

**Portal tile copy:** title "Skitterlings", tagline "run · hop · collect", icon `icon.svg`. The game exposes `best` and worlds-unlocked through `game-over` events if you want live stats on the tile.

---

## 4. Destination B — Lucid Winds / the shared sunbeam economy

Skitterlings routes **every** coin change through one object, `Wallet` (`earn/spend/set/balance`). To make Skitterlings coins **the same currency** as Lucid Winds, give it three endpoints (your existing sunbeam Cloud Functions). Set `endpoints` and Skitterlings will:

- **on load** → `getBalance` and adopt the shared balance,
- **on earn** (banking a run's coins) → `POST addCoins`,
- **on spend** (a 100-coin roll or a world purchase) → `POST spendCoins`,

while keeping the local balance as the optimistic source of truth (writes are fire-and-forget; reads reconcile on load).

### Wire it (pick one)
**Via launch + configure (cleanest):** the portal/host calls, after the iframe loads or as a `configure` message:
```js
send("configure", {
  endpoints: {
    getBalance:  "https://us-central1-skywolf.cloudfunctions.net/sunbeamBalance",
    addCoins:    "https://us-central1-skywolf.cloudfunctions.net/sunbeamAdd",
    spendCoins:  "https://us-central1-skywolf.cloudfunctions.net/sunbeamSpend"
  }
});
```
**Or same-origin direct** (if the game is on your origin): `iframe.contentWindow.SKITTERLINGS.configure({ endpoints: {…} })`.

### Endpoint contract (what the game sends / expects)
- `GET  getBalance?uid=<uid>` → **`200 { "balance": <number> }`**
- `POST addCoins`  body `{ "uid", "game":"skitterlings", "amount":<+int>, "reason":<string> }` → 200 (body ignored)
- `POST spendCoins` body `{ "uid", "game":"skitterlings", "amount":<+int>, "reason":<string> }` → 200 (body ignored)

`reason` is one of `"run"` (banked a run), `"roll"` (skin roll), `"world:<id>"` (world purchase), `"host"`/`"launch"`/`"dev"`. Use it for analytics/anti-abuse. These are the same shapes Lucid Winds should accept so the two games share one balance.

> **Authoritative balance:** the game is optimistic locally for snappy UX. Your Cloud Function is the source of truth — have it validate and, if it rejects/clamps, the next `getBalance` (on reload) reconciles. If you want hard server-authority per transaction, return the new balance from `addCoins/spendCoins` and we'll add a one-line apply (left out by default to avoid a round-trip per coin).

---

## 5. The full API surface (reference)

**`window.SKITTERLINGS`** (call from a same-origin host or via the returned window):
- `.configure({ uid?, parentOrigin?, endpoints?, returnUrl?, embedded? })` → updates config; setting `endpoints` triggers a balance pull.
- `.wallet` → `{ balance(), earn(n,reason), spend(n,reason)→bool, set(n) }` — drive the economy directly.
- `.emit(type, data)` → emit a custom message to the host.
- `.goBack()` → navigate to `returnUrl` and emit `navigate-back`.
- `.version` → `"1.0.0"`.

**Events emitted game → host** (`{ source:"skitterlings", type, data }`):
`ready` · `run-start {world}` · `game-over {score,best,coins,world}` · `coins {balance,delta,reason}` · `world-unlocked {id,count}` · `navigate-back`.

**Commands accepted host → game** (`{ target:"skitterlings", type, data }`):
`configure {…}` · `pause` · `resume` · `mute {muted}` · `grant-coins {amount}` · `to-menu`.

---

## 6. Security checklist

- **Set `origin`** (launch param) or `parentOrigin` (configure) to the real host origin in production. The bridge ignores inbound messages from any other origin and targets outbound messages to it. (Verified.)
- **Server-authoritative economy.** Treat the client as untrusted: validate `addCoins/spendCoins` server-side (rate-limit, cap, auth the `uid` against the session). The local balance is for UX only.
- **Strip `?coins=` in production.** It's a client-trusted grant for QA. Don't expose it in real launch URLs (and you can have the Cloud Function ignore `reason:"launch"`/`"dev"` credits).
- The `window.SK` console helpers (`addCoins`, `unlockAllWorlds`, `wipe`) are dev conveniences. They only affect the **local** save; with server-authoritative endpoints they can't grant real shared currency. Leave or strip — your call.

---

## 7. Decisions for the boss

1. **Embed (iframe in portal) or standalone page/tab?** Determines A1 vs A2.
2. **Shared sunbeam economy, or keep Skitterlings coins standalone for v1?** If shared, provide the 3 endpoints (§4).
3. **Cross-device saves for cosmetics?** Coins sync via endpoints; if you also want the critter collection to follow players across devices, mirror the `skitterlings_save_v1:<uid>` JSON to Firestore (your existing pattern). v1 can ship local-only.
4. **Leaderboard?** `game-over` emits `{score,best}`; point it at your existing board if wanted.
5. **Production host origin** for the `origin`/`parentOrigin` lock.

---

## 8. QA checklist

- [ ] Launch `GAME_URL?uid=test1` → play a run → reload → coins/best/collection persisted under `test1`.
- [ ] Launch with a different `uid` → independent save.
- [ ] Embedded: portal receives `ready`, `run-start`, `game-over`, `coins` messages; all carry the host origin.
- [ ] `send("grant-coins",{amount})` increments the in-game wallet; `pause`/`resume`/`mute` respond.
- [ ] With endpoints set: load pulls the shared balance; a roll/world-buy POSTs `spendCoins`; banking a run POSTs `addCoins`; wrong-origin messages are ignored.
- [ ] `?coins=` removed from production URLs.
- [ ] `goBack()` / `navigate-back` returns the player to the portal.

---

*Implementation note for whoever opens the code:* the seam is **section 11** of `index.html`. `Wallet` is the only place coins change; `Bridge` is the only place messages cross the frame; `SK_CFG` holds config. You shouldn't need to touch sections 1–10 (engine/UI) to integrate.
