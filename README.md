# US-Footwear
# us-64code-trade-analysis-USMCA
Analysis of U.S. 2025 footwear imports from a few different countries, looking for where there are opportunities to save through free trade agreements (FTAs) — especially with countries that are part of USMCA. 

**Tools:** Google BigQuery (SQL) · Looker Studio (dashboard)
**Data source:** USITC DataWeb — U.S. Imports for Consumption, HTS Chapter 64 (footwear), year 2025, trading partners: Canada, China, Colombia, Mexico.

📊 **Live dashboard:** _https://datastudio.google.com/reporting/6073bc7b-fa4b-4934-acd1-28ac5ac42153

💻**Whole query:**  https://console.cloud.google.com/bigquery?ws=!1m7!1m6!12m5!1m3!1smi-proyecto-data-987!2sus-central1!3sdf38012c-a7b4-473f-bd4b-c6a1da05979c!2e1

---

**Question**: how much duty does the U.S. pay on these footwear imports, and how much of it comes from FTA-partner countries whose goods might qualify for duty-free treatment?

## Data & method
- Pulled **410 rows** from USITC DataWeb (customs value, dutiable value, calculated duties) broken out by HTS line and country for one year (2025).
- Loaded the three measures into **BigQuery** and joined them into one `master` table on **HTS + country**.
- **Data-cleaning highlight:** the source exported values with comma decimals (e.g., `27644449,00`), which the loader misread and inflated every value 100×. I caught it with a sanity check against the source total, corrected it (÷100), and validated `SUM(customs_value)` ≈ $7.75B against DataWeb's reported total.
- Added an **effective duty rate** and a **USMCA-opportunity flag** (duty paid on Mexico/Canada-origin goods).

## Key SQL

**Building the cleaned master table (join + 100× fix + analysis columns):**
```sql
CREATE OR REPLACE TABLE `project.trade.master` AS
WITH joined AS (
  SELECT
    c.`HTS Number` AS hts,
    c.Country      AS country,
    c.Year         AS year,
    SAFE_CAST(c.`Customs Value`     AS FLOAT64)/100 AS customs_value,
    SAFE_CAST(d.`Dutiable Value`    AS FLOAT64)/100 AS dutiable_value,
    SAFE_CAST(u.`Calculated Duties` AS FLOAT64)/100 AS calculated_duties
  FROM `project.trade.customs`  c
  LEFT JOIN `project.trade.dutiable` d
    ON c.`HTS Number`=d.`HTS Number` AND c.Country=d.Country AND c.Year=d.Year
  LEFT JOIN `project.trade.duties`   u
    ON c.`HTS Number`=u.`HTS Number` AND c.Country=u.Country AND c.Year=u.Year
  WHERE c.`Data Type` IS NOT NULL
)
SELECT
  *,
  ROUND(SAFE_DIVIDE(calculated_duties, dutiable_value)*100, 1) AS effective_duty_rate_pct,
  CASE WHEN country IN ('Mexico','Canada') THEN calculated_duties ELSE 0 END AS potential_usmca_savings
FROM joined;
```

**Headline query (duties + USMCA opportunity by country):**
```sql
SELECT
  country,
  ROUND(SUM(calculated_duties))       AS duties_paid,
  ROUND(SUM(potential_usmca_savings)) AS usmca_savings_opportunity
FROM `project.trade.master`
GROUP BY country
ORDER BY usmca_savings_opportunity DESC;
```

## Findings

| Country | Duties paid | USMCA opportunity |
|---|---|---|
| China | ~$2,250,418,428 | $0 (not a free-trade partner) |
| Mexico | ~$6,079,237 | ~$6,079,237 |
| Canada | ~$1,370,860 | ~$1,370,860 |
| Colombia | ~$751,117 | $0 (covered by a separate FTA) |

### Three insights
1. **China carries essentially the entire duty burden** — ~$2.25B of ~$2.26B total footwear duties (**~99.6%**) — because it isn't a free-trade partner.
2. **Up to ~$7.45M in potential USMCA savings** — duties paid on Mexico ($6.08M) and Canada ($1.37M) origin footwear. This is an upper bound: the actual recoverable amount depends on how much of it meets USMCA rules of origin (see below).
3. **Colombia paid ~$751K** despite the U.S.–Colombia Trade Promotion Agreement (a separate preference-program opportunity), and footwear's **high effective duty rates** make accurate HTS classification and origin documentation especially valuable.

## Rules of origin & FTA context
- A product only enters duty-free under a free trade agreement if it meets that agreement's **rules of origin** — shipping from a partner country isn't enough on its own.
- **USMCA (Mexico & Canada)** → Product-specific rules live in **USMCA Chapter 4, Annex 4-B**, mirrored in **HTSUS General Note 11**. For footwear (HTS headings **6401–6405**), a good qualifies via a **tariff shift** (a change to 6401–6405 from any heading outside that group, except subheading 6406.10) **plus a regional value content (RVC) of at least 55% under the net cost method**. A de-minimis allowance lets up to 10% of value be non-originating. Qualifying goods are claimed with **SPI code "S"** and a certification of origin (nine minimum data elements; records kept 5 years).
- **COTPA/CTPA(U.S.–Colombia Trade Promotion Agreement). Colombia→** In force since **May 15, 2012**; most Colombian goods already enter the U.S. duty-free, with full phase-in by 2028. Eligible dutiable lines show **SPI code "CO"** in the HTSUS "Special" column; rules of origin are in **HTSUS General Note 34**. Importers can claim a **refund within one year** of import if a good qualified but the preference wasn't claimed at entry.
- **Bottom line**: the real opportunity is the share of the ~$7.45M (USMCA) and ~$751K (Colombia) that actually meets these rules — verifiable per shipment via the product's HTS origin rule, a certification of origin, and, for certainty, a **CBP binding ruling**.

## Caveats
- "USMCA opportunity" here uses **country of origin (Mexico/Canada) as a proxy**; actual eligibility depends on rules-of-origin analysis.
- 2025 figures are as-reported / may reflect a partial year.

## Skills demonstrated
SQL (joins, aggregation, data cleaning & validation) · BigQuery · trade-data domain (HTS classification, USMCA, customs valuation) · dashboarding (Looker Studio) · translating data into business insight.

## Sources
- USMCA text — Chapter 4 Rules of Origin & Annex 4-B (USTR): https://ustr.gov/trade-agreements/free-trade-agreements/united-states-mexico-canada-agreement
- USMCA footwear rule of origin summary (trade.gov): https://www.trade.gov/summary-usmca-fta-textiles
- CBP USMCA pages & FAQs: https://www.cbp.gov/trade/priority-issues/trade-agreements/free-trade-agreements/USMCA
- HTSUS — find a product's FTA rule via General Notes 11 (USMCA) & 34 (COTPA): https://hts.usitc.gov
- U.S.–Colombia TPA (CBP): https://www.cbp.gov/trade/free-trade-agreements/colombia
- U.S.–Colombia TPA (USTR): https://ustr.gov/trade-agreements/free-trade-agreements/colombia-tpa
- Trade data source — USITC DataWeb: https://dataweb.usitc.gov
