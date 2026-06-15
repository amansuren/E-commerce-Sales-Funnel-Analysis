# 🛒 E-commerce Sales Funnel Analysis Project

## Business Problem
An e-commerce store had no visibility into where users were dropping off across its purchase funnel — making it impossible to prioritise marketing spend or product improvements.
 
Using **30 days of event-level data** on BigQuery, I investigated one core question:
 
> **"Why are only 17% of visitors buying — and what can we do about it?"**


## 📸 Query Results - Funnel & Conversion Rates

### **Note:**
_This is a simulated dataset. Real-world e-commerce conversion rates typically fall between 1–3%. The elevated figures here are characteristic of the synthetic data and were used to focus on funnel methodology rather than benchmark comparison._

### Results at a Glance
 
| What We Measured | Result |
|-----------------|--------|
| Total Visitors (30 days) | 4,291 |
|  Completed Purchases | 709 |
|  Overall Conversion Rate | **17%** |
|  Total Revenue Generated | **$76,192** |
|  Average Order Value | **$107.46** |
|  Revenue Earned Per Visitor | $17.76 |
|  Shoppers Who Abandoned Their Cart | **47%** |
|  Average Time from Visit to Purchase | 24.56 minutes |




### Funnel Drop-Off Analysis
 
| Journey Step | Users Lost | % Who Left |
|---|---|---|
| **Visit -> Add to Cart** | **2,953** | **69% <- biggest problem** |
| Add to Cart -> Checkout | 384 | 29% |
| Checkout -> Payment | 184 | 19% |
| Payment -> Purchase | 61 | 8% |
 
**[Finding:]()** The first step loses **10× more customers** than any other stage combined. This is where the business should focus first.
69% of visitors never even add to cart. But once they do, 92% go on to buy. The product isn't the problem - the path to the cart is.


 
### Traffic Source Performance
 
Not all traffic is equal. Email is the smallest channel but converts at nearly 5× the rate of Social.
 
| Channel | Visitors | Purchases | Conversion | Total Revenue | AOV | Revenue/Visitor |
|---------|----------|-----------|------------|--------------|-----|-----------------|
| **Email** | 449 | 152 | **34% 🏆** | $15,344 | $100.95 | **$34.17** |
| Paid Ads | 824 | 173 | 21% | $18,438 | $106.58 | $22.38 |
| Organic Search | 1,757 | 300 | 17% | $32,709 | $109.03 | $18.62 |
| Social Media | 1,261 | 84 | 7% | $9,700 | $115.48 | $7.69 |
 
**[Finding:]()** Social media users browse the most expensive items (highest AOV at $115.48) but rarely buy — a classic window-shopper pattern and a prime retargeting opportunity. Email drives the least traffic but the highest revenue per visitor by a wide margin.
 
<!-- ![Traffic Source](screenshots/05_traffic_source.png) -->
  
<!-- ![Revenue by Source](screenshots/10_traffic_source_revenue_kpis.png) -->
 ### Cart Abandonment Opportunity
 
Of the 1,338 people who added items to their cart:
- ✅ **709 bought** (53%)
- ❌ **629 walked away** (47%)
> **If cart abandonment dropped from 47% to the industry benchmark of 30%, that's ~$292,000 in additional annual revenue — at today's traffic levels, with zero extra ad spend.**
 
![Cart Abandonment](screenshots/12_cart_abandonment.png)
 
---
 
### Weekly Performance
 
Revenue was consistent across 4 full weeks with no significant spikes or crashes — stable, but not growing.
 
| Week | Visitors | Orders | Revenue | Avg. Order |
|------|----------|--------|---------|------------|
| Jan 4 | 1,009 | 168 | $17,870 | $106 |
| Jan 11 | 946 | 157 | $16,933 | $108 |
| Jan 18 | 1,050 | 171 | $17,580 | $103 |
| Jan 25 | 972 | 156 | $18,465 | $118 |
| Feb 1 | 314 | 57 | $5,345 | $94 |
 
*Feb 1 is a partial week — not a real decline.*
 
The weekly plateau signals the business has hit a ceiling. More traffic alone won't fix it — the funnel needs to convert better first.


---
## 🔍 Key SQL Queries
 
### [-> Full query file with all analysis](https://github.com/amansuren/E-commerce-Sales-Funnel-Analysis/blob/a151c52df176d07a4a0c8ebb000759145fdc6126/sql_queries/analysis.md)

### 1. Funnel Stage Conversion Rates
 
Builds the full funnel in a single CTE and computes step-by-step conversion rates — from page view all the way to purchase.
 
```sql
WITH funnel_stages AS (
  SELECT
    COUNT(DISTINCT CASE WHEN event_type = 'page_view'      THEN user_id END) AS stage_1_views,
    COUNT(DISTINCT CASE WHEN event_type = 'add_to_cart'    THEN user_id END) AS stage_2_cart,
    COUNT(DISTINCT CASE WHEN event_type = 'checkout_start' THEN user_id END) AS stage_3_checkout,
    COUNT(DISTINCT CASE WHEN event_type = 'payment_info'   THEN user_id END) AS stage_4_payment,
    COUNT(DISTINCT CASE WHEN event_type = 'purchase'       THEN user_id END) AS stage_5_purchase
  FROM `sql-project-487019.funnel_data_01.user_events`
  WHERE event_date >= TIMESTAMP(DATE_SUB('2026-02-03', INTERVAL 30 DAY))
)
SELECT
  stage_1_views,
  stage_2_cart,
  ROUND(stage_2_cart * 100 / stage_1_views)     AS view_to_cart_rate,
  stage_3_checkout,
  ROUND(stage_3_checkout * 100 / stage_2_cart)      AS cart_to_checkout_rate,
  stage_4_payment,
  ROUND(stage_4_payment * 100 / stage_3_checkout)  AS checkout_to_payment_rate,
  stage_5_purchase,
  ROUND(stage_5_purchase * 100 / stage_4_payment)   AS payment_to_purchase_rate,
  ROUND(stage_5_purchase * 100 / stage_1_views)     AS overall_conversion_rate
FROM funnel_stages

```
#### Output
![imagge](https://github.com/amansuren/sales-funnel-analysis-using-SQL/blob/87d370d742d3faf134538b961edc87302add99b1/screenshots/3.png)

### 2. Funnel Breakdown by Traffic Source
 
Groups the same funnel logic by `traffic_source` to compare channel quality — not just volume.
 
```sql
WITH source_funnel AS (
  SELECT
    traffic_source,
    COUNT(DISTINCT CASE WHEN event_type = 'page_view'  THEN user_id END) AS views,
    COUNT(DISTINCT CASE WHEN event_type = 'add_to_cart'THEN user_id END) AS carts,
    COUNT(DISTINCT CASE WHEN event_type = 'purchase'   THEN user_id END) AS purchases
  FROM `sql-project-487019.funnel_data_01.user_events`
  WHERE event_date >= TIMESTAMP(DATE_SUB('2026-02-03', INTERVAL 30 DAY))
  GROUP BY traffic_source
)
SELECT
  traffic_source, views, carts, purchases,
  ROUND(carts     * 100 / views) AS cart_conversion_rate,
  ROUND(purchases * 100 / views) AS purchase_conversion_rate,
  ROUND(purchases * 100 / carts) AS cart_to_purchase_rate
FROM source_funnel
ORDER BY purchases DESC
```
#### Output

![imagge](https://github.com/amansuren/sales-funnel-analysis-using-SQL/blob/87d370d742d3faf134538b961edc87302add99b1/screenshots/5.png)
 
 
### 3. Time-to-Convert Analysis
 
Filters to only users who purchased (`HAVING` clause), then measures time elapsed between each funnel stage to surface where friction slows the journey.
 
```sql
WITH user_journey AS (
  SELECT
    user_id,
    MIN(CASE WHEN event_type = 'page_view'  THEN event_date END) AS view_time,
    MIN(CASE WHEN event_type = 'add_to_cart'THEN event_date END) AS cart_time,
    MIN(CASE WHEN event_type = 'purchase'   THEN event_date END) AS purchase_time
  FROM `sql-project-487019.funnel_data_01.user_events`
  WHERE event_date >= TIMESTAMP(DATE_SUB('2026-02-03', INTERVAL 30 DAY))
  GROUP BY user_id
  HAVING MIN(CASE WHEN event_type = 'purchase' THEN event_date END) IS NOT NULL
)
SELECT
  COUNT(*) AS converted_users,
  ROUND(AVG(TIMESTAMP_DIFF(cart_time,     view_time,    MINUTE)), 2) AS avg_view_to_cart_min,
  ROUND(AVG(TIMESTAMP_DIFF(purchase_time, cart_time,    MINUTE)), 2) AS avg_cart_to_purchase_min,
  ROUND(AVG(TIMESTAMP_DIFF(purchase_time, view_time,    MINUTE)), 2) AS avg_total_journey_min
FROM user_journey;
```
#### Output

![imagge](https://github.com/amansuren/sales-funnel-analysis-using-SQL/blob/87d370d742d3faf134538b961edc87302add99b1/screenshots/7.png)
---
## 🔑 Key Takeaways
 
**1. The funnel has one big leak and one big strength.**
The view-to-cart rate (31%) is the only broken stage. Once customers reach checkout, they almost always complete the purchase (92%). Fix the top, and revenue follows.
 
**2. Email is the highest-quality channel - and it's underused.**
With a 34% purchase rate and $34.17 earned per visitor, email outperforms every other channel by a wide margin. Growing the email list is the single best growth lever available.
 
**3. Social media needs a strategy rethink.**
Social drives 29% of traffic but only 12% of purchases. Social visitors look at premium items (highest AOV: $115.48) but don't buy they need to be retargeted with paid ads to convert.
 
**4. The lower funnel is a strength, not a problem.**
92% of people who enter payment details complete the purchase. The checkout experience is working no redesign needed there.
 
**5. Revenue is plateaued.**
Four weeks of flat $17–18K revenue signals the business has hit a ceiling with its current approach. The path to growth is converting existing traffic better, not buying more of it.
## ✅ Recommendations
 
| Priority | Action | Why It Matters |
|----------|--------|----------------|
| 🔴 Fix first | A/B test product pages — better CTAs, urgency signals, social proof | Addresses the 69% view-to-cart drop |
| 🔴 Fix first | Remove forced account creation at checkout | Leading cause of cart abandonment globally |
| 🟠 Next | Show full costs (incl. shipping) earlier in the flow | Eliminates surprise at checkout |
| 🟠 Next | Set up a cart abandonment email sequence | Recovers high-intent customers automatically |
| 🟡 Grow | Invest in growing the email list | Highest-ROI channel at $34.17/visitor |
| 🟡 Grow | Retarget social visitors with paid ads | Converts window-shoppers browsing premium items |
 
---
 
## 🛠️ Tools Used
 
- **Google BigQuery** - Cloud SQL execution on event-level data
- **Standard SQL** - CTEs, conditional aggregations, date functions, timestamp arithmetic
- **Power BI** - Dashboard visualisation of funnel KPIs
- **GitHub** - Version control and project showcase

## 📋 Data Dictionary
See [`data_dictionary.md`](./data_dictionary.md) for full schema documentation, column definitions, and analytical limitations.























