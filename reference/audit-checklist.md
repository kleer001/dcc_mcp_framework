# DCC MCP Audit Checklist

Walk this checklist against an existing DCC MCP repo to ensure conformance to the framework blueprint.

## Protocol Conformance

- [ ] **Envelope shape** — Commands use `{"type":"...","params":{...}}`
  - [ ] NOT `{"id":N,"method":"...","params":{...}}` (Natron anti-pattern)
  - [ ] NOT `{"command":"...","args":{...}}`
  
- [ ] **Response shape** — Responses use `{"status":"ok"|"error","result"|"error":"..."}`
  - [ ] Success: `{"status":"ok","result":{...}}`
  - [ ] Error: `{"status":"error","error":"<msg>"}`
  - [ ] NOT `{"ok":true,"data":{...}}`
  
- [ ] **Framing** — Newline-delimited JSON (`\n` terminator)
  - [ ] Check addon `_send()` appends `+ "\n"`
  - [ ] Check connection `_read_message()` splits on `b"\n"`
  
- [ ] **Handshake** — On client connect, addon sends handshake before accepting commands
  - [ ] `{"type":"handshake","{{dcc}}_version":"...","variant":"...","pid":...}`
  - [ ] Client reads handshake before sending commands
  - [ ] Handshake includes version, variant, PID
  
- [ ] **Timeout** — Connection has reasonable timeouts (10s typical)
  - [ ] Connect timeout
  - [ ] Response timeout per command
  - [ ] Exponential backoff on reconnect (2s, 4s, 8s, 16s)

## Thread Safety

- [ ] **Reader thread** — Addon has background reader thread
  - [ ] Reads all incoming messages from socket
  - [ ] Routes events (`type=="event"`) to event handler
  - [ ] Routes responses to response queue (FIFO)
  - [ ] Starts on connect, stops on disconnect
  
- [ ] **Main-thread marshalling** — Commands run on DCC main thread
  - [ ] Check `_run_in_dcc()` helper function
  - [ ] GUI mode uses appropriate idiom (executeInMainThread, QTimer, bpy.app.timers)
  - [ ] Headless mode uses `serve_forever()` with queue drain loop
  - [ ] Queue + result_queue pattern for async dispatch
  
- [ ] **Addon main thread** — Addon handler executes on DCC main thread
  - [ ] Handlers call `_run_in_dcc()` or similar
  - [ ] No direct API calls from socket thread
  - [ ] Results returned via queue (not direct return)
  
- [ ] **Daemon threads** — Socket listener is a daemon thread
  - [ ] Thread creation: `daemon=True`
  - [ ] No explicit join() needed on shutdown
  
- [ ] **SO_REUSEADDR** — Server socket has SO_REUSEADDR set
  - [ ] Allows quick restart without TIME_WAIT
  - [ ] `socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)`

## Lazy Imports

- [ ] **DCC import** — DCC module imported lazily
  - [ ] `def _get_dcc(): import nuke; return nuke`
  - [ ] Addon can be imported outside the DCC (for testing)
  - [ ] NOT `import nuke` at module level in addon
  
- [ ] **Test imports** — `pytest` can run without DCC installed
  - [ ] Mock mode available (`--mock` flag)
  - [ ] conftest.py uses mock connection for tests

## Confirm Guards

- [ ] **Destructive ops guarded** — All destructive operations require `confirm=True`
  - [ ] `delete_node(name, confirm=False)` — default is False
  - [ ] `execute_python(code, confirm=False)` — default is False
  - [ ] Any operation that permanently changes state
  
- [ ] **Annotation metadata** — Destructive tools marked with annotation
  - [ ] `@mcp.tool(annotations={"destructiveHint": True})`
  - [ ] Read-only tools marked: `@mcp.tool(annotations={"readOnlyHint": True})`
  - [ ] Idempotent tools marked: `@mcp.tool(annotations={"idempotentHint": True})`

## Mock Parity

- [ ] **Every handler has mock** — For each `_handle_*` in addon, a `_cmd_*` in mock
  - [ ] Count handlers vs mock commands
  - [ ] If count differs, enumerate missing ones
  
- [ ] **Mock state** — Mock maintains coherent state
  - [ ] Nodes dict (create updates, delete removes)
  - [ ] Connections list (create adds, delete removes)
  - [ ] Frame range, FPS, colorspace (set updates)
  - [ ] Script name, proxy mode (set updates)
  
- [ ] **Test runs in mock** — `pytest -m "not integration"` passes with `--mock`
  - [ ] No integration tests run (marked `@pytest.mark.integration`)
  - [ ] All core tools tested via mock
  
- [ ] **Mock response shape** — Mock responses match addon responses
  - [ ] Same `status` and `result`/`error` fields
  - [ ] Same field names and types in result

## Test Coverage

- [ ] **conftest.py** — Fixture provides mock connection
  - [ ] `@pytest.fixture` for `connection` or `mock_server`
  - [ ] Fixture starts mock on `localhost:{{PORT}}`
  - [ ] Fixture cleans up (stops server)
  - [ ] Can be used across all test modules
  
- [ ] **test_connection.py** — Protocol tests
  - [ ] Handshake read
  - [ ] Command/response framing
  - [ ] Error handling
  - [ ] Timeout behavior
  - [ ] Reconnect with backoff
  
- [ ] **test_<domain>.py** — Tools tested
  - [ ] Typical pattern: `test_<tool_name>(connection)`
  - [ ] Each tool has at least one test
  - [ ] Read-only tests can run in any order
  - [ ] Destructive tests check `confirm=True` requirement
  
- [ ] **Integration tests marked** — Tests requiring real DCC marked and deselected by default
  - [ ] `@pytest.mark.integration`
  - [ ] Run with `pytest -m integration` (skipped by default)
  - [ ] Documented in README or conftest

## Tools & Annotations

- [ ] **Core tools present** — Minimum viable tool set implemented
  - [ ] `ping` — Connectivity check
  - [ ] `get_scene_info` / `get_script_info` — Read scene metadata
  - [ ] `get_node_info` — Read node details
  - [ ] `create_node` — Create node
  - [ ] `modify_node` — Set parameters
  - [ ] `delete_node` — Remove node (with confirm=True)
  - [ ] `connect_nodes` — Wire nodes
  - [ ] `execute_python` — Run code (with confirm=True)
  
- [ ] **Tools modular** — Tools organized in domains
  - [ ] `tools/graph.py` — Node graph operations
  - [ ] `tools/script.py` or `tools/scene.py` — Scene/script operations
  - [ ] Each domain file has `def register(server)` entry point
  - [ ] Server imports and calls `domain.register(server)`
  
- [ ] **Annotations present** — Tools have metadata annotations
  - [ ] Read-only tools marked
  - [ ] Destructive tools marked
  - [ ] Idempotent tools marked if appropriate
  - [ ] Example: `@mcp.tool(annotations={"readOnlyHint": True})`

## Packaging & Distribution

- [ ] **pyproject.toml** — Project metadata correct
  - [ ] `name = "{{dcc}}mcp"` (lowercase)
  - [ ] `version = "0.1.0"` or higher
  - [ ] Dependencies listed (fastmcp or mcp[cli])
  - [ ] `[project.scripts]` entry: `{{dcc}}-mcp = {{dcc}}mcp.server:main`
  
- [ ] **mcp.json** — MCP config file present
  - [ ] Points to entry point (usually `src/{{dcc}}mcp/server.py`)
  - [ ] Specifies command to run (`python -m {{dcc}}mcp.server`)
  - [ ] Includes arguments if any (`--mock`, `--headless`, etc.)
  
- [ ] **README** — Installation and usage documented
  - [ ] How to install (pip, git clone, etc.)
  - [ ] How to run (with DCC, headless, mock)
  - [ ] Architecture overview
  - [ ] Reference to framework docs
  
- [ ] **LICENSE** — MIT or compatible
  - [ ] LICENSE file present
  - [ ] LICENSE header in source files (optional but good)

## Advanced Features

- [ ] **Events (if present)** — Bidirectional events work
  - [ ] `subscribe_events(event_types=[...])` command
  - [ ] Addon registers callbacks with DCC
  - [ ] Callbacks push events back to client
  - [ ] Client's reader thread routes to event handler
  - [ ] Events don't break mock mode (gracefully ignored)
  
- [ ] **RAG (if present)** — Offline doc search works
  - [ ] Docs ingestion script present
  - [ ] Embeddings cached on disk
  - [ ] `search_docs(query)` tool returns results
  - [ ] No external API calls
  
- [ ] **Memory (if present)** — Persistent state works
  - [ ] Memory dir created on first write
  - [ ] `memory_write`, `memory_read`, `memory_list` tools
  - [ ] Files can be YAML, JSON, or markdown
  - [ ] Survives across sessions
  
- [ ] **Discovery (if present)** — Auto-launch available
  - [ ] `discover_{{dcc}}()` returns installed versions
  - [ ] `launch_headless(path, port=...)` launches DCC
  - [ ] Main accepts `--headless` flag
  
- [ ] **Plugins (if present)** — Plugin discovery works
  - [ ] `tools/plugins/` directory exists
  - [ ] Each plugin file defines `register(server)`
  - [ ] `load_plugins(server)` loads all discovered

## Edge Cases & Gotchas

- [ ] **PySide2/6 compatible** — GUI imports handle both
  - [ ] Try/except import pattern (PySide6, fallback PySide2)
  
- [ ] **JSON serialization** — Non-JSON objects converted
  - [ ] DCC objects (nodes, etc.) converted to dicts
  - [ ] Fallback to `str()` if not serializable
  
- [ ] **Version gating** — Version/variant from handshake respected
  - [ ] Tools check `server.version.major`, `server.version.variant`
  - [ ] Features disabled gracefully if not available
  
- [ ] **Error messages clear** — Error messages are actionable
  - [ ] NOT just "Error"
  - [ ] Include context: node name, knob name, etc.
  - [ ] Suggest fixes if possible

## Repo Structure

- [ ] **Directory layout matches template**
  - [ ] `src/{{dcc}}mcp/` — Main package
  - [ ] `src/{{dcc}}mcp/server.py` — Entry point
  - [ ] `src/{{dcc}}mcp/connection.py` — Socket client
  - [ ] `src/{{dcc}}mcp/mock.py` — Mock server
  - [ ] `src/{{dcc}}mcp/tools/` — Tool modules
  - [ ] `{{dcc}}_addon/` — In-DCC addon code
  - [ ] `tests/` — Test suite
  - [ ] `mcp.json` — Config
  - [ ] `.scaffold.json` — Record of scaffold
  
- [ ] **.scaffold.json present** — Metadata about generation
  - [ ] Records template version, substitutions made
  - [ ] Do not edit by hand (for framework use)

## Scoring

- **Pass**: All green. No findings.
- **Near Pass**: 1-2 minor issues (documentation, style).
- **Fix Recommended**: 3-5 issues. Fixable with template replacement.
- **Refactor**: 6+ issues. Consider major refactor to match framework.

## Report Template

```markdown
# Audit Report: {{PRETTY_DCC}}-mcp

**Status:** [Pass / Near Pass / Fix Recommended / Refactor]

## Findings

### Critical (must fix)
- [ ] Missing confirm=True on delete_node
- [ ] Natron-style {id, method} envelope detected (line X)
- [ ] No mock parity for create_node handler

### Major (should fix)
- [ ] No event support (optional but recommended)
- [ ] Memory feature not persistent across sessions

### Minor (nice to have)
- [ ] README missing advanced features section
- [ ] Missing {{dcc}}_version in handshake

## Recommendations

1. Replace envelope with template from `templates/shared/...`
2. Add confirm=True guards to destructive ops
3. Implement mock parity for all handlers
4. Run `pytest -m "not integration"` with `--mock` flag

## Ground Truth

- Framework: https://github.com/kleer001/dcc_mcp_framework
- Reference: nuke-mcp (known-good)
```
