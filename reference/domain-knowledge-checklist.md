# Domain Knowledge Checklist: 12 Non-Templatable Components

This document lists the 12 components that **require DCC-specific knowledge and implementation**. For each, it explains:

1. **What it does** — Purpose and behavior
2. **Where to find it** — How to research in your DCC's docs/API
3. **Code pattern** — Structure to follow when implementing
4. **Reference repos** — Which existing MCP has a working example
5. **Common gotchas** — Pitfalls to avoid

---

## 1. Domain-Specific Tool Implementations (8-31 tools)

### What It Does
Create MCP tools that expose your DCC's core capabilities. Examples: create nodes, modify parameters, list assets, render, track, simulate, etc.

Each tool:
- Takes parameters from Claude
- Calls your DCC's Python API
- Returns structured results
- May require `confirm=True` if destructive

### Where to Find It
**In your DCC's documentation:**
- API reference (e.g., Nuke Python API, bpy documentation, `hou` module docs)
- Scripting guides and examples
- Source code of built-in scripts or plugins

**Research tasks:**
- List what your DCC's Python API can do (create, delete, modify, query, render, track, etc.)
- For each capability, find the API function/class and parameter schema
- Test the API in your DCC's script editor/console
- Document the exact function signature and return format

### Code Pattern

```python
# tools/domain.py (e.g., tools/geometry.py, tools/rendering.py)

def register(server):
    mcp = server.mcp
    conn = server.connection
    
    @mcp.tool(annotations={"readOnlyHint": True})
    def list_objects() -> dict:
        """List all objects in the scene."""
        return conn.send_command("list_objects", {})
    
    @mcp.tool()
    def create_object(name: str, type: str, position: list) -> dict:
        """Create a new object.
        
        Args:
            name: Object name
            type: Object type (e.g., 'mesh', 'light', 'camera')
            position: [x, y, z] position
        
        Returns:
            {"status": "ok", "result": {"id": "...", "name": "..."}}
        """
        return conn.send_command("create_object", {
            "name": name,
            "type": type,
            "position": position
        })
    
    @mcp.tool(annotations={"destructiveHint": True})
    def delete_object(name: str, confirm: bool = False) -> dict:
        """Delete an object."""
        if not confirm:
            return {
                "action": "delete_object",
                "name": name,
                "message": "Confirm deletion, then call with confirm=True"
            }
        return conn.send_command("delete_object", {"name": name})
```

**Addon-side implementation:**

```python
# addon.py

def _get_dcc():
    """Lazy import your DCC module."""
    import your_dcc_module  # e.g., 'nuke', 'bpy', 'hou'
    return your_dcc_module

def _handle_list_objects(params):
    """Addon receives this command."""
    dcc = _get_dcc()
    
    try:
        # TODO: Replace with your DCC's API calls
        # Example (Nuke): objects = nuke.allNodes()
        # Example (Blender): objects = bpy.data.objects[:]
        # Example (Houdini): objects = hou.node("/obj").children()
        
        objects = [...]  # Call your DCC API
        return {
            "status": "ok",
            "result": {"objects": objects}
        }
    except Exception as e:
        return {"status": "error", "error": str(e)}

def _handle_create_object(params):
    """Create object via DCC API."""
    dcc = _get_dcc()
    
    try:
        # TODO: Replace with your DCC's creation API
        obj = dcc.create_object(params["name"], params["type"])
        # Set position if your DCC supports it
        if hasattr(obj, 'position'):
            obj.position = params["position"]
        
        return {
            "status": "ok",
            "result": {
                "id": obj.id,  # or obj.name, or whatever identifies it
                "name": obj.name
            }
        }
    except Exception as e:
        return {"status": "error", "error": str(e)}

# Register in COMMANDS dict
COMMANDS = {
    "list_objects": _handle_list_objects,
    "create_object": _handle_create_object,
    # ... more handlers
}
```

**Mock-side implementation:**

```python
# mock.py

class Mock{{DCC}}Server:
    def __init__(self):
        self.objects = {}  # Simulate scene state
    
    def _cmd_list_objects(self, params):
        """Mock version (no DCC, just state dict)."""
        objs = [
            {"id": k, "name": v["name"], "type": v["type"]}
            for k, v in self.objects.items()
        ]
        return {"status": "ok", "result": {"objects": objs}}
    
    def _cmd_create_object(self, params):
        """Create in mock state."""
        obj_id = len(self.objects) + 1
        self.objects[obj_id] = {
            "name": params["name"],
            "type": params["type"],
            "position": params["position"]
        }
        return {
            "status": "ok",
            "result": {"id": obj_id, "name": params["name"]}
        }
```

### Reference Repos
- **Nuke** — `nuke-mcp/src/nukemcp/tools/*.py` (16 domains: graph, comp, deep, ML, etc.)
- **Blender** — `blender-mcp/src/blendermcp/tools/*.py` (31 domains: materials, modifiers, animation, etc.)
- **Houdini** — `houdini-mcp/src/houdinimcp/handlers/*.py` (25 handlers: geometry, rendering, nodes, etc.)

### Common Gotchas

- ❌ **Forgetting lazy import** — Never `import nuke` at module level; use `_get_dcc()` function
- ❌ **Forgetting try/except** — All DCC API calls can fail; always wrap in error handler
- ❌ **Returning wrong shape** — Must return `{"status": "ok", "result": {...}}` format
- ❌ **Forgetting mock handler** — For every `_handle_*`, add `_cmd_*` in mock.py for testing
- ❌ **Hard-coded assumptions** — Don't assume object names are unique, positions are 3D, etc.; check your DCC
- ✅ **Document parameter types** — Your docstring should match the tool signature exactly

---

## 2. Addon Lifecycle & Plugin Integration

### What It Does
Load and start the MCP server as a plugin in your DCC. Handle startup, shutdown, logging, and UI (if applicable).

Different per DCC:
- **Nuke** — PySide panel with start/stop button
- **Blender** — `bl_info` registration + properties panel
- **Houdini** — pythonrc hook + package installation
- **Maya** — userSetup.py + menu registration

### Where to Find It
**Research in your DCC's plugin documentation:**
- "How to write a plugin" guide
- "Startup scripts" or "initialization" documentation
- Examples of official or community plugins
- Plugin API for your DCC version

**For your DCC specifically:**
- **Nuke** — Nuke Python API guide → "Script Environment" → startup scripts, menu.py
- **Blender** — blender.org/api → "Add-ons" section
- **Houdini** — SideFX docs → "Extending Houdini" → packages, pythonrc
- **Maya** — Autodesk docs → "Developer Reference" → MEL/Python plugins

### Code Pattern

```python
# {{dcc}}_addon/{{dcc}}_mcp_addon.py

import threading
import socket
import logging

log = logging.getLogger("{{dcc}}_mcp_addon")

class {{DCC}}MCPServer:
    """MCP server running in {{DCC}} process."""
    
    def __init__(self, port={{PORT}}):
        self._port = port
        self._socket = None
        self._listener_thread = None
        self._running = False
        self._main_queue = None
    
    def start(self):
        """Start the MCP server."""
        log.info(f"Starting {{PRETTY_DCC}} MCP server on port {self._port}")
        
        # TODO: Choose ONE of these based on your DCC's threading model:
        
        # Option A: Queue-based (Nuke pattern)
        # self._main_queue = queue.Queue()
        # self._listener_thread = threading.Thread(target=self._socket_listen, daemon=True)
        # self._listener_thread.start()
        # Main thread must call: self.serve_forever() in a loop
        
        # Option B: Timer-based (Blender pattern)
        # self._listener_thread = threading.Thread(target=self._socket_listen, daemon=True)
        # self._listener_thread.start()
        # your_dcc.register_timer(self._poll_queue)  # Called ~10ms
        
        # Option C: Event-based (Houdini pattern)
        # self._listener_thread = threading.Thread(target=self._socket_listen, daemon=True)
        # self._listener_thread.start()
        # your_dcc.register_event_handler(self._on_event)
    
    def _socket_listen(self):
        """Background thread: listen for commands."""
        self._socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        self._socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        self._socket.bind(("localhost", self._port))
        self._socket.listen(1)
        
        log.info(f"Listening on localhost:{self._port}")
        
        while self._running:
            try:
                conn, addr = self._socket.accept()
                log.debug(f"Connection from {addr}")
                self._handle_connection(conn)
            except Exception as e:
                if self._running:
                    log.error(f"Socket error: {e}")
    
    def _handle_connection(self, conn):
        """Process commands from MCP client."""
        # TODO: Implement protocol handler
        # Read newline-delimited JSON
        # Parse command and dispatch to handler
        # Return response
        pass
    
    def serve_forever(self):
        """Main thread drains command queue (Nuke pattern)."""
        # TODO: Implement based on your threading model
        # For Nuke: while loop that gets from queue, executes, puts result
        # For Blender: timer callback
        # For Houdini: event callback
        pass
    
    def stop(self):
        """Stop the server gracefully."""
        log.info("Stopping {{PRETTY_DCC}} MCP server")
        self._running = False
        
        if self._listener_thread:
            self._listener_thread.join(timeout=2.0)
        
        if self._socket:
            try:
                self._socket.close()
            except OSError:
                pass


# DCC-specific initialization
# TODO: Replace with your DCC's plugin/startup pattern

# Nuke:
# def addMenu():
#     toolbar = nuke.menu("Nuke")
#     toolbar.addCommand("MCP/Start Server", "{{dcc}}_mcp_addon.start_server()")

# Blender:
# bl_info = {
#     "name": "{{PRETTY_DCC}} MCP",
#     "version": (0, 1, 0),
#     "blender": (4, 0, 0),
#     "category": "System",
# }

# Houdini:
# In pythonrc:
# import {{dcc}}_mcp_addon
# {{dcc}}_mcp_addon.start()

def start():
    """Start the MCP server."""
    global _server
    if _server is None:
        _server = {{DCC}}MCPServer()
        _server.start()
```

### Reference Repos
- **Nuke** — `nuke-mcp/nuke_addon/nuke_mcp_addon.py` (PySide panel + socket listener)
- **Blender** — `blender-mcp/blender_addon/` (bl_info registration)
- **Houdini** — `houdini-mcp/src/houdinimcp/__init__.py` (pythonrc hook)

### Common Gotchas

- ❌ **Blocking main thread** — If you block on socket.recv(), GUI will freeze (Blender, Nuke)
- ❌ **Not handling shutdown** — Sockets must close; threads must join; data must flush
- ❌ **Hard-coded paths** — Don't assume addon lives in specific folder; make paths relative
- ❌ **Forgetting daemon threads** — Background thread should be `daemon=True` or it'll prevent DCC exit
- ✅ **Use timeout on socket.recv()** — Allows checking `_running` flag periodically

---

## 3. Custom Helpers & Utilities

### What It Does
DCC-specific utilities that help implement tools. Examples:
- Confirmation patterns for destructive ops
- Rendering utilities (viewport, OpenGL, Mantra, Karma)
- Event deduplication (Houdini: collapse 5 parameter changes in 100ms into 1 event)
- Path conversion (relative to absolute, with variable substitution)

### Where to Find It
**Research in your DCC:**
- Common API patterns (e.g., "How do I check if operation succeeded?")
- Event handling (if applicable)
- Performance patterns (caching, deduplication)
- Rendering backends and APIs

### Code Pattern

```python
# tools/_helpers.py or addon.py

def require_confirm(message: str) -> dict:
    """Helper for destructive operations."""
    return {
        "action": "confirm_required",
        "message": message,
        "next_step": "Call again with confirm=True"
    }

def expand_path(path: str, context=None) -> str:
    """Expand variables in paths.
    
    TODO: Replace with your DCC's path expansion:
    - Nuke: nuke.filename(node) for rendering
    - Blender: bpy.path.abspath(path) for Blender paths
    - Houdini: hou.expandString(path) for expressions
    """
    # Example (Nuke):
    # if "$F" in path:
    #     frame = nuke.root()["frame"].value()
    #     path = path.replace("$F", f"{frame:04d}")
    return path

def deduplicate_events(events, window_ms=100):
    """Collapse rapid events (e.g., 5 param changes → 1 change event).
    
    TODO: Implement if your DCC fires rapid events:
    - Same node, same parameter within window → count 1
    - Different nodes → keep separate
    """
    pass
```

### Reference Repos
- **Houdini** — `houdini-mcp/src/houdinimcp/event_collector.py` (deduplication)
- **Nuke** — `nuke-mcp/src/nukemcp/tools/_helpers.py` (confirmation pattern)
- **Blender** — `blender-mcp/src/blendermcp/tools/_helpers.py`

### Common Gotchas

- ❌ **Over-engineering** — Only add helpers for patterns you use 3+ times
- ✅ **Document assumptions** — If helper assumes frame range is 1-1000, document it

---

## 4. Version Gating & Feature Availability

### What It Does
Enable/disable tools based on DCC version or variant. Examples:
- Nuke 17+ only: AI Denoise, Splats
- NukeX only: CameraTracker, Tracker4
- Blender 4.0+ only: Geometry Nodes
- Houdini headless: no playbar events

### Where to Find It
**Research in your DCC:**
- How to detect DCC version programmatically
- How to detect variants/editions (if applicable)
- Which features exist in which versions

### Code Pattern

```python
# addon.py or version.py

class Version:
    """DCC version information."""
    
    def __init__(self, handshake: dict):
        # handshake comes from MCP startup
        # {"version": "17.0v1", "variant": "NukeX"}
        self.version_str = handshake.get("version", "0.0.0")
        self.variant = handshake.get("variant", "standard")
        self._parse()
    
    def _parse(self):
        """Parse version string to tuple."""
        # TODO: Implement for your DCC
        # Example (Nuke): "17.0v1" → (17, 0, "v1")
        # Example (Blender): "4.1.0" → (4, 1, 0)
        parts = self.version_str.split(".")
        self.major = int(parts[0])
        self.minor = int(parts[1]) if len(parts) > 1 else 0
    
    def at_least(self, major: int, minor: int = 0) -> bool:
        """Check if version >= major.minor."""
        return (self.major, self.minor) >= (major, minor)
    
    @property
    def is_variant(self) -> bool:
        """Check if this is a special variant."""
        # TODO: Implement for your DCC
        # Example (Nuke): self.variant == "NukeX"
        # Example (Houdini): self.variant == "Indie"
        return self.variant != "standard"


# In tools registration:

def register(server):
    mcp = server.mcp
    version = server.version
    
    # Tool available only on 17+
    if version.at_least(17, 0):
        @mcp.tool()
        def ai_denoise():
            return conn.send_command("ai_denoise", {})
    
    # Tool available only on NukeX
    if "NukeX" in version.variant:
        @mcp.tool()
        def track_camera():
            return conn.send_command("track_camera", {})
```

### Reference Repos
- **Nuke** — `nuke-mcp/src/nukemcp/version.py` (detailed version + variant gating)
- **Blender** — Likely in addon registration (check blender_addon/server.py)

### Common Gotchas

- ❌ **Forgetting to check** — Tool registered but fails on older version
- ✅ **Test on multiple versions** — If you support 2+ versions, test on each

---

## 5. RAG/Documentation Search Pipeline

### What It Does
Index your DCC's documentation and provide `search_docs(query)` MCP tool for Claude to find answers offline.

Three phases:
1. **Ingest** — Download/copy docs
2. **Index** — Build searchable index (BM25 or embeddings)
3. **Search** — Return ranked results

### Where to Find It
**Research documentation sources for your DCC:**
- Official online docs (GitHub, website, PDF)
- Bundled docs (installed with DCC)
- Community resources (forums, GitHub wikis)
- Source code examples

**For your DCC specifically:**
- **Nuke** — Foundry docs (HTML), Nuke Python API docs
- **Blender** — docs.blender.org, Blender Python API
- **Houdini** — SideFX docs (HTML), example `.hip` files
- **Maya** — Autodesk docs, Python API docs

### Code Pattern

```python
# scripts/ingest_docs.py (already templated!)

# Use templates/advanced/scripts/ingest_docs.py.tmpl as-is
# It supports: local files, GitHub repos, web scraping

# TODO: Configure for your DCC's doc sources

# Example (Nuke):
# python scripts/ingest_docs.py --source web --url https://nuke.foundry.com/docs/17/
# python scripts/ingest_docs.py --source github --repo foundry/nuke-docs

# Example (Blender):
# python scripts/ingest_docs.py --source web --url https://docs.blender.org/api/
# python scripts/ingest_docs.py --source local --path /path/to/blender/docs

# Example (Houdini):
# python scripts/ingest_docs.py --source local --path $HFS/docs/
# # PLUS custom script to extract .hip examples:
# python scripts/ingest_hips.py $HFS/help/examples/*.hip
```

**Then build index:**

```bash
# Already templated in templates/advanced/rag.py.tmpl
python -m {{dcc}}mcp.rag --build
```

### Reference Repos
- **Nuke** — `nuke-mcp/src/nukemcp/rag.py` (BM25 search over docs)
- **Houdini** — `houdini-mcp/scripts/ingest_hips.py` (extracts patterns from `.hip` files!)
- **Templates** — `templates/advanced/rag.py.tmpl` (ready-to-use)

### Common Gotchas

- ❌ **Ignoring incomplete docs** — If docs are out of date, RAG will give wrong info
- ✅ **Keep docs fresh** — Re-ingest docs when DCC updates (monthly or quarterly)
- ✅ **Test search quality** — Try queries like "how do I create a node?" to verify results are relevant

---

## 6. Stateful Mock Server (if needed)

### What It Does
Simulate your DCC's scene/state for testing **without running the real DCC**.

Optional but strongly recommended if:
- Your addon has complex state (nodes, connections, parameters)
- You want fast tests (no DCC startup overhead)
- You want deterministic tests

### Where to Find It
**Study your DCC's data model:**
- What is the "scene" or "document"?
- What are the main objects (nodes, objects, layers)?
- How do they connect/relate?
- What are valid states?

### Code Pattern

```python
# mock.py

class Mock{{DCC}}Server:
    """Simulate {{DCC}} without running the real application."""
    
    def __init__(self):
        # TODO: Initialize scene state for your DCC
        self.nodes = {}  # {id: {name, class, inputs, outputs}}
        self.connections = []  # [(output, input), ...]
        self.scene_name = "untitled"
        self.frame_range = (1, 100)
        self.current_frame = 1
    
    def _cmd_create_node(self, params):
        """Mock: create node in state dict."""
        node_id = len(self.nodes) + 1
        self.nodes[node_id] = {
            "id": node_id,
            "name": params.get("name", f"node{node_id}"),
            "class": params.get("class"),
            "inputs": [],
            "outputs": []
        }
        return {
            "status": "ok",
            "result": {
                "id": node_id,
                "name": self.nodes[node_id]["name"]
            }
        }
    
    def _cmd_connect_nodes(self, params):
        """Mock: validate connection and add to state."""
        output_id = params.get("output")
        input_id = params.get("input")
        
        if output_id not in self.nodes or input_id not in self.nodes:
            return {"status": "error", "error": "Node not found"}
        
        self.connections.append((output_id, input_id))
        return {"status": "ok", "result": {"connected": True}}
    
    def _cmd_delete_node(self, params):
        """Mock: remove node and its connections."""
        node_id = params.get("id")
        
        if node_id not in self.nodes:
            return {"status": "error", "error": "Node not found"}
        
        del self.nodes[node_id]
        # Clean up connections
        self.connections = [
            (o, i) for o, i in self.connections
            if o != node_id and i != node_id
        ]
        
        return {"status": "ok", "result": {"deleted": node_id}}
```

### Reference Repos
- **Nuke** — `nuke-mcp/src/nukemcp/mock.py` (502 lines, detailed scene simulation)
- **Blender** — `blender-mcp/src/blendermcp/mock.py` (400+ lines with constraints)

### Common Gotchas

- ❌ **Over-simulating** — Don't recreate the entire DCC; just simulate enough for tests
- ✅ **Keep it simple** — A dict of state is usually enough

---

## 7. Multi-Phase Skills/Workflows

### What It Does
Multi-step procedures that Claude can execute. Examples:
- `retarget-fx-shot` — Copy FX rig, rename nodes, remap file paths
- `set-up-render-farm` — Configure rendering for farm submission
- `create-tracking-setup` — Set up markers and solvers for motion tracking

These are **not** individual tools, but **coordinated sequences** of tool calls with decision points.

### Where to Find It
**Identify common workflows in your DCC:**
- What do TDs/artists repeat weekly?
- What requires multiple steps and decisions?
- What has error-prone manual steps?

### Code Pattern

```yaml
# skills/retarget-fx-shot.md

# Retarget FX Rig to New Shot

## Overview
Copy a completed FX rig from one shot to another, remapping node names and file paths.

## Prerequisites
- Source shot has complete FX rig (constraints, expressions, simulations)
- Target shot has placeholder geometry or clean scene
- Both shots use same file structure (e.g., /shots/{shot}/geometry/)

## Workflow

### Phase 1: Copy Nodes
{{ invoke search_docs "copy nodes between shots" }}
1. Ask user for source shot ID (e.g., "SH010")
2. List all FX nodes in source shot
3. Copy selected nodes to clipboard
4. Paste into target shot
5. Result: duplicate FX rig, but with OLD references

### Phase 2: Rename Nodes
1. Parse node names for source shot ID (e.g., "sh010_rig_v02")
2. Ask user for target shot ID (e.g., "SH020")
3. Rename all nodes: replace "sh010" → "sh020"
4. Update expression strings in parameters
5. Result: nodes now reference target shot

### Phase 3: Remap File Paths
1. Scan all File nodes and cache nodes
2. Extract file paths and regex patterns
3. Match against source geometry pattern
4. Replace with target geometry path
5. Verify all paths resolve
6. Result: simulations/caches now reference target geometry

### Phase 4: Validate
1. Check all connections are intact
2. Verify no broken references
3. Run initial cook/cache to validate setup
4. Report any missing files or errors
5. Done!

## Decision Points
- Q: Use constraints from source? (affects rig reuse)
- Q: Preserve animation keyframes? (affects performance)
- Q: Reload caches from disk or recompute? (affects time)

## Common Issues
- **Paths don't match** — geometry structure differs between shots
- **Expressions break** — expressions reference absolute paths
- **Constraints fail** — target geometry topology differs
```

### Reference Repos
- **Nuke** — `nuke-mcp/skills/retarget-fx-shot.md` (copy + rename + remap)
- **Houdini** — `houdini-mcp/skills/retarget-fx-shot.md` (similar, but hou.copyNodesTo())

### Common Gotchas

- ❌ **Assuming user knows what to do** — Add decision points ("Should I ...?")
- ✅ **Document gotchas** — Explain why steps fail and how to fix

---

## 8. Installation & Discovery Tooling

### What It Does
Help users install the MCP and launch {{DCC}} in headless mode.

Two parts:
1. **Discovery** — Find {{DCC}} installation, detect version, check license
2. **Bootstrap** — Install addon, download docs, configure MCP client

### Where to Find It
**Research for your DCC:**
- Standard install locations per OS (Linux: /opt, macOS: /Applications, Windows: C:\Program Files)
- How to detect version (binary --version, environment vars, registry keys)
- License checking (env vars, license server, trial tokens)
- How to launch headless (--headless flag, hython, blender --background, etc.)

### Code Pattern

```python
# discovery.py (already templated!)
# Use templates/advanced/discovery.py.tmpl as starting point

# TODO: Implement for your DCC:

def _get_standard_paths(self) -> list[Path]:
    """Get standard {{DCC}} install paths per OS."""
    if self.platform == "linux":
        return [
            # TODO: Fill in Linux paths for your DCC
            # Example (Nuke): /opt/nuke, /opt/nuke_*/bin/nuke
            # Example (Houdini): /opt/hfs*, $HOME/.hou
        ]
    elif self.platform == "darwin":
        return [
            # TODO: Fill in macOS paths
            # Example (Nuke): /Applications/Nuke*.app
            # Example (Houdini): /Applications/Houdini*.app
        ]
    elif self.platform == "win32":
        return [
            # TODO: Fill in Windows paths
            # Example (Nuke): C:\Program Files\Nuke*
        ]

def _detect_version(self, path: Path) -> str:
    """Detect {{DCC}} version from path or binary."""
    # TODO: Implement for your DCC
    # Option A: Parse from folder name
    #   e.g., /opt/nuke17.0v1 → "17.0v1"
    # Option B: Run binary --version
    #   e.g., /opt/nuke/bin/nuke --version → parse output
    # Option C: Check metadata files
    #   e.g., /Applications/Nuke.app/Contents/Info.plist

def _check_license(self, path: Path) -> bool:
    """Check if {{DCC}} has valid license."""
    # TODO: Implement for your DCC
    # Option A: Check env var
    #   NUKE_LICENSE_FILE, HKEY_LOCAL_MACHINE\..., etc.
    # Option B: Try to start DCC with --license check flag
    # Option C: Check license server (RLM, Flexnet)

def launch_headless(install: {{DCC}}Install) -> subprocess.Popen:
    """Launch {{DCC}} in headless mode."""
    # TODO: Implement for your DCC
    # Example (Nuke): nuke -t -x script.nk
    # Example (Houdini): hython
    # Example (Blender): blender --background --python script.py
    # Example (Maya): maya -batch -c "..."
```

**Bootstrap script (already templated!):**

```bash
# scripts/bootstrap.sh (use templates/advanced/scripts/bootstrap.sh.tmpl)

# The template handles:
# - Platform detection
# - Python version check
# - Prerequisite check (git, uv)
# - Repo clone/update
# - uv pip install

# TODO: Customize for your DCC:
# - Addon installation paths (LINUX/macOS/Windows)
# - Plugin registration (pythonrc, packages, bl_info, etc.)
# - Docs download source (if applicable)
```

### Reference Repos
- **Nuke** — `nuke-mcp/src/nukemcp/discovery.py` (comprehensive, 503 lines)
- **Houdini** — `houdini-mcp/bootstrap.sh` (350 lines, platform detection)
- **Templates** — `templates/advanced/discovery.py.tmpl` (skeleton to fill in)

### Common Gotchas

- ❌ **Hard-coded paths** — Use standard OS paths, not user-specific
- ❌ **Forgetting headless mode** — Headless launch is different for each DCC
- ✅ **Test on all platforms** — Windows/macOS/Linux have different paths

---

## 9. Persistent Memory System

### What It Does
Store session data across {{DCC}} restarts. Use cases:
- Facility conventions (preferred nodes, colorspaces, naming)
- Corrections applied to shots (for future reference)
- User preferences and settings
- Project metadata

### Where to Find It
**Determine your DCC's workflow:**
- What information should TDs remember across sessions?
- What conventions are project-specific?
- What decisions are made once and reused?

### Code Pattern

```python
# memory.py (already templated!)
# Use templates/advanced/memory.py.tmpl

# It provides:
# - memory_read(key) — read a JSON file
# - memory_write(key, value) — write a JSON file
# - memory_list() — list all stored keys
# - memory_delete(key) — delete a file

# TODO: Wire into your workflow:

# In addon startup:
def _load_facility_conventions():
    """Load facility settings on startup."""
    from {{dcc}}mcp.memory import Memory
    mem = Memory()
    
    conventions = mem.read("facility/conventions")
    if conventions:
        # Apply conventions to new scenes
        apply_colorspace(conventions.get("colorspace", "linear"))
        set_naming_convention(conventions.get("naming", "snake_case"))

# In tools:
@mcp.tool()
def save_correction(shot_id: str, description: str) -> dict:
    """Log a correction for future reference."""
    from {{dcc}}mcp.memory import Memory
    mem = Memory()
    
    key = f"corrections/{shot_id}"
    corrections = mem.read(key) or []
    corrections.append({
        "date": datetime.now().isoformat(),
        "description": description
    })
    mem.write(key, corrections)
    
    return {"status": "ok", "result": {"logged": True}}
```

### Reference Repos
- **Nuke** — `nuke-mcp/src/nukemcp/memory.py` (137 lines, used for facility data)
- **Templates** — `templates/advanced/memory.py.tmpl` (ready-to-use)

### Common Gotchas

- ❌ **Storing too much** — Memory should be small, reference data (not caches)
- ✅ **Use sensible keys** — "facility/colorspace" is better than "config_1"

---

## 10. Event Systems & Callbacks

### What It Does
Push real-time notifications from {{DCC}} to Claude. Examples:
- "User created a node" → push `node_created` event
- "Parameter changed" → push `knob_changed` event
- "Frame changed" → push `frame_changed` event

Claude can react in real-time (e.g., "I see you created a Read node, want me to set the path?").

### Where to Find It
**Research your DCC's event system:**
- Does it have callbacks or observers? (most DCCs do)
- What events are available?
- How do you register for notifications?

**For your DCC specifically:**
- **Nuke** — nuke.addKnobChanged(), nuke.root().addCallback()
- **Blender** — bpy.app.handlers (on_frame_change, on_save, on_load)
- **Houdini** — hou_session callbacks (node, hip file, playbar events)
- **Maya** — cmds.scriptJob(), OpenMaya.MEventMessage

### Code Pattern

```python
# events.py (already templated!)
# Use templates/advanced/events.py.tmpl

# It provides:
# - EventLog (ring buffer of 100 events)
# - subscribe_events(types) — subscribe to event types
# - get_recent_events(limit) — fetch recent events
# - push_event(event_type, data) — push from addon

# TODO: Register callbacks in addon:

def _setup_event_callbacks():
    """Register callbacks for {{DCC}} events."""
    import {{API_MOD}}
    
    # Example (Nuke):
    # def on_knob_changed(knob):
    #     from {{dcc}}mcp.events import push_event, EventType
    #     node = knob.node()
    #     push_event(EventType.KNOB_CHANGED, {
    #         "node": node.name(),
    #         "knob": knob.name(),
    #         "value": knob.value()
    #     })
    # nuke.root().addCallback(on_knob_changed)
    
    # Example (Blender):
    # @bpy.app.handlers.frame_change_post
    # def on_frame_changed(scene):
    #     from {{dcc}}mcp.events import push_event, EventType
    #     push_event(EventType.FRAME_CHANGED, {"frame": scene.frame_current})
    
    # Example (Houdini):
    # def on_node_created(event):
    #     node = event.node
    #     from {{dcc}}mcp.events import push_event, EventType
    #     push_event(EventType.NODE_CREATED, {"name": node.name()})
    # hou.session.addNodeCreationCallback(on_node_created)
```

### Reference Repos
- **Nuke** — `nuke-mcp/src/nukemcp/events.py` (91 lines, ring buffer + callbacks)
- **Houdini** — `houdini-mcp/src/houdinimcp/event_collector.py` (deduplication + Houdini callbacks)
- **Templates** — `templates/advanced/events.py.tmpl` (ready-to-use)

### Common Gotchas

- ❌ **Event spam** — If callbacks fire rapidly, deduplicste (collapse 5 changes into 1)
- ✅ **Handle headless** — Headless DCCs may not have all callbacks (e.g., no playbar in hython)

---

## 11. CLAUDE.md Behavioral Rules

### What It Does
Document your DCC's unique constraints, conventions, and workflows for Claude. Helps Claude make better decisions.

Examples:
- "Never name nodes with spaces; use snake_case"
- "Destructive operations require confirm=True"
- "Always set colorspace before importing; it can't be changed after"
- "Memory system is used; read facility conventions on startup"

### Where to Find It
**Reflect on your DCC's best practices:**
- What do experienced users always do first?
- What mistakes are common?
- What constraints are non-obvious?
- What conventions does your facility use?

### Code Pattern

```markdown
# {{PRETTY_DCC}} MCP — Behavioral Guide

## Conventions

### Naming
- Use `snake_case` for all node names (e.g., `read_file_01`, not `ReadFile01`)
- Prefix names with functional type (e.g., `read_`, `adjust_`, `output_`)
- Avoid spaces; some tools don't support them

### Colorspace
- Set colorspace BEFORE importing media (can't change after in {{DCC}})
- Facility standard: `linear` for VFX work, `sRGB` for previewing
- Check facility conventions via memory: `read_memory("facility/colorspace")`

## Destructive Operations

All operations that change state require `confirm=True`:
- Creating nodes
- Deleting nodes
- Modifying parameters (not reading)
- Connecting nodes
- Changing expressions

Pattern:
1. Call tool without confirm → returns action info and message
2. User approves
3. Call tool again with `confirm=True` → executes

Examples:
```
# First call: ask for confirmation
create_node(class="Read", name="my_read")
→ returns: "Please confirm with create_node(..., confirm=True)"

# Second call: execute
create_node(class="Read", name="my_read", confirm=True)
→ returns: {"status": "ok", "result": {...}}
```

## Memory System

The memory system stores persistent data across sessions:
- Facility conventions (colorspace, naming, preferred nodes)
- Corrections log (for TDs to reference)
- Project metadata (frame ranges, aspect ratios, etc.)

Usage:
```
# Read facility conventions on startup
read_memory("facility")
→ returns: {"colorspace": "linear", "naming": "snake_case", ...}

# Log a correction
write_memory("corrections/SH010", {"description": "Fixed bloom in comp"})
```

## Common Gotchas

- ❌ Don't create nodes with spaces in names
- ❌ Don't assume frame ranges start at 0 (usually 1-100, 1001-1100, etc.)
- ❌ Don't set colorspace after importing; it will cause issues
- ✅ Always read facility conventions before setting defaults
- ✅ Use memory system to share context across sessions
```

### Reference Repos
- **Nuke** — `nuke-mcp/CLAUDE.md` (documents destructive patterns, naming, memory usage)
- **Houdini** — `houdini-mcp/CLAUDE.md` (similar patterns, Houdini-specific gotchas)

### Common Gotchas

- ❌ **Forgetting to document gotchas** — Users will hit them anyway
- ✅ **Be specific** — "Don't use spaces in names" is better than "Follow conventions"

---

## 12. BEST_PRACTICES.md — Hard-Won Lessons

### What It Does
Document patterns that separate hobby projects from production tools. Includes:
- Handler organization
- Mock parity strategy
- Testing approach
- Common bugs and fixes
- Performance tips

### Where to Find It
**Reflect on lessons learned:**
- What bug recurred 3+ times?
- What refactor made a big difference?
- What pattern emerged from successful tools?

### Code Pattern

See `templates/shared/BESTPRACTICES.md.tmpl` — it's already populated with:
- Handler organization (tools/ vs addon)
- Undo management (MUTATING_COMMANDS)
- Mock parity requirement
- Testing tiers
- Destructive operation pattern

**You should add:**
- {{DCC}}-specific gotchas (PySide6/2, lazy imports, etc.)
- Performance optimizations (caching, lazy evaluation)
- Troubleshooting guide (common errors and fixes)

### Reference Repos
- **Nuke** — `nuke-mcp/BESTPRACTICES.md` (very detailed)
- **Houdini** — `houdini-mcp/BESTPRACTICES.md`
- **Template** — `templates/shared/BESTPRACTICES.md.tmpl` (already includes handler org + undo)

### Common Gotchas

- ❌ **Forgetting to document patterns** — Future maintainers won't know why code is structured this way
- ✅ **Link to examples** — "See `tools/graph.py:42-60` for pattern"

---

## Summary Checklist

When implementing a new {{DCC}}-MCP, you must manually implement:

- [ ] **1. Domain-specific tools** (8-31 tools covering your DCC's capabilities)
- [ ] **2. Addon lifecycle** (startup, shutdown, GUI integration)
- [ ] **3. Custom helpers** (DCC-specific utilities)
- [ ] **4. Version gating** (which tools available in which versions)
- [ ] **5. RAG/docs** (ingest + search for your DCC's documentation)
- [ ] **6. Mock server** (simulate scene state for tests, if needed)
- [ ] **7. Skills/workflows** (multi-phase procedures unique to your DCC)
- [ ] **8. Discovery & bootstrap** (find install, launch headless, install addon)
- [ ] **9. Memory system** (wire into your workflow)
- [ ] **10. Event system** (register callbacks for your DCC)
- [ ] **11. CLAUDE.md** (document your DCC's rules for Claude)
- [ ] **12. BEST_PRACTICES.md** (add DCC-specific gotchas)

**Expected effort:** 1-2 weeks for 8-16 tool domains + infrastructure, vs. 4-6 weeks starting from scratch.

**The framework accelerates by providing:**
- Architecture and protocol (don't re-invent)
- Test infrastructure (mock fixtures, pytest marks)
- Advanced feature templates (events, memory, RAG, undo, etc.)
- Skill guidance (DCC-API mapping tables, step-by-step workflows)
- Reference implementations (nuke-mcp, blender-mcp, houdini-mcp as exemplars)
