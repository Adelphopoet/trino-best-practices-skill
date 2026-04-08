---
title: Treat Lakehouse as a Wrapper, Not a New Semantics Layer
impact: CRITICAL
impactDescription: "Prevents wrong guidance that ignores the underlying table format connector."
tags:
  - trino
  - lakehouse
  - connectors
description: "The Lakehouse connector combines Hive, Iceberg, Delta Lake, and Hudi behavior; agents MUST anchor advice in the underlying table type."
alwaysApply: true
---

## Treat Lakehouse as a Wrapper, Not a New Semantics Layer

The Lakehouse connector is not permission to invent generic behavior.

### Why it matters

The Trino documentation explicitly says Lakehouse combines Hive, Iceberg, Delta Lake, and Hudi features, and that behavior comes from the underlying connectors. Session properties, table properties, and operational caveats do not become magically uniform.

### How to detect

- The catalog uses `connector.name=lakehouse`.
- The review gives one-size-fits-all advice without identifying whether the table type is `HIVE`, `ICEBERG`, `DELTA`, or `HUDI`.
- The answer attributes connector behavior to Trino core.

### Correct

```sql
CREATE TABLE lake.sales.orders (
    order_id bigint,
    order_date date
)
WITH (
    type = 'ICEBERG',
    format = 'PARQUET'
);
```

Any advice for this table must be Iceberg-aware, even though it lives behind Lakehouse.

### Incorrect

```sql
-- Review comment:
-- "Lakehouse tables all support the same pruning, snapshot, and retry behavior."
```

That is made up.

### Fix

- Identify the table type first.
- Route Hive tables to Hive rules and Iceberg tables to Iceberg rules.
- If the table type is `DELTA` or `HUDI`, state that this package only provides wrapper-boundary guidance for those formats and avoid inventing deeper connector-specific behavior.
- Mention uncertainty if the table type is not visible.

### Reference

- https://trino.io/docs/current/connector/lakehouse.html
- https://trino.io/docs/current/connector.html
