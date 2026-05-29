# DCC MCP Framework — Bootstrap SPEC

> **Portable SPEC.** This document is self-contained and assumes no local filesystem state. It can be executed locally or in Claude Code on the web. All inputs are GitHub repos under `github.com/kleer001/`.

## Working environment & handoff

- **Target repo to build**: `github.com/kleer001/dcc_mcp_framework` (this repo). Branch + PR against it.
- **Reference repos** (clone read-only to extract templates; do not modify):
  - `github.com/kleer001/nuke-mcp` — most polished; primary template source.
  - `github.com/kleer001/blender-mcp` — clean `tools/_helpers.py`, queue+timer addon.
  - `github.com/kleer001/houdini-mcp` — headless launch, RAG, events (mcp[cli] idiom).
  - `github.com/kleer001/natron-mcp` — minimal; the `{id,method}` envelope anti-pattern to standardize away.
- **On the web**: `git clone` the four reference repos into a scratch dir, read source, build the framework. All verification is mock-mode only (no DCC binaries required).
- **All file paths below are relative to each repo's root** after cloning (e.g. `nuke-mcp/src/nukemcp/connection.py`).

## Context

`dcc_mcp_framework/` is an empty scaffolded repo. The goal is to turn it into a **mother-repo framework** whose deliverable is **a series of Claude skills** that let authors create new MCP (Model Context Protocol) server repos for DCC (Digital Content Creation) tools, or improve existing ones.

Four production DCC MCP repos already exist (`blender-mcp`, `houdini-mcp`, `natron-mcp`, `nuke-mcp`). Deep exploration shows they independently converged on **one shared architecture** but drifted on accidental details (e.g. Natron used `{"id","method"}` envelopes while the others use `{"type","params"}`). The framework's job is to capture the convergent core as reusable, deterministic templates + a single-source-of-truth blueprint, exposed through skills — so the next DCC MCP (Maya, C4D, Krita, Resolve, …) is built right in one shot instead of rediscovered.

### The convergent architecture (the thing being captured)
Two-process design:
```
Claude  --stdio JSON-RPC-->  MCP server (src/<dcc>mcp/, FastMCP)
        --TCP newline-JSON-->  in-DCC addon (TCP socket server)
        --calls-->            DCC Python API (bpy / hou / nuke / NatronEngine)
```
- **Envelope** (newline-delimited JSON): request `{"type":"<cmd>","params":{...}}\n`; response `{"status":"success"|"error","result":{...},"error":"..."}\n`.
- **Thread-safety**: DCC APIs are NOT thread-safe. Socket runs on a background thread; commands are marshalled to the DCC main thread via a queue + main-thread timer (`bpy.app.timers` / `QtCore.QTimer` / `nuke.executeInMainThread`) with a `threading.Event` to return the result. Headless mode drains the queue via `serve_forever()` on the main thread.
- **MCP side**: a `<Dcc>Connection` class — connect with exponential-backoff retry (3 attempts), read handshake (version/variant), newline framing, a reader thread separating pushed events from responses, reconnect-on-broken-pipe, per-command timeout. (Verbatim shape in `nuke-mcp/src/nukemcp/connection.py`.)
- **Tools**: modular — each domain is `tools/<domain>.py` exposing `def register(server)` with `@mcp.tool()` defs; `tools/_helpers.py` provides `send()` and `require_confirm()`. Destructive ops gated by `confirm=True` + a `destructiveHint` annotation.
- **Mock mode**: a mock connection / socket server keeps minimal in-memory state so the MCP server and tests run with no DCC installed (`--mock`). Every tool must have a mock handler (mock parity).
- **Tests**: `conftest.py` mock-server fixture; `test_connection.py` for the protocol; integration tests marked and deselected by default.

## Decisions (locked)
- **Distribution**: Claude Code **plugin** (`.claude-plugin/plugin.json`) bundling skills + reference + templates.
- **Template strategy**: **Hybrid** — ship copy-able token-substituted templates for the subtle near-identical core (connection, addon thread-marshalling, mock, conftest, pyproject, `.mcp.json`); have skills *generate* the thin DCC-API-specific tool/handler/mock modules from one canonical exemplar.
- **Skill set**: **4 skills + `reference/` docs** (reference is docs, not a triggerable skill).
- **SDK**: support **both** `fastmcp` (standalone v2) and `mcp[cli]` (official SDK). SDK-coupled template files ship in two variants; the scaffold skill branches on the author's choice. SDK-agnostic files (connection, addon, mock, conftest, helpers) are single-source.

## Target repo structure
```
dcc_mcp_framework/
├── .claude-plugin/plugin.json        # declares the 4 skills; plugin name/version/description
├── CLAUDE.md                         # rewrite stub: what this repo is (a skills+templates plugin)
├── README.md                         # how to install the plugin + use the skills
├── LICENSE                           # keep existing MIT
│
├── skills/
│   ├── scaffold-dcc-mcp/SKILL.md
│   ├── add-tool-domain/SKILL.md
│   ├── add-advanced-feature/SKILL.md
│   └── audit-dcc-mcp/SKILL.md
│
├── reference/                        # single source of truth (prose) — skills cite by relative path
│   ├── architecture.md               # two-process diagram, component responsibilities, lifecycle
│   ├── protocol.md                   # envelope spec, framing, handshake, error contract; Natron anti-pattern
│   ├── threading.md                  # GUI marshalling table per DCC; headless serve_forever; mock dispatch
│   ├── gotchas.md                    # PySide6/2, lazy DCC import, SO_REUSEADDR, daemon threads, reconnect, mcp[cli]<->fastmcp port notes
│   ├── advanced-features.md          # one section per opt-in feature: what/which repos/module/wiring/test
│   └── audit-checklist.md            # the conformance checklist audit-dcc-mcp walks
│
└── templates/
    ├── PLACEHOLDERS.md               # documents every token (see below)
    ├── shared/                       # SDK-agnostic — one copy
    │   ├── src/{{dcc}}mcp/connection.py.tmpl     # VERBATIM core from nuke connection.py (rename class/port)
    │   ├── src/{{dcc}}mcp/mock.py.tmpl           # MockState skeleton + dispatch; handlers generated
    │   ├── src/{{dcc}}mcp/tools/__init__.py.tmpl
    │   ├── src/{{dcc}}mcp/tools/_helpers.py.tmpl # send(), require_confirm()
    │   ├── {{dcc}}_addon/{{dcc}}_mcp_addon.py.tmpl  # _serve/serve_forever/_run_in_main-thread + ping/get_scene_info handlers
    │   ├── mcp.json.tmpl
    │   ├── tests/conftest.py.tmpl                # mock-server + connection fixtures
    │   └── tests/test_connection.py.tmpl
    ├── fastmcp/                       # SDK variant A
    │   ├── pyproject.toml.tmpl                   # dep: fastmcp>=2.14,<3.0
    │   ├── src/{{dcc}}mcp/server.py.tmpl         # from fastmcp import FastMCP; build_server + register wiring + main()
    │   └── src/{{dcc}}mcp/tools/graph.py.tmpl    # exemplar w/ annotations={"readOnlyHint":...}
    ├── mcp_cli/                       # SDK variant B
    │   ├── pyproject.toml.tmpl                   # dep: mcp[cli]>=1.4
    │   ├── src/{{dcc}}mcp/server.py.tmpl         # from mcp.server.fastmcp import FastMCP; ...
    │   └── src/{{dcc}}mcp/tools/graph.py.tmpl    # exemplar in mcp[cli] idiom
    └── advanced/                      # copy-on-demand modules for add-advanced-feature (stdlib-only)
        ├── events.py.tmpl
        ├── rag.py.tmpl
        ├── memory.py.tmpl
        ├── discovery.py.tmpl          # headless auto-launch
        └── plugins.py.tmpl            # plugin auto-discovery
```

**Placeholder tokens** (`PLACEHOLDERS.md`): `{{DCC}}` PascalCase (Nuke), `{{dcc}}` package stem (nuke), `{{PORT}}` (default 54321), `{{API_MOD}}` (bpy/hou/nuke/NatronEngine), `{{GUI_MARSHAL}}` (the one expression to run a fn on the main thread per DCC), `{{SCENE_NOUN}}` (script/scene/comp), `{{NODE_NOUN}}` (node/object). Scaffold does literal string substitution — no LLM judgment on the hard files.

## The four skills (SKILL.md frontmatter + workflow)

### 1. `scaffold-dcc-mcp`
`description`: "Scaffold a new DCC MCP server repository from the framework templates. Use whenever the user wants to create/bootstrap a new MCP server for a DCC tool (Maya, Cinema4D, Krita, Resolve, etc.), says 'new dcc mcp' or 'scaffold an mcp for <tool>'. Do NOT use for improving an existing MCP repo (use audit-dcc-mcp) or adding tools to one (use add-tool-domain)."
Workflow:
1. Read `reference/architecture.md` + `reference/protocol.md` first.
2. Interview / resolve identity: DCC name → `{{DCC}}`/`{{dcc}}`; Python API module; default port; **SDK choice (fastmcp vs mcp[cli])**; GUI-thread marshalling mechanism (`reference/threading.md` table); scene/node nouns. Ask before proceeding if unknown.
3. Instantiate `templates/shared/` + the chosen SDK variant (`templates/fastmcp/` or `templates/mcp_cli/`) with substitutions; echo the substitution map.
4. Generate the starter `tools/graph.py` from the matching exemplar, adapting the core tool set (ping, get_scene_info, create/get/delete node, connect, set/get param, set_frame, render, execute_python) to the DCC API.
5. Generate matching mock + addon handlers for exactly those commands (mock parity).
6. Verify: `pip install -e ".[dev]" && pytest -m "not integration"` green + `ruff check` clean in mock mode. Fail loudly if red — never hand back a broken scaffold.
7. Write the new repo's `.scaffold.json` record + README; print install one-liner and `.mcp.json` location.

### 2. `add-tool-domain`
`description`: "Add a new tool domain module (lighting, rendering, tracking, etc.) to an existing DCC MCP server. Use when the user wants to add tools/commands/a domain to an existing `*-mcp` repo. Do NOT use when scaffolding a new repo."
Workflow: read `reference/protocol.md` + the repo's `tools/_helpers.py` and one existing domain to match conventions → define commands (read-only vs destructive) → generate `tools/<domain>.py` (`register(server)` + annotated tools; destructive ⇒ `confirm=True` via `require_confirm`) → add matching addon `_handle_<cmd>` and mock `_cmd_<cmd>` (enforce mock parity) → wire `import` + `<domain>.register(server)` into `server.py` → generate `tests/test_<domain>.py` using the `connection` fixture → run pytest, must pass.

### 3. `add-advanced-feature`
`description`: "Add an opt-in advanced capability to a DCC MCP server: bidirectional events, offline RAG doc search, persistent memory resources, headless auto-launch, version/variant gating, plugin auto-discovery, undo integration, or recipe tools. Use when the user names one of those for a `*-mcp` repo."
Workflow: read the relevant section of `reference/advanced-features.md` → copy the corresponding `templates/advanced/*.tmpl` module → wire its `register(server)` (or reader-thread/event hookup for events; addon-side push for events) into `server.py` → add deps if any (designed stdlib-only) → add a test → run pytest.

### 4. `audit-dcc-mcp`
`description`: "Audit and improve an existing DCC MCP server against the framework blueprint — protocol conformance, thread-safety, mock parity, confirm-guards on destructive ops, test coverage. Use when the user wants to review/improve/bring-into-line an existing MCP repo, or says 'audit my mcp'."
Workflow: read `reference/audit-checklist.md` → walk it read-only against the target (envelope shape, framing, reader thread, backoff connect, GUI marshalling + headless path, confirm-guards, annotations, mock parity, conftest fixture, integration tests deselected, packaging) → produce prioritized findings (severity + file:line) → offer fixes that reference the same `templates/` (e.g. replace a hand-rolled connection with `connection.py.tmpl`, or fix Natron-style `{id,method}` envelope).

## Source files to template (ground truth)
- `nuke-mcp/src/nukemcp/connection.py` → `templates/shared/.../connection.py.tmpl` (verbatim core; reader thread, framing, backoff already correct).
- `nuke-mcp/nuke_addon/nuke_mcp_addon.py` (the `_serve`/`serve_forever`/`_run_in_nuke` machinery) → addon template.
- `nuke-mcp/src/nukemcp/server.py` (`build_server` + register wiring + `main()`) → `fastmcp/server.py.tmpl`; `houdini_mcp_server.py` / `natron_mcp_server.py` for the `mcp_cli` idiom.
- `nuke-mcp/src/nukemcp/tools/{_helpers.py,graph.py}` → `_helpers.py.tmpl` + `graph.py.tmpl` exemplars.
- `nuke-mcp/src/nukemcp/mock.py` + `tests/conftest.py` → `mock.py.tmpl` + `conftest.py.tmpl`.
- `nuke-mcp` `events.py`/`rag.py`/`memory.py`/`discovery.py`/`plugins.py` → `templates/advanced/`.

## Framework repo housekeeping
- Rewrite `CLAUDE.md` TODO stub to describe this as a skills+templates plugin (not a Python app).
- The framework repo is not itself a runnable MCP; keep its own footprint minimal. Decide during build whether to drop the scaffolded `main.py`/`tests/` so the framework's own `pytest` doesn't try to import `.tmpl` files (templates live under `templates/` with a `.tmpl` suffix precisely so they aren't collected/linted as framework code).

## Verification plan
1. **Golden path (FastMCP)**: run `scaffold-dcc-mcp` for a fictional `foo` DCC (`{{API_MOD}}=foo`, port 55555, SDK=fastmcp). Acceptance: the generated `foomcp` repo, zero hand edits, runs `pip install -e ".[dev]" && pytest -m "not integration"` green and `ruff check` clean. This exercises `test_connection.py` against the mock server — proves framing + handshake + core commands round-trip in mock mode. (Bake this gate into the skill, step 6.)
2. **Golden path (mcp[cli])**: repeat with SDK=mcp_cli; same green gate. Confirms both variants are valid.
3. **Add-domain regression**: in the toy repo, run `add-tool-domain` for `lighting`; new `test_lighting.py` passes via the mock fixture; tools appear in `--mock`.
4. **Advanced-feature smoke**: run `add-advanced-feature events`; events test passes (injected mock event routes to handler).
5. **Audit round-trip**: point `audit-dcc-mcp` at real `nuke-mcp` (known-good) → near-zero findings; at a deliberately broken copy (confirm guard removed, envelope swapped to `{id,method}`) → flags exactly those. Validates checklist precision/recall.
6. **DRY check**: diff scaffolded `foomcp/src/foomcp/connection.py` against `nuke-mcp/src/nukemcp/connection.py` — differ only in class name/port.
