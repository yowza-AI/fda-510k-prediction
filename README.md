# DeviceGap: Medical Device Whitespace Analysis Engine

> **Problem**: Medtech investors and strategy teams are blind to FDA device gaps. Where is innovation actually happening? Where is the technology stale? Where is the next opportunity?

**DeviceGap solves this** by joining the FDA's predicate-device citation graph, semantic embeddings of device descriptions and adverse events, patent filing activity, and reimbursement data to identify **device classes that are structurally broken or commercially open**.

Not a submission tool. Not a landscape browser. This answers: **"Where is the next device opportunity, what's the evidence, and how long is the window?"**

---

## The Problem DeviceGap Solves

**Current tools fail:**
- Predicate finders (Cruxi) help you clear *your* device, but don't show the landscape
- Patent databases show R&D activity but miss FDA decisions
- MAUDE adverse events have no exposure denominator (can't rank by raw event counts)
- Reimbursement data is scattered across CMS databases
- Nobody connects all four signals together

**Result**: Investors miss gaps. Startups overestimate whitespace. Market strategy is based on outdated taxonomy, not on what's actually happening.

---

## How It Works

```
FDA Data (openFDA APIs)
├─ 510(k) clearances (K-numbers, product codes, applicants, dates)
├─ 510(k) summary PDFs (device descriptions, intended use, predicate citations)
├─ Device classifications (product code taxonomy)
├─ MAUDE adverse events (failure mode narratives)
└─ Recalls + enforcement actions
    ↓
[Parse & Graph]
├─ Extract predicate citations from PDF text (hand-labeled eval set for accuracy)
├─ Build citation graph: K-numbers → predicates → K-numbers (lineage chains)
├─ Attach product codes, applicant names, decision dates
    ↓
[Semantic Layer]
├─ Embed device descriptions + intended-use text
├─ Embed MAUDE narrative text (separate index)
├─ Cluster failure narratives within device families (HDBSCAN)
├─ Measure narrative persistence across predicate generations
    ↓
[Patent Signal]
├─ Import patent abstracts (USPTO via existing patent project)
├─ Embed in same space as device descriptions
├─ Measure filing velocity near device regions
├─ Compute historical lag: patent filing → 510(k) clearance (per product code)
    ↓
[Reimbursement Filter]
├─ Map product codes to HCPCS billing codes
├─ Identify coverage gaps
├─ Demote whitespace that can't be reimbursed
    ↓
[Gap Classification]
├─ Contested Whitespace: sparse devices + accelerating patent filings (urgency: HIGH)
├─ Ignored Whitespace: sparse devices + flat patents + no demand signal (risk: demand desert)
├─ Broken Incumbency: dense devices + persistent failures + stale tech + entrant patents (opportunity: disruption)
    ↓
[Report Generation]
├─ Per-product-code markdown report with:
│  ├─ Predicate genealogy visualization
│  ├─ Staleness metrics (years since last substantive entrant)
│  ├─ Named failure modes (with paraphrased narratives)
│  ├─ Patent acceleration chart
│  ├─ Reimbursement pathway assessment
│  └─ ≥1 dated, falsifiable prediction (e.g., "expect ≥1 new entrant in XYZ within 24 months")
├─ Interactive landscape visualization (UMAP embed space, colored by epoch/failure/applicant)
└─ Prediction ledger (append-only: every prediction with outcome tracking)
```

---

## Why This Architecture Matters

### 1. Honest About Data Limitations

**MAUDE caveat (not hidden, stated in every report):**
- Adverse event counts have *no exposure denominator* (units sold unknown)
- Never rank device families by raw event counts
- Use persistence (same failure cluster across predicate generations) and co-occurrence instead
- Report what's observable; state what's not

**Predicate extraction risk (mitigated by design):**
- 510(k) summary PDFs have no standardized format
- Predicate citations are parsed via regex + context heuristics on K-number patterns
- Built an eval set of ~100 hand-checked PDFs to validate extraction accuracy
- Every predicate edge carries a confidence score
- Human review required before any claim gets published

### 2. Measured, Not Assumed

**Historical lag analysis (Phase 3):**
- Survives the classic "18–36 month window" prior by measuring it
- Extract patent filing dates and clearance dates for all historical entrants
- Compute median + IQR lag per product code (e.g., "cardiac monitors: 24mo median, 12–36mo IQR")
- State the caveat: *survivorship-conditioned* (lags observed only for entrants who actually arrived)

**Patent signal hygiene:**
- Count patent *families*, not raw filings (continuations/divisionals fake acceleration)
- Weight by maintenance-fee behavior (renewals = committed intent; lapsed = abandoned)
- Non-incumbent patent acceleration is signal; incumbents patenting near their own products is noise
- Frame as "capital committed to this space," not "products in development"

### 3. The Moat: Public Prediction Ledger

Every report includes ≥1 dated, falsifiable prediction. All predictions published append-only in a public ledger with:
- Claim (e.g., "≥1 new competitor 510(k) in product code XYZ")
- Date (prediction timestamp)
- Window (e.g., "within 24 months")
- Resolution criteria
- Eventual outcome

This is unusual. Most analysis firms hide track records. DeviceGap *accumulates* credibility in public. Wrong predictions aren't deleted—they're marked as missed, and the methodology is refined in Phase N+1.

### 4. Four Gap Types + Reimbursement Filter

```
Gap Type                  Signal                              Interpretation
────────────────────────────────────────────────────────────────────────────
Contested Whitespace      Sparse devices + patent↗            Entrants coming; window closing
Ignored Whitespace        Sparse devices + flat patents       Possible demand desert
Broken Incumbency         Dense + failures + stale + entrant↗ Disruption opportunity

                          + Reimbursement filter (applied to all)
No HCPCS/coverage path → Demoted + explanation: "gap you can't bill for is a trap"
```

Not "here are 47 gaps." Rather: "here are the 3–5 gaps that matter, ranked by type."

---

## Data Sources (All Public, All Free)

| Source | What | Role |
|--------|------|------|
| openFDA `/device/510k` API | 510(k) clearances | Core nodes (K-numbers, product codes, applicant, decision dates) |
| 510(k) summary PDFs | Device description, intended use, predicate citations | Text for embeddings + predicate graph edges |
| openFDA `/device/classification` | Product codes, device class, regulatory number | Taxonomy backbone |
| openFDA `/device/event` (MAUDE) | Adverse event narratives | Failure-mode semantic clusters |
| openFDA `/device/recall` | Recall root causes, severity | Outcome labels + failure persistence signal |
| USPTO PatentsView API | Patent abstracts, CPC, assignees, filing dates | Leading-indicator layer (ported from existing patent project) |
| CMS HCPCS + coverage data | Billing codes, coverage determinations | Reimbursement filter |

**Cost**: $0 for Phase 1–4. All APIs are free or bulk-downloadable. No cloud until a paying engagement.

---

## Tech Stack

| Layer | Technology | Why |
|-------|-----------|-----|
| **Ingestion** | Python fetchers per source | Bulk downloads preferred (respect rate limits) |
| **Storage** | DuckDB (dev) / Postgres (prod) | Fast local queries, clean migration path to SaaS |
| **Graph** | NetworkX | Manageable at ~200K nodes (predicate citations) |
| **Embeddings** | sentence-transformers (BGE/GTE models) | fp16-compatible (GPU constraint: Turing cards, no FlashAttention2) |
| **Vector Index** | FAISS on disk (dev) / pgvector (prod) | Fast kNN for density metrics |
| **Clustering** | HDBSCAN on MAUDE narratives | Density-based, finds natural clusters |
| **Visualization** | Plotly + Deck.gl for interactive viz | Single-file HTML reports |
| **Workflow** | Python 3.11+, uv/poetry, pytest | Typed (mypy-clean), config via .env |

---

## Build Phases

### Phase 1: Backbone (1–2 weeks)
- Bulk load 510(k), classification, recall data → DuckDB
- PDF summary fetcher + predicate extractor (with eval set)
- Graph construction + staleness metrics
- **Output**: 3 candidate product codes ranked by staleness + concentration

### Phase 2: Semantic Layer (1–2 weeks)
- Embed device descriptions + intended-use text
- Embed MAUDE narratives for candidate codes only
- HDBSCAN failure clusters
- **Output**: One product code with named, defensible persistent failure mode

### Phase 3: Joins (2 weeks)
- Patent ingestion (port from existing project) → same embedding space
- Manual HCPCS crosswalk for chosen code
- Gap taxonomy scoring
- **Empirical lag study**: historical patent-filing → 510(k)-clearance lag distribution
- **Output**: Full gap classification with all evidence layers

### Phase 4: Ship (1 week)
- Report template + interactive viz
- Prediction ledger (append-only JSON)
- Public editing pass
- **Output**: First published report with dated prediction

---

## For Technical Interviews

**What to look for in code review:**

1. **Predicate extraction is careful** (eval set, confidence scores)
   - This is the hardest data-quality problem
   - Look for: test coverage on PDF parsing, human review workflow

2. **Gap taxonomy is disciplined** (three types + filter, not ad-hoc scoring)
   - Not "here's our formula"; rather "here's why each type matters"
   - Look for: explicit decision criteria, no hidden weights

3. **Prediction ledger is honest** (wrong predictions not hidden)
   - Public track record is the moat
   - Look for: append-only structure, versioning of methodology

4. **Limitations are stated up-front** (MAUDE denominator, survivorship bias, etc.)
   - Not buried in a footnote
   - Look for: every report and viz carries its caveats

**Interview questions:**

- "Why is predicate extraction worth budgeting real time for?"
- "MAUDE has no denominator. How does the gap engine avoid ranking by raw event counts?"
- "A product code has no 510(k) summaries (only statements). How does the pipeline handle that?"
- "You measure historical lag, but survivorship-bias the results. What does that mean?"
- "Your prediction ledger shows a wrong prediction. How does that inform Phase N+1?"

---

## Full Code

**This repository contains architecture and design philosophy.** Full source code is private.

For technical interviews, I can grant read-only access:
- Complete Python ingestion pipeline (openFDA fetcher, PDF parser, predicate extractor)
- DuckDB schema and migrations
- Predicate graph construction (NetworkX)
- Embedding pipeline (sentence-transformers)
- HDBSCAN clustering + failure mode detection
- Patent layer + lag-distribution computation
- Gap scoring and report generation
- Prediction ledger + publish workflow
- Test suite (parser tests, eval set validation)

**Contact**: yow.stephend@gmail.com

---

## Development Approach

Architected and designed by me. Implementation accelerated with Claude as an AI design/coding partner.

This demonstrates: building a *defensible analysis tool* where honesty about limitations (not hiding them) is a feature. The prediction ledger is the moat—public credibility compounds.

---

**Status**: Phase 1 backbone in progress (predicate graph + staleness metrics)  
**Data**: openFDA bulk datasets + patent data via existing project  
**Deployment**: Local + eventual Tier 1 consulting ($2.5K–$15K per custom report)  
**Horizon**: Phase 4 (first public report with prediction) target: ~4 weeks
