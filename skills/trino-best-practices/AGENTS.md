# Trino Best Practices

**Version 1.0.0**  
Codex  
April 2026  
Documentation baseline: Trino current docs (480)

---

## Abstract

This reference manual teaches agents how to review Trino SQL, plans, connector choices, and execution settings without falling back to generic warehouse folklore. The emphasis is review-first: inspect SQL, `EXPLAIN`, connector semantics, table layout, and session settings before recommending changes. Connector behavior matters. File format matters. Query shape matters more than blind tuning.

---

## Table of Contents

1. [Plan Reading](#1-plan-reading)
   - 1.1 [Read Distributed EXPLAIN Fragments Before Tuning](#11-read-distributed-explain-fragments-before-tuning)
   - 1.2 [Recognize Dynamic Filtering in the Plan](#12-recognize-dynamic-filtering-in-the-plan)
2. [Query Shape](#2-query-shape)
   - 2.1 [Preserve Native Types in Filter Predicates](#21-preserve-native-types-in-filter-predicates)
   - 2.2 [Avoid Ad Hoc Functions on Filter Columns](#22-avoid-ad-hoc-functions-on-filter-columns)
   - 2.3 [Avoid SELECT Star on Trino Lakehouse Tables](#23-avoid-select-star-on-trino-lakehouse-tables)
   - 2.4 [Reduce Scan Volume Early Instead of Hoping LIMIT Saves You](#24-reduce-scan-volume-early-instead-of-hoping-limit-saves-you)
   - 2.5 [Aggregate Before the Join When Semantics Allow](#25-aggregate-before-the-join-when-semantics-allow)
3. [Joins](#3-joins)
   - 3.1 [Choose Join Distribution from Build-Side Size, Not Habit](#31-choose-join-distribution-from-build-side-size-not-habit)
   - 3.2 [Filter Join Inputs Before Joining](#32-filter-join-inputs-before-joining)
   - 3.3 [Watch for Join Row Explosion](#33-watch-for-join-row-explosion)
   - 3.4 [Use EXISTS or IN for Membership Tests](#34-use-exists-or-in-for-membership-tests)
4. [Types and Predicate Semantics](#4-types-and-predicate-semantics)
   - 4.1 [Avoid Implicit Type Mismatches on Join and Filter Keys](#41-avoid-implicit-type-mismatches-on-join-and-filter-keys)
   - 4.2 [Treat VARBINARY and VARCHAR as Different Domains](#42-treat-varbinary-and-varchar-as-different-domains)
   - 4.3 [Do Not Assume CTE Materialization](#43-do-not-assume-cte-materialization)
5. [Explain Usage](#5-explain-usage)
   - 5.1 [Use the Right EXPLAIN Mode for the Question](#51-use-the-right-explain-mode-for-the-question)
6. [Hive Connector](#6-hive-connector)
   - 6.1 [Verify Hive Dynamic Filtering and Dynamic Partition Pruning](#61-verify-hive-dynamic-filtering-and-dynamic-partition-pruning)
   - 6.2 [Filter Hive Partition Keys Directly](#62-filter-hive-partition-keys-directly)
   - 6.3 [Respect Hive File-Format Pruning Limits](#63-respect-hive-file-format-pruning-limits)
7. [Iceberg Connector](#7-iceberg-connector)
   - 7.1 [Reason About Iceberg Partition Transforms Explicitly](#71-reason-about-iceberg-partition-transforms-explicitly)
   - 7.2 [Use Iceberg Metadata Tables for Diagnosis](#72-use-iceberg-metadata-tables-for-diagnosis)
   - 7.3 [Use Iceberg Snapshots as First-Class Operational Evidence](#73-use-iceberg-snapshots-as-first-class-operational-evidence)
   - 7.4 [Review Iceberg File Count and File Size Before Blaming Trino](#74-review-iceberg-file-count-and-file-size-before-blaming-trino)
8. [Lakehouse Connector](#8-lakehouse-connector)
   - 8.1 [Treat Lakehouse as a Wrapper, Not a New Semantics Layer](#81-treat-lakehouse-as-a-wrapper-not-a-new-semantics-layer)
9. [Session and Execution](#9-session-and-execution)
   - 9.1 [Do Not Suggest retry_policy Blindly](#91-do-not-suggest-retry_policy-blindly)
   - 9.2 [Treat Spill and Memory Knobs as Secondary to Query Shape](#92-treat-spill-and-memory-knobs-as-secondary-to-query-shape)
   - 9.3 [Prefer Query-Shape Fixes Over Blind Session Tuning](#93-prefer-query-shape-fixes-over-blind-session-tuning)
10. [Operations and Reliability](#10-operations-and-reliability)
   - 10.1 [Review Fault-Tolerant Execution as a Cluster Capability](#101-review-fault-tolerant-execution-as-a-cluster-capability)
   - 10.2 [Separate Engine Behavior from Connector Behavior](#102-separate-engine-behavior-from-connector-behavior)
11. [Anti-patterns](#11-anti-patterns)
   - 11.1 [Do Not Cast Join Keys Inside the Join Predicate](#111-do-not-cast-join-keys-inside-the-join-predicate)
   - 11.2 [Do Not Wrap Time Columns in date() or date_trunc() Without Pruning Evidence](#112-do-not-wrap-time-columns-in-date-or-date_trunc-without-pruning-evidence)
   - 11.3 [Do Not Filter Late After Expensive Operators](#113-do-not-filter-late-after-expensive-operators)

---

## Review Protocol

Use this order:

1. Identify engine, catalog, connector, table format, and session properties.
2. Read the SQL and DDL before suggesting anything.
3. Read `EXPLAIN (TYPE DISTRIBUTED)` for plan shape.
4. Use `EXPLAIN ANALYZE` only when actual runtime evidence is needed and execution is safe.
5. Decide whether the issue belongs to SQL shape, table design, file layout, catalog configuration, or session configuration.
6. Cite rule names in the final review.
7. State uncertainty explicitly if connector or version detail is missing.

---

## 1. Plan Reading

Plan-reading rules are first because bad Trino advice usually starts with no plan evidence.

### 1.1 Read Distributed EXPLAIN Fragments Before Tuning

**Impact:** CRITICAL

Trino distributed plans show fragments and exchanges explicitly. Read them before touching join distribution, memory, retry, or spill settings.

**Why**

- `SINGLE`, `HASH`, `BROADCAST`, and `SOURCE` fragments expose where work is centralized or distributed.
- `RemoteExchange[REPARTITION]` often signals major network cost.
- `TableScan` and `ScanFilterProject` show whether filters stayed in Trino instead of being pushed down.

**Detect**

- Advice is being given without `EXPLAIN (TYPE DISTRIBUTED)`.
- The plan contains unexpected repartitioning or single-node fragments.
- Scan layouts include many more columns than the query needs.

**Fix**

- Run `EXPLAIN (TYPE DISTRIBUTED)` first.
- Record fragment types, exchange operators, scan predicates, and join nodes.
- Only then decide whether the fix belongs in SQL or settings.

**Reference**

- https://trino.io/docs/current/sql/explain.html

### 1.2 Recognize Dynamic Filtering in the Plan

**Impact:** HIGH

Do not claim dynamic filtering is active unless the plan shows it.

**Why**

- Trino exposes dynamic filtering through `dynamicFilterAssignments` at the join and dynamic filter predicates at the probe-side scan.
- On Hive, this can also enable dynamic partition pruning.

**Detect**

- The join has `dynamicFilterAssignments`.
- The probe-side scan shows a dynamic filter predicate.
- If neither appears, dynamic filtering is not the reason the query is fast.

**Fix**

- Keep build-side filters selective.
- Use raw equality join keys where possible.
- Confirm the feature from plan evidence, not from vibes.

**Reference**

- https://trino.io/docs/current/admin/dynamic-filtering.html

---

## 2. Query Shape

These rules focus on pushdown, pruning, scan volume, and early row reduction.

### 2.1 Preserve Native Types in Filter Predicates

**Impact:** CRITICAL

Typed columns should meet typed literals. Do not stringify dates, timestamps, numerics, or identifiers unless the domain is actually text.

**Why**

- Predicate pushdown is connector-specific, but Trino makes failed pushdown visible through `ScanFilterProject`.
- Bare strings against typed columns invite coercion and weaker planning.

**Detect**

- `cast(column AS varchar)` appears in `WHERE`.
- A typed column is compared to an untyped text literal.
- The plan shows a scan-side filter that should have been pushed down.

**Fix**

- Use documented typed literals such as `DATE '...'`, `TIMESTAMP '...'`, `UUID '...'`, and `DECIMAL '...'` where they exist.
- For numeric parameters, prefer numeric literals or an explicit `CAST(... AS bigint)` / `CAST(... AS integer)` at the boundary.
- Cast external parameters once outside the scan path if needed.

**Reference**

- https://trino.io/docs/current/optimizer/pushdown.html
- https://trino.io/docs/current/functions/conversion.html

### 2.2 Avoid Ad Hoc Functions on Filter Columns

**Impact:** CRITICAL

Function-wrapped scan columns are pruning risks unless the function exactly matches an intentional partition transform or partition key.

**Why**

- `date(ts)`, `date_trunc(...)`, `lower(col)`, and similar rewrites often leave Trino doing filter work after the read started.
- Range predicates on raw columns are usually the safer default.

**Detect**

- Functions around scan columns in `WHERE`.
- Weak partition pruning.
- `EXPLAIN` still shows scan-side filtering.

**Fix**

- Prefer half-open ranges on raw timestamps.
- If the table uses a transform-aware partition spec, verify the behavior against `SHOW CREATE TABLE` and `EXPLAIN`.

**Reference**

- https://trino.io/docs/current/optimizer/pushdown.html
- https://trino.io/docs/current/connector/hive.html
- https://trino.io/docs/current/connector/iceberg.html

### 2.3 Avoid SELECT Star on Trino Lakehouse Tables

**Impact:** MEDIUM

`SELECT *` is often a projection bug disguised as convenience.

**Why**

- Projection pushdown works best when the query names the needed columns.
- Wide Hive and Iceberg tables can carry declared columns you do not want in the scan path, including ordinary partition columns that are part of the schema.
- Hidden metadata columns such as `"$path"` still need explicit selection; `SELECT *` does not pull them in automatically.

**Detect**

- `SELECT *` on wide lakehouse tables.
- The plan layout contains many more columns than the result requires.

**Fix**

- Enumerate needed columns.
- Add hidden metadata columns only for diagnosis, since `SELECT *` does not include them automatically.

**Reference**

- https://trino.io/docs/current/optimizer/pushdown.html
- https://trino.io/docs/current/sql/select.html

### 2.4 Reduce Scan Volume Early Instead of Hoping LIMIT Saves You

**Impact:** HIGH

`LIMIT` is result shaping, not a scan-pruning strategy.

**Why**

- Without limit pushdown, Trino still reads and processes much more data than the result size suggests.
- The real win comes from partition predicates, business-key filters, and early reduction.

**Detect**

- Huge tables queried with only `LIMIT`.
- `LIMIT` appears after joins or aggregations.

**Fix**

- Add real predicates first.
- Treat `LIMIT` as secondary.

**Reference**

- https://trino.io/docs/current/optimizer/pushdown.html
- https://trino.io/docs/current/sql/select.html

### 2.5 Aggregate Before the Join When Semantics Allow

**Impact:** HIGH

If the result only needs aggregated facts, do not join raw fact rows first.

**Why**

- Pre-aggregation cuts exchange volume and hash-table size.
- This is frequently more effective than session tuning.

**Detect**

- Large fact-to-dimension join followed by grouping on the join key or dimension attribute.
- Tiny grouped output after a massive join.

**Fix**

- Aggregate the many-side to the needed grain first.
- Keep row-level joins only when row-level semantics are required.

**Reference**

- https://trino.io/docs/current/sql/select.html
- https://trino.io/docs/current/optimizer/cost-based-optimizations.html

---

## 3. Joins

Most expensive Trino failures are join-shape failures.

### 3.1 Choose Join Distribution from Build-Side Size, Not Habit

**Impact:** CRITICAL

Broadcast is for small filtered build sides. Partitioned joins are for larger joins that do not fit per node.

**Why**

- Broadcast replicates the build side to every worker.
- Partitioned redistributes both sides but scales to much larger joins.

**Detect**

- Someone wants to force `join_distribution_type` without plan evidence.
- The join plan shows `BROADCAST` or `REPARTITION`.
- The build side is clearly too large for broadcast.

**Fix**

- Prefer `AUTOMATIC` unless you have evidence.
- If forcing broadcast, validate filtered build-side size and memory fit.
- Use `join_max_broadcast_table_size` deliberately when capping broadcast.

**Reference**

- https://trino.io/docs/current/optimizer/cost-based-optimizations.html

### 3.2 Filter Join Inputs Before Joining

**Impact:** CRITICAL

Static filters should reach each join input before the hash join is built.

**Why**

- Dynamic filtering helps later, but static filters reduce scan, exchange, and hash-table work immediately.

**Detect**

- Filters appear only after the join.
- The build side is unfiltered even though it could be small.

**Fix**

- Push filters into the input relations.
- Use the plan to confirm the scan predicates moved down.

**Reference**

- https://trino.io/docs/current/admin/dynamic-filtering.html
- https://trino.io/docs/current/optimizer/cost-based-optimizations.html

### 3.3 Watch for Join Row Explosion

**Impact:** CRITICAL

Many slow joins are actually cardinality mistakes.

**Why**

- Duplicate keys on both sides multiply rows.
- That multiplication creates exchange, memory, and spill pain everywhere downstream.

**Detect**

- `EXPLAIN ANALYZE` shows join output much larger than both inputs.
- Fact-to-fact or many-to-many joins.
- `DISTINCT` appears after the join to clean up damage.

**Fix**

- Validate intended grain.
- Aggregate or deduplicate before the join.
- Re-check runtime stats after the rewrite.

**Reference**

- https://trino.io/docs/current/sql/explain-analyze.html
- https://trino.io/docs/current/optimizer/cost-based-optimizations.html

### 3.4 Use EXISTS or IN for Membership Tests

**Impact:** HIGH

If only left-side columns survive, the query probably wanted a membership test, not a full join.

**Why**

- `JOIN` plus `DISTINCT` often indicates a hidden semi-join problem.
- `EXISTS` or `IN` expresses the intent directly and avoids needless duplication.

**Detect**

- Output uses only left-table columns.
- `DISTINCT` is used after an inner join.

**Fix**

- Rewrite as `EXISTS` or `IN`.
- Keep full joins only when right-side payload columns are truly needed.

**Reference**

- https://trino.io/docs/current/sql/select.html
- https://trino.io/docs/current/sql/explain.html

---

## 4. Types and Predicate Semantics

Type discipline is a direct performance concern in Trino, not just style.

### 4.1 Avoid Implicit Type Mismatches on Join and Filter Keys

**Impact:** CRITICAL

Key types should be aligned before they reach the main query body.

**Why**

- Implicit coercions complicate planning.
- They can weaken pushdown and make joins more expensive.

**Detect**

- Key types differ across sources.
- Join or filter logic casts keys inline.

**Fix**

- Normalize types upstream or in a pre-projection.
- Do not standardize everything to `varchar` out of laziness.

**Reference**

- https://trino.io/docs/current/functions/conversion.html
- https://trino.io/docs/current/optimizer/pushdown.html

### 4.2 Treat VARBINARY and VARCHAR as Different Domains

**Impact:** HIGH

Bytes are not strings. Trino exposes separate function families because the distinction matters.

**Why**

- Hashes and encoded payloads frequently live in `varbinary`.
- Text comparison against binary values without conversion is wrong or brittle.

**Detect**

- Binary outputs from `sha256`, `md5`, or `from_hex` compared directly to text.
- Binary values cast to `varchar` just to make the query "work".

**Fix**

- Use `to_utf8` when text becomes bytes.
- Use `to_hex` / `from_hex` or base64 helpers when bytes must meet text.

**Reference**

- https://trino.io/docs/current/functions/string.html
- https://trino.io/docs/current/functions/binary.html

### 4.3 Do Not Assume CTE Materialization

**Impact:** CRITICAL

`WITH` in Trino is inlined. It is not a reliable one-time execution boundary.

**Why**

- Reusing a heavy CTE can duplicate work.
- Reusing a non-deterministic CTE can produce different results.

**Detect**

- Same CTE referenced multiple times.
- `random()`, `now()`, or similar logic inside the CTE.

**Fix**

- Materialize explicitly with a staging table or CTAS when reuse matters.
- Treat CTEs as readability constructs.

**Reference**

- https://trino.io/docs/current/sql/select.html

---

## 5. Explain Usage

### 5.1 Use the Right EXPLAIN Mode for the Question

**Impact:** HIGH

Different Trino explain modes answer different questions.

**Why**

- `EXPLAIN (TYPE DISTRIBUTED)` gives structure.
- `EXPLAIN ANALYZE` executes the query and reports actual runtime.
- `EXPLAIN (TYPE IO)` answers footprint questions.
- `EXPLAIN (TYPE VALIDATE)` checks parse and resolution.

**Detect**

- Runtime claims backed only by plain `EXPLAIN`.
- `EXPLAIN ANALYZE` used casually on statements that should not be executed.

**Fix**

- Pick the explain mode that matches the question.
- Be explicit when execution is required to get the answer.

**Reference**

- https://trino.io/docs/current/sql/explain.html
- https://trino.io/docs/current/sql/explain-analyze.html

---

## 6. Hive Connector

Hive reviews live or die on partition pruning, file-format awareness, and dynamic filtering.

### 6.1 Verify Hive Dynamic Filtering and Dynamic Partition Pruning

**Impact:** HIGH

Selective build-side joins can save a lot of Hive scan work when dynamic filtering is actually active.

**Why**

- Trino can push dynamic filters into the probe-side Hive scan.
- On partitioned Hive tables, this can prune partitions before they are read.

**Detect**

- `dynamicFilterAssignments` at the join.
- Dynamic filter predicates at the probe-side scan.

**Fix**

- Keep the build side selective.
- Keep equality joins on raw keys.
- Confirm the feature in the plan instead of assuming it.

**Reference**

- https://trino.io/docs/current/admin/dynamic-filtering.html
- https://trino.io/docs/current/connector/hive.html

### 6.2 Filter Hive Partition Keys Directly

**Impact:** CRITICAL

The best Hive partitions are the ones Trino never schedules.

**Why**

- Partition pruning depends on predicates on the actual partition keys.
- Filtering only raw timestamps or derived expressions is often weaker than filtering `ds`, `hour`, or similar declared keys.

**Detect**

- `SHOW CREATE TABLE` reveals partition keys that the query never mentions.
- Metadata diagnostics show many partitions touched.

**Fix**

- Filter declared partition columns directly.
- Keep those predicates typed.

**Reference**

- https://trino.io/docs/current/connector/hive.html

### 6.3 Respect Hive File-Format Pruning Limits

**Impact:** HIGH

Do not talk about ORC, Parquet, and text formats as if they all prune the same way.

**Why**

- Projection and typed predicates matter more on columnar formats.
- Weak filter shape and `SELECT *` waste those advantages.

**Detect**

- Wide `SELECT *` on ORC or Parquet.
- Regex-heavy or stringified filters on typed columns.

**Fix**

- Keep projection narrow.
- Keep predicates typed and selective.
- Distinguish partition pruning from file-format-level reduction.

**Reference**

- https://trino.io/docs/current/connector/hive.html
- https://trino.io/docs/current/optimizer/pushdown.html

---

## 7. Iceberg Connector

Iceberg reviews must be transform-aware and metadata-first.

### 7.1 Reason About Iceberg Partition Transforms Explicitly

**Impact:** CRITICAL

Iceberg partitioning is often transform-based, not raw-column based.

**Why**

- `month(ts)`, `day(ts)`, `bucket(id, n)`, and similar transforms change what pruning means.
- Advice that ignores the real transform is wrong by construction.

**Detect**

- `partitioning = ARRAY[...]` in table properties.
- Reviews that discuss pruning without naming the transform.

**Fix**

- Read the partition spec first.
- Align query advice and redesign advice with the actual transform.

**Reference**

- https://trino.io/docs/current/connector/iceberg.html

### 7.2 Use Iceberg Metadata Tables for Diagnosis

**Impact:** HIGH

Use `$partitions`, `$files`, `$snapshots`, and `$history` before guessing.

**Why**

- Iceberg exposes direct evidence about layout and history.
- Metadata tables beat hand-wavy storage speculation.

**Detect**

- The review guesses about file counts or partition spread.
- No metadata query was run on an Iceberg table with a layout or retention problem.

**Fix**

- Query metadata tables first.
- Base layout and maintenance recommendations on those results.

**Reference**

- https://trino.io/docs/current/connector/iceberg.html

### 7.3 Use Iceberg Snapshots as First-Class Operational Evidence

**Impact:** HIGH

Iceberg tables are versioned. Operate like they are versioned.

**Why**

- Time travel, rollback, and retention hygiene all depend on snapshot metadata.
- Direct file deletion is the wrong abstraction.

**Detect**

- Regression or rollback questions with no snapshot inspection.
- Metadata bloat from old snapshots.

**Fix**

- Inspect `$snapshots` and `$history`.
- Use time travel, rollback procedures, and `expire_snapshots` where appropriate.

**Reference**

- https://trino.io/docs/current/connector/iceberg.html

### 7.4 Review Iceberg File Count and File Size Before Blaming Trino

**Impact:** HIGH

Small files are often the real problem.

**Why**

- Excessive file counts hurt planning and split scheduling.
- Fixes may belong in write batching or table maintenance instead of query tuning.

**Detect**

- `$files` shows too many tiny files.
- Planning time is high relative to scanned bytes.

**Fix**

- Measure layout from metadata tables.
- If fragmented, consider write-side batching and Iceberg maintenance such as `optimize`.

**Reference**

- https://trino.io/docs/current/connector/iceberg.html

---

## 8. Lakehouse Connector

### 8.1 Treat Lakehouse as a Wrapper, Not a New Semantics Layer

**Impact:** CRITICAL

Lakehouse combines multiple connector behaviors. It does not erase them.

**Why**

- The Trino documentation explicitly says Lakehouse behavior comes from Hive, Iceberg, Delta Lake, and Hudi connectors.
- Table type matters for pruning, metadata, maintenance, and retries.

**Detect**

- Advice ignores whether the table is `HIVE`, `ICEBERG`, `DELTA`, or `HUDI`.
- The review attributes format behavior to Trino core.

**Fix**

- Identify table type first.
- Route the analysis to the underlying connector semantics.

**Reference**

- https://trino.io/docs/current/connector/lakehouse.html
- https://trino.io/docs/current/connector.html

---

## 9. Session and Execution

Session tuning is last on purpose.

### 9.1 Do Not Suggest retry_policy Blindly

**Impact:** HIGH

`retry_policy` is part of fault-tolerant execution, not a generic query-speed knob.

**Why**

- `TASK` retries require an exchange manager.
- Session-mode switching is not the normal path for performance work.
- Connector support varies.

**Detect**

- Retry advice appears with no mention of cluster configuration.
- The actual problem is query slowness, not retryable failures.

**Fix**

- Confirm the cluster is FTE-enabled first.
- Keep retry policy advice scoped to resiliency, not primary SQL optimization.

**Reference**

- https://trino.io/docs/current/admin/fault-tolerant-execution.html

### 9.2 Treat Spill and Memory Knobs as Secondary to Query Shape

**Impact:** HIGH

More memory or spill can make a structurally bad query slower instead of better.

**Why**

- Spill is documented as legacy functionality.
- Spilled queries can slow down by orders of magnitude.
- Memory tuning does not fix bad join shape, excessive grouping cardinality, or small-file layout.

**Detect**

- The first recommendation is to raise memory or enable spill.
- The plan already shows a structural problem.

**Fix**

- Rewrite the query first.
- Tune memory only after the remaining workload shape is defensible.

**Reference**

- https://trino.io/docs/current/admin/spill.html
- https://trino.io/docs/current/admin/fault-tolerant-execution.html

### 9.3 Prefer Query-Shape Fixes Over Blind Session Tuning

**Impact:** CRITICAL

Portable wins beat cluster-specific folklore.

**Why**

- SQL shape fixes survive across clusters better than force-setting `join_distribution_type`, retry settings, or wait timeouts.
- Session properties are sensitive to connector and cluster context.

**Detect**

- `SET SESSION` appears before plan evidence.
- Obvious SQL anti-patterns remain unaddressed.

**Fix**

- Inspect SQL and plan first.
- Remove structural anti-patterns before recommending settings.

**Reference**

- https://trino.io/docs/current/sql/explain.html
- https://trino.io/docs/current/optimizer/cost-based-optimizations.html
- https://trino.io/docs/current/admin/dynamic-filtering.html

---

## 10. Operations and Reliability

### 10.1 Review Fault-Tolerant Execution as a Cluster Capability

**Impact:** HIGH

Retries are not ambient behavior on a default Trino cluster.

**Why**

- Fault-tolerant execution is disabled by default.
- `QUERY` and `TASK` policies have different workload fit.
- Connector support varies, and large results have exchange-storage caveats.

**Detect**

- The review assumes failures should retry automatically.
- Exchange-manager presence is unknown.

**Fix**

- Confirm cluster configuration and connector support.
- Treat FTE as architecture, not superstition.

**Reference**

- https://trino.io/docs/current/admin/fault-tolerant-execution.html

### 10.2 Separate Engine Behavior from Connector Behavior

**Impact:** CRITICAL

The sentence "Trino does not support this" is often sloppy and wrong.

**Why**

- Planning is often an engine concern.
- Pushdown, metadata tables, file-format behavior, and some retry support are connector concerns.

**Detect**

- The review never names the connector.
- Engine, connector, and storage-format concerns are mixed together.

**Fix**

- Label each recommendation as engine, connector, file layout, table design, or session scope.
- State uncertainty when the connector is unknown.

**Reference**

- https://trino.io/docs/current/connector.html
- https://trino.io/docs/current/optimizer/pushdown.html
- https://trino.io/docs/current/connector/lakehouse.html

---

## 11. Anti-patterns

These are fast blockers. If you see them, call them out.

### 11.1 Do Not Cast Join Keys Inside the Join Predicate

**Impact:** CRITICAL

Join-time casting is almost always the wrong place to repair schema drift.

**Why**

- The join should operate on aligned keys.
- Inline casts complicate planning and weaken a clean join shape.

**Detect**

- `ON cast(left_key AS ...) = right_key`
- `ON left_key = cast(right_key AS ...)`

**Fix**

- Normalize keys upstream or in a pre-projection.
- If unavoidable, cast the smaller side once outside the join and document the trade-off.

**Reference**

- https://trino.io/docs/current/functions/conversion.html
- https://trino.io/docs/current/optimizer/pushdown.html

### 11.2 Do Not Wrap Time Columns in date() or date_trunc() Without Pruning Evidence

**Impact:** CRITICAL

This is a common pruning killer.

**Why**

- Raw ranges on timestamps are usually safer than function-wrapped predicates.
- If a table uses a transform-aware partition spec, prove the rewrite works before blessing it.

**Detect**

- `date(ts_col) = ...`
- `date_trunc(...) = ...`
- Pruning regressed after a readability rewrite.

**Fix**

- Prefer half-open ranges on raw timestamps.
- Filter the real partition key when available.

**Reference**

- https://trino.io/docs/current/optimizer/pushdown.html
- https://trino.io/docs/current/connector/hive.html
- https://trino.io/docs/current/connector/iceberg.html

### 11.3 Do Not Filter Late After Expensive Operators

**Impact:** HIGH

Late filtering makes joins, windows, and aggregations process rows they should never have seen.

**Why**

- Scan-time predicates are almost always cheaper than outer-query cleanup.
- Late filters inflate exchange and memory pressure.

**Detect**

- Outer query blocks filter columns that could have been filtered earlier.
- `EXPLAIN ANALYZE` shows large operator input and tiny post-filter output.

**Fix**

- Push filters into the earliest legal relation.
- If semantics truly prevent that move, say so explicitly.

**Reference**

- https://trino.io/docs/current/sql/explain-analyze.html
- https://trino.io/docs/current/optimizer/pushdown.html
