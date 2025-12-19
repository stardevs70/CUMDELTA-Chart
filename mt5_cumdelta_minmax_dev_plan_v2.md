# MT5 Indicator Modification Plan — Add MaxDelta/MinDelta Histograms (Separate Bars)

## 1) Project summary
You have an existing **MT5 (MQL5) indicator** (`CumDelta_Chart.mq5`) that displays **Net Delta** in a separate subwindow with an `IndicatorMode` setting:
- **Mode A:** cumulative "chart" view (cumulative delta)
- **Mode B:** non-cumulative "histogram/bar" view (net delta per bar)

Requested change (confirmed by client):
- **Add MaxDelta and MinDelta values as histogram bars per candle** that NetDelta will be drawn on top of (superimposed).
- MaxDelta and MinDelta must be displayed as **separate histograms** (two different data points), not a single filled range.
- The MaxDelta/MinDelta histograms must appear in **both IndicatorMode options**.
- The MaxDelta/MinDelta histograms must display **even when NetDelta = 0** (as long as candle data exists).
- Add **one color input** that controls the histogram color (same color used for both MaxDelta and MinDelta).
- **Do not change anything else** (no changes to calculations, DLL calls, data source, other visuals).

Deliverables:
1. Modified `.mq5` source file with MaxDelta/MinDelta histograms + color input.
2. Client will do functional testing (indicator requires a subscription that won't work in developer terminal).

---

## 2) Data definitions (what to draw)
For each candle (bar) of the chart timeframe:
- **NetDelta** = final delta sum for the candle (already computed as `deltaCandle`)
- **MaxDelta** = maximum value reached by the running delta sum within that candle (already computed as `highDelta`)
- **MinDelta** = minimum value reached by the running delta sum within that candle (already computed as `lowDelta`)

These are already computed in the indicator. We only add display logic.

---

## 3) Client Clarifications (Updated)

### 3.1) Histogram display relative to zero line
- **Both modes** must have a zero line
- Histograms draw **from 0 to the per-bar highDelta/lowDelta values**
- In cumulative mode: candles show cumulative values, but histograms still show per-bar values relative to 0
- Example: If Max=32 and Min=-25, draw histogram from 0 to 32 and from 0 to -25

### 3.2) 1M Timeframe
- 1M chart **definitely has different max and min values** for delta
- All 3 figures (MaxDelta, MinDelta, NetDelta) must display separately on 1M

### 3.3) Histogram Width and Visual Style
- Histogram width: **Same width as candle body**
- Histogram should **cover the wicks** on all timeframes
- Candles drawn on top of histograms

---

## 4) Visual design (implementation approach)

### Challenge: Scale mismatch in cumulative mode
In cumulative mode, candles are at values like -19500, but histograms would be at values like 32, -25 (around 0). They're on completely different scales.

### Solution: Use DRAW_HISTOGRAM2 to cover candle wicks
Instead of drawing from absolute 0, draw histograms that **cover the candle wicks**:
- Plot 0: **MaxDelta histogram** using `DRAW_HISTOGRAM2` - draws from candle OPEN to candle HIGH
- Plot 1: **MinDelta histogram** using `DRAW_HISTOGRAM2` - draws from candle OPEN to candle LOW
- Plot 2: existing **NetDelta candles** as `DRAW_COLOR_CANDLES` (drawn last so body stays on top)

This way:
- Histograms are on the **same coordinate system** as candles
- Histograms visually **cover/replace the wicks**
- Works in **both modes** (cumulative and non-cumulative)

---

## 5) Step-by-step implementation plan

### Step 1 — Add new input for histogram color
Add one input parameter:
```mql5
input color MinMaxHistogramColor = clrSilver; // applies to both MaxDelta and MinDelta histograms
```

---

### Step 2 — Add four indicator buffers (Max, MaxBase, Min, MinBase)
DRAW_HISTOGRAM2 requires 2 buffers per plot (value and base):
```mql5
double MaxDeltaBuffer[];    // High value
double MaxDeltaBase[];      // Base (candle open)
double MinDeltaBuffer[];    // Low value
double MinDeltaBase[];      // Base (candle open)
```

Update indicator properties:
```mql5
#property indicator_plots   3
#property indicator_buffers 9
```

Buffer index mapping:

| Purpose | Buffer index |
|---|---:|
| MaxDeltaBuffer (high) | 0 |
| MaxDeltaBase (open) | 1 |
| MinDeltaBuffer (low) | 2 |
| MinDeltaBase (open) | 3 |
| Candle Open | 4 |
| Candle High | 5 |
| Candle Low | 6 |
| Candle Close | 7 |
| Candle Color Index | 8 |

---

### Step 3 — Plot properties
```mql5
#property indicator_label1  "MaxDelta"
#property indicator_type1   DRAW_HISTOGRAM2
#property indicator_color1  clrSilver
#property indicator_width1  2

#property indicator_label2  "MinDelta"
#property indicator_type2   DRAW_HISTOGRAM2
#property indicator_color2  clrSilver
#property indicator_width2  2

#property indicator_label3  "CumDelta Chart"
#property indicator_type3   DRAW_COLOR_CANDLES
#property indicator_color3  clrRed,clrGreen,clrDarkGray
#property indicator_width3  2
```

In `OnInit()`, force both histogram plot colors from input:
```mql5
PlotIndexSetInteger(0, PLOT_LINE_COLOR, 0, MinMaxHistogramColor);
PlotIndexSetInteger(1, PLOT_LINE_COLOR, 0, MinMaxHistogramColor);
```

---

### Step 4 — Buffer binding in `OnInit()`
```mql5
// Plot 0: MaxDelta histogram (DRAW_HISTOGRAM2)
SetIndexBuffer(0, MaxDeltaBuffer, INDICATOR_DATA);
SetIndexBuffer(1, MaxDeltaBase, INDICATOR_DATA);

// Plot 1: MinDelta histogram (DRAW_HISTOGRAM2)
SetIndexBuffer(2, MinDeltaBuffer, INDICATOR_DATA);
SetIndexBuffer(3, MinDeltaBase, INDICATOR_DATA);

// Plot 2: Color candles (OHLC + color)
SetIndexBuffer(4, ColorCandlesBuffer1, INDICATOR_DATA);
SetIndexBuffer(5, ColorCandlesBuffer2, INDICATOR_DATA);
SetIndexBuffer(6, ColorCandlesBuffer3, INDICATOR_DATA);
SetIndexBuffer(7, ColorCandlesBuffer4, INDICATOR_DATA);
SetIndexBuffer(8, ColorCandlesColors, INDICATOR_COLOR_INDEX);
```

---

### Step 5 — Populate Max/Min buffers in the existing calculation loop
Where the indicator already computes highDelta, lowDelta, and candle values:

```mql5
// Both modes: histogram covers the wick area (from open to high/low)
if(IndicatorMode == 0) {
    // Cumulative mode
    MaxDeltaBuffer[ix] = highDelta + LastCloseCandle;  // Cumulative high
    MaxDeltaBase[ix] = openvalue;                       // Candle open
    MinDeltaBuffer[ix] = lowDelta + LastCloseCandle;   // Cumulative low
    MinDeltaBase[ix] = openvalue;                       // Candle open
} else {
    // Non-cumulative mode
    MaxDeltaBuffer[ix] = highDelta;   // Per-bar high
    MaxDeltaBase[ix] = 0;             // Zero line
    MinDeltaBuffer[ix] = lowDelta;    // Per-bar low
    MinDeltaBase[ix] = 0;             // Zero line
}
```

Rules:
- Populate these in **both** IndicatorMode values
- Populate them even if `deltaCandle == 0` (client requirement)
- If the candle has **no data** (e.g., `iBase < 0`), set all to `EMPTY_VALUE`

---

### Step 6 — Compile and smoke-check
- Compile in MetaEditor: **0 errors**
- Attach indicator to chart:
  - Input `MinMaxHistogramColor` appears
  - No runtime errors / array bounds errors in logs

---

## 6) Testing plan

### Developer-side (no subscription)
- Compile + attach to chart without crashing
- Verify plots and buffer counts are correct

### Client-side acceptance (with subscription)
1. In **both modes**, MaxDelta and MinDelta histograms appear covering the wicks
2. NetDelta candle body remains visible on top
3. Change `MinMaxHistogramColor` → both histograms update color
4. Bars with **NetDelta = 0** still show Max/Min histograms
5. **1M timeframe** shows all 3 values separately
6. **All timeframes** show histograms covering the wicks
7. No other behavior changes

---

## 7) Acceptance criteria
- MaxDelta and MinDelta histograms visible in both modes
- Histograms cover candle wicks on all timeframes including 1M
- Histogram width matches candle body width
- Candle body (Open/Close) visible on top of histograms
- One color input controls both histogram colors
- Existing logic/data source unchanged

---

## 8) Deliverables
- Updated `CumDelta_Chart.mq5` source file implementing:
  - Plot 0: MaxDelta histogram (DRAW_HISTOGRAM2, from open to high)
  - Plot 1: MinDelta histogram (DRAW_HISTOGRAM2, from open to low)
  - Plot 2: existing NetDelta color-candles (unchanged behavior)
  - Input: `MinMaxHistogramColor`
