# livespec-mcp

**Code intelligence for AI agents** — call graph, impact analysis, and
bidirectional **Functional Requirement ↔ code** traceability. Local-first,
zero external services, runs as an MCP server next to your editor.

Battle-tested on real codebases. v0.9 numbers vs the same Django 5.1.4
queries one release earlier:

| Tool on Django (40K symbols) | v0.8 | v0.9 |
|---|---:|---:|
| `find_dead_code` candidates | 824 | **514** (−38% noise) |
| `find_endpoints(framework='django')` | 20 | **162** (+8×) |
| `quick_orient` p95 | ~60 ms | ~60 ms |
| Partial reindex (touch 1 file) | 25 ms | **12 ms** (−51%) |

> Want the agentic flow without reading further?
> See [`docs/AGENT_QUICKSTART.md`](docs/AGENT_QUICKSTART.md) for the
> 10-step brownfield onboarding flow that 4 sessions of real-agent
> battle-testing converged on.

## 30-second tour

```bash
# Wire as an MCP server next to your editor (claude.ai/code, Cursor, ...).
# Pointed at any local repo via LIVESPEC_WORKSPACE.
livespec-mcp
```

```jsonc
// First call: cold-index the workspace once
> index_project()
{ "files_total": 2898, "symbols_total": 39789, "edges_total": 465179,
  "languages": {"python": 2786, "javascript": 112}, "watcher_started": false }

// Composite first-contact on an unfamiliar symbol
> quick_orient(qname="django.contrib.auth.middleware.AuthenticationMiddleware.process_request")
{ "kind": "method", "is_entry_point": false,
  "callers_count": 56, "callees_count": 3,
  "top_callees": [
    {"qualified_name": "django.contrib.auth.middleware.get_user", "pagerank": 0.000024},
    {"qualified_name": "django.utils.functional.SimpleLazyObject", "pagerank": 0.000209},
    {"qualified_name": "django.core.exceptions.ImproperlyConfigured", "pagerank": 0.000872}
  ],
  "requirements": [] }

// Wider blast radius on an RF-active codebase
> analyze_impact(target_type="requirement", target="RF-042")
{ "rf_id": "RF-042", "implementing_symbols": [...],
  "dependent_requirements": ["RF-088", "RF-091"],
  "impacted_callers": [...] }
```

Built for the questions an agent asks on an unfamiliar codebase:

- ¿Qué código implementa el RF-042?
- Si modifico `auth.verify`, ¿qué RFs y qué llamadores se ven afectados?
- ¿Qué módulos no tienen ningún RF asociado?
- ¿Qué RFs dependen de RF-042 transitivamente?

RF traceability is the differentiator. Most code-intel tools stop at "what
calls this function?". livespec layers Functional Requirement ↔ code links
on top so an agent on a serious-software-shop codebase can answer
*"changing this function affects RF-042, RF-088 and 3 dependent RFs"* in
one round-trip. RF agentic tools ship in the default surface;
RF mutation/management tools live in the optional `livespec-rf` plugin
that **auto-loads when the workspace already has RFs** — fresh repos get
a 16-tool surface, RF-active repos get 27, with no config.

### What "living" actually means here

| Layer | Lives | How |
|---|---|---|
| Symbol index | ✅ | xxh3 content-hash incremental, run `index_project` on demand |
| Call graph + edges | ✅ | re-resolved on every change; persistent `symbol_ref` |
| RF ↔ code links | ✅ | auto-scan of `@rf:` annotations after every `index_project` |
| RF ↔ RF graph | ✅ | explicit, cycle-checked; `link_rf_dependency` (plugin) |
| Drift detection | ✅ | body_hash + signature_hash on every symbol; `list_docs(only_stale=True)` (plugin) |
| **Generated docs content** | ❌ on-demand | `generate_docs` (plugin) needs an LLM-capable caller or an MCP host that supports sampling. Drift is *detected*, not *fixed*. |

So: traceability is live, docs are not. If your workflow is "give me an
agent that always knows which code implements which requirement, and which
tests probably break when X changes" — this is exactly what the project is
good at. If you wanted "writes my doc comments while I sleep" — not yet.

## Stack

- **FastMCP 2.14** (stdio transport)
- **SQLite** (single `docs.db` file, ACID, WAL, explicit migration framework)
- **tree-sitter + tree-sitter-language-pack** for multi-language parsing
- **Python `ast`** for high-precision Python extraction
- **NetworkX** for call graph and topological impact analysis (cached per
  index run)

100% local, zero external services, zero API keys required.

## Language support

Honest table — only languages with a passing test suite are claimed.

| Language | Status | What's covered |
|----------|--------|----------------|
| **Python** | ✅ Tested | Functions, classes, methods, decorators, calls — uses `ast` for full precision. Imports drive scoped resolution (P0.4). |
| **Go** | ✅ Tested | Functions, struct types via `type_spec`, struct methods, calls. **Scoped resolution** via `import` + alias (P1.A2 v0.4). |
| **Java** | ✅ Tested | Classes, methods, calls (`method_invocation`) |
| **JavaScript** | ✅ Tested | Function declarations, **arrow functions** assigned to const/let, classes, methods. **Scoped resolution** via ES6 `import` and CommonJS `require` (P1.A1 v0.4). |
| **TypeScript** | ✅ Tested | Same as JS plus typed signatures (`.ts` and `.tsx`). **Scoped resolution** via ES6 `import` (P1.A1 v0.4). |
| **Rust** | ✅ Tested | Free functions, struct/enum types, **`impl` block methods** as `Type::method`, traits. **Scoped resolution** via `use` declarations (P4.A3 v0.5). |
| **Ruby** | ✅ Tested | `def`, `class`, `module`, `singleton_method`, calls. Best-effort scoped resolution via `require_relative` + receiver field (P1.A4 v0.4). |
| **PHP** | ✅ Tested | Classes, methods, function/method/scoped call expressions. Best-effort scoped resolution via `use Namespace\X` for `Class::method()` (P1.A4 v0.4); instance-method calls are not scoped. |
| C, C++, C#, Kotlin, Swift, Scala | ⚠️ Untested | The generic tree-sitter extractor will *attempt* to parse these (they're listed in `EXT_LANGUAGE`) but no test suite covers them. Symbol coverage may be partial — open an issue with a fixture if you need a specific language hardened. |

The extractor is a heuristic over hardcoded tree-sitter node types
(`_DEF_NODE_TYPES`, `_CALL_NODE_TYPES` in `extractors.py`); it intentionally
trades completeness for simplicity. Use the per-language tests in
`tests/test_extractors.py` as the contract.

## Install

```bash
uv venv --python 3.12
uv pip install -e ".[dev]"
```

## Run as MCP server

```bash
livespec-mcp
```

By default it picks the **current working directory** as workspace, or
`LIVESPEC_WORKSPACE` if set. Persistent state lives in `.mcp-docs/docs.db`.

### Claude Code / Cursor wiring

```json
{
  "mcpServers": {
    "livespec": {
      "command": "uv",
      "args": ["--directory", "/path/to/livespec-mcp", "run", "livespec-mcp"],
      "env": {
        "LIVESPEC_WORKSPACE": "/path/to/your/project",
        "LIVESPEC_PLUGINS": "all"
      }
    }
  }
}
```

`LIVESPEC_PLUGINS=all` opts every plugin in regardless of DB state — useful
when bootstrapping RFs on a fresh repo. Default behavior: plugins
auto-load when their tables already have rows.

## Tools (16 default + 14 plugin = 30 max)

Every tool accepts an optional `workspace: str` argument. When omitted, the
server resolves to `LIVESPEC_WORKSPACE` env var or the current working
directory. The runtime caches one DB connection per workspace (LRU=8), so a
single MCP server instance can serve multiple repos in parallel.

### Default surface — code intel + RF agentic (16)

These tools answer the questions an agent ASKS on an unfamiliar codebase.
Always registered.

#### Indexing (1)
- `index_project(force=False, watch=False)` — walk, parse, persist. Read
  the `project://index/status` resource for current status (the legacy
  `get_index_status` tool was dropped in v0.9).

#### Code intelligence (12)
- `find_symbol(query, kind, limit)` — separator-agnostic name lookup.
- `get_symbol_source(qname)` — body slice only (lighter than full info).
- `who_calls(qname, max_depth=1)` — backward cone, slim agentic alias.
- `who_does_this_call(qname, max_depth=1)` — forward-direction counterpart.
- `quick_orient(qname)` — composite snapshot: metadata + docstring lead +
  top-5 callers/callees by PageRank + linked RFs + entry-point flag.
  Replaces 3-4 calls with one when an agent first lands on a symbol.
- `analyze_impact(target_type, target, max_depth)` — symbol/file/RF blast
  radius. `max_depth=1` covers the old "find references" use case.
- `get_project_overview(include_infrastructure=False)` — top symbols by
  PageRank; infra noise filtered by default.
- `git_diff_impact(base_ref='HEAD~1', head_ref='HEAD', max_depth=5)` —
  changed files → impacted callers → affected RFs → suggested test files.
  PR-review entry point.
- `find_dead_code(include_infrastructure=False)` — symbols with zero
  callers and zero RF links. Skips entry-point paths, framework
  decorators, `__main__` guards, list-stored callbacks.
- `find_orphan_tests(max_depth=10)` — test functions whose forward cone
  never reaches a non-test symbol.
- `find_endpoints(framework=None)` — symbols decorated with framework
  entry-point markers. `framework` ∈ {flask, fastapi, click, pytest,
  fastmcp, celery, django, None}.
- `audit_coverage()` — RF coverage report: modules without direct RF,
  modules implicitly covered (transitively reached), modules truly orphan,
  RFs without implementation, RFs with low avg confidence.

#### RF agentic — query, don't mutate (3)
- `list_requirements(status, module, priority, has_implementation)` —
  RF discovery surface.
- `get_requirement_implementation(rf_id)` — answers
  *"¿qué código implementa el RF-042?"*.
- `propose_requirements_from_codebase(module_depth=2, min_symbols_per_group=3,
  max_proposals=30, skip_already_covered=True)` — heuristic RF discovery
  on an RF-empty repo. Groups symbols by module + PageRank, proposes
  RF candidates with humanized title + suggested_symbols.

### `livespec-rf` plugin — RF mutation (11)

Auto-loads when the workspace DB has rf rows, or when `LIVESPEC_PLUGINS`
includes `rf`. Tools an *operator* runs to mutate RF state.

- `create_requirement(title, ...)`, `update_requirement(rf_id, ...)`,
  `delete_requirement(rf_id)` — cascade-removes rf_symbol links.
- `link_rf_symbol(rf_id, symbol_qname, relation, confidence, source, unlink)` —
  link / unlink a single RF↔symbol pair.
- `bulk_link_rf_symbols(mappings)` — batch-link N pairs in one transaction.
- `link_rf_dependency(parent_rf_id, child_rf_id, kind='requires')` /
  `unlink_rf_dependency` / `get_rf_dependency_graph` — RF→RF graph.
  `kind` ∈ {requires, extends, conflicts}; cycles rejected at insert time.
- `scan_rf_annotations()` — two-level matcher (`@rf:RF-NNN` vs.
  verb-anchored); auto-runs after every `index_project`.
- `scan_docstrings_for_rf_hints()` — surfaces RF candidates from existing
  docstrings (first sentence, leading verb). Returns
  `verb_histogram_top` for noticing dominant action verbs.
- `import_requirements_from_markdown(path)` — bulk-create RFs from
  `## RF-NNN: Title` Markdown specs. Idempotent.

### `livespec-docs` plugin — doc generation (3)

Auto-loads when the workspace DB has doc rows, or when `LIVESPEC_PLUGINS`
includes `docs`. Human-tier ceremony for managing generated docs.

- `generate_docs(target_type, identifier, content?, max_tokens?)` —
  three modes: caller_supplied / sampling / needs_caller_content. Works
  in Claude Code (caller mode) and Cursor/Desktop (sampling mode).
- `list_docs(target_type, only_stale=False)` — list or surface drifted
  docs (drift triggers on body_hash OR signature_hash mismatch).
- `export_documentation(format, out_subdir)` — markdown or JSON.

### Migrating from older versions

| Removed | Use instead |
|---|---|
| `find_references` (v0.1) | `analyze_impact(target_type='symbol', target=qname, max_depth=1)` |
| `get_symbol_info` (v0.7) | `quick_orient` (composite) + `get_symbol_source` (body) |
| `get_call_graph` (v0.7) | `who_calls` + `who_does_this_call` |
| `search`, `rebuild_chunks` (v0.7) | `find_symbol` + `quick_orient`; FTS surface dropped due to zero agent uptake |
| `list_files` (v0.7) | grep / ripgrep host with path glob |
| `start_watcher` / `stop_watcher` / `watcher_status` (v0.7) | re-run `index_project` on demand (watcher race-condition trap for editing agents) |
| `link_requirement_to_code` (v0.6 alias) | `link_rf_symbol` |
| `link_requirements` / `unlink_requirements` (v0.6 alias) | `link_rf_dependency` / `unlink_rf_dependency` |
| `get_requirement_dependencies` (v0.6 alias) | `get_rf_dependency_graph` |
| `get_index_status` (v0.9, deprecated in v0.8) | read the `project://index/status` resource |

## Resources

- `project://overview`
- `project://index/status`
- `project://requirements`
- `project://requirements/{rf_id}`
- `project://files/{path*}`
- `project://symbols/{qname*}`
- `doc://symbol/{qname*}`
- `doc://requirement/{rf_id}`

## Prompts (slash commands)

- `onboard_project`
- `analyze_change_impact`
- `audit_requirement_coverage`
- `extract_requirements_from_module`
- `document_undocumented_symbols`
- `refresh_stale_docs`
- `explain_symbol`

## Performance

Numbers from the v0.8/v0.9 battle-test harness (4 sessions / 4 profiles
/ 65+ logged calls in [`docs/AGENT_USAGE_DATA.md`](docs/AGENT_USAGE_DATA.md)).
Cold = first run; warm = cached run on the same workspace. Latency p95
measured with the in-process middleware
(`src/livespec_mcp/instrumentation.py`).

| Repo | Files / Symbols | `index_project` cold | `quick_orient` p95 | `get_project_overview` p95 |
|---|---:|---:|---:|---:|
| url-shortener-demo (Python) | 4 / 23 | ~50 ms | <5 ms | ~10 ms |
| livespec-mcp itself (Python+8 langs) | 84 / 495 | ~400 ms | ~60 ms | ~75 ms |
| jig (Python) | 130 / 1173 | ~600 ms | ~50 ms | ~80 ms |
| Django (Python, stress) | 2898 / 39789 | ~25 s | <100 ms | ~250 ms |
| warp subset (Rust, stress) | 5K / 50K | ~30 s | <100 ms | ~300 ms |

For repos > 30K symbols, pass `summary_only=True` on aggregator and
traversal tools (`audit_coverage`, `find_dead_code`, `find_orphan_tests`,
`find_endpoints`, `git_diff_impact`, `who_calls`, `who_does_this_call`,
`analyze_impact`) to keep payloads under ~200 KB. Counts stay exact
regardless of pagination — see `bench/run.py --large` for the Django
stress profile.

### v0.9 Django wins (same queries, one release apart)

| Tool | v0.8 | v0.9 | Why |
|---|---:|---:|---|
| `find_dead_code` | 824 candidates | **514** | non-Python skip + dotted-path string refs + Django `Meta` inner classes |
| `find_dead_code` functions | 450 | 189 | most of the drop comes from xregexp.js skipped + template tag / hasher protection |
| `find_endpoints(django)` | 20 (decorators only) | **162** | class-based view detection via inheritance from `View` / `LoginRequiredMixin` / etc. |
| Partial reindex on `requests` | 25 ms | **12 ms** | targeted `_resolve_refs` walk |

## Tests

```bash
uv run pytest -q
```

In-memory FastMCP `Client(mcp)` so tests run without subprocess or network.

## Agent vs human user

livespec ships two user shapes deliberately:

- **Agents** see the 16-tool default surface and the agentic-read RF tools
  (`list_requirements`, `get_requirement_implementation`,
  `propose_requirements_from_codebase`, `audit_coverage`). The composite
  `quick_orient` is the canonical first-contact tool — it returns
  metadata, docstring lead, top callers/callees by PageRank, linked RFs,
  and entry-point flags in one call.
- **Humans** (or operator scripts) reach for the plugin tools to mutate
  RF state and manage docs. Auto-load happens once the DB shows real RF
  or doc rows; before that, set `LIVESPEC_PLUGINS=all` (or `=rf` /
  `=docs`) to bootstrap.

This is why dropping `search`/`get_symbol_info` was safe: the battle-test
harness logged zero agent calls to those tools across 3 codebases. The
data trumped the prior intuition.

## Roadmap

| Fase | Estado | Contenido |
|------|--------|-----------|
| 0 — Bootstrap | ✅ | FastMCP server, project structure |
| 1 — Indexing | ✅ | tree-sitter + Python AST, file-incremental, call graph |
| 2 — Analysis | ✅ | NetworkX, impact, PageRank |
| 3 — Requirements | ✅ | CRUD + linking + annotation matcher |
| 4 — RAG/Embeddings | ✅ | AST chunking, FTS5, fastembed + sqlite-vec optional, RRF |
| 5 — Doc generation | ✅ | `generate_docs` (dual-mode), drift detect (body+signature), export |
| 6 — Polish | ✅ | 7 prompts, doc:// resources, two-level @rf: matcher with negation guard |
| 7 — v0.2 | ✅ | Multi-tenant state, tool consolidation 25→23, persistent refs, watcher, bench suite |
| 8 — v0.3 | ✅ | Auto-scan post-index, PageRank infra filter, scoped resolution by imports, `git_diff_impact`, embeddings smoke real, Ruby+PHP fixtures, hypothesis property tests, memory bench, GitHub Actions CI, `code://` resource, `delete_requirement`, markdown RF importer |
| 9 — v0.4 | ✅ | Scoped resolution for TS/JS/Go/Ruby/PHP, `find_dead_code` / `audit_coverage` / `find_orphan_tests`, `did_you_mean` on Symbol-not-found errors, watcher `atexit` cleanup, CI venv fix |
| 10 — v0.5 | ✅ | Bug fixes from real-repo demo, decorators as first-class field + `find_endpoints`, RF dependency graph (requires/extends/conflicts) with `analyze_impact` cascade, matcher multi-RF + confidence override + `@not_rf:` negation + golden dataset, Rust `use` scoped resolution |
| 11 — v0.6 | ✅ | Hardening: explicit migration framework, unified error shape, RF link tools renamed, deprecated `use_workspace` removed, Django stress test (40K symbols), graph cache, README pitch reframe |
| 12 — v0.7 | ✅ | Brownfield onboarding: `propose_requirements_from_codebase`, `bulk_link_rf_symbols`, `scan_docstrings_for_rf_hints`. Pagination on aggregator tools. Rust `pub` visibility-aware dead-code filter. `find_symbol` separator-agnostic |
| 13 — v0.8 | ✅ | Curation pass driven by 3-session battle-test data: 4 quick-win agentic tools (`quick_orient`, `who_calls`, `who_does_this_call`, `get_symbol_source`). 11 P2 bug fixes on `find_dead_code`, `audit_coverage`, `git_diff_impact`, `propose_requirements_from_codebase`. Plugin auto-detect framework — RF mutation (11 tools) and doc management (3 tools) move into auto-loading plugins. Tier-4 drops: `list_files`, `search`, `rebuild_chunks`, `get_call_graph`, `get_symbol_info`, watcher trio. Default surface 39 → 17 tools |
| 14 — v0.9 | ✅ | Django readiness: targeted `_resolve_refs` walk on partial reindex (closes v0.7 deferred). Pagination on `who_calls` / `who_does_this_call` / `analyze_impact`. `min_weight=0.6` filter mutes resolver fan-out. Django dead-code accuracy (skip non-Python, recognize dotted-path strings + `class Meta:`). Django CBV detection in `find_endpoints` (LoginView/FormView/LoginRequiredMixin/etc.). Drop `get_index_status`. Default surface 17 → 16. Wire-validated: Django `find_dead_code` 824 → 514, `find_endpoints(django)` 20 → 162 |
