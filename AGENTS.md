# AGENTS.md

IDX Portfolio Optimizer — Streamlit app that builds optimized IDX stock portfolios and proves them with an honest, look-ahead-free walk-forward backtest. Recruiter-facing repo. All text user-facing: English.

## Learning mode

The owner is a Beginner (programming and stack) and learning end to end. Session state lives in `C:\Users\naren\.vibe-wise\` (profile, progress, project map) — read it when resuming, never delete or rewrite history. Follow the `learn` skill at milestone checkpoints: ask the owner's reasoning before writing consequential code, get an Implementation checkpoint approval per step. The owner already made the design decisions; do not relitigate them or re-plan the project.

## Design docs (read the one you need)

All decisions live in `docs/`. Reach the right one by trigger:

- **Scope, features, non-goals** → `docs/PRD.md`
- **How modules behave, errors must surface** → `docs/TRD.md`
- **Exact data shapes, invariants, function signatures** → `docs/backend-schema.md`
- **Runtime states, pipeline order, edge cases** → `docs/app-flow.md`
- **Visual rules, color tokens, ban list** → `docs/design-brief.md`
- **What to build next and its acceptance check** → `docs/implementation-plan.md`

A doc change means the owner changed a decision; never edit them to match code you wrote.

## Environment facts (not recoverable from any config)

- Python runs only through the venv: `.\.venv\Scripts\python.exe` — there is no `python` on PATH (Windows Store alias stub); use `py -3.14` only for venv creation.
- Git identity is configured repo-locally (`renngraphic`, noreply email). Do not change; do not set global config.
- `gh` 2.103.0 at `C:\Program Files\GitHub CLI\` is authenticated as `renngraphic`.
- Risk-free default 7.4% = Indonesia 10Y yield verified 22 Oct 2026; everything else (ticker list, fees, history depth) came from the owner's stated preferences in `docs/PRD.md`.

## Commands

```
.\.venv\Scripts\streamlit.exe run app.py    # serve app (headless smoke: --server.headless true)
.\.venv\Scripts\python.exe -m pytest        # run tests
```

## Build rules

- Milestones in `docs/implementation-plan.md` order; one commit per milestone, Conventional Commits style (see `git log` for precedent). Run the full test suite before committing; commit green only.
- Push after each milestone commit — origin is `github.com/renngraphic/idx-portfolio`, branch `main`.
- `optimizer.py` and `backtest.py` stay free of streamlit imports (headless pytest, TRD §2). `data.py` is the only streamlit-importing module.
- Streamlit reruns the script per interaction; nothing derived goes outside `st.cache_data` in `data.py`.

## Skill routing

Reach for the skill that owns the task:

- **UI, plotly charts, theming (M4)** → `ui-ux-pro-max` (start from its design-system step) + `design-taste-frontend` for the final anti-slop pass
- **Writing tests for optimizer/backtest math (M5)** → `tdd`
- **Commit messages** → `caveman-commit`
- **Review before a milestone commit** → `caveman-review` (compact inline) or `code-review` (parallel standards+spec subagents)
- **Locating code or doc passages** → `cavecrew-investigator` instead of reading whole files inline
- **External facts: yfinance quirks, Yahoo data, Streamlit Cloud behavior** → `research` skill or websearch before guessing
- **A bug surviving two fix attempts** → `diagnosing-bugs`
- **A design question about module seams** → `codebase-design`
- **After finishing a milestone** → `retro`
- **Session running long / context pressure** → `handoff` before instead of dropping context
- **Streamlit Cloud deploy (human-only provisioning steps)** → `wizard`
- **PR bodies** → `pr`

Skip for this project (planning is complete): `grilling`, `to-spec`, `to-tickets`, `wayfinder`, `scaffold-exercises`, `migrate-to-shoehorn`.
