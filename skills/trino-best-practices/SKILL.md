---
name: trino-best-practices
description: MUST USE when reviewing Trino SQL, EXPLAIN plans, optimizer behavior, or performance-related session settings so recommendations stay connector-aware and Trino-specific, especially for Hive and Iceberg workloads.
license: Apache-2.0
metadata:
  author: Codex
  version: "1.0.0"
---

# Trino Best Practices

This skill teaches an agent how to review Trino SQL, read Trino execution plans, reason about connector behavior, and troubleshoot slow queries without collapsing into generic SQL advice.

## What the skill does

- Reviews SQL shape, predicate structure, joins, and aggregation strategy.
- Reviews `EXPLAIN`, `EXPLAIN ANALYZE`, and `EXPLAIN (TYPE IO)` output.
- Reviews session and catalog choices only after plan evidence exists.
- Distinguishes engine behavior from connector behavior.
- Covers Hive and Iceberg specifics, plus Lakehouse connector boundary rules.

## IMPORTANT: How to apply this skill

Before giving guidance, the agent MUST:

1. Inspect the query, DDL, plan, or config first. Do not optimize blind.
2. Identify the catalog, connector, table format, and session properties in play.
3. Read the relevant rule files from `rules/` instead of relying on generic SQL intuition.
4. Cite rule names in the answer, for example: `Per join-choose-distribution...`.
5. Prefer Trino- and connector-specific guidance over generic database advice.
6. State uncertainty explicitly when Trino version, connector behavior, or table design is unclear.
7. Separate recommendations into the correct layer: SQL shape, table design, file layout, catalog config, or session config.
8. If a Lakehouse table is `DELTA` or `HUDI`, do not pretend this package has full connector-specific coverage; apply the boundary rule, keep advice limited to generic SQL/plan/session evidence, and call out the missing connector-specific context.

## Review Procedures

### For SQL / query reviews

Read these first:

1. `rules/explain-use-explain-analyze-appropriately.md`
2. `rules/plan-explain-read-fragments.md`
3. `rules/query-filter-pushdown-preserve-types.md`
4. `rules/query-avoid-functions-on-filter-columns.md`
5. `rules/query-avoid-select-star.md`
6. `rules/query-limit-scan-early.md`
7. `rules/query-aggregate-before-join-when-valid.md`
8. `rules/join-choose-distribution.md`
9. `rules/join-filter-before-join.md`
10. `rules/join-watch-row-explosion.md`
11. `rules/join-use-semi-join-patterns.md`
12. `rules/types-avoid-implicit-mismatch.md`
13. `rules/types-varbinary-varchar-awareness.md`
14. `rules/anti-pattern-cast-on-join-key.md`
15. `rules/anti-pattern-date-wrap-breaks-pruning.md`
16. `rules/cte-dont-assume-materialization.md`

Check for:

- Native typed predicates instead of stringly comparisons
- Filters at scan level, not after joins
- Join build/probe sanity
- Early aggregation when semantics allow
- `SELECT *` on wide lakehouse tables
- Inline casts or functions on join/filter keys

### For EXPLAIN / plan reviews

Read these first:

1. `rules/explain-use-explain-analyze-appropriately.md`
2. `rules/plan-explain-read-fragments.md`
3. `rules/plan-dynamic-filter-recognition.md`
4. `rules/join-choose-distribution.md`
5. `rules/anti-pattern-late-filtering.md`
6. `rules/session-query-shape-over-tuning.md`

Check for:

- `Fragment` boundaries and exchange types
- `TableScan` / `ScanFilterProject` predicates
- `dynamicFilterAssignments` and probe-side dynamic filters
- Broadcast vs repartitioned joins
- Late filters, row explosion, or unexpected `SINGLE` bottlenecks

### For Hive / Iceberg / Lakehouse reviews

Read connector rules before proposing fixes:

- Hive: `rules/hive-dynamic-filtering.md`, `rules/hive-partition-pruning.md`, `rules/hive-file-format-pruning.md`
- Iceberg: `rules/iceberg-partitioning-transform-awareness.md`, `rules/iceberg-metadata-tables.md`, `rules/iceberg-snapshot-operations.md`, `rules/iceberg-file-size-and-layout.md`
- Lakehouse wrapper: `rules/lakehouse-connector-boundaries.md`

Routing:

- If the Lakehouse table type is `HIVE`, apply the Hive rules.
- If the Lakehouse table type is `ICEBERG`, apply the Iceberg rules.
- If the Lakehouse table type is `DELTA` or `HUDI`, apply only `rules/lakehouse-connector-boundaries.md` plus generic SQL / plan / session rules from this package, and state that deeper connector-specific guidance is out of scope for this skill.

Check for:

- Direct predicates on partition keys or transform-aware predicates
- Metadata-table evidence instead of guesses
- File-count / file-size pathologies
- Table type confusion inside the Lakehouse connector
- Unsupported connector-specific claims for Lakehouse `DELTA` / `HUDI` tables

### For execution / configuration reviews

Read these first:

1. `rules/session-query-shape-over-tuning.md`
2. `rules/session-retry-policy-caveats.md`
3. `rules/session-spill-and-memory-awareness.md`
4. `rules/ops-fault-tolerant-execution.md`
5. `rules/ops-connector-specific-behavior.md`

Check for:

- Whether SQL shape can be fixed before toggling session knobs
- Whether `retry_policy` advice matches cluster mode and connector support
- Whether spill is being used as a band-aid
- Whether the connector actually supports the recommended feature

## Output format for formal reviews

Use this structure:

```md
## Scope
- Engine / connector / table format / evidence reviewed

## Rules Checked
- `rule-name` - Compliant / Violation / Not enough evidence

## Findings
- Severity | `rule-name` | Evidence | Why it matters | Fix

## Open Questions
- Missing connector, version, table property, or session detail

## Recommended Next Step
- SQL rewrite / table design / file layout / catalog config / session config
```

## Rule categories by priority

| Priority | Category | Prefix | Typical outcome |
|---|---|---|---|
| 1 | Plan reading | `plan-` | Evidence before tuning |
| 2 | Query shape | `query-` | Less scan, less exchange |
| 3 | Joins | `join-` | Lower memory and row count |
| 4 | Types and predicates | `types-` | Better pushdown and cleaner joins |
| 5 | Explain usage | `explain-` | Correct diagnostic tool |
| 6 | Hive connector | `hive-` | Better pruning and dynamic filters |
| 7 | Iceberg connector | `iceberg-` | Transform-aware, metadata-first reviews |
| 8 | Lakehouse connector | `lakehouse-` | No fake unified semantics |
| 9 | Session and execution | `session-` | Safer, evidence-based tuning |
| 10 | Operations and reliability | `ops-` | Correct resiliency expectations |
| 11 | Anti-patterns | `anti-pattern-` | Fast rejection of common bad rewrites |

## Quick reference

- Plans first: `plan-explain-read-fragments`, `explain-use-explain-analyze-appropriately`
- Dynamic filtering: `plan-dynamic-filter-recognition`, `hive-dynamic-filtering`
- Join sanity: `join-choose-distribution`, `join-filter-before-join`, `join-watch-row-explosion`, `join-use-semi-join-patterns`
- Pushdown and pruning: `query-filter-pushdown-preserve-types`, `query-avoid-functions-on-filter-columns`, `anti-pattern-date-wrap-breaks-pruning`
- Types: `types-avoid-implicit-mismatch`, `types-varbinary-varchar-awareness`, `anti-pattern-cast-on-join-key`
- Hive / Iceberg / Lakehouse: `hive-*`, `iceberg-*`, `lakehouse-connector-boundaries`
- Reliability and tuning: `session-*`, `ops-*`

## Trigger phrases

Use this skill when the prompt contains things like:

- "Optimize this Trino query"
- "Review this EXPLAIN plan"
- "Why is predicate pushdown not happening?"
- "Why is partition pruning not happening?"
- "Is dynamic filtering helping here?"
- "Should this join be broadcast or partitioned?"
- "Review this Hive / Iceberg query"
- "Review this Lakehouse query and identify whether it is Hive or Iceberg"
- "Why is this Trino query slow?"
- "Should I change retry policy / spill / memory settings for this slow query?"

## Full compiled document

For the long-form compiled reference with every rule inline, read `AGENTS.md`.
