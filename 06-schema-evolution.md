# 06 — Schema Evolution and Automation

## How each kind of XSD change is handled

| XSD change | Handling |
|---|---|
| **New element** | Add a nullable column (`ALTER TABLE ... ADD COLUMN`). Old rows read as `NULL`. Until the column exists, values land in `unmapped`. |
| **Type change** (e.g. string → decimal) | Iceberg allows only widening: int → long, float → double, decimal precision increase. If the XSD type is unstable, keep it as `STRING` in Silver and cast in Gold. |
| **Single → repeating** (`maxOccurs` 1 → unbounded) | A struct cannot evolve into an array. Model any element that could plausibly repeat as `ARRAY` from day one. |
| **Rename or move** | Treat as a **new column**. Keep the old one and `COALESCE(new, old)` in a Gold view. Do not use Iceberg's column rename — the old data genuinely came from a different element. |
| **Removed element** | Keep the column; it simply stops being populated for new `xsd_version` values. Never drop columns. |

Always store `xsd_version` on every row, so "field did not exist yet" can be distinguished from "filer left it blank".

## Automation

### 1. CI schema diff (per XSD release)

1. Register the new XSD in the XSD version registry.
2. Diff it against the previous version (added / removed / changed elements and types).
3. For each added element, apply the nearest-anchor rule ([05](05-unmapped-field-routing.md)) to find its target table.
4. Generate `ALTER TABLE ... ADD COLUMN` DDL and the mapping-config update for review.
5. Flag non-additive changes (type narrowing, struct→array, moves) for manual design.

### 2. Runtime drift detection (per filing)

1. Compare the XPaths found in each filing against the mapped set.
2. Write unknown paths into the owning row's `unmapped` column.
3. Surface counts in `gold.schema_drift_monitor`; alert when any count is above zero.

### 3. Backfill after promotion

Once a field is promoted to a typed column:

1. Identify affected filings (those with the key in `unmapped`, or by `xsd_version`).
2. Re-parse from Bronze with the updated mapping.
3. Write with `MERGE INTO ... ON accession_no` (plus child key) — idempotent and safe to re-run.

```sql
MERGE INTO lake.silver.ncen_series t
USING reparsed_series s
ON  t.accession_no = s.accession_no
AND t.series_id    = s.series_id
WHEN MATCHED THEN UPDATE SET *
WHEN NOT MATCHED THEN INSERT *;
```

## Why this never loses data

1. **Bronze** keeps every raw document forever.
2. **Silver** captures anything unrecognized in `unmapped` on the correct row.
3. **Promotion + backfill** turns captured fields into typed columns at any time, for all history.
