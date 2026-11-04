# 04 — Gold Layer: Curated Views and Marts

## Purpose

Consumer-facing views that hide XSD version differences: they coalesce renamed or moved fields, apply casts, filter to the latest amendment, and expose business-friendly names.

## Example view

```sql
CREATE OR REPLACE VIEW lake.gold.fund_series_profile AS
SELECT
  f.cik,
  f.registrant_name,
  f.report_period_end,
  s.series_id,
  s.series_name,
  s.is_etf,
  -- handle a field renamed across XSD versions
  COALESCE(s.liquidity_classification,
           variant_get(s.unmapped, '$.legacyLiquidityFlag', 'string')) AS liquidity_class,
  size(s.share_classes)                                               AS share_class_count,
  f.xsd_version
FROM lake.silver.ncen_filing f
JOIN lake.silver.ncen_series s USING (accession_no)
WHERE f.is_latest_amendment = true;
```

## Suggested Gold outputs

| Output | Description |
|---|---|
| `fund_series_profile` | One row per series per reporting period, latest amendment only |
| `adviser_fund_map` | Adviser ↔ series relationships over time |
| `custodian_concentration` | Assets by custodian per period |
| `schema_drift_monitor` | Count of distinct `unmapped` keys per table and `xsd_version` |

## Drift monitor

```sql
CREATE OR REPLACE VIEW lake.gold.schema_drift_monitor AS
SELECT 'ncen_series' AS table_name, xsd_version, k AS unmapped_key,
       count(*) AS row_count, min(accession_no) AS sample_accession
FROM lake.silver.ncen_series
LATERAL VIEW explode(map_keys(unmapped)) t AS k     -- adapt for VARIANT
GROUP BY xsd_version, k
-- UNION ALL the same for ncen_filing and ncen_providers
;
```

Any row in this view means a field is arriving that has no typed column yet and is a candidate for promotion. Alert when the count goes above zero.
