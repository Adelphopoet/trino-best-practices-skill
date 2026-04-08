---
title: Do Not Assume CTE Materialization
impact: CRITICAL
impactDescription: "Prevents duplicated work and non-deterministic surprises from repeated WITH usage."
tags:
  - trino
  - cte
  - select
description: "In Trino, `WITH` relations are inlined; they are not a reliable materialization boundary."
alwaysApply: true
---

## Do Not Assume CTE Materialization

In Trino, a `WITH` clause is a readability tool, not a guaranteed caching or materialization mechanism.

### Why it matters

The Trino `SELECT` documentation explicitly warns that `WITH` SQL is inlined anywhere the named relation is used. Reusing an expensive CTE multiple times can repeat work, and reusing a non-deterministic CTE can produce different results.

### How to detect

- The same CTE is referenced multiple times.
- The CTE contains `random()`, `now()`, or other non-deterministic logic.
- The review assumes the CTE is executed once and reused.

### Correct

```sql
CREATE TABLE scratch.daily_orders AS
SELECT orderkey, custkey, totalprice
FROM iceberg.sales.orders
WHERE orderdate = DATE '2026-04-08';

SELECT a.orderkey
FROM scratch.daily_orders a
JOIN scratch.daily_orders b ON a.custkey = b.custkey;
```

If single execution matters, materialize it explicitly.

### Incorrect

```sql
WITH sampled AS (
    SELECT orderkey, random() AS r
    FROM tpch.sf1.orders
)
SELECT a.orderkey
FROM sampled a
JOIN sampled b ON a.orderkey = b.orderkey
WHERE a.r > 0.5;
```

This assumes `sampled` is reused as a single result, which Trino does not guarantee.

### Fix

- Use a temp table, CTAS, or permanent staging object when reuse matters.
- Treat CTEs as inline query fragments.
- Call out non-deterministic repeated CTE use as a correctness risk, not only a performance risk.

### Reference

- https://trino.io/docs/current/sql/select.html

