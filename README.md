# Diagnosing Customer Satisfaction in E-Commerce: A SQL Analysis

## Key Finding

**Delivery performance is the strongest factor associated with customer satisfaction in this marketplace.** Orders delivered more than a week after their estimated delivery date receive a 1–2 star review **78.3% of the time**, versus 9.0% for orders that arrive a week or more early (about 9 times as likely). Average review score falls in a near-monotonic pattern across delay buckets: 4.32 (early) → 4.20 (on time) → 3.18 (late) → **1.73 (very late)**. A second angle agrees: orders that were never delivered (canceled, unavailable, still processing) score even worse, as low as 1.27 on average. This is a strong association, not a proven causal claim.

Two other candidate factors weaken under closer inspection:

* **Seller size.** Large sellers (100+ orders) score slightly lower than small ones (4.05 vs. 4.14 stars when each order counts once, a gap of about 0.1 stars). Ten underperforming large sellers account for roughly 40% of that gap; the remaining 60% is unexplained.
* **Payment installment count.** Scores appear to fall as installments rise, but within fixed order-value bands the pattern is not visible, which suggests order value is the stronger factor. This used summed installments; a check using the maximum per order is still pending (see Limitations).

**Customer geography follows the delay pattern.** Raw state-level differences (e.g., Rio de Janeiro at 3.97 vs. São Paulo at 4.25) track differing late-delivery *rates* between states (13.3% vs. 5.9%) far more than regional sentiment. Once split by delivery status, every state’s on-time average clusters tightly between 4.15 and 4.40.

---

## Problem Statement

Olist’s marketplace depends on review scores, which influence seller reputation, marketplace trust, and (weakly) repeat business. This project uses SQL, not just visualization, to identify which operational factors (delivery performance, product category, seller behavior, payment friction, geography) are associated with customer satisfaction, and which only look important without evidence.

**Note on scope:** this project originally set out to diagnose customer *retention*. An early base-rate check found only about 3% of customers make a repeat purchase (2,801 of 93,358 customers with a delivered order), too rare and too thin a sample to support retention as the project’s spine. The project was reframed around customer *satisfaction* (review score), with retention kept as a small, clearly labeled secondary finding.

---

## Dataset Overview

* **Source:** Kaggle — [Brazilian E-Commerce Public Dataset by Olist](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce)
* **Scope:** 99,441 orders, 9 relational tables (orders, order_items, customers, products, sellers, payments, reviews, geolocation, category translation), 2016–2018
* **Currency:** all monetary values are in Brazilian reais (R$)
* **Tools:** SQLite3 (via Python’s `sqlite3` module, VSCode) · Pandas · Matplotlib/Seaborn for EDA visuals

**Order status:** 97.02% of orders are `delivered`. This is the population used for all delivery-delay and most satisfaction queries. The remaining 2.98% (shipped, canceled, unavailable, invoiced, processing, created, approved) have no delivery-delay signal and are analyzed separately (see Finding 1b).

**Delay definition:** delay is measured against the order’s *estimated* delivery date (`order_estimated_delivery_date`), so “a week late” means a week past the date promised to the customer.

---

## Data Cleaning & Preprocessing

Three data-quality issues were found, investigated, and handled; each is documented rather than silently patched:

**1. Customer identity key.** `customer_id` in the `customers` table is unique per *order*, not per person, so using it to detect repeat customers would make every customer look like a one-time buyer by construction. `customer_unique_id` is the correct person-level key and is used throughout.

**2. `reviews` table grain — duplicate review rows.** 547 orders (0.55%) had more than one review row (a few with 3). The initial assumption was that these were cosmetic resubmissions (safe to dedupe by keeping the most recent). Testing that assumption found **36.9% of these duplicate pairs had conflicting scores**, too high to treat as cosmetic. Rather than arbitrarily picking “most recent” or averaging, all 547 orders (1,098 review rows) were **excluded** from a `reviews_clean` view used in every later query. This trades 0.55% of orders for an unambiguous signal. See Limitations for a bias this exclusion could introduce.

**3. Missing product category.** 610 products (1.85%) had a null `product_category_name`. They were labeled `'unknown'` via a `products_clean` view (not a destructive update to the raw table). The `'unknown'` group showed unremarkable satisfaction (3.94 average, mid-pack), so the fill does not appear to distort category-level findings.

**Also checked, all clean:** zero full-row duplicates across all 9 tables; zero impossible date sequences (delivered before purchased); no unparseable dates in the date columns used; consistent, expected data types.

---

## Method

* **Delivery-delay buckets** derived from the actual delay distribution (median −11.95 days; most orders arrive early) rather than guessed round numbers. The resulting “Very late (7+ days)” threshold is consistent with the IQR-based outlier boundary (8.4 days).
* **Small-sample floors** for every grouped ranking (categories, sellers), set from each group’s real size distribution. For example, the category floor was raised from 30 to 100 after checking that a quarter of all categories had fewer than 75 orders.
* **Confound checks**, not first-glance correlation: the seller-size and installment effects were each checked against a competing explanation (a few underperforming sellers; order value).
* **Payment friction measured as `SUM(payment_installments)`** across all of an order’s payment rows, to handle the 0.9% of orders paid with several payment methods. Summing is a modeling choice, not the only reasonable one (see Limitations).
* Every satisfaction query reports **both mean review score and % low (1–2 star) reviews**, because review scores are heavily skewed toward 5 stars (mean 4.09) and a small shift in the mean can mask a larger shift in the share of unhappy customers.

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

Two angles (delay within delivered orders, and status across all orders) point the same direction: getting the order to the customer on time, or at all, is the factor most strongly associated with satisfaction. Early delivery scores slightly higher than on-time delivery (4.32 vs. 4.20), which may mean customers value arriving early, though this analysis cannot say why. This is an association, not proof of cause.

### 2. Category and Seller-Level Satisfaction

**Worst-performing categories** (n ≥ 100, floor set from the real category-size distribution):

| Category | Order Count | Avg. Score | % Low Score |
|---|---|---|---|
| **moveis_escritorio** (office furniture) | 1,644 | **3.51** | 25.61% |
| fashion_roupa_masculina | 124 | 3.76 | 25.00% |
| telefonia_fixa | 251 | 3.76 | 23.11% |
| audio | 357 | 3.84 | 21.29% |
| casa_conforto | 424 | 3.86 | 19.58% |

`moveis_escritorio` stayed the worst category after the size floor was raised from 30 to 100, so it is not a small-sample artifact. Even the worst category (3.51) is far above the 1.73 seen for very-late deliveries: category effects are real but moderate.

**Seller size — a small gap, partly explained by a few sellers:**

| Seller Tier | Seller Count | Total Orders | Avg. Score |
|---|---|---|---|
| 1-5 orders | 1,246 | 2,915 | 4.17 |
| 6-25 orders | 977 | 12,065 | 4.15 |
| 26-100 orders | 510 | 26,365 | 4.14 |
| 100+ orders | 230 | 67,388 | 4.06 |

*Table values are averages of seller-level averages (each seller counts once). With each order counting once, the averages are 4.05 for sellers with 100+ orders and 4.14 for sellers with 25 or fewer.*

Sellers with 100+ orders average 4.05 stars versus 4.14 for sellers with 25 or fewer orders, a gap of about 0.1 stars. Excluding the ten 100+ sellers whose average score is below 3.5 (worst: 2.27 average, 63.5% low-score, n = 126) raises the large-seller average to about 4.09 and narrows the gap to about 0.06. Those ten sellers therefore explain roughly 40% of the gap, and most of it remains.

Two cautions. First, this exclusion selects sellers by the outcome itself, so some of the narrowing is mechanical. Second, sellers below 3.5 also exist in smaller tiers (about 5% of 26–100-order sellers and 11% of 6–25-order sellers, versus about 4% of 100+ sellers), so large sellers do not have more bad actors. These ten matter because they carry about 2,900 orders (one alone, n = 1,336, sustains a 3.34 average).

**Business takeaway:** with limited ops resources, start with sellers rather than categories. Part of the seller-level gap sits with a short, named list (ten large sellers) that can be audited directly, whereas category dissatisfaction is diffuse and hard to fix.

### 3. Geography — mostly a delay-rate proxy

| | On-time avg score | Late avg score | Late-order rate |
|---|---|---|---|
| São Paulo (SP) | 4.33 | 2.91 | 5.9% |
| Rio de Janeiro (RJ) | 4.25 | 2.13 | 13.3% |
| Bahia (BA) | 4.15 | 2.56 | 13.7% |

When delivery is on time, satisfaction clusters tightly across all 27 states (4.15–4.40), so location itself matters little. States that looked worse in a raw ranking (RJ: 3.97 overall, BA: 3.93) have more than double São Paulo’s late-delivery rate, which accounts for most of the difference. One regional signal remains: how badly a late delivery is received varies by state (late-order average ranges 1.94–3.20). This is a more tentative finding given thinner late-order samples in some states.

### 4. Payment Installments — not visible within order-value bands

Raw data shows a mild decline in satisfaction as installment count rises (4.19 → 4.05 across 1–10 installments), but installment count and order value rise together (average order value climbs from R$120 to R$410 across that range). Holding order value roughly fixed:

| Value Band (R$) | 1 (no split) | 2-4 | 5-8 | 9+ |
|---|---|---|---|---|
| Under 100 | 4.23 | 4.19 | 4.18 | 4.17 |
| 100-300 | 4.15 | 4.11 | 4.12 | 4.08 |
| 300-700 | 4.05 | 4.02 | 4.03 | 4.03 |
| 700+ | 3.96 | 3.98 | 4.09 | 3.95 |

Within any fixed value band, satisfaction moves little with installment count (0.02–0.11 point differences, no consistent direction), far less than the roughly 0.2-point drop *between* value bands. In this data, order value looks like the stronger factor and installment count adds little once value is accounted for. The 700+ band is noisy. The bands are wide, and installments are summed across payment rows (see Limitations), so this is suggestive, not definitive.

### 5. Repeat Purchase Behavior (secondary finding, small sample)

Repeat customers are rare: **3.00% of customers with a delivered order** (2,801 of 93,358), contributing 5,434 of about 95,000 delivered orders.

| Customer Type | Order Count | Avg. Score | % Low Score |
|---|---|---|---|
| One-time customer | 89,873 | 4.15 | 12.86% |
| Repeat customer | 5,434 | 4.23 | 11.67% |

Repeat customers show modestly higher satisfaction. Since repeat purchasing was not randomly assigned, the more defensible reading is that satisfied customers are more likely to return, not that returning raises satisfaction. This query cannot establish direction. It is a minor, correctly scoped observation, not a basis for a retention strategy.

---

## Business Recommendations

*(Scoped to this dataset — see Limitations.)*

* **Prioritize delivery reliability.** It is the strongest factor associated with dissatisfaction in this analysis: a “very late” order is about 9 times as likely to receive a low score as an early one (78% vs. 9%).
* **Audit the specific underperforming sellers identified in Finding 2**, rather than treating high seller volume as a risk in itself. Ten large sellers account for roughly 40% of the large-seller gap.
* **Don’t over-invest in category-specific or region-specific satisfaction initiatives.** Category effects are real but moderate, and regional differences largely follow regional delivery performance.
* **Payment plan options do not look like a satisfaction lever** in this data, since installment count adds little once order value is accounted for.

---

## Limitations

* The data covers one marketplace (Olist, Brazil) over a fixed historical window (2016–2018). Findings describe this dataset, not e-commerce in general.
* All relationships are associations from grouped SQL aggregation, not controlled experiments or regression-adjusted estimates.
* **Reviews exclusion bias (not tested).** The 547 excluded orders had multiple reviews. Customers who submit a second review often had a problem, so excluding them may make satisfaction look slightly better than it is.
* **Delay buckets and missing dates.** 8 orders marked `delivered` have no delivery date. As the delay query is written, they fall into the “Very late” bucket, so the 3,240 figure may include up to 8 such orders. Excluding them is listed under Future Scope.
* **Installments are summed** across an order’s payment rows. For the 0.9% of orders with several payments this adds installments from different payments, which is not the same as “number of installments.” A check using the maximum per order is pending.
* **Seller exclusion rule.** The seller-size analysis excludes sellers by their average score, the outcome itself, so the narrowing it shows is partly mechanical.
* Seller and category rankings use volume floors chosen from each variable’s own size distribution. Findings below the floors are not reported (e.g., installment counts above about 13 are excluded from interpretation due to thin samples).
* The repeat-purchase finding is directional only. This analysis cannot say whether satisfaction causes retention or retention reflects prior satisfaction.

---

## Future Scope

* Rerun the delay buckets with `order_delivered_customer_date IS NOT NULL`, and update the counts if they change.
* Repeat the installment analysis using `MAX(payment_installments)` per order, and check whether the conclusion holds.
* Apply the same low-score-seller rule to every seller tier, or use a method that does not select on the outcome (for example, an empirical-Bayes estimate of seller averages).
* Test whether the excluded duplicate-review orders skew toward late deliveries.
* A logistic regression or similar model to test delivery delay, category, seller, and order value jointly, with coefficients and confidence intervals.
* Time-based analysis: did delivery performance or seller quality change over 2016–2018?
* Extend the seller analysis into a seller-scorecard query refreshed periodically.
* Geolocation-level (not just state-level) analysis using the `geolocation` table, which is not yet used.

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
