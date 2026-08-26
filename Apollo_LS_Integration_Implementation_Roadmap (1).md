# Apollo + LayerSignal Integration & Implementation Roadmap

**Version:** 2.0  
**Date:** 2026-08-26  
**Scope:** End-to-end integration of Python Apollo+LayerSignal engines with React dashboard, building 7 missing engines, Kite Connect live data pipeline, and deployment  
**Execution Sequence:** Build Engines → Integrate React → Connect Live Data → Test  
**Estimated Timeline:** 6–8 weeks  

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Architecture Overview](#2-architecture-overview)
3. [Current State Assessment](#3-current-state-assessment)
4. [Data Contract — The Immutable Bridge](#4-data-contract--the-immutable-bridge)
5. [Endpoint Mapping — Express to FastAPI](#5-endpoint-mapping--express-to-fastapi)
6. [Phase 0: Environment & Path Fixes](#6-phase-0-environment--path-fixes)
7. [Phase 1: signals_service.py (P0)](#7-phase-1-build-signals_servicepy-p0--critical)
8. [Phase 2: highs_service.py (P1)](#8-phase-2-build-highs_servicepy-p1)
9. [Phase 3: ml_service.py (P1)](#9-phase-3-build-ml_servicepy-p1)
10. [Phase 4: ipo_service.py (P2)](#10-phase-4-build-ipo_servicepy-p2)
11. [Phase 5: alerts_service.py (P2)](#11-phase-5-build-alerts_servicepy-p2)
12. [Phase 6: journal + analytics services (P3)](#12-phase-6-build-journal--analytics-services-p3)
13. [Phase 7: React Frontend Migration](#13-phase-7-react-frontend-migration)
14. [Phase 8: Kite Connect Pipeline](#14-phase-8-kite-connect-live-data-pipeline)
15. [Phase 9: WebSocket Layer](#15-phase-9-websocket-real-time-layer)
16. [Phase 10: Persistence Migration](#16-phase-10-persistence-migration)
17. [Phase 11: Testing & Deployment](#17-phase-11-testing--deployment)
18. [React Kill List](#18-react-kill-list--files-to-delete)
19. [Python File Cleanup](#19-python-file-cleanup)
20. [Risk Register](#20-risk-register)
21. [Timeline](#21-timeline)

---

## 1. Executive Summary

### The Problem

The React dashboard runs on **three parallel scoring systems** that produce different results for the same stock:

| System | Location | Lines | Accuracy |
|--------|----------|-------|----------|
| Python Apollo Engine | `apollo_core/scoring.py` | ~800 | **Ground truth** — 38 signals, multi-timeframe, vectorized NumPy |
| TypeScript Express | `server.ts` + `highs-service.ts` + `ml-service.ts` | ~2,700 | **Approximation** — re-implements scoring in TS, fake ML |
| React Client Fallback | `src/utils/enrichment.ts` | ~294 | **Fake** — hashCode()-based random numbers |

This "triple enrichment divergence" means the user sees different scores depending on which code path executes.

### The Solution

**Python FastAPI becomes the sole compute backend.** React becomes a pure frontend consumer. The Express server is deleted entirely.

```
[Current — Triple Enrichment]
Google Sheet → TS server (re-scores, fake ML) → React UI
                                              ↓ fallback
                                        React enrichment.ts (random numbers)

[Target — Python Single Source of Truth]
Google Sheet / Kite Connect → Parquet Files → Python Apollo+LS Engine
                                                    ↓
                                              FastAPI JSON (47 endpoints)
                                                    ↓
                                            React UI (pure consumer)
```

### What Gets Built

- 7 new Python service engines
- 22 new FastAPI endpoints
- 1 WebSocket server with 4 message types
- Kite Connect live data pipeline (3 components)
- Replacement of ~2,700 lines of TypeScript server code
- Migration of 11 SQLite tables from sql.js (WASM) to Python aiosqlite

### What Gets Deleted

- ~5,000 lines of TypeScript server code (Express, fake ML, fake enrichment, fake highs)
- **Zero** React components are modified (except `App.tsx` API base URL change)

---

## 2. Architecture Overview

### Target System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     DATA SOURCES                                │
│  Google Sheet CSV (Phase 1-7, interim)                           │
│  Kite Connect API (Phase 8+, final)                              │
└──────────────┬──────────────────────────┬────────────────────────┘
               │                          │
               ▼                          ▼
┌─────────────────────────────────────────────────────────────────┐
│                  APOLLO DATA REPOSITORY                          │
│  {SYMBOL}.parquet │ {SYMBOL}_4H.parquet │ {SYMBOL}_1W.parquet  │
│  bucket_cache/ │ profiling.db │ live_state.db                   │
└──────────────┬──────────────────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────────────────┐
│                   PYTHON FASTAPI BACKEND                         │
│                                                                  │
│  EXISTING ENGINES                    NEW ENGINES (to build)     │
│  ┌─────────────────────┐             ┌────────────────────┐     │
│  │ apollo_core/         │             │ signals_service.py │     │
│  │  scoring.py (38 sig) │             │ highs_service.py   │     │
│  │  indicators.py (17)  │             │ ml_service.py      │     │
│  │  gates.py (4 gates)  │             │ ipo_service.py     │     │
│  │  bucket_classifier   │             │ alerts_service.py  │     │
│  │  renko.py (R1-R8)    │             │ journal_service.py │     │
│  └──────────┬──────────┘             │ analytics_service.py│     │
│             │                         └──────────┬─────────┘     │
│  ┌──────────┴──────────┐                        │               │
│  │ layersignal_engine/  │                        │               │
│  │  core.py (L1-L3)     │                        │               │
│  └──────────┬──────────┘                        │               │
│             └──────────────┬─────────────────────┘               │
│                            ▼                                     │
│  ┌────────────────────────────────────────────────────────┐     │
│  │            FastAPI Routes (47 endpoints total)          │     │
│  │  25 existing + 22 new                                   │     │
│  └────────────────────────┬───────────────────────────────┘     │
│                           │                                      │
│  ┌────────────────────────┴───────────────────────────────┐     │
│  │            WebSocket Server (/ws)                       │     │
│  │  LIVE_PULSE │ NEW_ALERT │ CACHE_REFRESH │ STATE_DIFF   │     │
│  └────────────────────────┬───────────────────────────────┘     │
└───────────────────────────┼──────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                    REACT FRONTEND (Vite + React 19)              │
│  10 Tabs: Screener │ IPO │ New Highs │ ML │ Watchlist │ Scanner  │
│           Analytics │ Alerts │ Guidance │ System                   │
│  Pure JSON consumer — zero scoring logic on client              │
└─────────────────────────────────────────────────────────────────┘
```

### Data Flow — /api/signals Request

```
React: fetch('/api/signals')
  │
  ▼
FastAPI: signals_service.get_signals()
  │
  ├─ _fetch_data_source()  → Google Sheet CSV (interim) or Parquet (final)
  │   Controlled by: DATA_SOURCE=gsheet|parquet|kite_live (.env flag)
  │
  ├─ For each of ~1,400 stocks:
  │   ├─ Load {SYMBOL}.parquet from APOLLO_DATA_REPOSITORY
  │   ├─ indicators.compute_all(df) → 17 indicators
  │   ├─ scoring.score_stock(df, indicators) → Apollo_Score (0-148), action, SubScores
  │   ├─ layersignal_engine.core.calc_momentum_score(df) → LS_Score (20-100), ELStatus
  │   ├─ gates.evaluate(df, indicators) → Gates[5 bool], GatesExplanations[5 str]
  │   ├─ bucket_classifier.classify(df, indicators) → Bucket (L1-L4), LStage, FQS
  │   ├─ [if model exists] ml_service.predict(features) → ml_confidence (0.0-1.0)
  │   └─ Build sparkline (10-day closes), ThrowbackAlert, HistoricalL3Events
  │
  ├─ Compute SignalsSummary (funnel counts, averages, bucket/quality breakdowns)
  └─ Return JSON { stocks: SignalStock[49 fields][], summary: SignalsSummary }
```

### Key Design Decisions

1. **signals_service exposes a `SignalsService` class** with `get_signals()` method. Every other engine imports this class — no duplication.
2. **Data source abstraction**: `_fetch_gsheet()` (interim) vs `_load_from_parquet()` (final). A single `.env` flag (`DATA_SOURCE`) controls which path is used. Scoring logic never changes when switching data sources.
3. **SignalStock[49 fields] is the immutable contract.** Python JSON output must match the TypeScript interface exactly. React's `types.ts` will not need a single change.

---

## 3. Current State Assessment

### 3.1 React Frontend — What Exists

**Repository:** `https://github.com/AkhilB21/Google-AI-Sudio-Project`  
**Tech Stack:** Vite 6 + React 19 + TypeScript 5.8 + Tailwind CSS 4 + Recharts 3 + sql.js (WASM SQLite)  
**Entry Point:** `server.ts` runs Express on port 3000 with Vite middleware  
**State Management:** No React Router, no Redux — tabs via `activeTab` in `App.tsx`, all state via `useState`  

#### Frontend File Inventory

| File | Lines | Role | Post-Integration Action |
|------|-------|------|------------------------|
| `server.ts` | 1,284 | Express server, GSheet fetch, scoring, WebSocket, alert detection | **DELETE** |
| `server/db.ts` | ~300 | SQLite schema (11 tables), seeding, save/load | **MIGRATE** to Python |
| `server/highs-service.ts` | 734 | 52WH enrichment — 12 functions | **DELETE** — Python `highs_service.py` |
| `server/highs-routes.ts` | ~350 | 52WH REST endpoints (5) | **DELETE** — Python routes |
| `server/ml-service.ts` | 651 | Fake ML — logistic sigmoid, 9 factors | **DELETE** — real XGBoost in Python |
| `server/ml-routes.ts` | ~200 | ML REST endpoints (4) | **DELETE** — Python routes |
| `server/ipo-routes.ts` | ~400 | IPO endpoints (8) + scraper + classifier + graduator | **DELETE** — Python `ipo_service.py` |
| `server/ipo-classifier.ts` | ~200 | 4-zone IPO classification logic | **PORT** logic to Python |
| `server/ipo-scraper.ts` | ~250 | Chittorgarh.com scraper + 22 seeded records | **PORT** to Python |
| `server/ipo-graduator.ts` | ~100 | IPO graduation/archival logic | **PORT** to Python |
| `src/utils/enrichment.ts` | 294 | Client-side fake scorer (hashCode random) | **DELETE** |
| `src/types.ts` | 496 | All TypeScript interfaces — SignalStock, HighsStock, etc. | **KEEP** (immutable contract) |
| `src/App.tsx` | ~400 | Root component, state, WebSocket, data fetching | **MODIFY** (API base URL only) |
| `src/utils/calculations.ts` | ~100 | Quality/Risk/INR formatting helpers | **KEEP** |
| `src/utils/notifications.ts` | ~100 | Browser push + Web Audio API chime | **KEEP** |
| `src/components/tabs/*.tsx` | ~67KB | All 10 tab components | **KEEP** (no component changes) |
| `src/components/highs/*.tsx` | 11 files | New Highs sub-components | **KEEP** |
| `src/components/ml/*.tsx` | 8 files | ML sub-components | **KEEP** |
| `src/components/shell/*.tsx` | 4 files | AppBar, TabBar, MarketRegimeBanner, TickerFooter | **KEEP** |

#### SQLite Database — 11 Tables (sql.js WASM)

| Table | Key Columns | Seeded Rows |
|-------|-------------|-------------|
| `trades` | id, symbol, entryDate, exitDate, entryPrice, exitPrice, pnlPct, holdingDays, exitMode | ~20 demo |
| `journal_entries` | id, symbol, date, note, target, stopLoss, createdAt | ~5 demo |
| `alert_rules` | id, name, condition, channel, enabled | ~4 demo |
| `alert_logs` | id, source, type, timestamp, symbol, title, message, read | ~50 demo |
| `system_logs` | id, time, level, message | ~10 demo |
| `ipo_stocks` | id, symbol, company_name, issue_price, listing_date, cmp, zone, baseline, sector (+17 fields) | 22 real |
| `ipo_zone_history` | id, symbol, old_zone, new_zone, transition_type, cmp_at_transition | ~30 demo |
| `ipo_archive` | Same as ipo_stocks + graduated_at | ~5 demo |
| `daily_highs_snapshot` | id, date, symbol, high_52w, cmp, proximity_pct, proximity_tier, apollo_score | ~500 demo |
| `stock_ath` | symbol, all_time_high, ath_date, ath_cmp_pct | ~100 demo |
| `ml_model_metadata` | model_name, train_auc, test_auc, feature_count, calibration_status | 1 row |
| `ml_predictions_history` | date, symbol, ml_confidence, confidence_tier, forward_20d_actual | ~200 demo |

---

### 3.2 Python Backend — What Exists

**Repository:** `https://github.com/AkhilB21/Native-Screener-Architecture`  
**Zip:** `APOLLO_BT.V1.zip` — 137 Python files  
**Framework:** FastAPI + uvicorn on `0.0.0.0:8000`  
**Engine Versions:** Apollo v4.13 (38 signals), LayerSignal v20.1-fixed, Bucket Classifier v5.0

#### Fully Implemented Core Modules

| Module | Key Files | Function | Used By |
|--------|-----------|----------|----------|
| **Apollo Scoring** | `scoring.py`, `constants.py` | 26 mature + 12 IPO signals, 5 pools (A-E), Renko R1-R8, divergence, score 0-148 | Backtest, signals_service |
| **Indicators** | `indicators.py` | RSI(21,36,56), TP28, WC(21,50), ADX, DI+/-, ATR, MACD, Stochastic, VPT, SMA(20,50), Renko bricks | All modules |
| **Gates** | `gates.py` | G1 PE, G2 Liquidity, G3 52W-Low proximity, G4 52W-High distance — **informational only** (never blocks) | Scoring |
| **Bucket Classifier** | `bucket_classifier.py` | 8-bucket classification (Trending T1-T4, Recovery, Sideways, Decline, IPO) → L1-L4 tier | Scoring |
| **LayerSignal** | `layersignal_engine/core.py` | 3-layer detection (L1 accumulation, L2 momentum, L3 trigger), momentum score 0-100, exit layers 0-100 | Signals, charts |
| **Renko** | `renko.py` | Brick construction, 8 Renko signals R1-R8 (20 pts max) | Scoring |
| **Trade Engine** | `trade_engine.py` | Full backtest loop, 5 exit modes, ~1,100 lines | Backtest |
| **Guidance Engine** | `guidance_engine/` (8 files) | Trade analytics, MAE/MFE, regime, flags, aggregation — **strongest sub-system** | Guidance tab |
| **Feature Store** | `precompute_features.py`, `historical_batch_replay.py`, `signal_profiler.py` | 20 pre-computed features, ~1,250 signal-outcome pairs from 5yr history | ML training |
| **Data Pipeline** | `bridge_daily_update.py`, `signal_export.py` | GSheet → parquet → engines → CSV, multi-process signal generation | Data pipeline |
| **Data Repo** | `data_repo/repo.py` | Parquet load/save/list operations | All data access |

#### Existing FastAPI Endpoints (25)

| # | Method | Path | Data Source | Dashboard Tab |
|---|--------|------|-------------|---------------|
| 1 | GET | `/api/healthz` | None | System |
| 2 | GET | `/api/screener/data` | apollo_database.db | Screener (fake scores) |
| 3 | GET | `/api/screener/search` | apollo_database.db | AppBar search |
| 4 | GET | `/api/live-metrics` | Google Sheet | — |
| 5 | GET | `/api/positions` | live_state.db | Watchlist |
| 6 | GET | `/api/alerts` | live_state.db | Alerts (diff schema) |
| 7 | GET | `/api/health` | Filesystem checks | System (partial) |
| 8 | GET | `/api/guidance/trades` | trades.parquet | Analytics |
| 9 | GET | `/api/guidance/profiles` | profiles/*.parquet | Guidance |
| 10 | GET | `/api/guidance/profile/{sym}` | profiles/{SYM}.parquet | Guidance |
| 11 | GET | `/api/guidance/flags` | aggregation parquet | Guidance |
| 12 | GET | `/api/guidance/mae-mfe` | mae_mfe parquet | Guidance |
| 13 | GET | `/api/guidance/aggregation` | aggregation parquet | Analytics |
| 14 | GET | `/api/audit/report` | audit_report.json | — |
| 15 | GET | `/api/audit/issues` | audit_report.json | — |
| 16 | GET | `/api/simulator/results` | simulator_results/ | — |
| 17 | GET | `/api/market-intel/panel` | panel.parquet | — |
| 18 | GET | `/api/universe` | 3 JSON files | — |
| 19 | POST | `/api/inbox/process` | subprocess | — |
| 20 | GET | `/api/inbox/latest` | inbox_analysis.json | — |
| 21 | GET | `/api/live_radar/multi-timeframe` | {SYM}_*.parquet | Watchlist charts |
| 22 | GET | `/api/analytics/profile` | profiling.db | Analytics |
| 23 | GET | `/api/analytics/deep_dive/{sym}` | profiling.db | Analytics |
| 24 | GET | `/api/engine/walk_forward` | CSV | — |
| 25 | GET | `/api/engine/layer_signal/{sym}` | {SYM}.parquet | Watchlist charts |

#### Stub Files

| File | Status | Detail |
|------|--------|--------|
| `kite_broker_engine.py` | **STUB** | `kite.place_order()` commented out, returns `sim_order_id` |
| `live_market_streamer.py` | **STUB** | `on_ticks()` has `pass`, falls back to `[SIMULATED]` |
| `vectorized_scoring.py` | **PARTIAL** | Only Pool A signals (A1, A2, A3) vectorized — WIP |
| `data_repo/sources.py` | **EMPTY** | Placeholder comment only |

#### Known Issues

| Issue | File | Detail |
|-------|------|--------|
| Fake DB scoring | `db_builder.py:63` | `base_score = 50 + (prox_52w_high * 0.4)` — NOT real Apollo score |
| SQL injection | `analytics.py:130` | f-string in SQL: `f"WHERE e.symbol = '{symbol}'"` |
| Export bug | `signal_export.py:46-50` | Return for FLAT/HOLD indented inside if block — `UnboundLocalError` |
| Missing deps | `requirements.txt` | Missing `fastapi`, `uvicorn`, `kiteconnect` |
| 52 hardcoded paths | 37 files | Windows paths bypass `APOLLO_DATA_ROOT` env var |

---

### 3.3 Tab-by-Tab Gap Analysis

#### Tabs WITH Python Engine (Partial or Full)

| Tab | Python Status | Gap |
|-----|---------------|-----|
| **Screener** | Core scoring exists — `scoring.py` (38 signals), `indicators.py` (17), `gates.py` (4), `bucket_classifier.py`, `layersignal_engine/core.py` | No `/api/signals` endpoint serving the 49-field SignalRow. Existing `/api/screener/data` reads `apollo_database.db` which has **fake scores**. |
| **IPO** | `ipo_lookup.py` (listing date lookup only) | No live scraper, no zone classifier, no baseline computation, no graduation logic, no sector data. |
| **Watchlist** | `live_engine/watchlist.py` (JSON loader) | No journal persistence API, no position sizer API. |
| **Analytics** | `backtest_engine/` + `guidance_engine/` (post-trade) | No real-time universe analytics endpoint (score distributions, action breakdowns). |
| **Guidance** | `guidance_engine/` (8 files, 6 endpoints) — **strongest** | "Projected win rate" and "divergence re-entry" in React are fake client-side math. |
| **System** | `/api/health` endpoint only | Missing score histogram, log viewer, cache management. |

#### Tabs with NO Python Engine (Fake / Client-Side)

| Tab | Current Reality | What's Missing |
|-----|---------------|---------------|
| **New Highs** | `server/highs-service.ts` (734 lines) — proximity grading, NH-NL breadth, exhaustion, momentum, consistency, base rates, sector clusters, EL confluence | Full `highs_service.py` with 12 functions, ~700+ lines |
| **ML Confidence** | `server/ml-service.ts` (651 lines) — hardcoded logistic sigmoid, 9 weighted factors, fake feature importance | Real XGBoost meta-learner with purged CV, permutation importance, model persistence |
| **Scanner** | Client-side: 12 preset patterns with threshold checks | Optional — could stay in React (just filters SignalStock[]) |
| **Alerts** | `server.ts` — `detectAndLogStatusChanges()` every 30s, Express rule CRUD, Web Audio | Python alert rule engine, status-change detection, WebSocket push, rule persistence |

#### Engine Dependency Graph

```
signals_service (P0) — Foundation. 6 out of 10 tabs depend on this.
├── New Highs (P1) — calls get_signals() as input, adds proximity/exhaustion/momentum layers
├── ML Confidence (P1) — receives SignalStock[] features, runs XGBoost predict_proba()
├── IPO (P2) — cross-matches IPO stocks against SignalStock[] for Apollo/LS scores
├── Alerts (P2) — compares current SignalStock[] vs previous snapshot for state transitions
├── Scanner (client) — filters SignalStock[] against 12 presets (stays in React)
├── Analytics (client) — aggregates SignalStock[] via useMemo (stays in React)
└── WebSocket — broadcasts diff snapshots of SignalStock[] to React clients
```

---

## 4. Data Contract — The Immutable Bridge

The TypeScript interfaces in `src/types.ts` are the **immutable contract**. Python JSON output must match these exactly. React components will not change.

### 4.1 SignalStock — 49 Fields (Universal Contract)

This is the central data type. 6 out of 10 tabs consume it.

| # | Field | Type | Python Source | Notes |
|---|-------|------|---------------|-------|
| 1 | `Symbol` | string | GSheet CSV `Ticker` column | |
| 2 | `Date` | string | Today's date (ISO) | |
| 3 | `Apollo_Action` | 'HOLD'\|'ENTRY'\|'EXIT'\|'FLAT' | `scoring.classify_score(score)` | |
| 4 | `Apollo_Score` | number (0-148) | `scoring.score_stock(df, indicators)` | 26 mature + 12 IPO signals |
| 5 | `Pct_Change` | number | GSheet CSV or `(Close-prev)/prev` | |
| 6 | `LayerSignal_Action` | 'HOLD'\|'ENTRY'\|'EXIT'\|'FLAT' | `layersignal_engine.core.evaluate_trade_signals()` | |
| 7 | `LayerSignal_Score` | number (20-100) | `layersignal_engine.core.calc_momentum_score(df)` | |
| 8 | `Exit_Pressure` | number (5-95) | `layersignal_engine.core.compute_exit_layers(df)` | |
| 9-13 | OHLCV | number | GSheet CSV or last parquet row | |
| 14 | `High52W` | number | `df['High'].rolling(252).max()` | |
| 15 | `Low52W` | number | `df['Low'].rolling(252).min()` | |
| 16-18 | `RSI21/36/56` | number | `indicators.compute_rsi(df, period)` | |
| 19 | `ADX` | number | `indicators.compute_adx(df)` | |
| 20 | `ATR_Pct` | number | `indicators.compute_atr(df) / Close * 100` | |
| 21-23 | `20D/50D/200D_SMA` | number? | `indicators.compute_sma(df, period)` | Null if insufficient bars |
| 24 | `PE` | number | GSheet CSV or `fundamental_repo.py` | |
| 25 | `Stochastic` | number | `indicators.compute_stochastic(df)` | |
| 26 | `52W_Prox` | number | `(Close / High52W) * 100` | Glue: 5 lines |
| 27 | `CMP` | number | GSheet CSV `Close` | |
| 28 | `Traded_Value` | number | GSheet CSV `Traded Value` | |
| 29 | `Bucket` | 'L1'\|'L2'\|'L3'\|'L4' | `bucket_classifier.classify()` → map 8-bucket to L1-L4 | |
| 30 | `LStage` | string | From bucket_classifier stage description | |
| 31 | `ELStatus` | 'EL1'\|'EL2'\|'EL3'\|'EL4'\|'NONE' | `layersignal_engine.core` entry layer | |
| 32 | `Conviction` | number (0-1) | `score / 148` normalized | Glue: 1 line |
| 33 | `Gates` | [bool x5] | `gates.evaluate()` + G5 Quality (RSI stack) | Glue: Python has 4 gates, add G5 |
| 34 | `Renko` | 'GREEN'\|'RED'\|'NEUTRAL' | `renko.get_renko_trend(df)` | |
| 35 | `MCap` | 'Large'\|'Mid'\|'Small' | Classify from CMP * Volume thresholds | Glue: ~10 lines |
| 36 | `FQS` | 'A'\|'B'\|'C'\|'D' | Score>=110→A, >=85→B, >=60→C, else D | Glue: ~5 lines |
| 37 | `Sparkline` | number[10] | Last 10 closes from parquet | |
| 38 | `ThrowbackAlert` | boolean? | CMP < 20SMA, 20SMA >= 50SMA, score >= 55 | Glue: ~10 lines |
| 39 | `ml_confidence` | number? | `ml_service.predict()` if model exists | Null if no model |
| 40 | `SubScores` | object? | {trend, momentum, volatility, volume, marketFilter} | Glue: from pool scores ~20 lines |
| 41 | `GatesExplanations` | [string x5]? | Human-readable pass/fail per gate | Glue: ~15 lines |
| 42 | `HistoricalL3Events` | Array? | 3 events from parquet L3 triggers + outcome | Glue: ~30 lines |
| 43 | `isIPO` | boolean? | Cross-reference `ipo_listing_dates.json` | |
| 44 | `ipoData` | IPOStock? | Full IPO object if isIPO | From ipo_service |
---
### 4.2 HighsStock — 45+ Fields

```typescript
// Reference: src/types.ts lines 136-188
interface HighsStock {
  symbol: string;
  name: string;
  CMP: number;
  High52W: number;
  Low52W: number;
  proximity_pct: number;
  distance_pct: number;
  proximity_tier: 'AT' | 'NEAR' | 'APPROACHING' | 'EXTENDED' | 'FAR';
  Apollo_Score: number;
  gate_count: number;
  Gates: [boolean, boolean, boolean, boolean, boolean];
  Quality: 'STRONG' | 'GOOD' | 'MODERATE' | 'WEAK';
  FQS: 'A' | 'B' | 'C' | 'D';
  ELStatus: 'EL1' | 'EL2' | 'EL3' | 'EL4' | 'NONE';
  Sector: string;
  Bucket: 'L1' | 'L2' | 'L3' | 'L4';
  isIPO?: boolean;
  ipoData?: IPOStock;
  // P1 fields
  RSI21: number; RSI36: number; RSI56: number;
  Exit_Pressure: number;
  ADX: number;
  exhaustion_flags: string[];
  exhaustion_severity: 'NONE' | 'MILD' | 'MODERATE' | 'STRONG';
  return_5d: number | null;
  return_20d: number | null;
  gap_up_count_10d: number;
  momentum_profile: 'GRINDING' | 'CONSOLIDATING' | 'MODERATE' | 'ACCELERATING' | 'SPIKE';
  // P2 fields
  high_type: 'FRESH' | 'RECLAIMED' | 'EARLY_REPEAT' | 'REPEAT' | 'EXTENDED_STAY';
  days_near_high: number;
  all_time_high: number;
  ath_cmp_pct: number;
  days_since_ath: number;
  is_at_ath: boolean;
  ipo_confluence: boolean;
  is_watchlist_match?: boolean;
  // P3 fields
  consistency: {
    high_close_count: number; total_days: number; consistency_pct: number;
    max_dd_from_high: number;
    pattern: 'LAUNCHPAD' | 'BUILDING' | 'CHOPPY' | 'ONE_TOUCH';
  };
  base_rates: {
    sample_size: number; status: 'READY' | 'INSUFFICIENT_DATA';
    win_rate: number; avg_return: number; median_return: number;
    best_case: number; worst_case: number;
  };
  el_confluence_score: number;
  is_dual_signal: boolean;
  Sparkline?: number[];
  Pct_Change?: number;
  Traded_Value?: number;
}
```

**Python source for each field:**

| Field Group | Python Source |
|-------------|---------------|
| Base stock data | Inherited from SignalStock via `signals_service.get_signals()` |
| proximity_pct, distance_pct | `highs_service.compute_proximity_tier()` — `(CMP/High52W)*100` |
| proximity_tier | `compute_proximity_tier()` — AT(>=99%), NEAR(95-99%), APPROACHING(90-95%), EXTENDED(>102%) |
| exhaustion_flags | `highs_service.detect_rsi_divergence()` — 4 checks if proximity>=95% |
| exhaustion_severity | `detect_rsi_divergence()` — NONE/MILD(1-2 flags)/MODERATE(2+flags,not INVSTACK)/STRONG(3+ or INVSTACK) |
| momentum_profile | `highs_service.compute_rate_of_approach()` — SPIKE/ACCELERATING/GRINDING/CONSOLIDATING |
| return_5d, return_20d | `compute_rate_of_approach()` — from sparkline history |
| gap_up_count_10d | `compute_rate_of_approach()` — count gap-up days in last 10 |
| high_type | `highs_service.classify_fresh_vs_repeat()` — FRESH/RECLAIMED/EXTENDED_STAY/REPEAT/EARLY_REPEAT |
| consistency | `highs_service.compute_consistency_score()` — LAUNCHPAD/BUILDING/CHOPPY/ONE_TOUCH |
| base_rates | `highs_service.compute_base_rates()` — deterministic placeholder (empirical later) |
| el_confluence_score, is_dual_signal | `highs_service.compute_el_confluence()` — 15-cell lookup matrix (tier x EL) |

### 4.3 MLStockPrediction — 40+ Fields

```typescript
// Reference: src/types.ts lines 407-451
interface MLStockPrediction {
  // All SignalStock base fields (Symbol, CMP, Pct_Change, Apollo_Score, LS_Score, etc.)
  // + Model outputs:
  ml_confidence: number;       // 0-1 (P(5%+ gain in 20d))
  ml_confidence_pct: number;  // 0-100
  ml_confidence_10d: number;   // 0-1 (P(3%+ in 10d))
  ml_confidence_5d: number;    // 0-1 (P(2%+ in 5d))
  ml_expected_20d_return: number;  // continuous expected %
  confidence_tier: 'HIGH' | 'MODERATE' | 'NEUTRAL' | 'LOW';
  // Confluence flags
  is_triple_confluence: boolean;  // Apollo>=80 + EL1/2 + ML>=70%
  is_high_conviction: boolean;   // ML>=70%
  is_contrarian: boolean;       // ML>=65% AND Apollo<80
  is_dual_engine: boolean;      // Apollo>=80 AND ELStatus!='NONE'
  // Explainability
  contributions: Array<{feature, label, value, impact, category}>;
  trade_sizing: {
    confidence_pct, tier, recommended_allocation_pct,
    risk_reward_ratio, max_risk_per_trade_pct,
    suggested_stop_loss_pct, action_summary
  };
}
```

**Python source:**

| Field Group | Python Source |
|-------------|---------------|
| Base fields | Inherited from SignalStock |
| ml_confidence (3 horizons) | `ml_service.predict(features, horizon)` — XGBoost predict_proba() |
| confidence_tier | HIGH(>=70%), MODERATE(55-69%), NEUTRAL(45-54%), LOW(<45%) |
| Confluence flags | Derived from Apollo_Score + ELStatus + ml_confidence thresholds |
| contributions | `ml_service.explain(features)` — SHAP values or permutation importance deltas |
| trade_sizing | `ml_service.compute_sizing(confidence_tier)` — allocation/R:R/stop-loss by tier |

### 4.4 Supporting Types

| Type | Key Fields | Used By |
|------|-----------|----------|
| `SignalsSummary` | total, liquid, scored, signalBearing, ENTRY/HOLD/EXIT/FLAT counts, avgScore, avgExitPressure, buckets{L1-L4}, quality{STRONG/GOOD/MODERATE/WEAK}, generatedAt | ScreenerTab header |
| `IPOStock` | 28 fields: symbol, company_name, issue_price, listing_date, listing_price, all_time_high, all_time_low, ipo_size, sector, exchange, promoter_stake, current_pe, cmp, zone, ipo_baseline, listing_stage, days_since_listing, distance_to_baseline_pct, distance_to_ath_pct, return_from_issue_pct, baseline_ratio, ath_recovery_pct, Apollo_Score?, LayerSignal_Score?, Apollo_Action?, Quality?, Bucket?, Gates?, RSI21?, ADX?, ATR_Pct? | IPOTab |
| `IPOSummary` | total, zones{new_high, recovery, under_pressure, broken_ipo}, stages{fresh, mature}, avg_return_from_issue, best/worst_performer | IPOTab header |
| `AlertItem` | id, source(Apollo|LayerSignal|System|IPO), type(Entry|Exit|Regime|Scoring|System), timestamp, symbol?, title, message, read | AlertsTab |
| `TradeRecord` | id, symbol, entryDate, exitDate, entryPrice, exitPrice, pnlPct, holdingDays, exitMode(Target Hit|Stop Loss|Time Exit|Signal Exit) | WatchlistTab |
| `SystemHealthData` | apolloScanTime, apolloDuration, apolloProcessed, apolloStatus, layerScanTime, layerDuration, layerPatterns, layerStatus, dbSizeMB, dbTables, lastDbUpdate, apiEndpoints[] | SystemTab |
| `MLModelInfo` | status(LOADED|TRAINING|MODEL_NOT_LOADED), model_name, model_type, target, train_auc, test_auc, avg_precision, train_samples, test_samples, purged_rows, embargo_days, feature_count, trained_at, calibration_status | MLConfidenceTab |
| `HighsBreadthData` | total_stocks, new_highs, new_lows, nh_pct, nl_pct, net_hl, regime(STRONG_BULLISH|MODERATE_BULLISH|NEUTRAL|MODERATE_BEARISH|STRONG_BEARISH) | NewHighsTab |
| `HighsTierSummary` | AT{count, delta}, NEAR{count, delta}, APPROACHING{count, delta}, EXTENDED{count, delta} | NewHighsTab |
| `SectorCluster` | sector, stock_count, at_count, near_count, avg_proximity_pct, avg_apollo_score, top_stocks[], strength | NewHighsTab |
| `HighsApiResponse` | breadth, tier_summary, clusters[], watchlist_alerts[], total, page, limit, data[] | NewHighsTab |

---
## 5. Endpoint Mapping — Express to FastAPI

### 5.1 Existing Express Endpoints (29)

| # | Method | Path | Source File | Dashboard Tab |
|---|--------|------|-----------|-------------|
| 1 | GET | `/api/health` | server.ts:851 | System |
| 2 | GET | `/api/signals` | server.ts:856 | Screener (via App.tsx) |
| 3 | POST | `/api/signals/sync` | server.ts:867 | System (cache clear) |
| 4 | GET | `/api/db/trades` | server.ts:926 | Watchlist |
| 5 | POST | `/api/db/trades` | server.ts:943 | Watchlist |
| 6 | GET | `/api/db/journal` | server.ts:957 | Watchlist |
| 7 | POST | `/api/db/journal` | server.ts:974 | Watchlist |
| 8 | GET | `/api/db/alerts` | server.ts:989 | Alerts |
| 9 | POST | `/api/db/alerts/test` | server.ts:1008 | Alerts |
| 10 | POST | `/api/db/alerts/simulate-status-change` | server.ts:1044 | Alerts |
| 11 | POST | `/api/db/alerts/mark-read` | server.ts:1082 | Alerts |
| 12 | GET | `/api/db/rules` | server.ts:1100 | Alerts |
| 13 | POST | `/api/db/rules` | server.ts:1118 | Alerts |
| 14 | PATCH | `/api/db/rules/:id` | server.ts:1131 | Alerts |
| 15 | DELETE | `/api/db/rules/:id` | server.ts:1145 | Alerts |
| 16 | GET | `/api/system/health` | server.ts:1240 | System |
| 17 | GET | `/api/ipo/stocks` | ipo-routes.ts:61 | IPO |
| 18 | GET | `/api/ipo/stocks/:symbol` | ipo-routes.ts:134 | IPO |
| 19 | GET | `/api/ipo/summary` | ipo-routes.ts:184 | IPO |
| 20 | GET | `/api/ipo/zone-history` | ipo-routes.ts:260 | IPO |
| 21 | GET | `/api/ipo/zone-history/:symbol` | ipo-routes.ts:278 | IPO |
| 22 | POST | `/api/ipo/refresh` | ipo-routes.ts:299 | IPO |
| 23 | POST | `/api/ipo/graduate/:symbol` | ipo-routes.ts:331 | IPO |
| 24 | GET | `/api/ipo/archive` | ipo-routes.ts:353 | IPO |
| 25 | GET | `/api/highs/breadth` | highs-routes.ts:43 | New Highs |
| 26 | GET | `/api/highs/sectors` | highs-routes.ts:59 | New Highs |
| 27 | GET | `/api/highs/watchlist-alerts` | highs-routes.ts:75 | New Highs |
| 28 | GET | `/api/highs/stocks` | highs-routes.ts:114 | New Highs |
| 29 | POST | `/api/highs/snapshot` | highs-routes.ts:293 | New Highs |
| 30 | GET | `/api/ml/info` | ml-routes.ts:29 | ML Confidence |
| 31 | GET | `/api/ml/importance` | ml-routes.ts:43 | ML Confidence |
| 32 | GET | `/api/ml/stocks` | ml-routes.ts:57 | ML Confidence |
| 33 | POST | `/api/ml/retrain` | ml-routes.ts:174 | ML Confidence |

### 5.2 Existing Python FastAPI Endpoints (25)

See Section 3.2 — 25 endpoints already working.

### 5.3 New Python Endpoints to Build (22)

| # | Method | Path | Service | Dashboard Tab | Priority |
|---|--------|------|---------|---------------|----------|
| 1 | GET | `/api/signals` | signals_service | Screener (+ 5 tabs via props) | P0 |
| 2 | GET | `/api/highs/stocks` | highs_service | New Highs | P1 |
| 3 | GET | `/api/highs/breadth` | highs_service | New Highs | P1 |
| 4 | GET | `/api/highs/sectors` | highs_service | New Highs | P1 |
| 5 | GET | `/api/highs/watchlist-alerts` | highs_service | New Highs | P1 |
| 6 | POST | `/api/highs/snapshot` | highs_service | New Highs | P1 |
| 7 | GET | `/api/ml/stocks` | ml_service | ML Confidence | P1 |
| 8 | GET | `/api/ml/info` | ml_service | ML Confidence | P1 |
| 9 | GET | `/api/ml/importance` | ml_service | ML Confidence | P1 |
| 10 | POST | `/api/ml/retrain` | ml_service | ML Confidence | P1 |
| 11 | GET | `/api/ipo/stocks` | ipo_service | IPO | P2 |
| 12 | GET | `/api/ipo/stocks/{symbol}` | ipo_service | IPO | P2 |
| 13 | GET | `/api/ipo/summary` | ipo_service | IPO | P2 |
| 14 | GET | `/api/ipo/zone-history` | ipo_service | IPO | P2 |
| 15 | GET | `/api/ipo/zone-history/{symbol}` | ipo_service | IPO | P2 |
| 16 | POST | `/api/ipo/refresh` | ipo_service | IPO | P2 |
| 17 | POST | `/api/ipo/graduate/{symbol}` | ipo_service | IPO | P2 |
| 18 | GET | `/api/ipo/archive` | ipo_service | IPO | P2 |
| 19 | GET/POST/DELETE | `/api/db/trades` | journal_service | Watchlist | P3 |
| 20 | GET/POST | `/api/db/journal` | journal_service | Watchlist | P3 |
| 21 | GET/POST/DELETE | `/api/db/alerts` + `/api/db/rules` | alerts_service | Alerts | P2 |
| 22 | GET | `/api/system/health` | analytics_service | System | P3 |

### 5.4 Complete Mapping Table

| Express Endpoint | Python Replacement | Match Status |
|----------------|-------------------|--------------|
| GET /api/signals | GET /api/signals (NEW) | **NEW — Critical gap** |
| GET /api/health | GET /api/health (existing) | Close match |
| POST /api/signals/sync | POST /api/signals/sync (NEW) | NEW |
| GET/POST /api/db/trades | GET/POST /api/db/trades (NEW) | NEW |
| GET/POST /api/db/journal | GET/POST /api/db/journal (NEW) | NEW |
| GET /api/db/alerts | GET /api/alerts (existing, schema mismatch) | Schema fix needed |
| POST /api/db/alerts/test | POST /api/db/alerts/test (NEW) | NEW |
| POST /api/db/alerts/simulate | POST /api/db/alerts/simulate (NEW) | NEW |
| POST /api/db/alerts/mark-read | POST /api/db/alerts/mark-read (NEW) | NEW |
| GET/POST /api/db/rules | GET/POST /api/db/rules (NEW) | NEW |
| PATCH/DELETE /api/db/rules/:id | PATCH/DELETE /api/db/rules/{id} (NEW) | NEW |
| GET /api/system/health | GET /api/health (existing, partial) | Enhancement needed |
| GET/POST /api/ipo/* | GET/POST /api/ipo/* (NEW, 8 endpoints) | NEW — full replacement |
| GET/POST /api/highs/* | GET/POST /api/highs/* (NEW, 5 endpoints) | NEW — full replacement |
| GET/POST /api/ml/* | GET/POST /api/ml/* (NEW, 4 endpoints) | NEW — full replacement |

---
## 6. Phase 0: Environment & Path Fixes

**Duration:** 1 day  **Depends on:** Nothing  **Blocks:** All subsequent phases

This phase is a prerequisite. Every subsequent phase assumes a clean, runnable Python environment with correct paths.

### Step 0.1: Fix 52 Hardcoded Windows Paths

**Problem:** 37 Python files contain literal `C:\Users\Akhil\...` paths that bypass the `APOLLO_DATA_ROOT` environment variable. These will crash on any Linux/Mac/server deployment.

**Files affected (37):**

| # | File | Approx Hardcoded Paths |
|---|------|----------------------|
| 1 | `apollo_core/scoring.py` | 3 |
| 2 | `apollo_core/indicators.py` | 2 |
| 3 | `apollo_core/gates.py` | 1 |
| 4 | `apollo_core/bucket_classifier.py` | 2 |
| 5 | `apollo_core/precompute_features.py` | 4 |
| 6 | `apollo_core/renko.py` | 1 |
| 7 | `apollo_core/constants.py` | 2 |
| 8 | `layersignal_engine/core.py` | 3 |
| 9 | `layersignal_engine/__init__.py` | 1 |
| 10 | `data_repo/repo.py` | 5 |
| 11 | `data_repo/sources.py` | 0 (empty, but needs init) |
| 12-20 | `bridge_daily_update.py`, `signal_export.py`, `db_builder.py`, `historical_batch_replay.py`, `signal_profiler.py`, `trade_engine.py`, `simulator.py`, `audit_system.py`, `inbox_processor.py` | ~15 total |
| 21-25 | `guidance_engine/*.py` (8 files) | ~6 total |
| 26-30 | `live_engine/*.py` (watchlist, alert_manager, positions) | ~3 total |
| 31-37 | `apollo_api/routes/*.py` + `main.py` + `kite_broker_engine.py` + `live_market_streamer.py` + `vectorized_scoring.py` | ~5 total |

**Action — Create path resolver utility:**

**File:** `apollo_core/path_resolver.py` (NEW)

```python
import os
from pathlib import Path

APOLLO_DATA_ROOT = os.environ.get(
    "APOLLO_DATA_ROOT",
    Path(__file__).resolve().parent.parent.parent / "data"
)

def data_path(*parts: str) -> Path:
    """Resolve any data file path relative to APOLLO_DATA_ROOT."""
    return Path(APOLLO_DATA_ROOT).joinpath(*parts)

def repo_path(symbol: str, timeframe: str = "daily") -> Path:
    """Get parquet file path for a symbol."""
    suffix = "" if timeframe == "daily" else f"_{timeframe}"
    return data_path("data_repo", f"{symbol}{suffix}.parquet")
```

Then in every affected file, replace all hardcoded paths:

```python
# BEFORE:
df = pd.read_parquet(r"C:\Users\Akhil\...\data_repo\RELIANCE.parquet")

# AFTER:
from apollo_core.path_resolver import repo_path
df = pd.read_parquet(repo_path("RELIANCE"))
```

**Verification:** `rg -l 'C:\\' apollo_core/ layersignal_engine/ data_repo/ guidance_engine/ live_engine/ apollo_api/` returns 0 files.

### Step 0.2: Fix requirements.txt

**Problem:** The existing `requirements.txt` is missing critical dependencies. FastAPI server cannot start without them.

**Add these packages:**

```txt
# === Existing (keep) ===
pandas>=2.0
numpy>=1.24
scikit-learn>=1.3
joblib>=1.3
matplotlib>=3.7
openpyxl>=3.1

# === Missing — ADD ===
fastapi>=0.104
uvicorn[standard]>=0.24
pydantic>=2.5
websockets>=12.0
aiosqlite>=0.19
httpx>=0.25
beautifulsoup4>=4.12
lxml>=4.9
python-dotenv>=1.0

# === Phase 3 (ML) — ADD now to avoid later install ===
xgboost>=2.0
shap>=0.43

# === Phase 8 (Kite) — ADD now ===
kiteconnect>=4.1

# === Development ===
pytest>=7.4
pytest-asyncio>=0.21
httpx  # for TestClient
```

**Verification:** `pip install -r requirements.txt` completes with no errors.

### Step 0.3: Fix Known Bugs

| Bug | File:Line | Fix | Priority |
|-----|-----------|-----|----------|
| SQL injection | `apollo_api/routes/analytics.py:130` | Replace f-string `f"WHERE e.symbol = '{symbol}'"` with parameterized query `cursor.execute(..., (symbol,))` | **P0 Security** |
| UnboundLocalError | `signal_export.py:46-50` | Dedent `return` for FLAT/HOLD so they are NOT inside the `if` block | **P0 Runtime** |
| Fake DB scoring | `db_builder.py:63` | `base_score = 50 + (prox_52w_high * 0.4)` is NOT real Apollo scoring. Add `# TODO: Replace with scoring.score_stock()`. Do NOT use for /api/signals. | **P1 Correctness** |
| Missing init | `data_repo/sources.py` | File is empty placeholder. Add basic structure or remove import references. | **P2 Import** |

### Step 0.4: Create Service Layer Directory Structure

```bash
apollo_api/
  services/              # NEW — all service modules live here
    __init__.py
    signals_service.py   # Phase 1
    highs_service.py     # Phase 2
    ml_service.py        # Phase 3
    ipo_service.py       # Phase 4
    alerts_service.py    # Phase 5
    journal_service.py   # Phase 6
    analytics_service.py # Phase 6
  routes/
    signals.py           # Phase 1 — NEW route file
    highs.py             # Phase 2 — NEW route file
    ml.py                # Phase 3 — NEW route file
    ipo.py               # Phase 4 — NEW route file
    alerts.py            # Phase 5 — NEW route file
    db.py                # Phase 6 — trades, journal
    system.py            # Phase 6 — health, logs
    # ... existing route files untouched ...
  models/
    __init__.py          # NEW — Pydantic response models
    signals.py           # Phase 1
    highs.py             # Phase 2
    ml.py                # Phase 3
    ipo.py               # Phase 4
    alerts.py            # Phase 5
    db.py                # Phase 6
```

### Step 0.5: Verify Baseline

```bash
# 1. Install dependencies
pip install -r requirements.txt

# 2. Set environment
export APOLLO_DATA_ROOT=/path/to/your/data
cp .env.example .env  # create if not exists

# 3. Start existing FastAPI server
uvicorn apollo_api.main:app --host 0.0.0.0 --port 8000

# 4. Verify all 25 existing endpoints return 200
curl -s http://localhost:8000/api/healthz | python -m json.tool
curl -s http://localhost:8000/api/guidance/trades | python -m json.tool
# ... test remaining 23 endpoints
```

**Phase 0 Exit Criteria:** Server starts, all 25 existing endpoints return valid JSON, zero hardcoded Windows paths, zero import errors.

---
## 7. Phase 1: Build signals_service.py (P0 — Critical)

**Duration:** 3-4 days  **Depends on:** Phase 0  **Blocks:** Phases 2, 3, 4, 5, 7, 9, 10

**Why P0:** This is the single most critical deliverable in the entire project. 6 out of 10 dashboard tabs (Screener, New Highs, ML, IPO, Scanner, Analytics) consume `SignalStock[]` from `/api/signals`. The React frontend already calls this endpoint — it just gets fake data from Express today. Once this endpoint returns real Apollo+LayerSignal scores, the majority of the dashboard lights up with real data.

### Step 1.1: Create Pydantic Response Models

**File:** `apollo_api/models/signals.py`

These models MUST match the TypeScript interfaces in `src/types.ts` field-for-field. This is the immutable contract.

```python
from pydantic import BaseModel
from typing import Optional, List

class SubScores(BaseModel):
    trend: float
    momentum: float
    volatility: float
    volume: float
    marketFilter: float

class HistoricalL3Event(BaseModel):
    date: str
    symbol: str
    entry_price: float
    outcome_20d_pct: float
    exit_reason: str

class SignalStock(BaseModel):
    # === Identity (3) ===
    Symbol: str
    Date: str                      # ISO date string
    Pct_Change: float

    # === Apollo Scoring (2) ===
    Apollo_Action: str             # HOLD | ENTRY | EXIT | FLAT
    Apollo_Score: float            # 0-148

    # === LayerSignal (3) ===
    LayerSignal_Action: str        # HOLD | ENTRY | EXIT | FLAT
    LayerSignal_Score: float       # 20-100
    Exit_Pressure: float           # 5-95

    # === Price Data (9) ===
    Open: float
    High: float
    Low: float
    Close: float
    Volume: float
    High52W: float
    Low52W: float
    CMP: float
    Traded_Value: float

    # === Technical Indicators (7) ===
    RSI21: float
    RSI36: float
    RSI56: float
    ADX: float
    ATR_Pct: float
    Stochastic: float
    PE: float

    # === Moving Averages (3, nullable) ===
    SMA_20D: Optional[float] = None
    SMA_50D: Optional[float] = None
    SMA_200D: Optional[float] = None

    # === Derived (4) ===
    Prox_52W: float                # (Close / High52W) * 100
    Conviction: float              # Apollo_Score / 148
    FQS: str                       # A | B | C | D

    # === Classification (5) ===
    Bucket: str                    # L1 | L2 | L3 | L4
    LStage: str
    ELStatus: str                  # EL1 | EL2 | EL3 | EL4 | NONE
    Renko: str                     # GREEN | RED | NEUTRAL
    MCap: str                      # Large | Mid | Small
    Gates: List[bool]              # [G1, G2, G3, G4, G5]

    # === Optional Enrichment (7) ===
    Sparkline: Optional[List[float]] = None
    ThrowbackAlert: Optional[bool] = None
    ml_confidence: Optional[float] = None
    SubScores: Optional[SubScores] = None
    GatesExplanations: Optional[List[str]] = None
    HistoricalL3Events: Optional[List[HistoricalL3Event]] = None
    isIPO: Optional[bool] = None


class SignalsSummary(BaseModel):
    total: int
    liquid: int
    scored: int
    signalBearing: int
    entryCount: int
    holdCount: int
    exitCount: int
    flatCount: int
    avgScore: float
    avgExitPressure: float
    buckets: dict
    quality: dict
    generatedAt: str


class SignalsResponse(BaseModel):
    stocks: List[SignalStock]
    summary: SignalsSummary
```

**Critical rule:** Field names must match TypeScript `SignalStock` in `src/types.ts` EXACTLY. Verify by comparing against `src/types.ts` lines 1-135. The JSON keys are case-sensitive — `Apollo_Score` not `apollo_score`.

### Step 1.2: Create SignalsService Class

**File:** `apollo_api/services/signals_service.py`

This is the orchestration layer. It calls existing engine functions and assembles the 49-field output. It does NOT re-implement any scoring logic.

```python
class SignalsService:
    def __init__(self):
        self._cache = None
        self._cache_time = None
        self._cache_ttl = int(os.environ.get("SIGNALS_CACHE_TTL", "300"))
        self._prev_snapshot = None  # for alerts diff

    def get_signals(self, force_refresh: bool = False) -> SignalsResponse:
        if not force_refresh and self._cache and self._cache_time:
            if time.time() - self._cache_time < self._cache_ttl:
                return self._cache
        stocks = self._score_all_stocks()
        summary = self._compute_summary(stocks)
        response = SignalsResponse(stocks=stocks, summary=summary)
        self._cache = response
        self._cache_time = time.time()
        return response
```

**Core scoring loop for a single stock (the heart of the system):**

```python
    def _score_single_stock(self, symbol: str) -> Optional[SignalStock]:
        # 1. Load OHLCV data from parquet
        df = self._load_ohlcv(symbol)
        if df is None or len(df) < 60:
            return None

        # 2. Compute indicators (17 indicators from indicators.py)
        indicators = compute_all_indicators(df)

        # 3. Apollo scoring (0-148, 38 signals)
        apollo_result = score_stock(df, indicators)
        apollo_score = apollo_result["total_score"]
        apollo_action = classify_score(apollo_score)
        pool_scores = apollo_result.get("pool_scores", {})

        # 4. LayerSignal
        ls_score = calc_momentum_score(df)
        exit_pressure = compute_exit_layers(df)
        ls_action = evaluate_trade_signals(df)

        # 5. Gates (G1-G4 from gates.py + G5 Quality glue)
        gates_result = evaluate_gates(df, indicators)
        gates_bools = [
            gates_result.get("G1_PE", False),
            gates_result.get("G2_Liquidity", False),
            gates_result.get("G3_52W_Low", False),
            gates_result.get("G4_52W_High", False),
            self._compute_g5_quality(indicators),
        ]
        gates_explanations = self._format_gates_explanations(gates_result, gates_bools)

        # 6. Bucket classification (8-bucket -> L1-L4)
        bucket_result = classify_bucket(df, indicators)
        bucket = bucket_result["tier"]
        lstage = bucket_result.get("stage_description", "")
        fqs = self._compute_fqs(apollo_score)

        # 7. Assemble SignalStock (49 fields)
        return self._assemble_signal_stock(
            symbol=symbol, df=df, indicators=indicators,
            apollo_score=apollo_score, apollo_action=apollo_action,
            pool_scores=pool_scores, ls_score=ls_score,
            ls_action=ls_action, exit_pressure=exit_pressure,
            gates_bools=gates_bools, gates_explanations=gates_explanations,
            bucket=bucket, lstage=lstage, fqs=fqs,
        )
```

**Glue logic methods (do NOT exist in Python yet — must be written):**

| Method | Lines | Logic | Source Reference |
|--------|-------|-------|-----------------|
| `_compute_g5_quality(indicators)` | ~5 | RSI21 > RSI36 > RSI56 AND all between 40-70 | New gate — not in Python | |
| `_compute_fqs(score)` | ~5 | Score>=110->A, >=85->B, >=60->C, else D | `server.ts:520-530` equivalent | |
| `_compute_mcap(cmp, traded_value)` | ~10 | Large(>500cr), Mid(100-500cr), Small(<100cr) daily traded value | `server.ts:540-560` | |
| `_detect_throwback_alert(df, score, sma20, sma50)` | ~10 | CMP < 20SMA, 20SMA >= 50SMA, score >= 55 | `server.ts:600-630` | |
| `_build_sub_scores(pool_scores)` | ~20 | Map 5 pool scores to SubScores object | `server.ts:450-480` | |
| `_format_gates_explanations(gates_result, gates_bools)` | ~15 | Human-readable pass/fail per gate | `server.ts:490-520` | |
| `_get_historical_l3_events(symbol, df)` | ~30 | Find last 3 L3 triggers from parquet history with 20d outcome | New — scan layersignal history | |
| `_check_is_ipo(symbol)` | ~5 | Check ipo_listing_dates.json | Existing: `ipo_lookup.py` | |

**Implementation Notes:**

1. `compute_all_indicators(df)` is a wrapper you must write that calls the 17 individual indicator functions from `apollo_core/indicators.py` and returns a dict. Check `indicators.py` for exact function signatures — some take `(df, period)` while others take `(df)`.
2. `score_stock(df, indicators)` returns a dict from `scoring.py`. Inspect to find exact return structure: key fields are `total_score`, `action`, and per-pool scores. Pools map to SubScores: Pool A (Trend), Pool B (Momentum), Pool C (Volatility), Pool D (Volume), Pool E (Market Filter).
3. `calc_momentum_score(df)` from `layersignal_engine/core.py` — verify the exact return type (float 20-100 vs dict). Adapt accordingly.
4. `classify_bucket(df, indicators)` — verify return structure. You need `tier` (L1-L4) and `stage_description`.
5. `get_renko_trend(df)` — verify this exists in `renko.py`. If not, Renko is inside `scoring.py` as Pool B. Extract it.
6. `evaluate_gates(df, indicators)` — returns dict with boolean results. Map to G1-G4. G5 is new glue logic.

### Step 1.3: Create FastAPI Route

**File:** `apollo_api/routes/signals.py`

```python
from fastapi import APIRouter, Query
from apollo_api.services.signals_service import SignalsService
from apollo_api.models.signals import SignalsResponse

router = APIRouter(prefix="/api", tags=["signals"])
service = SignalsService()

@router.get("/signals", response_model=SignalsResponse)
async def get_signals(force_refresh: bool = Query(False)):
    return service.get_signals(force_refresh=force_refresh)

@router.post("/signals/sync")
async def sync_signals():
    service.invalidate_cache()
    result = service.get_signals(force_refresh=True)
    service.save_snapshot_for_diff(result.stocks)
    return {"status": "ok", "stocks_scored": len(result.stocks)}
```

Register in `apollo_api/main.py`:

```python
from apollo_api.routes.signals import router as signals_router
app.include_router(signals_router)
```

### Step 1.4: Data Source Abstraction — GSheet (Interim Path)

Until Phase 8 (Kite Connect), the system uses Google Sheets as the data source via `bridge_daily_update.py`.

```python
    def _get_symbol_list(self) -> List[str]:
        source = os.environ.get("DATA_SOURCE", "gsheet")
        if source == "parquet":
            return self._list_parquet_symbols()
        elif source == "kite_live":
            return self._list_kite_symbols()
        else:
            return self._fetch_gsheet_symbols()

    def _load_ohlcv(self, symbol: str) -> Optional[pd.DataFrame]:
        path = repo_path(symbol)
        if path.exists():
            return pd.read_parquet(path)
        return None
```

**NOTE:** For GSheet source, the symbol universe comes from GSheet CSV. OHLCV always comes from pre-built parquet files (generated by `bridge_daily_update.py`). The GSheet provides fundamentals (PE, Traded Value) and the symbol list.

### Step 1.5: Caching Strategy

| Parameter | Value | Rationale |
|-----------|-------|----------|
| Default TTL | 300 seconds (5 min) | Matches React 5-minute refetch interval |
| Env override | `SIGNALS_CACHE_TTL` | Tune without code change |
| Invalidation | `POST /api/signals/sync` | Manual force-refresh from System tab |
| Thread safety | Not required (single uvicorn worker) | If multi-worker, add `asyncio.Lock` |

### Step 1.6: Testing Checklist

```bash
# 1. Unit: All 49 fields populated for a single stock
pytest tests/test_signals_service.py::test_single_stock_49_fields -v

# 2. Unit: SignalsSummary counts match actual stock counts
pytest tests/test_signals_service.py::test_summary_counts -v

# 3. Integration: /api/signals returns valid JSON matching Pydantic model
pytest tests/test_api_signals.py::test_signals_endpoint -v

# 4. Comparison: Score matches standalone scoring.py output
pytest tests/test_signals_service.py::test_score_matches_engine -v

# 5. Performance: Full universe (1,400 stocks) in < 60 seconds
pytest tests/test_signals_service.py::test_performance_full_universe -v

# 6. Cache: Second call returns faster (cache hit)
pytest tests/test_signals_service.py::test_cache_hit -v

# 7. Sync: POST /api/signals/sync invalidates cache
pytest tests/test_api_signals.py::test_sync_endpoint -v
```

**Phase 1 Exit Criteria:** `GET /api/signals` returns valid JSON with 1,400+ stocks, each having all 49 fields populated. React Screener tab displays real Apollo scores. Performance: full score cycle under 60 seconds.

---


## 8. Phase 2: Build highs_service.py (P1)

**Duration:** 3-4 days  **Depends on:** Phase 1  **Blocks:** New Highs tab

### Step 2.1: Create Pydantic Models for New Highs

**File:** `apollo_api/models/highs.py`

The `HighsStock` model has 45+ fields. It extends the base SignalStock fields with 52WH-specific enrichment. Reference: `src/types.ts` lines 136-188.

**Key models to create:**

| Model | Fields | Used By |
|-------|--------|----------|
| `ConsistencyData` | high_close_count, total_days, consistency_pct, max_dd_from_high, pattern (LAUNCHPAD\|BUILDING\|CHOPPY\|ONE_TOUCH) | HighsStock |
| `BaseRatesData` | sample_size, status (READY\|INSUFFICIENT_DATA), win_rate, avg_return, median_return, best_case, worst_case | HighsStock |
| `HighsStock` | 45+ fields (see Section 4.2 above for complete field list) | New Highs tab |
| `HighsBreadthData` | total_stocks, new_highs, new_lows, nh_pct, nl_pct, net_hl, regime | NH-NL breadth bar |
| `HighsTierSummary` | AT{count, delta}, NEAR{count, delta}, APPROACHING{count, delta}, EXTENDED{count, delta} | Tier cards |
| `SectorCluster` | sector, stock_count, at_count, near_count, avg_proximity_pct, avg_apollo_score, top_stocks[], strength | Sector panel |
| `HighsApiResponse` | breadth, tier_summary, clusters[], watchlist_alerts[], total, page, limit, data[] | Full response |

**IMPORTANT:** These models MUST match the TypeScript interfaces exactly. Compare against `src/types.ts` line by line.

### Step 2.2: Port 12 Functions from server/highs-service.ts

**File:** `apollo_api/services/highs_service.py`

This service calls `signals_service.get_signals()` as input, then adds 52WH-specific enrichment. It does NOT re-score stocks.

| # | Function Name | Port From | Lines (TS) | Logic Summary |
|---|--------------|-----------|------------|---------------|
| 1 | `compute_proximity_tier(cmp, high52w)` | `highs-service.ts:45-80` | ~35 | AT(>=99%), NEAR(95-99%), APPROACHING(90-95%), EXTENDED(>102%), FAR(<90%) |
| 2 | `detect_rsi_divergence(df, proximity_pct)` | `highs-service.ts:82-160` | ~78 | 4 exhaustion checks if proximity>=95%: RSI>80, RSI divergence, volume divergence, stoch>85. Returns flags[] + severity. |
| 3 | `compute_rate_of_approach(df, proximity_pct)` | `highs-service.ts:162-240` | ~78 | 5d/20d returns, gap-up count, momentum_profile (SPIKE/ACCELERATING/GRINDING/CONSOLIDATING) |
| 4 | `classify_fresh_vs_repeat(df, high52w)` | `highs-service.ts:242-320` | ~78 | FRESH (first time at 52W high), RECLAIMED (was <90%, now back), REPEAT, EXTENDED_STAY (>20 days near high), EARLY_REPEAT |
| 5 | `compute_consistency_score(df, high52w)` | `highs-service.ts:322-400` | ~78 | Count days closing within 3% of 52W high in last 60d. Pattern: LAUNCHPAD(>70%), BUILDING(40-70%), CHOPPY(20-40%), ONE_TOUCH(<20%) |
| 6 | `compute_base_rates(proximity_tier, high_type)` | `highs-service.ts:402-460` | ~58 | Deterministic lookup table (empirical data later). Returns win_rate, avg_return, etc. |
| 7 | `compute_el_confluence(tier, el_status)` | `highs-service.ts:462-520` | ~58 | 15-cell lookup matrix (5 tiers x 3 EL statuses). Score 0-100. |
| 8 | `compute_nh_nl_breadth(stocks)` | `highs-service.ts:522-570` | ~48 | Count new highs, new lows, compute NH-NL breadth, market regime. |
| 9 | `compute_sector_clusters(stocks)` | `highs-service.ts:572-640` | ~68 | Group by sector, aggregate proximity/score/top stocks. |
| 10 | `compute_tier_summary(stocks)` | `highs-service.ts:642-680` | ~38 | Count per tier (AT/NEAR/APPROACHING/EXTENDED) with delta vs prior day. |
| 11 | `generate_watchlist_alerts(stocks, watchlist)` | `highs-service.ts:682-720` | ~38 | Find watchlist stocks at/near 52W highs. |
| 12 | `save_daily_snapshot(stocks, date)` | `highs-service.ts:722-750` | ~28 | Persist daily highs snapshot to DB. |

**Porting approach:**

1. Read the TypeScript source for each function line by line.
2. Convert JavaScript idioms to Python: `array.filter()` to list comprehension, `Math.max()` to `max()`, `&&` to `and`, ternary `a ? b : c` to `b if a else c`.
3. Functions 1-7 operate on individual stocks (per-stock enrichment). Input: one stock's parquet data + SignalStock fields.
4. Functions 8-12 operate on the full list (aggregate analysis). Input: `List[SignalStock]` filtered to `proximity_pct >= 90%`.
5. The `name` and `Sector` fields on HighsStock come from GSheet CSV (not in parquet). Load from GSheet export or a sector mapping file.

### Step 2.3: Create FastAPI Routes

**File:** `apollo_api/routes/highs.py`

| # | Method | Path | Query Params | Response | Description |
|---|--------|------|-------------|----------|-------------|
| 1 | GET | `/api/highs/stocks` | page, limit, tier, sort, order | HighsApiResponse | Paginated 52W highs with full enrichment |
| 2 | GET | `/api/highs/breadth` | — | HighsBreadthData | NH-NL breadth data |
| 3 | GET | `/api/highs/sectors` | — | List[SectorCluster] | Sector-level clustering |
| 4 | GET | `/api/highs/watchlist-alerts` | — | List[dict] | Watchlist stocks at/near highs |
| 5 | POST | `/api/highs/snapshot` | — | {status, count} | Save daily snapshot |

Register in `main.py`:

```python
from apollo_api.routes.highs import router as highs_router
app.include_router(highs_router)
```

### Step 2.4: Test

```bash
curl -s http://localhost:8000/api/highs/stocks?limit=5 | python -m json.tool
curl -s http://localhost:8000/api/highs/breadth | python -m json.tool
curl -s http://localhost:8000/api/highs/sectors | python -m json.tool
curl -s http://localhost:8000/api/highs/watchlist-alerts | python -m json.tool
curl -s -X POST http://localhost:8000/api/highs/snapshot | python -m json.tool
```

**Phase 2 Exit Criteria:** All 5 `/api/highs/*` endpoints return valid JSON. New Highs tab displays proximity tiers, exhaustion flags, momentum profiles, sector clusters, and NH-NL breadth with real data.

---


## 9. Phase 3: Build ml_service.py (P1)

**Duration:** 4-5 days  **Depends on:** Phase 1  **Blocks:** ML Confidence tab

### Step 3.1: Understand the Feature Store

Before writing any ML code, read these 3 files completely:

| File | Purpose | Key Output |
|------|---------|------------|
| `apollo_core/precompute_features.py` | Pre-computes 20 features per stock per day | `features/{SYMBOL}_features.parquet` |
| `apollo_core/historical_batch_replay.py` | Creates labeled dataset (signal -> 20d forward return) | `signal_outcomes.parquet` ~1,250 pairs from 5yr history |
| `apollo_core/signal_profiler.py` | Profiles signal quality distributions | Statistics parquet |

**Action:** Understand the 20 feature names, the label definition (5%+ gain in 20 days?), and data shapes. The XGBoost model consumes this exact feature set.

### Step 3.2: Create ML Service

**File:** `apollo_api/services/ml_service.py`

**Class structure:**

```python
class MLService:
    def __init__(self):
        self.model_5d = None   # XGBClassifier
        self.model_10d = None
        self.model_20d = None  # Primary model
        self.feature_names = []
        self.metadata = {}
        self._load_models()

    def predict(self, features, horizon="20d") -> float:
        """P(positive return) for single stock. Returns 0.0-1.0."""

    def predict_all(self, stocks_features, horizon="20d") -> Dict[str, float]:
        """Batch prediction for all stocks."""

    def explain(self, features) -> List[Dict]:
        """SHAP-based feature contribution breakdown."""

    def train(self, horizon="20d") -> Dict[str, Any]:
        """Train with purged GroupTimeSeriesSplit CV."""

    def get_feature_importance(self) -> List[Dict]:
        """Global feature importance ranking."""
```

**Key implementation details:**

| Aspect | Specification |
|--------|--------------|
| Model type | `XGBClassifier` (binary: 5%+ gain in 20d yes/no) |
| CV strategy | `GroupTimeSeriesSplit(n_splits=5)` with 5-day embargo |
| Target AUC | > 0.60 (above random, meaningful edge) |
| Hyperparameters | n_estimators=500, max_depth=4, lr=0.05, subsample=0.8, early_stopping_rounds=30 |
| Explainability | `shap.TreeExplainer` for per-stock contribution breakdown |
| Persistence | `joblib.dump()` to `models/xgb_{horizon}.joblib` |
| Metadata | `models/ml_metadata.json` — AUC, feature_count, trained_at, calibration_status |

**Multi-horizon predictions:**

| Horizon | Target | Label Definition |
|---------|--------|---------------|
| 5d | `ml_confidence_5d` | P(2%+ gain in 5 days) |
| 10d | `ml_confidence_10d` | P(3%+ gain in 10 days) |
| 20d | `ml_confidence` (primary) | P(5%+ gain in 20 days) |

### Step 3.3: Integrate with signals_service

The ML service does NOT re-score stocks. It receives features and returns predictions. The integration point is in `signals_service._score_single_stock()`, after assembling base SignalStock:

```python
# In SignalsService._score_single_stock():
if self._ml_service and self._ml_service.is_model_loaded:
    features = self._extract_ml_features(df, indicators)
    stock.ml_confidence = self._ml_service.predict(features, "20d")
```

The `_extract_ml_features()` method reads from the pre-computed `features/{SYMBOL}_features.parquet` file (generated by `precompute_features.py`). If the file doesn't exist, `ml_confidence` stays `None`.

### Step 3.4: MLStockPrediction Assembly

`GET /api/ml/stocks` enriches SignalStock[] with ML-specific fields:

| Field | Computation |
|-------|-------------|
| `ml_confidence` (3 horizons) | `ml_service.predict(features, horizon)` |
| `confidence_tier` | HIGH(>=70%), MODERATE(55-69%), NEUTRAL(45-54%), LOW(<45%) |
| `is_triple_confluence` | Apollo>=80 AND ELStatus in (EL1,EL2) AND ml_confidence>=0.70 |
| `is_high_conviction` | ml_confidence >= 0.70 |
| `is_contrarian` | ml_confidence>=0.65 AND Apollo_Score<80 |
| `is_dual_engine` | Apollo_Score>=80 AND ELStatus!=NONE |
| `contributions[]` | SHAP values: {feature, label, value, impact, category} |
| `trade_sizing` | Tier-based: allocation%, R:R ratio, stop-loss%, action_summary |

### Step 3.5: Create FastAPI Routes

**File:** `apollo_api/routes/ml.py`

| # | Method | Path | Description |
|---|--------|------|-------------|
| 1 | GET | `/api/ml/stocks` | All stocks with ML predictions, confluence flags, SHAP, trade sizing |
| 2 | GET | `/api/ml/info` | Model metadata: status, AUC, feature count, calibration |
| 3 | GET | `/api/ml/importance` | Global feature importance ranking |
| 4 | POST | `/api/ml/retrain` | Trigger model retraining with latest data |

### Step 3.6: ML Training Workflow

| Step | Action | Command / File |
|------|--------|---------------|
| 1 | Generate feature parquets | `python -m apollo_core.precompute_features --universe all` |
| 2 | Build labeled dataset | `python -m apollo_core.historical_batch_replay --horizon 20d` |
| 3 | Train model | `POST /api/ml/retrain` or `python -m apollo_api.services.ml_service --train` |
| 4 | Verify AUC | `GET /api/ml/info` — target AUC > 0.60 |
| 5 | Calibration (future) | Platt scaling or isotonic regression on held-out set |

**Important:** If no model is trained, `ml_confidence` returns `null` for all stocks. The ML tab shows a "Model Not Loaded" state with a "Train Model" button (already in React). The system is fully functional without ML — it is an enhancement, not a dependency.

**Phase 3 Exit Criteria:** `GET /api/ml/info` returns model metadata. `GET /api/ml/stocks` returns predictions with SHAP contributions. `POST /api/ml/retrain` completes. ML Confidence tab displays real XGBoost predictions.

---


## 10. Phase 4: Build ipo_service.py (P2)

**Duration:** 3 days  **Depends on:** Phase 1 (for cross-matching scores)  **Blocks:** IPO tab

### Step 4.1: Port IPO Scraper from TypeScript

**Source:** `server/ipo-scraper.ts` (~250 lines)  
**Target:** Part of `apollo_api/services/ipo_service.py`

The TypeScript scraper fetches IPO listing data from chittorgarh.com. Port to Python:

| TS Construct | Python Equivalent |
|-------------|-------------------|
| `fetch(url)` | `httpx.AsyncClient.get(url)` |
| `DOMParser` / `querySelectorAll` | `BeautifulSoup4(html, 'lxml').select()` |
| 22 seeded fallback records | Keep in `data/ipo_seeded_data.json` |

**Scraping flow:**
1. Fetch main IPO page from chittorgarh.com
2. Parse table rows: company name, symbol, issue price, listing date, listing price, sector, exchange, IPO size
3. For each IPO stock, look up current CMP, ATH, ATL from parquet data
4. Merge with seeded data for stocks not on the website

### Step 4.2: Port IPO Zone Classifier

**Source:** `server/ipo-classifier.ts` (~200 lines)

4-zone classification system — port the logic exactly:

| Zone | Condition | Description |
|------|-----------|-------------|
| `new_high` | CMP > All-Time High | Stock surpassed its IPO listing high |
| `recovery` | CMP between ATH and Baseline | Recovering from below IPO baseline |
| `under_pressure` | CMP between Baseline and 80% of Issue Price | Struggling but above critical level |
| `broken_ipo` | CMP < 80% of Issue Price | Catastrophic — IPO has failed |

**IPO Baseline** is typically the listing price or the 20-day post-listing average high. The TypeScript uses the listing price as baseline.

```python
def classify_ipo_zone(cmp, issue_price, listing_price, all_time_high, ipo_baseline) -> str:
    if cmp > all_time_high:     return "new_high"
    elif cmp >= ipo_baseline:   return "recovery"
    elif cmp >= issue_price * 0.8: return "under_pressure"
    else:                       return "broken_ipo"
```

### Step 4.3: Port IPO Graduator

**Source:** `server/ipo-graduator.ts` (~100 lines)

**Graduation rule:** IPO stock graduates (moves to archive) when:
- Listed for > 365 days AND
- Currently in `new_high` zone

On graduation: move from `ipo_stocks` table to `ipo_archive` table, record `graduated_at` timestamp.

### Step 4.4: Cross-Reference with Signals Service

For each IPO stock, enrich with Apollo and LayerSignal scores:

```python
async def get_ipo_stocks(self):
    ipo_stocks = self._load_ipo_stocks()
    signals = self._signals_service.get_signals()  # uses cache
    signals_map = {s.Symbol: s for s in signals.stocks}

    for ipo in ipo_stocks:
        signal = signals_map.get(ipo.symbol)
        if signal:
            ipo.Apollo_Score = signal.Apollo_Score
            ipo.LayerSignal_Score = signal.LayerSignal_Score
            ipo.Apollo_Action = signal.Apollo_Action
            ipo.Quality = signal.FQS
            ipo.Bucket = signal.Bucket
            ipo.Gates = signal.Gates
            ipo.RSI21 = signal.RSI21
            ipo.ADX = signal.ADX
            ipo.ATR_Pct = signal.ATR_Pct
```

### Step 4.5: IPOStock 28-Field Model

Reference: `src/types.ts` for the complete `IPOStock` interface. Key fields:

| Field | Source |
|-------|--------|
| symbol, company_name, issue_price, listing_date | Scraper / seeded data |
| listing_price, all_time_high, all_time_low | Parquet data |
| ipo_size, sector, exchange | Scraper |
| promoter_stake, current_pe | GSheet / parquet |
| cmp | Latest close from parquet |
| zone | `classify_ipo_zone()` |
| ipo_baseline | listing_price (or configurable) |
| listing_stage, days_since_listing | Computed from listing_date |
| distance_to_baseline_pct, distance_to_ath_pct | Computed |
| return_from_issue_pct, baseline_ratio, ath_recovery_pct | Computed |
| Apollo_Score, LayerSignal_Score, Apollo_Action, Quality, Bucket, Gates, RSI21, ADX, ATR_Pct | Cross-ref from signals_service |

### Step 4.6: Create 8 FastAPI Routes

**File:** `apollo_api/routes/ipo.py`

| # | Method | Path | Port From | Description |
|---|--------|------|-----------|-------------|
| 1 | GET | `/api/ipo/stocks` | `ipo-routes.ts:61` | All IPO stocks with zone + Apollo/LS scores |
| 2 | GET | `/api/ipo/stocks/{symbol}` | `ipo-routes.ts:134` | Single IPO stock detail |
| 3 | GET | `/api/ipo/summary` | `ipo-routes.ts:184` | Zone counts, stage breakdown, avg returns |
| 4 | GET | `/api/ipo/zone-history` | `ipo-routes.ts:260` | All zone transitions |
| 5 | GET | `/api/ipo/zone-history/{symbol}` | `ipo-routes.ts:278` | Zone transitions for one stock |
| 6 | POST | `/api/ipo/refresh` | `ipo-routes.ts:299` | Trigger scraper, reclassify zones |
| 7 | POST | `/api/ipo/graduate/{symbol}` | `ipo-routes.ts:331` | Graduate an IPO stock to archive |
| 8 | GET | `/api/ipo/archive` | `ipo-routes.ts:353` | List graduated/archived IPO stocks |

**Phase 4 Exit Criteria:** All 8 `/api/ipo/*` endpoints return valid JSON. IPO tab displays zone classification, transitions, and Apollo/LS enrichment correctly.

---


## 11. Phase 5: Build alerts_service.py (P2)

**Duration:** 3 days  **Depends on:** Phase 1 (for diff detection)  **Blocks:** Alerts tab

### Step 5.1: Design the Alert System

The current Express server has `detectAndLogStatusChanges()` running every 30 seconds. It compares current stock states against the previous scan and logs transitions. The Python replacement needs:

1. **State-change detection** — Compare current `SignalStock[]` snapshot vs previous
2. **Rule engine** — User-defined rules (condition + channel + enabled)
3. **WebSocket push** — Real-time alert delivery to React
4. **Persistence** — Alert logs and rules in aiosqlite

### Step 5.2: State-Change Detection

**File:** `apollo_api/services/alerts_service.py`

The `detect_changes()` method compares the current scoring snapshot against the previous one (stored by `signals_service.save_snapshot_for_diff()`). It detects 5 types of transitions:

| # | Transition | Condition | Alert Source | Alert Type |
|---|-----------|-----------|-------------|------------|
| 1 | Action change | `old.action != new.Apollo_Action` | Apollo | Entry/Exit |
| 2 | Score threshold | `old.score < 100 <= new.Apollo_Score` | Apollo | Scoring |
| 3 | Bucket change | `old.bucket != new.Bucket` | Apollo | Regime |
| 4 | EL Status change | `old.el != new.ELStatus` | LayerSignal | Entry/Exit |
| 5 | All gates pass | `sum(old.gates) < 5 AND sum(new.Gates) == 5` | Apollo | Scoring |

**For each transition, create an `AlertItem`:**

```python
@dataclass
class AlertItem:
    id: Optional[int] = None
    source: str       # Apollo | LayerSignal | System | IPO
    type: str         # Entry | Exit | Regime | Scoring | System
    timestamp: str    # ISO
    symbol: Optional[str] = None
    title: str = ""
    message: str = ""
    read: bool = False
```

### Step 5.3: Rule Engine

Users can create custom alert rules via the Alerts tab. Each rule has:

| Field | Type | Example |
|-------|------|--------|
| name | string | "Score above 100" |
| condition | JSON string | `{"field": "Apollo_Score", "op": ">=", "value": 100}` |
| channel | string | "browser" \| "websocket" \| "log" |
| enabled | boolean | true |

**Supported operators:** `>=`, `<=`, `>`, `<`, `==`

**Evaluation:** On each 30s scan cycle, for each stock, evaluate all enabled rules. If a rule fires and hasn't fired for that stock in the last hour (dedup window), create an AlertItem.

### Step 5.4: FastAPI Routes

**File:** `apollo_api/routes/alerts.py`

| # | Method | Path | Port From | Description |
|---|--------|------|-----------|-------------|
| 1 | GET | `/api/db/alerts` | `server.ts:989` | Alert logs (paginated, filterable) |
| 2 | POST | `/api/db/alerts/test` | `server.ts:1008` | Generate test alert |
| 3 | POST | `/api/db/alerts/simulate-status-change` | `server.ts:1044` | Simulate state change |
| 4 | POST | `/api/db/alerts/mark-read` | `server.ts:1082` | Mark alerts as read |
| 5 | GET | `/api/db/rules` | `server.ts:1100` | List all alert rules |
| 6 | POST | `/api/db/rules` | `server.ts:1118` | Create new rule |
| 7 | PATCH | `/api/db/rules/{id}` | `server.ts:1131` | Update rule |
| 8 | DELETE | `/api/db/rules/{id}` | `server.ts:1145` | Delete rule |

### Step 5.5: Background Alert Scheduler

```python
from apscheduler.schedulers.asyncio import AsyncIOScheduler

scheduler = AsyncIOScheduler()

async def alert_scan_job():
    """Run every 30 seconds during market hours (9:15 AM - 3:30 PM IST)."""
    if not is_market_hours():
        return
    signals = signals_service.get_signals(force_refresh=True)
    alerts = alerts_service.detect_changes(signals.stocks)
    await alerts_service.persist_alerts(alerts)
    await websocket_manager.broadcast({"type": "NEW_ALERT", "data": alerts})
    signals_service.save_snapshot_for_diff(signals.stocks)

scheduler.add_job(alert_scan_job, 'interval', seconds=30)
```

**Phase 5 Exit Criteria:** Alert detection runs on 30s cycle during market hours. State transitions generate alerts. Rules can be CRUD'd via API. Alerts persist to DB. WebSocket pushes work. Alerts tab displays real-time alerts.

---


## 12. Phase 6: Build journal + analytics services (P3)

**Duration:** 2 days  **Depends on:** Phase 1  **Blocks:** Watchlist tab (journal), System tab (health)

### Step 6.1: journal_service.py — Trade & Journal CRUD

**File:** `apollo_api/services/journal_service.py`

This service manages trade records and journal entries. It replaces the SQLite operations in `server/db.ts` (~300 lines) and `server.ts` lines 926-980.

**Database tables:**

```sql
CREATE TABLE IF NOT EXISTS trades (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    symbol TEXT NOT NULL,
    entry_date TEXT NOT NULL,
    exit_date TEXT,
    entry_price REAL NOT NULL,
    exit_price REAL,
    pnl_pct REAL,
    holding_days INTEGER,
    exit_mode TEXT  -- 'Target Hit' | 'Stop Loss' | 'Time Exit' | 'Signal Exit'
);

CREATE TABLE IF NOT EXISTS journal_entries (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    symbol TEXT NOT NULL,
    date TEXT NOT NULL,
    note TEXT,
    target REAL,
    stop_loss REAL,
    created_at TEXT DEFAULT (datetime('now'))
);
```

**FastAPI Routes:**

| # | Method | Path | Description |
|---|--------|------|-------------|
| 1 | GET | `/api/db/trades` | List all trades (filterable by symbol) |
| 2 | POST | `/api/db/trades` | Create new trade record |
| 3 | GET | `/api/db/journal` | List journal entries (filterable by symbol) |
| 4 | POST | `/api/db/journal` | Create new journal entry |

**React data shape:** The React Watchlist tab expects `TradeRecord` objects. Match the TypeScript interface exactly.

### Step 6.2: analytics_service.py — System Health

**File:** `apollo_api/services/analytics_service.py`

This service powers the System tab. It provides:

1. **System health** — engine status, scan times, database sizes
2. **Score histogram** — distribution of Apollo_Score across universe
3. **Log viewer** — recent system logs from SQLite

```python
@router.get("/api/system/health")
async def get_system_health() -> SystemHealthData:
    return SystemHealthData(
        apolloScanTime=...,
        apolloDuration=...,
        apolloProcessed=len(signals_service.get_signals().stocks),
        apolloStatus="OK",
        layerScanTime=...,
        layerDuration=...,
        layerPatterns=...,
        layerStatus="OK",
        dbSizeMB=...,
        dbTables=[...],
        lastDbUpdate=...,
        apiEndpoints=[...],  # auto-discovered from FastAPI app.routes
    )
```

**Phase 6 Exit Criteria:** Trade CRUD works via API. Journal CRUD works. System tab shows real health data.

---


## 13. Phase 7: React Frontend Migration

**Duration:** 2 days  **Depends on:** Phases 1-6 complete  **Blocks:** Nothing (final integration)

This is the simplest phase. Because we maintained the immutable data contract (49-field SignalStock matching TypeScript interface), the React components need zero changes to their rendering logic.

### Step 7.1: Delete Express Server Files

See Section 18 (React Kill List) for the complete list. Summary:

```bash
rm server.ts
rm server/db.ts
rm server/highs-service.ts
rm server/highs-routes.ts
rm server/ml-service.ts
rm server/ml-routes.ts
rm server/ipo-routes.ts
rm server/ipo-classifier.ts
rm server/ipo-scraper.ts
rm server/ipo-graduator.ts
rm src/utils/enrichment.ts
```

### Step 7.2: Modify App.tsx — API Base URL

**File:** `src/App.tsx`

The only code change in the entire React codebase:

```typescript
// BEFORE (Express on same port as Vite):
const API_BASE = '';

// AFTER (Python FastAPI on port 8000):
const API_BASE = 'http://localhost:8000';
```

Or better, use an environment variable:

```typescript
const API_BASE = import.meta.env.VITE_API_BASE || 'http://localhost:8000';
```

Then add to `.env`:
```
VITE_API_BASE=http://localhost:8000
```

### Step 7.3: Remove sql.js Dependency

The React app currently uses `sql.js` (WASM SQLite) for client-side persistence of trades, journal, alerts, and IPO data. After migration:

1. Remove `sql.js` import and initialization from `App.tsx`
2. Remove `initDatabase()`, `loadFromDB()`, `saveToDB()` calls
3. All data now comes from FastAPI endpoints (already called in App.tsx for other data)
4. Remove `@hpcc-js/wasm` or `sql.js` from `package.json`

### Step 7.4: Update Vite Dev Server Config

Since React now runs on port 5173 (Vite default) and API is on port 8000, add a proxy for development:

```typescript
// vite.config.ts
export default defineConfig({
  server: {
    proxy: {
      '/api': {
        target: 'http://localhost:8000',
        changeOrigin: true,
      }
    }
  }
})
```

This eliminates the need for `VITE_API_BASE` — just use relative URLs `/api/...`.

### Step 7.5: Remove Enrichment Fallback

**File:** `src/utils/enrichment.ts` (DELETE this file entirely)

In `App.tsx`, remove the fallback call that triggers when the API returns empty data. With the Python backend, the API will always return real data.

### Step 7.6: Tab-by-Tab Verification

| Tab | API Endpoints Called | Verification | Expected Result |
|-----|---------------------|--------------|-----------------|
| Screener | `GET /api/signals` | Scores are real (not hashCode random) | Real Apollo scores 0-148 |
| IPO | `GET /api/ipo/stocks`, `/api/ipo/summary` | Zones match classifier logic | 4 zones with correct counts |
| New Highs | `GET /api/highs/stocks`, `/api/highs/breadth` | Proximity tiers, exhaustion flags | Real proximity from parquet |
| ML Confidence | `GET /api/ml/stocks`, `/api/ml/info` | Real XGBoost predictions (or MODEL_NOT_LOADED) | SHAP contributions display |
| Watchlist | `GET /api/db/trades`, `/api/db/journal` | CRUD works | Trades persist across refresh |
| Scanner | Filters client-side SignalStock[] | No API change needed | Works automatically |
| Analytics | Aggregates client-side SignalStock[] | No API change needed | Works automatically |
| Alerts | `GET /api/db/alerts`, `GET /api/db/rules` | Real-time alerts via WebSocket | New alerts appear in real-time |
| Guidance | `GET /api/guidance/*` (6 existing endpoints) | Already working | No change needed |
| System | `GET /api/system/health`, `GET /api/health` | Real health data | Scan times, DB sizes |

### Step 7.7: Remove package.json Dependencies

```bash
npm uninstall sql.js  # or @hpcc-js/wasm
```

**Phase 7 Exit Criteria:** React app runs on Vite dev server. All 10 tabs display real data from Python FastAPI. No Express server running. Zero scoring logic in React.

---


## 14. Phase 8: Kite Connect Live Data Pipeline

**Duration:** 4-5 days  **Depends on:** Phase 0  **Blocks:** Real-time data (can run parallel to Phases 1-7)

This phase replaces the Google Sheet data source with Zerodha Kite Connect for live market data. It can be developed in parallel with Phases 1-7 because the `DATA_SOURCE` env var abstracts the data source.

### Step 8.1: Implement kite_broker_engine.py

**Current state:** STUB — `kite.place_order()` is commented out, returns `sim_order_id`.

**Target state:** Full Kite Connect integration.

**File:** `kite_broker_engine.py`

| Component | Implementation |
|-----------|---------------|
| Authentication | `kiteconnect.KiteConnect(api_key)` + login flow with request_token |
| Session management | Store `access_token` in `live_state.db`, auto-refresh before expiry |
| Historical data | `kite.historical_data(instrument_token, from_date, to_date, "day")` |
| Live quotes | `kite.quote(instrument_tokens)` — batch quote for all stocks |
| Order placement | `kite.place_order(...)` — for future automated trading |
| Instrument lookup | `kite.instruments("NSE")` — get instrument_token for each symbol |

**Authentication flow:**

1. Set `KITE_API_KEY` in `.env`
2. First run: open browser for login, get `request_token` from redirect URL
3. Exchange `request_token` for `access_token` via `kite.request_access_token()`
4. Store `access_token` + `refresh_token` in `live_state.db`
5. Subsequent runs: load from DB, check expiry, refresh if needed

### Step 8.2: Implement live_market_streamer.py

**Current state:** STUB — `on_ticks()` has `pass`, falls back to `[SIMULATED]`.

**Target state:** Real-time tick streaming via Kite WebSocket.

**File:** `live_market_streamer.py`

| Component | Implementation |
|-----------|---------------|
| WebSocket connection | `kiteconnect.ticker.KiteTicker(api_key, access_token)` |
| Subscribe | `ticker.subscribe([instrument_tokens])` |
| on_ticks callback | Parse tick data, update in-memory OHLCV cache |
| on_connect | Re-subscribe after reconnection |
| Parquet writer | Every 5 minutes during market hours, flush tick data to daily parquet |

**Tick data processing:**

```python
def on_ticks(self, ws, ticks):
    for tick in ticks:
        symbol = self._token_to_symbol[tick["instrument_token"]]
        # Update in-memory OHLCV for current candle
        self._current_candle[symbol] = {
            "Open": tick["ohlc"]["open"],
            "High": max(self._current_candle[symbol]["High"], tick["last_price"]),
            "Low": min(self._current_candle[symbol]["Low"], tick["last_price"]),
            "Close": tick["last_price"],
            "Volume": tick["volume"],
        }
```

### Step 8.3: Data Pipeline Integration

**Switch data source:**

```bash
# In .env, change:
DATA_SOURCE=kite_live
KITE_API_KEY=your_api_key
KITE_API_SECRET=your_api_secret
```

**Pipeline flow:**

```
9:00 AM  →  Kite Streamer starts, subscribes to 1,400 instruments
9:15 AM  →  Market open. Ticks flow in, in-memory candles build
Every 5m →  Flush incomplete candles to {SYMBOL}_intraday.parquet
3:30 PM  →  Market close. Flush final candles, trigger end-of-day:
             1. bridge_daily_update.py runs (GSheet still for fundamentals)
             2. signals_service cache invalidated
             3. POST /api/signals/sync triggered
             4. Full re-score on completed daily candles
```

### Step 8.4: Historical Data Backfill

For the initial setup, backfill historical data from Kite:

```python
# Backfill last 2 years of daily data for all stocks
def backfill_historical(kite, symbols, days=500):
    for symbol in symbols:
        token = get_instrument_token(kite, symbol)
        from_date = datetime.now() - timedelta(days=days)
        data = kite.historical_data(token, from_date, datetime.now(), "day")
        df = pd.DataFrame(data)
        df.to_parquet(repo_path(symbol))
```

**Phase 8 Exit Criteria:** Live ticks stream during market hours. Parquet files update with daily data. `DATA_SOURCE=kite_live` works. `GET /api/signals` scores based on Kite data.

---


## 15. Phase 9: WebSocket Real-Time Layer

**Duration:** 2-3 days  **Depends on:** Phase 1  **Blocks:** Real-time alerts, live updates

### Step 9.1: WebSocket Server Setup

**File:** `apollo_api/websocket_manager.py`

```python
from fastapi import WebSocket, WebSocketDisconnect
from typing import List
import json
import asyncio

class ConnectionManager:
    def __init__(self):
        self.active_connections: List[WebSocket] = []

    async def connect(self, websocket: WebSocket):
        await websocket.accept()
        self.active_connections.append(websocket)

    def disconnect(self, websocket: WebSocket):
        self.active_connections.remove(websocket)

    async def broadcast(self, message: dict):
        for conn in self.active_connections[:]:
            try:
                await conn.send_json(message)
            except:
                self.disconnect(conn)

    async def send_personal(self, websocket: WebSocket, message: dict):
        await websocket.send_json(message)

manager = ConnectionManager()
```

**Route registration:**

```python
# In main.py or a separate routes/websocket.py:
@router.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await manager.connect(websocket)
    try:
        while True:
            # Client can send ping/subscribe messages
            data = await websocket.receive_text()
            # Handle client messages (if needed)
    except WebSocketDisconnect:
        manager.disconnect(websocket)
```

### Step 9.2: Four Message Types

| # | Type | Trigger | Payload | Consumed By |
|---|------|---------|---------|-------------|
| 1 | `LIVE_PULSE` | Every 60s during market hours | `{type: "LIVE_PULSE", nifty: 22450, banknifty: 48200, total_scored: 1400, timestamp: "..."}` | MarketRegimeBanner, TickerFooter |
| 2 | `NEW_ALERT` | Alert detection (30s cycle) | `{type: "NEW_ALERT", alerts: AlertItem[]}` | Alerts tab, browser notification |
| 3 | `CACHE_REFRESH` | POST /api/signals/sync completes | `{type: "CACHE_REFRESH", summary: SignalsSummary}` | App.tsx triggers refetch |
| 4 | `STATE_DIFF` | Score change detected for any stock | `{type: "STATE_DIFF", changes: [{symbol, field, old, new}]}` | Screener tab live updates |

### Step 9.3: React WebSocket Client

The React `App.tsx` already has WebSocket code. It just needs the URL changed:

```typescript
// BEFORE:
const ws = new WebSocket('ws://localhost:3000');

// AFTER:
const ws = new WebSocket('ws://localhost:8000/ws');
```

The message handling in React already handles these 4 types. No component changes needed.

### Step 9.4: Market Hours Detection

```python
from datetime import datetime
import pytz

def is_market_hours() -> bool:
    ist = pytz.timezone('Asia/Kolkata')
    now = datetime.now(ist)
    # Market: Mon-Fri, 9:15 AM - 3:30 PM IST
    if now.weekday() >= 5:
        return False
    market_open = now.replace(hour=9, minute=15, second=0, microsecond=0)
    market_close = now.replace(hour=15, minute=30, second=0, microsecond=0)
    return market_open <= now <= market_close
```

**Phase 9 Exit Criteria:** React connects to `/ws`. `LIVE_PULSE` messages arrive every 60s. `NEW_ALERT` messages push in real-time. `CACHE_REFRESH` triggers data refetch.

---


## 16. Phase 10: Persistence Migration

**Duration:** 2 days  **Depends on:** Phase 6 (journal/alerts services)  **Blocks:** Full production readiness

### Step 10.1: Current State — sql.js (Client-Side SQLite in WASM)

The React app uses `sql.js` (SQLite compiled to WebAssembly) running in the browser. This means:

- Data lives in browser memory/localStorage — lost on clear
- No sharing between users/devices
- No server-side queries possible
- Demo data is seeded on first load

### Step 10.2: Target State — aiosqlite (Server-Side)

All 11 tables migrate from browser-side `sql.js` to server-side `aiosqlite`.

**File:** `apollo_api/db.py` (NEW)

```python
import aiosqlite
import os
from pathlib import Path

DB_PATH = Path(os.environ.get("APOLLO_DB_PATH", "data/apollo_dashboard.db"))

class Database:
    def __init__(self):
        self._conn = None

    async def connect(self):
        self._conn = await aiosqlite.connect(DB_PATH)
        self._conn.row_factory = aiosqlite.Row
        await self._create_tables()

    async def close(self):
        if self._conn:
            await self._conn.close()

    async def _create_tables(self):
        await self._conn.executescript(SCHEMA_SQL)
        await self._conn.commit()

    async def execute(self, sql, params=()):
        cursor = await self._conn.execute(sql, params)
        await self._conn.commit()
        return cursor

    async def fetchall(self, sql, params=()):
        cursor = await self._conn.execute(sql, params)
        return await cursor.fetchall()

    async def fetchone(self, sql, params=()):
        cursor = await self._conn.execute(sql, params)
        return await cursor.fetchone()

db = Database()
```

### Step 10.3: Schema Definition

```sql
-- SCHEMA_SQL constant
CREATE TABLE IF NOT EXISTS trades (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    symbol TEXT NOT NULL,
    entry_date TEXT NOT NULL,
    exit_date TEXT,
    entry_price REAL NOT NULL,
    exit_price REAL,
    pnl_pct REAL,
    holding_days INTEGER,
    exit_mode TEXT
);

CREATE TABLE IF NOT EXISTS journal_entries (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    symbol TEXT NOT NULL,
    date TEXT NOT NULL,
    note TEXT,
    target REAL,
    stop_loss REAL,
    created_at TEXT DEFAULT (datetime('now'))
);

CREATE TABLE IF NOT EXISTS alert_rules (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL,
    condition TEXT NOT NULL,
    channel TEXT DEFAULT 'browser',
    enabled INTEGER DEFAULT 1
);

CREATE TABLE IF NOT EXISTS alert_logs (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    source TEXT,
    type TEXT,
    timestamp TEXT DEFAULT (datetime('now')),
    symbol TEXT,
    title TEXT,
    message TEXT,
    read INTEGER DEFAULT 0
);

CREATE TABLE IF NOT EXISTS system_logs (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    time TEXT DEFAULT (datetime('now')),
    level TEXT,
    message TEXT
);

CREATE TABLE IF NOT EXISTS ipo_stocks (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    symbol TEXT UNIQUE NOT NULL,
    company_name TEXT,
    issue_price REAL,
    listing_date TEXT,
    listing_price REAL,
    all_time_high REAL,
    all_time_low REAL,
    ipo_size TEXT,
    sector TEXT,
    exchange TEXT,
    promoter_stake REAL,
    current_pe REAL,
    cmp REAL,
    zone TEXT,
    ipo_baseline REAL,
    listing_stage TEXT,
    days_since_listing INTEGER,
    distance_to_baseline_pct REAL,
    distance_to_ath_pct REAL,
    return_from_issue_pct REAL,
    baseline_ratio REAL,
    ath_recovery_pct REAL,
    Apollo_Score REAL,
    LayerSignal_Score REAL,
    Apollo_Action TEXT,
    Quality TEXT,
    Bucket TEXT,
    Gates TEXT,
    RSI21 REAL,
    ADX REAL,
    ATR_Pct REAL
);

CREATE TABLE IF NOT EXISTS ipo_zone_history (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    symbol TEXT NOT NULL,
    old_zone TEXT,
    new_zone TEXT,
    transition_type TEXT,
    cmp_at_transition REAL,
    timestamp TEXT DEFAULT (datetime('now'))
);

CREATE TABLE IF NOT EXISTS ipo_archive (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    symbol TEXT UNIQUE NOT NULL,
    graduated_at TEXT DEFAULT (datetime('now')),
    -- same fields as ipo_stocks
    company_name TEXT, issue_price REAL, listing_date TEXT, listing_price REAL,
    all_time_high REAL, all_time_low REAL, cmp REAL, zone TEXT, sector TEXT
);

CREATE TABLE IF NOT EXISTS daily_highs_snapshot (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    date TEXT NOT NULL,
    symbol TEXT NOT NULL,
    high_52w REAL,
    cmp REAL,
    proximity_pct REAL,
    proximity_tier TEXT,
    apollo_score REAL
);

CREATE TABLE IF NOT EXISTS stock_ath (
    symbol TEXT PRIMARY KEY,
    all_time_high REAL,
    ath_date TEXT,
    ath_cmp_pct REAL
);

CREATE TABLE IF NOT EXISTS ml_model_metadata (
    model_name TEXT PRIMARY KEY,
    train_auc REAL,
    test_auc REAL,
    feature_count INTEGER,
    calibration_status TEXT
);

CREATE TABLE IF NOT EXISTS ml_predictions_history (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    date TEXT NOT NULL,
    symbol TEXT NOT NULL,
    ml_confidence REAL,
    confidence_tier TEXT,
    forward_20d_actual REAL
);
```

### Step 10.4: Data Migration

The existing `sql.js` database has demo/seeded data. For production, this data is not critical (it's demo). For a smooth transition:

1. Export from browser SQLite: Use the browser DevTools to export the database, or simply re-seed from the Python side.
2. The IPO seeded data (22 records) lives in `ipo_listing_dates.json` and `my_watchlist.json` — these are already server-side files.
3. Trade and journal demo data can be re-seeded from the TypeScript seed values in `server/db.ts`.

### Step 10.5: App Lifecycle

```python
# main.py
@app.on_event("startup")
async def startup():
    await db.connect()  # Initialize aiosqlite
    await signals_service.initialize()  # Warm cache

@app.on_event("shutdown")
async def shutdown():
    await db.close()
```

**Phase 10 Exit Criteria:** All CRUD operations use aiosqlite. Data persists across server restarts. React no longer uses sql.js.

---


## 17. Phase 11: Testing & Deployment

**Duration:** 3-4 days  **Depends on:** All phases

### Step 11.1: Unit Tests

| Test File | Tests | What It Verifies |
|-----------|-------|------------------|
| `tests/test_path_resolver.py` | 3 | APOLLO_DATA_ROOT resolution, repo_path(), data_path() |
| `tests/test_g5_gate.py` | 5 | RSI stack alignment: all pass, all fail, partial pass, boundary |
| `tests/test_fqs.py` | 5 | Score-to-grade mapping: 110+, 85-109, 60-84, <60, boundary |
| `tests/test_mcap.py` | 4 | Large/Mid/Small classification at boundaries |
| `tests/test_ipo_classifier.py` | 8 | 4 zones x 2 boundary cases |
| `tests/test_proximity_tier.py` | 6 | AT/NEAR/APPROACHING/EXTENDED/FAR + boundary at 99% |
| `tests/test_exhaustion.py` | 5 | NONE/MILD/MODERATE/STRONG severity |
| `tests/test_ml_service.py` | 8 | predict, predict_all, explain, train, feature_importance |
| `tests/test_alerts.py` | 6 | 5 transition types + rule evaluation |

### Step 11.2: Integration Tests

| Test File | Tests | What It Verifies |
|-----------|-------|------------------|
| `tests/test_api_signals.py` | 5 | /api/signals returns valid JSON, all 49 fields, summary counts, cache, sync |
| `tests/test_api_highs.py` | 5 | /api/highs/* 5 endpoints return valid JSON |
| `tests/test_api_ml.py` | 4 | /api/ml/* 4 endpoints return valid JSON |
| `tests/test_api_ipo.py` | 8 | /api/ipo/* 8 endpoints return valid JSON |
| `tests/test_api_alerts.py` | 8 | /api/db/alerts + /api/db/rules CRUD |
| `tests/test_api_db.py` | 4 | /api/db/trades + /api/db/journal CRUD |
| `tests/test_websocket.py` | 3 | Connect, receive LIVE_PULSE, NEW_ALERT broadcast |

### Step 11.3: Contract Tests — Data Integrity

These tests verify the Python JSON output matches the TypeScript interface exactly:

```python
# tests/test_contract.py
import requests

RESPONSE = requests.get("http://localhost:8000/api/signals").json()
STOCK = RESPONSE["stocks"][0]
EXPECTED_FIELDS = [
    "Symbol", "Date", "Apollo_Action", "Apollo_Score", "Pct_Change",
    "LayerSignal_Action", "LayerSignal_Score", "Exit_Pressure",
    "Open", "High", "Low", "Close", "Volume",
    "High52W", "Low52W", "RSI21", "RSI36", "RSI56",
    "ADX", "ATR_Pct", "Stochastic", "PE",
    "SMA_20D", "SMA_50D", "SMA_200D",
    "Prox_52W", "CMP", "Traded_Value",
    "Bucket", "LStage", "ELStatus", "Conviction",
    "Gates", "Renko", "MCap", "FQS",
    "Sparkline", "ThrowbackAlert", "ml_confidence",
    "SubScores", "GatesExplanations", "HistoricalL3Events", "isIPO",
]

def test_all_49_fields_present():
    for field in EXPECTED_FIELDS:
        assert field in STOCK, f"Missing field: {field}"

def test_score_range():
    assert 0 <= STOCK["Apollo_Score"] <= 148

def test_ls_score_range():
    assert 20 <= STOCK["LayerSignal_Score"] <= 100

def test_gates_count():
    assert len(STOCK["Gates"]) == 5
    assert all(isinstance(g, bool) for g in STOCK["Gates"])

def test_action_valid():
    assert STOCK["Apollo_Action"] in ("HOLD", "ENTRY", "EXIT", "FLAT")
    assert STOCK["LayerSignal_Action"] in ("HOLD", "ENTRY", "EXIT", "FLAT")
```

### Step 11.4: End-to-End Test

```bash
# 1. Start Python backend
uvicorn apollo_api.main:app --host 0.0.0.0 --port 8000 &

# 2. Start React frontend (in separate terminal)
cd frontend && npm run dev &

# 3. Run E2E test (Playwright or manual checklist)
# - Open http://localhost:5173
# - Verify Screener tab shows real scores
# - Click IPO tab, verify zone data
# - Click New Highs, verify proximity tiers
# - Create a trade in Watchlist, refresh, verify it persists
# - Create an alert rule, verify it triggers
# - Open browser console, verify no errors
# - Check WebSocket connection established
```

### Step 11.5: Docker Deployment

**File:** `Dockerfile`

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 8000
CMD ["uvicorn", "apollo_api.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**File:** `docker-compose.yml`

```yaml
version: '3.8'
services:
  backend:
    build: ./python-backend
    ports:
      - "8000:8000"
    environment:
      - APOLLO_DATA_ROOT=/app/data
      - DATA_SOURCE=parquet
      - SIGNALS_CACHE_TTL=300
    volumes:
      - ./data:/app/data
    restart: unless-stopped

  frontend:
    build: ./frontend
    ports:
      - "5173:80"  # Nginx serving built React
    depends_on:
      - backend
    environment:
      - VITE_API_BASE=http://backend:8000
```

**Phase 11 Exit Criteria:** All unit tests pass. All integration tests pass. Contract test verifies 49/49 fields. E2E test covers all 10 tabs. Docker containers start and serve the full stack.

---
## 18. React Kill List — Files to Delete

These files are deleted during Phase 7. They contain fake scoring, fake ML, or Express server code fully replaced by Python engines.

| # | File | Lines | Reason | Replacement |
|---|------|-------|--------|-------------|
| 1 | `server.ts` | 1,284 | Express server, GSheet fetch, fake scoring, WebSocket, alert detection | Python FastAPI + signals_service + alerts_service |
| 2 | `server/db.ts` | ~300 | SQLite schema (11 tables), seeding, save/load in sql.js | Python aiosqlite (Phase 10) |
| 3 | `server/highs-service.ts` | 734 | Fake 52WH enrichment — 12 functions | Python highs_service.py (Phase 2) |
| 4 | `server/highs-routes.ts` | ~350 | 5 Express endpoints for 52WH | Python routes/highs.py (Phase 2) |
| 5 | `server/ml-service.ts` | 651 | Fake ML — logistic sigmoid, 9 weighted factors | Python ml_service.py with real XGBoost (Phase 3) |
| 6 | `server/ml-routes.ts` | ~200 | 4 Express endpoints for ML | Python routes/ml.py (Phase 3) |
| 7 | `server/ipo-routes.ts` | ~400 | 8 Express endpoints + embedded scraper/classifier/graduator | Python routes/ipo.py + ipo_service.py (Phase 4) |
| 8 | `server/ipo-classifier.ts` | ~200 | 4-zone IPO classification logic | Ported to Python ipo_service.py (Phase 4) |
| 9 | `server/ipo-scraper.ts` | ~250 | Chittorgarh.com scraper + 22 seeded records | Ported to Python httpx+BS4 (Phase 4) |
| 10 | `server/ipo-graduator.ts` | ~100 | IPO graduation/archival logic | Ported to Python ipo_service.py (Phase 4) |
| 11 | `src/utils/enrichment.ts` | 294 | Client-side fake scorer — hashCode()-based random | Deleted — no replacement needed |
| **TOTAL** | **~4,763 lines** | | |

**Files NOT deleted (kept as-is):**

| File | Lines | Why Kept |
|------|-------|----------|
| `src/types.ts` | 496 | Immutable data contract |
| `src/App.tsx` | ~400 | Only API base URL change |
| `src/utils/calculations.ts` | ~100 | Formatting helpers |
| `src/utils/notifications.ts` | ~100 | Browser push + Web Audio API |
| `src/components/tabs/*.tsx` | ~67KB | All 10 tab components |
| `src/components/highs/*.tsx` | 11 files | New Highs sub-components |
| `src/components/ml/*.tsx` | 8 files | ML sub-components |
| `src/components/shell/*.tsx` | 4 files | AppBar, TabBar, MarketRegimeBanner, TickerFooter |

---

## 19. Python File Cleanup

### 19.1 Stub Files — Need Full Implementation

| File | Status | Action | Phase |
|------|--------|--------|-------|
| `kite_broker_engine.py` | STUB — `place_order()` commented out | Full implementation | Phase 8 |
| `live_market_streamer.py` | STUB — `on_ticks()` has `pass` | Full implementation | Phase 8 |
| `vectorized_scoring.py` | PARTIAL — only Pool A vectorized | Complete or delete | Post-MVP |
| `data_repo/sources.py` | EMPTY — placeholder | Remove or implement | Phase 8 |

### 19.2 Files That Stay Unchanged (Core Engines)

These files are the production engines. Do NOT modify their internal logic. They are imported by the service layer.

| Module | Files | Status |
|--------|-------|--------|
| Apollo Scoring | `scoring.py`, `constants.py` | Production — keep as-is |
| Indicators | `indicators.py` | Production — keep as-is |
| Gates | `gates.py` | Production — keep as-is |
| Bucket Classifier | `bucket_classifier.py` | Production — keep as-is |
| LayerSignal | `layersignal_engine/core.py` | Production — keep as-is |
| Renko | `renko.py` | Production — keep as-is |
| Trade Engine | `trade_engine.py` | Production — keep as-is |
| Guidance Engine | `guidance_engine/` (8 files) | Production — keep as-is |
| Feature Store | `precompute_features.py`, `historical_batch_replay.py`, `signal_profiler.py` | Production — keep as-is |
| Data Pipeline | `bridge_daily_update.py`, `signal_export.py` | Production — fix bug in export, keep rest |
| Data Repo | `data_repo/repo.py` | Production — fix paths, keep logic |

### 19.3 New Files Created (Summary)

| File | Phase | Lines (est) |
|------|-------|-------------|
| `apollo_core/path_resolver.py` | 0 | ~20 |
| `apollo_api/models/signals.py` | 1 | ~80 |
| `apollo_api/services/signals_service.py` | 1 | ~350 |
| `apollo_api/routes/signals.py` | 1 | ~30 |
| `apollo_api/models/highs.py` | 2 | ~80 |
| `apollo_api/services/highs_service.py` | 2 | ~700 |
| `apollo_api/routes/highs.py` | 2 | ~60 |
| `apollo_api/services/ml_service.py` | 3 | ~400 |
| `apollo_api/routes/ml.py` | 3 | ~80 |
| `apollo_api/services/ipo_service.py` | 4 | ~500 |
| `apollo_api/routes/ipo.py` | 4 | ~120 |
| `apollo_api/services/alerts_service.py` | 5 | ~400 |
| `apollo_api/routes/alerts.py` | 5 | ~100 |
| `apollo_api/services/journal_service.py` | 6 | ~150 |
| `apollo_api/services/analytics_service.py` | 6 | ~100 |
| `apollo_api/routes/db.py` | 6 | ~80 |
| `apollo_api/routes/system.py` | 6 | ~60 |
| `apollo_api/websocket_manager.py` | 9 | ~60 |
| `apollo_api/db.py` | 10 | ~100 |
| **TOTAL** | | **~3,450 lines** |

---

## 20. Risk Register

| # | Risk | Probability | Impact | Mitigation |
|---|------|------------|--------|------------|
| R1 | `score_stock()` return structure differs from assumptions | High | High | Read `scoring.py` completely before writing signals_service. Write adapter if needed. |
| R2 | `indicators.py` function signatures differ from expected | Medium | High | Read `indicators.py` completely. Write `compute_all_indicators()` wrapper. |
| R3 | LayerSignal `calc_momentum_score()` returns dict, not float | Medium | Medium | Check return type. Write adapter. |
| R4 | 1,400 stocks take > 60s to score (performance) | Medium | Medium | Profile with `cProfile`. Optimize hot paths. Consider multiprocessing. |
| R5 | XGBoost AUC < 0.55 (no edge) | Medium | Low | System works without ML. ML is enhancement, not dependency. |
| R6 | Kite Connect rate limits during backfill | Medium | Low | Implement rate limiter. Use historical_data API with delays. |
| R7 | Chittorgarh.com scraper breaks (HTML change) | Medium | Low | Keep 22 seeded records as fallback. Scraper is non-critical. |
| R8 | React WebSocket reconnection handling | Low | Medium | React already has reconnect logic. Test thoroughly. |
| R9 | aiosqlite concurrent access issues | Low | Medium | Single writer pattern. All writes through `db.execute()`. |
| R10 | TypeScript interface field name mismatch | Low | High | Run contract test (Section 11.3) before any frontend testing. |
| R11 | Parquet files missing for some stocks | Medium | Medium | Graceful skip in signals_service. Log warnings. Return partial results. |
| R12 | GSheet CSV format changes | Low | Medium | Parse defensively. Use column name matching, not column index. |
| R13 | Memory usage scoring 1,400 stocks (each loads parquet) | Medium | Medium | Load and score one at a time. Free DataFrame after scoring. |
| R14 | Docker volume permissions for data/ | Low | Low | Use proper user mapping in Dockerfile. |
| R15 | Browser notifications blocked by default | Low | Low | Already handled in React. User must allow notifications. |

---

## 21. Timeline

### Week-by-Week Breakdown (6-8 weeks)

| Week | Phase | Days | Deliverables |
|------|-------|------|-------------|
| **Week 1** | Phase 0 | 1 | Path fixes, requirements.txt, bug fixes, directory structure |
| | Phase 1 (P0) | 4 | signals_service.py, Pydantic models, GET /api/signals, 49-field contract verified |
| **Week 2** | Phase 2 (P1) | 4 | highs_service.py (12 functions), 5 /api/highs/* endpoints |
| | Phase 3 start (P1) | 1 | ML service skeleton, feature store understanding |
| **Week 3** | Phase 3 (P1) | 4 | XGBoost model, SHAP, 4 /api/ml/* endpoints |
| | Phase 4 start (P2) | 1 | IPO scraper port |
| **Week 4** | Phase 4 (P2) | 2 | IPO zone classifier, graduator, 8 /api/ipo/* endpoints |
| | Phase 5 (P2) | 3 | alerts_service.py, state-change detection, rule engine, 8 /api/db/* endpoints |
| **Week 5** | Phase 6 (P3) | 2 | journal_service, analytics_service, system health |
| | Phase 7 | 2 | Delete Express files, change API URL, remove sql.js, tab verification |
| | Phase 10 | 1 | aiosqlite migration, schema, seed data |
| **Week 6** | Phase 9 | 2 | WebSocket server, 4 message types |
| | Phase 11 | 3 | Unit tests, integration tests, contract tests |
| **Week 7** | Phase 8 (parallel) | 5 | Kite Connect auth, live streamer, historical backfill, DATA_SOURCE=kite_live |
| **Week 8** | Phase 11 cont. | 3 | E2E testing, Docker, deployment |
| | Buffer | 2 | Fix issues, polish |

### Critical Path

```
Phase 0 (1d) → Phase 1 (4d) → Phase 2 (4d) → Phase 5 (3d) → Phase 7 (2d) → Phase 11 (3d)
                  ↓              ↓
              Phase 3 (5d)   Phase 4 (3d)

Parallel track:
Phase 8 (Kite) — can start after Phase 0, runs in parallel
Phase 9 (WebSocket) — can start after Phase 1
Phase 10 (DB) — can start after Phase 6
```

### Milestones

| Milestone | Date | Criteria |
|-----------|------|----------|
| M1: Core Scoring Live | End Week 1 | GET /api/signals returns 1,400 stocks with real 49-field data |
| M2: 52WH + ML Live | End Week 3 | New Highs and ML tabs show real data |
| M3: Full API Coverage | End Week 5 | All 47 endpoints (25 existing + 22 new) return valid JSON |
| M4: Express Eliminated | End Week 5 | React runs on Vite, all data from Python |
| M5: Real-Time | End Week 6 | WebSocket connected, alerts push live |
| M6: Production | End Week 8 | Docker deployed, Kite live data, all tests passing |
