# Triage Labels

The skills speak in terms of five canonical triage roles. This file maps those roles to the actual label strings used in this repo's issue tracker.

| Label in mattpocock/skills | Label in our tracker | Meaning                                  |
| -------------------------- | -------------------- | ---------------------------------------- |
| `needs-triage`             | `needs-triage`       | Maintainer needs to evaluate this issue  |
| `needs-info`               | `needs-info`         | Waiting on reporter for more information |
| `ready-for-agent`          | `factory`            | Fully specified, ready for an AFK agent  |
| `ready-for-human`          | `ready-for-human`    | Requires human implementation            |
| `wontfix`                  | `wontfix`            | Will not be actioned                     |

When a skill mentions a role (e.g. "apply the AFK-ready triage label"), use the corresponding label string from this table.

`factory` is the software factory's queue label: an issue labelled `factory` is built by `/factory run` (see `AGENTS.md`). The labels `factory:needs-info` and `factory:needs-human` belong to the factory; skills must not add or remove them.

A ticket that is `ready-for-agent` but still has an open blocker gets `factory:blocked` instead of `factory` (see `docs/agents/issue-tracker.md`); `factory/bin/unblock` promotes it when its blockers are done.

Edit the "Label in our tracker" column to match whatever vocabulary you actually use.
