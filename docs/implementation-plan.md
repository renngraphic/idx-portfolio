# Implementation Plan — IDX Portfolio Optimizer

Companion to [PRD.md](PRD.md), [TRD.md](TRD.md), [backend-schema.md](backend-schema.md), [app-flow.md](app-flow.md), [design-brief.md](design-brief.md). This is the build order: what gets written, in what order, against which acceptance check, into which commit.

## Principles

- Vertical, not horizontal: each milestone ends working (runs green end-to-end for its scope), then one commit. No branch on hold.
- Backend before frontend, pure before cached: `optimizer.py` and `backtest.py` first (they are the hard math and can be built from synthetic data with no internet).
- Every design decision documented in PRD/flow/schema is treated as fixed and testable here — M5 re-derives the invariants from first principles instead of re-reading them off the docs.

## Environment (preflight, before M0's commit)

| Check | If it fails |
|---|---|
| `py -3.14 -m venv .venv` | — |
| `.venv\Scripts\pip install streamlit yfinance plotly pytest` (numpy/pandas/scipy ride along) | Fallback: `winget install Python.Python.3.12` → rebuild venv (system change — confirmed with owner first) |
| `pip freeze > requirements.txt` (pin everything incl. transitive) | — |
| `git init`, first `.gitignore` (.venv/, __pycache__, .streamlit/secrets.toml, .nojekyll) | — |

## M0 — Scaffold

| Task | File |
|---|---|
| Repo layout: `app.py data.py optimizer.py backtest.py tests/ docs/ README.md requirements.txt .gitignore` | all |
| Streamlit config: `.streamlit/config.toml` (base dark + design-brief §7 tokens) | config only |
| Verify `python -m pytest` runs green on an empty suite | — |

**Accept:** `streamlit run app.py` shows a hello page; venv pinned. **Commit:** `chore: scaffold project with pinned deps`.

## M1 — data.py (PRD F2, F3; schema §1–2, §4)

Build `normalize_tickers`, `load_prices` (+ `st.cache_data`, download → drop loop per dropout tolerance), `monthly_returns`, `stock_metrics`, `correlation`, and a throwaway `st.download_button`-free console check via `python -c` over real Yahoo data.

**Accept:** all 15 suggested tickers probed live (PRD F1 validation); ADRO-split-era histories check out or get swapped; each dropped ticker reasons the exact failing count. **Commit:** `feat(data): cached yfinance loader and per-stock metrics`.

## M2 — optimizer.py (PRD F6; schema §6)

Three SLSQP functions (`min_vol_weights`, `max_sharpe_weights`, `efficient_frontier`), annualized mu/cov input. Feasible-set-spread for the frontier grid, `InfeasiblePortfolio` guard, `NotConverged` row flag.

**Accept (hand-check, no internet needed):** synthetic 6-asset set — min-vol ≠ max-Sharpe, both respect cap and sum-to-1, frontier monotone; package-manual `pytest` smoke added here. **Commit:** `feat(optimizer): cap-constrained frontier, min-vol, max-sharpe`.

## M3 — backtest.py (PRD F7–F10; schema §3)

`walk_forward` with the window-slicing regime `[t − lookback, t)` (strict), `BacktestResult` dataclass, equal-weight path with the same cost rule, turnover = Σ|Δw| ledger.

**Accept:** hand-computed 2-rebalance fixture reconcile to the digit: turnover, cost, equity; optimizer receives mu/cov of exactly `lookback` rows. **Commit:** `feat(backtest): monthly walk-forward with turnover costs`.

## M4 — app.py (PRD F11–F12 + all; app-flow §1–6; design-brief §3–8)

wire sidebar (multiselect w/ 15 presuggested slug list, sliders, inputs, clear-cache button) → pipeline (state A–G exact per flow table) → story scroll render (8 blocks in order incl. caveat block and drawdown/cost line) → plotly template from `config.toml` tokens.

**Accept:** flows E (4 stocks → single-line stoppage), F (yfinance mocked total failure), D (dead ticker drop warning) all manually triggered once; screenshots taken here for M6; no ban-list item present (design-brief §5). **Commit:** `feat(app): story-scroll UI with verdict-first layout`.

## M5 — tests/ (TRD §7 1–7; schema invariants)

Fix over synthetic frames (deterministic seeds): no-look-ahead window recorder (test 1 first — it is the PRD-decisive one), weight invariants at 20/30/50% caps, ordering, 2-rebalance digit-exact fixture, equal-weight parity, drop behavior w/ mocked yfinance, `InfeasiblePortfolio` message.

**Accept:** full `pytest` green locally; timeout-hard (fixtures never touch network). **Commit:** `test: invariants, no-look-ahead guard, fixture arithmetic`.

## M6 — README.md + polish

Methodology, screenshot embeds (M4 captures), PRD caveat list, run instructions, project-structure diagram, doc links (docs/*.md), one-line "choose this stack because" note.

**Accept:** fresh-clone boot check on a second machine or clean venv (`pip install -r requirements.txt && pytest && streamlit run app.py`). **Commit:** `docs: readme with methodology and screenshots`.

## Dependency map

```
env → M0 → M1 → M2 → M3 → M4 → M5 → M6
           └──────── M2/M3 only need a doc-fixed prices fixture (M1's loader is nice-to-have, not a blocker) ────────┘
```

M4 depends on all of M1–M3; M5 depends on M2–M3; M6 depends on M4. M2+M3 sequence: M3's objective callable injects M2's `max_sharpe_weights` — write M2 first.

## Risk choreography (pre-agreed triggers)

| Trigger | Move |
|---|---|
| streamlit/yfinance wheel install fails on 3.14 | winget Python 3.12, rebuild venv, repin (owner said yes to the class of change; confirm before running) |
| A suggested ticker's Yahoo history is broken (rename/split, e.g. ADRO) | swap another liquid name into the presuggestion list — decision is M1's owner moment |
| Yahoo rate-limits during development | work on cached fixture data until cooldown; `st.cache_data` keeps demos offline-safe |
| Frontier visually not smooth at points=50 | bump to 75–100 before touching SLSQP tolerances |
