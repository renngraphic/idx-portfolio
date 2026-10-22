# App Flow — IDX Portfolio Optimizer

Companion to [PRD.md](PRD.md) (requirements F1–F12), [TRD.md](TRD.md) (module contracts), and [design-brief.md](design-brief.md) (visual rules). This document owns the runtime behavior: what happens when, from first load to every sidebar change.

## 1. States

The app is a Streamlit session — every sidebar interaction restarts the script top-to-bottom. States from the user's view:

| State | Trigger | What is visible |
|---|---|---|
| **A. Fresh load** | First run of session | Sidebar populated with defaults (15 suggested tickers, 7y, cap 30%, rf 7.4%, cost 0.25%); compute pipeline starts immediately on the default selection — no empty page. |
| **B. Loading** | Any (re)compute | Progress message per pipeline stage (§3): "Downloading prices…" → "Optimizing frontier…" → "Running walk-forward backtest…". Cached steps pass silently. |
| **C. Rendered** | Pipeline succeeded | Full story scroll (§4) with every section populated. |
| **D. Warned** | Some tickers dropped | Rendered + amber-warning messages directly under the sidebar naming dropped tickers, `.JK` normalized and all accepted tickers still analyzed. |
| **E. Input-rejected** | < 5 valid stocks or invalid inputs block | Page reduced to a clear single-line message ("Need ≥ 5 valid stocks — currently 4"), pipeline stops before download. Sidebar stays interactive. |
| **F. Download failed** | yfinance total failure (`DownloadError`) | Red error stating rate-limit/retry hint, "Clear cache" suggestion; render stops, partial page never shown. |
| **G. Infeasible model** | `InfeasiblePortfolio` (should not occur under UI guard; code must not trust UI) | Error naming the fix ("cap × stocks < 100% — raise cap or add stocks"). |

## 2. Interaction surface (sidebar) → pipeline trigger

| Control | Type | Default | Recompute effect |
|---|---|---|---|
| Ticker multiselect | list, presuggested 15 | all 15 | Full pipeline; ticker order does not matter (dedup + `.JK` normalize first) |
| History years | slider 4–10 | 7 | Full pipeline; new cache key (§5) |
| Weight cap | slider 5–60% | 30% | Optimizer + backtest re-run; download reused |
| Risk-free rate | number input 0–15% | 7.4% (verified 22 Oct 2026) | Sharpe recalc + backtest re-run; download reused |
| Cost per turnover | number input 0–2% | 0.25% | Backtest re-run only; download, metrics, frontier reused |
| Clear-cache button | button | — | Cache bust, full pipeline on next run |

Minimal-recompute rule: a change triggers the smallest suffix of the pipeline that could produce a different number. Download is the most expensive step, so only years/tickers bust it.

## 3. Compute pipeline (per triggered run)

```
1. normalize_tickers      (data.py)     ── .JK suffix, dedup; bare "BBCA" → "BBCA.JK"
2. count = len(accepted)                 ── if count < 5  → state E, stop        (PRD F1 guard)
3. load_prices            (data.py)     ── st.cache_data hit → instant
   ├─ success: (frame, dropped)          ── dropped non-empty → state D warnings
   └─ failure: DownloadError             ── state F, stop
4. post-drop count < 5                    ── state E naming the dropped-to-death tickers
5. monthly_returns         (data.py)     ── month-end resample, aligned dates across tickers
6. stock_metrics + correlation           ── annualized: μ × 252, σ × √252, Sharpe, max-DD
7. mu, cov annualized (single normalization point, TRD §3)
8. min_vol_weights         (optimizer)   ── frontier floor point
9. max_sharpe_weights      (optimizer)   ── frontier star point, accent-marked
10. efficient_frontier     (optimizer)   ── 50-target SLSQP grid
11. walk_forward           (backtest)    ── monthly re-opt on 3y trailing window,
                                            turnover = Σ|Δw|, cost = rate × turnover,
                                            equal-weight rebalanced same rule
12. ^JKSE series aligned to backtest window (raw price return, no costs)
13. render story scroll (§4)            ── state C
```

Failure at any step raises through to the state handler for that step (E/F/G above); the render never shows a half-computed page — sections appear only when their inputs exist.

## 4. Rendered page order (state D/C)

Exactly the design-brief §4 story scroll; mapped to data:

1. Header strip + last data date (from cache entry timestamp)
2. Verdict block ← backtest result metrics
3. Caveats (four fixed lines, always visible)
4. Per-stock metrics table ← step 6
5. Correlation heatmap + auto one-line read-out ← step 6
6. Efficient frontier + min-vol/max-Sharpe weight bars ← steps 8–10
7. Backtest equity curves, drawdown, cost-drag line ← steps 11–12
8. Methodology strip with repo links

## 5. Cache behavior

- Key: `(accepted tickers, years, calendar month)` — a new calendar month silently refreshes for walk-forward realism; TTL 12h.
- Hit → state B is nearly invisible; sidebar changes to cap/rf/cost never re-download.
- "Clear cache" button → full pipeline including download on the next run.

## 6. Edge cases

| Case | Handling |
|---|---|
| User types dead/wrong ticker | Step 3 drops + warns (state D); pipeline continues on survivors |
| Yahoo rate-limits the batch | `DownloadError` → state F; cache makes the retry free after a wait |
| All 15 suggestions dropped | state E; message shows survivors count, not a stack trace |
| Cap slider below 100/n | UI guard + optimizer guard both raise state G; never a partial frontier chart |
| Equal-weight & strategy identical (cap ≥ 100%) possible | Both rendered; comparison still valid, note appears in methodology strip |
| RF/cost inputs cleared to empty | Input kwargs prevent blank (min 0, fixed step); empty string impossible |
| Turnover 0 (weights unchanged month to month) | Cost = 0 that month, perfectly ordinary line in the cost series |
