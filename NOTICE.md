# Notice

This kit is licensed under GPL-3.0 (`LICENSE`). It contains or modifies:

| Files | Origin | License | Changes |
|---|---|---|---|
| `kit/.claude/skills/factory/SKILL.md`, `kit/.claude/agents/*.md`, `kit/factory/bin/job`, `gate`, `route`, `kit/factory/dashboard.html`, `kit/factory/checks.sh`, `kit/factory/config.json` | [zazencodes/zazencodes-season-3](https://github.com/zazencodes/zazencodes-season-3), `src/software-factory-claude-code-v2` | GPL-3.0 | Agents: `tools:` lines and sections for CodeGraph, LLM Wiki, LSP, Matt Pocock's skills, ponytail and TokenEconomizer; `ADAPT` markers for project text; API/CLI contract review in `reviewer-ux`. Skill: runs `factory/bin/unblock`. Config: `labels.blocked`. `job`, `gate`, `route`, `dashboard.html`, `checks.sh`: unchanged. |
| `kit/factory/bin/unblock`, `kit/.github/workflows/factory-unblock.yml`, `kit/docs/adr/0001-codegraph-graph-only.md`, `kit/factory-docs/`, `stacks/`, `INSTALL.md` | This kit | GPL-3.0 | — |
| `kit/docs/agents/issue-tracker.md`, `triage-labels.md`, `domain.md` | [mattpocock/skills](https://github.com/mattpocock/skills), `setup-matt-pocock-skills` templates | MIT, © Matt Pocock | Filled in for a GitHub tracker; the factory's labels and dependent-ticket rules added. |

Installed by `INSTALL.md` from their own sources (not redistributed here):
CodeGraph (Apache-2.0), LLM Wiki server and skills (MIT, © Microsoft),
ponytail (MIT), TokenEconomizer (MIT, © Microsoft), Matt Pocock's skills
plugin (MIT).
