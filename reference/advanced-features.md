# Advanced Features

Optional, pluggable capabilities that can be added to any DCC MCP server.

## Events: Bidirectional Communication

**File:** `templates/advanced/events.py`

Allow the addon to push events to the client (e.g., "node_created", "knob_changed").

### What It Does

- Addon registers DCC callbacks (e.g., `nuke.addOnCreate()`, `bpy.app.handlers.frame_change_post`)
- When event fires in DCC, addon pushes `{"type":"event","event_type":"...","data":{...}}` to client
- Client's reader thread routes event to handler
- Claude can react to events (log, update state, suggest next steps)

### Implementation

**Addon side:**

```python
def _push_event(event_type: str, data: dict):
    """Send event to connected client."""
    msg = json.dumps({"type": "event", "event_type": event_type, "data": data}) + "\n"
    client_socket.sendall(msg.encode("utf-8"))

def _on_node_created():
    nuke = _get_nuke()
    node = nuke.thisNode()
    _push_event("node_created", {"name": node.name(), "class": node.Class()})

def _handle_subscribe_events(params):
    """Subscribe to event types."""
    event_types = params.get("event_types", [])
    for et in event_types:
        if et == "node_created":
            nuke.addOnCreate(_on_node_created)
        # ... other event types
    return {"status": "ok", "result": {"subscribed": event_types}}
```

**Server side:**

```python
from nukemcp import events

def my_event_handler(msg):
    """Handle event from addon."""
    event_type = msg["event_type"]
    data = msg["data"]
    if event_type == "node_created":
        print(f"Node created: {data['name']}")

conn.set_event_handler(my_event_handler)
```

### Supported Events (Per DCC)

- **Nuke**: node_created, node_deleted, knob_changed, script_loaded, script_saved
- **Blender**: frame_changed, object_created, object_deleted, material_updated
- **Houdini**: node_created, node_deleted, parameter_changed, hip_loaded, hip_saved
- **Natron**: node_created, node_deleted, input_changed, project_loaded, project_saved

### Mock Parity

Mock doesn't push events (no event callbacks). Tests should not rely on events; use polling instead:

```python
# Good
result = conn.send_command("get_script_info")
assert result["node_count"] == 1

# Bad (won't work in mock)
# Wait for node_created event, then check
```

## RAG: Offline Document Search

**File:** `templates/advanced/rag.py`

Embed DCC documentation (API docs, user guides) and provide offline full-text search.

### What It Does

- On startup, reads DCC documentation from disk (or GitHub)
- Indexes documents with embeddings (using Claude API or local embeddings)
- Claude can search: "How do I set up a camera in Houdini?"
- Server returns relevant snippets without network

### Implementation

```python
from nukemcp import rag

# Initialize RAG with DCC docs
rag.init(docs_dir="./docs", embedding_model="text-embedding-3-small")

# Tool: search docs
@mcp.tool()
def search_docs(query: str, limit: int = 5) -> dict:
    """Search DCC documentation."""
    results = rag.search(query, limit=limit)
    return {"status": "ok", "result": results}
```

### Setup

```bash
# Fetch docs from DCC GitHub/download
python -m nukemcp.scripts.ingest_docs --source "https://docs.foundry.com/nuke/..."

# Embed (one-time)
python -c "from nukemcp import rag; rag.init('docs/', embed=True)"
```

### Performance

- **Index time** — 5-10 minutes for 1000 docs (first-time)
- **Search time** — < 100ms per query (in-memory)
- **Storage** — ~100MB per 1000 docs

### No External API

RAG runs offline. Does **not** call OpenAI or other external services for search (though embedding may if using API-based models).

## Memory: Persistent Session State

**File:** `templates/advanced/memory.py`

Persist Claude's internal state across sessions (e.g., "user prefers physically-based materials").

### What It Does

- Server maintains a file-based memory store (JSON, YAML, or markdown)
- Tools can write: `memory.write("user_prefs.yaml", {...})`
- Tools can read: `prefs = memory.read("user_prefs.yaml")`
- Claude can reference past session data

### Implementation

```python
from nukemcp import memory

# Write
memory.write_file("project/myproject.md", """# MyProject
- Scene type: 3D
- Rendering engine: Cycles
- Target resolution: 4K
""")

# Read (as dict, parsed from YAML frontmatter or JSON)
project = memory.read_file("project/myproject.md")
print(project["rendering_engine"])  # "Cycles"
```

### Storage

By default, `~/.nukemcp/memory/` (or `~/.blendermcp/memory/`, etc.):

```
~/.nukemcp/memory/
├── project/
│   ├── myproject.md
│   └── other.md
└── user/
    └── preferences.yaml
```

### Tools

- `memory_write(path, content)` — Write or overwrite file
- `memory_read(path)` — Read file, parse if YAML/JSON
- `memory_list(path)` — List files in directory
- `memory_delete(path)` — Delete file

## Discovery: Auto-Launch & Locate DCC

**File:** `templates/advanced/discovery.py`

Auto-detect and launch DCC; find installations on the system.

### What It Does

- Scan common install paths (e.g., `/opt/Nuke`, `C:\Program Files\Blender`, `/Applications/Houdini/`)
- Return installed versions and paths
- Optionally launch headless DCC for user
- Return port info for MCP to connect to

### Implementation

```python
from nukemcp.discovery import discover_nuke, launch_headless

# Discover
result = discover_nuke()
print(result.has_nuke)  # True
print(result.best)      # NukeInstall(version="17.0", path="/opt/Nuke17.0")

# Launch
proc = launch_headless("/opt/Nuke17.0/nuke-13.0-64", port=54321)
print(proc.pid)

# In main()
if args.headless:
    result = discover_nuke()
    nuke_path = result.best.executable
    proc = launch_headless(nuke_path, port=args.port)
    # Now addon is running on port; client can connect
```

### Platform-Specific Paths

- **Nuke (Linux)** — `/opt/Nuke*`, `~/Nuke*`
- **Nuke (Mac)** — `/Applications/Nuke*`
- **Nuke (Windows)** — `C:\Program Files*\Nuke*`
- **Blender (Linux)** — `/usr/bin/blender`, `/opt/blender*`
- **Blender (Mac)** — `/Applications/Blender.app/...`
- **Houdini (Linux)** — `/opt/hfs*`, `~/hfs*`
- **Houdini (Windows)** — `C:\Program Files\SideFX\Houdini*`

### Headless Launch

For headless, launch with `--no-gui` flag (or DCC-specific equivalent):

```python
# Nuke
subprocess.Popen([
    nuke_exe,
    "--nxpl",  # No UI
    "-t",      # Startup addon script
    "startup.py"
])

# Blender
subprocess.Popen([
    blender_exe,
    "--background",  # Headless
    "--factory-startup",
    "--python", "startup.py"
])
```

## Plugins: Auto-Discovery & Loading

**File:** `templates/advanced/plugins.py`

Auto-discover and load plugin tools from a directory.

### What It Does

- Scan `tools/plugins/` for `*.py` files
- Each file should define `register(server)` function
- Load and register all found plugins at startup

### Implementation

```python
from nukemcp.plugins import load_plugins

# In server.py build_server()
load_plugins(server)  # Looks for tools/plugins/*.py

# User can drop new tool domains in tools/plugins/
# No need to edit server.py
```

### Plugin File Format

**File:** `tools/plugins/myfeature.py`

```python
def register(server):
    """Register plugin tools."""
    mcp = server.mcp
    conn = server.connection
    
    @mcp.tool()
    def my_custom_tool(param: str) -> dict:
        """Do something custom."""
        return conn.send_command("my_custom_cmd", {"param": param})
```

## Undo Integration

**File:** `templates/advanced/undo.py` (planned)

Support undo/redo for commands executed via MCP.

### Concept

- On each command, push an undo point in DCC
- If command fails, undo automatically
- Allow explicit undo/redo from Claude

### Implementation (Sketch)

```python
@mcp.tool()
def create_and_undo():
    """Create node, then undo."""
    conn.send_command("create_node", {...})
    conn.send_command("undo", {})
```

**Status:** Framework-defined but not yet implemented in reference repos.

## Recipes: Canned Workflows

**File:** `templates/advanced/recipes.py` (planned)

Ship common workflows (e.g., "set up denoising in Blender", "create Nuke 3D geo setup").

### Concept

- Store recipes in YAML/JSON
- Tool: list recipes, run recipe by name
- Recipe is a sequence of tool calls

### Example Recipe

```yaml
# recipes/denoise_setup.yaml
name: Setup Denoising in Cycles
steps:
  - tool: modify_node
    params:
      node_name: render_settings
      knobs: { use_denoising: true, denoiser: "OptiX" }
  - tool: modify_node
    params:
      node_name: compositor
      knobs: { enabled: true }
```

**Status:** Framework-defined but not yet implemented in reference repos.

## Version Gating

Use handshake version/variant to enable/disable features:

```python
version = parse_version(handshake)

# Only enable in Nuke 17+
if version.major < 17:
    disable_advanced_tools()

# Only in NukeX (not open-source Nuke)
if handshake["variant"] != "NukeX":
    disable_nuke_studio_only_tools()
```

## Dependency Management

All advanced features should be **stdlib-only** or gracefully degrade:

```python
try:
    import anthropic  # For RAG embeddings (optional)
    HAS_ANTHROPIC = True
except ImportError:
    HAS_ANTHROPIC = False

def register(server):
    if not HAS_ANTHROPIC:
        log.warning("RAG disabled: anthropic SDK not installed")
        return
    # ... register RAG tools
```

Never require external deps for core MCP functionality.
