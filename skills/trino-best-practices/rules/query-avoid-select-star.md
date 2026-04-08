---
title: Avoid SELECT Star on Trino Lakehouse Tables
impact: MEDIUM
impactDescription: "Reduces unnecessary column reads and avoids accidental dependence on hidden or problematic columns."
tags:
  - trino
  - projection
  - query-shape
description: "Agents SHOULD prefer explicit projection over `SELECT *`, especially on wide Hive and Iceberg tables."
alwaysApply: false
---

## Avoid SELECT Star on Trino Lakehouse Tables

`SELECT *` is cheap to type and expensive to review. On Trino, it weakens projection pushdown and makes wide-table scans harder to reason about.

### Why it matters

Projection pushdown lets the connector read only required columns. `SELECT *` removes that advantage and pulls every declared table column, including ordinary partition columns that are part of the schema. Hidden metadata columns such as `"$path"` or `"$file_modified_time"` still require explicit selection.

### How to detect

- `SELECT *` appears on Hive, Iceberg, or Lakehouse tables.
- The plan `Layout` exposes many more columns than the final output needs.
- The query later discards most columns with another projection.

### Correct

```sql
SELECT orderkey, custkey, orderdate, totalprice
FROM iceberg.sales.orders
WHERE orderdate >= DATE '2026-04-01';
```

This keeps projection narrow and easier for the connector to optimize.

### Incorrect

```sql
SELECT *
FROM iceberg.sales.orders
WHERE orderdate >= DATE '2026-04-01';
```

The scan now has to consider every column, even if the consumer only needs four of them.

### Fix

- Enumerate required columns explicitly.
- Add hidden metadata columns such as `"$path"` only when diagnosing storage behavior, since `SELECT *` does not include them automatically.
- Treat `SELECT *` as debugging-only unless the full row is genuinely required.

### Reference

- https://trino.io/docs/current/optimizer/pushdown.html
- https://trino.io/docs/current/sql/select.html
