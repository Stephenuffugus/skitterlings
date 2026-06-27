# Skitterlings — Competitive Research & Stress Test

A research + hardening pass: what comparable games do that we could adopt, and how the build holds up under abuse. Research was done by parallel web-research agents (sources cited inline); the stress test by a headless harness that boots the real game and hammers it.

---

## Stress test — results (all passed)

Headless harness boots `index.html` and drives it hard. **No failures.**

| Test | Result |
|---|---|
| 100-world endurance (60k frames, no-death) | Reaches **world 100/100**; on-screen arrays stay tiny (max **3** obstacles, **5** coins, **35** particles) — **no leaks**; heap grew **4.6MB** total; **no NaN/Infinity** in physics |
| Input spam (8× jump+release+key+pointer per frame ×4000) | Survives; physics stays finite |
| Gacha abuse (700 rolls) | **0 duplicates** (701 owned); favourites correctly capped at 3; collection opens without error |
| Save robustness | Corrupt JSON / empty / garbage-typed fields / **localStorage quota-exceeded** all handled → defaults or memory fallback, **no crash** |
| Numeric extremes (score 79,589) | Speed stays **capped at 800**; world **clamped at 100**; all values finite; Glimmer chase stays finite |
| Determinism | `nameFor(seed)` stable across calls |
| Performance | **0.26 ms/frame** of JS update+render (16.7ms/60fps budget) — huge headroom |

**Weakness found & fixed:** the collection rendered *every* owned critter at once, and `owned` grows unbounded with rolls — fine at 700, but thousands would hitch on open. **Fixed:** the grid now caps at the **300** best/newest (already sorted favourites → rarity → newest) with a "+N more" note. Robustness is otherwise excellent.

**Remaining watch-items (not bugs):** `save.owned` + localStorage grow with every roll (≈60KB JSON at ~5k rolls — within quota but worth a future prune/dedupe); the per-obstacle 9-pass outline is the heaviest draw but bounded (few obstacles on screen).

---

## What we could improve — by category

### A. Endless-runner depth (one-button)
Our *feel* primitives are already right — coyote time (0.09s) + jump buffering (0.13s) + variable-height hold-jump match Canabalt's "systematic forgiveness." What's missing is variety, goals, and skill expression. Patterns the best runners use that we lack:
- **Per-run missions/goals** (Jetpack Joyride: 3/run; Alto: 3 goals/level) → 3 goals at run start paying coins/a roll. *The single biggest "one-more-run" driver.* `[Quick win]`
- **Combo/score multiplier + near-miss bonus** (Robot Unicorn Attack, Temple Run, Burnout) → consecutive coin pickups / tight obstacle clears build x2→x8; a miss resets. Pure depth, no new input/art. `[Quick win]`
- **Reactive score feedback** → scale the HUD score with combo, fire milestone banners (`pushBanner` exists), ramp particles/shake with the multiplier. `[Quick win]`
- **Power-ups** (Jetpack Joyride magnet/shield/2x) → a 4th spawn type: Glimmer Dust (slow-mo), magnet, bubble shield. `[Bigger bet]`
- **Transform/"vehicle" critters** (Jetpack Joyride vehicles redefine the one input) → ride a *collected* critter ~10s that re-skins the button (glider = hold-to-float, bouncer, gravity-flip). **This finally connects the 415-critter collection to gameplay.** `[Bigger bet, highest ceiling]`
- **Speed-proof authored chunk spawner** (Canabalt caps gap at ⅔ speed; "Sure Footing") → replace loose random gaps with a bag of ~15 difficulty-tagged, guaranteed-clearable obstacle chunks → fair at 800px/s *and* learnable patterns. `[Bigger bet]`
- **Make the Glimmer pay off** → it's currently visual-only; have it drop bonus coins / a guaranteed-rare token when you keep a combo alive beneath it. Turns our best story beat into a mechanic. `[Quick win]`

*Sources: [Tuning Canabalt](https://www.gamedeveloper.com/design/tuning-canabalt), [Jetpack Joyride power-ups/vehicles](https://jetpackjoyride.fandom.com/wiki/Power-Ups), [Alto's goals](https://altosadventure.fandom.com/wiki/Goals), [Robot Unicorn Attack](https://gamefaqs.gamespot.com/flash/996571-robot-unicorn-attack/faqs/59954), [Sure Footing PCG](https://www.gamedeveloper.com/design/keep-running-procedural-level-generation-in-sure-footing), [Crossy Road monetization](https://www.pocketgamer.biz/what-can-you-learn-from-crossy-road/).*

### B. Retention, progression & (kid-safe) live-ops
Our loop is a solid collection meta but has **no reason-to-return-today** and **no completion goals**. Industry D1 median ~22% / D7 ~8%; the gap is almost always onboarding + a daily-habit + goal layer. For a *kids* game, add these **without** FOMO/loss-pressure/paid randomness (Toca Boca / Sago Mini standard).
- **FTUE: first free roll in <60s**, taught with tooltips, un-failable. *Highest-leverage D1 lever (up to +50%).* `[Quick win]`
- **Daily login gift + forgiving streak** (auto streak-freeze; never a "you lost your streak" punishment) — biggest single missing piece for D7. `[Quick win]`
- **Daily run-missions** (2–3, all completable from normal play). `[Quick win]`
- **Achievement/sticker badges** tied to the story & collection ("Met the Glimmer," "World 25," "10 critters"). Cosmetic recognition — ideal for kids. `[Quick win]`
- **Collection-set / "critter-dex" goals**: group the named ones into sets/biomes with completion bonuses → turns infinite collecting into completable goals. `[Bigger bet]`
- **Weekly featured world + light earn-by-playing seasonal events** (a "Glimmer Festival"). `[Bigger bet]`
- **Sunbeam shared currency + portal cross-promo** with Lucid Winds — earning in one game gives a reason to open the other (portfolio cross-promo can lift retention ~5% → revenue +30–100% with near-zero UA cost). `[Bigger bet]`
- **Don't add an energy/lives system** — it punishes your most-engaged players and adds nothing to a skill runner.

**Economy/gacha advice:** add a **pity guarantee** (soft pity: odds rise with rolls; hard pity: guaranteed-new after N) and make **every roll move you forward** (dupes → a little progress currency). **Disclose odds in plain, kid-readable language.** Keep rolls **100% coin-earned, never real-money** — that's our biggest ethical/regulatory advantage (FTC's $20M Genshin settlement bans selling loot boxes to under-16s; PEGI defaults loot-box games to 16; coin-only sidesteps all of it). Tune sources/sinks so a typical session is a *meaningful fraction* of a roll (frequent small wins), with new worlds as the long-term coin sink.

*Sources: [retention benchmarks](https://maf.ad/en/blog/mobile-game-retention-benchmarks/), [Supersonic D1–D7](https://supersonic.com/learn/blog/6-ways-to-boost-your-games-retention-from-d1-d7/), [Duolingo forgiving streaks](https://blog.duolingo.com/how-duolingo-streak-builds-habit/), [pity systems](https://mwm.ai/glossary/pity-system), [FTC/Genshin](https://www.bleepingcomputer.com/news/gaming/ftc-cracks-down-on-genshin-impact-gacha-loot-box-practices/), [AADC Children's Code](https://www.makeuseof.com/what-is-age-appropriate-design-code/), [Toca Boca](https://grokipedia.com/page/Toca_Boca).*

### C. Collection-loop depth (collectible / gacha critters)
We already built a **goldmine we aren't spending**: a finite roster of **415 named critters, each tagged with a `biome`** — but it's only used to *name* matching seeds; players never see it as a list, and **equipping is purely cosmetic** (rarity/species give zero gameplay effect). Patterns we're missing:
- **A finite, visible "Skitterdex" with X/415 (n%) and locked silhouette slots.** Today the grid shows only owned, so there's no "collect them all" target. Iterate `ROSTER` (not `save.owned`), show greyed `?????` slots + a progress bar. Empty slots create the Zeigarnik tension that drives completion (the Pokédex / Critterpedia pattern). **Keystone fix.** `[Quick win]`
- **Biome set-completion bonuses** (we already tag every named critter's biome) → completing a biome grants coins + a guaranteed roll + an **exclusive "biome-guardian" skitterling**. `[Bigger bet]`
- **Every-10 milestone rewards + a full-dex grand prize** (Pokémon S/V gives a reward per 10 entries + Shiny Charm at 100%). `[Quick win]`
- **Equip-bonuses (rarity/biome → small in-run perks)** — our biggest untapped lever. Right now critters are pretty but meaningless (Spore's "infinite-but-samey" trap). Key a kid-friendly perk off `traits.rarIdx` (Rare = small coin magnet, Legendary = +10% coins, biome-complete = a themed perk). Makes "which do I equip?" a real choice. `[Bigger bet, highest long-term impact]`
- **Soft "luck meter" pity** — track rolls-since-rare; ramp Epic+ odds with a visible "✨ guaranteed shiny soon" bar. Feel-good only (no money involved). `[Quick win]`
- **One forgiving free daily roll** (Neko Atsume drip-feed) — cumulative days, never punishing reset. `[Quick win]`
- **Nickname favourites** + a tiny bond meter (Tamagotchi effect makes an infinite pool feel personal — No Man's Sky lets you rename creatures). `[Quick win]`
- **Showcase / share a collection card PNG** (render via existing `Skins.draw` → `canvas.toBlob()` → Web Share API). Showcase, **not** stranger-trading (kid safety). `[Quick win]`

*Sources: [Zeigarnik/quest-logs](https://www.psychologyofgames.com/2013/03/the-zeigarnik-effect-and-quest-logs/), [Pokédex rewards](https://www.thegamer.com/pokemon-scarlet-violet-all-pokedex-completion-rewards/), [layered rewards/set bonuses](https://www.gamedeveloper.com/design/game-design-theory-applied-a-layered-rewards-system), [Spore postmortem](https://www.gamedeveloper.com/design/spore-my-view-of-the-elephant), [pity systems](https://dotgg.gg/how-gacha-pity-systems-actually-work-soft-pity-hard-pity-and-the-50-50/), [Neko Atsume](https://alexiamandeville.medium.com/game-design-breakdown-the-simplicity-of-neko-atsume-a8616a937a47), [Tamagotchi effect](https://en.wikipedia.org/wiki/Tamagotchi_effect).*

### D. Game feel, audio, accessibility, web performance

**Audio (our biggest single gap — there's no music and a known iOS pitfall):**
- **MUST-FIX — iOS/mobile AudioContext unlock.** The context is created `suspended` under autoplay policy; nothing plays until `resume()` runs inside a real gesture. Resume on the tap-to-start **and** on `visibilitychange` (mobile suspends on lock/app-switch, killing audio), plus prime iOS with a 1-sample silent buffer. *(We partially do this — harden it.)* `[Quick win]`
- ±20% per-trigger **pitch variation** on coin/jump SFX (kills repetition fatigue); **rising pentatonic** pitch as a coin/combo streak builds (always consonant). `[Quick win]`
- Proper scheduled **ADSR** ramps; route **master Gain → DynamicsCompressor → destination** with separate music/SFX buses; a persisted **music on/off + volume** toggle. `[Quick win]`
- **Procedural generative music with zero assets** — a lookahead oscillator scheduler over a pentatonic scale (chiptune square/triangle), plus a two-layer base+blend **equal-power crossfade** that intensifies with game speed, and **duck** music under hit/stage SFX. `[Bigger bet]`
- *Caveat:* iOS routes WebAudio through the ringer channel, so a muted hardware switch silences it even after a correct unlock — so **audio must stay non-essential** to gameplay (it is).

**Feel/juice:** we already have particles, shake, squash/stretch, banners. Add **hit-stop** (freeze 60–100ms on crash) for impact, and tie juice intensity to the combo multiplier (see §A). `[Quick win]`

**Accessibility (kids' game — some are compliance MUST-DOs):**
- **`prefers-reduced-motion`** — respect it: cut/reduce screen-shake, parallax scroll, and particle density (motion-sensitivity & vestibular safety). **Must-do.** `[Quick win]`
- **Photosensitivity/flashing safety** — keep flashes mild, no full-screen rapid flashing (WCAG ≤3/s). Our shake/auras are gentle; verify nothing strobes. **Must-do.** `[Quick win]`
- **Don't rely on colour alone** — obstacles already use shape + outline + shadow (good); keep it. Add a high-contrast/colourblind-friendly check on the 100 palettes. `[Quick win]`
- **Input forgiveness** — already strong (coyote 0.09s + buffer 0.13s + tap-anywhere). ✓
- Larger text option / readable menus; we're mostly fine (whole-screen tap target is ideal for kids). 

**Web/PWA performance & distribution:**
- Already strong: single-file (fast load), DPR capped at 2.5, bounded per-frame draws (stress: 0.26ms JS/frame), offline PWA, A2HS manifest.
- Consider: a `prefers-reduced-motion`/battery-aware low-power mode; and packaging for web-game portals (Poki/CrazyGames) which expect a simple embed + their SDK — easy given the single-file build.

*Sources: [MDN WebAudio best practices](https://developer.mozilla.org/en-US/docs/Web/API/Web_Audio_API/Best_practices), [web.dev game audio](https://web.dev/articles/webaudio-games), [iOS unlock gist](https://gist.github.com/TimvanScherpenzeel/c870b35358fb96fa643d9ed1ea606efd), [WebKit ringer-channel bug](https://bugs.webkit.org/show_bug.cgi?id=237322), [dynamic WebAudio music](https://cschnack.de/blog/2020/webaudio/).*

---

## Consolidated priority backlog (all four streams)

Three independent research streams converged on the same gaps: **no reason to return today**, **no completion goal**, and **collected critters don't matter in-game.** Everything below is kid-safe (no IAP, no FOMO, no punishing streaks, coin-only gacha).

### Do-first (cheap, high-impact, low-risk)
1. **The Skitterdex** — turn the existing 415 named critters into a visible "X / 415" album with locked silhouette slots + every-10 milestone rewards. *The keystone — gives the whole collect loop a goal. Pure UI over data we already have.*
2. **Combo multiplier + near-miss bonus + reactive score feedback** — real skill depth from the single button (reuses `pushBanner`/`fx`/`shake`); no new art/input.
3. **Per-run missions** (3 goals → coins/roll) + the **Glimmer pays off** (drops a bonus when you chase it well) — the "one-more-run" driver, and it makes our best story beat a mechanic.
4. **Soft pity luck meter + plain-language odds + dupes-give-progress** — keeps the coin-only gacha feeling fair (and clearly clear of loot-box regulation).
5. **One forgiving free daily roll / "Today's Critter"** (date-seeded, deterministic) + cumulative (non-punishing) streak — the missing daily-habit hook.
6. **Accessibility/safety must-dos**: respect `prefers-reduced-motion`, verify no harmful flashing, harden the iOS audio unlock. Compliance-relevant for a kids' game.
7. **Achievement/sticker badges** tied to the story & collection; **nickname favourites**; **share a collection-card PNG**.

### Bigger bets (highest ceiling, need balance/content)
1. **Equip-bonuses + transform/"vehicle" critters** — make rarity/biome give small in-run perks, and let you *ride* a collected critter that re-skins the one button. **This is the move that connects the 415-critter collection to gameplay** and keeps it fresh for hours (flagged by both the runner and collectible research).
2. **Biome set-completion bonuses + exclusive guardian critters** (we already tag biomes).
3. **Speed-proof authored chunk spawner** — replace loose random gaps with difficulty-tagged, guaranteed-clearable patterns (fairness at top speed + learnable mastery).
4. **Procedural generative music** (zero-asset WebAudio) + adaptive intensity.
5. **Weekly featured world + generous seasonal events**, and the **sunbeam cross-currency portal** loop with Lucid Winds (portfolio cross-promo).

### Suggested first sprint
Skitterdex (1) + combo/near-miss/score-juice (2) compound immediately and are all low-risk; then missions + Glimmer-reward (3) and pity + daily (4,5). Equip-bonuses/transform critters (bigger bet #1) is the highest-ceiling follow-up once the meta goals exist.
