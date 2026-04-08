---
title: Aggregate Before the Join When Semantics Allow
impact: HIGH
impactDescription: "Reduces join input size, network exchange, and build-side memory."
tags:
  - trino
  - aggregation
  - joins
description: "When the final result is grouped by a join key or dimension, agents SHOULD consider pre-aggregating the many-side before joining."
alwaysApply: false
---

## Aggregate Before the Join When Semantics Allow

Many slow Trino queries join raw fact rows first and aggregate later, even when the join only needs aggregated facts.

### Why it matters

Pre-aggregation shrinks rows before the join, which reduces exchange volume and memory pressure on hash joins. This is often a bigger win than session tuning.

### How to detect

- A large fact table is joined to a dimension and only then grouped by the join key or a dimension attribute.
- `EXPLAIN ANALYZE` shows massive join input and tiny final grouped output.
- The joined dimension is used only for attributes after grouping.

### Correct

```sql
WITH fact AS (
    SELECT custkey, sum(totalprice) AS revenue
    FROM iceberg.sales.orders
    WHERE orderdate >= DATE '2026-04-01'
    GROUP BY custkey
)
SELECT c.region, sum(f.revenue)
FROM fact f
JOIN iceberg.sales.customers c ON c.custkey = f.custkey
GROUP BY 1;
```

This cuts the join down to one row per customer instead of one row per order.

### Incorrect

```sql
SELECT c.region, sum(o.totalprice)
FROM iceberg.sales.orders o
JOIN iceberg.sales.customers c ON c.custkey = o.custkey
WHERE o.orderdate >= DATE '2026-04-01'
GROUP BY 1;
```

This may join far more rows than the result shape actually needs.

### Fix

- Pre-aggregate on the fact-side join key when the final semantics permit it.
- Keep the join at the smallest valid grain.
- Do not pre-aggregate if the downstream logic needs row-level detail.

### Reference

- https://trino.io/docs/current/sql/select.html
- https://trino.io/docs/current/optimizer/cost-based-optimizations.html

