# jacard-copper-core-skill

Agent skill for using the **Jacard Copper Core MCP** well — retail catalog search (keyword / semantic / hybrid), deterministic inventory listing, product detail, collections, and readonly SQL over `llm_*` views.

## Install

```bash
npx skills add https://github.com/Olympus-Skills/jacard-copper-core-skill
```

This installs the skill into your agent (e.g. `.claude/skills/`). It assumes the **Jacard Copper Core MCP is already configured** in your agent with a valid API key — the skill is usage guidance, not connection setup.

## Contents

- [`SKILL.md`](SKILL.md) — self-contained usage guide: tool selection, scoping, budgets/errors, the full tool reference, and the `llm_*` SQL views reference.
