# Session Log

One entry per working session, newest last. Conventions: milestone states verified, not assumed; "Next" = exact entry point of the following session.

## Session 1 — 22 Oct 2026

**Decisions made (with owner, grilling rounds):**
- Stack: Streamlit; layout: 4 modules (`app.py`, `data.py`, `optimizer.py`, `backtest.py`) + requirements + tests
- Location `E:\projects\idx-portfolio`; git repo day one, commit per milestone
- History 7y default (slider 4–10), walk-forward 3y lookback, monthly rebalance
- Long-only, cap 30% default (UI input), min 5 stocks
- Backtest: walk-forward (honest), cost 0.25% per turnover unit (IDX broker blend 0.15 buy / 0.25 sell), benchmarks ^JKSE + equal-weight (same costs)
- rf = 7.4% default (Indonesia 10Y verified live 22 Oct 2026: 7.41–7.44%) — constant-rate caveat documented
- Ticker list finalized (15 names, 3 banks), auto-`.JK`, auto-drop with warning
- UI: quant-terminal direction, monochrome + amber `#E8B84B`, mono numerals, story scroll, caveats front and center

**Docs created (docs/):** PRD.md, TRD.md, design-brief.md, app-flow.md, backend-schema.md, implementation-plan.md (M0–M6), AGENTS.md (doc pointers + skill routing)

**Environment changed:** skills installed to `~\.config\opencode\skills\`: `taste-skill` clone + junction `design-taste-frontend`; `ui-ux-pro-max` installed via its npx CLI + 6 junctions (`ui-ux-pro-max`, `design`, `design-system`, `ui-styling`, `banner-design`, `brand`, `slides`); `gh` 2.103.0 via winget, authenticated as renngraphic. Skill junctions visible to opencode after restart.

**Build progress:** M0 complete — venv on Python 3.14.6 (no 3.12 fallback needed, wheels clean), requirements.txt pinned (streamlit 1.66.1, yfinance 1.7.0, plotly 7.1.0, pytest 9.1.1, pandas 3.1.0, numpy 2.5.4, scipy 1.18.1), `.streamlit/config.toml` tokens, hello app.py, Streamlit HTTP-200 smoke test, empty pytest run, repo live at github.com/renngraphic/idx-portfolio (public, main).

**Commits:** `cc463b6` M0 scaffold, `96cfa51` AGENTS.md.

**Learning mode:** onboarded (Beginner × Beginner, defaults); state at `C:\Users\naren\.vibe-wise\`. Next checkpoints land at M1 (data-cleaning tolerance rules) and M2 (optimizer design reasoning).

**Next session starts here:** M1 `data.py` — checkpoint first: missing-data drop tolerance, gap-filling policy, then implement loaders against live Yahoo probe of all 15 tickers (ADRO split risk lives here).

**Open items:** global git identity still unset (repo-local configured); Streamlit Cloud deploy deferred; commit author rewrite undecided beyond M0's amend.
