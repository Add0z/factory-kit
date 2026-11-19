# Changelog

## v1.1.0

- `factory/KIT_VERSION` records the installed version; `UPDATE.md` updates a
  project from it.
- `protected_paths` now also covers the factory's own rules (`factory/bin/*`,
  `factory/KIT_VERSION`, `.claude/agents/*`, `.claude/skills/factory/*`, the
  unblock workflow): a job can no longer change them. Humans still can.

## v1.0.1

- INSTALL.md: baseline build before adding gates, dev-server check, wider
  clash check, copy-pasteable commands (literal file list, subshells,
  idempotent labels, detected CodeGraph asset), version checks, the
  TokenEconomizer per-machine and cost notes.
- `stacks/java-spring.md`: no duplicate asciidoctor block, merge YAML under
  `spring:`, which gate catches what, Boot 3 Flyway, known Initializr
  problems (Redis chat memory, metrics registries), Gradle excludes.

## v1.0.0

First release.
