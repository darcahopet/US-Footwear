## US footwear trade analysis
Analysis of U.S. 2024 footwear imports around the world, finding opportunities to save through free trade agreements (FTAs).

**Tools:** Google BigQuery (SQL) · Tableau Public (dashboard) · USITC DataWeb

**Data source:** USITC DataWeb, HTS Chapter 64, 2024, 130+ countries

---

**Question:** How much of U.S. footwear imports come from countries with FTAs, and is there unclaimed tariff savings among them?

## Data & method

- Downloaded customs value, dutiable value, and calculated duties from USITC DataWeb — 6,513 rows split by HTS line, country, and program for 2024. Built a fourth reference dataset covering all 20 U.S. FTA partners, with FTA name, SPI code, and effective date.
- Loaded all four into **BigQuery** and joined them into one `master_all` table on **HTS + country + program**, aggregating each source table to one row per key before joining to guarantee no duplication.
- **Data validation:** after the initial build, reconciled total duties against DataWeb's reported figure and found a **10x understatement** — traced to a locale mismatch introduced when source files were re-saved through Excel (semicolon delimiters, comma decimals). Rebuilt the ingestion path from a clean export and confirmed the corrected total ties to DataWeb exactly.
- Replaced a naive average duty rate with a **dutiable-value-weighted rate**, since a simple average let low-value rows with near-zero dutiable value distort the figure.

## Key findings

1. **Non-FTA countries dominate the duty burden** — $3.341M across 130+ countries, versus 5.449K combined among 20 FTA partners. China alone accounts for $1.280M.
2. **USMCA has the largest single recovery opportunity** — $2.960K in duties paid by Mexico- and Canada-origin goods that could potentially be recovered with proper rules-of-origin documentation.
3. **CAFTA-DR is underutilized** — 6 countries still paying $912K despite duty-free eligibility.
4. **Dutiable-weighted effective duty rate: 12.8%** across HTS Chapter 64.

## Caveats

FTA savings figures are upper-bound estimates; actual eligibility depends on rules-of-origin analysis per shipment and product.

## Key SQL

**Master table build (grouped joins, no duplication):**

```sql
CREATE OR REPLACE TABLE `mi-proyecto-data-987.trade_all.master_all` AS

WITH cu AS (
  SELECT
    `HTS Number`          AS hts,
    UPPER(TRIM(Country))  AS country_key,
    UPPER(TRIM(Program))  AS program_key,
    ANY_VALUE(Country)    AS country,
    ANY_VALUE(Program)    AS program,
    SUM(`2024`)           AS customs_value
  FROM `mi-proyecto-data-987.trade_all.customs`
  GROUP BY hts, country_key, program_key
),
du AS (
  SELECT
    `HTS Number`          AS hts,
    UPPER(TRIM(Country))  AS country_key,
    UPPER(TRIM(Program))  AS program_key,
    SUM(`2024`)           AS dutiable_value
  FROM `mi-proyecto-data-987.trade_all.dutiable`
  GROUP BY hts, country_key, program_key
),
dt AS (
  SELECT
    `HTS Number`          AS hts,
    UPPER(TRIM(Country))  AS country_key,
    UPPER(TRIM(Program))  AS program_key,
    SUM(`2024`)           AS calculated_duties
  FROM `mi-proyecto-data-987.trade_all.duties`
  GROUP BY hts, country_key, program_key
),
ft AS (
  SELECT
    UPPER(TRIM(country))  AS country_key,
    ANY_VALUE(fta_name)   AS fta_name,
    ANY_VALUE(spi_code)   AS spi_code
  FROM `mi-proyecto-data-987.trade_all.fta_partners`
  GROUP BY country_key
)

SELECT
  cu.hts, cu.country, cu.program,
  cu.customs_value, du.dutiable_value, dt.calculated_duties,
  ROUND(SAFE_DIVIDE(dt.calculated_duties, du.dutiable_value) * 100, 1) AS effective_duty_rate_pct,
  CASE WHEN ft.country_key IS NOT NULL THEN ft.fta_name ELSE 'No FTA' END AS fta_status,
  CASE WHEN ft.country_key IS NOT NULL THEN ft.spi_code ELSE NULL END AS spi_code,
  CASE WHEN ft.country_key IS NOT NULL THEN dt.calculated_duties ELSE 0 END AS fta_savings_opportunity
FROM cu
LEFT JOIN du ON cu.hts = du.hts AND cu.country_key = du.country_key AND cu.program_key = du.program_key
LEFT JOIN dt ON cu.hts = dt.hts AND cu.country_key = dt.country_key AND cu.program_key = dt.program_key
LEFT JOIN ft ON cu.country_key = ft.country_key;
```

**Validation query (row count, total duties, weighted rate):**

```sql
SELECT
  COUNT(*)                                                  AS row_count,
  ROUND(SUM(calculated_duties))                             AS total_duties,
  ROUND(SUM(calculated_duties)/SUM(dutiable_value)*100, 1)  AS weighted_rate
FROM `mi-proyecto-data-987.trade_all.master_all`;
```

**Live dashboard:** 
https://public.tableau.com/app/profile/darlin.campos/viz/USimportsoffootwear2024Update/Dashboard1

**Kaggle notebook:**   
https://www.kaggle.com/code/darlyncampos/us-footwear-trade-analysis-duties-fta-savings


**Takeaways:** Beyond the dollar findings, this project surfaced a real data-integrity issue — a 10x understatement caused by an encoding mismatch in the ingestion pipeline — which was traced to root cause and verified against source. That reconciliation discipline is as central to the project as the FTA analysis itself.

**Takeaways:** This project highlights savings opportunities that U.S. importers are not taking advantage of. It also shows potential for supply diversification beyond China, using countries that already have FTAs in place. The world map makes it easy to spot both the duty burden and the untapped opportunities at a glance. 
