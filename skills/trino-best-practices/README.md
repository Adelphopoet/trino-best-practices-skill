# trino-best-practices

`trino-best-practices` is a production-ready `SKILL.md` package for agents that need to review Trino SQL, read Trino plans, and troubleshoot Hive and Iceberg workloads without giving generic warehouse advice. It also includes Lakehouse boundary guidance so agents do not fake uniform semantics across table types.

## What problems it solves

- Slow Trino queries with no evidence-driven diagnosis
- Bad predicate shape that kills pushdown or pruning
- Join regressions caused by wrong distribution or row explosion
- Confusion between Trino engine behavior and connector behavior
- Weak reviews of Hive and Iceberg table/query design
- Bad Lakehouse advice that ignores the underlying table type
- Blind tuning of session properties before reading the plan

## What is included

- `SKILL.md`: compact execution workflow for agents
- `AGENTS.md`: compiled long-form reference manual
- `metadata.json`: versioned package metadata and official references
- `rules/`: atomic Trino rules with examples and official docs links

Lakehouse note:

- Hive-backed and Iceberg-backed Lakehouse tables are covered through the Hive and Iceberg rule sets.
- Delta-backed and Hudi-backed Lakehouse tables are only covered at the connector-boundary level in this package.

## Trigger phrases

- "Optimize this Trino query"
- "Review this EXPLAIN plan"
- "Why is partition pruning not happening?"
- "Is dynamic filtering helping here?"
- "Review this Trino Iceberg table design"
- "Review this Lakehouse query and tell me whether it is Hive or Iceberg underneath"
- "Should I force broadcast join in Trino?"
- "Why is this Hive table scanning so much data?"

## Installation

### Claude Code

```bash
mkdir -p .claude/skills
cp -R skills/trino-best-practices .claude/skills/
```

### Cursor / project-local skills folder

```bash
mkdir -p .cursor/skills
cp -R skills/trino-best-practices .cursor/skills/
```

### Manual copy

Copy the folder to any skill root your agent loads:

```bash
mkdir -p ./skills
cp -R skills/trino-best-practices ./skills/
```

## Example prompts

```text
Optimize this Trino query. Focus on pushdown, join distribution, and Iceberg partition pruning.
```

```text
Review this Trino EXPLAIN ANALYZE output. Tell me whether dynamic filtering and broadcast joins are helping.
```

```text
Why is partition pruning not happening on this Hive table? Here is the SQL and SHOW CREATE TABLE output.
```

```text
Is dynamic filtering helping here, or is the query still scanning the full probe side?
```

```text
Review this Trino Iceberg table design. Check partition transforms, file layout, and metadata-table diagnostics.
```

## Notes

- The package is optimized for Trino current documentation behavior.
- The rules intentionally separate SQL shape, connector behavior, file layout, and session tuning.
- If your agent runtime uses a different skill root, keep the folder contents unchanged and move the directory as-is.
