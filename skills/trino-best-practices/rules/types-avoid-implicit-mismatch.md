---
title: Avoid Implicit Type Mismatches on Join and Filter Keys
impact: CRITICAL
impactDescription: "Prevents hidden coercions that hurt pushdown, cost estimation, and join efficiency."
tags:
  - trino
  - types
  - predicates
description: "Join and filter keys SHOULD meet in the same native type before they reach the main query body."
alwaysApply: true
---

## Avoid Implicit Type Mismatches on Join and Filter Keys

When key types disagree, Trino has to coerce something. That is usually a smell.

### Why it matters

Implicit coercions complicate planning, can block connector pushdown, and make joins more expensive or brittle. The safest shape is a stable key type on both sides before the join or filter executes.

### How to detect

- One side is `varchar`, the other `bigint`, `date`, `timestamp`, or `varbinary`.
- `SHOW CREATE TABLE` exposes mismatched key types across sources.
- The query casts keys inline or relies on bare string parameters.

### Correct

```sql
WITH normalized_orders AS (
    SELECT cast(order_id_txt AS bigint) AS order_id, totalprice
    FROM hive.stage.orders_raw
)
SELECT o.order_id, f.status
FROM normalized_orders o
JOIN iceberg.sales.fulfillment f
  ON f.order_id = o.order_id;
```

The type mismatch is handled once, before the join.

### Incorrect

```sql
SELECT o.order_id_txt, f.status
FROM hive.stage.orders_raw o
JOIN iceberg.sales.fulfillment f
  ON cast(o.order_id_txt AS bigint) = f.order_id;
```

This leaves type repair inside the join path itself.

### Fix

- Normalize key types upstream or in a pre-projection.
- Compare typed values to typed values.
- Do not standardize everything to `varchar` unless the domain is genuinely textual.

### Reference

- https://trino.io/docs/current/functions/conversion.html
- https://trino.io/docs/current/optimizer/pushdown.html

