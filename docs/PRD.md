# PRD — IDX Portfolio Optimizer

**Status:** Approved (design finalized via grilling sessions, Oct 2026)
**Location:** `E:\projects\idx-portfolio\`

## 1. Purpose

A web app that takes a selection of 5–10 Indonesian (IDX) stocks and builds the best portfolio mix based on risk and return, then shows honestly whether that optimization actually helps over time.

**Target:** a GitHub portfolio project written to be read by recruiters: clean structure, honest backtest methodology, documented caveats, tested math.

## 2. Problem

A concentrated basket of familiar IDX blue chips (e.g. 3 banks + telco + mining) is usually far less diversified than it looks. Naive equal weighing ignores correlations; naive Markowitz tutorials leak look-ahead data and ignore real-world brokerage costs, so their backtests flatter the strategy.

## 3. Scope

### 3.1 In scope (functional requirements)

| # | Requirement |
|---|-------------|
| F1 | Stock selection: multiselect of 5–10 tickers. The app appends `.JK` automatically — the user types `BBCA`, the app queries `BBCA.JK`. Default suggestion list: BBCA, BBRI, BMRI, TLKM, EXCL, ASII, ADRO, ANTM, PGAS, UNVR, ICBP, INDF, KLBF, CPIN, GGRM (3 banks; consumer, food, and mining names for real diversification). Minimum 5 valid stocks — the weight-cap math is infeasible below that. |
| F2 | Price history: daily adjusted close via yfinance, 7 years by default (slider 4–10). 7 years minus a 3-year walk-forward lookback leaves ~4 years of genuinely out-of-sample backtest. |
| F3 | Data hygiene: tickers with missing/dead Yahoo history are auto-dropped with a visible warning naming them (handles renames and corporate actions such as the ADRO split). Downloads cached with `st.cache_data` because Yahoo rate-limits. |
| F4 | Per-stock metrics: annualized return, annualized volatility, Sharpe ratio, max drawdown. Annualization: daily mean × 252 and daily std × √252. |
| F5 | Correlation heatmap (plotly) to expose false diversification. |
| F6 | Markowitz optimization under long-only constraints, weights sum to 1, per-stock cap default 30% (UI input). Efficient frontier plotted as a solved curve (SLSQP at a grid of target returns), with the min-volatility and max-Sharpe portfolios highlighted and their weights shown. |
| F7 | Walk-forward backtest (monthly): at each month-end t, re-optimize max-Sharpe on the trailing 3-year window ending at t — data strictly before t, no look-ahead. Apply those weights to month t+1. |
| F8 | Market realism: turnover = Σ\|Δw\| per rebalance; cost = 0.25% per unit of turnover by default (IDX broker blend: ~0.15% buy, ~0.25% sell incl. 0.1% sell tax), UI-editable. |
| F9 | Benchmarks: ^JKSE price return, and a monthly-rebalanced equal-weight portfolio with the same costs applied. Equal-weight out-of-sample is the real bar: beating it is the honest test of whether optimization helps. |
| F10 | Backtest reporting: equity curves, CAGR, volatility, Sharpe, max drawdown, cumulative cost drag — for the strategy and both benchmarks. |
| F11 | Risk-free rate: UI-editable input, default 7.4% (Indonesia 10Y government bond yield, verified 22 Oct 2026). |
| F12 | Caveats displayed in-app (expander): constant rf across the history window is a simplification (Sharpe values are approximate; the walk-forward comparison stays fair because every strategy shares the rate); ^JKSE is a price index excluding dividends while adjusted close includes them; survivorship bias (picking today's liquid names flatters the backtest); Yahoo data quality. |

### 3.2 Non-goals (out of scope)

- Deployment to Streamlit Community Cloud (build is deploy-ready: pinned requirements, no secrets, cached data — deployment happens later only if Yahoo behaves on the shared cloud IP).
- Live/automated risk-free-rate fetching (no reliable free IDR-government-bond ticker on Yahoo; a hardcoded editable default avoids a rot-prone second data source).
- Short selling.
- Live trading, brokerage integration, tax-aware rebalancing.
- Intraday or fundamentals data; the universe is daily close based only.

## 4. Non-functional requirements

- **Stack:** Streamlit (UI, caching), yfinance (data), scipy SLSQP (optimizer), plotly (charts), pytest (tests).
- **Structure:** four small modules — `app.py` (UI only), `data.py`, `optimizer.py`, `backtest.py`. Optimizer and backtest are pure functions with no streamlit imports so they run headless under pytest. Plus `requirements.txt` (pinned versions), `.gitignore`, `README.md`, `docs/`, `tests/`.
- **Python:** 3.14 venv first; if streamlit/yfinance wheels fail on 3.14, rebuild on 3.12 (installed via winget) and pin the working versions.
- **Git:** repo initialized day one; one commit per milestone (scaffold, data, optimizer, backtest, app, tests, README).

## 5. Success criteria

1. `pytest` passes, including the no-look-ahead test: the backtest's optimizer is provably only ever called with windows ending before month t — the test that proves the walk-forward is honest.
2. A 5+ stock default selection runs end-to-end locally (`streamlit run app.py`) with all 15 suggested tickers validated against live Yahoo data.
3. The backtest panel shows the strategy vs ^JKSE and vs equal-weight with costs applied — and the independently-derived shapes of those curves are reproducible from returned series.
4. README carries the methodology and the caveat list (constant rf, ^JKSE no dividends, survivorship bias, data quality) — the answers interviewers probe for.

## 6. Milestones

| M | Deliverable |
|---|-------------|
| M0 | Scaffold: folder, git init, .gitignore, venv, pinned requirements |
| M1 | `data.py`: cached download, auto `.JK`, auto-drop, per-stock metrics |
| M2 | `optimizer.py`: cap-constrained frontier, min-vol, max-Sharpe |
| M3 | `backtest.py`: monthly walk-forward, turnover costs, two benchmarks |
| M4 | `app.py`: sidebar inputs, heatmap, frontier, backtest panels, caveats |
| M5 | `tests/`: invariants + no-look-ahead test |
| M6 | README with screenshots, final commit |

## 7. Known risks

- Yahoo wheels/instability on very new Python 3.14 — fallback path defined.
- Yahoo rate limits — mitigated by `st.cache_data`.
- The walk-forward result will look worse than a frozen optimum; that difference is the point, and the README owns the story.
