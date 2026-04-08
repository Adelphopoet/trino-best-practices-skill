---
title: Use the Right EXPLAIN Mode for the Question
impact: HIGH
impactDescription: "Gets the right evidence without executing the wrong workload."
tags:
  - trino
  - explain
  - diagnostics
description: "Agents SHOULD choose between `EXPLAIN`, `EXPLAIN ANALYZE`, `EXPLAIN (TYPE IO)`, and `EXPLAIN (TYPE VALIDATE)` deliberately."
alwaysApply: true
---

## Use the Right EXPLAIN Mode for the Question

Trino gives different explain modes for different jobs. Use the right one or you get the wrong evidence.

### Why it matters

`EXPLAIN` shows planned shape, `EXPLAIN ANALYZE` executes the query and reports actual runtime statistics, `EXPLAIN (TYPE IO)` estimates read/write footprint, and `EXPLAIN (TYPE VALIDATE)` checks statement validity. Mixing them up either hides runtime facts or executes more than intended.

### How to detect

- A plan-shape review uses `EXPLAIN ANALYZE` even though execution is risky or expensive.
- A runtime review uses plain `EXPLAIN` and then claims row skew or CPU hotspots.
- A scan-footprint question ignores `EXPLAIN (TYPE IO)`.

### Correct

```sql
EXPLAIN (TYPE DISTRIBUTED)
SELECT custkey, count(*)
FROM hive.sales.orders
WHERE orderdate >= DATE '2026-04-01'
GROUP BY 1;
```

Use this for plan shape, exchanges, and scan predicates.

```sql
EXPLAIN ANALYZE
SELECT custkey, count(*)
FROM hive.sales.orders
WHERE orderdate >= DATE '2026-04-01'
GROUP BY 1;
```

Use this when you need actual runtime, input rows, blocked time, and skew evidence.

### Incorrect

```sql
EXPLAIN ANALYZE
INSERT INTO iceberg.sales.daily_summary
SELECT *
FROM iceberg.sales.orders;
```

This executes the statement. Do not use it casually on expensive or side-effecting workloads.

### Fix

- Use `EXPLAIN (TYPE DISTRIBUTED)` for structure.
- Use `EXPLAIN ANALYZE` for measured runtime behavior.
- Use `EXPLAIN (TYPE IO)` for footprint questions.
- Use `EXPLAIN (TYPE VALIDATE)` when the immediate question is syntax or resolution.

### Reference

- https://trino.io/docs/current/sql/explain.html
- https://trino.io/docs/current/sql/explain-analyze.html

