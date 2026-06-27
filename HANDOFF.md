# SKITTERLINGS — Build Handoff

**What it is:** a one-button endless runner. The runner moves automatically; you **only jump** (hold = higher hop) to clear obstacles — **no duck/slide**. The run is a *journey* that cross-fades through **100 themed worlds** as you survive. Distance = score, coins drop + accrue, coins buy skin rolls and starting worlds. Headline features: a **procedural skin engine** (hundreds of deterministic critters from seeds, no art assets) and **100 hand-tuned worlds** rendered from a small library of parametric draw routines.

The game is a **single self-contained `index.html`** — vanilla HTML/CSS/JS, zero dependencies, zero build step. Runs by double-clicking (serve over http for the PWA/SW). This matches the studio stack; keep it single-file.

> Renamed from the prototype "VAULT" → **Skitterlings**. Save key is `skitterlings_save_v1` (fresh; no migration from the old key).

---

## What's built (working)

- **Core loop** — fixed virtual resolution (960×300) scaled to any screen, `requestAnimationFrame` with dt clamp, portrait + landscape, mobile-first.
- **Physics — JUMP only.** Gravity, variable-height jump (release-early cut), plus **coyote time** and **jump buffering** for forgiving input. The duck mechanic is fully removed (control, physics, hitbox, duck-under flyers all gone).
- **Spawner** — ground obstacles (1–3 clusters, varied height/width) + **low gliders** (flyers you hop, never duck-under). Gliders sit in a low band so a hop always clears them; gap scales with speed so every obstacle is clearable at max speed. Speed ramps 340→780.
- **Procedural skins** (`Skins`) — `seed → traits` via FNV-1a hash + mulberry32. Expanded trait space: 6 bodies × 8 ears × 5 tails × 6 patterns × 6 eyes × 9 accessories × 3 color schemes × rarity → thousands of distinct deterministic critters. Rarity 60/24/11/4/1. Drawn as parametric canvas shapes; run/jump poses.
- **100 worlds** (`THEMES`, section 4) — 10 biome regions of 10. Each = palette (`sky0/sky1/ground/line/ob/fly/accent`) + `parallax` + `obStyle` + `flyStyle` + `star`. A run **cross-fades to the next world every `CFG.stageLen` score** (colors lerp, parallax + stars crossfade, obstacles carry their spawn-world style). The journey runs **forward** through all 100 and clamps at the last; reaching a world **unlocks** it (persisted).
- **Renderer libraries** (section 8) — `BG` (18 parallax), `OB` (30 obstacle), `FLY` (6 flyer) maps. All dispatched through `drawParallax`/`drawObstacle` wrapped in **try/catch**, so a bad theme can never crash the run loop.
- **Named roster** (`ROSTER`, section 5) — 142 named skitterlings (`seed = name`). The Vault rolls **unowned named critters first**, then falls back to fresh numeric seeds. Collection has an **Owned / All** toggle; All shows the full roster with `???` locks.
- **Economy** — coins, gacha roll (100 coins), world purchases (= unlock + set starting world; reaching unlocks for free), equip flow, NEW BEST flag.
- **Persistence** (`Store`) — localStorage with **in-memory fallback** if storage throws (sandbox-safe). Saves coins, best, owned seeds, equipped, unlocked worlds, start world, mute, furthest world reached.
- **Audio** (`Sfx`) — tiny WebAudio SFX (jump/coin/stage/hit), mute toggle, guarded.
- **PWA** — `manifest.webmanifest` (fullscreen), `sw.js` (cache `skitterlings-v1`), `icon.svg`. SW registers only over http(s).

## File map (all gameplay in `index.html`)
Banner-commented sections in the `<script>`:
1. STORE 2. RNG/HASH 3. SKINS 4. THEMES (100 worlds) 5. ROSTER 6. SAVE/WALLET 7. AUDIO 8. RENDERERS (parallax/obstacle/flyer) 9. GAME (CFG, physics, spawner, collision, journey, render, loop) 10. UI 11. INTEGRATION (portal/economy seams — `SK_CFG`, `Wallet`, `Bridge`, `window.SKITTERLINGS`).
**All balance lives in `CFG`** (section 9). **All integration lives in section 11** — see `INTEGRATION.md`.

---

## Seams to wire (priority order — the post-build work you described)

**→ The full portal + Lucid Winds integration spec is `INTEGRATION.md`. The seam is built — it's `section 11 (INTEGRATION)` at the bottom of the `<script>`.** Summary:

1. **Currency source — DONE as a seam.** All coin changes route through one object, **`Wallet`** (`earn/spend/set/balance`) in section 11 — no scattered `save.coins +=`. Default is local. To share the **sunbeam** balance, set `SK_CFG.endpoints = { getBalance, addCoins, spendCoins }` (Cloud Function URLs) via `window.SKITTERLINGS.configure(...)`, a `configure` postMessage, or launch param — no source edits. See INTEGRATION.md §4.
2. **Sky Wolf Studio portal + Lucid Winds wiring — DONE as a seam.** `Bridge` (postMessage) + `window.SKITTERLINGS` API + launch params (`uid/embed/origin/return/world`) cover both iframe-embed and standalone-page launch. Emits `ready/run-start/game-over/coins/world-unlocked/navigate-back`; accepts `configure/pause/resume/mute/grant-coins/to-menu`. See INTEGRATION.md §3–§5.
3. **Firebase Auth + Firestore** — pass the uid as `?uid=` (namespaces the save to `skitterlings_save_v1:<uid>`). Mirror that JSON blob to Firestore for cross-device cosmetics; `Store` stays local-first, Firestore is the sync layer. Coins sync via the endpoints in seam #1.
4. **Leaderboard** — write `best` to Firestore on game over; top-N read on the title screen.
5. **Audio swap** — replace synth blips with real SFX/music. `Sfx` is the only call site; keep the method names (`jump/coin/stage/hit`).
6. **More worlds / skins** — push to `THEMES` (reference an existing `parallax`/`obStyle`/`flyStyle`, or add a renderer to the `BG`/`OB`/`FLY` map) and `ROSTER`. ~1 line per world.

## Guardrails (do not "simplify" these)

- **Keep the procedural engine.** Do NOT replace seed→traits with a hardcoded list of skin assets. "Hundreds of critters for free" depends on generation; hundreds of skins = hundreds of seeds, already true.
- **Keep determinism.** Same seed must always yield the same critter and rarity, forever. Don't add time/random into `traits()`. This is what makes skins portable across the studio and printable.
- **Keep the storage fallback.** The localStorage→memory try/catch must survive; it's what lets the file run in any sandbox without throwing.
- **Keep every obstacle clearable by a single jump.** There is no duck. Spawn heights/bands are tuned so a hop clears everything (including low gliders) at `speedMax`. If you retune `CFG`, re-verify.
- **Keep the renderer try/catch.** `drawParallax`/`drawObstacle` swallow per-draw errors so one malformed theme can't kill the loop.

## Test checklist
Manual: jump clears single + triple clusters at max speed; gliders require a hop and a hop clears them; tab-switch doesn't teleport the runner; coins increment and persist across reload; roll deducts 100, adds a named critter, never dupes; equip updates the running sprite; world buy gates on coins; reaching a world unlocks it; mute persists.
Automated: there's a headless harness pattern (stub DOM/canvas, run the extracted `<script>`) that boots the game, drives a simulated run with restart-on-death, opens every screen, and sweeps every renderer over all 100 themes + 142 roster seeds. Re-run it after engine changes.

## Deploy
Codespaces + GitHub, no build. Publish the folder to any static host.
