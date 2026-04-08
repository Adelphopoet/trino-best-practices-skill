---
title: Preserve Native Types in Filter Predicates
impact: CRITICAL
impactDescription: "Improves predicate pushdown chances and avoids unnecessary `ScanFilterProject` work in Trino."
tags:
  - trino
  - pushdown
  - predicates
description: "Use typed literals and aligned predicate types so Trino and the connector can reason about filters without extra coercion."
alwaysApply: true
---

## Preserve Native Types in Filter Predicates

Trino reviews should start by checking whether filter predicates keep the table column in its native type.

### Why it matters

Predicate pushdown is connector-specific, but Trino makes it visible in the plan: if pushdown succeeds, the plan no longer shows Trino-side filter work for that clause. Stringifying dates, timestamps, numerics, or UUID-like keys makes pushdown less reliable and often harder to cost correctly.

### How to detect

- A `DATE`, `TIMESTAMP`, numeric, or binary column is compared to a bare string literal.
- The plan shows `ScanFilterProject` with a filter that could have been pushed.
- The SQL casts the column instead of using a typed literal on the constant side.

### Correct

```sql
SELECT orderkey, totalprice
FROM iceberg.sales.orders
WHERE orderdate >= DATE '2026-04-01'
  AND custkey = 12345;
```

The predicate stays in the column's native type, which is the cleanest pushdown shape.

### Incorrect

```sql
SELECT orderkey, totalprice
FROM iceberg.sales.orders
WHERE cast(orderdate AS varchar) >= '2026-04-01'
  AND cast(custkey AS varchar) = '12345';
```

This pushes coercion into the scan path and makes connector pushdown less likely.

### Fix

- Use documented typed literals such as `DATE '...'`, `TIMESTAMP '...'`, `UUID '...'`, and `DECIMAL '...'` where they exist.
- For numeric parameters, prefer numeric literals or an explicit `CAST(... AS bigint)` / `CAST(... AS integer)` at the boundary.
- Preserve the column type in the predicate.
- If incoming parameters are text, cast them once outside the scan path.

### Reference

- https://trino.io/docs/current/optimizer/pushdown.html
- https://trino.io/docs/current/functions/conversion.html
