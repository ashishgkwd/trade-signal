# TASKS.md — Unified Trade Signal Indicator

Implementation plan for `indicators/unified-trade-signal.pine`.
Follow milestones in order. Do not skip to polish (Milestone 3) before Milestone 1 is validated.

---

## Pre-Implementation: Confirm Open Questions

- [ ] **Confirm bearish RSI condition with user.** PRD assumes RSI 60–70 falling (symmetric mirror of bullish 30–40 rising), but this is marked as an assumption. Confirm before writing Section 5. Default to 60–70 falling if unreachable.

---

## Milestone 1 — Core Indicator Logic

Goal: A working `.pine` file that plots bull/bear markers. Verified against individual source indicators on RELIANCE and INFY.

### Task 1.1 — File skeleton & inputs (Section 1)

Create `indicators/unified-trade-signal.pine` with:
- `//@version=5` declaration and `indicator()` call (`overlay=true`, `max_bars_back=500`)
- Seven section comment stubs (Sections 1–7 as specified in PRD §B1)
- All 14 input parameters with exact defaults from PRD §B1 table
- Group inputs logically: NWE group, EMA group, ATF group, RSI group, Display group

### Task 1.2 — NWE calculation (Section 2)

**Important constraint identified from source file:** The repainting mode in `nadariya-watson-envelope.pine` runs entirely inside `barstate.islast` using a line-drawing array approach. It does **not** produce `upper`/`lower` as per-bar series values, making `ta.crossover()` / `ta.crossunder()` unusable with it.

Implementation approach:
- **Non-repainting mode** (when `nwe_repaint = false`): Use the endpoint method from the source directly — computes `out`, `mae`, `upper = out + mae`, `lower = out - mae` as series on every bar. `ta.crossover()` works normally here.
- **Repainting mode** (when `nwe_repaint = true`): Adapt the Gaussian kernel loop to run on every bar (not just `barstate.islast`) using a fixed lookback. This is a reformulation: for the current bar, compute the weighted sum over the last `n` bars each tick. This will be slower but produces a per-bar series. Use `n = 499` as in the original (capped by `max_bars_back=500`).
- Signal assignments use **breakout interpretation** (opposite of LuxAlgo default — add a comment explaining this):
  ```pine
  nwe_bull = ta.crossover(close, nwe_upper)
  nwe_bear = ta.crossunder(close, nwe_lower)
  ```

### Task 1.3 — EMA calculation (Section 3)

Straightforward. Extract from `ema-cross-strategy.pine`:
- `fast_ema = ta.ema(close, ema_fast)`
- `slow_ema = ta.ema(close, ema_slow)`
- `ema_bull = fast_ema > slow_ema`
- `ema_bear = fast_ema < slow_ema`

Use position check (not crossover) per PRD §B1b.

### Task 1.4 — ATF calculation (Section 4)

**Important constraint identified from source file:** The `calcDev()` function in `adaptive-trend-finder.pine` is gated on `barstate.islast` — it returns `[na, na, na, na]` on every other bar. This means `detectedSlope` is only non-`na` on the most recent bar.

Consequence:
- Historical ATF signal markers will not appear on past bars (slope is `na` there).
- For the primary use case — live bar-close alerts — this is acceptable and correct.
- Document this limitation in a code comment.

Implementation:
- Port the `calcDev()` function verbatim from `adaptive-trend-finder.pine`, converting v6 syntax to v5 where needed (e.g., `for int i` → `for i`, `float x = na` type annotations may need adjustment).
- Use short-term periods array: `[20, 30, 40, 50, 60, 70, 80, 90, 100, 110, 120, 130, 140, 150, 160, 170, 180, 190, 200]` (19 entries).
- Call all 19 `calcDev()` invocations. Select `detectedSlope` from the entry with the highest Pearson's R.
- Signal:
  ```pine
  atf_bull = detectedSlope > 0
  atf_bear = detectedSlope < 0
  ```
- If TradingView throws an execution time limit error, expose a `atf_fixed_period` input (default `na`) as a fallback that bypasses period selection.

### Task 1.5 — RSI calculation (Section 5)

Use `ta.rsi()` built-in (do not copy from `relative-strength-index.pine`):
```pine
rsi_val         = ta.rsi(close, rsi_len)
rsi_in_zone     = rsi_val >= rsi_lo    and rsi_val <= rsi_hi
rsi_rising      = rsi_val > rsi_val[rsi_lookback]
rsi_bull        = rsi_in_zone and rsi_rising

rsi_in_bear_zone = rsi_val >= rsi_bear_lo and rsi_val <= rsi_bear_hi
rsi_falling      = rsi_val < rsi_val[rsi_lookback]
rsi_bear         = rsi_in_bear_zone and rsi_falling
```
Boundary values are inclusive (`>=`, `<=`).

### Task 1.6 — Signal logic (Section 6)

```pine
bull_signal = nwe_bull and ema_bull and atf_bull and rsi_bull
bear_signal = nwe_bear and ema_bear and atf_bear and rsi_bear
```

No additional logic — all four conditions required simultaneously.

### Task 1.7 — Primary plots (Section 7, part A)

Add always-on signal markers and background highlight:
- Green triangle below bar on `bull_signal`
- Red triangle above bar on `bear_signal`
- `bgcolor()` with high transparency (85%) for both signals

Do **not** add overlay lines or RSI table yet (those are Milestone 3).

### Task 1.8 — Milestone 1 validation

Load unified indicator alongside all four individual source indicators on the same Heikin Ashi daily chart.

Checks on RELIANCE and INFY (6-month lookback):
- [ ] Every bull marker on unified → verify NWE crossover, EMA 13>48, ATF slope>0, RSI 30–40 rising on individual indicators simultaneously
- [ ] Every bear marker on unified → verify mirror of above
- [ ] Find a bar where only 3 of 4 criteria are met → confirm no marker appears
- [ ] RSI boundary values (30, 40, 60, 70 exactly) → confirm signal fires (inclusive bounds)
- [ ] ATF slope = 0 → confirm no marker

---

## Milestone 2 — Alert Conditions

Goal: Two `alertcondition()` calls in Section 7 enabling TradingView push/email alerts.

### Task 2.1 — Add alert conditions (Section 7, part B)

```pine
alertcondition(bull_signal,
    title   = "Bull Signal — All Criteria Met",
    message = "{{ticker}} — Bullish signal on Heikin Ashi Daily: NWE breakout above upper band, EMA 13 > EMA 48, ATF uptrend, RSI 30–40 rising.")

alertcondition(bear_signal,
    title   = "Bear Signal — All Criteria Met",
    message = "{{ticker}} — Bearish signal on Heikin Ashi Daily: NWE breakdown below lower band, EMA 13 < EMA 48, ATF downtrend, RSI 60–70 falling.")
```

`{{ticker}}` is a TradingView dynamic placeholder — do not replace it with a hardcoded value.

### Task 2.2 — Milestone 2 validation

- [ ] Create a test alert on one Nifty 50 stock using "Bull Signal — All Criteria Met" condition
- [ ] Confirm notification fires on next qualifying bar close
- [ ] Confirm alert message contains the correct ticker name (not a literal `{{ticker}}`)

---

## Milestone 3 — Polish

Goal: Final production-ready file with overlays, RSI table, and clean comments.

### Task 3.1 — `show_overlays` toggle (Section 7, part C)

Wire `show_overlays` input to control visibility of:
- NWE upper band (teal line)
- NWE lower band (red line)
- Fast EMA / 13 (orange line)
- Slow EMA / 48 (blue line)
- ATF trend line / midline (purple line) — note: ATF line is only drawn on `barstate.islast` via `line.new()`; replicate that draw call conditionally under `show_overlays`

Pattern:
```pine
plot(show_overlays ? nwe_upper : na, "NWE Upper", color.teal)
```

For ATF, use conditional `line.new()` / `line.set_*` calls inside the `barstate.islast` block.

### Task 3.2 — RSI corner table (Section 7, part D)

Display a small `table` in the chart corner (avoids pane-separation conflict with `overlay=true`):
- Current RSI value (2 decimal places)
- Zone label: "Bullish zone (30–40)" | "Bearish zone (60–70)" | "Neutral"
- Rising/falling indicator

Use `table.new()` with `var` for persistence, update cells each bar.

### Task 3.3 — Inline comments pass

Add a comment at the top of each section explaining its role. Add inline comments for:
- NWE signal direction inversion (breakout vs. mean-reversion — why it differs from LuxAlgo original)
- NWE repainting mode reformulation (why the per-bar loop differs from the source)
- ATF `barstate.islast` constraint (what it means for historical markers vs. live alerts)
- RSI boundary inclusivity (`>=`, `<=`)

### Task 3.4 — Milestone 3 validation

- [ ] `show_overlays = true` → all five overlay lines appear; `show_overlays = false` → none appear
- [ ] RSI table updates correctly across stocks in all three zones (bullish, bearish, neutral)
- [ ] Full validation pass on TCS and 2 additional Nifty 50 stocks
- [ ] All edge cases from PRD §D confirmed:
  - [ ] NWE repainting behaviour noted (historical bars may shift)
  - [ ] Gap open past NWE band fires correctly
  - [ ] ATF slope = 0 → no signal
  - [ ] ATF low Pearson's R → slope sign still used (no minimum R threshold — future enhancement)
