# Trade Signal

A [TradingView](https://www.tradingview.com/) Pine Script project for a **unified trade signal** indicator. It combines four technical criteria—Nadaraya–Watson envelope (NWE), EMA trend, adaptive trend finder (ATF), and RSI—so a single script can replace manually checking multiple indicators on each chart when scanning Nifty 50 names on a daily **Heikin Ashi** timeframe.

**Disclaimer:** This repository is tooling and documentation for technical analysis ideas. It is not financial advice. Trading involves risk.

---

## What it does

A **bullish** signal requires all of the following on the same bar:

- NWE: price breaks out (close crosses above the upper band), per breakout interpretation (not LuxAlgo’s default mean-reversion framing).
- EMA: fast EMA is above slow EMA.
- ATF: positive slope on the adaptive trend line.
- RSI: in the bullish band (default 30–40) and **rising** over the lookback window.

A **bearish** signal is the symmetric mirror: NWE cross under lower band, fast EMA below slow, ATF negative slope, RSI in the bearish band (default 60–70) and **falling**.

See [PRD.md](PRD.md) for the full product specification and design notes.

---

## Repository layout

| Path | Purpose |
|------|---------|
| [indicators/unified-trade-signal.pine](indicators/unified-trade-signal.pine) | Main deliverable: combined indicator (alerts, optional overlays, validation toggles). |
| `indicators/*.pine` | Reference implementations: NWE, EMA strategy, ATF, RSI (and related variants). |
| [PRD.md](PRD.md) | Product requirements and technical design. |
| [inputs/](inputs/) | Original problem statement and PRD-generation prompt (optional context). |
| [CLAUDE.md](CLAUDE.md) | Notes for AI-assisted editing (logic summary, defaults, validation checklist). |

There is **no build step**: Pine runs in TradingView’s editor. Edit `.pine` files locally, then paste into TradingView to validate.

---

## How to use

1. Open [indicators/unified-trade-signal.pine](indicators/unified-trade-signal.pine) and copy its contents into **TradingView → Pine Editor → New**.
2. Set the chart to **daily** Heikin Ashi (timeframe and bar type are assumed by the script; they are not forced in code).
3. Apply the indicator to a symbol (e.g. Nifty 50 constituents). Use **Alerts** if you want notifications across many tickers.
4. Optionally cross-check against the separate reference indicators in `indicators/` on the same symbol and settings.

**Repainting:** With NWE repainting-style smoothing enabled (default in the unified script), behavior matches the LuxAlgo-style repainting mode—suitable for bar-close alerts; treat backtests with care.

---

## Default parameters (summary)

| Group | Defaults |
|-------|----------|
| NWE | Bandwidth `8.0`, multiplier `3.0`, source `close`, repainting smoothing on |
| EMA | Fast `13`, slow `48` |
| ATF | Deviation multiplier `2.0`; period can auto-select (20–200) or be fixed |
| RSI | Length `14`; bullish zone `30–40`; bearish zone `60–70`; trend lookback `3` bars |

Display overlays (NWE, EMA, ATF) are off by default; enable them in the script inputs when you want visuals on the chart.

---

## Validation

Suggested manual checks: compare the unified script to the four reference indicators on the same **daily Heikin Ashi** chart; try names such as RELIANCE, INFY, TCS. Probe boundary RSI values (e.g. 30, 40, 60, 70) and confirm repainting behavior matches your expectations.
