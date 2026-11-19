# factory-kit

A software factory for Claude Code, ready to install into any GitHub project:
GitHub issues labeled `factory` become reviewed pull requests through
intake → spec → build → gate → review → approve, with the tooling around it.

- **Software factory**: from
  [zazencodes/zazencodes-season-3](https://github.com/zazencodes/zazencodes-season-3/tree/main/src/software-factory-claude-code-v2),
  extended with dependent tickets (`factory:blocked` + `factory/bin/unblock` +
  a GitHub workflow), project knowledge, code graph and skills in every agent.
- **[mattpocock/skills](https://github.com/mattpocock/skills)**: `/grill-with-docs`,
  `/to-spec`, `/to-tickets`, `/triage` feed the factory; TDD and design skills
  inside it.
- **[CodeGraph](https://github.com/codegraph-ai/CodeGraph)**: code graph MCP
  (callers, impact, related tests), graph-only.
- **[LLM Wiki](https://github.com/microsoft/llmwiki)**: the project's knowledge
  base MCP, read by the agents.
- **[ponytail](https://github.com/DietrichGebert/ponytail)**: the smallest
  complete change, in chat and in every agent.
- **[TokenEconomizer](https://github.com/microsoft/TokenEconomizer)**: learns
  from past sessions to cut cost.

## Install

Open Claude Code in the root of your project and say:

```
Read https://raw.githubusercontent.com/Add0z/factory-kit/v1.1.0/INSTALL.md and follow it.
```

Requirements: a GitHub repo, `gh` logged in, Claude Code 2.1.290+, Python
3.10+, Node 20+. Stack overlays in `stacks/` (Java/Spring so far) add
stack-specific gates and agent text.

## Update

In a project that has `factory/KIT_VERSION`:

```
Read https://raw.githubusercontent.com/Add0z/factory-kit/main/UPDATE.md and follow it.
```

It applies only what changed in the kit since that version and keeps the
project's own text. Changes per version: `CHANGELOG.md`.

## Use

`kit/factory-docs/01-workflow.md` (Portuguese) walks through the whole flow,
from `/grill-with-docs` to a merged PR.

## License

GPL-3.0 (see `LICENSE`), because the factory files are derived from
zazencodes' GPL-3.0 work. Origins and changes: `NOTICE.md`.
