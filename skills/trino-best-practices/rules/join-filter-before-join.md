---
title: Filter Join Inputs Before Joining
impact: CRITICAL
impactDescription: "Reduces join input volume, exchange cost, and build-side memory before the hash table is created."
tags:
  - trino
  - join
  - filtering
description: "Dynamic filtering is helpful, but agents SHOULD still push static filters into join inputs as early as possible."
alwaysApply: true
---

## Filter Join Inputs Before Joining

A join does not become cheap because the final `WHERE` clause is selective. Trino still has to build and probe the join from whatever rows reach the join node.

### Why it matters

Static filters reduce the data that has to be scanned, exchanged, and hashed. Dynamic filtering may help later, but it is not a substitute for obvious pre-join filtering.

### How to detect

- Filters on either table appear only after the join.
- The build side is not filtered even though the SQL can filter it.
- `EXPLAIN ANALYZE` shows a large join input and a small post-join output.

### Correct

```sql
SELECT o.orderkey, c.region
FROM (
    SELECT orderkey, custkey
    FROM iceberg.sales.orders
    WHERE orderdate >= DATE '2026-04-01'
) o
JOIN (
    SELECT custkey, region
    FROM iceberg.sales.customers
    WHERE nationkey = 1
) c
  ON c.custkey = o.custkey;
```

Both sides are reduced before the hash join is built.

### Incorrect

```sql
SELECT o.orderkey, c.region
FROM iceberg.sales.orders o
JOIN iceberg.sales.customers c
  ON c.custkey = o.custkey
WHERE o.orderdate >= DATE '2026-04-01'
  AND c.nationkey = 1;
```

This can leave more work inside the join than necessary.

### Fix

- Push static filters into each input relation.
- Make the build side small before the join.
- Use the plan to confirm scan predicates moved down.

### Reference

- https://trino.io/docs/current/admin/dynamic-filtering.html
- https://trino.io/docs/current/optimizer/cost-based-optimizations.html

