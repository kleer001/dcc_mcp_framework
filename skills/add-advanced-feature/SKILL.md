# add-advanced-feature Skill

## Description

Add an opt-in advanced capability to a DCC MCP server: bidirectional events, offline RAG doc search, persistent memory resources, headless auto-launch, version/variant gating, plugin auto-discovery, undo integration, or recipe tools.

## Trigger Patterns

- "Add event support to my nuke-mcp"
- "I want to enable RAG for Houdini docs in my houdini-mcp"
- "Let's add persistent memory to blender-mcp"
- "Add plugin discovery to my foo-mcp"

## Workflow

### 1. Identify Feature

Ask the user: "Which feature do you want to add?"

Options:
1. **Events** — Bidirectional push (node_created, knob_changed, etc.)
2. **RAG** — Offline doc search (embedding + vector search)
3. **Memory** — Persistent session state (read/write files)
4. **Discovery** — Auto-detect and launch DCC
5. **Plugins** — Auto-load tool modules from directory
6. **Undo** — Undo/redo support (planned)
7. **Recipes** — Canned workflows (planned)

### 2. Read Reference

- `reference/advanced-features.md` — Feature documentation
- Relevant section describes: what, which repos have it, module, wiring, test

### 3. Copy Module

Copy the appropriate template from `templates/advanced/`:

- `events.py.tmpl` → `src/{{dcc}}mcp/events.py`
- `rag.py.tmpl` → `src/{{dcc}}mcp/rag.py`
- `memory.py.tmpl` → `src/{{dcc}}mcp/memory.py`
- `discovery.py.tmpl` → `src/{{dcc}}mcp/discovery.py`
- `plugins.py.tmpl` → `src/{{dcc}}mcp/plugins.py`

### 4. Wire Into `server.py`

**For events:**

```python
from {{dcc}}mcp import events

# After mcp creation:
events.register(server)

# For event push (addon-side):
conn.set_event_handler(events.handle_event)
```

**For RAG:**

```python
from {{dcc}}mcp import rag

rag.init(docs_dir="./docs")
rag.register(server)  # Adds search_docs() tool
```

**For memory:**

```python
from {{dcc}}mcp import memory

memory.register(server)  # Adds memory_read, memory_write, memory_list
```

**For discovery:**

```python
from {{dcc}}mcp import discovery

if args.headless:
    result = discovery.discover_{{dcc}}()
    proc = discovery.launch_headless(result.best.executable, port=args.port)
```

**For plugins:**

```python
from {{dcc}}mcp.plugins import load_plugins

# At end of build_server():
load_plugins(server)
```

### 5. Add Dependencies (if any)

Update `pyproject.toml` if feature requires new deps:

```toml
[project.optional-dependencies]
dev = ["pytest>=7.0", "ruff>=0.1"]
events = []  # stdlib-only
rag = ["anthropic>=1.0"]  # Optional
memory = []  # stdlib-only
discovery = []  # stdlib-only
plugins = []  # stdlib-only
```

All features should be **stdlib-only or gracefully degrade** if optional deps missing.

### 6. Add Tests

Create test file (e.g., `tests/test_events.py`):

**Events test:**
```python
def test_subscribe_events(connection):
    result = connection.send_command("subscribe_events", {
        "event_types": ["node_created"]
    })
    assert result["subscribed"] == ["node_created"]
```

**Memory test:**
```python
def test_memory_write_read(connection):
    connection.send_command("memory_write", {
        "path": "test.md",
        "content": "# Test"
    })
    result = connection.send_command("memory_read", {"path": "test.md"})
    assert "Test" in result
```

### 7. Verify

```bash
pytest tests/test_<feature>.py -v
```

Feature works in mock mode.

## Feature Details

### Events

- **What** — Addon pushes notifications when things happen (node created, parameter changed)
- **Tools** — `subscribe_events(types), unsubscribe_events(types)`
- **Addon side** — Register callbacks, push events via `_push_event(type, data)`
- **Server side** — Set event handler on connection

### RAG

- **What** — Embed DCC documentation and enable full-text search
- **Tool** — `search_docs(query, limit=5) -> docs`
- **Setup** — Ingest docs once, then search offline
- **No external API** — Search runs locally

### Memory

- **What** — Read/write persistent files across sessions
- **Tools** — `memory_read(path)`, `memory_write(path, content)`, `memory_list(path)`
- **Storage** — `~/.{{dcc}}mcp/memory/` directory
- **Formats** — YAML, JSON, markdown

### Discovery

- **What** — Auto-detect DCC installations and launch headless
- **Tools** — Used in `main()` for `--headless` flag
- **Implementation** — Scan standard install paths per OS/DCC
- **Returns** — List of installs with version, path, executable

### Plugins

- **What** — Auto-load tool modules from `tools/plugins/`
- **Each plugin** — Define `register(server)` function
- **No server.py changes** — Just call `load_plugins(server)`
- **Discovery** — Scan for `*.py` in `tools/plugins/`

## Checklist

- [ ] Feature understood (read reference/advanced-features.md)
- [ ] Module copied from templates/advanced/
- [ ] Dependencies added to pyproject.toml (if any)
- [ ] Wired into server.py (imports, register calls)
- [ ] Addon-side integration (if applicable)
- [ ] Tests written and pass
- [ ] Graceful degradation if optional deps missing
- [ ] Ruff clean

## Related Skills

- `scaffold-dcc-mcp` — Created this repo
- `add-tool-domain` — Add tool domains
- `audit-dcc-mcp` — Review for conformance
