---
title: Filter Hive Partition Keys Directly
impact: CRITICAL
impactDescription: "Turns expensive partition enumeration into targeted partition reads."
tags:
  - trino
  - hive
  - partition-pruning
description: "Hive queries SHOULD filter declared partition keys directly and with native types whenever the workload allows it."
alwaysApply: false
---

## Filter Hive Partition Keys Directly

On Hive tables, the cheapest data is the data whose partitions were never scheduled.

### Why it matters

Hive partition pruning happens when Trino can infer that only specific partitions are relevant. If the query filters only derived expressions or non-partition columns, Trino may still enumerate and inspect far too much data.

### How to detect

- `SHOW CREATE TABLE` shows `partitioned_by`, but the query does not filter those columns.
- The query filters a timestamp column while the table is actually partitioned by a derived `ds` or `hour` column.
- Metadata diagnostics using `"$partition"` reveal many partitions touched.

### Correct

```sql
SELECT user_id, page_url
FROM hive.web.page_views
WHERE ds = DATE '2026-04-08'
  AND country = 'US';
```

This filters the declared partition columns directly.

### Incorrect

```sql
SELECT user_id, page_url
FROM hive.web.page_views
WHERE date(view_time) = DATE '2026-04-08'
  AND country = 'US';
```

If `ds` is the partition key, this shape makes pruning weaker than it should be.

### Fix

- Filter the actual partition columns from the table definition.
- Keep those predicates typed.
- Use `"$partition"` only for diagnosis, not as a substitute for proper table design.

### Reference

- https://trino.io/docs/current/connector/hive.html

