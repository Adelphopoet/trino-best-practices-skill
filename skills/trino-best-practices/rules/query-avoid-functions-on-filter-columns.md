---
title: Avoid Ad Hoc Functions on Filter Columns
impact: CRITICAL
impactDescription: "Prevents accidental loss of pushdown or pruning on scan-time predicates."
tags:
  - trino
  - pushdown
  - pruning
description: "Wrapping filter columns in functions SHOULD be treated as suspicious unless the exact partition transform makes it intentional."
alwaysApply: true
---

## Avoid Ad Hoc Functions on Filter Columns

In Trino, filters are easiest to optimize when the scan column is used directly. As soon as you wrap the scan column in `date()`, `date_trunc()`, `substr()`, or similar logic, you risk losing pushdown and pruning.

### Why it matters

Trino and connectors reason best about predicates on raw columns or explicit partition keys. Function-wrapped columns often leave Trino doing scan-time filtering instead of the connector pruning work. The main exception is when the function matches a known partition transform or materialized partition key by design.

### How to detect

- `WHERE date(ts_col) = ...`, `date_trunc(...)`, `lower(partition_col)`, `substr(key, ...)`.
- `EXPLAIN` still shows a scan-side filter.
- Hive or Iceberg partition pruning is unexpectedly weak.

### Correct

```sql
SELECT *
FROM hive.web.events
WHERE event_ts >= TIMESTAMP '2026-04-08 00:00:00'
  AND event_ts < TIMESTAMP '2026-04-09 00:00:00';
```

Range predicates on the raw timestamp are the safest default shape.

### Incorrect

```sql
SELECT *
FROM hive.web.events
WHERE date(event_ts) = DATE '2026-04-08';
```

This forces Trino to reason through a function on the scan column and can block connector pruning.

### Fix

- Rewrite day filters as half-open ranges on the raw column.
- If the table is intentionally partitioned by a transform, filter the transform-aware key or confirm the transform is exploitable.
- Treat scan-column functions as a deliberate exception, not a default pattern.

### Exceptions / trade-offs

If the table design explicitly exposes a partition key or transform-aligned expression that Trino can prune on, using that exact key can be valid. Verify it with `SHOW CREATE TABLE` and `EXPLAIN`.

### Reference

- https://trino.io/docs/current/optimizer/pushdown.html
- https://trino.io/docs/current/connector/hive.html
- https://trino.io/docs/current/connector/iceberg.html

