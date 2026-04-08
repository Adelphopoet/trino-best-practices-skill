---
title: Do Not Suggest retry_policy Blindly
impact: HIGH
impactDescription: "Avoids session advice that conflicts with cluster mode, exchange-manager requirements, or connector support."
tags:
  - trino
  - session
  - retry-policy
description: "The `retry_policy` session property only makes sense on clusters configured for fault-tolerant execution, and changing modes casually is not the normal fix path."
alwaysApply: false
---

## Do Not Suggest retry_policy Blindly

`retry_policy` is not a generic performance knob. It is part of fault-tolerant execution strategy.

### Why it matters

Trino documents `retry-policy` / `retry_policy` as fault-tolerant execution controls. `TASK` requires an exchange manager, and switching modes on a live cluster via session is not the normal tuning path. Some connectors do not support retries at all.

### How to detect

- Advice says "set `retry_policy=TASK`" without mentioning cluster configuration.
- The cluster has no exchange manager.
- The issue is query slowness, not retryable failures.

### Correct

```sql
SET SESSION retry_policy = 'NONE';
```

Use session-level overrides sparingly, typically to disable retries on a cluster already configured for fault-tolerant execution when troubleshooting.

### Incorrect

```sql
SET SESSION retry_policy = 'TASK';
SELECT * FROM hive.sales.orders;
```

This assumes the cluster is built for task retries and the connector supports them.

### Fix

- First confirm whether the cluster is configured for fault-tolerant execution.
- Confirm exchange-manager availability before suggesting `TASK`.
- Keep retry policy discussions separate from pure SQL-shape tuning.

### Reference

- https://trino.io/docs/current/admin/fault-tolerant-execution.html

