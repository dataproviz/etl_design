# 03 — Silver Layer: Per-Section Typed Tables

## Purpose

Typed, queryable tables, one per logical section of the form. Each table's grain is defined by a repeating element in the XML (its *anchor* — see [05-unmapped-field-routing.md](05-unmapped-field-routing.md)).

| Table | Grain | Key |
|---|---|---|
| `ncen_filing` | One row per filing (registrant-level data) | `accession_no` |
| `ncen_series` | One row per fund series in the filing | `accession_no`, `series_id` |
| `ncen_providers` | One row per service provider, any role | `accession_no`, `series_id`, `provider_role`, name/LEI |

Every table carries:

- `xsd_version` — the schema version the row was parsed from.
- `unmapped` — a catch-all for XPaths not yet mapped to a typed column.

## DDL

```sql
-- Header: one row per filing
CREATE TABLE lake.silver.ncen_filing (
  accession_no        STRING,
  cik                 STRING,
  xsd_version         STRING,
  report_period_end   DATE,
  registrant_name     STRING,
  investment_co_type  STRING,     -- N-1A, N-2, N-3 ...
  file_number         STRING,
  lei                 STRING,
  is_first_filing     BOOLEAN,
  -- new XSD fields are added here via ALTER TABLE ADD COLUMN
  unmapped            VARIANT,    -- or MAP<STRING,STRING> / JSON STRING if engine lacks Iceberg v3
  is_latest_amendment BOOLEAN,
  parse_ts            TIMESTAMP
) USING iceberg
PARTITIONED BY (years(report_period_end))
TBLPROPERTIES ('format-version' = '3');

-- Child: one row per fund series in the filing
CREATE TABLE lake.silver.ncen_series (
  accession_no     STRING,
  series_id        STRING,        -- S000012345
  series_name      STRING,
  is_etf           BOOLEAN,
  is_index_fund    BOOLEAN,
  lending_activity STRUCT<
     is_securities_lending: BOOLEAN,
     lending_agents: ARRAY<STRUCT<name:STRING, lei:STRING, is_affiliated:BOOLEAN>>
  >,
  share_classes    ARRAY<STRUCT<class_id:STRING, class_name:STRING, ticker:STRING>>,
  xsd_version      STRING,
  unmapped         VARIANT
) USING iceberg
PARTITIONED BY (bucket(16, series_id))
TBLPROPERTIES ('format-version' = '3');

-- Child: one row per service provider, any role
CREATE TABLE lake.silver.ncen_providers (
  accession_no   STRING,
  series_id      STRING,          -- null if registrant-level
  provider_role  STRING,          -- ADVISER, SUB_ADVISER, CUSTODIAN, TRANSFER_AGENT, PRICING_SERVICE, ADMIN ...
  provider_name  STRING,
  lei            STRING,
  file_number    STRING,
  is_affiliated  BOOLEAN,
  country        STRING,
  attributes     VARIANT,         -- role-specific fields that differ by provider type
  xsd_version    STRING,
  unmapped       VARIANT
) USING iceberg
PARTITIONED BY (provider_role)
TBLPROPERTIES ('format-version' = '3');
```

## Design rules

- **Section isolation.** A new field in one section changes only that section's table.
- **Arrays for anything that can repeat.** An Iceberg struct cannot evolve into an array, so elements that could plausibly become repeating (`maxOccurs > 1`) are modelled as `ARRAY` from the start.
- **`unmapped` captures drift.** Unknown XPaths are stored as key/value pairs relative to the row's anchor, e.g. `{"newLiquidityFlag": "Y"}`.
- **Idempotent writes.** Silver is written with `MERGE INTO ... ON accession_no` (plus the child key), so re-parsing from Bronze can be run any number of times.
- **Unstable types stay as strings.** If an XSD type is likely to change in a non-widening way, keep it as `STRING` in Silver and cast in Gold.

## `unmapped` column type

| Engine support | Type |
|---|---|
| Iceberg v3 with `VARIANT` support in your engine | `VARIANT` |
| Otherwise | `MAP<STRING,STRING>` or JSON `STRING` |

Verify that every engine reading the tables (Spark, Athena/Trino, Snowflake, etc.) supports Iceberg v3 `VARIANT` before adopting it.

## Promoting a field

```sql
ALTER TABLE lake.silver.ncen_series
  ADD COLUMN liquidity_classification STRING AFTER is_index_fund;
-- then backfill from unmapped (or re-parse Bronze) for affected filings
```
