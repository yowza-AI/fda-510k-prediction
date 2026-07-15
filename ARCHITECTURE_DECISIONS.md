# FDA_510k Prediction Engine: Architectural Decision Records

Predicts FDA device submissions 3-5 years forward by analyzing patent filing trends as leading indicators.

---

## Core Insight

**Patent filings precede FDA submissions by 2-4 years.** Companies file patents to protect IP, then commercialize via FDA 510k submissions. By analyzing *where patents are clustering*, you can forecast *where device approvals will follow*.

```
2023-2024: Heavy patents in "cardiac monitoring + AI"
   ↓
2025-2026: Patent clusters stabilize, intensity plateaus
   ↓
2026-2027: Predict surge in cardiac monitoring 510k submissions
   ↓
2027+: Monitor actual FDA approvals, score prediction accuracy
```

---

## ADR-1: Leverage Patent Landscape Clustering (Upstream Signal)

**Decision**: Use HDBSCAN clusters from patent_review as the foundation for FDA predictions.

**Why It Matters**: Patents are *early indicators* of R&D investment. FDA 510k submissions are the *outcome*.

**Architecture**:

```
Patent Data (USPTO HUPD)
    ↓
[Patent Embedding + Clustering] (from patent_review)
    ↓
Cluster Features:
├─ Domain (CPC category)
├─ Cluster centroid (semantic position)
├─ Cluster size (filing volume)
├─ Growth rate (year-over-year delta)
├─ Player concentration (top filers)
└─ Filing velocity (accelerating or plateauing?)
    ↓
[Time-Series Forecasting] ← NEW LAYER
    ↓
Predict 510k submissions (3-5 year horizon)
```

**Why This Choice**:
- Patent_review already ingests + clusters USPTO data
- Reusing embeddings + clusters avoids duplicate work
- Timestamps are clean (filing date is deterministic)
- CPC taxonomy maps naturally to FDA device categories

**Trade-off**: Tight coupling to patent_review version. If patent_review's embedding model changes, forecasts recalibrate.

---

## ADR-2: Time-Series Forecasting (Not Static Classification)

**Decision**: Predict *volume + timing* of 510k submissions, not just "yes/no, will we see devices here?"

**Model Pipeline**:

```
For each patent cluster:
  1. Extract features: size, growth rate, top filer diversity, intensity
  2. Align cluster to FDA device category (CPC → device class mapping)
  3. Query historical 510k submissions for that category (2015-2025)
  4. Fit ARIMA/Prophet/LSTM to: "cluster size → 510k volume, 3yr lag"
  5. Forecast 2026-2029 submissions
  6. Confidence intervals (what if growth slows? accelerates?)
```

**Why Time-Series (not classification)**:
- Classification (yes/no) loses nuance. You want *how many* + *when*.
- 510k volume trends follow patent clusters with consistent lag
- Forecasting uncertainty is valuable (investors: range, not point estimate)

**Rejected Alternatives**:

| Approach | Pros | Cons |
|----------|------|------|
| **Time-series forecast** | Predicts volume + timing | Requires historical alignment (messy) |
| **Binary classifier** | Simple, fast | Loses quantity/timing information |
| **LLM analyst** | Flexible reasoning | Non-deterministic, hard to backtest |

---

## ADR-3: Historical Alignment via "Patent Lag" Window

**Decision**: Correlate historical patent clusters (2015-2020) with actual 510k submissions (2017-2025), extract lag distribution.

**Why It Matters**: You don't know *a priori* that a 2-year lag holds. You discover it from data.

**Process**:

```
For each 510k submission (2017-2025):
  1. Extract device category + filing date
  2. Query patent clusters for that category, 2-4 years prior
  3. Find the closest cluster by semantics + time
  4. Record lag: (510k_date - cluster_date)
  5. Build lag distribution by category
  
Result: Lag distribution per device category
  ├─ Cardiac monitoring: 2.1 ± 0.8 years (median lag, std dev)
  ├─ Orthopedic implants: 3.2 ± 1.1 years
  └─ Diagnostics: 1.5 ± 0.9 years
```

**Why This Choice**:
- Unsupervised learning from historical data
- Different device categories have different lag profiles
- Accounts for regulatory velocity (fast-track vs. standard)
- Creates uncertainty estimates (±std dev = confidence interval)

**Trade-off**: Requires matching 510k submissions to patent clusters. Some submissions may have *no* prior patent (bought from supplier, minimal R&D). Those are exceptions, flagged in output.

---

## ADR-4: Backtesting: Blind Hindcasting

**Decision**: Validate forecast accuracy by "predicting" 2025 submissions using only 2015-2021 data.

**Process**:

```
Split at 2021-12-31:
├─ Train data: Patents 2015-2021 + 510ks 2017-2021 (calibrate lag)
├─ Test data: 510ks 2022-2025 (actual submissions)
└─ Holdout: Patents 2022-2025 (not used in training)

Forecast 2022-2025 510k volume per category using trained lag
Compare forecast → actual
Score: MAE, MAPE, AUC (for binary: will we see ≥N submissions?)
```

**Why Backtesting**:
- Validates lag assumptions (does 2.1yr lag actually hold?)
- Catches overfitting (rich historical data, could fit noise)
- Surfaces category-specific issues (cardiology forecasts well, diagnostics don't)
- Quantifies uncertainty (real forecast should have same error bars)

**Honest Reporting**:
- "Cardiac monitoring forecasts: MAE=15 submissions/year (good)"
- "Diagnostics forecasts: MAE=45 submissions/year (poor, likely noise)"
- "Segments by FDA track status (expedited, 510k standard) show different lag"

---

## ADR-5: Reuse patent_review's Clustering (Not Retrain)

**Decision**: Import precomputed HDBSCAN clusters from patent_review's artifact (DuckDB).

**Why It Matters**: Avoids computational duplication; clusters are versioned + deterministic.

**Data Flow**:

```
patent_review outputs: artifact_v1.duckdb
├─ Cluster assignments (patent_id → cluster_id)
├─ Cluster centroids (embeddings)
├─ Metadata (size, top filers, year span)
└─ Versions tracked (if embedding model changes, new version)

FDA_510k reads this artifact:
├─ Loads cluster features
├─ Aligns clusters to FDA categories
├─ Trains time-series forecaster
└─ Produces 510k forecast report
```

**Why Reuse**:
- Single source of truth for patent clustering
- If patent_review releases v2 (better embeddings), FDA forecasts automatically benefit
- Simpler architecture (no duplicate clustering logic)

**Coordination**: FDA_510k CI must depend on patent_review artifact release.

---

## ADR-6: Multi-Layer Uncertainty (Not Point Predictions)

**Decision**: Forecasts include three uncertainty bands:
1. **Model uncertainty** (ARIMA 95% CI): What if trend continues normally?
2. **Lag uncertainty** (from backtest std dev): What if lag distribution shifts?
3. **External uncertainty** (user-defined): Regulatory changes, market disruption

**Output Format**:

```json
{
  "category": "Cardiac Monitoring",
  "forecast_2027": {
    "point_estimate": 340,
    "model_ci": [320, 360],
    "lag_ci": [310, 370],
    "external_scenario_aggressive": 400,
    "external_scenario_conservative": 280
  }
}
```

**Why Multiple Bands**:
- Shows *where uncertainty comes from* (model, lag variance, unknowns)
- Investors/manufacturers can stress-test scenarios
- Honest about limits (we don't know external shocks)

---

## ADR-7: Investor Thesis vs. Regulatory Foresight (Use-Case Aware)

**Decision**: Same underlying model, different output formats per audience.

**Investor Briefing**:
```
"Cardiac monitoring 510ks will surge 35% in 2027-2028 (confidence: high)
 → Patent cluster grew 3.2x 2021-2024
 → Lag profile suggests peak submissions in Q2-Q4 2027
 → Top filers (Abbott, GE, Medtronic) dominate; fragmentation risk low"
```

**Regulatory Foresight** (FDA/notified bodies):
```
"Expected cardiac monitoring submission volume: 340 ± 30 in 2027
 Breakdown by predicate device class (510k tracking):
  - Holter monitors: 45 (mature, stable)
  - Wearables: 120 (growing, AI-heavy)
  - Remote monitoring platforms: 175 (fast growth, likely expedited)
 Risk: diagnostics category forecasts poorly (high variance, may need retuning)"
```

**Why Two Formats**:
- Investors want narrative, thesis, risk/opportunity
- Regulators want precise numbers, category breakdown, confidence intervals
- Same data, different stories

---

## ADR-8: Continuous Recalibration (Streaming 510k Approvals)

**Decision**: As 2025-2026 510k submissions arrive, retrain forecasts (quarterly updates).

**Process**:

```
Every quarter:
  1. Ingest new 510k submissions from FDA database (automated)
  2. Update historical alignment (patent ↔ 510k matches)
  3. Recalibrate lag distribution
  4. Retrain time-series models
  5. Update forecasts for 2027-2029
  6. Compare new forecast to prior quarter's 2027 prediction (tracking error)
```

**Why Recalibration**:
- Real data updates beliefs (2025 submissions reveal lag was 2.3yr, not 2.1yr)
- Models drift (external shocks: new regulations, pandemic supply chains)
- Transparency (publish backtest-vs-actual scorecard quarterly)

**Tracking Error Published**: "Our 2024-forecasted 2027 volume: 340. Actual Q1 2027: 85. Annualized track: ~340 (on target)."

---

## Summary: Patent-to-FDA Signal Chain

| Stage | Data | Method | Output |
|-------|------|--------|--------|
| **Patent Clustering** | USPTO HUPD (2015-2025) | HDBSCAN (from patent_review) | Cluster features, CPC taxonomy |
| **Lag Discovery** | Historical 510ks (2017-2025) | Correlation with clusters | Lag distribution per category |
| **Forecasting** | Cluster features 2022-2025 | ARIMA/Prophet per category | 510k volume forecast ±CI |
| **Backtesting** | Held-out 2025 510ks | Blind hindcast (2015-2021 train) | MAE, MAPE, category accuracy |
| **Delivery** | Forecast + uncertainty | Investor/regulator format | Actionable predictions |

---

**Document Owner**: Architecture team  
**Last Updated**: July 2026
