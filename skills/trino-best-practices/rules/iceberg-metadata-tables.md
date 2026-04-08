---
title: Use Iceberg Metadata Tables for Diagnosis
impact: HIGH
impactDescription: "Replaces guesses about files, partitions, and history with direct Iceberg evidence."
tags:
  - trino
  - iceberg
  - metadata
description: "Agents SHOULD query Iceberg metadata tables such as `$partitions`, `$files`, `$snapshots`, and `$history` before speculating about layout or retention."
alwaysApply: false
---

## Use Iceberg Metadata Tables for Diagnosis

Iceberg gives you metadata tables. Use them before guessing.

### Why it matters

Iceberg stores rich metadata that Trino exposes directly. This makes it possible to diagnose partition spread, file counts, snapshot churn, and physical paths without scraping storage by hand.

### How to detect

- A review guesses about file counts, partition spread, or snapshot history.
- The issue is about layout, retention, or recent table changes.
- No metadata-table query was run even though the table is Iceberg.

### Correct

```sql
SELECT partition, record_count, file_count
FROM lake.sales."orders$partitions"
ORDER BY record_count DESC;
```

```sql
SELECT file_path, file_size_in_bytes, record_count
FROM lake.sales."orders$files";
```

These queries give direct evidence about physical layout.

### Incorrect

```sql
-- Review comment:
-- "This table probably has too many files. Try a bigger memory limit."
```

That is speculation. Ask the metadata tables first.

### Fix

- Inspect `$partitions`, `$files`, `$snapshots`, and `$history` when diagnosing Iceberg.
- Use metadata evidence to decide whether the fix belongs in SQL, maintenance, or write layout.

### Reference

- https://trino.io/docs/current/connector/iceberg.html

