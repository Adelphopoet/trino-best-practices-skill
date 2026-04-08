---
title: Watch for Join Row Explosion
impact: CRITICAL
impactDescription: "Catches many-to-many joins that turn a reasonable scan into an unbounded memory and exchange problem."
tags:
  - trino
  - join
  - explain-analyze
description: "Agents MUST check whether a join multiplies rows unexpectedly instead of assuming the key cardinality is safe."
alwaysApply: true
---

## Watch for Join Row Explosion

Slow Trino joins are often cardinality bugs, not engine bugs.

### Why it matters

If both join inputs contain duplicate keys, Trino will happily multiply rows. That amplifies exchange, memory use, spill pressure, and downstream aggregation cost.

### How to detect

- `EXPLAIN ANALYZE` shows join output rows far above both inputs.
- The query joins fact-to-fact or otherwise many-to-many keys.
- A `DISTINCT` appears after the join as damage control.

### Correct

```sql
WITH shipment AS (
    SELECT orderkey, max(shipped_at) AS shipped_at
    FROM hive.sales.shipments
    GROUP BY orderkey
)
SELECT o.orderkey, s.shipped_at
FROM hive.sales.orders o
LEFT JOIN shipment s ON s.orderkey = o.orderkey;
```

The right side is reduced to one row per key before joining.

### Incorrect

```sql
SELECT o.orderkey, s.shipped_at
FROM hive.sales.orders o
JOIN hive.sales.shipments s ON s.orderkey = o.orderkey
JOIN hive.sales.events e ON e.orderkey = o.orderkey;
```

If shipments and events both contain multiple rows per order, the join explodes.

### Fix

- Validate key cardinality before joining.
- Aggregate or deduplicate to the intended grain.
- Re-run `EXPLAIN ANALYZE` and compare join output to join input.

### Reference

- https://trino.io/docs/current/sql/explain-analyze.html
- https://trino.io/docs/current/optimizer/cost-based-optimizations.html

