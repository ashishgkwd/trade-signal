# PRD: Unified Trade Signal Indicator (Pine Script)

---

## A. Product Overview

### Problem Statement
A retail trader manually scans all 50 Nifty 50 stocks daily (~50 minutes, ~1 minute per chart) to find trade setups that meet criteria across four separate TradingView indicators. This process is slow, error-prone, and requires keeping four indicator windows open simultaneously per chart.

### Target User
A single retail trader using TradingView (any paid plan), trading Nifty 50 equities on a daily Heikin Ashi chart.

### Value Proposition
A single Pine Script indicator file that encodes all four indicator signal criteria. When applied to a chart — or when TradingView alerts are set on all 50 Nifty stocks — it fires only when every criterion aligns simultaneously, eliminating manual scanning entirely.

### Scope
One `.pine` file. No backend, no database, no Python. All logic runs inside TradingView.

---

## B. Technical Design

### B1. Unified Pine Script Indicator

**Target version: Pine Script v5.** Rationale: v5 is mature, stable, and fully supported across all TradingView plans. Pine Script v6 is recent and adoption/documentation is still sparse; migration is trivial later.

The file will be structured in seven sections:

```
//@version=5
indicator("Unified Trade Signal", overlay=true, max_bars_back=500)

// ── Section 1: Inputs ──────────────────────────────────────────────────
// ── Section 2: NWE Calculation ────────────────────────────────────────
// ── Section 3: EMA Calculation ────────────────────────────────────────
// ── Section 4: ATF Calculation ────────────────────────────────────────
// ── Section 5: RSI Calculation ────────────────────────────────────────
// ── Section 6: Signal Logic ───────────────────────────────────────────
// ── Section 7: Plots & Alerts ─────────────────────────────────────────
```

#### Input Parameters (all configurable, with specified defaults)

| Parameter | Input Name | Default | Notes |
|-----------|-----------|---------|-------|
| NWE Bandwidth | `nwe_h` | `8.0` | Gaussian kernel bandwidth |
| NWE Multiplier | `nwe_mult` | `3.0` | Envelope width multiplier |
| NWE Source | `nwe_src` | `close` | Price source |
| NWE Repainting | `nwe_repaint` | `true` | See §F on repainting implications |
| Fast EMA period | `ema_fast` | `13` | |
| Slow EMA period | `ema_slow` | `48` | |
| ATF Deviation Multiplier | `atf_mult` | `2.0` | Channel band width |
| RSI Period | `rsi_len` | `14` | |
| RSI Bullish Lower Bound | `rsi_lo` | `30` | For bullish zone check |
| RSI Bullish Upper Bound | `rsi_hi` | `40` | For bullish zone check |
| RSI Bearish Lower Bound | `rsi_bear_lo` | `60` | For bearish zone check |
| RSI Bearish Upper Bound | `rsi_bear_hi` | `70` | For bearish zone check |
| RSI Trend Lookback | `rsi_lookback` | `3` | Bars to confirm RSI direction |
| Show Indicator Overlays | `show_overlays` | `false` | Toggle NWE bands, EMAs, ATF line |

---

#### B1a. NWE Calculation (from `indicators/nadariya-watson-envelope.pine`)

Extract the Gaussian kernel regression logic directly. The core function:

```pine
gauss(x, h) => math.exp(-(math.pow(x, 2) / (math.pow(h, 2) * 2)))
```

**Repainting mode** (`nwe_repaint = true`): Compute the Nadaraya-Watson estimate for the current bar by looping over the last `n` bars with Gaussian weighting. Compute the Smoothed Absolute Error (`sae`) as the envelope half-width.

```pine
nwe_upper = nwe_val + nwe_mult * sae
nwe_lower = nwe_val - nwe_mult * sae
```

**Non-repainting mode**: Use the endpoint / static coefficient method from the source file.

**Signal assignment (breakout interpretation per user spec):**
```pine
nwe_bull = ta.crossover(close, nwe_upper)   // close breaks above upper band
nwe_bear = ta.crossunder(close, nwe_lower)  // close breaks below lower band
```

> **Design decision — NWE signal direction:** The original LuxAlgo indicator treats `close > upper` as a mean-reversion sell and `close < lower` as a mean-reversion buy. The user explicitly requires the opposite (breakout interpretation). The unified script follows the user's specification. This is documented in code comments.

---

#### B1b. EMA Calculation (from `indicators/ema-cross-strategy.pine`)

```pine
fast_ema = ta.ema(close, ema_fast)
slow_ema = ta.ema(close, ema_slow)

ema_bull = fast_ema > slow_ema   // long bias confirmed
ema_bear = fast_ema < slow_ema   // short bias confirmed
```

Position check (`fast > slow`) is used rather than a strict crossover check — simpler and more reliable as a bias filter. The crossover/crossunder conditions from the source file are available as an optional stricter mode if needed later.

---

#### B1c. ATF Calculation (from `indicators/adaptive-trend-finder.pine`)

The ATF uses logarithmic linear regression over multiple periods (19 lookback candidates) and selects the one with the highest Pearson's R. Extract the `calcDev()` function and period-selection loop verbatim from the source file.

Trend direction signal:
```pine
atf_bull = detectedSlope > 0   // upward trend
atf_bear = detectedSlope < 0   // downward trend
```

> **Performance note:** The multi-period regression loop is computationally heavy. Use `max_bars_back=500`. If TradingView throws a "Script execution time limit" error, reduce the number of test periods or expose a fixed-period mode as an input.

---

#### B1d. RSI Calculation

Use TradingView's built-in `ta.rsi()` directly — no need to copy from the RSI source file:

```pine
rsi_val = ta.rsi(close, rsi_len)
```

**Bullish RSI condition** — RSI in the 30–40 zone AND rising:
```pine
rsi_in_zone = rsi_val >= rsi_lo and rsi_val <= rsi_hi
rsi_rising  = rsi_val > rsi_val[rsi_lookback]
rsi_bull    = rsi_in_zone and rsi_rising
```

**Bearish RSI condition** — RSI in the 60–70 zone AND falling:
```pine
rsi_in_bear_zone = rsi_val >= rsi_bear_lo and rsi_val <= rsi_bear_hi
rsi_falling      = rsi_val < rsi_val[rsi_lookback]
rsi_bear         = rsi_in_bear_zone and rsi_falling
```

> **Assumption:** The requirements define RSI 30–40 rising for bullish but do not specify a bearish RSI condition. RSI 60–70 falling is the symmetrical mirror and is used as the default. Confirm before final build.

---

#### B1e. Combined Signal Logic

```pine
// Bullish: ALL four must be true simultaneously
bull_signal = nwe_bull and ema_bull and atf_bull and rsi_bull

// Bearish: ALL four must be true simultaneously
bear_signal = nwe_bear and ema_bear and atf_bear and rsi_bear
```

---

### B2. Signal Output

#### Primary signal markers (always visible, no toggle):
```pine
plotshape(bull_signal, title="Bull Signal", style=shape.triangleup,
          location=location.belowbar, color=color.lime, size=size.normal)

plotshape(bear_signal, title="Bear Signal", style=shape.triangledown,
          location=location.abovebar, color=color.red, size=size.normal)
```

#### Optional background highlight:
```pine
bgcolor(bull_signal ? color.new(color.lime, 85) : bear_signal ? color.new(color.red, 85) : na)
```

#### Overlay plots (shown only when `show_overlays = true`):
- NWE upper band — teal line
- NWE lower band — red line
- Fast EMA (13) — orange line
- Slow EMA (48) — blue line
- ATF trend line — purple line

#### RSI display:
Show current RSI value in a small table in the chart corner (avoids the `overlay=true` / separate-pane conflict). The table updates each bar and shows whether RSI is in the bullish or bearish zone.

---

### B3. Alert Conditions

```pine
alertcondition(bull_signal,
    title   = "Bull Signal — All Criteria Met",
    message = "{{ticker}} — Bullish signal on Heikin Ashi Daily: NWE breakout above upper band, EMA 13 > EMA 48, ATF uptrend, RSI 30–40 rising.")

alertcondition(bear_signal,
    title   = "Bear Signal — All Criteria Met",
    message = "{{ticker}} — Bearish signal on Heikin Ashi Daily: NWE breakdown below lower band, EMA 13 < EMA 48, ATF downtrend, RSI 60–70 falling.")
```

`{{ticker}}` is a TradingView dynamic placeholder that resolves to the symbol name at alert-fire time.

**How this replaces manual scanning:** Add the indicator to one chart, then use TradingView's "Alert on Multiple Symbols" feature (Pro+ / Premium) to cover the entire Nifty 50 watchlist with a single alert setup. When criteria align on any stock, a push notification / email / webhook fires automatically.

---

### B4. TradingView Screener Compatibility

**Not feasible.** TradingView's built-in screener supports only native built-in indicators. Custom Pine Script indicators cannot be added as screener filter criteria — this is a hard platform limitation.

**Confirmed alternative:** The `alertcondition()` approach in §B3 is the correct and complete solution. On Pro+ / Premium, one alert with "Any Symbol from Watchlist" achieves the same outcome as a screener scan, with zero daily manual effort.

---

## C. Usage Instructions

### Step 1 — Add the indicator to TradingView
1. Open TradingView → Pine Script Editor (bottom panel).
2. Paste the full contents of `indicators/unified-trade-signal.pine`.
3. Click **Save**, then **Add to chart**.

### Step 2 — Configure the chart
1. Click the chart type selector (top-left toolbar) → select **Heikin Ashi**.
2. Set the timeframe to **1D** (daily).
3. Load any Nifty 50 stock ticker.

### Step 3 — Set up alerts across Nifty 50
1. Click the **Alert** button (clock icon, top toolbar) → **Create Alert**.
2. Under **Condition**, select the unified indicator name → choose **"Bull Signal — All Criteria Met"** or **"Bear Signal — All Criteria Met"**.
3. *(Pro+ / Premium only)* Select **"Any Symbol from Watchlist"** and choose your saved Nifty 50 watchlist.
4. Set notification type — push notification recommended for real-time use.
5. Click **Create**.

> **Basic / Pro plan users:** The "Any Symbol from Watchlist" option is not available. Create one alert per stock (50 total). This is a one-time ~10-minute setup, not a daily task.

### Step 4 — Interpret signals
| Marker | Meaning |
|--------|---------|
| Green triangle below bar | All four bullish criteria met — potential long entry |
| Red triangle above bar | All four bearish criteria met — potential short entry |
| No marker | Criteria not fully aligned — no trade |

Toggle `show_overlays = true` to display individual indicator lines and diagnose which criterion is lagging on a near-miss.

---

## D. Validation & Testing Strategy

### Method
Apply the unified indicator alongside all four individual indicators on the same Heikin Ashi daily chart. A signal from the unified indicator must correspond exactly to a bar where every individual indicator confirms its respective condition.

### Validation Checklist (test on RELIANCE, INFY, TCS)
- [ ] Bull signal fires → confirm: NWE close crossed above upper band, EMA 13 > EMA 48, ATF slope > 0, RSI in 30–40 and rising.
- [ ] Bear signal fires → confirm mirror of the above.
- [ ] Only 3 of 4 conditions met on a bar → confirm no marker appears.
- [ ] With `nwe_repaint=true`, NWE line may shift retroactively on historical bars — note this behaviour; confirm alerts fire at bar close only.
- [ ] RSI exactly at 30 or 40 — boundary is inclusive (`>=`, `<=`); verify signal fires.
- [ ] ATF slope = 0 (sideways market) → no signal fires.

### Edge Case Table

| Scenario | Expected Behaviour |
|----------|--------------------|
| 3 of 4 criteria met | No marker |
| NWE repaints mid-bar | Signal may flicker; alert fires only on bar close — acceptable |
| RSI = 30 exactly | Included in bullish zone (≥ 30) |
| RSI = 40 exactly | Included in bullish zone (≤ 40) |
| ATF slope = 0 | Neither bull nor bear — no signal |
| ATF low Pearson's R (weak trend) | Slope sign is still used; a minimum Pearson's R threshold input is a future enhancement |
| Gap open past NWE band | `ta.crossover` / `ta.crossunder` fires on the gap bar — correct |

---

## E. Extensibility

**Swapping an indicator:** Each indicator occupies a clearly delimited section. To replace RSI with MACD, rewrite Section 5 and update the `rsi_bull` / `rsi_bear` variables in Section 6. No other sections are affected.

**Changing the stock universe:** Update the TradingView watchlist. The script is stock-agnostic.

**Changing the timeframe:** Change the chart timeframe. No hardcoded timeframe references exist in the script. For multi-timeframe signals, wrap calculations with `request.security()` — straightforward to add but out of current scope.

---

## F. Constraints & Assumptions

| Item | Detail |
|------|--------|
| Timeframe | Daily only. User must set chart to 1D manually. |
| Candle type | Heikin Ashi. `close`, `open`, `high`, `low` automatically use HA values when the chart type is set to HA. No manual HA computation needed in the script. |
| Repainting | NWE with `nwe_repaint=true` repaints — historical signals may shift after new bars form. Set `nwe_repaint=false` for backtesting to avoid lookahead bias. For live alerts (bar close), repainting is acceptable. |
| TradingView plan | Multi-symbol alerts require **Pro+ or higher**. Basic/Pro: 50 individual alert setups needed (one-time). ATF's regression loop may approach execution time limits on very long bar histories. |
| Pine Script version | v5. Upgrading to v6 later is a minor syntactic change. |
| Alert latency | TradingView alert delivery can lag by seconds to minutes. Acceptable for a daily timeframe. |
| NWE signal direction | Breakout interpretation used (above upper = bullish), differing from the original LuxAlgo mean-reversion convention. Intentional per user spec; documented in code comments. |
| Bearish RSI condition | Assumed as RSI 60–70 falling (mirror of bullish 30–40 rising). **Confirm with user before final build.** |

---

## G. Implementation Milestones

### Milestone 1 — Core indicator logic
**Deliverable:** `indicators/unified-trade-signal.pine`

- All four computations integrated: NWE (Gaussian kernel regression), EMA (fast/slow), ATF (log regression + Pearson R), RSI (built-in `ta.rsi()`).
- `bull_signal` and `bear_signal` boolean variables implemented.
- Green / red triangle markers plotted.
- All input parameters exposed with correct defaults.
- Verified on RELIANCE and INFY: every marker matches a bar where all four individual indicators confirm simultaneously. No false positives in a 6-month lookback.

### Milestone 2 — Alert conditions
**Deliverable:** Updated `.pine` file with `alertcondition()` blocks.

- Both `alertcondition()` calls added with `{{ticker}}` in the message.
- User sets a test alert on one stock; confirms a notification fires on the next valid signal.
- Alert setup instructions for the full Nifty 50 watchlist documented (see §C Step 3).

### Milestone 3 — Polish
**Deliverable:** Final, clean `.pine` file.

- `show_overlays` toggle wires up NWE bands, EMAs, ATF trend line.
- RSI value displayed in a corner table with zone indicator.
- Inline comments explain each section and non-obvious logic choices.
- Validation pass on 5+ Nifty 50 stocks.
- All edge cases from §D tested and confirmed.

---

## Source Files Reference

| Indicator | Source File |
|-----------|------------|
| Nadaraya-Watson Envelope | `indicators/nadariya-watson-envelope.pine` |
| EMA Cross Strategy | `indicators/ema-cross-strategy.pine` |
| Adaptive Trend Finder | `indicators/adaptive-trend-finder.pine` |
| RSI | `indicators/relative-strength-index.pine` (reference only — use `ta.rsi()` built-in) |
| **Output** | `indicators/unified-trade-signal.pine` *(to be created)* |
