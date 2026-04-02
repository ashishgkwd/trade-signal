# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **TradingView Pine Script** project. The goal is a single unified indicator (`indicators/unified-trade-signal.pine`) that scans Nifty 50 stocks for trade setups matching all four criteria simultaneously, replacing a ~50-minute daily manual chart review.

## No Build System

There is no build, test, or lint toolchain. Pine Script runs entirely inside TradingView's browser-based editor. Development workflow:
1. Write/edit `.pine` files locally
2. Copy-paste into TradingView Pine Script Editor
3. Validate visually against the four individual source indicators

## Repository Layout

```
indicators/          # Source indicator files (reference implementations)
  nadariya-watson-envelope.pine   # NWE: Gaussian kernel regression (LuxAlgo, v5)
  ema-cross-strategy.pine         # EMA cross (v4, strategy format)
  adaptive-trend-finder.pine      # ATF: log regression, best Pearson-R period (v6, 369 lines)
  relative-strength-index.pine    # RSI reference (v6)
inputs/              # Original problem statement and PRD generation prompt
PRD.md               # Full product requirements — read this before making changes
```

The deliverable file `indicators/unified-trade-signal.pine` does **not yet exist** — it is the primary task.

## Combined Signal Logic

All four conditions must be true simultaneously:

```
Bullish = NWE_crossover  AND EMA(13) > EMA(48) AND ATF_positive_slope AND RSI in [30,40] rising
Bearish = NWE_crossunder AND EMA(13) < EMA(48) AND ATF_negative_slope AND RSI in [60,70] falling
```

**NWE signal direction is inverted from LuxAlgo default** — this script uses breakout interpretation (close crosses above upper band = bullish), not mean-reversion.

## Key Design Decisions (from PRD.md)

- **Target version:** Pine Script v5 (not v6, even though some source files use v6 syntax)
- **Candle type:** Heikin Ashi — TradingView handles this via chart settings, no manual calculation needed
- **Timeframe:** Daily — enforced by user chart setup, no `timeframe.change()` needed
- **NWE repainting:** Enabled (`nwe_repaint = true`). Acceptable for bar-close alerts; not suitable for backtesting.
- **ATF implementation:** Loop over 19 periods (20–200 bars), pick highest Pearson's R. Computationally heavy; avoid adding more iterations.
- **RSI bearish zone:** 60–70 falling (symmetric mirror of bullish 30–40 rising) — confirm with user if ambiguous

## Default Input Parameters

| Parameter | Default | Notes |
|-----------|---------|-------|
| `nwe_h` | 8.0 | Gaussian bandwidth |
| `nwe_mult` | 3.0 | Envelope width multiplier |
| `ema_fast` / `ema_slow` | 13 / 48 | — |
| `atf_mult` | 2.0 | Channel band deviation |
| `rsi_len` | 14 | — |
| `rsi_lo` / `rsi_hi` | 30 / 40 | Bullish RSI zone |
| `rsi_bear_lo` / `rsi_bear_hi` | 60 / 70 | Bearish RSI zone |
| `rsi_lookback` | 3 | Bars to confirm RSI direction |
| `show_overlays` | false | Toggle NWE/ATF visual overlays |

## Validation

Test unified indicator against the four individual source indicators on the same Heikin Ashi daily chart. Reference stocks: RELIANCE, INFY, TCS. Markers should only appear when all four individual indicators confirm simultaneously. Test boundary RSI values (exactly 30, 40, 60, 70) and repainting behavior.
