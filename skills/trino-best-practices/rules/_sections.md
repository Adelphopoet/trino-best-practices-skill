# Rule Sections

This file defines the canonical section order, filename prefixes, and review intent for contributors extending `trino-best-practices`.

## 1. Plan Reading

- Prefix: `plan-`
- Purpose: Teach agents how to read Trino fragments, exchanges, scans, and dynamic filters before proposing fixes.

## 2. Query Shape

- Prefix: `query-`
- Purpose: Preserve pushdown, cut scan volume, and simplify query shape before touching session knobs.

## 3. Joins

- Prefix: `join-`
- Purpose: Review join distribution, build/probe sizing, filtering order, membership tests, and row-explosion risk.

## 4. Types and Predicate Semantics

- Prefix: `types-`
- Purpose: Preserve native types and avoid coercions that break pushdown, pruning, or join efficiency.

## 5. Explain Usage

- Prefix: `explain-`
- Purpose: Pick the right Trino explain mode for the question being answered.

## 6. Hive Connector

- Prefix: `hive-`
- Purpose: Review partition pruning, dynamic filtering, file-format implications, and statistics in Hive-backed reads.

## 7. Iceberg Connector

- Prefix: `iceberg-`
- Purpose: Review partition transforms, metadata tables, snapshots, and file-layout health for Iceberg tables queried by Trino.

## 8. Lakehouse Connector

- Prefix: `lakehouse-`
- Purpose: Prevent agents from inventing semantics that differ from the underlying table format connector.

## 9. Session and Execution

- Prefix: `session-`
- Purpose: Keep session-property guidance evidence-based and subordinate to query-shape fixes.

## 10. Operations and Reliability

- Prefix: `ops-`
- Purpose: Cover fault-tolerant execution, retry support, and connector-specific operational caveats.

## 11. Anti-patterns

- Prefix: `anti-pattern-`
- Purpose: Encode fast rejection tests for common Trino mistakes that repeatedly block pushdown, pruning, or join efficiency.
