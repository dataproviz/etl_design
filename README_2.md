# SEC N-CEN XML → Apache Iceberg: Design Overview

## Problem

SEC Form N-CEN filings are XML documents validated against an XSD that the SEC revises over time. New releases add elements, occasionally change types or cardinality, and sometimes move or rename fields.

The transformation into Iceberg tables must **never lose a field** — including fields introduced by an XSD version the pipeline has not been updated for yet.

## Guiding principle

> Never let the parsing step be the only copy of the data.

Raw XML is preserved, typed tables evolve additively, and anything the parser does not recognize is captured rather than dropped.

## Architecture (medallion)

```
EDGAR N-CEN filings          XSD version registry
  (primary_doc.xml)            (schema diff → DDL)
          │                          │
          ▼                          ▼
┌──────────────────────────────────────────────┐
│ Bronze: ncen_raw                             │
│ one row per filing, raw XML, immutable       │
└──────────────────────────────────────────────┘
          │  parse (re-runnable)
          ▼
┌──────────────────────────────────────────────┐
│ Silver: typed per-section tables             │
│   ncen_filing │ ncen_series │ ncen_providers │
│ every table: xsd_version + unmapped VARIANT  │
└──────────────────────────────────────────────┘
          │
          ▼
┌──────────────────────────────────────────────┐
│ Gold: curated views and marts                │
│ coalesced renames, casts, business names     │
└──────────────────────────────────────────────┘
```

| Layer | Guarantee |
|---|---|
| Bronze | Nothing is ever lost; any filing can be reprocessed |
| Silver | Typed and queryable; evolves additively; drift captured in `unmapped` |
| Gold | Consumers are insulated from XSD version differences |

## Documents

| File | Contents |
|---|---|
| [01-design-options.md](01-design-options.md) | The design options considered and why they are combined |
| [02-bronze-layer.md](02-bronze-layer.md) | `ncen_raw` table design |
| [03-silver-layer.md](03-silver-layer.md) | Per-section typed tables, `unmapped` column, merge strategy |
| [04-gold-layer.md](04-gold-layer.md) | Curated views and drift monitoring |
| [05-unmapped-field-routing.md](05-unmapped-field-routing.md) | How an unknown field is routed to the correct table and row |
| [06-schema-evolution.md](06-schema-evolution.md) | XSD change handling, automation, and backfill |

> Element and column names throughout are illustrative. Map them to the actual element names in the N-CEN XSD version in use.
