## US footwear trade analysis
Analysis of U.S 2024 footwear imports around the world, finding opportunities to save some from free trade agreements (FTA).

**Tools:** Google BigQuery (SQL). Tableau Public (dashboard). USITC (DataWeb).

**Data source:** USITC DataWeb, HTS Chapter 64, 2024, 130+ countries.

---
**Question**: How much of the imports of footwear to U.S are coming from countries with FTAs? Are there opportunities to save on tariffs from these countries?

## Data & method 
-	Downloaded datasets (customs value, dutiable value, calculate duties) with **3381 rows** from USITC DataWeb split out by HTS line (8 digits) and country on the year 2024. Also created another dataset with all U.S. FTA partners, **20 rows** one for each country, next to FTA name and how is classified on SPI code (code use to find the treatment and its benefits), finally since when its in place. 
-	Loaded the four datasets into **BigQuery**, then joined them into one `master_all` table on **HTS + country + FTA**.
-	**Data-cleaning highlight:** while building the query, found a naming issue with the year column (2024) that prevented the table from running — resolved by wrapping it in backticks.
-	After joining the tables, created three new analysis columns: **effective duty rate, FTA status, and FTA savings opportunity** — calculated only for the 20 countries that have free trade agreements with the U.S.
## Key findings 
1.	**Non-FTA countries dominate the duty burden —** 130 countries are paying $334M vs. 20 countries FTA partners are paying ~$535K combined. With China alone drives most of that $334M ($128M).
2.	**USMCA have the biggest FTA savings opportunity —** $296K in duties paid by Mexico and Canada that could potentially be recovered with proper rules-of-origin documentation. 
3.	**CAFTA-DR is underutilized —** 6 countries still paying $91K despite having full duty-free access.

## Caveats 
FTA savings figures are upper-bound estimates; actual eligibility depends on rules-of-origin analysis per shipment. So, you have to go into each country and rules depending on products. 

## Key SQL
**Master table building (join + analysis columns)**

```sql
-- Master table: all countries that traded footwear (HTS Ch. 64) with the U.S. in 2024
CREATE OR REPLACE TABLE `mi-proyecto-data-987.trade_all.master_all` AS
SELECT
  c.`HTS Number`  AS hts,
  c.Country       AS country,
  c.Description   AS description,
  SAFE_CAST(REPLACE(CAST(c.`2024` AS STRING), ',', '.') AS FLOAT64)      AS customs_value,
  SAFE_CAST(REPLACE(CAST(d.`2024` AS STRING), ',', '.') AS FLOAT64)      AS dutiable_value,
  SAFE_CAST(REPLACE(CAST(u.`2024` AS STRING), ',', '.') AS FLOAT64)      AS calculated_duties,
  -- Effective duty rate: duties ÷ dutiable value × 100, rounded to 1 decimal
  ROUND(SAFE_DIVIDE(
    SAFE_CAST(REPLACE(CAST(u.`2024` AS STRING), ',', '.') AS FLOAT64),
    SAFE_CAST(REPLACE(CAST(d.`2024` AS STRING), ',', '.') AS FLOAT64)
  ) * 100, 1)    AS effective_duty_rate_pct,
  CASE  -- Flag FTA agreement name per country, or 'No FTA' if none exists
    WHEN f.country IS NOT NULL THEN f.fta_name
    ELSE 'No FTA'
  END   AS fta_status,
  CASE -- SPI code used on customs documents to claim FTA preferential rate 
    WHEN f.country IS NOT NULL THEN f.spi_code
    ELSE NULL
  END     AS spi_code,
  CASE -- Upper-bound FTA savings: duties paid by FTA partners that could be $0 with proper preference claim
    WHEN f.country IS NOT NULL
    THEN SAFE_CAST(REPLACE(CAST(u.`2024` AS STRING), ',', '.') AS FLOAT64)
    ELSE 0
  END    AS fta_savings_opportunity
--Joining of all datasets in one, naming each column from the update data
FROM `mi-proyecto-data-987.trade_all.customs`  AS c
LEFT JOIN `mi-proyecto-data-987.trade_all.dutiable` AS d
  ON c.`HTS Number` = d.`HTS Number` AND c.Country = d.Country
LEFT JOIN `mi-proyecto-data-987.trade_all.duties` AS u
  ON c.`HTS Number` = u.`HTS Number` AND c.Country = u.Country
LEFT JOIN `mi-proyecto-data-987.trade_all.fta_partners` AS f
  ON UPPER(TRIM(c.Country)) = UPPER(TRIM(f.country))
WHERE c.`Data Type` IS NOT NULL;

-- Validation: duties and FTA savings opportunity grouped by agreement type
SELECT
  fta_status,
  COUNT(DISTINCT country)  AS countries, -- FTA vs non-FTA country count 
  ROUND(SUM(customs_value)) AS total_customs_value, 
  ROUND(SUM(calculated_duties)) AS total_duties, -- Total duties paid per FTA group
  ROUND(SUM(fta_savings_opportunity))  AS fta_opportunity -- Total recoverable savings per group
FROM `mi-proyecto-data-987.trade_all.master_all`
GROUP BY fta_status
ORDER BY total_duties DESC;

-- Country-level summary for Tableau world map: one row per country with totals
SELECT
  country,
  fta_status,
  spi_code,
  ROUND(SUM(customs_value))           AS total_customs_value,
  ROUND(SUM(calculated_duties))       AS total_duties,
  ROUND(AVG(effective_duty_rate_pct)) AS avg_duty_rate_pct,
  ROUND(SUM(fta_savings_opportunity)) AS fta_opportunity
FROM `mi-proyecto-data-987.trade_all.master_all`
GROUP BY country, fta_status, spi_code
ORDER BY total_duties DESC;
```


**Live dashboard:** https://public.tableau.com/views/USimportsoffootwearontheworld/Dashboard1?:language=en-US&:sid=&:redirect=auth&:display_count=n&:origin=viz_share_link

**Kaggle notebook:** https://www.kaggle.com/code/darlyncampos/us-footwear-trade-analysis-duties-fta-savings 

**Takeaways:** This project highlights savings opportunities that U.S. importers are not taking advantage of. It also shows potential for supply diversification beyond China, using countries that already have FTAs in place. The world map makes it easy to spot both the duty burden and the untapped opportunities at a glance. 
