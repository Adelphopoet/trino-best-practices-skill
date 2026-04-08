---
title: Choose Join Distribution from Build-Side Size, Not Habit
impact: CRITICAL
impactDescription: "Avoids forcing broadcast joins that blow memory or partitioned joins that add unnecessary exchange cost."
tags:
  - trino
  - join
  - distribution
description: "Review `join_distribution_type` and the plan together; broadcast is for small filtered build sides, partitioned is for larger joins."
alwaysApply: true
---

## Choose Join Distribution from Build-Side Size, Not Habit

Trino hash joins can run as broadcast or partitioned joins. The right choice depends on the filtered build side, not on cargo-cult defaults.

### Why it matters

Broadcast joins replicate the build side to every worker. They are often faster when the build side is small. Partitioned joins redistribute both sides and are slower, but they scale to much larger joins because the build data is spread across the cluster.

### How to detect

- `EXPLAIN` shows `BROADCAST` or `REPARTITION` exchanges around the join.
- Someone wants to force `join_distribution_type` without estimating build-side size after filters.
- The build side clearly will not fit in memory on each node.

### Correct

```sql
SET SESSION join_distribution_type = 'AUTOMATIC';

SELECT count(*)
FROM hive.sales.store_sales s
JOIN hive.dim.date_dim d
  ON s.sold_date_sk = d.date_sk
WHERE d.d_year = 2026
  AND d.is_holiday = 'Y';
```

This lets Trino choose, which is usually right when statistics are usable and the build side is selective.

### Incorrect

```sql
SET SESSION join_distribution_type = 'BROADCAST';

SELECT *
FROM iceberg.fact.large_orders o
JOIN iceberg.fact.large_shipments s
  ON o.orderkey = s.orderkey;
```

Forcing broadcast on a large build side is a clean way to buy memory failures.

### Fix

- Prefer `AUTOMATIC` unless you have plan evidence to override it.
- If forcing `BROADCAST`, verify the filtered build side comfortably fits on each worker.
- If a broadcast cap is needed, use `join_max_broadcast_table_size` deliberately.

### Reference

- https://trino.io/docs/current/optimizer/cost-based-optimizations.html

