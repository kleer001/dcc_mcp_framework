# scaffold-dcc-mcp Skill

## Description

Create a new DCC MCP server repository from framework templates. Use whenever the user wants to bootstrap a new MCP server for a DCC tool (Maya, Cinema4D, Krita, Resolve, etc.).

## Trigger Patterns

User says something like:
- "I want to create a new MCP server for Cinema4D"
- "Scaffold an MCP for Krita"
- "Let's build a Maya MCP"
- "I need a new DCC MCP"

Do NOT use for:
- Improving an existing MCP repo (use `audit-dcc-mcp`)
- Adding tools to an existing repo (use `add-tool-domain`)

## Workflow

### 1. Interview & Requirements

Read:
- `reference/architecture.md` (understand two-process design)
- `reference/protocol.md` (envelope, handshake, framing)

Ask the user:
1. **DCC name** → Derive `{{DCC}}` (PascalCase), `{{dcc}}` (lowercase)
2. **Python API module** → `{{API_MOD}}` (e.g., `bpy`, `hou`, `c4d`, `natron`)
3. **SDK choice** → FastMCP (`fastmcp>=2.14`) or mcp[cli] (`mcp[cli]>=1.4`)
4. **GUI marshalling** → `{{GUI_MARSHAL}}` from `reference/threading.md` table
5. **Nouns** → `{{SCENE_NOUN}}` (script/scene/document), `{{NODE_NOUN}}` (node/object)
6. **Port** → `{{PORT}}` (suggest 54322+ if Nuke is 54321)
7. **Display name** → `{{PRETTY_DCC}}` (e.g., "Maxon Cinema 4D")

Confirm the substitution map before proceeding.

### 2. Instantiate Templates

Using the substitution map, generate:

**From `templates/shared/`:**
- `src/{{dcc}}mcp/connection.py` (VERBATIM core; only class names/port change)
- `src/{{dcc}}mcp/mock.py` (mock server skeleton)
- `src/{{dcc}}mcp/tools/__init__.py` (empty or minimal)
- `src/{{dcc}}mcp/tools/_helpers.py` (send, require_confirm)
- `{{dcc}}_addon/{{dcc}}_mcp_addon.py` (addon server, GUI marshal idiom)
- `tests/conftest.py` (mock-server + connection fixtures)
- `tests/test_connection.py` (protocol tests)
- `mcp.json` (entry point config)

**From `templates/fastmcp/` OR `templates/mcp_cli/` (user's choice):**
- `pyproject.toml` (with correct SDK dependency)
- `src/{{dcc}}mcp/server.py` (entry point, tool registration)
- `src/{{dcc}}mcp/tools/graph.py` (exemplar tool domain)

**Generate from exemplar:**
- `src/{{dcc}}mcp/tools/graph.py` — Adapt nuke-mcp/tools/graph.py
  - Replace Nuke-specific APIs (`nuke.toNode`, `nuke.createNode`) with `{{API_MOD}}` equivalents
  - Keep tool signatures and annotations unchanged
  - Document API differences in comments

**New files:**
- `CLAUDE.md` (describe the generated repo; reference to framework)
- `README.md` (installation, how to run, architecture overview)
- `BESTPRACTICES.md` (DCC-specific best practices, gotchas, checklist) ← **Strongest feature**
- `.scaffold.json` (record of substitutions made for future audits)

### 3. Generate Tool Exemplars

Core tool set for the scaffold (adapter from `templates/fastmcp/tools/graph.py.tmpl`):

1. `ping()` — Connectivity check
2. `get_scene_info()` — Scene metadata (name, frame range, {{NODE_NOUN}} count)
3. `get_{{NODE_NOUN}}_info(name)` — {{NODE_NOUN}} details
4. `create_{{NODE_NOUN}}(class, name, position)` — Create {{NODE_NOUN}}
5. `modify_{{NODE_NOUN}}(name, params)` — Set parameters
6. `delete_{{NODE_NOUN}}(name, confirm=False)` — Delete with confirm guard
7. `connect_{{NODE_NOUN}}s(output, input)` — Wire {{NODE_NOUN}}s
8. `execute_code(code, confirm=False)` — Run Python (with confirm guard)

For each tool:
- **Addon handler** (`_handle_<cmd>`) — Calls `{{API_MOD}}` APIs
- **Mock handler** (`_cmd_<cmd>`) — Simulates behavior without DCC
- **Tool registration** — `@mcp.tool()` with annotations

### 4. Ensure Mock Parity

For every addon handler, a matching mock handler exists:
- Same response shape (`{"status":"ok","result":{...}}`)
- Same error format (`{"status":"error","error":"..."}`)
- Maintains coherent state (nodes dict, connections, frame range)

Verify by running tests in mock mode.

### 5. Verify (Golden Path)

**Critical gate:**

```bash
cd <generated-repo>
pip install -e ".[dev]"
pytest -m "not integration"  # Mock mode
ruff check . && ruff format .
```

If tests fail or ruff complains, **do not hand back the repo**. Debug and fix:
- Mock parity issue (missing `_cmd_*` handler)?
- Syntax error in generated code?
- Import error (missing __init__.py)?

Only return a repo that passes these checks.

### 6. Summary & Handoff

Print:
1. **Repo structure** — Tree of generated files
2. **Substitution map** — All {{...}} replacements made
3. **Install one-liner** — How to clone and install
4. **mcp.json location** — Where to configure in Claude Code
5. **Next steps** — Add domains, advanced features, implement addon logic

Example:
```
✓ Scaffolded foo-mcp (SDK: fastmcp, Port: 55555, API: foo)

Substitution map:
  {{DCC}} = Foo
  {{dcc}} = foo
  {{PRETTY_DCC}} = Fictional Foo DCC
  {{API_MOD}} = foo
  {{PORT}} = 55555
  {{GUI_MARSHAL}} = foo.run_on_main_thread(func)
  {{SCENE_NOUN}} = scene
  {{NODE_NOUN}} = object

Installation:
  pip install -e "./foo-mcp[dev]"
  pytest -m "not integration" --mock
  python -m foomcp.server --mock

mcp.json location:
  ./foo-mcp/mcp.json

Next:
  1. Implement real {{API_MOD}} API calls in addon handlers
  2. Test with real DCC (pytest -m integration)
  3. Add tool domains via add-tool-domain skill
  4. Push to GitHub; share mcp.json
```

## Implementation Notes

- **SDK choice** — Affects only `server.py` and `pyproject.toml`. Connection, addon, mock, tests are identical.
- **API mapping** — Scaffold provides tool *signatures* (e.g., `create_node(class, name)`); user implements DCC-specific logic later.
- **Error handling** — Addon handlers should use try/except and return `{"status":"error","error":"..."}`.
- **Headless support** — Addon should support both GUI (`start()`) and headless (`serve_forever()`) modes.

## Failure Cases

**If mock tests fail:**
- Missing import in generated code? Check __init__.py files.
- _cmd_* handler missing? Verify mock.py has all commands.
- Syntax error? Validate template substitution.

**If ruff fails:**
- Check line length (100 chars default)
- Check import order (stdlib → third-party → local)
- Auto-fix with `ruff format .`

**If install fails:**
- `pip install -e ".[dev]"` requires setuptools; check pyproject.toml
- Check Python version >= 3.10

## Related Skills

- `add-tool-domain` — Add new tools to this repo
- `add-advanced-feature` — Add events, RAG, memory, etc.
- `audit-dcc-mcp` — Review this repo for conformance
