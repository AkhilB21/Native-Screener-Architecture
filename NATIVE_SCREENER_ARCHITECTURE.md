# Native Screener — Architecture Notes for Implementation

**Status: Future Implementation (post v4.8 stabilization)**
**Date: 2026-08-04**

---

## 1. Decision: Replace Chartink Pipeline with Native Parquet Screener

**Rationale:** Apollo now has 2400+ stocks with D/4H/W OHLCV data in `apollo_data/*.parquet`. The Chartink scraper (Playwright-based) has recurring auth failures (419), TOS risk, and only covers NIFTY 500. A native screener eliminates external dependencies, runs in seconds, covers the full universe, and enables analytics Chartink never provided (volume surge, price-volume divergence, etc.).

**What gets removed:** Entire `chartink_pipeline/` module (fetcher.py, auth.py, watchlist_generator.py, run_pipeline.py) and Playwright dependency.

**What gets created:** `nse_engine/native_screener.py` — reads Parquet data, computes Recovery + Momentum tables, outputs same CSV format the dashboard already consumes.

**Dashboard impact:** None. `_render_scan_intelligence()` in `dashboard.py` reads CSV files — it doesn't care where they come from. Output format stays identical.

---

## 2. Stock Phase Classification (Core Architecture)

Instead of bucketing stocks into fixed categories and having different scoring criteria per bucket, the screener will use **stock-agnostic normalized features** that automatically adapt to each stock's own behavior.

### The Four Phases

Every stock in the universe is classified into one of four phases based on its current state. The phase is **not a bucket with different scoring rules** — it's an emergent property of the normalized features.

| Phase | Description | Screener Action |
|-------|-------------|-----------------|
| **Momentum** | Making new highs, accelerating price movement, multi-TF trend alignment | Appears in Momentum Candidates table |
| **Recovery** | Pulling back in an uptrend, RSI bouncing from oversold zones, structural trend intact | Appears in Recovery Candidates table |
| **Breaking Down** | Falling below key support, MA structure deteriorating, distribution patterns | Excluded from both tables (Avoid) |
| **Bottoming Out** | Too early to act — selling may be exhausting but no confirmation of reversal yet | Excluded from both tables (Watch / Too Early) |

### How a Stock Enters a Phase

The phase is NOT determined by a separate classification step with hardcoded thresholds. Instead, it **emerges from the normalized features** themselves. A stock naturally lands in the right phase based on its feature values:

- High RSI percentile + high volume z-score + price near range top + MA alignment → **Momentum**
- Low RSI percentile + rising volume + price near range bottom + MA structure intact → **Recovery**
- Deteriorating MA structure + declining volume on up-days + price breaking supports → **Breaking Down**
- Very low RSI but no volume confirmation + no MA support + continued new lows → **Bottoming Out**

**Key principle:** The same scoring formula runs for every stock. No per-bucket parameters, no per-bucket weights. The features encode the phase information; the scorer doesn't need to know which phase a stock is in.

### Why Not Per-Bucket Scoring?

| Problem | Explanation |
|---------|-------------|
| **Discontinuity** | A stock at the boundary between two buckets gets a different score on the same price action depending on which side it falls. |
| **Parameter explosion** | 3 buckets x 5 scoring pillars = 15 weights to tune vs 5. More parameters = higher overfitting risk. |
| **Bucket migration** | A stock can shift buckets as volatility changes, causing inconsistent scoring without any real change in the thesis. |
| **Curve-fitting risk** | Tailoring criteria to limited stock subsets is still curve-fitting, just at a smaller scale. |

---

## 3. Feature Normalization (Handles Stock Diversity)

All features are expressed as **relative metrics** — where the current value sits within the stock's own recent distribution. This means the same scoring formula works across large-caps, mid-caps, and small-caps without any explicit bucketing.

| Raw Metric (Fragile) | Normalized Metric (Robust) |
|---------------------|--------------------------|
| RSI(21) > 60 | RSI(21) percentile rank vs its own 6-month rolling history |
| Volume > 20-day average | Volume z-score vs its own 60-day rolling distribution |
| 5-day gain > 5% | 5-day gain percentile vs its own 90-day rolling distribution |
| Price above 50-DMA | Price percentile vs its own 200-day range |
| ATR = 15 | ATR as % of stock's median ATR over 120 days |
| RSI(21) on weekly | Weekly RSI percentile vs its own 26-week rolling history |

**Why this is not curve-fitting:**
1. The normalization window is rolling (always recent, not full backtest period)
2. The scoring formula is identical for every stock — no per-stock parameter optimization
3. We're changing the input representation, not adding parameters

---

## 4. Recovery Scoring (Port Existing System)

Recovery scoring already exists in `chartink_pipeline/watchlist_generator.py` (`_compute_recovery_score`). The native screener will port and refine this logic, fed from Parquet data instead of Chartink.

### Current Recovery Score Components (0-100)

| Component | Points | Source |
|-----------|--------|--------|
| Daily RSI 21 in 35-50 zone (recovery zone) | 20 | Apollo's primary entry zone |
| Weekly RSI confirmation | 15 | Weekly trend not overbought |
| 4H RSI momentum confirm (RSI21_4H > 50) | 20 | Higher-TF doesn't contradict |
| % fall from high (more fall = more recovery room) | 20 | Opportunity sizing |
| % gain from low (some bounce already started) | 15 | Early confirmation |
| Other RSI cross-checks | 10 | RSI 36, RSI 56 alignment |

**Plan:** Port as-is initially, then refine with empirical data from the historical momentum analysis (Section 7 below).

---

## 5. Momentum Scoring (NEW — To Be Built)

This is the new system that doesn't exist yet. Two-phase approach:

### Phase A: Empirical Discovery (Build First)

Before writing any scoring criteria, run a historical analysis script against Parquet data:

1. **Define "momentum success"** — e.g., stock gains >= 15% within 20 trading days with max drawdown < 8%
2. **Generate candidate features** at every historical bar for every stock (20-30 features)
3. **Identify winners** — stocks that met the success definition
4. **Profile winners vs non-winners** — which features had the highest separation?
5. **Derive evidence-based weights** from the effect sizes

### Phase B: Build the Scorer (From Discovery Results)

Starting framework (to be refined by Phase A results):

| Pillar | Max Points | Description |
|--------|-----------|-------------|
| Price Momentum Strength | 25 | Rate of change / acceleration, not just level |
| Volume Confirmation | 20 | Rising volume on up-days, surge on breakout |
| Trend Structure / MA Alignment | 20 | Price above key MAs in right order, MA slopes |
| Higher-TF Confirmation | 20 | Weekly + 4H trend aligned with daily momentum |
| Risk-Reward Profile | 15 | Distance from support, not too far from range low |

**Key design rule:** Best momentum entries are NOT at the top — they're when momentum is **accelerating from a moderate base**.

---

## 6. Two-Layer Architecture

### Layer 1: Normalized Features (Handles 80% of stock diversity)

Every input to the scorer is a relative metric (percentile rank, z-score, or normalized value). The scorer never sees raw RSI, raw volume, or raw % gain. One scorer works across the entire 2400-stock universe.

### Layer 2: Broad Regime Overlays (Not per-stock buckets)

A small number of market-level filters applied uniformly:

| Overlay | Purpose | Example |
|---------|---------|---------|
| Market-cap gate | Exclude illiquid stocks where slippage kills you | Min avg daily volume >= Rs 5L or min market cap >= Rs 500Cr |
| Trend regime | Momentum criteria work differently in trending vs ranging markets | NIFTY 50 above/below its 50-DMA |
| Volatility regime | Widen or tighten thresholds based on market volatility | Use India VIX or cross-sectional median ATR |

These are on/off overlays or single-threshold adjustments — not per-stock scoring modifications.

---

## 7. Implementation Roadmap

| Step | Task | Effort | Depends On |
|------|------|--------|------------|
| 1 | Build historical momentum analysis script | 1 session | Parquet data for 2400 stocks |
| 2 | Run analysis, identify winning feature profiles | Manual review | Step 1 |
| 3 | Derive evidence-based momentum scoring weights | Manual review | Step 2 |
| 4 | Build `native_screener.py` — Recovery table from Parquet | Low | Existing recovery score logic |
| 5 | Build `native_screener.py` — Momentum table from Parquet | Medium | Step 3 (weights) |
| 6 | Add volume analytics (surge, up/down ratio, PVD) | Medium | Step 4 |
| 7 | Output CSVs in existing dashboard format | Trivial | Step 4 + 5 |
| 8 | Dashboard path fix (nse_output vs chartink_output) | Trivial | Step 7 |
| 9 | Remove `chartink_pipeline/` and Playwright dep | Trivial | Step 8 verified working |

---

## 8. Relationship to Existing Bucket Classifier

The existing `bucket_classifier.py` uses 3 structural metrics (200-DMA slope, 50-200 DMA gap, 6M return) to classify stocks into 7 buckets. This is a **separate system** from the native screener's phase classification.

| Aspect | Bucket Classifier (existing) | Native Screener Phases (new) |
|--------|---------------------------|---------------------------|
| Purpose | Label for backtest UI display / colour-coding | Determine which table a stock appears in (Recovery vs Momentum) |
| Input | 3 structural metrics | Normalized multi-feature profile (RSI, volume, MA, price structure, higher-TF) |
| Granularity | 7 buckets | 4 phases (2 actionable, 2 excluded) |
| Used as gate? | No (reference-only since v3.4.1) | No — phases emerge from features, not from separate classification |

The bucket classifier continues to serve its purpose in the backtest engine. The native screener's phase classification is independent and serves the live screening use case.

---

## 9. Known Path Issue

`dashboard.py` line 1179 looks for CSVs in `nse_output/`:
```python
output_dir = Path(_PROJECT_ROOT) / "nse_output"
recovery_csvs = sorted(output_dir.glob("chartink_apollo_ranked_*.csv"), reverse=True)
```

But `watchlist_generator.py` line 35 writes to `chartink_output/`:
```python
DEFAULT_OUTPUT_DIR = _PROJECT_ROOT / "chartink_output"
```

This path mismatch must be resolved — either the dashboard reads from `chartink_output/` or the native screener writes to `nse_output/`. Verify on VPS which directory the dashboard actually picks up.
