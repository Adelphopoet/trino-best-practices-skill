---
title: Do Not Cast Join Keys Inside the Join Predicate
impact: CRITICAL
impactDescription: "Avoids expensive join-time coercion and preserves cleaner build/probe planning."
tags:
  - trino
  - anti-pattern
  - joins
description: "Inline casts in `JOIN ... ON` are a review blocker unless there is a proven, unavoidable interoperability boundary."
alwaysApply: true
---

## Do Not Cast Join Keys Inside the Join Predicate

Casting inside the join predicate is one of the fastest ways to make a Trino join worse.

### Why it matters

The join should consume aligned keys, not repair them on the fly. Inline casts on join keys complicate planning, can reduce pushdown opportunities, and often hide an upstream schema mismatch that should be fixed once instead of per row.

### How to detect

- `ON cast(left_key AS ...) = right_key`
- `ON left_key = cast(right_key AS ...)`
- Mismatched key types across tables with no normalization step before the join

### Correct

```sql
WITH left_norm AS (
    SELECT cast(order_id_txt AS bigint) AS order_id, payload
    FROM hive.stage.orders_raw
)
SELECT l.order_id, r.status
FROM left_norm l
JOIN iceberg.sales.order_status r
  ON r.order_id = l.order_id;
```

The type repair happens once, before the join.

### Incorrect

```sql
SELECT l.order_id_txt, r.status
FROM hive.stage.orders_raw l
JOIN iceberg.sales.order_status r
  ON cast(l.order_id_txt AS bigint) = r.order_id;
```

This leaves the join itself responsible for cleaning up schema drift.

### Fix

- Normalize key types in staging or a pre-projection.
- If you must cast, cast the smaller input once before the join and document the downside.

### Reference

- https://trino.io/docs/current/functions/conversion.html
- https://trino.io/docs/current/optimizer/pushdown.html

