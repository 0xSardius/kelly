# Kelly — Product Requirements Document

> **Naming note (2026-08-21):** This product is named **Kelly**. The PRD below was drafted under the working title "SolOracle" — all references to SolOracle mean Kelly.

**Version:** 1.0 (2026-08-21)
**Status:** Ready for build
**Owner:** Justin
**Codename:** SolOracle (working title) → **Kelly** (final)
**Consumer:** Claude Code — this PRD is written to be executed phase-by-phase. Each phase has explicit acceptance criteria. Do not skip Phase 0.

---

## 1. One-liner

An autonomous prediction market trading agent that trades tokenized Kalshi markets on Solana (via DFlow), runs a deterministic favorite-longshot-fade strategy with an LLM veto/sizing layer, and records every judgment in a tamper-evident ledger from day one.

## 2. Problem & Opportunity

- Kalshi's full market catalog is tokenized as SPL tokens on Solana via the DFlow Prediction Markets API (live since Dec 2025). Positions are real SPL tokens with on-chain settlement in USDC.
- Academic evidence (CEPR study of ~300K Kalshi contracts) documents a persistent favorite-longshot bias: cheap contracts (longshots) systematically underperform their implied probability, with average pre-fee returns around -20%.
- Kalshi is subsidizing builders: a $2M+ grants program and Builder Codes that pay developers fees/rewards scaling with volume routed through their integration.
- The generic "LLM bot that bets" is commoditized. The defensible assets are (a) a verified public track record and (b) a named, testable strategy. This PRD builds both, in private mode first.

## 3. Goals & Non-Goals

### Goals (v1)
1. **Autonomous income (primary):** run the longshot-fade strategy on the owner's capital, starting in paper mode, graduating to live microbets.
2. **Ledger-first:** every judgment logged with timestamp, inputs, rule chain, entry price, and outcome — hashed for tamper-evidence — from the first paper trade. This is a hard requirement, not a nice-to-have.
3. **Deterministic strategy core:** the trade screen is rule-based and fully reproducible. The LLM may only veto or size a position, never originate one.
4. **Risk rails:** daily loss limit, per-position cap, liquidity floor, edge floor, resolution-window filter — all enforced in code, not prompts.
5. **Builder Code integration:** register and attach the owner's Builder Code so the agent's own volume earns rebates.

### Non-Goals (v1 — explicitly out of scope)
- Public signal service / x402 monetization (v2)
- Copy-trade vaults or pooled capital (v3; requires legal review)
- Lilly (astrology agent) integration (separate product; shares the provider + ledger layers)
- Any UI beyond a minimal read-only dashboard
- Market making

## 4. Users

- **v1:** single user (the owner). Private deployment.
- **v2+ (design for, don't build):** signal subscribers, grant reviewers, public leaderboard viewers. The ledger schema must support public verification later without migration.

## 5. Strategy Specification (the core IP)

### 5.1 Primary strategy: Favorite-Longshot Fade (FLF)

Systematically take the NO side of overpriced longshots (equivalently, the YES side of underpriced favorites), where the bias is documented.

**Deterministic screen — a market is a candidate if ALL of the following pass:**

| Parameter | Default | Config key |
|---|---|---|
| Longshot price band (YES ask) | 3¢–15¢ | `flf.longshot_band` |
| Favorite price band (YES bid) | 85¢–97¢ | `flf.favorite_band` |
| Min market liquidity (open interest or book depth in $) | $10,000 | `flf.min_liquidity_usd` |
| Min days to resolution | 3 | `flf.min_days_to_resolution` |
| Max days to resolution | 45 | `flf.max_days_to_resolution` |
| Min modeled edge after fees (see 5.3) | 4¢ / contract | `flf.min_edge_cents` |
| Category allowlist | politics, econ, weather, sports | `flf.categories` |
| Exclude markets flagged ambiguous-resolution | true | `flf.exclude_ambiguous` |

**Position logic:**
- Longshot candidate → buy NO (sell the lottery ticket).
- Favorite candidate → buy YES.
- Hold to resolution by default. Optional early exit: take-profit at +15¢, stop-loss at -10¢ (carry-over from the Oracle/Mahoraga design; configurable, off by default in v1 because FLF is a hold-to-resolution thesis).

### 5.2 LLM layer (veto + sizing ONLY)

For each screened candidate, the LLM receives the market question, resolution criteria, current prices, and recent news context, and returns strict JSON:

```json
{
  "veto": false,
  "veto_reason": null,
  "confidence": 0.0-1.0,
  "notes": "string"
}
```

- `veto: true` removes the candidate (e.g., the LLM identifies news that makes the longshot genuinely live, or ambiguous resolution language).
- `confidence` scales position size within the risk caps (see 5.4). It can shrink a position to zero; it can never exceed the cap.
- The LLM must never propose markets, directions, or prices. If the LLM response is malformed or times out → treat as veto (fail closed).
- Every LLM input/output pair is stored in the ledger row.

### 5.3 Edge model & fees

- Kalshi fee formula: `ceil(0.07 × P × (1-P) × 100)/100` per contract per side; DFlow adds fill/market-based fees and Solana tx fees (<$0.01/tx). Model all three.
- Edge = (historical bias-adjusted fair value − market price) − total fees. The bias adjustment table is seeded from published favorite-longshot research and recalibrated from our own resolved ledger every 50 resolutions.
- `min_edge_cents` applies to the post-fee number.

### 5.4 Risk rails (enforced in code, checked before every order)

| Rail | Default | Config key |
|---|---|---|
| Max per-position size | 2% of bankroll | `risk.max_position_pct` |
| Max concurrent positions | 15 | `risk.max_positions` |
| Max exposure per category | 30% of bankroll | `risk.max_category_pct` |
| Daily loss limit (realized + unrealized) | 5% of bankroll → halt trading until next UTC day | `risk.daily_loss_limit_pct` |
| Kill switch | env flag halts all order placement immediately | `risk.kill_switch` |
| Live-mode bankroll (initial) | $500 | `risk.bankroll_usd` |

## 6. System Architecture

### 6.1 Stack

- **Runtime:** Cloudflare Workers + Durable Objects (port patterns from the existing Oracle/Mahoraga harness: gather → analyze → manage → execute loop on a cron alarm).
- **DB:** Cloudflare D1 (ledger, positions, config, calibration).
- **Language:** TypeScript throughout.
- **Solana:** `@solana/web3.js` + `@solana/spl-token`; agent hot wallet keypair stored in Workers Secrets; RPC via a dedicated provider (e.g., Helius/QuickNode) — do not use public RPC for order flow.
- **LLM:** Anthropic API (reuse the multi-provider client from Oracle if ported; Anthropic-only is acceptable for v1).

### 6.2 Provider layer (the key abstraction)

Define a `MarketProvider` interface so venues are swappable (this preserves the Jupiter-Prediction fallback path and future Polymarket support):

```
interface MarketProvider {
  listMarkets(filter): Market[]          // discovery + metadata
  getQuote(marketId, side, qty): Quote   // price + fees + slippage est
  buildOrder(intent): UnsignedTx | Order // venue-specific
  submit(signed): Fill
  getPositions(wallet): Position[]
  redeem(marketId): UnsignedTx           // settled winners → USDC
}
```

**v1 implementations:**
1. `KalshiDemoProvider` — Kalshi demo REST API (paper mode). Auth: API Key ID + RSA-PSS request signing (timestamp + method + path). Use the official kalshi-typescript SDK where possible.
2. `DFlowProvider` — live mode. Metadata API for the Series → Event → Market hierarchy and Yes/No outcome mints; Trade API returns ready-to-sign Solana transactions; positions read by scanning the wallet's SPL token accounts filtered via `filter_outcome_mints`, enriched with the batch markets endpoint; redemption of winning tokens via the same CLP flow. WebSocket feed for price/orderbook updates. Dev endpoints are keyless; production requires an API key from pond.dflow.net.

**Verify at build time (Phase 0):** current DFlow endpoint paths, auth requirements, fee schedule, and Builder Code attachment mechanism. These moved fast in 2026 — treat any hardcoded assumption in this PRD as stale until confirmed against live docs (docs.kalshi.com, pond.dflow.net).

### 6.3 Agent loop (Durable Object, cron every 15 min; configurable)

1. **Scan:** provider.listMarkets → apply deterministic FLF screen.
2. **Judge:** for new candidates, run LLM veto/sizing. Write judgment row (status `pending`) with content hash.
3. **Risk check:** apply all rails against current portfolio state.
4. **Execute:** place order (paper or live). Record fill, fees, tx signature.
5. **Manage:** refresh open positions; enforce daily loss limit; process resolutions; redeem winners; mark ledger rows `resolved` with outcome + P&L.
6. **Calibrate:** every 50 resolutions, recompute bias-adjustment table and per-rule hit rates; write a calibration snapshot row.

### 6.4 Ledger (D1) — minimum schema

```sql
judgments(
  id, created_at, market_ticker, venue, question, resolution_criteria,
  screen_snapshot_json,      -- every screen input at decision time
  llm_input_json, llm_output_json,
  direction, entry_price_cents, size_contracts, fees_cents,
  status,                    -- pending|filled|vetoed|skipped_risk|resolved
  resolved_at, outcome, pnl_cents,
  row_hash,                  -- sha256 of canonical row content
  prev_hash                  -- hash chain for tamper-evidence
)
positions(...), fills(...), calibration_snapshots(...), config(...), daily_stats(...)
```

- Hash-chain every judgment row (`row_hash` includes `prev_hash`) so the private ledger can be publicly verified later without trust. Optionally anchor the head hash on-chain (cheap Solana memo tx) once per day — build the hook, ship it off by default.

### 6.5 Dashboard (minimal, read-only)

Single Workers route serving JSON + one static page: bankroll curve, open positions, resolved P&L, win rate vs. implied probability (calibration plot), rail status (e.g., "daily limit hit"). No auth complexity — protect with Cloudflare Access.

## 7. Revenue Model

1. **Trading P&L** — primary in v1. Honest expectation: sensitive to realized edge; the bear case is negative. Paper phase exists to measure before risking capital.
2. **Builder Code rebates** — attach owner's Builder Code to all live volume. Free basis points on trades the agent makes anyway.
3. **Kalshi/DFlow builder grants** — apply once the paper track record exists (a hash-chained ledger + calibration analysis is a strong grant artifact). Target the "AI agents / analytics" category.
4. **(v2, not built now)** x402 signal service reading directly from the same ledger.

## 8. Build Phases & Acceptance Criteria

### Phase 0 — Verify & scaffold (Days 1–2)
- [ ] Fetch and pin current DFlow Metadata/Trade API docs and Kalshi API docs; record any deltas from this PRD in `docs/assumptions.md`.
- [ ] Register Kalshi demo account + API keys; generate agent Solana keypair; register Builder Code.
- [ ] Repo scaffold: Workers + DO + D1 migrations + wrangler config + typed provider interface. CI: typecheck + unit tests.
- **Accept:** `wrangler dev` runs; D1 migrations apply; a `KalshiDemoProvider.listMarkets` smoke test returns live demo markets.

### Phase 1 — Screen + ledger in paper mode (Week 1)
- [ ] FLF deterministic screen implemented with all config keys from §5.1.
- [ ] Ledger with hash chain; every screened candidate produces a row even when skipped (status matters for calibration).
- [ ] Paper fills against Kalshi demo order book.
- **Accept:** 72-hour unattended run produces ≥20 judgment rows with valid hash chain and zero crashes.

### Phase 2 — LLM veto/sizing + risk rails (Week 2)
- [ ] LLM layer per §5.2 with fail-closed behavior and full I/O logging.
- [ ] All §5.4 rails enforced; unit tests prove the daily loss limit halts execution.
- **Accept:** injected malformed LLM response results in veto, not a trade; simulated -5% day halts trading and dashboard reflects it.

### Phase 3 — Paper campaign (Weeks 3–6)
- [ ] Run continuously; target ≥100 resolved judgments.
- [ ] Calibration job produces bias table + per-rule hit rates.
- **Accept:** written go/no-go memo auto-generated from ledger: realized edge vs. `min_edge_cents`, calibration plot, rail trigger counts. Go-live requires positive post-fee expected value on the resolved sample.

### Phase 4 — DFlow live mode (Weeks 6–8, gated on Phase 3 "go")
- [ ] `DFlowProvider`: quote → build tx → sign → submit → confirm; position scan via token accounts; redemption flow.
- [ ] Builder Code attached; fees reconciled per fill.
- [ ] Live with $500 bankroll and all rails at defaults.
- **Accept:** one full live lifecycle completed end-to-end (entry → resolution → redemption → USDC in wallet) with the ledger row matching on-chain reality.

### Phase 5 — Hardening (ongoing)
- [ ] Alerting (email/Telegram) on: rail trigger, provider errors, wallet balance drift vs. ledger.
- [ ] Daily hash-anchor toggle; weekly automated performance summary.

## 9. Key Risks & Mitigations

| Risk | Mitigation |
|---|---|
| Realized edge ≈ 0 after fees | Phase 3 gate; go-live only on measured positive EV. The ledger + Builder Code + grants still have value if trading doesn't. |
| DFlow API instability / thin on-chain liquidity in small markets | Liquidity floor rail; provider abstraction keeps Kalshi-direct and Jupiter as fallbacks. |
| LLM hallucination inflating confidence | LLM cannot originate trades; deterministic screen + edge floor bound the damage; fail-closed on malformed output. |
| Hot wallet compromise | Minimal bankroll, Workers Secrets, dedicated wallet, daily balance-vs-ledger reconciliation alert. |
| Regulatory posture (US, autonomous trading of own funds on a CFTC-regulated venue) | v1 is owner-only capital — no pooled funds, no signals sold. Revisit with counsel before any v2/v3 public monetization. |
| Ambiguous market resolution | `exclude_ambiguous` flag + LLM veto specifically prompted to check resolution criteria. |

## 10. Config & Secrets (wrangler)

Secrets: `KALSHI_API_KEY_ID`, `KALSHI_PRIVATE_KEY_PEM`, `DFLOW_API_KEY`, `SOLANA_KEYPAIR`, `RPC_URL`, `ANTHROPIC_API_KEY`, `BUILDER_CODE`.
Vars: all `flf.*` and `risk.*` keys from §5, `mode` = `paper | live`, `cron_interval_min`.

## 11. Success Metrics

- **Phase 3:** ≥100 resolved judgments; calibration report generated; go/no-go decision documented.
- **Phase 4 (first 60 live days):** positive net P&L after all fees OR a documented decision to halt; zero rail violations; ledger↔wallet reconciliation clean; Builder Code rebates visibly accruing.
- **Strategic:** ledger head hash chain unbroken since row 1 — the future moat, whatever v2 becomes.

## 12. Explicit instructions to Claude Code

1. Start with Phase 0. Verify live API docs before writing provider code; update `docs/assumptions.md` with anything that contradicts this PRD and flag it.
2. Keep the deterministic screen, risk rails, and ledger free of any LLM involvement — pure TypeScript, unit-tested.
3. Never place a live order unless `mode=live` AND Phase 3 gate artifacts exist in the repo.
4. Prefer official SDKs (kalshi-typescript, DFlow's published clients) over hand-rolled signing.
5. Small PRs per phase; each phase's acceptance criteria are the merge checklist.
