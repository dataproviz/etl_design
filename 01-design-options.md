# 01 — Design Options

Five options were considered. The chosen design combines options 1, 2, 3 and 5.

## Option 1 — Raw-first Bronze layer (safety net)

Store each filing's full XML as-is in an Iceberg table with filing metadata (`accession_no`, `cik`, `filing_date`, `xsd_version`, `raw_xml`, `ingest_ts`).

- **Pros:** zero data loss; any filing can be re-parsed and backfilled when new fields are mapped.
- **Cons:** not directly useful for querying.

## Option 2 — XSD-driven schema with additive Iceberg evolution

Generate the Iceberg schema from the XSD itself (e.g. spark-xml `XSDToSchema` or Python `xmlschema`) rather than inferring it from sample files — inference silently drops fields that don't appear in the sample.

When a new N-CEN technical specification is published, diff the new XSD against the current schema and apply `ALTER TABLE ... ADD COLUMN` (including nested struct fields), or write with `mergeSchema` enabled. Iceberg tracks columns by field ID, so adds and type widening are metadata-only and old snapshots remain readable.

- **Pros:** typed, queryable, controlled evolution.
- **Cons:** requires a schema-diff and DDL step per XSD release.

## Option 3 — Typed columns plus a catch-all `unmapped` column

Map known XPaths to typed columns; route every unrecognized XPath into an `unmapped` column (`VARIANT` on Iceberg v3, otherwise `MAP<STRING,STRING>` or JSON `STRING`). New fields are captured the day they appear, before any schema change. A monitor on distinct unmapped keys signals when to promote a field to a real column.

- **Pros:** nothing drops, even for unanticipated XSD changes.
- **Cons:** unmapped data is semi-structured until promoted.

## Option 4 — Long key-value (EAV) table

`accession_no, xpath, repeat_index / parent_path, value`. The schema never changes; pivoted views are built on top.

- **Pros:** absorbs any change with zero schema work.
- **Cons:** painful to query, loses types, heavy pivots. Suitable only as an audit/lineage layer — **not chosen** as the primary model.

## Option 5 — Normalize by section into child tables

N-CEN has repeating sections (fund series, advisers, sub-advisers, custodians, transfer agents, pricing services, brokers, …). Model each as its own Iceberg table keyed by `accession_no` plus series/entity ID, instead of one giant nested row.

- **Pros:** each table evolves independently; a new field in one section never touches the others.
- **Cons:** more tables and joins.

## Chosen combination

```
Bronze:  raw_xml + xsd_version            (Option 1, immutable)
Silver:  per-section typed tables          (Options 2 + 5)
         + unmapped VARIANT/MAP column     (Option 3)
Gold:    curated views that coalesce renamed/versioned fields
```
