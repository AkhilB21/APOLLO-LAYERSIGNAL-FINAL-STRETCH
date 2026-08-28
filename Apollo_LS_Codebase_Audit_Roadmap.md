# Apollo-LS Codebase Audit Roadmap

**VPS/Cloud Kernel Extraction and Integration Guide**  
For Implementor Use  
Source: APOLLO_BT.V1 Codebase  
Date: August 2026  
Confidential v1.0

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Codebase Inventory](#2-codebase-inventory)
   - 2.1 [Files to KEEP (Runtime Kernel)](#21-files-to-keep-runtime-kernel)
   - 2.2 [Modules to REMOVE (VPS-Redundant)](#22-modules-to-remove-vps-redundant)
3. [Engine Return Structure Reference](#3-engine-return-structure-reference)
   - 3.1 [scoring.py: eval_signals()](#31-scoringpy-eval_signals)
   - 3.2 [scoring.py: classify_score()](#32-scoringpy-classify_score)
   - 3.3 [layersignal_engine/core.py: calc_momentum_score()](#33-layersignal_enginecorepy-calc_momentum_score)
   - 3.4 [layersignal_engine/core.py: compute_exit_layers()](#34-layersignal_enginecorepy-compute_exit_layers)
   - 3.5 [gates.py: evaluate_gate_row()](#35-gatespy-evaluate_gate_row)
   - 3.6 [bucket_classifier.py: classify_stock_bucket()](#36-bucket_classifierpy-classify_stock_bucket)
   - 3.7 [renko.py: eval_renko_signals()](#37-renkopy-eval_renko_signals)
   - 3.8 [indicators.py: compute_all_indicators()](#38-indicatorspy-compute_all_indicators)
4. [Critical Fixes: Type Consistency and Naming](#4-critical-fixes-type-consistency-and-naming)
   - 4.1 [int/Float Inconsistency](#41-intfloat-inconsistency)
   - 4.2 [Naming Convention Chaos](#42-naming-convention-chaos)
   - 4.3 [Additional Naming Inconsistencies](#43-additional-naming-inconsistencies)
5. [PE (Price-to-Earnings) Omission Plan](#5-pe-price-to-earnings-omission-plan)
   - 5.1 [Where PE Appears in the React Frontend](#51-where-pe-appears-in-the-react-frontend)
   - 5.2 [Where PE Appears in the Python Backend](#52-where-pe-appears-in-the-python-backend)
   - 5.3 [Changes Required](#53-changes-required)
6. [Step-by-Step Extraction Procedure](#6-step-by-step-extraction-procedure)
   - Step 1: [Create VPS Project Structure](#step-1-create-vps-project-structure)
   - Step 2: [Copy the 8 Kernel Files](#step-2-copy-the-8-kernel-files)
   - Step 3: [Create COLUMN_MAP in signals_service.py](#step-3-create-column_map-in-signals_servicepy)
   - Step 4: [Apply float() Wrappers at Boundary](#step-4-apply-float-wrappers-at-boundary)
   - Step 5: [Handle PE Omission](#step-5-handle-pe-omission)
   - Step 6: [Fix Import Paths](#step-6-fix-import-paths)
   - Step 7: [Smoke Test](#step-7-smoke-test)
7. [Data Flow Diagram](#7-data-flow-diagram)
8. [React Frontend Contract: SignalStock Interface](#8-react-frontend-contract-signalstock-interface)
9. [Verification Checklist](#9-verification-checklist)

---

## 1. Executive Summary

This document provides a comprehensive, step-by-step audit roadmap for extracting the Apollo-LayerSignal computation kernel from the existing APOLLO_BT.V1 codebase and preparing it for deployment on a VPS/Cloud environment that serves the React Dashboard. The original codebase contains approximately 25,800 lines of Python code across 15+ modules, but only about 3,200 lines (eight files) constitute the runtime kernel required to power the real-time dashboard. Everything else belongs to desktop-only workflows such as backtesting, broker integration, Google Sheet syncing, walk-forward validation, and data pipeline management.

The audit identifies three categories of issues that must be addressed before writing the Phase 1 signals_service.py orchestration layer. First, there is a systematic int/float type inconsistency where calc_momentum_score() and compute_exit_layers() return bare integers while the React frontend and scoring.py signals use floats. Second, there is pervasive naming convention chaos caused by ChartAlert export artifacts: indicator columns carry names like `"RSI rsi (21,C)"` while internal variables use clean names like `"rsi21"`. Third, the PE (Price-to-Earnings) field is referenced in the React types and three component files, but the backend has no fundamental data source for it. All three issues have safe, minimal-impact solutions that fix at the orchestration boundary without modifying the original engine code.

The recommended approach is to create a clean new project on the VPS, copy only the eight kernel files, apply a column alias map and float() wrappers in the signals_service.py orchestration layer, and validate with a smoke test on a single stock before proceeding to Phase 1 implementation. This 2-to-3-hour audit investment prevents a 2-to-3-week debugging nightmare later.

---

## 2. Codebase Inventory

The APOLLO_BT.V1 codebase contains Python modules organized into ten directories plus root-level scripts. The audit examined every file to classify it as either essential for the VPS dashboard (KEEP) or redundant (REMOVE). The classification criterion is simple: a file is KEEP if it contains pure computation logic that the signals_service.py orchestration layer must call at runtime to generate SignalStock data. Everything else, including backtesting, live trading, data pipeline, and desktop utilities, is REMOVE.

### 2.1 Files to KEEP (Runtime Kernel)

The following eight files form the complete runtime kernel. They total approximately 3,200 lines of pure computation code with no external dependencies beyond numpy and pandas. Each file is self-contained: it accepts DataFrame inputs, performs calculations, and returns results. There are no database calls, no network requests, no file I/O, and no broker integrations within these files.

| **File** | **Lines** | **Key Functions** | **Role in Dashboard** |
|---|---|---|---|
| apollo_core/scoring.py | 962 | eval_signals(), classify_score(), eval_divergence_signal(), eval_ipo_signals() | Evaluates 26 signals (A1-A12, B1-B6, C1-C7, D1, E1-E5, R1-R8 via renko.py), returns dict[str, tuple[float, bool]]. classify_score() maps total score to (action, position_pct). |
| apollo_core/indicators.py | 379 | compute_all_indicators(), compute_rsi(), compute_adx(), compute_macd(), compute_stochastic(), compute_vpt(), compute_sma(), compute_tp28(), compute_wc50() | Computes 17 technical indicators from raw OHLCV DataFrame. Returns enriched DataFrame with all Apollo-named indicator columns. |
| apollo_core/gates.py | 136 | evaluate_gate_row(), compute_gate_frame() | Evaluates 4 gates (G1 PE, G2 Liquidity, G3 52W-Low Proximity, G4 52W-High Distance). Returns dict with g1-g4 status + values + all_pass boolean. Informational only (never blocks entry). |
| apollo_core/bucket_classifier.py | 379 | classify_stock_bucket(), classify_bucket(), compute_bucket_metrics() | Classifies stock into 8 structural buckets (4 Trending Up tiers, Recovery, Sideways, Decline, IPO). Returns (bucket_str, display_metrics_dict). |
| apollo_core/renko.py | 498 | eval_renko_signals(), compute_renko_bricks(), compute_renko_rsi(), map_renko_to_daily() | Evaluates 8 Renko signals (R1-R8). Returns dict[str, tuple[float, bool]]. Also provides Renko brick computation and daily-bar-to-Renko index mapping. |
| layersignal_engine/core.py | 352 | calc_momentum_score(), compute_exit_layers(), evaluate_trade_signals() | calc_momentum_score() returns 0-100 integer momentum score (5 sub-components). compute_exit_layers() returns 0-100 integer exit pressure (4 layers). evaluate_trade_signals() returns (trades, metrics, state). |
| apollo_core/constants.py | ~150 | SIGNAL_DEFS, POOL_ORDER, POOL_MAX, POOL_NAMES, IPO_SIGNAL_DEFS, ENTRY_THRESHOLD, EXIT_THRESHOLD, RENKO_HARD_GATE, PE_MAX, etc. | All scoring constants, signal definitions, thresholds, and pool configurations. Referenced by scoring.py, gates.py, and bucket_classifier.py. |
| apollo_core/core_types.py | ~50 | Type definitions (if any) | Type aliases and dataclass definitions used across engine modules. Check for TypedDict or dataclass definitions. |

### 2.2 Modules to REMOVE (VPS-Redundant)

The following modules are desktop-only tools, data pipelines, or analysis utilities that have no role in serving the React Dashboard. They should NOT be copied to the VPS. The original codebase on your local machine remains the authoritative source for these files, available for any future product that may need backtesting, broker integration, or offline analysis capabilities.

| **Module/Directory** | **Lines** | **Reason for Removal** |
|---|---|---|
| backtest_engine/ | ~3,500 | Desktop-only backtesting tool (backtest.py, report_generator.py, app.py, eod2_loader.py, backtest_history.py). Not needed for real-time dashboard. |
| live_engine/ | ~3,800 | Old live engine being replaced by FastAPI + React. Contains signal_monitor.py, alert_manager.py, daily_report.py, dashboard.py, cli.py, state_store.py, watchlist.py, live_feed.py, data_replay.py, multi_monitor.py, run_headless.py, run_live.py. |
| guidance_engine/ | ~2,500 | Post-trade guidance analysis (analyzer.py, aggregator.py, flag_generator.py, mae_mfe.py, recorder.py, regime.py, schemas.py). Phase 9+ feature, not needed for initial VPS deployment. |
| nse_engine/ | ~1,200 | NSE-specific data pipeline (data_feed.py, scanner.py, run_pipeline.py). VPS will use a different data provider. |
| simulator/ | ~1,800 | Parameter sweep and walk-forward validation tools (walk_forward.py, regime_optimizer.py, indicator_sweep.py, mtf_sweep.py, alignment_dryrun.py, indicator_engine_test.py, regime_analysis.py). Desktop analysis tools. |
| data_repo/ | ~1,000 | Old data loading from parquet files and Excel (repo.py, sources.py, sync.py). VPS will use API-based data feeds. |
| cross_report_pipeline.py | 373 | Legacy watchlist builder using RSI trajectory + momentum vectors. Superseded by signals_service.py. |
| signal_profiler.py + profile_analytics.py | ~800 | Analysis tools for historical signal profiling. Desktop-only. |
| vectorized_scoring.py + threshold_calibrator.py | ~600 | Experimental vectorized scoring and threshold calibration. Not production code. |
| walk_forward_validator.py + historical_batch_replay.py | ~1,200 | Backtesting validation tools. Desktop-only. |
| trade_engine.py + kite_broker_engine.py | ~1,500 | Trade execution and Zerodha Kite broker integration. Not relevant for dashboard-only deployment. |
| live_market_streamer.py + telegram_notifier.py | ~800 | Real-time market data streaming and Telegram notifications. Will be replaced by API-based feeds. |
| bridge_sync.py + bridge_push.py + bridge_daily_update.py | ~400 | Old bridge scripts for syncing between desktop tools. No longer needed. |
| gsheet_repo.py + fundamental_repo.py + fundamentals.py | ~600 | Google Sheet and EtMoney fundamental data fetchers. No fundamental data available for VPS. |
| signal_export.py + to_excel.py + json_dumper.py | ~400 | Export utilities for CSV/Excel/JSON output. VPS serves data via FastAPI, not file exports. |
| fix_outcomes.py + outcome_backfiller.py + regime_prefilter.py | ~300 | One-off data fix scripts. No longer needed. |
| ipo_lookup.py | ~80 | IPO listing dates lookup. May be needed for IPO tab (Phase 4). Evaluate later. |
| db_builder.py + fetch_marketcap.py + convert_mcap.py + map_mcap.py | ~500 | Database and market cap data management. VPS will use API-based market cap data. |
| build_universe.py + merge_universe.py + run_universe.py + update_universe.py | ~400 | Universe management scripts. VPS universe will be managed via API. |
| aux_data_sync.py + nifty_vix_downloader.py | ~200 | Auxiliary data downloads. VPS will use API-based data. |
| All .bat, .vbs, .xlsx, .csv files | ~500 | Windows batch scripts, VB scripts, and generated output files. Not applicable to Linux VPS. |
| adhoc_scripts/ | ~600 | One-off patch scripts (patch.py, patch2-4.py, sandbox_layersignal_bt.py, etc.). Historical debugging artifacts. |
| apollo_ui/ (old Vite dashboard) | ~800 | Old React dashboard (ScreenerTab, LayerSignalTheme, DeepDiveModal, etc.). Being replaced by the new Next.js React frontend. |
| check_*.py scripts | ~400 | Debugging scripts (check_head.py, check_cols.py, check_caps.py, check1.py, check_transrail.py, check_finalized.py). Development artifacts. |
| data/ directories, .db files, .json files | ~2,000 | Local data caches (parquet dir references, apollo_database.db, apollo_universe.json, watchlist JSONs, market cap JSONs). VPS will build its own data. |

**Total lines to REMOVE:** approximately 22,600. **Total lines to KEEP:** approximately 3,200. The KEEP ratio is roughly 12.4% of the total codebase, confirming that the vast majority of the codebase is desktop-tooling irrelevant to the VPS dashboard.

---

## 3. Engine Return Structure Reference

This section documents the exact return types of every function that signals_service.py will call. The implementor must understand these structures precisely because the orchestration layer must transform each return value into the fields expected by the React SignalStock interface (49 fields). Each function signature, parameter list, and return type has been verified against the actual source code.

### 3.1 scoring.py: eval_signals()

The eval_signals() function is the heart of the Apollo scoring system. It evaluates 21 candlestick-based signals across 5 pools (A through E) plus 5 divergence/volatility pool D signals and 1 hidden bullish divergence D1 signal, for a total of 26 signals. Each signal returns a (points: float, fired: bool) tuple. The function does NOT aggregate or sum the points; that is the responsibility of the caller (signals_service.py).

**Function Signature:**

```python
def eval_signals(df_d: dict, df_4h: dict, df_w: dict, d_dates, h_dates,
                 d_idx: int, h_idx: int, w_idx: int, d_trough_pos, h_trough_pos,
                 d_rsi21, d_rsi36, h_rsi21, h_rsi36, w_rsi21, w_rsi56,
                 plus_di, minus_di, macd_line, crossover_price) -> dict:
```

**Return Type:** `dict[str, tuple[float, bool]]`

Returns a dictionary mapping signal IDs to (points, fired) tuples. Example keys include A1 through A4 (Pool A: Trough Detection), B1 through B6 (Pool B: Recovery Signals), C1 through C7 (Pool C: Trend + Momentum), D1 (Hidden Bullish Divergence), E1 through E5 (Pool E: Screening Filters). Points are float values (e.g., 10.0, 8.0, 5.0, 3.5) and fired is a boolean indicating whether the signal triggered. The caller must sum all .fired points and add Renko R-pool points separately.

### 3.2 scoring.py: classify_score()

**Function Signature:**

```python
def classify_score(score: float, r_points: float | None = None) -> tuple[str, str]:
```

**Return Type:** `tuple[str, str]`

Returns a 2-tuple of (action_label, position_pct_string). The action_label is one of: "FULL STRONG", "FULL ENTRY", "STANDARD", "WATCHLIST", "DO NOT ENTER", or "RENKO GATED". The position_pct_string is one of: "100%", "60-70%", or "0%". When r_points is provided and below RENKO_HARD_GATE, a BUY-level classification is downgraded to "RENKO GATED" with 0% position. The function does not modify the score; it only classifies it.

### 3.3 layersignal_engine/core.py: calc_momentum_score()

**Function Signature:**

```python
def calc_momentum_score(df, i) -> int:
```

**Return Type:** `int (0 to 100)`

> **CRITICAL:** This function returns a bare Python int, not a float. The docstring says "0-100 Momentum Score" but the implementation uses integer literals throughout (return 0, return 10, return 8, etc.) and arithmetic on integers. The five sub-components are: Price Momentum (max 25 pts), RSI Strength (max 25 pts), Volume Confirmation (max 20 pts), Multi-TF Approximation (max 20 pts), and Trend Quality (max 10 pts). Returns 0 when i < 30 or when primary RSI is NaN. The signals_service.py MUST wrap this in float() to match the React frontend expectation.

### 3.4 layersignal_engine/core.py: compute_exit_layers()

**Function Signature:**

```python
def compute_exit_layers(df, i) -> int:
```

**Return Type:** `int (0 to 100), EXPLICITLY cast via int()`

> **CRITICAL:** Line 261 explicitly returns `int(el1 + el2 + el3 + el4)`. The four exit layers are: EL1 - RSI drawdown from 90-bar peak (max 30 pts), EL2 - Weighted RSI average below 47 (max 25 pts), EL3 - Price below 20-day low (max 25 pts), and EL4 - Price below 2% stop-loss of 20-day low (max 20 pts). Returns 0 when i < 90 or when any RSI is NaN. The signals_service.py MUST wrap this in float() for consistency.

### 3.5 gates.py: evaluate_gate_row()

**Function Signature:**

```python
def evaluate_gate_row(row: pd.Series, pe: float | None = None) -> dict:
```

**Return Type:** `dict with 11 keys:`

| **Key** | **Type** | **Description** |
|---|---|---|
| g1_pe | float \| None | PE value (rounded to 1 decimal). None when pe parameter is None or NaN. |
| g1_status | str | "OK" or "FAIL: PE<=0" or "FAIL: PE>{PE_MAX}". |
| g2_avg_traded_value | float \| None | 20-day average traded value (close * volume), rounded to 0 decimals. |
| g2_status | str | "OK" or "LOW" (when below MIN_AVG_TRADED_VALUE). |
| g3_dist_from_52w_low_pct | float \| None | Distance from 52-week low as percentage. |
| g3_status | str | "OK" or "FAR" (when above MAX_52W_LOW_DIST * 100). |
| g4_dist_to_52w_high_x | float \| None | 52-week high divided by close (multiples). |
| g4_status | str | "OK" or "NEAR" (when below MIN_52W_HIGH_DIST_MULT). |
| all_pass | bool | True only if ALL evaluated gates have status "OK". |

**Important:** Gates NEVER block entry (v4.7 change). The docstring states: "Gates NEVER block entry --- this is for dashboard display only." G1 is skipped entirely when pe is None. The all_pass boolean is informational for the React frontend display.

### 3.6 bucket_classifier.py: classify_stock_bucket()

**Function Signature:**

```python
def classify_stock_bucket(df_d: pd.DataFrame, df_w: pd.DataFrame = None,
                          lookback_bars: int = 250, min_trading_days: int = 60) -> tuple[str, dict]:
```

**Return Type:** `tuple[str, dict[str, float]]`

Returns a 2-tuple where the first element is the bucket classification string and the second element is a display_metrics dictionary with 18 keys. The bucket string is one of 8 values: "Trending Up - Tier 1 (PRIME)", "Trending Up - Tier 2 (CONFIRMED)", "Trending Up - Tier 3 (EMERGING)", "Trending Up - Tier 4 (WATCH)", "Recovery", "Sideways / Basing", "Structural Decline", or "IPO / New Listing".

The display_metrics dict contains these keys, all rounded to 2 decimal places: dist_from_52w_high, adx, bbw_pctile, dma50_slope_pct, price_vs_50dma_pct, price_vs_200dma_pct, macd_hist, stoch_k, rsi21, rsi36, rsi56, rsi21_w, rsi36_w, rsi56_w, atr_pct, ret_1w, ret_1m, ret_3m. Each value defaults to 0.0 when the underlying indicator is NaN. The function internally calls classify_bucket() and compute_bucket_metrics().

### 3.7 renko.py: eval_renko_signals()

**Function Signature:**

```python
def eval_renko_signals(renko_daily: pd.DataFrame, renko_4h: pd.DataFrame | None,
                       r_idx: int, renko_rsi: pd.Series, h_renko_rsi: pd.Series | None) -> dict:
```

**Return Type:** `dict[str, tuple[float, bool]]`

Returns a dictionary mapping Renko signal IDs (R1 through R8) to (points, fired) tuples, identical in structure to eval_signals(). The 8 signals are: R1 (Renko RSI Oversold Recovery, 5 pts graded), R2 (Renko RSI Above 50, 4 pts binary), R3 (Renko Brick Pattern Bullish, 3 pts graded), R4 (Renko Trend Alignment, 3 pts binary), R5 (Renko 4H Lead Signal, 3 pts binary), R6 (Renko No Bearish Divergence, 2 pts binary), R7 (Renko Bullish Divergence, 4 pts binary), and R8 (Renko Hidden Bullish Divergence, 3 pts binary). Returns all zeros via _zero_results() when data is insufficient (r_idx < 1 or RSI is NaN).

### 3.8 indicators.py: compute_all_indicators()

**Function Signature:**

```python
def compute_all_indicators(df: pd.DataFrame) -> pd.DataFrame:
```

**Return Type:** `pd.DataFrame (input columns + 17 indicator columns)`

Accepts a raw OHLCV DataFrame (columns: open, high, low, close, volume) and returns a new DataFrame with the original columns plus 17 computed indicator columns using the Apollo naming convention. The output column names are the ChartAlert export names. The function computes all indicators internally (no external data needed) using only the OHLCV columns.

---

## 4. Critical Fixes: Type Consistency and Naming

Two systematic issues were identified during the audit that will cause bugs if not addressed before Phase 1 implementation. Both issues stem from the organic, iterative development history of the codebase and are easily fixed at the orchestration boundary without modifying any engine code.

### 4.1 int/Float Inconsistency

The LayerSignal engine returns bare integers while the Apollo scoring engine returns floats and the React frontend expects number (which is float in JavaScript). This mismatch will cause type errors or silent data truncation when the signals_service.py passes LayerSignal results to the API response.

| **Function** | **File** | **Line** | **Return Type** | **Issue** |
|---|---|---|---|---|
| calc_momentum_score() | layersignal_engine/core.py | 43, 54, 63-73, 196 | int | Uses bare int literals (return 0, return 10, return 8). Arithmetic on ints produces int. |
| compute_exit_layers() | layersignal_engine/core.py | 261 | int (explicit) | Explicitly casts: return int(el1 + el2 + el3 + el4). |
| eval_signals() | apollo_core/scoring.py | 258 | dict[str, tuple[float, bool]] | Correct: uses float literals (10.0, 8.0). No fix needed. |
| eval_renko_signals() | apollo_core/renko.py | 292 | dict[str, tuple[float, bool]] | Correct: uses float (round(r1_pts, 2)). No fix needed. |

**Fix Strategy:**

Wrap both LayerSignal return values in `float()` at the call site in signals_service.py. This is a 2-line change at the orchestration boundary. Do NOT modify the original engine functions, because (a) they are tested and working as-is, (b) changing return types could break other callers in the legacy codebase, and (c) the fix at the boundary is simpler and safer.

```python
ls_score = float(calc_momentum_score(df, i))  # force float
exit_pressure = float(compute_exit_layers(df, i))  # force float
```

### 4.2 Naming Convention Chaos

The codebase uses three different naming conventions for the same indicator, depending on the module. This is because the original data came from ChartAlert/Spider CSV exports which used verbose column names, while the internal computation modules use clean abbreviated names. The LayerSignal engine reads columns using the ChartAlert names, while the bucket classifier uses its own internal names.

| **Concept** | **indicators.py Output Column** | **bucket_classifier Variable** | **layersignal_engine Reads As** |
|---|---|---|---|
| RSI (21-period) | `"RSI rsi (21,C)"` | rsi21 | `df["RSI rsi (21,C)"].iloc[i]` |
| RSI (36-period) | `"RSI rsi (36)"` | rsi36 | `df["RSI rsi (36)"].iloc[i]` |
| RSI (56-period) | `"RSI rsi (56)"` | rsi56 | `df["RSI rsi (56)"].iloc[i]` |
| SMA (20) | `"SMA (20)"` | dma20 | `df["SMA (20)"].iloc[i]` |
| SMA (50) | `"SMA (50)"` | dma50 | `df["SMA (50)"].iloc[i]` |
| ADX (14) | `"ADX ADX (14,14,y,n)"` | adx | N/A (computed fresh by indicators.py) |
| MACD Line | `"MACD macd (12,26,9)"` | N/A | N/A (computed fresh by indicators.py) |
| MACD Signal | `"Signal macd (12,26,9)"` | N/A | N/A (computed fresh by indicators.py) |
| MACD Histogram | `"macd (12,26,9)_hist"` | N/A | N/A (computed fresh by indicators.py) |
| ATR (14) | `"ATR (14)"` | atr | N/A (computed fresh by indicators.py) |
| Stochastic %K | `"Stoch %K (14,3)"` | stoch_k | N/A (computed fresh by indicators.py) |
| Stochastic %D | `"Stoch %D (14,3)"` | N/A | N/A (computed fresh by indicators.py) |
| VPT | `"VPT (1)"` | N/A | N/A (computed fresh by indicators.py) |
| TP28 | `"Result Typical Price (28,n)"` | N/A | N/A (not used by LS engine) |
| WC50 | `"Result Weighted Close (50,y)"` | N/A | N/A (not used by LS engine) |
| WC21 | `"Result Weighted Close (21)"` | N/A | N/A (not used by LS engine) |
| +DI | `"+DI ADX (14,14,y,n)"` | N/A | N/A (not used by LS engine) |
| -DI | `"-DI ADX (14,14,y,n)"` | N/A | N/A (not used by LS engine) |

**Fix Strategy:**

Create a `COLUMN_MAP` dictionary in signals_service.py that maps ChartAlert names to clean internal names. Apply this map once after `compute_all_indicators()` returns, using `df.rename(columns=COLUMN_MAP, inplace=True)`. All downstream code in signals_service.py then uses the clean names. The LayerSignal engine will continue to use the original ChartAlert names (it receives a separate DataFrame copy).

### 4.3 Additional Naming Inconsistencies

Beyond indicator column names, there are additional inconsistencies in how different engines name their parameters and outputs. The bucket_classifier.py uses "dma" (Displaced Moving Average) to refer to SMAs, while indicators.py and layersignal_engine use "SMA". The bucket_classifier computes its own RSI internally using compute_rsi() while scoring.py receives pre-computed RSI arrays as numpy arrays. The signals_service.py must be aware of these differences and handle the data transformation between engines correctly.

---

## 5. PE (Price-to-Earnings) Omission Plan

The PE ratio is a fundamental data point that the original codebase sourced from EtMoney via gsheet_repo.py and fundamental_repo.py. Since no fundamental data source is available for the VPS deployment, PE must be omitted. The audit verified that this is safe to do: the backend already handles missing PE gracefully, and the React frontend already has fallback rendering for null PE values.

### 5.1 Where PE Appears in the React Frontend

| **File** | **Line** | **Usage** | **Impact of Omission** |
|---|---|---|---|
| src/types.ts | 25 | `PE: number` (required field in SignalStock interface) | Change to `PE?: number \| null` (optional with null default). |
| src/components/tabs/ScreenerTab.tsx | 377 | Table column header (sortable) | Column will exist but show "N/A" for all rows. |
| src/components/tabs/ScreenerTab.tsx | 510 | `stk.PE?.toFixed(1)` in table cell | Optional chaining returns undefined, renders as empty cell. |
| src/components/DetailPanel.tsx | 429 | `stock.PE?.toFixed(1) \|\| "N/A"` | Explicit fallback to "N/A" when PE is null. |
| src/utils/enrichment.ts | 60 | `item.PE ? item.PE < 65 : true` (G5 Quality gate) | Defaults to true when PE is missing, meaning G5 never penalizes. |

### 5.2 Where PE Appears in the Python Backend

In gates.py, the G1 (PE Sanity) gate evaluates `pe: float | None`. When pe is None, the function skips G1 entirely and sets g1_status to "OK" and g1_pe to None. This is already the desired behavior for the VPS deployment. The signals_service.py should always pass `pe=None` to `evaluate_gate_row()`, and the API response should always set `PE: null` in the SignalStock payload. No changes to gates.py are required.

### 5.3 Changes Required

Only two changes are needed. First, in the React frontend types.ts, change line 25 from `"PE: number;"` to `"PE?: number \| null;"`. Second, in the Python signals_service.py, always set the PE field to null in the response payload. Everything else (gates.py, ScreenerTab, DetailPanel, enrichment.ts) already handles the null case correctly through optional chaining, fallback rendering, and conditional skipping.

---

## 6. Step-by-Step Extraction Procedure

This section provides the exact sequence of steps an implementor must follow to extract the kernel, apply fixes, and validate the result. Each step includes the specific command or code change required. The implementor should execute these steps in order, verifying each step before proceeding to the next.

### Step 1: Create VPS Project Structure

Create a new clean project directory on the VPS with the following structure. This project will be the foundation for the entire Apollo-LS Dashboard backend. The structure separates the extracted kernel from the new orchestration and API layers, ensuring clean boundaries.

```
apollo-ls-dashboard/
├── core/                    # Extracted kernel (8 files)
│   ├── __init__.py
│   ├── scoring.py              # From apollo_core/scoring.py
│   ├── indicators.py           # From apollo_core/indicators.py
│   ├── gates.py                # From apollo_core/gates.py
│   ├── bucket_classifier.py     # From apollo_core/bucket_classifier.py
│   ├── renko.py                # From apollo_core/renko.py
│   ├── layersignal.py           # From layersignal_engine/core.py
│   ├── constants.py             # From apollo_core/constants.py
│   └── types.py                 # From apollo_core/core_types.py (if non-empty)
├── api/                     # New orchestration + FastAPI
│   ├── __init__.py
│   ├── models.py               # Pydantic models for SignalStock
│   ├── signals_service.py      # Orchestration layer (Phase 1)
│   └── routes/
│       ├── __init__.py
│       └── signals.py            # FastAPI route: GET /api/signals
├── requirements.txt
└── main.py                  # FastAPI app entry point
```

### Step 2: Copy the 8 Kernel Files

Copy the following files from the original APOLLO_BT.V1 codebase to the VPS project `core/` directory. Do NOT modify any of these files during the copy. They must remain exactly as-is to preserve the tested and working computation logic. The fixes will be applied at the orchestration boundary (Step 4), not inside the kernel files themselves.

```bash
# Copy commands (adjust source path as needed)
cp apollo_core/scoring.py apollo-ls-dashboard/core/scoring.py
cp apollo_core/indicators.py apollo-ls-dashboard/core/indicators.py
cp apollo_core/gates.py apollo-ls-dashboard/core/gates.py
cp apollo_core/bucket_classifier.py apollo-ls-dashboard/core/bucket_classifier.py
cp apollo_core/renko.py apollo-ls-dashboard/core/renko.py
cp layersignal_engine/core.py apollo-ls-dashboard/core/layersignal.py
cp apollo_core/constants.py apollo-ls-dashboard/core/constants.py
cp apollo_core/core_types.py apollo-ls-dashboard/core/types.py
```

### Step 3: Create COLUMN_MAP in signals_service.py

The column alias map translates ChartAlert export names to clean internal names. This map is applied once after `compute_all_indicators()` returns, and all downstream code in signals_service.py uses the clean names. The LayerSignal engine will receive a separate copy with the original names.

```python
# In api/signals_service.py
COLUMN_MAP = {
    "RSI rsi (21,C)": "rsi21",
    "RSI rsi (36)": "rsi36",
    "RSI rsi (56)": "rsi56",
    "SMA (20)": "sma20",
    "SMA (50)": "sma50",
    "ADX ADX (14,14,y,n)": "adx",
    "+DI ADX (14,14,y,n)": "plus_di",
    "-DI ADX (14,14,y,n)": "minus_di",
    "ATR (14)": "atr",
    "MACD macd (12,26,9)": "macd_line",
    "Signal macd (12,26,9)": "macd_signal",
    "macd (12,26,9)_hist": "macd_hist",
    "Stoch %K (14,3)": "stoch_k",
    "Stoch %D (14,3)": "stoch_d",
    "VPT (1)": "vpt",
    "Result Typical Price (28,n)": "tp28",
    "Result Weighted Close (50,y)": "wc50",
    "Result Weighted Close (21)": "wc21",
}
```

### Step 4: Apply float() Wrappers at Boundary

In signals_service.py, wrap all LayerSignal engine returns in `float()`. This is the only place where the int/float mismatch is fixed. The wrapper ensures that JSON serialization produces numeric values (e.g., 75.0) rather than integer values (e.g., 75), and that the React frontend receives consistent float types for all score fields.

```python
# In api/signals_service.py, wherever LayerSignal results are consumed:
from core.layersignal import calc_momentum_score, compute_exit_layers

ls_score = float(calc_momentum_score(df, i))      # force float
exit_pressure = float(compute_exit_layers(df, i))  # force float
```

### Step 5: Handle PE Omission

In signals_service.py, always pass `pe=None` to `evaluate_gate_row()` and always set `PE: null` in the SignalStock response. In the React frontend, change types.ts line 25 from `"PE: number;"` to `"PE?: number | null;"`. These are the only two changes needed to cleanly omit PE.

```python
# In signals_service.py
gate_results = evaluate_gate_row(last_row, pe=None)  # always None

# In response assembly
signal_stock["PE"] = None  # always null
```

```typescript
// In React src/types.ts line 25
PE?: number | null;  // was: PE: number;
```

### Step 6: Fix Import Paths

The original kernel files import from each other using `"from apollo_core."` and `"from apollo_core.indicators import compute_rsi"` style imports. After copying to the VPS project, the import paths must be updated. The renko.py file imports `compute_rsi` from `apollo_core.indicators`, which must become `"from core.indicators import compute_rsi"`. The scoring.py file imports from `apollo_core.constants` and `apollo_core.indicators`, which must become `"from core.constants import ..."` and `"from core.indicators import ..."`. Review each of the 8 kernel files for import statements that reference `"apollo_core"` and update them to `"core"`.

```bash
# Find all apollo_core references in the kernel files
cd apollo-ls-dashboard
rg "apollo_core" core/

# Replace all occurrences (verify each one before applying)
sed -i 's/from apollo_core/from core/g' core/*.py
sed -i 's/import apollo_core/import core/g' core/*.py
```

### Step 7: Smoke Test

Before proceeding to Phase 1 implementation, validate the extracted kernel with a single-stock smoke test. This test loads one stock's daily OHLCV data from a CSV or parquet file, runs it through `compute_all_indicators()` -> `eval_signals()` -> `classify_score()` -> `calc_momentum_score()` -> `compute_exit_layers()` -> `evaluate_gate_row()` -> `classify_stock_bucket()`, and prints all return values. This confirms that the kernel is self-contained, that all import paths are correct, and that the return structures match the documentation in Section 3.

```python
# smoke_test.py (save in apollo-ls-dashboard/ root)
import pandas as pd
import numpy as np
from core.indicators import compute_all_indicators
from core.scoring import eval_signals, classify_score
from core.gates import evaluate_gate_row
from core.bucket_classifier import classify_stock_bucket
from core.renko import eval_renko_signals, compute_renko_bricks, compute_renko_rsi, map_renko_to_daily
from core.layersignal import calc_momentum_score, compute_exit_layers

# 1. Load test data (replace with your test CSV path)
df = pd.read_csv("test_data/TATASTEEL_daily.csv")
df = df[['date','open','high','low','close','volume']].dropna()

# 2. Compute indicators
df_enriched = compute_all_indicators(df)
print(f"Indicators OK: {len(df_enriched.columns)} columns")

# 3. Scoring (pass appropriate params per Section 3.1)
# ... (see full test implementation)

# 4. LayerSignal
i = len(df_enriched) - 1
ls = float(calc_momentum_score(df_enriched, i))
ep = float(compute_exit_layers(df_enriched, i))
print(f"Momentum: {ls} (type: {type(ls).__name__})")
print(f"Exit Pressure: {ep} (type: {type(ep).__name__})")

# 5. Gates
row = df_enriched.iloc[-1]
gates = evaluate_gate_row(row, pe=None)
print(f"Gates: {gates}")

# 6. Bucket
bucket, metrics = classify_stock_bucket(df_enriched)
print(f"Bucket: {bucket}")
print(f"Metrics keys: {list(metrics.keys())}")

print("\n✅ SMOKE TEST PASSED")
```

---

## 7. Data Flow Diagram

The following describes the end-to-end data flow from raw OHLCV data to the React Dashboard SignalStock response. Understanding this flow is essential before writing any code. Each arrow represents a function call with specific input and output types.

**Stage 1 - Indicator Computation:**

Raw OHLCV DataFrame (columns: date, open, high, low, close, volume) is passed to `compute_all_indicators(df)` which returns a DataFrame with the original columns plus 17 indicator columns (RSI, SMA, ADX, MACD, Stochastic, VPT, ATR, TP28, WC50, WC21, +DI, -DI). The output DataFrame is then renamed using COLUMN_MAP to use clean internal names (rsi21, rsi36, rsi56, sma20, sma50, adx, etc.).

**Stage 2 - Renko Computation (Parallel):**

The original daily OHLCV DataFrame (before indicator enrichment) is passed to `compute_renko_bricks(df)` which returns a Renko brick DataFrame. Then `compute_renko_rsi(renko_df)` computes RSI on the Renko bricks. Finally, `map_renko_to_daily(renko_df, daily_df)` creates a mapping series that tells us which Renko brick corresponds to each daily bar.

**Stage 3 - Apollo Scoring:**

The enriched DataFrame (from Stage 1) is split into daily, 4-hour (if available), and weekly timeframes. Trough detection runs on RSI series. `eval_signals()` is called with the last bar indices from each timeframe, returning a dict of 26 signal results. `eval_renko_signals()` is called with the last Renko brick index, returning a dict of 8 Renko signal results. All fired signal points are summed to produce the total Apollo score. `classify_score()` maps the total score to an action label and position percentage.

**Stage 4 - LayerSignal Scoring:**

A copy of the enriched DataFrame (with original ChartAlert column names, NOT renamed) is passed to `calc_momentum_score(df, last_idx)` which returns a 0-100 momentum score (int, wrapped to float). `compute_exit_layers(df, last_idx)` returns a 0-100 exit pressure score (int, wrapped to float). These two scores, combined with the momentum regime and state, determine the LayerSignal action (ENTRY, HOLD, EXIT, FLAT).

**Stage 5 - Gate Evaluation:**

The enriched DataFrame's last row is passed through `compute_gate_frame()` to get gate-specific columns, then `evaluate_gate_row()` is called with `pe=None` to produce a dict with G1-G4 status values and all_pass boolean. The gates produce 5 boolean values for the React Gates array (G1-G4 from evaluate_gate_row + G5 computed as a new RSI stack alignment check in signals_service.py).

**Stage 6 - Bucket Classification:**

`classify_stock_bucket(df_d, df_w)` returns a (bucket_string, display_metrics_dict) tuple. The bucket_string (one of 8 values) is mapped to the React L1-L4 tier: L1 = PRIME, L2 = CONFIRMED, L3 = EMERGING, L4 = WATCH, and Recovery/Sideways/Decline/IPO map to a stage_description string. The display_metrics dict provides all numerical metrics for the React dashboard.

**Stage 7 - Response Assembly:**

The signals_service.py assembles all results into a SignalStock dict matching the 49-field React interface. This dict includes: Symbol, Date, Apollo_Action, Apollo_Score, LayerSignal_Action, LayerSignal_Score, Exit_Pressure, Open/High/Low/Close/Volume, RSI21/RSI36/RSI56, ADX, ATR_Pct, Bucket (L1-L4), LStage, ELStatus, Conviction, Gates ([bool x 5]), Renko (GREEN/RED/NEUTRAL), MCap, FQS, Sparkline, PE (null). The response is returned as `{stocks: SignalStock[], summary: SignalsSummary}`.

---

## 8. React Frontend Contract: SignalStock Interface

The React frontend defines a 49-field SignalStock interface in src/types.ts. The signals_service.py must produce a JSON object that matches every field. The following table maps each field to its source in the Python engine. Fields marked "New" do not exist in the current Python engines and must be computed in signals_service.py.

| **Field** | **Type** | **Source / Computation** |
|---|---|---|
| Symbol | string | From OHLCV data |
| Date | string | Last bar date from OHLCV data |
| Apollo_Action | string | classify_score() first element (e.g., "STANDARD", "WATCHLIST") |
| Apollo_Score | number | Sum of all fired signal points (0-148 with Renko, 0-128 without) |
| Pct_Change | number | (close[i] - close[i-1]) / close[i-1] * 100 |
| LayerSignal_Action | string | Derived from momentum score and exit pressure (ENTRY/HOLD/EXIT/FLAT) |
| LayerSignal_Score | number | float(calc_momentum_score(df, i)) |
| Exit_Pressure | number | float(compute_exit_layers(df, i)) |
| Open/High/Low/Close | number | Last bar OHLC from OHLCV data |
| Volume | number | Last bar volume from OHLCV data |
| High52W / Low52W | number | 52-week high/low from rolling window on close |
| RSI21 / RSI36 / RSI56 | number | From enriched DataFrame (rsi21, rsi36, rsi56 columns) |
| ADX | number | From enriched DataFrame (adx column) |
| ATR_Pct | number | atr / close * 100 |
| 20D_SMA / 50D_SMA / 200D_SMA | number? | From enriched DataFrame or computed via rolling(20/50/200).mean() |
| PE | number \| null | Always null (no fundamental data source) |
| Stochastic | number | Stoch %K value from enriched DataFrame |
| 52W_Prox | number | (close - low_52w) / low_52w * 100 |
| CMP | number | Last close price |
| Traded_Value | number | close * volume (last bar) |
| Bucket | string | L1-L4 mapped from bucket_classifier tier |
| LStage | string | Stage description from bucket classifier |
| ELStatus | string | EL1-EL4 or NONE based on exit pressure thresholds |
| Conviction | number | 0.0-1.0 derived from score, gates, and bucket tier |
| Gates | boolean[5] | [G1, G2, G3, G4, G5] from gate evaluation + RSI stack |
| Renko | string | GREEN/RED/NEUTRAL from last Renko brick direction |
| MCap | string | Large/Mid/Small from market cap data (API-based) |
| FQS | string | A/B/C/D quality grade from score + gates + exit pressure |
| Sparkline | number[] | Last 30-60 close prices for mini chart |
| ml_confidence | number? | New: ML meta-learner probability (Phase 6+) |
| SubScores | object? | New: {trend, momentum, volatility, volume, marketFilter} (Phase 6+) |
| GatesExplanations | string[5]? | New: Human-readable gate explanations (Phase 1) |
| HistoricalL3Events | array? | New: Historical L3 event log (Phase 5+) |
| isIPO | boolean? | New: IPO flag from ipo_lookup.py (Phase 4) |
| ipoData | object? | New: IPO details object (Phase 4) |

---

## 9. Verification Checklist

Before proceeding to Phase 1 (signals_service.py implementation), the implementor must verify each item in this checklist. Any failure indicates a problem with the kernel extraction or fix application that must be resolved before continuing.

| **#** | **Verification Item** | **How to Verify** | **Expected Result** |
|---|---|---|---|
| 1 | All 8 kernel files copied to core/ | `ls -la core/` | 8 .py files + __init__.py |
| 2 | No import errors in kernel | `python -c "from core.scoring import eval_signals"` | No ImportError or ModuleNotFoundError |
| 3 | compute_all_indicators() works | Pass a 200-row OHLCV DataFrame | Returns DataFrame with 17 extra columns |
| 4 | COLUMN_MAP applied correctly | Check df.columns after rename | All columns use clean names (rsi21, not "RSI rsi (21,C)") |
| 5 | eval_signals() returns dict of tuples | Call with last-bar indices | dict[str, tuple[float, bool]] with 26+ keys |
| 6 | classify_score() returns tuple | Call with score=85, r_points=10 | ("STANDARD", "60-70%") |
| 7 | calc_momentum_score() returns float | Check type(result) | type is float, not int |
| 8 | compute_exit_layers() returns float | Check type(result) | type is float, not int |
| 9 | evaluate_gate_row(pe=None) works | Call with pe=None | g1_pe=None, g1_status="OK", all_pass=True |
| 10 | classify_stock_bucket() returns tuple | Call with daily+weekly DataFrames | (bucket_str, dict with 18 numeric keys) |
| 11 | eval_renko_signals() returns dict | Call with Renko data at r_idx | dict with R1-R8 keys, all tuple[float, bool] |
| 12 | PE field is null in response | Check JSON output | `"PE": null` |
| 13 | No original codebase references remain | `rg "apollo_core" core/` | No matches |
| 14 | Smoke test passes end-to-end | Run single-stock test | All engines return expected types, no errors |