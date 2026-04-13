# Trino Agent Skills

Agent skills for Trino-focused SQL review, plan analysis, and connector-aware performance guidance.

## Repository Structure

```text
trino-best-practices-skill/
├── AGENTS.md
├── README.md
└── skills/
    └── trino-best-practices/
        ├── AGENTS.md
        ├── README.md
        ├── SKILL.md
        ├── agents/
        │   └── openai.yaml
        ├── metadata.json
        └── rules/
```

## Available Skill

### Trino Best Practices

Review-first guidance for:

- Trino SQL optimization
- `EXPLAIN` and `EXPLAIN ANALYZE` reviews
- Hive and Iceberg connector behavior
- Lakehouse connector boundaries
- Session-setting and execution-tuning sanity checks

Skill path: [`skills/trino-best-practices/`](./skills/trino-best-practices/)

For humans:

- Overview: [`skills/trino-best-practices/SKILL.md`](./skills/trino-best-practices/SKILL.md)
- Full compiled guide: [`skills/trino-best-practices/AGENTS.md`](./skills/trino-best-practices/AGENTS.md)

## Installation

Copy the skill directory into any supported agent skill root:

```bash
mkdir -p ~/.codex/skills
cp -R skills/trino-best-practices ~/.codex/skills/
```

## Notes

- The repo is intentionally shaped like a multi-skill repository, even though it currently contains one skill.
- `skills/trino-best-practices/agents/openai.yaml` is UI metadata and should stay aligned with `SKILL.md`.
