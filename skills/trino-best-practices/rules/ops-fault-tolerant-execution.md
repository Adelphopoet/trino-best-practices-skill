---
title: Review Fault-Tolerant Execution as a Cluster Capability
impact: HIGH
impactDescription: "Prevents retry guidance that ignores connector support, result-size limits, or exchange-manager requirements."
tags:
  - trino
  - operations
  - fault-tolerant-execution
description: "Fault-tolerant execution is off by default and MUST be reviewed as a cluster capability, not assumed as ambient behavior."
alwaysApply: false
---

## Review Fault-Tolerant Execution as a Cluster Capability

Retries in Trino are not automatic unless the cluster is configured for them.

### Why it matters

Fault-tolerant execution is disabled by default. `QUERY` and `TASK` policies have different trade-offs, `TASK` requires an exchange manager, and some connectors do not support retries. Large result sets also have caveats without external exchange storage.

### How to detect

- The problem statement assumes worker failures should retry automatically.
- The advice mentions `QUERY` or `TASK` but not cluster configuration.
- There is no evidence of exchange-manager configuration.

### Correct

```properties
retry-policy=TASK
```

This belongs in cluster configuration, together with exchange-manager setup, when the workload actually needs task retries.

### Incorrect

```text
"Trino retries failed tasks by default, so this query should recover automatically."
```

That statement is false on a default cluster.

### Fix

- Confirm cluster configuration first.
- Confirm connector support for retries.
- Treat FTE as an operational architecture choice, not a per-query magic switch.

### Reference

- https://trino.io/docs/current/admin/fault-tolerant-execution.html

