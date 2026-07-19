# Fork Audit & Roadmap

> Audit of the LibreFolio codebase performed on 2026-07-19 (fork: `adf513/LibreFolio`,
> synced with `Librefolio/LibreFolio` at `1eefb608`). Purpose: assess LibreFolio as the
> starting point for a personal portfolio management tool with richer performance
> analytics and, eventually, options trading support.

## Verdict

**Adapt this codebase — do not start from scratch.** The engineering quality is far
above typical open-source portfolio trackers. Rebuilding what already works here
(transaction model, FX routing, broker importers, performance math, test suite) would
cost 6–12 months before reaching feature parity. Effort should go into the two things
the project lacks: **risk/performance analytics** and **options support**.

For calibration: the popular alternatives (Ghostfolio, Wealthfolio, Portfolio
Performance) also lack options support and have less rigorous performance math.
Starting from scratch would only make sense for an options-first product with
real-time Greeks — a different tool requiring a market-data subscription anyway.

## Codebase snapshot

| Area | Details |
|---|---|
| Backend | FastAPI + SQLModel/Alembic + SQLite, ~42k lines Python |
| Frontend | SvelteKit / Svelte 5 + Tailwind 4 + ECharts 6, ~96k lines TS/Svelte |
| API client | Auto-generated from OpenAPI with Zod validation (`openapi-zod-client`) |
| Tests | 1,600+ backend test functions (~90% coverage), Playwright e2e, Vitest unit |
| i18n | en / es / fr / it |
| Docs | MkDocs site, developer wiki, developer journal |

## Strengths

1. **Portfolio engine architecture** (`backend/app/services/portfolio_engine.py`).
   Layered pipeline: transaction classifier (internal vs. external flows) → *pure,
   no-I/O* daily-state builder → derived views (summary/history/allocation/
   performance). Pure-function cores are easy to test and extend — this directly
   matters for adding new metrics.
2. **Rigorous data model** (`backend/app/db/models.py`). Unified `Transaction` table
   with explicit sign rules per type, linked transaction pairs (transfers, FX
   conversions), per-broker cash pools, asset events (dividends, splits, maturity).
   Every enum documented in-line. This is the hardest part of a portfolio tool to get
   right, and it is already right.
3. **Real performance math** (`backend/app/utils/financial/roi_utils.py`). TWRR,
   MWRR (XIRR via NPV root-finding), simple ROI — computed over a daily NAV vector,
   not the naive "current value / invested" of most hobby tools.
4. **FIFO v3 lot engine** (upstream, merged into this fork). Per-lot tracking with a
   lots UI — the foundation any options P&L work needs.
5. **12 broker import parsers (BRIM)** (`backend/app/services/brim_providers/`):
   IBKR, Schwab, Trading212, Degiro, Revolut, eToro, Coinbase, Directa, Finpension,
   Freetrade, generic CSV — with sample report fixtures. The single biggest
   contributor to frictionless UX; miserable to rebuild.
6. **Multi-currency done properly**: FX providers for ECB, BoE, Fed, SNB with
   conversion routing, plus `DataQualityReport` that surfaces data gaps instead of
   silently producing wrong numbers.
7. **Fingerprint-keyed blob caching** of engine results with range slicing —
   recomputation only when underlying data actually changes.
8. **AI export to clipboard** already exists — a hook for LLM-assisted analysis.
9. **Active upstream** — fast-moving, high commit quality, conventional commits.

## Weaknesses / inefficiencies

1. **No risk analytics.** No Sharpe, Sortino, max drawdown, volatility, beta, or
   benchmark-relative comparison anywhere in the backend — despite an `INDEX` asset
   type existing specifically for benchmarks. Easiest gap to close: the daily NAV
   series the metrics need already exists.
2. **No derivatives support.** `AssetType` has no OPTION; no strike/expiry/
   multiplier/underlying concepts anywhere.
3. **Pure-Python `Decimal` daily loop.** The engine iterates day × asset × broker in
   Python. Fine for personal scale (caching mitigates), but a ceiling exists for
   backtesting-style workloads. Not worth fixing now.
4. **Scraper-fragile price providers.** Yahoo Finance (curl-cffi impersonation),
   JustETF and Borsa Italiana scrapers pinned as git dependencies — break when sites
   change. No options-chain data source.
5. **SQLite-only; in-process caches; JWT secret regenerated on restart if unset** —
   acceptable for single-user self-hosting, limiting for multi-user.
6. **Italian-language internal docs** (`TODO_FUTURI.md`, developer journal) and an
   Italian-market tilt. UI itself is fully English.
7. **Large custom frontend surface**: hand-rolled Svelte 5 adapter for TanStack Table
   v8, custom chart layer — high quality but a big surface to learn before making UI
   changes confidently.

## Roadmap

### Phase 0 — Sync and baseline ✅ / in progress

- [x] Sync fork with upstream (`18d60eeb` → `1eefb608`, fast-forward, pushed).
- [ ] Run the backend test suite green locally.
- [ ] Get the dev server running; import real broker history via BRIM; verify
      numbers against broker statements. Trust in the numbers underpins everything.

### Phase 1 — Analytics layer ("more informative about my performance")

Highest value/effort ratio. Build a new `risk_utils.py` beside `roi_utils.py` —
pure functions over the existing daily NAV series:

- Max drawdown + drawdown curve, rolling volatility, Sharpe / Sortino,
  rolling 1y / 3y returns.
- **Benchmark comparison**: the `INDEX` asset type already fetches index prices —
  show "my TWRR vs. S&P 500 / MSCI World over the same period", plus a *what-if*
  line ("if every deposit had bought VWCE instead"). The most honest performance
  insight a personal tool can give.
- Per-asset contribution to return; income view (dividend/interest calendar,
  trailing-12-month yield).
- One new "Analytics" page in the frontend (ECharts already available).
- **Consider contributing this phase upstream** — risk metrics are universally
  wanted; if merged, the fork stays small and mergeable permanently.

### Phase 2 — Options support (incremental)

1. **Model**: add `AssetType.OPTION` with `underlying_asset_id`, `strike`, `expiry`,
   `right` (call/put), `multiplier` (100); additive Alembic migration. Reuse
   BUY/SELL for open/close (quantity = contracts, amount = premium × multiplier —
   existing sign rules fit). Add `AssetEventType` EXERCISE / ASSIGNMENT / EXPIRE
   (EXPIRE maps naturally onto the existing MATURITY_SETTLEMENT semantics).
2. **Import**: extend the existing IBKR/Schwab BRIM parsers to recognize OCC option
   symbols — replaces manual entry from day one.
3. **Metrics that matter for premium strategies**: realized premium captured per
   month, win rate, annualized return on collateral (CSPs / covered calls), open
   exposure by expiry and underlying, strategy grouping (link legs of a spread /
   wheel via a strategy tag). Skip live Greeks initially — daily marks from free
   sources are unreliable; realized-P&L analytics are what inform behavior.
4. **Engine**: options flow through the existing daily-state builder as another
   asset with a multiplier; expiry becomes a maturity-style event the engine
   already understands.

### Phase 3 — Frictionless UX

- **IBKR Flex Queries** scheduled auto-import (APScheduler infrastructure already
  exists) — eliminates manual CSV export, the biggest remaining friction.
- Surface data-gap / stale-price nudges from the existing `DataQualityReport`
  instead of silent staleness.

### Extra-value ideas

- **MCP server over the existing API** so an LLM assistant can answer questions
  like "how did my covered calls on NVDA do this quarter vs. just holding?"
  directly against live data. The OpenAPI schema makes this cheap to build and it
  upgrades the existing AI-export feature into a live assistant.
- **Decision journal**: attach a thesis/note at trade time, review at close —
  pairs well with options win-rate stats.

## Fork-maintenance policy

Upstream moves fast. To keep merges cheap:

- Sync from upstream frequently (`git fetch upstream && git merge upstream/main`).
- Keep changes **additive**: new modules, new routes, new migrations — avoid
  wholesale rewrites of core engine files.
- Contribute generic features (Phase 1 analytics) upstream where possible.
