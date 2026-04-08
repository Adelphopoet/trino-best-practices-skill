---
title: Reason About Iceberg Partition Transforms Explicitly
impact: CRITICAL
impactDescription: "Prevents wrong advice about pruning, filter shape, and table design on transformed partitions."
tags:
  - trino
  - iceberg
  - partitioning
description: "Iceberg partitioning may use transforms such as `month(ts)` or `bucket(id, n)`; agents MUST review those transforms before recommending filters or redesigns."
alwaysApply: false
---

## Reason About Iceberg Partition Transforms Explicitly

Iceberg partitions are not always raw columns. Trino reviews that ignore transform definitions miss the point.

### Why it matters

Iceberg supports transform-based partitioning. A table partitioned by `month(order_ts)` is not the same thing as a table partitioned by `order_ts` or by a materialized `ds` column. Query and design advice has to match the actual transform.

### How to detect

- `SHOW CREATE TABLE` or table properties show `partitioning = ARRAY[...]`.
- The partition spec includes transforms such as `month(...)`, `day(...)`, `bucket(...)`, or `truncate(...)`.
- The review talks about partition pruning without naming the transform.

### Correct

```sql
CREATE TABLE lake.sales.orders (
    order_id bigint,
    order_ts timestamp(6),
    amount double
)
WITH (
    format = 'PARQUET',
    partitioning = ARRAY['month(order_ts)']
);
```

Any pruning discussion for this table must start from the `month(order_ts)` transform.

### Incorrect

```sql
-- Review comment:
-- "The table is partitioned by order_ts, so a day filter will always prune perfectly."
```

That statement is false if the actual transform is monthly or bucketed.

### Fix

- Inspect the Iceberg partition spec first.
- Align query advice with the real transform.
- If the transform is wrong for the workload, treat it as a table-design issue, not a session-tuning issue.

### Reference

- https://trino.io/docs/current/connector/iceberg.html

