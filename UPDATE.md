# Update the factory kit

You are a coding agent updating the software factory installed in this
project from this kit. Apply what changed in the kit since the installed
version, keep everything the project made its own, verify, and report. Work
in the project's root and reply in the user's language. Ask the user before
anything marked **[USER]** and before pushing.

## 1. Versions

- Installed: `cat factory/KIT_VERSION`. If the file is missing, the factory
  was installed before versioning: ask the user which version it matches
  **[USER]**, defaulting to `v1.0.0`.
- Latest: `gh release list -R Add0z/factory-kit` or
  `git ls-remote --tags https://github.com/Add0z/factory-kit` (highest
  `vX.Y.Z`). If it equals the installed one, say so and stop.
- Read `https://raw.githubusercontent.com/Add0z/factory-kit/<latest>/CHANGELOG.md`
  for every version after the installed one, and tell the user in a few lines
  what the update brings. Go on only after a yes **[USER]**.

## 2. What changed

```bash
OLD=<installed>; NEW=<latest>
curl -fsSL "https://github.com/Add0z/factory-kit/compare/$OLD...$NEW.diff" -o /tmp/kit-update.diff
grep '^diff --git' /tmp/kit-update.diff
```

Files under `kit/` map to the same path in the project (`kit/factory/bin/job`
→ `factory/bin/job`). Then handle each changed file:

- **Unchanged in the project** (it equals the kit's `$OLD` version:
  `curl -fsSL https://raw.githubusercontent.com/Add0z/factory-kit/$OLD/kit/<path> | diff -q - <path>`):
  replace it with the `$NEW` version.
- **Customized in the project** (agents with project text in place of the
  `ADAPT` comments, `checks.sh`, `config.json`, docs): apply only the kit's
  change. Download both kit versions and patch:
  `diff -u old new | patch --dry-run <path>`, then without `--dry-run` if it
  applies cleanly. If a hunk fails, make the same change by hand, keeping the
  project's text. Never put an `ADAPT` comment back in a file that had it
  replaced.
- **`factory/config.json`**: add new keys; take a new default only where the
  project still has the old one; for `protected_paths`, add the kit's new
  entries and keep the project's own.
- **New in the kit**: copy it. **Deleted in the kit**: delete it if the
  project's copy is unchanged; otherwise ask **[USER]**.
- **`INSTALL.md`**: a new or changed setup step (a new tool, a new label, a
  new section in `AGENTS.md`) is part of the update: do it now, as
  `INSTALL.md` at `$NEW` says.
- **`stacks/<stack>.md`**, if the project applied that stack (its gates are
  in the build file): apply the changes that concern the factory; changes to
  project code (build file, config, tests) only after a yes **[USER]**.
- `README.md`, `NOTICE.md`, `CHANGELOG.md`, `UPDATE.md` itself: nothing to do.

Leave the project's own work alone: issues, `factory/jobs/`, `.wiki/`
pages, `GLOSSARY.md`, `docs/adr/` beyond the kit's file.

## 3. Verify and hand over

1. Run `INSTALL.md` step 5 at `$NEW` (Verify), all of it.
2. `echo $NEW > factory/KIT_VERSION`.
3. Commit on the current branch with the subject
   `Update factory kit to $NEW`, show the user the diff, and ask before
   pushing **[USER]**.
4. Tell the user what changed, any hunk you merged by hand, and to restart
   Claude Code if a plugin, MCP server or agent changed.
