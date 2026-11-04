# 05 — Routing Unmapped Fields to the Right Table

## Question

When the parser meets a field it does not recognize, how do we know whether it belongs in `ncen_filing.unmapped`, `ncen_series.unmapped`, or `ncen_providers.unmapped`?

## Answer: route by position in the XML tree, not by name

Each Silver table is anchored to one element in the document: the element whose repetition defines the table's grain (one row per occurrence). An unknown field goes to the `unmapped` column of the table whose anchor is its **nearest ancestor**.

## Section registry

The registry is the single hand-maintained artifact that decides which table a field belongs to. It is small and rarely changes.

| Silver table | Anchor XPath (defines one row) | Row key |
|---|---|---|
| `ncen_filing` | `/edgarSubmission` (root — the default) | `accession_no` |
| `ncen_series` | `/edgarSubmission/formData/seriesInfo/series` | `accession_no`, `series_id` |
| `ncen_providers` | `.../series/investmentAdvisers/adviser`, `.../custodians/custodian`, … (multiple anchors, each mapped to a `provider_role`) | `accession_no`, `series_id`, role, name/LEI |

## Nearest-anchor rule — examples

```
/edgarSubmission/formData/registrantInfo/newEsgFlag
   → nearest anchor = root               → ncen_filing.unmapped

/edgarSubmission/formData/seriesInfo/series[3]/newLiquidityFlag
   → nearest anchor = series             → ncen_series.unmapped (row for series[3])

/edgarSubmission/formData/seriesInfo/series[3]/custodians/custodian[2]/newRegion
   → nearest anchor = custodian          → ncen_providers.unmapped (that custodian's row)
```

Because routing follows the anchor *element*, the value lands on the exact row it belongs to — a new field under the third series goes onto the third series' row, not just the right table.

## Parser sketch (Python / lxml)

```python
from lxml import etree

# Section registry: anchor path (no indexes) -> (table, provider_role)
ANCHORS = {
    "/edgarSubmission": ("ncen_filing", None),
    "/edgarSubmission/formData/seriesInfo/series": ("ncen_series", None),
    "/edgarSubmission/formData/seriesInfo/series/investmentAdvisers/adviser": ("ncen_providers", "ADVISER"),
    "/edgarSubmission/formData/seriesInfo/series/custodians/custodian":       ("ncen_providers", "CUSTODIAN"),
}

# Known paths per table, relative to the anchor (generated from the XSD mapping config)
MAPPED = {
    "ncen_series": {"seriesId", "seriesName", "isEtf", "isIndexFund"},  # ...
    # "ncen_filing": {...}, "ncen_providers": {...}
}

def strip(tag):
    """Drop the XML namespace from a tag."""
    return etree.QName(tag).localname

def generic_path(el):
    """/a/b/c without [n] indexes."""
    ancestors = reversed(list(el.iterancestors()))
    return "/" + "/".join(strip(a.tag) for a in ancestors) + "/" + strip(el.tag)

def nearest_anchor(el):
    node = el
    while node is not None:
        p = generic_path(node)
        if p in ANCHORS:
            return node, p
        node = node.getparent()

def route_unmapped(root):
    """Yield (table, anchor_element, relative_path, value) for every unknown leaf."""
    for el in root.iter():
        if not isinstance(el.tag, str) or len(el):   # leaves only; skip comments/PIs
            continue
        anchor_el, anchor_path = nearest_anchor(el)
        table, _ = ANCHORS[anchor_path]
        rel = generic_path(el)[len(anchor_path):].lstrip("/")
        if rel not in MAPPED.get(table, set()):
            yield table, anchor_el, rel, el.text
        # attributes are leaves too
        for k, v in el.attrib.items():
            rel_a = f"{rel}/@{k}" if rel else f"@{k}"
            if rel_a not in MAPPED.get(table, set()):
                yield table, anchor_el, rel_a, v
```

Group the yielded tuples by `anchor_el`; each group becomes that row's `unmapped` object, e.g. `{"newLiquidityFlag": "Y"}`.

Keys are stored **relative to the anchor**, so they are stable regardless of which series index they came from, and they match the column name you will create when the field is promoted.

## Edge cases

| Case | Handling |
|---|---|
| New **repeating** element inside a known section (e.g. `esgStrategy` list under `series`) | Routes to `ncen_series`, stored as an array: `{"esgStrategies/esgStrategy": ["A", "B"]}`. Detect repetition by counting same-named siblings under the anchor. |
| Entire **new repeating section** (e.g. a new provider type) | Falls to the nearest existing anchor (series or root) so nothing is lost. The drift monitor flags unmapped keys whose values are arrays of objects as candidates for a new anchor or `provider_role`. |
| Field **moves between sections** across XSD versions | Routes by its current position. `xsd_version` is on every row; reconcile old and new locations with `COALESCE` in a Gold view. |
| Unknown **attributes** | Treated as leaves with `@name` suffix on the relative path. |

## Deciding ahead of time from the XSD

When a new XSD version is published, every new element in the diff has a parent `complexType`. Apply the same nearest-anchor rule to the XSD path in CI; it tells you which table — and which `ALTER TABLE` — each new field needs *before* any filing arrives. Runtime routing to `unmapped` is the safety net for anything the diff misses.
