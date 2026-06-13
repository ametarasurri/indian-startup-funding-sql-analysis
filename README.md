# Indian Startup Funding Analysis (2016–2019)

---

> ### 🌐 Interactive Version Available
> For the full interactive experience — charts, tabbed navigation, and visual insights; please click the link below:
> **👉 [View Interactive Project Website](https://ametarasurri.github.io)**

---

## Project Overview

This project analyzes Indian startup funding patterns using SQL to extract actionable business insights from a real-world dataset. The analysis covers **3,044 funding records** spanning 2015–2020, focusing on city-level activity, sector trends, top startups, month-over-month funding movement, and investment stage distribution.

Five key business questions a growth analyst, VC analyst, or business strategist might ask about the Indian startup ecosystem.

---

## Dataset

- **Source:** [Indian Startup Funding — Kaggle](https://www.kaggle.com/datasets/sudalairajkumar/indian-startup-funding)
- **Size:** 3,044 rows, 9 columns
- **Coverage:** 2015 – mid 2020
- **Key columns:** `startup_name`, `date`, `industry_vertical`, `city`, `investors_name`, `investment_type`, `amount_usd`

---

## Data Cleaning & Preprocessing

Raw data contained several quality issues that were addressed before analysis:

- **Inconsistent city names** — Bengaluru and Bangalore treated as separate cities; Gurugram vs Gurgaon; New Delhi vs Delhi. Standardised to single values.
- **Non-numeric funding amounts** — `amount_usd` stored as text with commas (e.g. `20,00,00,000`). Cleaned and converted to numeric (REAL) type.
- **Inconsistent industry labels** — eCommerce, ECommerce, E-commerce all standardised to E-Commerce.
- **Redundant column** — `remarks` column (86% null) dropped as it added no analytical value.

Data was preprocessed using Python and Claude AI before SQL analysis.

**Why 2016–2019 only for trend analysis:**
- **2015** — `industry_vertical` column contains hyper-granular sub-vertical descriptions rather than broad sector labels, making year-over-year sector comparison impossible.
- **2020** — Only 7 records exist, likely due to mid-year data cutoff. Statistically insignificant for trend analysis.
- All non-geographic queries retain the full dataset including null-city records where relevant.

---

## Tools Used

- **SQLite** (via SQLiteOnline) — query execution
- **Python + Claude AI** — data preprocessing and cleaning
- **GitHub** — portfolio documentation
- **Google Antigravity** — creating interactive website

---

## Business Questions & SQL Analysis

---

### Query 1 — Which cities attract the most startup funding?

**Business Question:** Which Indian cities dominate startup investment activity?

```sql
SELECT 
    city,
    COUNT(*) AS total_deals,
    ROUND(COUNT(*) * 100.0 / (SELECT COUNT(*) FROM startup_funding_clean), 2) AS pct_of_total_deals,
    CONCAT(ROUND(SUM(amount_usd) / 1000000000, 2), 'B') AS total_funding_billions
FROM startup_funding_clean
WHERE city IS NOT NULL AND city != ''
GROUP BY city
ORDER BY total_deals DESC
LIMIT 10;
```

**Concepts used:** Aggregation, subquery for percentage calculation, CONCAT for formatting, NULL filtering

**Results:**

| City | Total Deals | % of All Deals | Total Funding |
|------|-------------|---------------|---------------|
| Bangalore | 841 | 27.63% | 18.46B |
| Mumbai | 567 | 18.63% | 4.92B |
| Delhi | 456 | 14.98% | 3.29B |
| Gurgaon | 337 | 11.07% | 3.87B |
| Pune | 105 | 3.45% | 0.63B |
| Hyderabad | 99 | 3.25% | 0.40B |
| Chennai | 97 | 3.19% | 0.72B |
| Noida | 92 | 3.02% | 1.26B |
| Ahmedabad | 38 | 1.25% | 0.11B |
| Jaipur | 30 | 0.99% | 0.15B |

**Insight:** Bangalore dominates with 27.63% of all deals and $18.46B in funding — more than 3x Mumbai's total. The top 4 cities (Bangalore, Mumbai, Delhi, Gurgaon) account for 72.31% of all deals, revealing heavy geographic concentration. Chennai ranks 7th with 97 deals and $0.72B, punching above its weight relative to deal count.

*Note: 180 records (5.9%) had missing city values and were excluded from geographic analysis. These records were retained in all non-geographic queries.*

---

### Query 2 — Which sectors are growing year over year?

**Business Question:** Which sectors consistently attract the most investment? Are any emerging?

```sql
WITH yearly_sector AS (
    SELECT 
        substr(date, 7, 4) AS year,
        industry_vertica AS sector,
        COUNT(*) AS total_deals,
        ROUND(SUM(amount_usd) / 1000000000, 2) AS funding_billions
    FROM startup_funding_clean
    WHERE industry_vertica IS NOT NULL 
    AND industry_vertica != ''
    AND substr(date, 7, 4) IN ('2016', '2017', '2018', '2019')
    GROUP BY year, industry_vertica
),
ranked AS (
    SELECT 
        year,
        sector,
        total_deals,
        funding_billions || 'B' AS funding_billions,
        DENSE_RANK() OVER (
            PARTITION BY year 
            ORDER BY total_deals DESC
        ) AS sector_rank
    FROM yearly_sector
)
SELECT *
FROM ranked
WHERE sector_rank <= 5
ORDER BY year, sector_rank;
```

**Concepts used:** CTE, DENSE_RANK window function, PARTITION BY, date extraction with substr(), NULL filtering

**Results (Top 3 per year shown):**

| Year | Sector | Total Deals | Funding | Rank |
|------|--------|-------------|---------|------|
| 2016 | Consumer Internet | 539 | 1.92B | 1 |
| 2016 | Technology | 190 | 0.68B | 2 |
| 2016 | E-Commerce | 166 | 0.97B | 3 |
| 2017 | Consumer Internet | 309 | 2.60B | 1 |
| 2017 | Technology | 223 | 0.98B | 2 |
| 2017 | E-Commerce | 96 | 5.94B | 3 |
| 2018 | Consumer Internet | 92 | 1.73B | 1 |
| 2018 | Technology | 62 | 0.54B | 2 |
| 2018 | Finance | 37 | 1.09B | 3 |
| 2019 | E-Commerce | 19 | 1.15B | 1 |
| 2019 | FinTech | 8 | 1.22B | 2 |
| 2019 | Finance | 8 | 0.30B | 2 |

**Insight:** Consumer Internet dominated every year from 2016–2018, consistently ranking #1 by deal volume. 2019 marked a clear shift — E-Commerce took the top spot while FinTech emerged strongly at #2, signalling a maturation of the Indian startup ecosystem away from pure consumer apps toward commerce and financial services. Healthcare showed consistent presence in the top 5 across all years, reflecting sustained investor interest in healthtech.

---

### Query 3 — Top 5 funded startups per major city

**Business Question:** Who are the biggest players in each major Indian city?

Cities were selected based on Query 1 results (top 8 cities by deal volume), ensuring analytical continuity across the project.

```sql
WITH startup_totals AS (
    SELECT 
        startup_name,
        city,
        COUNT(*) AS total_rounds,
        ROUND(SUM(amount_usd) / 1000000000, 2) AS total_funding_billions
    FROM startup_funding_clean
    WHERE startup_name IS NOT NULL 
    AND startup_name != ''
    AND city IS NOT NULL
    AND city != ''
    AND amount_usd IS NOT NULL
    GROUP BY startup_name, city
),
ranked AS (
    SELECT 
        startup_name,
        city,
        total_rounds,
        total_funding_billions || 'B' AS total_funding,
        DENSE_RANK() OVER (
            PARTITION BY city
            ORDER BY total_funding_billions DESC
        ) AS rank_in_city
    FROM startup_totals
)
SELECT *
FROM ranked
WHERE rank_in_city <= 5
AND city IN ('Bangalore', 'Mumbai', 'Delhi', 'Gurgaon', 'Pune', 'Hyderabad', 'Chennai', 'Noida')
ORDER BY city, rank_in_city;
```

**Concepts used:** CTE, DENSE_RANK window function, PARTITION BY city, multi-condition filtering

**Results (Selected highlights):**

| Startup | City | Rounds | Total Funding | Rank |
|---------|------|--------|---------------|------|
| Flipkart | Bangalore | 5 | 4.06B | 1 |
| Rapido Bike Taxi | Bangalore | 1 | 3.90B | 2 |
| Paytm | Bangalore | 2 | 1.46B | 3 |
| Snapdeal | Delhi | 2 | 0.70B | 1 |
| Zomato | Gurgaon | 4 | 0.43B | 1 |
| Nykaa | Mumbai | 5 | 0.21B | 4 |
| Paytm | Noida | 2 | 1.01B | 1 |

**Insight:** Bangalore is home to India's most heavily funded startups with Flipkart leading at $4.06B across 5 rounds. Funding figures reflect pre-2020 rounds only — companies like Zomato ($0.43B in dataset) went on to raise significantly more post-2020 before their 2021 IPO.

> **⚠️ Data Quality Flag:** Zoho appears in Chennai's rankings despite being famously 100% bootstrapped and never having taken external funding. This is a known scraping error in the source dataset — Zoho was likely misidentified as a funding recipient rather than an investor or acquirer. This record was flagged and noted but not removed, to preserve raw data integrity.

---

### Query 4 — Month over Month funding trends

**Business Question:** How does startup funding fluctuate month to month? Are there seasonal patterns or funding spikes?

```sql
WITH monthly_totals AS (
    SELECT 
        substr(date, 7, 4) AS year,
        substr(date, 4, 2) AS month,
        COUNT(*) AS total_deals,
        ROUND(SUM(amount_usd) / 1000000000, 2) AS funding_billions
    FROM startup_funding_clean
    WHERE amount_usd IS NOT NULL
    AND substr(date, 7, 4) IN ('2016', '2017', '2018', '2019')
    GROUP BY year, month
),
with_lag AS (
    SELECT 
        year,
        month,
        total_deals,
        funding_billions || 'B' AS funding_billions,
        LAG(funding_billions, 1) OVER (
            ORDER BY year, month
        ) || 'B' AS prev_month_funding,
        ROUND(
            (funding_billions - LAG(funding_billions, 1) OVER (ORDER BY year, month)) 
            * 100.0 / 
            LAG(funding_billions, 1) OVER (ORDER BY year, month), 
        2) AS pct_change
    FROM monthly_totals
)
SELECT *
FROM with_lag
ORDER BY year, month;
```

**Concepts used:** CTE, LAG window function, MoM percentage change calculation, date extraction with substr()

**Key findings:**

| Month | Event |
|-------|-------|
| Aug 2017 | +1517% spike — single massive round distorted the month |
| Mar 2017 | +679% spike |
| Aug 2019 | +1410% spike — likely Rapido's $3.9B round |
| Oct 2018 | -91% collapse — market-wide funding slowdown |
| Jan (any year) | NULL prev_month — expected, no prior month to compare |

**Insight:** Indian startup funding is highly lumpy — not smooth or seasonal. A single mega-round from a Flipkart or Rapido can distort an entire month's figures by over 1000%. This makes monthly deal count a more reliable trend indicator than monthly funding amount. The 2018 slowdown (visible Oct–Dec 2018) reflects a broader market correction that preceded the 2019 recovery.

---

### Query 5 — Which investment stage dominates each city?

**Business Question:** Are Indian cities primarily funding early-stage startups or scaling mature ones?

```sql
WITH city_stage AS (
    SELECT 
        city,
        CASE 
            WHEN investment_type LIKE '%Seed%' THEN 'Early Stage'
            WHEN investment_type LIKE '%Angel%' THEN 'Early Stage'
            WHEN investment_type LIKE '%Pre-Series%' THEN 'Early Stage'
            WHEN investment_type LIKE '%Series A%' THEN 'Growth Stage'
            WHEN investment_type LIKE '%Series B%' THEN 'Growth Stage'
            WHEN investment_type LIKE '%Series C%' THEN 'Late Stage'
            WHEN investment_type LIKE '%Series D%' THEN 'Late Stage'
            WHEN investment_type LIKE '%Series E%' THEN 'Late Stage'
            WHEN investment_type LIKE '%Private Equity%' THEN 'Late Stage'
            ELSE 'Other'
        END AS stage,
        COUNT(*) AS total_deals,
        ROUND(SUM(amount_usd) / 1000000000, 2) AS funding_billions
    FROM startup_funding_clean
    WHERE city IS NOT NULL AND city != ''
    AND investment_type IS NOT NULL AND investment_type != ''
    GROUP BY city, stage
),
ranked AS (
    SELECT 
        city,
        stage,
        total_deals,
        funding_billions || 'B' AS funding_billions,
        DENSE_RANK() OVER (
            PARTITION BY city
            ORDER BY total_deals DESC
        ) AS stage_rank
    FROM city_stage
)
SELECT *
FROM ranked
WHERE stage_rank = 1
AND city IN ('Bangalore', 'Mumbai', 'Delhi', 'Gurgaon', 'Pune', 'Hyderabad', 'Chennai', 'Noida')
ORDER BY total_deals DESC;
```

**Concepts used:** CASE WHEN with LIKE for bucketing, CTE, DENSE_RANK window function, PARTITION BY city

**Results:**

| City | Dominant Stage | Total Deals | Funding |
|------|---------------|-------------|---------|
| Bangalore | Late Stage | 417 | 14.19B |
| Mumbai | Late Stage | 280 | 4.41B |
| Delhi | Early Stage | 273 | 0.29B |
| Gurgaon | Late Stage | 164 | 3.54B |
| Hyderabad | Early Stage | 60 | 0.06B |
| Pune | Early Stage | 58 | 0.11B |
| Chennai | Late Stage | 53 | 0.67B |
| Noida | Early Stage | 50 | 0.06B |

**Insight:** India's startup funding landscape reveals a clear maturity divide. Bangalore leads Late Stage dominance with 417 deals worth $14.19B — confirming its position as home to India's most mature unicorns. Mumbai, Gurgaon, and Chennai follow the same Late Stage pattern. In contrast, Delhi, Hyderabad, Pune, and Noida are predominantly Early Stage ecosystems, suggesting these cities are nurturing the next generation of startups rather than scaling existing ones. This has direct implications for investors — Bangalore and Mumbai for returns, emerging cities for early bets.

---

## Key Findings Summary

1. **Bangalore is India's undisputed startup capital** — 27.63% of all deals, $18.46B in funding, and home to the most heavily funded startups including Flipkart, Rapido, Paytm, and Ola.

2. **Consumer Internet dominated 2016–2018** but 2019 signalled a pivot toward E-Commerce and FinTech — reflecting ecosystem maturation.

3. **Top 4 cities (Bangalore, Mumbai, Delhi, Gurgaon) account for 72% of all deals** — India's startup activity is heavily concentrated geographically.

4. **Funding is lumpy, not seasonal** — single mega-rounds can spike monthly totals by 1000%+. Deal count is a more stable trend metric than funding amount.

5. **Clear maturity divide between cities** — Bangalore, Mumbai, Gurgaon back Late Stage companies while Delhi, Pune, Hyderabad, Noida nurture Early Stage startups.

---

[GitHub: ametarasurri](https://github.com/ametarasurri)
