---
title: Review Iceberg File Count and File Size Before Blaming Trino
impact: HIGH
impactDescription: "Prevents small-file pathologies from being misdiagnosed as pure engine tuning issues."
tags:
  - trino
  - iceberg
  - file-layout
description: "Slow Iceberg reads often come from file layout; agents SHOULD inspect `$files` and maintenance options before proposing memory or join tweaks."
alwaysApply: false
---

## Review Iceberg File Count and File Size Before Blaming Trino

If an Iceberg table is split into too many tiny files, Trino pays for it in planning, split scheduling, and scan overhead.

### Why it matters

Iceberg stores direct file metadata, which makes layout problems diagnosable. Small-file pathologies often require write-side batching or maintenance operations, not just query rewrites.

### How to detect

- `$files` shows a huge number of small files for the queried partitions.
- Planning time is high relative to data volume.
- The workload writes tiny batches into many partitions.

### Correct

```sql
SELECT avg(file_size_in_bytes) AS avg_file_size, count(*) AS files
FROM lake.sales."orders$files";
```

Inspect layout first, then decide whether the table needs maintenance.

### Incorrect

```sql
-- Review comment:
-- "The scan is slow, so increase query_max_memory and force broadcast."
```

That ignores a common physical-layout root cause.

### Fix

- Inspect `$files` first.
- If layout is fragmented, review write batching and Iceberg maintenance procedures such as `optimize`.
- Keep the diagnosis split between SQL shape and storage layout.

### Reference

- https://trino.io/docs/current/connector/iceberg.html
