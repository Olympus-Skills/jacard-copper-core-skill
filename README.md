# jacard-copper-core-skill

Agent skill for using the **Jacard Copper Core MCP** well — retail catalog search (keyword / semantic / hybrid), deterministic inventory listing, product detail, collections, and readonly SQL over `llm_*` views.

## Install

```bash
npx skills add https://github.com/Olympus-Skills/jacard-copper-core-skill
```

This installs the skill into your agent (e.g. `.claude/skills/`). It assumes the **Jacard Copper Core MCP is already configured** in your agent with a valid API key — the skill is usage guidance, not connection setup.

## Contents

- [`SKILL.md`](SKILL.md) — when to use each tool, scoping, budget/error handling.
- [`references/tools.md`](references/tools.md) — full tool contracts and example arguments.
- [`references/sql-views.md`](references/sql-views.md) — `llm_*` view columns and safe SQL patterns.
