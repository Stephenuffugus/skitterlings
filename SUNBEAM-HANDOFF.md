# Sunbeam ⇄ Skitterlings — Economy Handoff (for the Lucid Winds dev/agent)

You own **sunbeam** (the shared cross-game currency + its Cloud Functions / Firestore). I own **Skitterlings' in-game economy** (what coins cost, what they buy, when they're earned). This doc is the **contract between them** so you can stand up the endpoints and get it live for testing now.

Skitterlings is already built to call your endpoints — **no changes needed on my side**. You build 3 HTTP endpoints to the spec below, then point Skitterlings at them by passing it 3 URLs + an auth token (one `configure` call, shown in §6). Everything here is verified against the actual game build by an automated harness.

---

## 1. Division of responsibility

| | **Skitterlings (me)** | **Sunbeam (you)** |
|---|---|---|
| Coin **sources** | Distance + pickups per run; banks on game-over (`reason:"run"`) | — |
| Coin **sinks** | Skin rolls (100, `reason:"roll"`), world unlocks (`reason:"world:<id>"`) | — |
| Wallet display | Shows balance in-game, optimistic | — |
| **Authoritative balance** | Optimistic local cache only | **Source of truth** (Firestore `wallets/{uid}.balance`) |
| Identity | Forwards `uid` + auth token | Verifies token → uid; authorizes |
| Anti-cheat / validation | — | **Validate amount, rate-limit, idempotency, clamp** |
| Reconciliation | Pulls your balance on load; adopts `{balance}` you return on writes | Returns authoritative balance |

Net: **I tell you "player X earned/spent N coins for reason R (txid T)." You decide what actually happens to the shared balance and tell me the new total.**

---

## 2. The 3 endpoints (what Skitterlings calls)

Base behavior: local balance is optimistic; my writes are **fire-and-forget** but I **adopt any `{balance}` you return** (so you can be authoritative per-transaction). On load I call `getBalance` and adopt it.

### `GET {getBalance}?uid=<uid>`
Headers: `Authorization: Bearer <token>` (when a token is configured).
**Response `200`:** `{ "balance": <number> }` → I set the wallet to this.

### `POST {addCoins}` — player earned coins
Headers: `Authorization: Bearer <token>`, `Content-Type: application/json`
**Body:**
```json
{ "uid": "player_42", "game": "skitterlings", "op": "add",
  "amount": 100, "reason": "run", "txid": "sk_lq3k9_a1b2c3", "ts": 1719500000000 }
```
**Response `200`:** `{ "balance": <newTotal> }` (optional but recommended — I'll adopt it).

### `POST {spendCoins}` — player spent coins
Same headers + body shape, with `"op": "spend"`, `"reason": "roll"` or `"world:<id>"`.
**Response `200`:** `{ "balance": <newTotal> }` (optional; I'll adopt it).

> If you **reject** a spend (insufficient shared funds, etc.), return the unchanged authoritative balance — my next `getBalance` (and any returned `{balance}`) reconciles the player back. I stay optimistic locally in between, so design for that (see §4).

### Field reference
| field | type | meaning |
|---|---|---|
| `uid` | string | player id (your Firebase uid, passed to me via `?uid=`) |
| `game` | string | always `"skitterlings"` (namespacing for your analytics/limits) |
| `op` | string | `"add"` or `"spend"` |
| `amount` | int > 0 | coins to add/spend |
| `reason` | string | `"run"` · `"roll"` · `"world:<id>"` · `"host"` · `"launch"`/`"dev"` (test grants — **you should ignore/curve these in prod**) |
| `txid` | string | **idempotency key**, unique per transaction (`sk_…`) |
| `ts` | int | client epoch ms (advisory only — don't trust for ordering) |

---

## 3. Auth, CORS, idempotency (the must-dos for go-live)

- **Auth:** when a `token` is configured, I send `Authorization: Bearer <token>`. Use your Firebase Auth ID token. Verify it server-side (`admin.auth().verifyIdToken`) and confirm it matches `uid` in the body/query. **Don't trust `uid` alone.**
- **CORS:** the endpoints are called from the game's browser origin (e.g. `https://games.skywolf.studio`). Send `Access-Control-Allow-Origin: <game origin>`, allow `Authorization, Content-Type`, and handle `OPTIONS` preflight.
- **Idempotency:** I may retry-ish (reloads, flaky networks) — **dedupe by `txid`**. Apply each `txid` at most once; on a repeat, return the current balance without re-applying.
- **Validation:** clamp/reject absurd `amount`s, rate-limit per uid, and **ignore credits with `reason` in `{launch, dev, host}`** in production (those are my QA/host grants, not earned currency).

---

## 4. What Skitterlings guarantees (so you can design for it)

- **Optimistic local:** I update the on-screen balance immediately, then call you. If you're down, gameplay continues on the local number and reconciles on next load. → Your balance is truth; transient local drift is expected and self-heals.
- **No client-side enforcement of the shared cap:** a spend always succeeds locally if the *local* number allows it. If the *shared* balance is actually lower (multi-device), your `spendCoins` should reject and return the real balance; I adopt it. So **enforce affordability server-side**.
- **Per-transaction txid**, fire-and-forget POST, GET on load. That's the whole surface.

---

## 5. Reference implementation (Firebase Functions v2 + Firestore)

Drop-in skeleton — adjust to your project. Implements auth, CORS, idempotency, atomic balance, and returns `{balance}`.

```js
// functions/sunbeam.js
const { onRequest } = require("firebase-functions/v2/https");
const admin = require("firebase-admin");
admin.initializeApp();
const db = admin.firestore();

const ALLOW_ORIGIN = "https://games.skywolf.studio";   // the game origin
function cors(res) {
  res.set("Access-Control-Allow-Origin", ALLOW_ORIGIN);
  res.set("Access-Control-Allow-Headers", "Authorization, Content-Type");
  res.set("Access-Control-Allow-Methods", "GET, POST, OPTIONS");
}
async function authUid(req) {
  const h = req.get("Authorization") || "";
  const m = h.match(/^Bearer (.+)$/);
  if (!m) throw new Error("no token");
  const decoded = await admin.auth().verifyIdToken(m[1]);
  return decoded.uid;
}

exports.sunbeamBalance = onRequest(async (req, res) => {
  cors(res); if (req.method === "OPTIONS") return res.status(204).end();
  try {
    const uid = await authUid(req);
    const doc = await db.doc(`wallets/${uid}`).get();
    res.json({ balance: doc.exists ? (doc.data().balance || 0) : 0 });
  } catch (e) { res.status(401).json({ error: "unauthorized" }); }
});

async function applyDelta(req, res, sign) {
  cors(res); if (req.method === "OPTIONS") return res.status(204).end();
  try {
    const uid = await authUid(req);
    const { amount, reason, txid, game } = req.body || {};
    if (game !== "skitterlings" || !txid || !(amount > 0))
      return res.status(400).json({ error: "bad request" });
    if (["launch", "dev", "host"].includes(reason))   // ignore QA/host grants in prod
      { const d = await db.doc(`wallets/${uid}`).get(); return res.json({ balance: d.exists ? d.data().balance||0 : 0 }); }

    const balance = await db.runTransaction(async (tx) => {
      const txnRef = db.doc(`sunbeamTxns/${txid}`);
      if ((await tx.get(txnRef)).exists) {                       // idempotent: already applied
        const w = await tx.get(db.doc(`wallets/${uid}`));
        return w.exists ? (w.data().balance || 0) : 0;
      }
      const wref = db.doc(`wallets/${uid}`);
      const cur = (await tx.get(wref)).data()?.balance || 0;
      const next = cur + sign * amount;
      if (next < 0) throw new Error("insufficient");            // server-authoritative affordability
      tx.set(wref, { balance: next }, { merge: true });
      tx.set(txnRef, { uid, game, op: sign > 0 ? "add" : "spend", amount, reason, ts: Date.now() });
      return next;
    });
    res.json({ balance });
  } catch (e) {
    if (e.message === "insufficient") {                          // return real balance so the game reconciles
      try { const uid = await authUid(req); const d = await db.doc(`wallets/${uid}`).get();
        return res.status(200).json({ balance: d.exists ? d.data().balance||0 : 0 }); } catch (_) {}
    }
    res.status(e.message === "no token" ? 401 : 500).json({ error: e.message });
  }
}
exports.sunbeamAdd   = onRequest((req, res) => applyDelta(req, res, +1));
exports.sunbeamSpend = onRequest((req, res) => applyDelta(req, res, -1));
```

(If Lucid Winds writes to the same `wallets/{uid}.balance`, the currency is automatically shared. Use whatever your existing sunbeam ledger is — the only requirement is the request/response shapes above.)

---

## 6. Wiring Skitterlings to your endpoints

Skitterlings is deployed as a static page (the URL the portal uses). Configure it **at runtime** — pick one:

**A) Same-origin / window reference:**
```js
gameWindow.SKITTERLINGS.configure({
  token: firebaseIdToken,
  endpoints: {
    getBalance: "https://us-central1-PROJECT.cloudfunctions.net/sunbeamBalance",
    addCoins:   "https://us-central1-PROJECT.cloudfunctions.net/sunbeamAdd",
    spendCoins: "https://us-central1-PROJECT.cloudfunctions.net/sunbeamSpend"
  }
});
```

**B) Embedded iframe (portal posts a message):**
```js
iframe.contentWindow.postMessage({ target: "skitterlings", type: "configure",
  data: { token: firebaseIdToken, endpoints: { getBalance, addCoins, spendCoins } } }, GAME_ORIGIN);
```

**C) Launch params** (token via URL is less safe — prefer A/B for the token):
`GAME_URL?uid=<uid>&embed=1&origin=<host>` then push the token via `configure`.

`uid` **must** be the `?uid=` launch param (it namespaces the local save before load). The token + endpoints can be set anytime via `configure`; setting `endpoints` immediately triggers a `getBalance` pull.

**Events you can also listen to** (game → host `postMessage`, `source:"skitterlings"`): `coins {balance,delta,reason}`, `game-over {score,best,coins,world}`, `run-start`, `world-unlocked`, `ready`. Use `coins` to mirror the live sunbeam total in the Lucid Winds / portal UI.

---

## 7. Go-live checklist (sunbeam side)

- [ ] 3 endpoints deployed: `sunbeamBalance` (GET), `sunbeamAdd` (POST), `sunbeamSpend` (POST).
- [ ] Verify Bearer token → uid; reject mismatch.
- [ ] CORS allows the game origin + `Authorization` header + `OPTIONS` preflight.
- [ ] Idempotent on `txid`; atomic balance update (transaction).
- [ ] `spend` enforces affordability server-side; returns real `{balance}` on reject.
- [ ] `reason` in `{launch,dev,host}` ignored (no real credit) in prod.
- [ ] Writes to the shared sunbeam ledger so Lucid Winds and Skitterlings share one balance.
- [ ] Hand me back the 3 URLs → I (or the portal) drop them into `configure`. Test: load → balance pulls; roll → `spendCoins`; finish a run → `addCoins`; reload → balance matches sunbeam.

---

*My side is done and verified. The only integration code I run is in `index.html` section 11 (`Wallet` → `syncRemote`/`pullBalance`). Full portal/embed details are in `INTEGRATION.md`; this doc is the economy-specific slice for you.*
