---
title: Rule Title
impact: CRITICAL
impactDescription: "One-sentence outcome of following the rule"
tags:
  - trino
  - category
  - connector-or-topic
description: "One narrow engineering decision or anti-pattern in Trino terms."
alwaysApply: false
---

## Rule Title

Short Trino-specific explanation of the rule.

### Why it matters

Explain the exact Trino engine, optimizer, connector, or file-format behavior that makes this rule important.

### How to detect

- Concrete SQL signal
- Concrete `EXPLAIN` / `EXPLAIN ANALYZE` signal
- Concrete catalog, table-property, or session-property signal

### Correct

```sql
-- Good example
SELECT 1;
```

One short note about why the example is correct in Trino.

### Incorrect

```sql
-- Bad example
SELECT 1;
```

One short note about why the example is risky or wrong in Trino.

### Fix

- Specific refactoring step
- State whether the fix belongs in SQL, table design, file layout, catalog config, or session config

### Exceptions / trade-offs

Only include this section when the rule has a real, non-trivial exception.

### Reference

- Official Trino documentation URL only
