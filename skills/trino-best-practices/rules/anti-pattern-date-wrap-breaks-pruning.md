---
title: Do Not Wrap Time Columns in date() or date_trunc() Without Pruning Evidence
impact: CRITICAL
impactDescription: "Prevents common timestamp rewrites that quietly destroy pruning."
tags:
  - trino
  - anti-pattern
  - pruning
description: "Function-wrapping a timestamp column in the predicate path SHOULD be treated as a pruning risk until `EXPLAIN` proves otherwise."
alwaysApply: true
---

## Do Not Wrap Time Columns in date() or date_trunc() Without Pruning Evidence

This is a common lakehouse mistake: rewrite a clean range predicate into a function call and then wonder why scan cost explodes.

### Why it matters

Trino and its connectors usually prune most reliably on raw columns or explicit partition keys. `date(ts_col) = ...` and `date_trunc(...) = ...` are review smells because they often push function logic into the scan path.

### How to detect

- `WHERE date(ts_col) = DATE '...'`
- `WHERE date_trunc('day', ts_col) = ...`
- Partition pruning stopped working after a "readability" rewrite

### Correct

```sql
SELECT *
FROM iceberg.sales.orders
WHERE order_ts >= TIMESTAMP '2026-04-08 00:00:00'
  AND order_ts < TIMESTAMP '2026-04-09 00:00:00';
```

This is the default safe shape.

### Incorrect

```sql
SELECT *
FROM iceberg.sales.orders
WHERE date(order_ts) = DATE '2026-04-08';
```

This often blocks the pruning shape you actually wanted.

### Fix

- Use half-open ranges on the raw timestamp.
- If the table is partitioned by a derived key, filter that key directly or prove the transform-aware expression is effective with `EXPLAIN`.

### Reference

- https://trino.io/docs/current/optimizer/pushdown.html
- https://trino.io/docs/current/connector/hive.html
- https://trino.io/docs/current/connector/iceberg.html

