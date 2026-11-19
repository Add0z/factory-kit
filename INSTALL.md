# Install the factory kit

You are a coding agent installing this software factory into the user's
project. Copy what's generic, adapt what's project-specific, verify it, and
report. Work in the project's root. Don't clone this repository.

Files come from this kit at a fixed tag:

```
KIT=https://raw.githubusercontent.com/Add0z/factory-kit/v1.0.0
```

Ask the user before anything marked **[USER]**, before pushing, and before
changing the project's own code (as opposed to the factory's files). Reply in
the user's language.

What gets installed: the software factory (`/factory`: intake → spec → build →
gate → review → approve, one PR per GitHub issue) plus dependent-ticket
handling (`factory:blocked` + `factory/bin/unblock`), Matt Pocock's skills,
CodeGraph (code graph MCP), LLM Wiki (knowledge-base MCP), ponytail, and
TokenEconomizer. Origins and licenses: `NOTICE.md`.

## 0. Check the project

Stop and tell the user if any of these fail:

- It's a git repo with an `origin` on GitHub, and `gh auth status` shows a user
  who can push to it and manage its labels. No `origin`: offer
  `gh repo create <name> --private --source . --push` **[USER: name,
  visibility]**; a repo with no commits needs a first commit. `gh` login is
  interactive: ask the user to run `! gh auth login`.
- Its default branch (`gh repo view --json defaultBranchRef`) exists on
  `origin`. If it prints an empty name on a new repo, run
  `gh repo edit --default-branch <current branch>`.
- `claude --version` is 2.1.290 or later; `python3 --version` 3.10 or later;
  `node --version` 20 or later.
- None of the files in step 1 already exist. If one does, show the user the
  clash and ask before overwriting.

## 1. Copy

Copy each file from `$KIT/kit/<path>` to `<path>` in the project, unchanged:

```
.claude/skills/factory/SKILL.md
.claude/agents/intake.md
.claude/agents/spec-writer.md
.claude/agents/builder.md
.claude/agents/reviewer-code.md
.claude/agents/reviewer-security.md
.claude/agents/reviewer-ux.md
.claude/agents/reviewer-ui.md
.claude/agents/approver.md
factory/config.json
factory/checks.sh
factory/bin/job
factory/bin/gate
factory/bin/route
factory/bin/unblock
factory/dashboard.html
.github/workflows/factory-unblock.yml
docs/agents/issue-tracker.md
docs/agents/triage-labels.md
docs/agents/domain.md
docs/adr/0001-codegraph-graph-only.md
factory-docs/01-workflow.md
```

```bash
KIT=https://raw.githubusercontent.com/Add0z/factory-kit/v1.0.0
for f in <the paths above>; do
  mkdir -p "$(dirname "$f")" && curl -fsSL "$KIT/kit/$f" -o "$f"
done
chmod +x factory/checks.sh factory/bin/job factory/bin/gate factory/bin/route factory/bin/unblock
```

If the project already has a `docs/adr/0001-*`, renumber the kit's ADR to the
next free number and fix the reference to it in `AGENTS.md` (step 4).

## 2. Adapt the factory

Find the project's stack and commands in `AGENTS.md`, `CLAUDE.md`,
`package.json`, `Makefile`, `pyproject.toml`, `build.gradle`/`pom.xml` and CI
config. **If a file in `$KIT/stacks/` matches the stack (for example
`stacks/java-spring.md`), read it now and apply it in this step**; it has
exact text for everything below.

- **`factory/checks.sh`**: replace the placeholder with the project's real
  checks: install dependencies if a fresh worktree needs them, then lint,
  typecheck, tests and build. It runs with a job's worktree as the working
  directory, must be non-interactive, must not rewrite files (no `--fix`), must
  exit non-zero on any failure, and keeps `set -euo pipefail`. Prefer putting
  lint/format/static checks inside the build tool so `checks.sh` only runs it.
- **`factory/config.json`**: `base_branch` = the default branch;
  `merge_method` one the repo allows
  (`gh api repos/{owner}/{repo} --jq '{allow_merge_commit, allow_squash_merge, allow_rebase_merge}'`);
  `protected_paths`: keep the two factory entries, add the migrations folder if
  the project has one (new files pass the gate, edits to merged ones don't),
  never add the project's tests; `ports.base`: move it if 5100–5199 is in use;
  a backend with no screens: drop `"ui": "haiku"` from `surface_reviewers.ui`.
  Leave `concurrency`, `max_rounds`, `routing` and prices as they are.
- **Agents**: every `<!-- ADAPT: ... -->` comment in `.claude/agents/` must be
  replaced by project-specific text (what it asks for), or deleted when it
  doesn't apply. Don't change anything else: structure, file names, first-line
  formats, one-line replies and `tools:` lines stay. Check with
  `grep -rn "ADAPT" .claude/agents` (must print nothing).
- **`.gitignore`**: append

  ```gitignore
  /factory/jobs/
  /.worktrees/
  .claude/settings.local.json
  ```

  If it ignores `bin/` (common in Java projects), also add `!/factory/bin/`.
  Never ignore `.claude/` or `factory/`.
- **Labels**:

  ```bash
  gh label create factory --color 5319E7 --description "Build this with the software factory"
  gh label create factory:needs-info --color FBCA04 --description "The factory asked the author a question"
  gh label create factory:needs-human --color D93F0B --description "The factory needs a human decision on the PR"
  gh label create factory:blocked --color C5DEF5 --description "Waits for its blocking issues; factory/bin/unblock queues it when they are done"
  for l in needs-triage needs-info ready-for-human wontfix; do gh label create "$l" 2>/dev/null || true; done
  ```

## 3. Install the tools

Each tool's files live outside the repo (per machine). Skip a download that's
already there with the same version.

**Language server (if one exists for the stack).** Claude Code's official LSP
plugins: `jdtls-lsp` (Java), `typescript-lsp`, `pyright-lsp`, `gopls-lsp`,
`rust-analyzer-lsp`, and others. Install the binary its README names, then
`claude plugin install <name>@claude-plugins-official --scope project`.
The agents already list the `LSP` tool; it is simply absent without a plugin.

**Matt Pocock's skills.**

```bash
claude plugin install mattpocock-skills@claude-plugins-official --scope project
```

The kit's `docs/agents/*.md` are what `/setup-matt-pocock-skills` would write
for this setup (GitHub tracker; the `ready-for-agent` role is the `factory`
label, and a ticket with open blockers gets `factory:blocked`; single-context
`GLOSSARY.md` + `docs/adr/`), so the user doesn't need to run it. Step 4 adds
the `## Agent skills` block.

**CodeGraph** (v0.21.0). Don't use the npm wrapper `@astudioplus/codegraph-mcp`:
it sends telemetry to PostHog by default. Use the release binary and check its
checksum. Pick the asset for this machine (`darwin-arm64`, `darwin-x64`,
`linux-x64`, `linux-arm64`):

```bash
A=codegraph-server-darwin-arm64
mkdir -p ~/.codegraph/bin && cd ~/.codegraph/bin
gh release download v0.21.0 -R codegraph-ai/CodeGraph -p "$A*" --clobber
[ "$(shasum -a 256 $A | cut -d' ' -f1)" = "$(cut -d' ' -f1 $A.sha256)" ] && chmod +x $A
```

Stop if the checksums differ. Never strip macOS quarantine attributes
(`xattr -d`). It runs graph-only (see the ADR you copied).

**LLM Wiki** (microsoft/llmwiki at a fixed commit; not on npm):

```bash
C=32da49bf97743431a8cb3ff46e0e31c7fc629b18
mkdir -p ~/.llmwiki && cd ~/.llmwiki
curl -fsSL "https://raw.githubusercontent.com/microsoft/llmwiki/$C/plugin/server/llmwiki-mcp.cjs" -o llmwiki-mcp.cjs
curl -fsSL "https://raw.githubusercontent.com/microsoft/llmwiki/$C/LICENSE" -o LICENSE
echo $C > VERSION
shasum -a 256 llmwiki-mcp.cjs   # expect 8f48e94c9177260a1f6802eebc7065bd32b3eac1e18d95e15bde565e0f9bbfa3
```

Then, in the project root, create the wiki. With `.wiki` as its argument the
server refuses to start until the folder exists; without one it offers
`wiki_init`:

```bash
printf '%s\n' \
 '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"setup","version":"1"}}}' \
 '{"jsonrpc":"2.0","method":"notifications/initialized"}' \
 "{\"jsonrpc\":\"2.0\",\"id\":3,\"method\":\"tools/call\",\"params\":{\"name\":\"wiki_init\",\"arguments\":{\"path\":\"$PWD\"}}}" \
 | (node ~/.llmwiki/llmwiki-mcp.cjs & P=$!; sleep 5; kill $P)
for d in entities concepts sources; do touch .wiki/wiki/$d/.gitkeep; done
for s in wiki-ask wiki-research wiki-ingest wiki-save wiki-fix; do
  mkdir -p .claude/skills/$s
  curl -fsSL "https://raw.githubusercontent.com/microsoft/llmwiki/$C/plugin/skills/$s/SKILL.md" -o .claude/skills/$s/SKILL.md
  cp ~/.llmwiki/LICENSE .claude/skills/$s/LICENSE
done
```

Don't put a `.gitkeep` in `.wiki/raw/`: it would count as a source. `.wiki/`
is committed.

**`.mcp.json`** in the project root (use the CodeGraph asset name from above;
add an `--exclude` for any other large generated folder):

```json
{
  "mcpServers": {
    "codegraph": {
      "command": "${HOME}/.codegraph/bin/codegraph-server-darwin-arm64",
      "args": ["--mcp", "--graph-only", "--exclude", ".worktrees", "--exclude", ".wiki"],
      "env": { "CODEGRAPH_TELEMETRY": "off" }
    },
    "llmwiki": {
      "command": "node",
      "args": ["${HOME}/.llmwiki/llmwiki-mcp.cjs", ".wiki"]
    }
  }
}
```

and in `.claude/settings.json` (create it, or merge into it):
`"enabledMcpjsonServers": ["codegraph", "llmwiki"]`.

**ponytail** (only from `DietrichGebert/ponytail`):

```bash
claude plugin marketplace add DietrichGebert/ponytail --scope project
claude plugin install ponytail@ponytail --scope project
```

Leave its `webhookUrl` empty (its metrics stay in `~/.config/ponytail/band/`).

**TokenEconomizer** (v0.1.0; runs from its clone):

```bash
mkdir -p ~/.local/share && cd ~/.local/share
[ -d TokenEconomizer ] || git clone --branch v0.1.0 --depth 1 https://github.com/microsoft/TokenEconomizer.git
claude plugin marketplace add "$HOME/.local/share/TokenEconomizer" --scope user
cd <project root>
claude plugin install tokeneconomizer@tokeneconomizer --scope project
TE="$HOME/.local/share/TokenEconomizer/plugin/scripts/te"
$TE --host claude config set microskills=on toolprune=propose
$TE --host claude toolprune refresh-catalog
```

**Never `toolprune=auto`**: it writes `permissions.deny` into
`.claude/settings.local.json`, which applies to the factory's subagents too,
and would remove CodeGraph, LLM Wiki, `LSP` and `Skill`: tools a chat session
rarely uses but the agents need.

## 4. `AGENTS.md`

Edit `CLAUDE.md` if it exists, else `AGENTS.md` (create `AGENTS.md` if
neither does). Every agent reads it. Make sure it says how to set up a fresh
checkout, how to start the dev server on a given port, the validation
commands (matching `factory/checks.sh`), the single-test command, the
formatter command, and the project's conventions. Then add these sections,
filling the `<...>`:

```markdown
## Agent tooling

- **<lsp plugin>**: diagnostics and go-to-definition (`LSP` tool).
- **CodeGraph** MCP (`codegraph`): code graph for callers, impact, related
  tests and entry points. Graph-only, telemetry off, index in `~/.codegraph`;
  a reversible choice, see `docs/adr/0001-codegraph-graph-only.md`. Binary:
  `~/.codegraph/bin/<asset>` from the CodeGraph v0.21.0 release.
- **LLM Wiki** MCP (`llmwiki`) and `/wiki-ask`, `/wiki-research`,
  `/wiki-ingest`, `/wiki-save`, `/wiki-fix`: the knowledge base in `.wiki/`.
  Put sources in `.wiki/raw/` and run `/wiki-ingest`. Server:
  `~/.llmwiki/llmwiki-mcp.cjs` (microsoft/llmwiki, commit in `~/.llmwiki/VERSION`).
- **mattpocock-skills**: `/grill-with-docs`, `/to-spec`, `/to-tickets`,
  `/triage`, `/diagnosing-bugs`, `/improve-codebase-architecture`.

The factory agents use CodeGraph and the wiki read-only (they read the main
checkout, not the job worktree). Only humans write to the wiki.

## Agent skills

### Issue tracker

GitHub Issues of this repo, via the `gh` CLI. See `docs/agents/issue-tracker.md`.

### Triage labels

The five canonical roles; `ready-for-agent` is the `factory` label, and a
ready ticket with open blockers waits as `factory:blocked`. See
`docs/agents/triage-labels.md`.

### Domain docs

Single-context: `GLOSSARY.md` and `docs/adr/` at the repo root. See `docs/agents/domain.md`.

## Ponytail

The `ponytail` plugin is active in every session and, through its
`SubagentStart` hook, in every factory agent: write only what the task needs,
reuse what exists before adding code, never cut validation, error handling,
security or tests for risky logic.

- Precedence: the spec and the factory's file formats and one-line replies
  come first, then this file's conventions, then ponytail.
- `// shortcut: <what and when to revisit>` marks a deliberate shortcut;
  `/ponytail-debt` lists them.

## TokenEconomizer

Learns from this repo's sessions to cut cost. Settings are per machine
(`te` = `~/.local/share/TokenEconomizer/plugin/scripts/te --host claude`):
`te config set microskills=on toolprune=propose`, and
`te toolprune refresh-catalog` after Claude Code updates. **Never set
`toolprune=auto` here**: its deny list applies to the factory's subagents and
would remove tools they need. Apply a proposal only for tools no agent in
`.claude/agents/` lists. The builder reads learned facts with `te microskills`;
the loop guard skips subagents by design.

## Software factory

`/factory run` builds every open GitHub issue labeled `factory` through
intake → spec → build → gate → review → approve, one PR per issue. State is in
`factory/jobs/` (gitignored); `factory/bin/job` is its only writer. Checks:
`factory/checks.sh`. Issues with open blockers wait as `factory:blocked`;
`factory/bin/unblock` (run by `/factory run` and the `factory-unblock`
workflow) moves them to `factory` once every blocker is closed as completed.
Dashboard: `cd factory && python3 -m http.server`, then open
http://localhost:8000/dashboard.html. Workflow guide: `factory-docs/01-workflow.md`.

Each job works in its own git worktree at `.worktrees/<job-id>` (gitignored).
A tool run from the repo root that doesn't read `.gitignore` sees those copies
too; exclude `.worktrees/` in its config.
```

## 5. Verify

1. `bash factory/checks.sh` exits 0 on the current branch. If it fails, the
   factory can never pass a job: fix the script, or tell the user what in the
   project is broken (don't fix project code without asking **[USER]**).
2. `factory/bin/job queue` and `factory/bin/unblock --dry-run` run without an
   error.
3. `grep -rn "ADAPT" .claude/agents` prints nothing. Each agent's frontmatter
   has only `name`, `description`, `tools`, `model` (and `hooks` on the
   builder).
4. `python3 -m json.tool factory/config.json` and `.mcp.json` succeed.
5. `git check-ignore factory/jobs/x .worktrees/x .claude/settings.local.json`
   prints all three; `git check-ignore factory/bin/job` prints nothing; `git
   status --porcelain --untracked-files=all` lists every copied file.
6. A headless check, from the project root (MCP servers and plugins load only
   in a new session):

   ```bash
   claude -p 'Tooling check. (1) Call mcp__llmwiki__wiki_status and mcp__codegraph__codegraph_symbol_search with any class name from this repo; one line each. (2) Use the Agent tool with subagent_type "reviewer-code" and prompt "Tooling test only, not a factory job: write no files. Say whether a ponytail ruleset is in your context (quote its first line) and whether you can see the skills ponytail:ponytail-review and mattpocock-skills:codebase-design." Report its reply.' \
     --allowedTools 'Agent,ToolSearch,mcp__llmwiki__*,mcp__codegraph__*' < /dev/null
   ```

   Both MCP calls answer, and the subagent reports `PONYTAIL MODE ACTIVE` and
   sees both skills.
7. `"$HOME/.local/share/TokenEconomizer/plugin/scripts/te" --host claude status`
   runs, and `~/.tokeneconomizer/errors.log` is empty or absent.

## 6. Hand over

Stay on the branch the user is on. Commit the install there with a one-line
message and show the user the diff. **Ask before pushing [USER]**: the factory
branches every job from `origin/<base_branch>`, and the `factory-unblock`
workflow runs only once it is on GitHub (on a private repo it uses a few
seconds of Actions minutes per closed issue).

Then tell the user, in a few lines:

- Restart Claude Code so the plugins and MCP servers load.
- How to work: `factory-docs/01-workflow.md` (`/grill-with-docs` →
  `/to-spec` or `/to-tickets` → `/factory run`).
- Every job ends as a PR. The approver merges what it approves; the rest wait
  labeled `factory:needs-human`.
- Intake runs on Fable, once per issue; depending on the plan, Fable can bill
  to usage credits.
- To start carefully, set `concurrency` to 1 in `factory/config.json`.
