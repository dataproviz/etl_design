# 02 — Bronze Layer: `ncen_raw`

## Purpose

An immutable, append-only copy of every filing exactly as received. It is the source of truth for reprocessing whenever Silver parsing logic or schemas change.

## DDL

```sql
CREATE TABLE lake.bronze.ncen_raw (
  accession_no      STRING,        -- 0001234567-26-000123
  cik               STRING,
  form_type         STRING,        -- N-CEN, N-CEN/A
  filing_date       DATE,
  period_end_date   DATE,
  xsd_version       STRING,        -- from the XML namespace or schemaLocation
  raw_xml           STRING,        -- full primary_doc.xml, untouched
  source_uri        STRING,        -- s3://.../accession/primary_doc.xml
  xml_sha256        STRING,        -- dedup and integrity check
  ingest_ts         TIMESTAMP
) USING iceberg
PARTITIONED BY (years(filing_date))
TBLPROPERTIES (
  'format-version' = '2',
  'write.parquet.compression-codec' = 'zstd'
);
```

## Sample row

| accession_no | cik | form_type | filing_date | period_end_date | xsd_version | raw_xml |
|---|---|---|---|---|---|---|
| 0001234567-26-000123 | 0000123456 | N-CEN | 2026-03-14 | 2025-12-31 | v2 | `<edgarSubmission>…</edgarSubmission>` |

## Rules

- **Append-only.** Rows are never updated or deleted.
- **Amendments** (`N-CEN/A`) are stored as their own rows; "latest amendment" logic is resolved in Silver.
- **Deduplicate** on `xml_sha256` so re-ingesting the same document is a no-op.
- **Always record `xsd_version`.** It lets downstream layers distinguish "field did not exist in this version" from "filer left it blank".
