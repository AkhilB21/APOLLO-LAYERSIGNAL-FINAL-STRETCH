Deepseek

# Apollo Web Dashboard — Architecture v1.0

A React (Vite) + FastAPI dashboard that turns the Apollo project's existing artifacts
into a decision surface for the positional-momentum workflow. It replaces the current
3-tab proof-of-concept UI with a 7-tab, data-driven dashboard.

## Principles

1. **Artifacts are the source of truth.** Every tab reads an existing project output
   (parquet, JSON, SQLite, CSV). No new database, no re-derivation of engine state.
2. **Non-invasive.** The dashboard only *reads*; it never writes to engine outputs.
   The one write target is the accumulator panel parquet (Market Intel), owned by a
   small append-only writer.
3. **Flags inform, never gate.** All guidance/screener flags render as badges; no
   tab makes an entry/exit decision.
4. **Single-user, local-first.** No auth. Designed to run on the user's Windows box
   (localhost:5173 → proxy → localhost:8000) and previewable in the hosted sandbox
   via the Vite `allowedHosts` wildcard.

## Data sources (grounded in the repo)

| # | Feed | Path / source | Contents |
|---|------|---------------|----------|
| 1 | Live screener | Google Sheet via `apollo_core/gsheet_repo.py` | Per-ticker CMP, %Change, VPT, %dist 50-SMA, Stoch, Proximity 52-WH, Vol Spread |
| 2 | Live monitor | `live_state.db` (SQLite, `live_engine/state_store.py`) | `monitor_state` (in_trade, entry price/date, peak_rsi, trail_activation, cooldown, pending_exit) + `alerts` |
| 3 | Post-trade | `output/trades.parquet`, `output/profiles/{SYMBOL}.parquet+.md`, `output/mae_mfe_*.parquet`, `output/aggregation_summary.*` | TradeRecords, per-stock behavioral flags, MAE/MFE, systemic aggregations |
| 4 | Audit | `audit_report.json/html` | Per-symbol severity/category issues, health score |
| 5 | Simulator | `simulator/` run outputs (JSON/CSV/parquet) | walk-forward OOS, indicator sweeps, regime analysis |
| 6 | Universe | `apollo_universe.json`, `smallcap500.json`, `etmoney_stocks_list.json`, `my_watchlist.json` | Symbol sets + watchlist |
| 7 | Market intel | Daily reports (screener CSVs, RSI dashboards, gainers) + accumulator panel `market_intel/panel.parquet` | Daily membership `date\|symbol\|list_source\|rank\|pct_chg\|ltp\|sector` |

## API contract (FastAPI, extends `apollo_api/main.py`)

All responses `{ "status": "ok"|"error", "data": ..., "meta": { "generated", "data_cutoff" } }`.
Every endpoint serves cached filesystem data; cache invalidated by file mtime.

```
GET /api/live-metrics                 → feed 1 (GSheet, TTL 60s)          [exists]
GET /api/positions                    → feed 2, monitor_state rows        [new]
GET /api/alerts                       → feed 2, alerts feed               [new]
GET /api/audit/report                 → feed 4, audit_report.json         [new]
GET /api/audit/issues?symbol=&severity= → feed 4, filtered                [new]
GET /api/guidance/trades              → feed 3, trades.parquet            [new]
GET /api/guidance/profiles            → feed 3, all profiles (flags)      [new]
GET /api/guidance/profile/{symbol}    → feed 3, one profile + .md          [new]
GET /api/guidance/mae-mfe             → feed 3, MAE/MFE report            [new]
GET /api/guidance/aggregation         → feed 3, aggregation summary       [new]
GET /api/simulator/results            → feed 5, latest run outputs        [new]
GET /api/market-intel/panel           → feed 7, accumulator panel         [new]
GET /api/universe                     → feed 6, universe + watchlist      [new]
GET /api/health                       → engine/module status, config_hash [new]
```

`apollo_api/main.py` currently defines only `/api/live-metrics`. Add a thin
router module per tab (`apollo_api/routes/{live,guidance,audit,simulator,intel}.py`)
so the file stays small. Shared helpers (`_read_json`, `_read_parquet`,
`_mtime_cached`) live in `apollo_api/utils.py`.

## Frontend (React + Vite, `apollo_ui/`)

- **Routing:** keep lightweight — single `App.jsx` with view-state (as today) OR
  react-router if deep-linking per symbol is wanted. Recommend view-state for now.
- **Data fetching:** `axios` (already present). A `useApi(resource, pollMs)` hook
  centralizes polling + error handling + `lastUpdated`.
- **Charting:** none installed today. Add **`lightweight-charts`** (tradingview)
  for score/price charts — smallest footprint, candlestick-native.
- **Styling:** keep existing CSS-variable theme (slate/cyan). Add a small
  component set: `Badge`, `FlagChip`, `ProximityBar` (exists), `MetricBadge` (exists),
  `SortableTable`, `Sidebar` (exists), `StatCard`.

### Tab structure

| Tab | Purpose | Primary feeds | Key views |
|-----|---------|---------------|-----------|
| **Live Screen** | Real-time NSE 500 screener | 1, 6 | Sortable/filterable table; sector filter; circuit-limit badge; RSI-stack flag; 52-WH proximity bar; gap badges |
| **Positions & Alerts** | Live monitor, human-in-the-loop | 2 | In-trade cards (entry, trail, cooldown), pending-exit confirm/dismiss, alert feed, score history chart |
| **Market Intel** | Momentum-list builder | 7, 6 | 52-WH proximity ranks, gainers/losers membership, multi-TF RSI stack, sector breadth, accumulator panel trends |
| **Post-Trade / Guidance** | What worked / per-stock flags | 3 | Trade records table, per-stock profile drill-down (flags, MAE/MFE, n-count), rule suggestions, aggregation summary |
| **Backtest Lab** | Simulator results | 5 | Walk-forward OOS table, indicator sweep charts, regime analysis; engine version + config_hash shown per run |
| **Data Health** | Feed integrity | 4 | Health score, errors/warnings by category, per-symbol drill-down, coverage |
| **Engine Health** | System status | filesystem + config | Module status (real, not hardcoded), last data cutoff, config-drift check, universe size |

### Recommended build order (each lands independently)

1. **API layer**: `utils.py` + all `/api/*` routes (thin, no UI dependency).
2. **Positions & Alerts** — highest value, already-real data (SQLite today).
3. **Post-Trade / Guidance** — richest analytical content.
4. **Live Screen** upgrades (sector filter, badges, sorting).
5. **Data Health** (simple JSON-driven).
6. **Market Intel** — depends on accumulator panel writer existing.
7. **Backtest Lab** + **Engine Health** (real status, not hardcoded).

## Accumulator panel (Market Intel prerequisite)

Append-only parquet written daily by `nse_engine`/report ingest:

```python
# market_intel/panel.parquet  (append-only, partitioned by date)
# schema: date | symbol | list_source | rank | pct_chg | ltp | sector
# list_source in {screener, rsi_daily, rsi_weekly, gainers_daily, gainers_weekly}
```

Writer: `market_intel/accumulate.py` — idempotent per (date, source); no delete.
This is the ONLY dashboard-related write, and it is a pure log, not engine state.

## Open decisions (defaults chosen, reversible)

1. **Deployment:** local Windows (localhost). Sandbox preview works via
   `allowedHosts`. *Default kept.*
2. **Live screener source:** keep Google Sheet (feed 1) as-is — it already works.
   *No migration to local artifacts in v1.*
3. **Market Intel embedding:** dashboard reads the accumulator panel + raw report
   CSV listing; does not render Chartink HTML. *Default kept.*
4. **Priority order:** Positions & Alerts, then Post-Trade/Guidance. *Default kept.*
5. **Auth:** none (single-user). *Default kept.*
6. **Simulator surfacing:** summary + drill-down tables per run. *Default kept.*


REVIEWER'S GUIDANCE: (This is guidance only, not binding and not obligatory on our part to follow it. We only implement what we agree to, if we agree to none, we implement none of the reviewer's suggestions).

GEMINI U/UX Designed

Based on the detailed architecture document, the HTML/JS source code, and the visual patterns established in the LayerSignal mockups (`image_ac681b.png` and `image_ac5cd3.png`), here is the comprehensive UI/UX design specification for the unified dashboard.

The design retains the extremely high data density, 3-column layout paradigm, and dark mode aesthetic (`#16161A` base) of the reference material while organizing the expansive Apollo and LayerSignal feature sets into a logical, mentally cohesive workflow.

---

## 1. Navigation Structure & Tab Hierarchy

To manage the massive feature inventory without overwhelming the user, the dashboard utilizes a **Global App Bar** combined with a **Dual-Mode Workspace** (Apollo vs. LayerSignal).

**Global App Bar (Persistent)**

* **Top Left:** Branding (`LAYER SIGNAL`), Mode Switcher Toggle (`[Apollo: Backtest] / [LayerSignal: Live]`), Global Stock Search.
* **Top Right:** Alert Center with badge (Unified Alerts 3.2), Command Center Status (Kite API 3.4), Engine Health Indicator (1.9), Global Settings.
* **Bottom Footer:** Scrolling Live Ticker (3.1.1), Market Regime Banner (3.1.3), VIX/Nifty Status (3.1.2, 3.1.4).

**Workspace 1: APOLLO (Historical, Analytics, Scoring)**

* **Tab A:** Market Intel (Universe Screener, IPO Scoring)
* **Tab B:** Stock Deep Dive (Single Stock Analytics, Cross-Report, Guidance)
* **Tab C:** Trade Engine (Backtest Logs, Post-Trade Analysis)
* **Tab D:** System Validator (Profile Analytics, Walk-Forward Validation)

**Workspace 2: LAYERSIGNAL (Live Monitoring, Execution, Scanning)**

* **Tab A:** Command & Scanner (Market Scanner, Sector Heatmap)
* **Tab B:** Live Watchlist (RSI Stacks, Multi-Timeframe Entry/Exit Layers)
* **Tab C:** Execution Desk (Position Sizer, Trade Log, Correction Cycles)

---

## 2. Wireframe Specifications & Information Hierarchy

The core layout adopts the proven **3-Column Resizable Grid** from the `image_ac681b.png` reference: Left Panel (Navigation/Lists), Center Panel (Primary Content/Deep Dives), and Right Panel (Collapsible Charts/Context).

### Workspace 1: APOLLO MODE

#### View 1.1: Market Intel & Screener (Center Panel)

* **Top Action Bar:** Universe Funnel Summary (1.1.1), Toggle for Regular vs. IPO Stocks (1.7.3), CSV Export (1.1.10).
* **Left Filter Sidebar:** Multi-filter panel with sliders for composite score, PE, volume, and checkboxes for Bucket, Action, Gate status (1.1.8).
* **Main Data Table:** Mirrors the density of `image_ac5cd3.png`.
* *Columns:* Symbol, Score Bar (1.1.2), Action Badge (1.1.3), Structural Bucket (1.1.4), Renko Gate ✅/🚫 (1.1.5), G1-G4 Mini-icons (1.1.6). Expandable rows for IPO Signal Breakdown (1.7.2).



#### View 1.2: Stock Deep Dive (Select a stock from Intel)

* **Left Panel (340px):** Searchable list of stocks in current funnel.
* **Center Panel (Primary Focus):**
* *Header:* Symbol, Action Rationale Card (1.2.2), Structural Bucket Detail (1.2.3), Conviction Pyramid (1.3.1).
* *Top Row Widgets:* Score Breakdown Gauge (1.2.1), Fundamental Overlay Card (1.2.5), Overfit Risk Meter (1.6.5).
* *Middle Row Cards:* Gate Detail Panel with proximity bars (1.2.6), Renko Brick pattern panel (1.2.7).
* *Bottom Row:* Post-Trade Behavior Summary and Guidance Recommendations text block (1.8.1, 1.8.4).


* **Right Panel (Collapsible, 660px):**
* 52-Week Range Visualizer (1.2.8).
* Technical Indicator Dashboard (1.2.4).
* Price Chart with Indicator Overlays (1.2.9).



#### View 1.3: Trade Engine & Validation

* **Center Panel (Split View):**
* *Top Half (Equity Curve):* Line chart mapping cumulative PnL, drawdowns, and Train/Test splits (1.4.6, 1.6.1).
* *Bottom Half (Trade Log):* Compact data table of historical trades (1.4.1), Per-Symbol Trade History (1.4.7), Re-entry Analytics (1.4.10).


* **Right Panel:**
* Aggregate PnL Dashboard (1.4.2).
* MAE/MFE Diagnostics Distribution Histogram + Natural SL Discovery slider (1.4.5, 1.5.3).
* Exit Mode / Reason breakdown pie charts (1.4.3, 1.4.4).



---

### Workspace 2: LAYERSIGNAL MODE

#### View 2.1: Watchlist & Active Management

* **Left Panel (Watchlist - 340px):** Mirrors the left column of `image_ac681b.png`.
* Symbol, Sparklines (2.1.1), Auto-refresh indicator (2.1.7).
* Entry (L1-L3) and Exit (EL1-EL3) visual state badges in a compact horizontal bar (2.1.2, 2.1.3).


* **Center Panel (Deep Dive):**
* *Top Grid:* Entry Layer Progress Tracker (2.3.1) showing multi-timeframe progression (Weekly -> 3-Day -> Daily). Throwback Detection badge (2.3.5).
* *Middle Grid:* Exit Layer Status Dashboard (2.4.1). Three large cards for EL1, EL2, EL3 status and probabilities (2.4.4).
* *Bottom Grid:* Position Sizer & Trade Card (2.6.1). R:R Gauge, risk inputs (2.6.2, 2.6.3).


* **Right Panel (Collapsible Charts):**
* RSI Triple Stack Chart (2.2.1) + Divergence markers (2.2.4).
* L1/L2/L3 Chart views synced to different timeframes (2.3.2, 2.3.3, 2.3.4).



#### View 2.2: Market Scanner & Command Center

* **Left Panel (Scanner Filters):** Checkboxes for Stage Filters (L3 Active, L2 Ready, TB Active) (2.5.2).
* **Center Panel (Scanner Results):** High-density table matching `image_ac5cd3.png`.
* Includes Momentum Score (2.1.4), Volume vs Avg (2.5.1), and Scan Progress bar (2.5.5).


* **Right Panel (Command / Live Exec):** Kite Broker Live Positions table (3.4.1), Quick Order Placement (3.4.2).

---

## 3. Architecture Annotation Mapping

| UI Element / Zone | Mapped Architecture Requirement | Description |
| --- | --- | --- |
| **Global Footer** | 3.1.1, 3.1.2, 3.1.3, 3.1.4 | Live index ticker, VIX regime card, and Nifty moving average state. |
| **Global Top Bar** | 1.9.1, 1.9.3, 3.2.1, 3.2.3 | Engine version/computation status, Unified alert feed drop-down and badge count. |
| **Apollo Center Grid** | 1.2.1, 1.2.5, 1.3.1, 1.3.3 | Apollo composite score gauge, FQS score overlay, Conviction pyramid, Momentum regime badge. |
| **Apollo Right Panel** | 1.5.3, 1.4.5, 1.2.4, 1.2.9 | Natural SL distribution, MAE/MFE histograms, multi-timeframe technical indicators, base price chart. |
| **LayerSignal Left Nav** | 2.1.1, 2.1.2, 2.1.3 | Live watchlist with sparklines, L-stage entry badges, EL-status exit badges. |
| **LayerSignal Center** | 2.3.1, 2.3.6, 2.4.1, 2.6.1 | Entry/Exit progress pipelines, historical L3 recovery tables, dynamic trade position sizer. |
| **LayerSignal Right** | 2.2.1, 2.2.2, 2.2.5, 2.4.3 | Shared RSI 21/36/56 stack chart, multi-timeframe RSI tabs, Exit Layer chart overlays (Green MA/RSI lines). |
| **Analytics Modals** | 1.5.1, 1.5.2, 1.6.2, 2.7.2 | Drill-down charts for Conviction/Regime alpha queries, Alpha Degradation bar charts, Recovery profile charts. |

---

## 4. UI/UX Design Principles & Feature Density Rules

To match and exceed the visual standard set by the LayerSignal reference images while incorporating the massive SQLite backend dataset, apply the following principles:

### A. Information Density & Layout

* **Typography Scale:** Utilize a rigid type scale (e.g., `10px` for metadata, `11px` for table data, `13px` for primary labels, `16px-24px` for key metrics). This allows for extreme density without overlapping.
* **Micro-Visualizations:** Embed data directly into tables and lists. Use CSS-based horizontal bars for scores (e.g., Composite Score 0-148), sparklines for 30-day trends, and stacked horizontal bars for L1-L4 progression directly in the Watchlist row.
* **Card Anatomy:** Use a dark base (`--bg1: #1E1E24`) with lighter borders (`rgba(255,255,255,0.1)`). Group related data (e.g., Gate 1-4 passes) into singular, tightly padded flexbox containers rather than spreading them across the screen.

### B. Progressive Disclosure

* **Hover States:** The composite score in the Market Intel view (1.1.2) must display a color-gradient cell, but hover interactions should reveal the Pool A-E sub-score breakdown in a tooltip to prevent UI clutter.
* **Expandable Rows:** In the Scanner (2.5) and Trade Log (1.4.1) tables, clicking a row should expand it inline to reveal the "Entry State Snapshot" (1.4.8) or "Convergence Signals Detail" (1.3.2) rather than forcing a page navigation.
* **Collapsible Right Panel:** The 660px right panel containing heavy graphical elements (RSI Stacks, Candlestick charts) can be toggled via a narrow `16px` collapse strip to instantly reclaim screen real estate for the data tables.

### C. Visual Weight & Color Coding (Accessibility & Affordance)

* **Consistent Semantic Color:**
* *Green (`#19C985`):* Bullish, Pass, L3 Active, Target Hit.
* *Red (`#FF5F57`):* Bearish, Fail, EL3 Exit, Stop Loss Hit.
* *Amber (`#F5A623`):* Watch, Building, Throwback (TB) Active.
* *Blue (`#4DA8F7`):* L2 Active, Information/Selected state.


* **Contrast & Focus:** Ensure text contrast against `#16161A` meets WCAG 2.1 AA. Muted text (`--tx3: #6E6E88`) is used for non-essential units (e.g., "sessions", "%"), prioritizing the data values in stark white (`--tx0: #E8E8F0`).

### D. Shared SQLite Data Handling

* **Single Source of Truth:** Because both Apollo and LayerSignal read from the unified SQLite database, ensure that navigating from an Apollo Backtest Trade (Tab 1.4) to a LayerSignal Chart (Tab 2.2) maintains the same selected `symbol` context. The UI must transition seamlessly without re-fetching data, simply changing the "lens" (historical vs. live) applied to the `indicator_snapshots` and `price_ohlcv` tables.