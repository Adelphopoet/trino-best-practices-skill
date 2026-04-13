# AGENTS.md

This repo stores agent skills for Trino work.

## Repository Layout

```text
trino-best-practices-skill/
├── AGENTS.md
├── README.md
└── skills/
    └── trino-best-practices/
        ├── SKILL.md
        ├── AGENTS.md
        ├── README.md
        ├── metadata.json
        ├── agents/openai.yaml
        └── rules/*.md
```

## Maintainer Rules

- Keep the skill under `skills/trino-best-practices/`.
- Treat `SKILL.md` as the trigger surface and operating guide.
- Treat `AGENTS.md` as the compiled long-form reference.
- Keep `agents/openai.yaml` in sync with `SKILL.md` metadata and scope.
- Add new rule files under `rules/` with stable kebab-case names.
- Do not invent generic SQL advice when a Trino- or connector-specific rule exists.

## Skill-Creator Notes

- The required piece is `SKILL.md` with valid frontmatter.
- `agents/openai.yaml` is recommended and included.
- The in-skill `README.md` and `metadata.json` are packaging extras. They are fine for a repo like this, but they are not required by the bare minimum skill format.
