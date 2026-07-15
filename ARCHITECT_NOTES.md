# FDA_510k: Architect Interview Context

**Predictive signal architecture**: Using patent filing trends to forecast FDA device submissions 3-5 years ahead.

---

## The Core Insight You're Demonstrating

You've discovered and implemented a *leading indicator* for regulatory outcomes. Patent filings → FDA submissions is a causal chain with measurable lag. This shows:

1. **Data-driven hypothesis validation** (lag isn't assumed; extracted from historical data)
2. **Cross-domain modeling** (connect patent universe to regulatory universe)
3. **Uncertainty quantification** (not point predictions, but confidence intervals with known variance sources)
4. **Continuous feedback loops** (backtest validation + quarterly recalibration)

---

## What to Look For in Code Review

### 1. Lag Discovery is Data-Driven, Not Hardcoded

You extract lag from *history*, not assume it. Search for:
```python
# Good: discovers lag distribution from data
def align_patents_to_510k(patent_clusters, fda_submissions):
    lags = []
    for submission in fda_submissions:
        cluster = find_nearest_cluster(submission.category, submission.date)
        lag = submission.date - cluster.filing_date
        lags.append(lag)
    
    lag_distribution = distribution_stats(lags)  # median, std, quantiles
    return lag_distribution
```

**Why this matters**: Proves you don't hardcode assumptions. Different device categories have different lags. The code *discovers* this.

**Interview question**: "Why is the lag distribution interesting? What if it's not normally distributed?"

### 2. Backtesting is Blind and Honest

Look for:
```python
# Good: trains on 2015-2021 data, tests on 2022-2025
train_cutoff = datetime(2021, 12, 31)
train_patents = patents[patents.filing_date <= train_cutoff]
train_submissions = submissions[submissions.date <= train_cutoff]

# Fit lag model on train data
lag_model = fit_lag_distribution(train_patents, train_submissions)

# Score on held-out test data
test_submissions = submissions[submissions.date > train_cutoff]
predictions = forecast_volume(test_patents[test_patents.filing_date > train_cutoff], lag_model)
accuracy = score(predictions, test_submissions)
```

**Why this matters**: Shows you don't evaluate on data you trained on (classic overfitting). Blind hindcasting is the gold standard.

**Interview question**: "What if your backtesting accuracy is 50%? How do you debug that?"

### 3. Multiple Uncertainty Sources, Not Squashed to One Number

Look for separate treatment of:
- Model uncertainty (ARIMA CI)
- Lag variance (from historical alignment)
- External scenarios (user-defined "what-if")

**Why this matters**: Acknowledges *where* uncertainty comes from. A manager asking "will we hit 340 submissions?" gets "point: 340, but model ±20, lag ±30, external ±60—so really 260-420."

**Interview question**: "Why would you report a range instead of a single forecast?"

### 4. Time-Series Forecasting, Not Static Models

The model *changes over time*. Not "classify patent cluster A into device category X." Instead: "how many submissions will category X produce per year, 2026-2029?"

```python
# Good: models temporal dynamics
class SubmissionForecaster:
    def __init__(self, lag_distribution, historical_volume):
        self.lag = lag_distribution  # 2.1 ± 0.8 years for category X
        self.historical = historical_volume  # volume by year
        
    def forecast(self, patent_volume_2022_2025):
        # Shift patent volume by lag, then smooth with trend
        lagged_volume = shift(patent_volume_2022_2025, self.lag.median)
        trend = fit_trend(self.historical[-5:])  # last 5 years
        forecast = lagged_volume + trend_adjustment
        return forecast
```

**Interview question**: "If you only had one year of patent data, could you forecast 510k submissions?"

---

## Technical Interview Questions

### Foundational

1. **"Why is patent filing a better signal than market research surveys?"**
   - Answer shows you understand: patents are *behavioral* (companies actually invest), surveys are opinion (companies say what they think they'll do)

2. **"What would invalidate the patent-to-FDA lag assumption?"**
   - Good answers: acquired tech (buy existing patents, no internal R&D), regulatory changes (fast-track approved new categories), market shifts (pandemic supply chains)

3. **"How do you know your backtest accuracy will hold in the future?"**
   - This is the honest question. You don't, fully. Good answer: "Quarterly recalibration with tracking error published. If lag changes, we detect and adapt."

### Architecture & Modeling

4. **"You reuse clusters from patent_review. What breaks if patent_review changes its embedding model?"**
   - Tests: Do you understand coupling? Versioning? Can you articulate the dependency?

5. **"Lag is normally distributed. What if it's bimodal (two distinct lag profiles)?"**
   - Tests: Do you know your assumptions? Can you detect when they break?

6. **"Your ARIMA model assumes linear trend. What if the 2026-2027 510k surge is actually *sigmoid* (S-curve)?"**
   - Tests: Do you know when to choose different models? Can you swap out ARIMA for LSTM?

### Uncertainty & Risk

7. **"You report uncertainty bands. How do you validate they're actually calibrated?"**
   - Good answer: "Use prediction interval coverage (PI Coverage = % of actuals that fell in the 95% CI). Should be 95%. If it's 60%, my intervals are too narrow."

8. **"An investor asks: 'Will cardiac monitoring 510ks exceed 400 in 2027?' Your model says 340 ± 50. How do you answer?"**
   - Tests: Can you translate model output to business decision? (Probability ~16% it exceeds 400, given normal distribution)

### Data Engineering

9. **"You need to add a new device category mid-project. How does your system handle new categories with no historical lag data?"**
   - Good answer: "Use category of related devices as prior. Cardiac monitoring delay similar to cardiac diagnostics. Update as real data arrives."

10. **"FDA data quality: sometimes 510k submissions are missing abstracts or device categories. How does your pipeline handle this?"**
    - Tests: Do you validate? Reject bad records? Flag them? Use fallback?

---

## What This Says About You as an Architect

- **Comfortable with uncertainty**: Not all problems have crisp answers. You quantify and track uncertainty instead of pretending it doesn't exist
- **Data-driven hypothesis**: You don't assume lag—you extract it from history
- **Cross-domain integration**: Connect patent trends (supply-side signals) to FDA submissions (demand-side outcomes)
- **Feedback loops**: Backtesting → recalibration → tracking error publication. Model improves continuously
- **Aware of model risk**: Different device categories may need different forecasters. Honest about where forecasts are weak

---

## Next Conversations

If this resonates:

1. **Signal Strength**: Which other regulatory domains have good leading indicators? (CE marking delays? NMPA approvals?)
2. **Real-time Streaming**: How would this work if FDA published submissions via API (instead of quarterly dumps)?
3. **Causal Inference**: Beyond correlation (patent lag → 510k), can you prove causality? (Does accelerating patents actually *cause* more submissions, or is it spurious correlation with market demand?)
4. **Integration with patent_review**: How tightly coupled should FDA_510k be to patent_review? Could it stand alone?

---

**Status**: Designed and architected by me. Modeling implementation accelerated with Claude as design/coding partner.

**Contact for code review**: yow.stephend@gmail.com
