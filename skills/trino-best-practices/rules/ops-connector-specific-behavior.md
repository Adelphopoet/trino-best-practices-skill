---
title: Separate Engine Behavior from Connector Behavior
impact: CRITICAL
impactDescription: "Stops reviews from attributing connector limitations or features to Trino core."
tags:
  - trino
  - operations
  - connectors
description: "Agents MUST label whether a recommendation is about Trino engine behavior, connector behavior, file format, or table design."
alwaysApply: true
---

## Separate Engine Behavior from Connector Behavior

Trino is the engine. Pushdown, metadata exposure, retry support, and table-format semantics often come from the connector.

### Why it matters

Bad reviews often say "Trino does not support X" when the real answer is "this connector does not push X down" or "this table format does not expose that pruning signal." The fix path changes completely depending on that boundary.

### How to detect

- The answer uses "Trino" as a blanket label for Hive-, Iceberg-, or Lakehouse-specific behavior.
- The catalog and connector are not identified.
- The advice mixes engine, connector, and storage-format concerns in one sentence.

### Correct

```text
The join shape is a Trino engine concern.
The missing predicate pushdown is a connector concern.
The small-file issue is a table-layout concern.
```

That is the right separation.

### Incorrect

```text
Trino cannot prune this query.
```

That sentence is incomplete to the point of being useless.

### Fix

- Name the catalog and connector.
- State whether the issue belongs to engine planning, connector pushdown, file format, table design, or session policy.
- If the connector is unknown, say so explicitly.

### Reference

- https://trino.io/docs/current/connector.html
- https://trino.io/docs/current/optimizer/pushdown.html
- https://trino.io/docs/current/connector/lakehouse.html

