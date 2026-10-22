# Design Brief — IDX Portfolio Optimizer

Companion to [PRD.md](PRD.md) (what/why) and [TRD.md](TRD.md) (how). This document owns how it looks and the style rules it must never break. Design direction set via interview, Oct 2026.

## 1. One-line vision

A dark, disciplined quant terminal that treats the visitor as an analyst, not a customer — money-and-rigor, chart-first, zero decoration, and honest to the pixel about what the backtest can and cannot prove.

## 2. Audience & proof burden

- **Primary:** a recruiter or hiring engineer scanning the deployed app for ~2 minutes, in English.
- **Reading mode:** top-to-bottom scan, one section landing per screen. Every panel must survive being screenshotted alone — each section is a portfolio exhibit.
- **Copy tone:** institutional-brief. Short declaratives, numbers first, no marketing voice, no exclamation marks. One dry aside per section maximum.

## 3. Art direction — "quant terminal"

Alongside the Streamlit-native framing (which limits some choices; see §8), the app commits to these direction rules:

**Surface.** Deep near-black navy background, slightly raised panel tone for cards/tables, 1px hairline borders in a low-contrast line color. No drop shadows doing structural work; no glassmorphism; no rounded-blob containers.

**Type.** Tabular monospaced numerals everywhere numbers appear in columns (JetBrains Mono or IBM Plex Mono family) — aligned digit columns are the credibility trick. Labels and body in the matching sans (IBM Plex Sans). Headings die the moment they are not working: no display pyrotechnics.

**Color.** Monochrome discipline: grayscale plus exactly **one accent — amber-gold** (`#E8B84B` tested against dark navy) marking the max-Sharpe portfolio, the active selection, and key deltas. Red exists **only** for drawdown. The obvious blue/purple gradient look is banned — it is the exact average the direction rejects.

**Charts (plotly, styled identical across the app):**

| Series | Treatment |
|---|---|
| Optimized strategy | Accent solid, 2px |
| Equal-weight benchmark | Muted gray solid, 1.5px |
| ^JKSE | Dashed gray, 1.5px |
| Min-vol portfolio | Dimmer accent or open marker |
| Efficient frontier | Single hairline curve, small markers at grid points |
| Correlation heatmap | **Single-hue sequential** (deep navy → amber), never blue-to-red; cell values rendered as text at readable size |
| Drawdown | Fill under curve in the drawdown red family |

Common rules: no 3D, no animations beyond Streamlit defaults, no dual y-axes, axis labels never truncated, gridlines faint, legend horizontal, currency on the y-axis in plain `%`.

## 4. Layout — the story scroll

One long page, narrative order (PRD F1–F12), sidebar fixed:

0. **Side bar (always visible):** ticker multiselect, years slider, cap slider, rf input, cost input, clear-cache button. Grouped with hairlines, mono values.
1. **Header strip:** project name, one-line thesis ("walk-forward optimized IDX portfolios, backtest costs included"), last data date. No hero image, no centered giant headline.
2. **Verdict block (first-class):** the backtest headline numbers — strategy CAGR/vol/Sharpe/max-DD vs equal-weight vs ^JKSE — as the first thing with any math on it. The interview story leads; the derivation follows.
3. **Caveats, front and center:** directly under the verdict, visible by default — the four honesty points (constant rf, ^JKSE excludes dividends, survivorship bias, Yahoo data quality) as tight labeled lines, not an expander graveyard. Opt-out icon for the truly disinterested remains in the sidebar.
4. **Per-stock metrics table:** mono numerals, right-aligned, hairline row rules only.
5. **Correlation heatmap** with a one-line read-out ("highest pair correlation: X").
6. **Efficient frontier** with min-vol and max-Sharpe pinned and their weight bars beside the chart.
7. **Backtest equity curves + drawdown**, including the cumulative cost-drag line — the costs are up front, not footnoted.
8. **Methodology strip:** 4 short lines linking to PRD/TRD in the repo.

Screen order is a prose essay: each section ends by setting up the next one with exactly one sentence.

## 5. Explicit anti-slop ban list

No gradients. No icon-card rows. No centered hero over three gray lines. No purple/blue default palettes. No emoji in the UI. No "AI-made" captions, no glassmorphism, no neon glow, no skeleton shimmer, no confetti, no rounded-2xl-everything, no lorem anything, no chartjunk like dual axes or smoothed series that change the data's story.

## 6. Accessibility & ergonomics

- Contrast: body text ≥ 4.5:1 on the panel color; the accent is never the only encoding (mark max-Sharpe with label + accent, not color alone).
- Touch/click targets ≥ 40px in the sidebar; slider labels show live values in mono.
- Every chart is described by a one-sentence takeaway caption; a recruiter reading only captions gets the full story.

## 7. Tokens (single source of truth for M4)

| Token | Value | Use |
|---|---|---|
| `bg` | `#0B0E14` | Page background |
| `surface` | `#11151F` | Panels, tables |
| `border` | `#232A38` | Hairlines |
| `fg` | `#E6EAF2` | Primary text |
| `muted` | `#8B93A7` | Secondary text, benchmark series |
| `accent` | `#E8B84B` | Max-Sharpe, active states, key deltas — nothing else |
| `drawdown-red` | `#D9646B` | Drawdown fills/lines only |

Streamlit theme config (`theme.base="dark"` + custom element colors via `config.toml`), plotly template matching these tokens, both derived from this table only — no second palette introduced later.

## 8. Known constraints

- Streamlit limits true typographic control; §3/§4 are enforceable inside Streamlit (charts, tables, layout, copy) — custom CSS via `st.markdown` unsafe HTML is allowed only for tokens/hairline rules; where Streamlit refuses (hero typography), the terminal discipline is to adjust, not to hack.
- The single accent across heat-sequential + max-Sharpe requires verifying the heatmap remains readable at the amber end; fallback is a desaturated amber→navy ramp.

## 9. Definition of designed

1. A first-time visitor reaches the verdict numbers within one screenful of scrolling.
2. Every number in every table aligns in mono digits.
3. The heatmap reads as one hue family.
4. The entire page contains zero items from the §5 ban list, verifiable by a reviewer in one pass.
5. Both glossy screenshots in the README communicate "this person makes deliberate, defensible design decisions."
