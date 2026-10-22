# TRD — IDX Portfolio Optimizer

Companion to [PRD.md](PRD.md), which owns the what and the why. This document owns the how: modules, data contracts, algorithms, error handling, and the test strategy. Version 1.0, Oct 2026.

## 1. Technology & environment

| Component | Choice | Notes |
|---|---|---|
| Language | Python 3.14 venv (`.venv` via `py -3.14`) | Fallback: Python 3.12 via `winget install Python.Python.3.12`, then rebuild venv and pin whatever versions installed cleanly. |
| UI | Streamlit | Sidebar inputs, reruns on interaction; all Yahoo calls cached (§5). |
| Data | yfinance (batch `yf.download`, `auto_adjust=True` → adjusted close) | Daily bars, `.JK` suffix tickers, ^JKSE index for benchmark. |
| Optimizer | `scipy.optimize.minimize`, SLSQP | Constraint set per §4.2. |
| Charts | plotly | Heatmap, frontier, equity/drawdown curves. |
| Tests | pytest | Headless; imports only `optimizer.py`, `backtest.py`, and data utilities. |
| Runtime deps listed in `requirements.txt` with exact pinned versions recorded from the working venv; numpy/pandas/scipy also installed into the venv (system copies exist but the venv is self-contained). |

Run: `streamlit run app.py`. Tests: `pytest` from repo root.

## 2. Architecture

```
app.py (Streamlit UI, presentation only)
  │ sidebar inputs: tickers, years, cap, rf, cost
  ▼
data.py ──load_prices()──► DataFrame[dates × tickers]  (st.cache_data)
  │       ──stock_metrics(), correlation()
  ▼
optimizer.py ──frontier / min-vol / max-sharpe (pure scipy, no streamlit)
  ▼
backtest.py ──walk_forward() (pure pandas/scipy, re-optimizes monthly)
  ▼
app.py renders: metrics table, heatmap, frontier + weight tables,
backtest equity/drawdown + metrics vs ^JKSE and equal-weight, caveats expander
```

Rules:

- **`optimizer.py` and `backtest.py` import nothing from Streamlit.** They are pure functions over numpy/pandas objects so pytest runs headless and the code is portable.
- `app.py` contains no math beyond formatting; all computation lives in the three modules.
- `data.py` is the only module allowed to import `st` (for `st.cache_data`).

## 3. Data contracts

- **Adj-close frame:** `pd.DataFrame`, DatetimeIndex (ascending, no duplicate dates), one column per accepted ticker. Columns keep full Yahoo tickers (`BBCA.JK`) internally; the UI strips `.JK` for display.
- **Normalized inputs to optimizer:** `mu` (annualized mean returns, len-n vector), `cov` (annualized covariance: daily covariance × 252, n×n). Computed in `data.py`, consumed by `optimizer.py` and `backtest.py` — one normalization point, no re-annualization downstream.
- **Weights:** 1-D array length n, `sum(w) == 1`, `0 ≤ w ≤ cap`.
- **Backtest result:** dataclass holding strategy equity curve (indexed by month-end), equal-weight equity curve, `^JKSE` equity curve, per-rebalance turnover and cost series, final-weights DataFrame.
- **Warnings channel:** `load_prices()` returns `(frame, dropped: dict[ticker, reason])`; `app.py` renders `st.warning` per dropped ticker.

## 4. Module specifications

### 4.1 `data.py`

| Function | Contract |
|---|---|
| `normalize_tickers(list[str]) -> list[str]` | Appends `.JK` to bare tickers, leaves full and index tickers untouched. |
| `load_prices(tickers, years, ttl) -> (frame, dropped)` | One batch `yf.download(..., auto_adjust=True)` call; wrapped in `st.cache_data` (§5). Validation (§6): tickers whose adjusted close is missing for more than a tolerance of the window are dropped with a reason. Raises `DownloadError` (custom exception) if the whole download fails — the app shows `st.error` and stops, never renders an empty-chart page. |
| `monthly_returns(frame) -> DataFrame` | Month-end resample → pct-change. Feeds both backtest loops. |
| `stock_metrics(frame, rf) -> pd.DataFrame` | Per ticker: annualized return (daily mean × 252), volatility (daily std × √252), Sharpe = (return − rf)/vol, max drawdown = min over t of (price/cummax − 1). |
| `correlation(frame) -> pd.DataFrame` | Pearson correlation of daily returns. |

### 4.2 `optimizer.py`

Problem formulation (PRD F6): minimize `wᵀΣw` subject to `Σw = 1`, `0 ≤ wᵢ ≤ cap`, and—for frontier points—`μᵀw = target`.

| Function | Contract |
|---|---|
| `min_vol_weights(mu, cov, cap) -> np.ndarray` | Single SLSQP solve from equal-weight start; returns weights or raises `InfeasiblePortfolio` when the model has no solution (see guard below). |
| `max_sharpe_weights(mu, cov, cap, rf) -> np.ndarray` | Directly maximizes `(μᵀw − rf)/√(wᵀΣw)` via SLSQP (negative-Sharpe minimization). |
| `efficient_frontier(mu, cov, cap, rf, points=50) -> pd.DataFrame` | SLSQP at a grid of target returns spanning `[min achievable, max achievable]` (bounds computed from `min(μᵢ), min(cap-scaled upper bound)` and `max μᵢ`); returns columns `target_return, vol, weights, is_min_vol, is_max_sharpe`. Uses `ftol` tightened so the curve is visually smooth; skips QP-unreachable targets silently. |

Feasibility guard (applies to all three): if `cap × n < 1` the sum-to-1 constraint is unsatisfiable — `InfeasiblePortfolio` names the fix in its message ("need ≥ ceil(1/cap) stocks; currently n"). The UI separately enforces PRD's minimum of 5 stocks so users never hit the exception in normal flow; the guard exists because code must not trust UI.

### 4.3 `backtest.py`

| Function | Contract |
|---|---|
| `walk_forward(monthly_prices, lookback_months=36, cost_rate=0.0025, objective=max_sharpe_weights) -> BacktestResult` | Loop over month-ends t after the first `lookback_months`: fit `mu/cov` on the trailing window `[t − lookback, t)` (strictly excluding t's own month returns), call `objective`, charge `cost = cost_rate × Σ|Δw|` against the next month, invest remaining capital. Produces the strategy curve, equal-weight curve (monthly rebalanced, **same cost_rate charged on its turnover**), turnover/cost series, and the weights ledger. |

Precision points:

- Window slicing is the honesty mechanism (PRD F7): the mu/cov passed to the optimizer are computed only from rows `[t − lookback, t)`. Implemented as slicing on the resampled monthly frame, so a regression off-by-one is structurally impossible.
- Return accounting: strategy month return = `wᵀ r_{t}` on weights held into t, then costs applied at rebalance. Equity compounds geometrically.
- `^JKSE` comparison series is fetched once with the same `load_prices` path and compounded raw (price index, no costs, no dividends — the caveat lives in the UI expander, not in the math).

### 4.4 `app.py`

Layout, top to bottom: sidebar (multiselect with the 15 presuggested names, years slider 4–10 default 7, cap slider default 30%, rf number input default 7.4%, cost input default 0.25%) → auto-drop warnings → per-stock metrics table → correlation heatmap → frontier chart with min-vol/max-Sharpe highlighted + both weight tables → backtest equity curves + drawdown + metrics table (strategy vs equal-weight vs ^JKSE, including cumulative cost drag) → caveats expander (constant rf simplification; ^JKSE excludes dividends while adjusted close includes them; survivorship bias; Yahoo data quality). English only.

## 5. Caching & performance

- `st.cache_data` returns the download keyed on `(tickers, years, calendar month)` with TTL 12h — a new calendar month naturally forces a refresh for walk-forward realism; explicit "Clear cache" button in the sidebar for debugging.
- Workload sizing: 15 tickers × ~10 years daily ≈ trivial for pandas; 50-point SLSQP frontier ≈ sub-second. No async, no chunking needed.
- SLSQP is deterministic; no seeds anywhere.

## 6. Error handling & trust boundaries

| Boundary | Guard |
|---|---|
| User input | `.JK` normalization; ≥ 5 accepted tickers enforced before compute; slider bounds are the PRD bounds. No free-text weights. |
| Yahoo download | Partial data → drop + warn (never silently fill); full failure → `DownloadError` → `st.error` with retry hint (Yahoo rate limits); all downloads cached so retries are cheap. |
| Optimizer | `InfeasiblePortfolio` raised, never swallowed; SLSQP non-convergence returns the last feasible iterate with an explicit warning flag rather than a silent garbage vector. |
| Backtest | Assert invariants inside the loop at debug level: weights in `[0, cap]`, sum 1 before cost application; negative equity impossible by construction (costs ≤ turnover value). |
| Display | `.JK` stripped for all user-facing labels; percents formatted ×100 with fixed digits. |

## 7. Test strategy (`tests/`)

Synthetic fixtures: deterministic RNG price frames (e.g. 6 assets, one with deliberately high mean/low vol so the optimizer has a clear answer; a N×3y monthly frame for backtests). No network in tests.

1. **No-look-ahead (the PRD-decisive test):** monkeypatch the objective to record the `(start, end)` of every mu/cov window the backtest hands it; assert every window's end is strictly before the month whose returns it invests into, and every window is exactly `lookback_months` long.
2. Weight invariants: sums to 1, bounds respected, for min-vol and max-Sharpe across several caps (30%, 20%, 50%).
3. Ordering: min-vol vol ≤ max-Sharpe vol at the same constraints; frontier target returns monotone in realized vol.
4. Backtest arithmetic: a hand-computed 2-rebalance case where turnover, cost, and resulting equity are checked to the digit (charges `cost_rate × Σ|Δw|`, compounds correctly).
5. Equal-weight parity: identical cost treatment to the strategy — the equity curves are computed by the same accounting function, only the weight-derivation differs.
6. Data hygiene: a ticker with a short history is dropped and reported, errors surface as `DownloadError` on total failure (yfinance mocked).
7. Infeasibility: `cap × n < 1` raises `InfeasiblePortfolio` with the actionable message.

## 8. Milestone-to-module mapping

M0 scaffold + pinned deps → M1 `data.py` (§4.1, §5, §6 download path) → M2 `optimizer.py` (§4.2) → M3 `backtest.py` (§4.3) → M4 `app.py` (§4.4) → M5 tests (§7) → M6 README (screenshots, methodology, PRD's caveat list). One commit per milestone.
