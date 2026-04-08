---
title: Prefer Query-Shape Fixes Over Blind Session Tuning
impact: CRITICAL
impactDescription: "Keeps reviews focused on durable wins instead of brittle cluster-specific toggles."
tags:
  - trino
  - session
  - tuning
description: "Agents MUST read SQL and plan evidence before recommending session properties such as join distribution, retry settings, dynamic-filter wait time, or memory limits."
alwaysApply: true
---

## Prefer Query-Shape Fixes Over Blind Session Tuning

In Trino, query shape usually beats session tinkering.

### Why it matters

Session properties are cluster- and connector-sensitive. SQL rewrites that preserve pushdown, improve join shape, or reduce scan scope are usually more portable and more reliable than forcing a knob because one workload once improved.

### How to detect

- Recommendations start with `SET SESSION` before any plan evidence.
- The query contains obvious anti-patterns such as `SELECT *`, late filtering, or inline casts on join keys.
- The review does not separate SQL issues from connector or cluster issues.

### Correct

```sql
EXPLAIN (TYPE DISTRIBUTED)
SELECT orderkey, totalprice
FROM hive.sales.orders
WHERE orderdate >= DATE '2026-04-01';
```

First measure the shape. Then decide whether any session setting is still justified.

### Incorrect

```sql
SET SESSION join_distribution_type = 'PARTITIONED';
SET SESSION hive.dynamic_filtering_wait_timeout = '30s';

SELECT *
FROM hive.sales.orders;
```

This is tuning by superstition.

### Fix

- Inspect SQL and `EXPLAIN` first.
- Remove structural anti-patterns.
- Only recommend session changes when the remaining problem is clearly session-scoped.

### Reference

- https://trino.io/docs/current/sql/explain.html
- https://trino.io/docs/current/optimizer/cost-based-optimizations.html
- https://trino.io/docs/current/admin/dynamic-filtering.html

