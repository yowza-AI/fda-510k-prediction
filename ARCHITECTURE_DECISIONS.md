# DeviceGap: Architectural Decision Records

Medical device gap analysis: joining predicate citations, embeddings, adverse events, patents, and reimbursement data to identify whitespace opportunities.

---

## ADR-1: Honest About Data Limitations (Not Hidden)

**Decision**: Every report, viz, and score states its data caveats up-front. MAUDE denominator problem, predicate extraction confidence, survivorship bias—all in the product, not footnotes.

**Why It Matters**: This is a *credibility tool*, not a dashboarding tool. Wrong predictions in public are better than hidden limitations. The prediction ledger is only valuable if people trust it.

**Example**:
```
This report identifies ≥1 persistent failure mode using MAUDE adverse events.
Caveat: MAUDE counts have no exposure denominator (units sold unknown).
Interpretation: Named failure clusters are signals for investigation, not statistical validation.
```

**Trade-off Accepted**: Honesty costs adoptability short-term. Builds moat long-term (competitors won't state limits; you will, and you'll be proven right).

---

## ADR-2: Predicate Extraction via Eval Set, Not Heuristics Alone

**Decision**: Build a hand-checked eval set (~100 PDFs) to validate predicate extraction accuracy. Score every edge with a confidence metric.

**Why It Matters**: Predicate citations are *the* linchpin. Wrong edges = wrong graph = wrong gap analysis.

**The Risk**:
- 510(k) summary PDFs have no standardized format
- Predicate citations are parsed as K-numbers (`K\d{6}`) in context
- Some summaries are "statements" with no technical content (no predicates)
- Applicant names are unnormalized ("Foo Inc" vs "Foo Incorporated")

**Mitigation**:
- Hand-check ~100 PDFs, record extraction accuracy per type (e.g., "K-number in title: 98% accuracy", "K-number in body: 87%")
- Every extracted edge carries a `confidence_score` (0–1)
- Human review required before edges feed published claims
- Report coverage % per product code (e.g., "83% of K-numbers have summaries with predicates")

**Why This Matters for Architecture**: The system is only as good as its graph. Data quality is non-negotiable.

---

## ADR-3: Empirical Lag Distribution (Not Assumed Window)

**Decision**: Instead of claiming "products take 18–36 months from patent to 510(k)," measure it from historical data.

**Why It Matters**: "18–36 months" is widely quoted but unvalidated. The actual lag varies by product code.

**Method**:
```
For each product code:
  1. Extract all 510(k) clearances (applicant, decision date)
  2. Match applicant to patent portfolio (USPTO, same applicant)
  3. For clearances by new entrants, find prior patents by same assignee
  4. Compute lag distribution: clearance_date - max(patent_filing_dates)
  5. Report: median ± IQR per product code

Example result:
  Cardiac monitoring: median 24 months, IQR 12–36 months
  Orthopedic implants: median 36 months, IQR 24–48 months
```

**Caveat (required)**:
- Survivorship-conditioned: lags observed only for entrants who *actually arrived*
- Doesn't capture abandoned filings
- Doesn't account for defensive/licensing patents

**Why This Choice**: Measured data beats priors. The IQR is the real deliverable (shows variation by code).

---

## ADR-4: Four Gap Types + Reimbursement Filter

**Decision**: Not a flexible scoring matrix. Four discrete gap types + one severity modifier (reimbursement).

**Why It Matters**: Clarity. "Here are the 3–5 gaps that matter, ranked by type" beats "here are 47 gaps with scores 0.73–0.91."

**Gap Types**:

| Type | Predicate-Graph Signal | Patent Signal | Interpretation |
|------|------------------------|---------------|-----------------|
| **Contested Whitespace** | Sparse devices | Accelerating filings | Entrants coming; window closing; urgency HIGH |
| **Ignored Whitespace** | Sparse devices | Flat/zero patents | Possible demand desert; risk HIGH unless demand signal exists |
| **Broken Incumbency** | Dense devices + persistent failures | Non-incumbent acceleration | Disruption opportunity; tech is stale; market ready for new entrant |

**Reimbursement Filter** (applied to all):
- No HCPCS code → "gap you can't bill for" → demote severity + explain

**Why This Structure**: Three types map to distinct business strategies. Clear enough for investor conversation; precise enough for engineering.

---

## ADR-5: Persistent Failure Mode (Not Raw Event Counts)

**Decision**: Rank failures by narrative persistence across predicate generations, not by raw MAUDE event count.

**Why It Matters**: MAUDE has a huge denominator problem (no units sold). Ranking by count is misleading.

**Method**:
```
1. Cluster MAUDE narratives (device family + semantic embeddings + HDBSCAN)
2. For each cluster, track across predicate generations:
   - Does cluster recur in next-generation devices?
   - Do clusters share similar keywords (persistent theme)?
3. If cluster persists + co-occurs with recalls, name it as a failure mode
4. Report: name + narrative examples (paraphrased, anonymized) + generation span
```

**Example (Cardiac Devices)**:
- Failure Mode: "Battery depletion before expected service life"
- Observed: All predicate generations 2010–2024
- MAUDE clusters: 4 distinct narrative clusters, consistent terminology
- Associated recalls: 3 Class II recalls, 2016–2022
- Interpretation: Persistent, known issue in lineage

**Why This Choice**: Persistence is a signal. Raw counts are noise.

---

## ADR-6: Separate Semantic Indices for Device vs. MAUDE Narratives

**Decision**: Embed device descriptions + MAUDE narratives in the *same* model space, but maintain separate indices.

**Why It Matters**: Device descriptions optimize for class prediction; MAUDE narratives optimize for failure clustering. Same embedding space lets you measure proximity; separate indices keep queries fast.

**Architecture**:
```
sentence-transformers model (BGE-large or GTE-large)
    ↓
[Device text index]  ← intended use + description
    ↓ (same embedding space)
    ← proximity = semantic similarity across classes
    ↓
[MAUDE narrative index]  ← failure descriptions
    ↓
Query: kNN search in device space finds nearby device regions
       kNN search in MAUDE space finds similar failure narratives
       Cross-index similarity reveals failure concentration near device clusters
```

**Why Separate**:
- MAUDE narratives are rare per device (most devices have 0 events)
- Mixing with device text dilutes device-class signal
- Independent indices mean independent scaling (device index may grow 100×; MAUDE stays bounded)

---

## ADR-7: DuckDB (Dev) + Postgres (Prod) Migration Path

**Decision**: Start with DuckDB for speed. Schema designed so migration to Postgres is straightforward (Phase 4+).

**Why It Matters**: Phase 1–3 are solo/research. Phase 4+ might need multi-tenant SaaS or custom reports at scale.

**Trade-off**:
- DuckDB: fast, local, single-process, perfect for Phase 1–4
- Schema written to be Postgres-compatible (typed columns, FK constraints, no DuckDB-specific features)
- When Postgres needed: schema mostly just works; indexes and conn pooling added

**Why This Approach**: Don't over-engineer early. DuckDB is faster for Phase 1–4 work. Path to scaling is clear if/when revenue justifies it.

---

## ADR-8: Public Prediction Ledger (Append-Only)

**Decision**: Every prediction published with date, claim, resolution criteria, and eventual outcome. Ledger is append-only; wrong predictions aren't deleted.

**Why It Matters**: This is the moat. Public track record compounds credibility.

**Format**:
```jsonl
{"date": "2026-07-15", "product_code": "ABC", "claim": "≥1 new competitor 510(k)", "window_months": 24, "resolution_date": "2028-07-15", "outcome": null}
{"date": "2026-06-01", "product_code": "XYZ", "claim": "persistent failure mode in cardiac leads", "evidence": "narrative cluster across 3 generations", "outcome": "confirmed_by_recall_2026-09"}
```

**Why Public**:
- Competitors hide track records
- You don't; you're building credibility
- Wrong predictions → methodology improvement → next prediction better

---

## Summary: Honesty as Architecture

Every decision above is driven by one principle:

**Trust through Transparency, Not Glossy Presentation**

| Decision | Transparency Principle |
|----------|------------------------|
| Caveats in product | MAUDE denominator stated everywhere |
| Eval set for extraction | Confidence scores on edges, coverage % reported |
| Measured lag, not assumed | IQR with survivorship caveat |
| Persistence over counts | Explicit: "not ranking by raw events" |
| Separate indices | Tradeoff explained in reports |
| Append-only predictions | Wrong predictions public, not hidden |

This is unusual. Most analysis tools emphasize confidence. DeviceGap emphasizes honesty. Over years, honesty wins credibility.
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
