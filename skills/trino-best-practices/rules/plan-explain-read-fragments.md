---
title: Read Distributed EXPLAIN Fragments Before Tuning
impact: CRITICAL
impactDescription: "Prevents blind tuning by forcing evidence from fragment layout, scans, and exchanges."
tags:
  - trino
  - explain
  - plan
description: "Agents MUST read `EXPLAIN (TYPE DISTRIBUTED)` before suggesting join, memory, or session changes."
alwaysApply: true
---

## Read Distributed EXPLAIN Fragments Before Tuning

In Trino, fragment boundaries and exchange operators tell you where data is scanned, repartitioned, broadcast, or forced through a single node. That is the starting point for any credible review.

### Why it matters

Trino executes distributed plans as fragments with explicit exchange boundaries. A query that looks simple in SQL may still contain `RemoteExchange[REPARTITION]`, `BROADCAST`, or `SINGLE` bottlenecks that dominate runtime. Tuning without reading that shape is guesswork.

### How to detect

- No `EXPLAIN (TYPE DISTRIBUTED)` was reviewed before performance advice.
- The plan contains `Fragment ... [SINGLE]`, `RemoteExchange[REPARTITION]`, or `LocalExchange[HASH]`.
- Scan nodes show unexpected `ScanFilterProject` operators or full-width `Layout` output.

### Correct

```sql
EXPLAIN (TYPE DISTRIBUTED)
SELECT c.region, sum(o.totalprice)
FROM lake.sales.orders o
JOIN lake.sales.customers c ON c.custkey = o.custkey
WHERE o.orderdate >= DATE '2026-04-01'
GROUP BY 1;
```

Review fragment types, scan predicates, and exchange operators before deciding whether the query needs a SQL rewrite or a session change.

### Incorrect

```sql
SET SESSION join_distribution_type = 'BROADCAST';
SET SESSION query_max_memory = '50GB';

SELECT c.region, sum(o.totalprice)
FROM lake.sales.orders o
JOIN lake.sales.customers c ON c.custkey = o.custkey
GROUP BY 1;
```

Changing session properties before reading the distributed plan skips the only hard evidence Trino gives you.

### Fix

- Run `EXPLAIN (TYPE DISTRIBUTED)` first.
- Record fragment types, exchanges, scan operators, and join nodes.
- Only then decide whether the problem belongs in SQL shape, table design, or session configuration.

### Reference

- https://trino.io/docs/current/sql/explain.html

