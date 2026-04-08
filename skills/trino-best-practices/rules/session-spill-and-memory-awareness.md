---
title: Treat Spill and Memory Knobs as Secondary to Query Shape
impact: HIGH
impactDescription: "Avoids turning a shape problem into a slower, more fragile spilled query."
tags:
  - trino
  - session
  - memory
description: "Memory and spill settings matter, but they SHOULD follow evidence about join shape, aggregation cardinality, and layout problems."
alwaysApply: true
---

## Treat Spill and Memory Knobs as Secondary to Query Shape

If a query is huge for structural reasons, more memory or spill usually just lets it fail later and slower.

### Why it matters

Trino documents spill-to-disk as legacy functionality and warns that spilled queries can become slower by orders of magnitude. Memory limits still matter, but they are not a substitute for reducing build-side size, grouping cardinality, or scan volume.

### How to detect

- The first recommendation is "raise `query_max_memory`" or "enable spill".
- The plan already shows a massive broadcast build side, huge grouping cardinality, or row explosion.
- The query is slow because of small-file layout or unpruned scans, not just memory.

### Correct

```sql
SELECT custkey, sum(totalprice)
FROM iceberg.sales.orders
WHERE orderdate >= DATE '2026-04-01'
GROUP BY custkey;
```

Start by reducing scan scope and grouping shape. Tune memory only after that review.

### Incorrect

```sql
SET SESSION query_max_memory = '80GB';
SET SESSION spill_enabled = true;

SELECT *
FROM iceberg.fact.large_orders o
JOIN iceberg.fact.large_shipments s ON o.orderkey = s.orderkey;
```

This tries to paper over a likely join-shape problem.

### Fix

- Rewrite the query first.
- Review join distribution, filters, and aggregation grain.
- If memory tuning is still needed, document why the remaining workload is structurally valid.

### Reference

- https://trino.io/docs/current/admin/spill.html
- https://trino.io/docs/current/admin/fault-tolerant-execution.html
