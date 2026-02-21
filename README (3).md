# Service → Install Conversion Funnel Analysis

**Kaplan-Meier survival analysis of service-to-install conversion timing across a garage door rollup**

---

## Business Question

> *For customers who first came to us for a service call, how many go on to book a Replacement or Install — and how long does it take?*

A plain conversion rate can't answer this correctly. Customers who had their first service call six months ago have had far less time to convert than one who came in three years ago. Counting them the same understates your true conversion rate and makes newer cohorts look worse than they are.

This notebook uses **Kaplan-Meier survival analysis** to produce unbiased, time-adjusted conversion probability estimates at any horizon (6, 12, 18, 24 months), broken down by brand and market type.

---

## What's in This Repo

```
├── service_to_install_funnel.ipynb   # Main analysis notebook
├── README.md                         # This file
└── outputs/
    ├── fig_overall_km.png            # Overall KM conversion curve
    ├── fig_brand_km.png              # KM curves by brand
    ├── fig_market_km.png             # KM curves by market type
    ├── fig_erpsc.png                 # Brand scorecard: conv rate & ERPSC
    ├── brand_scorecard.csv           # Summary metrics per brand
    └── service_to_install_outreach_list.csv  # Actionable outreach targets
```

---

## Methodology

### Funnel Definition

| Parameter | Choice |
|---|---|
| Funnel entry | Customer's **first-ever service invoice** at a given brand |
| Conversion event | Any subsequent `Replacement / Install` invoice at the **same brand** |
| Conversion scope | Same brand only (not cross-brand within rollup) |
| Customer identifier | `customer_id` as-is |
| Study end / censoring cutoff | `2026-01-01` |
| Exclusion criterion | Customers who had an install *before or on* their first service call |

### Why Kaplan-Meier?

KM handles two problems that break a simple conversion rate:

1. **Censoring** — Many customers haven't converted *yet* but still might. KM marks them as "still at risk" rather than counting them as failures.

2. **Time heterogeneity** — A customer from 2022 has had 3+ years to convert; one from 2025 has had months. KM separates these correctly by computing conversion probability conditional on reaching each time point.

The KM estimator at each event time $t$:

$$\hat{S}(t) = \prod_{t_i \leq t} \left(1 - \frac{d_i}{n_i}\right)$$

Where $d_i$ = conversions at $t_i$, $n_i$ = customers still at risk. Confidence intervals use Greenwood's formula with the complementary log-log transformation for stability near boundary values.

The notebook **does not use `lifelines`** — KM is implemented from scratch in pure NumPy/Pandas, so there are no additional dependencies beyond the standard data science stack.

### Expected Revenue Per Service Customer (ERPSC)

A composite metric computed per brand:

```
ERPSC = KM conversion probability @ 24 months × median install revenue
```

This is the expected revenue attributable to acquiring one new service customer, and is the primary metric for comparing brands on a risk-adjusted basis.

---

## Setup

### Requirements

```
pandas
numpy
matplotlib
```

No additional installs required. Survival analysis is implemented natively.

### Running the Notebook

1. Clone this repo
2. Place your invoice data at the path set in `DATA_PATH` (Section 1)
3. Run all cells top-to-bottom

```bash
git clone <repo-url>
cd service-to-install-funnel
jupyter notebook service_to_install_funnel.ipynb
```

### Data Schema

The notebook expects a single sheet with these columns:

| Column | Type | Description |
|---|---|---|
| `brand` | string | Brand name |
| `location` | string | Location name |
| `invoice_no` | string | Invoice identifier |
| `invoice_date` | date | Invoice date |
| `overall_market` | string | Market segment (Residential, Commercial, etc.) |
| `market_type` | string | Transaction type — must include `"Service"` and `"Replacement / Install"` |
| `customer_id` | string/int | Customer identifier |
| `order_status` | string | Order status — cancelled orders are excluded |
| `total_amount` | float | Invoice revenue |

---

## Outputs

### Figures

| Figure | Description |
|---|---|
| `fig_overall_km.png` | Cumulative conversion curve across all brands with 95% CI and milestone annotations |
| `fig_brand_km.png` | Per-brand KM curves on the same axes for direct comparison |
| `fig_market_km.png` | KM curves split by market type (Residential / Commercial / Mixed) |
| `fig_erpsc.png` | Side-by-side brand scorecard: 24-month conversion rate and ERPSC |

### CSVs

**`brand_scorecard.csv`** — One row per brand with:
- Funnel entrant count
- Converter count
- Raw conversion rate
- KM-adjusted rates at 6, 12, 24 months
- Median install revenue
- ERPSC @ 24 months

**`service_to_install_outreach_list.csv`** — Unconverted service customers currently inside the peak conversion velocity window, ranked by remaining conversion potential. Columns:

| Column | Description |
|---|---|
| Customer ID | Customer identifier |
| Brand | Brand |
| Location | Location |
| Market | Market segment at first service call |
| First Service Date | Funnel entry date |
| Days Since Service | Age in funnel as of study end |
| Remaining Conv Potential (%) | Estimated additional conversion probability available |

---

## Notebook Structure

| Section | Contents |
|---|---|
| **1 — Load & Clean** | Data ingestion, type coercion, date filtering, cancellation exclusion |
| **2 — Cohort Construction** | Funnel entry, exclusion criteria, survival column derivation, revenue enrichment |
| **3 — Kaplan-Meier** | KM implementation, overall curve, brand-level curves, market-level curves |
| **4 — Visualisations** | All four figures |
| **5 — Revenue Impact** | Install revenue distributions, ERPSC computation |
| **6 — Outreach List** | Peak velocity window detection, candidate scoring and export |
| **7 — Summary Scorecard** | Combined brand scorecard table and export |

---

## Limitations & Assumptions

- **Customer identity is taken at face value.** The `customer_id` field contains mixed formats (account numbers, phone numbers, `CASH`). No deduplication or fuzzy matching is performed. Customers with non-unique IDs may be undercounted.
- **Same-brand conversion only.** A customer who got service at Brand A and later booked an install at Brand B is not counted as a conversion. This is intentional — it attributes revenue correctly to the brand that generated the service relationship.
- **Censoring is non-informative.** KM assumes that customers who are censored (didn't convert by the study end date) have the same underlying conversion probability as those still being observed. This assumption holds here because censoring is at a fixed calendar date, not triggered by customer behavior.
- **Revenue figures use first install only.** Subsequent installs by the same converted customer are not included in ERPSC. This understates long-run customer value.

---

## Next Steps

- **Cox Proportional Hazards model** — Add covariates (brand, market type, seasonality, geography) to identify which factors statistically accelerate or delay conversion, with confidence intervals.
- **Log-rank test** — Formally test whether brand/market differences in conversion curves are statistically significant.
- **RFM segmentation** — Layer Recency/Frequency/Monetary value scoring onto the outreach list to further prioritize highest-value targets.
- **Repeat install analysis** — Extend the model to track second and third installs, building a fuller customer lifetime value picture.

---
