# Diagnosing Customer Satisfaction in E-Commerce: A SQL Analysis

## Key Finding

**Delivery performance is the dominant driver of customer satisfaction in this marketplace** — orders delivered more than a week late receive a 1-2 star review **78.3% of the time**, versus 9.0% for orders that arrive a week or more early. Average review score falls in a near-perfectly monotonic pattern across delay buckets: 4.32 (early) → 4.20 (on time) → 3.18 (late) → **1.73 (very late)**. This is independently confirmed from a second angle: orders that were never delivered at all (canceled, unavailable, still processing) score even worse — as low as 1.27 average.

Two other apparent drivers — **seller size** and **payment installment count** — initially looked like real effects, but both **dissolved once properly controlled for**. Large-volume sellers appeared to score worse on average (4.06 vs. 4.17 for small sellers) until a small number of specifically underperforming sellers were excluded, at which point the gap nearly closed (4.11). Installment count appeared to correlate with lower scores until order value was held constant, at which point the effect all but disappeared — expensive orders simply get split into more installments *and* score slightly lower, independently of each other.

**Customer geography shows a similar pattern to seller size**: raw state-level differences (e.g., Rio de Janeiro at 3.97 vs. São Paulo at 4.25) are explained almost entirely by differing late-delivery *rates* between states (13.3% vs. 5.9%), not by regional customer sentiment. Once split by delivery status, every state's on-time average clusters tightly between 4.15 and 4.40.

---

## Problem Statement

Olist's marketplace lives and dies on review scores — they drive seller reputation, marketplace trust, and (weakly) repeat business. This project uses SQL — not just visualization — to identify which operational factors (delivery performance, product category, seller behavior, payment friction, geography) are actually associated with customer satisfaction, versus factors that only look important without evidence.

**Note on scope:** this project originally set out to diagnose customer *retention*. An early base-rate check found only 3.12% of customers make a repeat purchase — too rare and too thin a sample to support retention as the project's spine. Rather than dataset-shop for a different business to force the original framing, the project was honestly reframed around customer *satisfaction* (review score), with the retention question kept as a small, clearly-labeled secondary finding rather than the headline.

---

## Dataset Overview

* **Source:** Kaggle — [Brazilian E-Commerce Public Dataset by Olist](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce)
* **Scope:** ~99,441 orders, 9 relational tables (orders, order_items, customers, products, sellers, payments, reviews, geolocation, category translation), 2016–2018
* **Tools:** SQLite3 (via Python's `sqlite3` module, VSCode) · Pandas · Matplotlib/Seaborn for EDA visuals

**Order status:** 97.02% of orders are `delivered`. This is the population used for all delivery-delay and most satisfaction queries — the remaining 2.98% (shipped, canceled, unavailable, invoiced, processing, created, approved) have no delivery-delay signal and are analyzed separately (see Finding 1b).

---

## Data Cleaning & Preprocessing

Three real data-quality issues were found, investigated, and fixed — each is documented rather than silently patched:

**1. Customer identity key.** `customer_id` in the `customers` table is unique per *order*, not per person — using it to detect repeat customers would make every customer look like a one-time buyer by construction. `customer_unique_id` is the correct person-level key and is used throughout.

**2. `reviews` table grain — duplicate review rows.** 547 orders (0.55%) had more than one review row (a few with 3). Initial assumption was that these were cosmetic resubmissions (safe to dedupe by keeping the most recent). Testing this assumption instead of trusting it found **36.9% of these duplicate pairs had genuinely conflicting scores** — too high to treat as cosmetic. Rather than arbitrarily picking "most recent" or fabricating an average, all 547 orders (1,098 review rows) were **excluded** from a `reviews_clean` view used in every subsequent query. This trades 0.55% of orders for an unambiguous, defensible signal in the rest.

**3. Missing product category.** 610 products (1.85%) had a null `product_category_name`. Filled as `'unknown'` via a `products_clean` view (not a destructive update to the raw table). The `'unknown'` group later showed unremarkable satisfaction (3.94 avg, mid-pack) — confirming the fill didn't distort category-level findings.

**Also verified, all clean:** zero full-row duplicates across all 9 tables; zero impossible date sequences (delivered-before-purchased); zero unparseable dates across every date column used in the analysis; consistent, expected dtypes throughout.

---

## Method

* **Delivery-delay buckets** derived from the actual delay distribution (median −11.95 days, most orders arrive early), not guessed round numbers — the resulting "Very late (7+ days)" threshold independently matches the IQR-based statistical outlier boundary (8.4 days).
* **Small-sample floors** for every grouped ranking (categories, sellers) set from the real size distribution of each group, not a fixed guess — e.g., the category floor was raised from an initial 30 to 100 after checking that a quarter of all categories had fewer than 75 orders.
* **Confound testing**, not first-glance correlation: both the seller-size effect and the payment-installment effect were explicitly tested against a competing explanation (bad-actor sellers; order value) rather than reported as-is.
* **Payment friction measured as `SUM(payment_installments)`** across all of an order's payment rows (not `MAX`), correctly handling the 0.9% of orders paid via multiple distinct payment methods (e.g., two credit cards plus a voucher).
* Every satisfaction query reports **both mean review score and % of low (1-2 star) reviews** — because review scores are heavily skewed toward 5 stars (mean 4.09), a small shift in the mean can mask a larger shift in the share of genuinely unhappy customers.

---

## Findings

### 1. Delivery Delay → Satisfaction (headline finding)

| Delivery Status | Order Count | Avg. Score | % Low Score (1-2★) |
|---|---|---|---|
| Early (7+ days) | 70,528 | 4.32 | 8.96% |
| On time (0-7 days early) | 17,149 | 4.20 | 10.20% |
| Late (1-7 days) | 4,390 | 3.18 | 36.08% |
| **Very late (7+ days)** | **3,240** | **1.73** | **78.27%** |

**1b. Order status alone (independent of delay), for comparison:**

| Order Status | Order Count | Avg. Score | % Low Score |
|---|---|---|---|
| processing | 294 | 1.27 | 92.86% |
| unavailable | 593 | 1.53 | 84.82% |
| invoiced | 305 | 1.62 | 82.95% |
| canceled | 601 | 1.79 | 77.04% |
| shipped | 1,021 | 2.00 | 69.64% |
| **delivered** | **95,307** | **4.16** | **12.79%** |

Two independent angles (delay-within-delivered, and status-across-all-orders) point the same direction: getting the order to the customer on time — or at all — is the dominant driver of satisfaction in this marketplace. Early delivery slightly outperforms on-time delivery (4.32 vs. 4.20), suggesting customers are actively delighted by arriving ahead of schedule, not merely neutral about it. This is a strong association, not a proven causal claim.

### 2. Category and Seller-Level Satisfaction

**Worst-performing categories** (n ≥ 100, floor set from the real category-size distribution):

| Category | Order Count | Avg. Score | % Low Score |
|---|---|---|---|
| **moveis_escritorio** (office furniture) | 1,644 | **3.51** | 25.61% |
| fashion_roupa_masculina | 124 | 3.76 | 25.00% |
| telefonia_fixa | 251 | 3.76 | 23.11% |
| audio | 357 | 3.84 | 21.29% |
| casa_conforto | 424 | 3.86 | 19.58% |

`moveis_escritorio` remained the worst category even after the size floor was raised from 30 to 100 — a real, stable pattern, not small-sample noise. Even the worst category (3.51) is far above the 1.73 seen for very-late deliveries: category effects are real but moderate.

**Seller size — a correlation that dissolved under testing:**

| Seller Tier | Seller Count | Total Orders | Avg. Score |
|---|---|---|---|
| 1-5 orders | 1,246 | 2,915 | 4.17 |
| 6-25 orders | 977 | 12,065 | 4.15 |
| 26-100 orders | 510 | 26,365 | 4.14 |
| 100+ orders | 230 | 67,388 | 4.06 |

Large sellers initially appeared to underperform. Isolating and excluding just the sellers with avg score < 3.5 among the 100+ group (worst: 2.27 avg, 63.5% low-score, n=126) raised the remaining large-seller average to **4.11** — nearly matching small sellers. **The apparent "scale hurts satisfaction" effect is not structural; it's driven by a small, identifiable number of specific underperforming sellers** (one alone, n=1,336, sustains a 3.34 average). This reframes a vague trend into an actionable one.

**Business takeaway:** with limited ops resources, focus on **sellers, not categories** — category dissatisfaction is diffuse and hard to "fix," while seller dissatisfaction is concentrated in a short, named list that can be audited directly.

### 3. Geography — mostly a delay-rate proxy

| | On-time avg score | Late avg score | Late-order rate |
|---|---|---|---|
| São Paulo (SP) | 4.33 | 2.91 | 5.9% |
| Rio de Janeiro (RJ) | 4.25 | 2.13 | 13.3% |
| Bahia (BA) | 4.15 | 2.56 | 13.7% |

When delivery is on time, satisfaction clusters tightly across all 27 states (4.15–4.40) — location itself barely matters. States that looked worse in a raw ranking (RJ: 3.97 overall, BA: 3.93) are explained almost entirely by having more than double São Paulo's late-delivery rate, not by regional sentiment. The one independent regional signal: how badly a late delivery is received varies by state (late-order avg ranges 1.94–3.20) — a secondary, more tentative finding given thinner sample sizes in some states' late buckets. This reinforces Finding 1 from a different angle rather than adding a new driver.

### 4. Payment Installments — a second dissolved correlation

Raw data shows a mild decline in satisfaction as installment count rises (4.19 → 4.05 across 1–10 installments) — but installment count and order value move together (avg order value climbs from ₹120 to ₹410 across that same range). Controlling for order value directly:

| Value Band | 1 (no split) | 2-4 | 5-8 | 9+ |
|---|---|---|---|---|
| Under 100 | 4.23 | 4.19 | 4.18 | 4.17 |
| 100-300 | 4.15 | 4.11 | 4.12 | 4.08 |
| 300-700 | 4.05 | 4.02 | 4.03 | 4.03 |
| 700+ | 3.96 | 3.98 | 4.09 | 3.95 |

Within any fixed value band, satisfaction barely moves with installment count (0.02–0.11 point differences, no consistent direction) — far smaller than the ~0.2-point drop seen *between* value bands regardless of installments. **Payment friction, as measured here, is not a meaningful independent driver of dissatisfaction — order value is.** This is the second instance in this project of an apparent driver dissolving into a more fundamental variable once tested (the first: seller size).

### 5. Repeat Purchase Behavior (secondary finding, small sample)

Repeat customers are rare in this marketplace — **3.00% of customers** (2,801 of 93,358), contributing 5,434 of ~95,000 delivered orders. This rarity is why retention was demoted from the project's main focus (see Problem Statement).

| Customer Type | Order Count | Avg. Score | % Low Score |
|---|---|---|---|
| One-time customer | 89,873 | 4.15 | 12.86% |
| Repeat customer | 5,434 | 4.23 | 11.67% |

Repeat customers show modestly higher satisfaction. Given repeat purchasing wasn't randomly assigned, the more defensible reading is that **satisfied customers are more likely to return**, not that returning itself improves satisfaction — this query cannot establish direction. Reported as a minor, correctly-scoped observation, not a basis for a retention strategy.

---

## Business Recommendations

*(Scoped to this dataset — see Limitations.)*

* **Prioritize delivery reliability above all else.** It is the single largest, most consistent driver of dissatisfaction found in this analysis — a "Very late" order is roughly 9x more likely to receive a low score than an early one.
* **Audit the specific underperforming sellers identified in Finding 2**, rather than treating high seller volume as a risk factor in itself — the data shows it isn't volume, it's a short list of named sellers.
* **Don't over-invest in category-specific or region-specific satisfaction initiatives.** Category effects are real but moderate; regional effects are mostly a symptom of regional delivery performance, not a distinct problem to solve directly.
* **Payment plan options are not a satisfaction lever** in this data — installment count has no meaningful independent effect once order value is accounted for.

---

## Limitations

* Dataset covers a fixed historical window (2016–2018) for one marketplace (Olist, Brazil) — findings describe this dataset's patterns, not a universal e-commerce rule.
* The reviews-deduplication decision (excluding 547 ambiguous orders) removes 0.55% of orders rather than guessing at a "true" score — a defensible but not the only possible choice.
* Seller- and category-level rankings use volume floors chosen from each variable's own size distribution; findings below those floors are not reported as conclusions (e.g., installment counts above ~13 are excluded from interpretation due to thin samples).
* The repeat-purchase finding (Finding 5) is directional only — this analysis cannot establish whether satisfaction causes retention or retention reflects prior satisfaction.
* All relationships reported are associations from grouped SQL aggregation, not controlled experiments or regression-adjusted estimates — "predicts" and "associated with" are used deliberately in place of "causes" throughout.

---

## Future Scope

* Logistic regression or a similar model to jointly test delivery delay, category, seller, and value effects with real coefficients and p-values (mirroring the joint-model validation approach used in the companion loan-approval EDA project).
* Time-based analysis: has delivery performance or seller quality improved or worsened over the 2016–2018 window?
* Extend the seller-tier "bad actor" analysis into an ongoing seller-scorecard query, refreshed periodically.
* Geolocation-level (not just state-level) analysis of delivery delay, using the `geolocation` table not yet used in this project.

---

## Tools & Technologies
Python · Pandas · Matplotlib · Seaborn · SQLite3 (window functions, CTEs, `EXPLAIN QUERY PLAN`)

---

## Project Structure
```bash
ecommerce-satisfaction-sql-analysis/
│
├── data/
│   └── (Olist CSVs)
├── notebooks/
│   └── sql_analysis.ipynb
├── olist.db
└── README.md
```

---

## Author
**Mohammad Adil**
Data Analyst | Risk & Business Analytics
