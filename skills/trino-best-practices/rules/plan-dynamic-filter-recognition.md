---
title: Recognize Dynamic Filtering in the Plan
impact: HIGH
impactDescription: "Distinguishes real runtime pruning from wishful thinking."
tags:
  - trino
  - dynamic-filtering
  - plan
description: "Agents MUST verify dynamic filtering from plan evidence such as `dynamicFilterAssignments` and probe-side dynamic filter predicates."
alwaysApply: true
---

## Recognize Dynamic Filtering in the Plan

Dynamic filtering in Trino is visible in the plan. If the plan does not show it, do not claim it is helping.

### Why it matters

Dynamic filtering improves selective joins by collecting values from the build side and pushing them into the probe-side scan. On supported connectors such as Hive, this can also enable dynamic partition pruning. The presence or absence of this feature changes what kind of rewrite is worth doing.

### How to detect

- `InnerJoin` shows `dynamicFilterAssignments`.
- Probe-side `ScanFilterProject` or `TableScan` shows a `dynamicFilter` predicate.
- If both are absent, dynamic filtering is not active for that join, or the connector/join shape does not support it.

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

This is the right diagnostic shape: a selective build side joined on an equality key that can produce dynamic filters.

### Incorrect

```sql
EXPLAIN
SELECT count(*)
FROM hive.sales.store_sales s
JOIN hive.dim.date_dim d
  ON date(s.sold_at) = d.calendar_date
WHERE d.d_year = 2026;
```

Expressions on join keys make dynamic filtering less likely to engage and harder for the connector to use.

### Fix

- Keep the build side selective.
- Use equality joins on raw join keys when possible.
- Verify the plan instead of assuming the feature is active.

### Reference

- https://trino.io/docs/current/admin/dynamic-filtering.html

