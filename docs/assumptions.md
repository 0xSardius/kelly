# Assumptions & API Reality Check

Phase 0 verification of PRD assumptions against live docs. Convention: **CONFIRMED** (PRD assumption holds), **CHANGED** (reality differs — provider code must follow this doc, not the PRD), **UNVERIFIED** (could not confirm; treat as risk).

Verified: 2026-08-21. Re-verify anything here that matters before Phase 4 go-live.

---

## Kalshi (paper-mode provider)

### Base URLs — CHANGED (new recommended hosts)

| Env | Recommended REST | Legacy (still works) |
|---|---|---|
| Production | `https://external-api.kalshi.com/trade-api/v2` | `https://api.elections.kalshi.com/trade-api/v2` |
| Demo | `https://external-api.demo.kalshi.co/trade-api/v2` | `https://demo-api.kalshi.co/trade-api/v2` |

Demo environment exists; sign up at demo.kalshi.co. Demo and production credentials/API keys are separate.

### Auth — CONFIRMED
- Headers: `KALSHI-ACCESS-KEY`, `KALSHI-ACCESS-SIGNATURE`, `KALSHI-ACCESS-TIMESTAMP`.
- String-to-sign: `timestamp + METHOD + path` where path **includes** `/trade-api/v2` and **excludes** the query string. Timestamp in **milliseconds**.
- RSA-PSS SHA-256, MGF1(SHA-256), salt length = digest length (32; Node: `crypto.constants.RSA_PSS_SALTLEN_DIGEST`), signature base64-encoded.
- WebSocket handshake: same three headers, signing `timestamp + "GET" + "/trade-api/ws/v2"`.

### SDK — CONFIRMED (with caveat)
- Official npm package `kalshi-typescript` (3.26.0, published 2026-07-28, weekly cadence tracking the OpenAPI spec). Handles request signing. It's an OpenAPI-generated axios client with no visible source repo — official per docs.kalshi.com, but treat as generated code. Decision: use it for signing/auth at minimum; wrap behind our provider interface.

### Market data — CHANGED (fixed-point migration; several PRD fields dead)
- Endpoints (relative to `/trade-api/v2`): `GET /series` (has `category` filter — **category lives on series**, deprecated on events, absent on markets), `GET /events`, `GET /markets` (limit ≤1000, cursor pagination), `GET /markets/{ticker}`, `GET /markets/{ticker}/orderbook`.
- Cent-integer price fields are gone: use `yes_bid_dollars`, `yes_ask_dollars`, `no_bid_dollars`, `no_ask_dollars`, `last_price_dollars` (fixed-point strings like `"0.1500"`) and `volume_fp`, `volume_24h_fp`, `open_interest_fp`.
- **`liquidity_dollars` is deprecated and always returns "0.0000"** → our `flf.min_liquidity_usd` rail must be computed from **orderbook depth + open interest**, not a liquidity field.
- `expiration_time` deprecated → `latest_expiration_time` / `expected_expiration_time`. Resolution text: `rules_primary`, `rules_secondary` (still present — feeds LLM veto prompt).
- Orderbook: response is `orderbook_fp` with `yes_dollars`/`no_dollars` arrays of `[price, qty]` string pairs, best-to-worst, **bids only per side** (yes-bid at P ≡ no-ask at 1−P).

### Orders — CHANGED (V2 endpoint)
- Use `POST /portfolio/events/orders` (Create Order V2): `side: bid|ask` (single-book direction, not yes/no+buy/sell), fixed-point dollar `price`, fixed-point `count`, `time_in_force`, `client_order_id`, `post_only`, `reduce_only`. Legacy `POST /portfolio/orders` deprecates "no earlier than May 6, 2026" — do not build on it.
- Portfolio reads: `GET /portfolio/balance` (cents int + `balance_dollars`), `/portfolio/positions`, `/portfolio/fills`, `/portfolio/settlements`. Old fills/orders roll to `/historical/*`.

### Fees — CONFIRMED with additions
- General taker: `ceil_to_cent(0.07 × C × P × (1−P))` — coefficient unchanged (effective Jul 7, 2026).
- S&P 500 / NASDAQ-100 series: **0.035** taker. Maker fee (designated series only): **0.0175**; most markets have free resting orders.
- Rounding is actually to $0.0001 (centicent) with a whole-cent rebate accumulator — model `ceil_to_cent` conservatively.
- Fee changes are queryable: `GET /series/fee_changes`, `GET /events/fee_changes`. UNVERIFIED: exact per-series exception table (fee-schedule PDF behind bot checkpoint — fetch `kalshi.com/docs/kalshi-fee-schedule.pdf` from a browser once).

### Rate limits — CHANGED (token buckets)
- Basic tier: 200 read tokens/s, 100 write tokens/s; most requests cost 10 tokens → **~20 reads/s, ~10 writes/s**. Per-endpoint costs: `GET /account/endpoint_costs`. Free self-serve upgrade to Advanced tier once ≥1 of last 100 orders was API-placed.
- Plenty for a 15-min cron loop; batch market fetches with `limit=1000`.

### WebSocket — CONFIRMED (new hosts; demo has WS)
- Prod `wss://external-api-ws.kalshi.com/trade-api/ws/v2`; demo `wss://external-api-ws.demo.kalshi.co/trade-api/ws/v2`.
- Channels: `ticker_v2`, `orderbook_delta`, `trade`, `fill`, `market_positions`, `market_lifecycle_v2`, etc. Subscribe: `{"id":1,"cmd":"subscribe","params":{"channels":[...],"market_tickers":[...]}}`.
- v1 uses REST polling on the cron; WS is optional later.

### Builder Codes — CHANGED (important)
- **There is no builder-code field on the Kalshi REST Trade API.** Builder Codes (announced Dec 2025) are the crypto-side program: attribution and fees run through the **DFlow / Jupiter on-chain integrations** — on DFlow via `platformFeeBps` + `feeAccount` on the trade/swap build. Permissionless to earn.
- Consequence: **paper mode (Kalshi demo REST) earns nothing and needs no builder code; the Builder Code only matters in Phase 4 (DFlow live).**
- Grants: kalshi.com/builders — "$2M in Grants & Developer Support", co-sponsored by Solana and Base; AI agents are an eligible category. (The Superteam Earn "Kalshi Builder Codes Grants" listing is closed; apply via kalshi.com/builders.)
- UNVERIFIED: exact revenue-share/rebate percentages (not published; DFlow onboards via hello@dflow.net).

### PRD impact summary (Kalshi)
1. `KalshiDemoProvider` targets `external-api.demo.kalshi.co`, Create Order V2, fixed-point dollar fields.
2. Liquidity rail computed from orderbook depth + `open_interest_fp` (no liquidity field exists).
3. Category screen implemented via **series** category, joined onto markets.
4. Internally we keep integer cents (per build plan); convert at the provider boundary from fixed-point dollar strings.
5. Builder Code work moves entirely to Phase 4 (DFlow); nothing to attach in paper mode.

---

## DFlow (live-mode provider)

### HEADLINE — CHANGED: keyless dev tier withdrawn; production API key is the critical path

Between 2026-06-08 and 2026-08-21, DFlow removed all prediction-markets docs from pond.dflow.net and shut the keyless dev tier:

- `dev-prediction-markets-api.dflow.net` (dev Metadata API + WS) — **NXDOMAIN, gone**.
- `dev-quote-api.dflow.net` still serves keyless **spot** quotes, but returns `route_not_found` for every prediction-market outcome mint (routing disabled on dev).
- Production hosts are alive but **403 without `x-api-key`**: Trade API `https://quote-api.dflow.net`, Metadata API `https://prediction-markets-api.dflow.net`.
- The API itself is fully operational (live markets/trades verified 2026-08-21); it's the *access* that became partner-gated.

**Consequence:** nothing prediction-related can be built or tested against DFlow without a production key. **Request it now** (Phase 0, not Phase 4): Google Form linked from pond.dflow.net/get-started/api-key + email hello@dflow.net stating the prediction-markets use case; 2–5 business-day SLA. Ask explicitly for: prod Metadata access, prediction routing on the Trade API, the private WS URL, current fee/rebate schedule, and whether a partner sandbox exists.

### Base URLs — CHANGED (and the installed skill is wrong)

| Service | URL | Notes |
|---|---|---|
| Trade API | `https://quote-api.dflow.net` | `/order`, `/order-status`; key required |
| Metadata API | `https://prediction-markets-api.dflow.net` | `/api/v1/*`; key required |
| Metadata WS | `wss://prediction-markets-api.dflow.net/api/v1/ws` | private per-partner endpoint; key required |
| Proof KYC | `https://proof.dflow.net` | `GET /verify/{address}` |

⚠️ **The sendaifun skill at `.agents/skills/dflow` is partly stale/incorrect**: its Metadata base URL `api.prod.dflow.net` does not resolve, its WS URL/shape is invented, and endpoints like `/api/v1/milestones/*` and `/api/v1/categories` don't exist. Its `filter_outcome_mints`, `markets/batch`, orderbook, and general endpoint catalog are broadly right. **Code from this file, not the skill.** Real official examples: github.com/DFlowProtocol/cookbook (active, prediction-markets scripts included).

### Auth & KYC — CHANGED (KYC is new information)

- `x-api-key` header on every request including WS upgrade. (Workers note: setting headers on a WS upgrade requires `fetch` with `Upgrade: websocket`, not `new WebSocket()`.)
- **Buying outcome tokens requires the receiving wallet to pass Proof KYC** (Stripe Identity, free, at dflow.net/proof) — enforced at `/order` time. Selling/redeeming doesn't require it; quotes without `userPublicKey` work unverified. → **The agent's hot wallet must complete Proof KYC before Phase 4**; gate buys on `GET https://proof.dflow.net/verify/{address}`.
- Trade API sets no CORS headers; server-side calls only (fine — we're a Worker).
- Rate limits: no published numbers. Poll `/order-status` at ~2s.

### Trade flow — CONFIRMED (RFQ/swap-style, async fills)

- `GET /order` is quote + ready-to-sign transaction in one call (mint-in → mint-out, Jupiter-style). Prediction trades are **always async**: the tx escrows stablecoin + limit params; DFlow's settlement authority fills against Kalshi (limit IOC) and settles on-chain; unfilled amounts refund in `revertMint`.
- Key params: `inputMint`, `outputMint`, `amount` (scaled int, 6dp; 1 contract = 1_000_000), `userPublicKey`, `slippageBps`, `predictionMarketSlippageBps`, `allowAsyncExec`, `predictionMarketInitPayer` (~0.02 SOL if we're first to tokenize a market), `outcomeAccountRentRecipient`.
- We sign (`VersionedTransaction.deserialize` → sign → send) and submit to **our own RPC**; DFlow does not submit. Then poll `GET /order-status?signature=...&lastValidBlockHeight=...` → `pending|open|pendingClose|closed|expired|failed` with `fills[]`/`reverts[]`.
- **Kalshi clearing maintenance Thursdays 3:00–5:00 AM ET — orders revert; block order submission in that window** (add to risk rails as a trading-calendar check).
- Minimum order 0.01 USDC but ≥1 whole contract (no fractional trading unless market flags it).

### Metadata endpoints — CONFIRMED (catalog verified live/archive)

- `GET /api/v1/events` (`withNestedMarkets`, `status`, cursor), `GET /api/v1/markets`, `GET /api/v1/market/{ticker}`, `GET /api/v1/market/by-mint/{mint}`, `POST /api/v1/markets/batch` ({tickers?, mints?} ≤100), `GET /api/v1/outcome_mints`, `POST /api/v1/filter_outcome_mints` ({addresses} ≤200), `GET /api/v1/orderbook/{ticker}` → `{sequence, yes_bids: {price: size}, no_bids: {...}}`, `GET /api/v1/trades`, `GET /api/v1/series`, `GET /api/v1/search`, candlesticks per event/market.
- Markets are only tokenized on-chain after their first DFlow trade (`isInitialized`); omit the `isInitialized` filter to see the full catalog.
- Status lifecycle: `initialized → active → inactive ⇄ active → closed → determined → finalized`; only `active` trades; redemption at `determined|finalized` gated by `accounts[settlementMint].redemptionStatus === "open"` and `result` (`"yes"|"no"|""` + `scalarOutcomePct` for scalar markets).

### Positions — CONFIRMED (one delta: Token-2022)

Flow exactly as PRD assumed, with one correction: **outcome tokens are Token-2022 mints**, so position scans use `getParsedTokenAccountsByOwner(wallet, {programId: TOKEN_2022_PROGRAM_ID})` → `filter_outcome_mints` → `markets/batch` (mints keyed by settlement mint USDC `EPjFW...` or CASH `CASHx...`).

### Redemption — CONFIRMED

Same `/order` swap: winning outcome mint → settlement mint at 1.00. Losing tokens have no route — **burn them to close the account and reclaim rent** (rent to `outcomeAccountRentRecipient`). Winning redemption auto-closes the emptied account.

### Fees & Builder Codes — CHANGED vs PRD framing

- DFlow charges **builders per API key**, Kalshi-style probability-weighted: base tier ≈ `roundup(0.07·c·p·(1−p)) + 0.01·c·p·(1−p)` (taker scale 0.09 at <$50M/30d volume, tiering down to 0.08). This replaces paying Kalshi directly — the edge model's fee term uses DFlow's builder fee schedule, not Kalshi's retail formula, in live mode.
- **No Kalshi Builder Code parameter exists in the DFlow API.** The monetization analogues are: (1) our own platform fee via **`platformFeeScale`** (0–999, 3 decimals, applied to `c·p·(1−p)`, paid in stablecoin to our `feeAccount`; note `platformFeeBps` is spot-only), and (2) **DFlow's per-key rebate program** (>$100k/30d volume: ≥3% of gross fees, VIP tiers to 30%).
- March-2026 numbers; may have been renegotiated into partner terms — confirm on key issuance. UNVERIFIED: current partner fee schedule.

### SDKs — CHANGED
No prediction-markets SDK exists (npm `@dflow-protocol/*` packages predate prediction markets). Build `DFlowProvider` on raw `fetch` + web3.js, cribbing from DFlowProtocol/cookbook.

### PRD impact summary (DFlow)
1. **New Phase 0 owner task, critical path: request DFlow production API key** (2–5 day SLA) and **KYC the agent wallet via Proof**.
2. Edge model live-mode fee term = DFlow builder fee tiers (0.09 taker base), not Kalshi retail 0.07.
3. "Builder Code attachment" (PRD §Phase 4) becomes: set `platformFeeScale` + `feeAccount` if we want to collect our own fee, plus per-key rebates automatically. Nothing to attach per-order beyond the API key itself.
4. New risk rail: block order submission during Kalshi clearing maintenance (Thu 3–5 AM ET).
5. Position scan uses Token-2022 program, not classic SPL.
6. Add losing-token burn (rent reclaim) to the manage step.
