# Backend Schema — IDX Portfolio Optimizer

Companion to [TRD.md](TRD.md), which describes module behavior; this document pins the data: every object the backend passes around, with type, shape, dtype, invariants, and owner. There is **no database and no server** — the backend is a set of pure Python functions over pandas/numpy objects plus one external source (Yahoo). Persisted state is limited to the st.cache_data entry and the git repo itself.

Stability rule: field names in this document are identical to names in code; changing a row here means changing the TRD with it.

## 1. Source data (Yahoo, via yfinance)

| Object | Shape | Notes |
|---|---|---|
| Raw yfinance download | `DataFrame[Date × MultiIndex(Field, Ticker)]`, `Float64` (nullable) | Fields used: `Close` (auto-adjusted = adjusted close), `Volume`*unused*. Index = trading days, ascending, tz-naive. |
| ^JKSE series | `Series[Date] → Float64` | Price index; fetched in the same batch via ticker `"^JKSE"`. |

Schema of the *served* adj-close frame (`PriceFrame`, produced by `data.load_prices`):

| Property | Contract |
|---|---|
| Index | `DatetimeIndex`, ascending, no dups, tz-naive |
| Columns | full Yahoo tickers, `STR`, `^JKSE` excluded (handled separately by the backtest benchmark path) |
| Values | `float64`, no `NaN` tolerated beyond §1-drop tolerance; gaps forward-filled **only** within a tolerance, never at series head/tail |
| Rows | one per trading day in the requested window |
| Invariants | every accepted column spans ≥ 90% of window trading days (else dropped before this frame exists) |

## 2. Derived analytics objects

| Object | Type/shape | Owner | Invariants |
|---|---|---|---|
| `monthly_returns` | `DataFrame[month-end × ticker]`, `float64` | data.py | pct-change of month-end closes; first row `NaN`-dropped; columns == PriceFrame columns |
| `mu` | `np.ndarray[n]`, annualized | data.py | `mu_i = mean(daily r_i) × 252` |
| `cov` | `np.ndarray[n,n]`, annualized, symmetric | data.py | daily cov × 252; PSD by construction |
| `metrics` | `pd.DataFrame[n × 4]` → `return, vol, sharpe, max_drawdown` | data.py | same annualization; `max_drawdown ≤ 0`; display adds ticker label |
| `corr` | `pd.DataFrame[n×n]`, `float64` | data.py | diag = 1; values in [−1, 1] |
| `weights` | `np.ndarray[n]`, `float64` | optimizer.py | `sum == 1 ± 1e-6`; `0 ≤ w ≤ cap + 1e-6` |
| `frontier` | `pd.DataFrame[points × (target_return, vol, sharpe, is_min_vol, is_max_sharpe, weights)]` | optimizer.py | `weights` cell = `np.ndarray[n]`; monotone target_return ↑, no duplicates; exactly one `is_max_sharpe`, one `is_min_vol` |

## 3. Backtest contract

`BacktestResult` dataclass (backtest.py), all series indexed by backtest month-end:

| Field | Type | Meaning |
|---|---|---|
| `strategy_equity` | `Series[Date → float]` | gross-of-cash geometric compounding after costs |
| `equal_weight_equity` | `Series[Date → float]` | monthly rebal, same cost rule |
| `benchmark_equity` | `Series[Date → float]` | ^JKSE raw price, no costs |
| `turnover` | `Series[Date → float]` | `Σ\|Δw\|` charged at each rebalance |
| `costs` | `Series[Date → float]` | `rate × turnover`, cumulative version via `.cumsum()` on render |
| `weights_history` | `DataFrame[month-end × ticker]` | weights held *into* each month |
| `meta` | `dict` | `n_windows`, `lookback_months`, `first_rebalance_date`, `cost_rate`, `rf` |

Invariants: all three equity series share one index; first value = 1.0; equity ≥ 0; `len(weights_history) + 1 == len(turnover)` style ledger consistency; benchmark starts at 1.0 on the same date.

## 4. Cache schema (st.cache_data on load_prices)

| Key component | Type |
|---|---|
| `tickers` | `tuple[str, ...]` (accepted, sorted — order must not split the cache) |
| `years` | `int` |
| `month_key` | `str` `"YYYY-MM"` (forces refresh on new calendar month) |

Value: serialized PriceFrame + `dropped: dict[str, str]` + `fetched_at: Timestamp`. TTL 12h. Never persisted to disk beyond Streamlit's own cache directory.

## 5. Error taxonomy

| Error | Type | Raised by | Payload |
|---|---|---|---|
| `DownloadError` | custom Exception | data.py | message with retry hint; never silent |
| `InfeasiblePortfolio` | custom Exception | optimizer.py | names the numeric fix ("need ≥ ceil(1/cap) stocks") |
| `NotConverged` | warning flag on frontier rows | optimizer.py | keeps last feasible iterate; not an exception (§ TRD 4.2) |

Mapping to App-flow states E/F/G is 1:1: one error → one user-facing state.

## 6. Function signatures (backend perimeter)

```python
# data.py
def normalize_tickers(raw: list[str]) -> list[str]
def load_prices(tickers: list[str], years: int) -> tuple[pd.DataFrame, dict[str, str]]
def monthly_returns(frame: pd.DataFrame) -> pd.DataFrame
def stock_metrics(frame: pd.DataFrame, rf: float) -> pd.DataFrame
def correlation(frame: pd.DataFrame) -> pd.DataFrame

# optimizer.py
def min_vol_weights(mu: np.ndarray, cov: np.ndarray, cap: float) -> np.ndarray
def max_sharpe_weights(mu: np.ndarray, cov: np.ndarray, cap: float, rf: float) -> np.ndarray
def efficient_frontier(mu: np.ndarray, cov: np.ndarray, cap: float,
                       rf: float, points: int = 50) -> pd.DataFrame

# backtest.py
def walk_forward(monthly: pd.DataFrame, lookback_months: int = 36,
                 cost_rate: float = 0.0025,
                 objective: Callable[[np.ndarray, np.ndarray, float, float], np.ndarray] = max_sharpe_weights
                ) -> BacktestResult
```

Nothing outside these signatures crosses module boundaries. app.py consumes only these (plus BacktestResult fields).
