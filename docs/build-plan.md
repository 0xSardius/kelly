# Kelly — Build Plan

Companion to `docs/PRD.md`. This translates the PRD phases into concrete tasks for this repo. Check items off as they land; each phase's PRD acceptance criteria are the merge gate.

## Proposed repo layout

```
kelly/
├── src/
│   ├── index.ts               # Worker entry: fetch routes + scheduled (cron) handler
│   ├── agent/
│   │   ├── loop.ts            # Durable Object: scan → judge → risk → execute → manage → calibrate
│   │   ├── screen.ts          # Deterministic FLF screen (pure functions, no I/O)
│   │   ├── edge.ts            # Kalshi fee formula + DFlow fees + bias-adjusted edge model (pure)
│   │   ├── risk.ts            # All rails from PRD §5.4 (pure, unit-tested)
│   │   ├── judge.ts           # LLM veto/sizing client — strict JSON parse, fail-closed
│   │   └── calibrate.ts       # Bias table recompute every 50 resolutions
│   ├── ledger/
│   │   ├── ledger.ts          # Judgment writes, status transitions, D1 access
│   │   └── hash.ts            # Canonical serialization + sha256 hash chain
│   ├── providers/
│   │   ├── types.ts           # MarketProvider interface + shared Market/Quote/Fill/Position types
│   │   ├── kalshi-demo.ts     # Phase 0–3 (paper)
│   │   └── dflow.ts           # Phase 4 (live)
│   ├── dashboard/
│   │   └── routes.ts          # Read-only JSON + one static page
│   └── config.ts              # Typed config: flf.*, risk.*, mode, cron_interval_min
├── migrations/                # D1 SQL migrations (0001_init.sql, ...)
├── test/                      # Vitest — screen, edge, risk, hash chain, judge fail-closed
├── docs/
│   ├── PRD.md
│   ├── assumptions.md         # Deltas between PRD and live API docs (Phase 0 output)
│   └── go-no-go.md            # Phase 3 gate artifact (auto-generated from ledger)
├── wrangler.toml
├── package.json
└── .github/workflows/ci.yml   # typecheck + vitest
```

Tooling: `vitest` with `@cloudflare/vitest-pool-workers` (runs tests inside workerd, real D1 bindings), `typescript` strict mode, `wrangler` v4.

## Phase 0 — Verify & scaffold (Days 1–2)

**Claude tasks:**
- [ ] Research pass: fetch current DFlow Prediction Markets API docs (pond.dflow.net / docs) and Kalshi API docs (docs.kalshi.com). Pin: endpoint paths, auth, fee schedule, Builder Code attachment, demo-env behavior, kalshi-typescript SDK status. Write `docs/assumptions.md` with every delta from the PRD.
- [ ] Scaffold: `package.json`, `tsconfig.json` (strict), `wrangler.toml` (DO binding, D1 binding, cron trigger, vars), `migrations/0001_init.sql` (full §6.4 schema — all six tables, not just judgments), CI workflow.
- [ ] `providers/types.ts` — the `MarketProvider` interface, typed against what the research pass confirmed.
- [ ] `KalshiDemoProvider.listMarkets` + RSA-PSS request signing (or SDK) + a smoke test.

**Owner (Justin) tasks — only you can do these:**
- [ ] Register Kalshi demo account at demo.kalshi.co, create API Key ID + RSA private key.
- [ ] **Request DFlow production API key NOW** — critical path, 2–5 business-day SLA (Google Form via pond.dflow.net/get-started/api-key + email hello@dflow.net stating prediction-markets use). The keyless dev tier was withdrawn mid-2026; nothing DFlow-side works without it. See assumptions.md for what to ask for.
- [ ] Generate the agent Solana keypair (dedicated, never reused), then **complete Proof KYC for that wallet** at dflow.net/proof — DFlow blocks outcome-token buys for unverified wallets. Needed by Phase 4 but KYC + key SLA make it worth starting early.
- [ ] Cloudflare account ready; `wrangler login`; create D1 database.
- [ ] Anthropic API key; dedicated RPC provider key (Helius/QuickNode) — needed by Phase 4, fine to defer.
- ~~Register Builder Code~~ — no such API mechanism exists (see assumptions.md §Builder Codes): monetization is `platformFeeScale`/`feeAccount` + per-key rebates on DFlow, and the grants program at kalshi.com/builders.

**Accept:** `wrangler dev` runs; migrations apply; `KalshiDemoProvider.listMarkets` smoke test returns live demo markets.

## Phase 1 — Screen + ledger, paper mode (Week 1)

- [ ] `screen.ts`: full §5.1 screen, every threshold from config, returns a per-market pass/fail trace (which rule failed — feeds calibration).
- [ ] `hash.ts` + `ledger.ts`: canonical JSON serialization (stable key order), `row_hash = sha256(canonical(row) + prev_hash)`, genesis row handling, chain-verify function (used by tests and a dashboard endpoint).
- [ ] Every screened candidate writes a row — filled, vetoed, or skipped.
- [ ] Paper fill engine against the Kalshi demo order book (cross the spread at ask/bid, record modeled fees).
- [ ] Durable Object loop wired to cron; idempotent per market per scan (no duplicate judgments for an unchanged candidate).
- [ ] Tests: screen edge cases (band boundaries, liquidity floor, resolution window), hash chain integrity, chain-verify catches a mutated row.

**Accept:** 72-hour unattended run → ≥20 judgment rows, valid hash chain, zero crashes.

## Phase 2 — LLM veto/sizing + risk rails (Week 2)

- [ ] `judge.ts`: Anthropic call with strict JSON schema parse; malformed/timeout/refusal → veto (fail closed); full I/O logged to the ledger row.
- [ ] `risk.ts`: all §5.4 rails as pure functions over (portfolio state, candidate, config); checked before every order; daily loss limit sets a halt flag until next UTC day; kill-switch env flag short-circuits execution.
- [ ] Sizing: `size = min(confidence × max_position, all caps)` — LLM can only shrink.
- [ ] Tests: injected malformed LLM output → veto row, no order; simulated -5% day → halt + dashboard reflects it; category/position caps enforced.

**Accept:** both PRD §8 Phase-2 injection tests pass.

## Phase 3 — Paper campaign (Weeks 3–6)

- [ ] Continuous run; monitor via dashboard.
- [ ] `calibrate.ts`: every 50 resolutions recompute bias table + per-rule hit rates → calibration_snapshot row.
- [ ] Go/no-go memo generator: reads ledger → `docs/go-no-go.md` (realized edge vs `min_edge_cents`, calibration plot data, rail trigger counts).

**Accept:** ≥100 resolved judgments; memo generated; go-live only on positive post-fee EV.

## Phase 4 — DFlow live mode (Weeks 6–8, gated on Phase 3 "go")

- [ ] `dflow.ts`: metadata (Series → Event → Market, outcome mints), quote, build-tx → sign (Workers Secrets keypair) → submit → confirm; position scan via SPL token accounts + batch markets enrichment; redemption flow.
- [ ] Builder Code attached to every order; per-fill fee reconciliation.
- [ ] Live-order guard: refuses unless `mode=live` AND `docs/go-no-go.md` exists with a "GO" verdict.
- [ ] $500 bankroll, all rails at defaults.

**Accept:** one full live lifecycle (entry → resolution → redemption → USDC in wallet) with ledger row matching on-chain reality.

## Phase 5 — Hardening (ongoing)

- [ ] Alerts (email/Telegram): rail triggers, provider errors, wallet-vs-ledger balance drift.
- [ ] Daily hash-anchor toggle (Solana memo tx, off by default); weekly performance summary.

## Standing decisions

- Package manager: npm unless you say otherwise.
- Tests: Vitest + workers pool; the pure modules (`screen`, `edge`, `risk`, `hash`) must be testable without any binding.
- PRD specifies `@solana/web3.js`; DFlow returns ready-to-sign transactions so Solana surface area is small (deserialize, sign, send). If research shows DFlow's own client handles this, prefer it.
- Money is integer cents everywhere; contract counts are integers. No floats in ledger or risk math (floats allowed only inside the edge model's probability calcs, rounded at the boundary).
- One PR per phase; acceptance criteria are the merge checklist.
