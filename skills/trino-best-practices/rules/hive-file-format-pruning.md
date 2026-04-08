---
title: Respect Hive File-Format Pruning Limits
impact: HIGH
impactDescription: "Improves scan efficiency by aligning projection and predicate shape with what ORC and Parquet can exploit."
tags:
  - trino
  - hive
  - file-format
description: "Agents SHOULD reason about Hive reads in terms of projection, typed predicates, and file-format metadata instead of assuming every format prunes equally."
alwaysApply: false
---

## Respect Hive File-Format Pruning Limits

Hive performance is not just about partitions. File format and projection shape matter too.

### Why it matters

Trino can benefit from columnar formats such as ORC and Parquet because they expose enough metadata to avoid unnecessary reads in many cases. But that benefit is weaker when queries read every column, use weak predicates, or rely on text-heavy patterns that cannot be exploited early.

### How to detect

- The query uses `SELECT *` on wide ORC or Parquet tables.
- Filters are regex-heavy or function-wrapped instead of typed and selective.
- Review comments talk about pruning as if CSV, JSON, ORC, and Parquet behaved the same.

### Correct

```sql
SELECT orderkey, orderdate, totalprice
FROM hive.sales.orders_orc
WHERE orderdate >= DATE '2026-04-01'
  AND orderdate < DATE '2026-04-08';
```

Narrow projection plus typed predicates gives the connector and reader the best chance to avoid useless reads.

### Incorrect

```sql
SELECT *
FROM hive.sales.orders_orc
WHERE regexp_like(cast(orderdate AS varchar), '^2026-04');
```

This defeats the clean, typed filter shape that columnar readers benefit from.

### Fix

- Prefer explicit projection.
- Prefer typed predicates over text rewrites.
- When diagnosing scan cost, distinguish partition pruning from file-format-level reduction.

### Reference

- https://trino.io/docs/current/connector/hive.html
- https://trino.io/docs/current/optimizer/pushdown.html

