# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

**Kelly** — an autonomous prediction market trading agent. It trades tokenized Kalshi markets on Solana (via DFlow), runs a deterministic favorite-longshot-fade (FLF) strategy with an LLM veto/sizing layer, and records every judgment in a hash-chained, tamper-evident ledger.

The product was drafted under the working title "SolOracle" — any reference to SolOracle means Kelly. The authoritative spec is `docs/PRD.md`; execute it phase-by-phase (Phase 0 → 5), treating each phase's acceptance criteria as the merge checklist. **Where `docs/assumptions.md` (Phase 0 verification, 2026-08-21) contradicts the PRD, assumptions.md wins** — it records live-API reality (fixed-point dollar fields, V2 order endpoint, DFlow key gating, KYC requirement, etc.).

Reference skills: `.agents/skills/kalshi-docs` is a nightly mirror of official Kalshi docs — trust it. `.agents/skills/dflow` (sendaifun) is **partly stale** (wrong Metadata base URL, invented WS shape/endpoints) — for DFlow, code from `docs/assumptions.md` and github.com/DFlowProtocol/cookbook instead.

## Hard invariants (do not violate)

1. **The LLM never originates trades.** It may only veto a screened candidate or scale its size downward within risk caps. The deterministic screen, risk rails, and ledger are pure TypeScript with zero LLM involvement, and unit-tested.
2. **Fail closed.** A malformed or timed-out LLM response is treated as a veto, never a trade.
3. **Ledger-first.** Every screened candidate produces a hash-chained judgment row (`row_hash` includes `prev_hash`) — including vetoed and risk-skipped candidates. LLM input/output pairs are stored on the row.
4. **Risk rails are enforced in code, not prompts**, and checked before every order: per-position cap, max positions, category exposure cap, daily loss limit (halt until next UTC day), kill switch.
5. **Never place a live order** unless `mode=live` AND the Phase 3 go/no-go gate artifacts exist in the repo.
6. **No public RPC for order flow** — use the dedicated RPC provider from `RPC_URL`.

## Stack & architecture

- **Runtime:** Cloudflare Workers + Durable Objects; agent loop runs on a cron alarm (default 15 min): scan → judge → risk check → execute → manage → calibrate.
- **DB:** Cloudflare D1 (judgments, positions, fills, calibration_snapshots, config, daily_stats).
- **Language:** TypeScript throughout.
- **Solana:** `@solana/web3.js` + `@solana/spl-token`; hot wallet keypair in Workers Secrets.
- **LLM:** Anthropic API.
- **Provider abstraction:** all venue access goes through the `MarketProvider` interface (`listMarkets`, `getQuote`, `buildOrder`, `submit`, `getPositions`, `redeem`). v1 implementations: `KalshiDemoProvider` (paper mode, Kalshi demo REST with RSA-PSS request signing) and `DFlowProvider` (live mode, DFlow Metadata + Trade APIs returning ready-to-sign Solana transactions). Prefer official SDKs (kalshi-typescript, DFlow clients) over hand-rolled signing.

## Commands

Repo is pre-scaffold (Phase 0 in progress); once scaffolded these are the expected commands — update this section as tooling lands:

- `wrangler dev` — run Worker locally
- `wrangler d1 migrations apply <db>` — apply D1 migrations
- Typecheck + unit tests run in CI; keep both green per phase.

## Config

Secrets (wrangler): `KALSHI_API_KEY_ID`, `KALSHI_PRIVATE_KEY_PEM`, `DFLOW_API_KEY`, `SOLANA_KEYPAIR`, `RPC_URL`, `ANTHROPIC_API_KEY`, `BUILDER_CODE`.
Vars: `flf.*` (screen parameters), `risk.*` (rails), `mode` (`paper|live`), `cron_interval_min`. Defaults for every key are in PRD §5.
