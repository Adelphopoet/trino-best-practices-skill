---
title: Reduce Scan Volume Early Instead of Hoping LIMIT Saves You
impact: HIGH
impactDescription: "Cuts scan, network, and memory work that LIMIT alone often does not avoid."
tags:
  - trino
  - limit
  - scan
description: "Agents SHOULD require real predicates or upstream reduction before heavy operators; `LIMIT` is not a substitute for scan pruning."
alwaysApply: false
---

## Reduce Scan Volume Early Instead of Hoping LIMIT Saves You

In Trino, `LIMIT` is not a reliable scan-reduction strategy. If the connector cannot push it down, Trino still has to read and process much more data than the final result size suggests.

### Why it matters

The expensive part is usually reading, filtering, repartitioning, and joining data. A small `LIMIT` at the top of the query does not undo the cost of scanning an unbounded table below it.

### How to detect

- A huge table is queried with `LIMIT 10` but no selective predicate.
- `LIMIT` appears only after joins or aggregations.
- `EXPLAIN` still shows wide scans and exchanges below the limit.

### Correct

```sql
SELECT orderkey, totalprice
FROM iceberg.sales.orders
WHERE orderdate >= DATE '2026-04-01'
ORDER BY orderdate DESC
LIMIT 100;
```

This combines a selective predicate with a small result target.

### Incorrect

```sql
SELECT orderkey, totalprice
FROM iceberg.sales.orders
LIMIT 100;
```

This is a sampling attempt, not a pruning strategy.

### Fix

- Add partition, date, or business-key predicates at scan level.
- Push filters into subqueries before joins.
- Treat `LIMIT` as result shaping, not primary optimization.

### Reference

- https://trino.io/docs/current/optimizer/pushdown.html
- https://trino.io/docs/current/sql/select.html

