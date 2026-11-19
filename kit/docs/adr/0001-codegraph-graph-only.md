# 0001. CodeGraph runs graph-only (no embeddings, no memory layer)

- Status: accepted
- Date: 2026-11-18
- Revisit when: the codebase has a few dozen domain classes, or agents keep
  failing to find code because they don't know the symbol names.

## Context

CodeGraph (MCP server `codegraph` in `.mcp.json`) can run in two modes:

- **Full**: builds the code graph and also embeds every symbol with an ONNX
  model (`bge-small` by default). That adds natural-language semantic search
  ("where do we compute the budget balance?"), similar/duplicate code
  detection, and the memory layer (`codegraph_memory_*` tools: notes attached
  to files and functions, stored in `~/.codegraph`).
- **Graph-only** (`--graph-only`): only the structural graph. No model
  download, no ~1.5 GB RAM requirement, 10-50x faster indexing.

## Decision

Run CodeGraph with `--graph-only`.

- The repo is tiny; searching by symbol name finds everything, so semantic
  search adds cost without value yet.
- The structural tools the factory agents rely on all work in graph-only mode:
  `codegraph_symbol_search` (by name), `codegraph_get_callers` /
  `codegraph_get_callees`, `codegraph_analyze_impact`,
  `codegraph_find_related_tests`, `codegraph_find_entry_points`,
  `codegraph_traverse_graph`, dependency and module tools.
- The memory layer would duplicate the LLM Wiki (`.wiki/`). Wiki pages are in
  git and shared; CodeGraph memories live only on one machine. The wiki is the
  project's single knowledge base. **Keep the memory tools off even if
  embeddings are enabled later.**

## Consequences

- No natural-language code search and no duplicate-code detection.
- Agents must know (or grep for) a name to find code.

## How to revert (enable semantic search)

1. In `.mcp.json`, remove `"--graph-only"` from the `codegraph` args. Optionally
   pick a model with `"--embedding-model", "bge-small"` (default) or
   `"static"` (much faster; needs a model in
   `~/.codegraph/static_models/jina-code-static-256`, see the CodeGraph README).
2. Restart Claude Code. The first index embeds in the background; name search
   keeps working meanwhile. Needs ~1.5 GB free RAM for ONNX models.
3. Check which tools gained semantic matching (`codegraph_symbol_search`
   becomes hybrid BM25 + semantic) and add any new search tool the agents
   should use to the `tools:` lines in `.claude/agents/*.md`. Do **not** add
   `codegraph_memory_*` (see Decision).
4. Update this ADR's status to "superseded by 000N" and write the new ADR.
