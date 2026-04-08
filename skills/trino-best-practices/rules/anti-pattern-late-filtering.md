---
title: Do Not Filter Late After Expensive Operators
impact: HIGH
impactDescription: "Prevents joins, windows, and aggregations from processing rows that should have been removed at scan time."
tags:
  - trino
  - anti-pattern
  - filtering
description: "Filters that can be applied before joins, windows, or aggregations SHOULD not be left for outer query blocks."
alwaysApply: true
---

## Do Not Filter Late After Expensive Operators

Late filtering makes Trino do work it never needed to do.

### Why it matters

If a predicate can be evaluated at scan time but is left above a join, window, or aggregation, the expensive operator processes too many rows. That inflates exchange volume, memory pressure, and runtime.

### How to detect

- The outer query filters columns that originate from the scan and could have been filtered earlier.
- `EXPLAIN ANALYZE` shows large input to a join or window and tiny output after the filter.
- A top-level `WHERE` clause is doing cleanup that belongs in the input relations.

### Correct

```sql
WITH filtered_orders AS (
    SELECT orderkey, custkey, totalprice
    FROM iceberg.sales.orders
    WHERE orderdate >= DATE '2026-04-01'
)
SELECT custkey, sum(totalprice)
FROM filtered_orders
GROUP BY custkey;
```

The filter happens before the aggregation.

### Incorrect

```sql
SELECT custkey, sum(totalprice)
FROM (
    SELECT *
    FROM iceberg.sales.orders
) o
WHERE orderdate >= DATE '2026-04-01'
GROUP BY custkey;
```

That outer filter is later than it needs to be.

### Fix

- Push predicates into the earliest legal relation.
- Re-check the plan for smaller scan and join inputs.
- If the filter cannot move because of semantics, say that explicitly.

### Reference

- https://trino.io/docs/current/sql/explain-analyze.html
- https://trino.io/docs/current/optimizer/pushdown.html

