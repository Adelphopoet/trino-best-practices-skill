---
title: Verify Hive Dynamic Filtering and Dynamic Partition Pruning
impact: HIGH
impactDescription: "Can remove large amounts of probe-side Hive work when the join shape supports it."
tags:
  - trino
  - hive
  - dynamic-filtering
description: "For Hive-backed joins, agents SHOULD check whether dynamic filters reach the probe scan and whether the join keys align with partition pruning opportunities."
alwaysApply: false
---

## Verify Hive Dynamic Filtering and Dynamic Partition Pruning

Hive is one of the places where Trino dynamic filtering pays real rent.

### Why it matters

When dynamic filtering is active, Trino can prune probe-side splits and, on Hive partitioned tables, skip partitions that cannot match the join. This is especially valuable for selective dimension-to-fact joins.

### How to detect

- The query joins a large Hive fact table to a selective smaller table.
- `EXPLAIN` shows `dynamicFilterAssignments` on the join and a dynamic filter on the probe-side scan.
- The probe-side partition key is the same business key being filtered by the join.

### Correct

```sql
EXPLAIN
SELECT count(*)
FROM hive.sales.store_sales s
JOIN hive.dim.date_dim d
  ON s.sold_date_sk = d.date_sk
WHERE d.d_year = 2026
  AND d.is_holiday = 'Y';
```

This is the classic dynamic-filtering shape: selective build side, equality join, large probe side.

### Incorrect

```sql
EXPLAIN
SELECT count(*)
FROM hive.sales.store_sales s
JOIN hive.dim.date_dim d
  ON date(s.sold_at) = d.calendar_date
WHERE d.d_year = 2026;
```

The expression on the probe-side key weakens dynamic filter usefulness and pruning.

### Fix

- Keep the join as an equality on the raw key.
- Make the build side selective before the join.
- Confirm the feature in the plan; do not assume it from query shape alone.

### Reference

- https://trino.io/docs/current/admin/dynamic-filtering.html
- https://trino.io/docs/current/connector/hive.html

