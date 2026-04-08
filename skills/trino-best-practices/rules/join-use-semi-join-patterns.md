---
title: Use EXISTS or IN for Membership Tests
impact: HIGH
impactDescription: "Lets Trino plan a membership test instead of materializing a full join plus dedup."
tags:
  - trino
  - semi-join
  - joins
description: "If the query only needs columns from the left side, agents SHOULD prefer `EXISTS` or `IN` over `JOIN ... DISTINCT`."
alwaysApply: false
---

## Use EXISTS or IN for Membership Tests

If the goal is "keep left rows that have a match", write that directly.

### Why it matters

An inner join followed by `DISTINCT` often means the query really wanted a membership test. In Trino that shape can be planned more cleanly as a semi-join.

### How to detect

- The final output uses only left-table columns.
- The query uses `JOIN` and then `DISTINCT` to remove duplicate left rows.
- The right side only contributes existence, not payload columns.

### Correct

```sql
SELECT c.custkey
FROM iceberg.sales.customers c
WHERE EXISTS (
    SELECT 1
    FROM iceberg.sales.orders o
    WHERE o.custkey = c.custkey
      AND o.orderdate >= DATE '2026-04-01'
);
```

This expresses membership directly.

### Incorrect

```sql
SELECT DISTINCT c.custkey
FROM iceberg.sales.customers c
JOIN iceberg.sales.orders o
  ON o.custkey = c.custkey
WHERE o.orderdate >= DATE '2026-04-01';
```

The join creates duplicate customer rows only to remove them later.

### Fix

- Rewrite pure membership joins as `EXISTS` or `IN`.
- Keep full joins for cases where right-side columns are actually needed.

### Reference

- https://trino.io/docs/current/sql/select.html
- https://trino.io/docs/current/sql/explain.html

