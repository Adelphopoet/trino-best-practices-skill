---
title: Use Iceberg Snapshots as First-Class Operational Evidence
impact: HIGH
impactDescription: "Enables safe time travel, rollback reasoning, and metadata-retention hygiene."
tags:
  - trino
  - iceberg
  - snapshots
description: "Agents SHOULD use snapshot metadata and procedures for time travel, rollback, and retention reviews instead of treating Iceberg like plain files."
alwaysApply: false
---

## Use Iceberg Snapshots as First-Class Operational Evidence

Iceberg tables are versioned objects. Reviews should act like it.

### Why it matters

Time travel, rollback, and metadata cleanup are built into the Iceberg model. Snapshot-aware troubleshooting is safer and more informative than ad hoc file-path debugging.

### How to detect

- The question is about a regression, recent data change, or rollback.
- The review ignores `$snapshots` or `$history`.
- Old snapshots are piling up and metadata is growing.

### Correct

```sql
SELECT snapshot_id, committed_at
FROM lake.sales."orders$snapshots"
ORDER BY committed_at DESC;
```

```sql
ALTER TABLE lake.sales.orders
EXECUTE expire_snapshots(retention_threshold => '7d');
```

This is the right operational vocabulary for Iceberg retention and history.

### Incorrect

```sql
-- Review comment:
-- "Delete the old files from object storage and rerun the query."
```

Direct file deletion ignores Iceberg's metadata model and can corrupt expectations.

### Fix

- Use snapshot metadata for history.
- Use time travel or rollback procedures when appropriate.
- Use `expire_snapshots` for retention hygiene instead of ad hoc storage cleanup.

### Reference

- https://trino.io/docs/current/connector/iceberg.html

