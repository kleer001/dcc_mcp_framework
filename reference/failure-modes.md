# Failure Modes: What If Your DCC Doesn't Fit the Pattern?

The DCC MCP Framework assumes your DCC has:
- Python API (or equivalent scripting language)
- Plugin/addon system
- Main thread for API calls
- Some way to receive commands

If your DCC is missing one of these, here's what to do.

---

## Failure Mode 1: DCC Has No Python API

**Problem:** Your DCC is proprietary, closed-source, or only has C++ SDK.

**Examples:** Some old tools, some commercial software with no scripting layer.

**Solutions (in priority order):**

### Option A: Use HTTP/REST Bridge (Recommended)
Instead of Python, write a simple HTTP server in the DCC's native language:

```python
# addon_bridge.cs (C#, or whatever the DCC uses)
// Listen on localhost:{{PORT}}, accept JSON over HTTP
// Dispatch to native DCC APIs
// Return JSON responses

// Example (pseudo-code):
public class MCPBridge {
    void HandleRequest(string jsonCommand) {
        if (jsonCommand["type"] == "create_node") {
            var node = DCC_API.CreateNode(jsonCommand["params"]["class"]);
            return JsonResponse("ok", {"name": node.Name});
        }
    }
}
```

**From MCP side:** Unchanged. Connection.py makes socket calls; addon speaks HTTP instead of Python.

**Claude's role:** Implement the HTTP bridge in the DCC's native language, following the same command/response pattern.

### Option B: Use IPC/Message Queue
If HTTP is overkill, use named pipes, sockets, or message queues:
- **Windows:** Named pipes (`\\.\pipe\{{dcc}}-mcp`)
- **Unix:** Unix domain sockets (`/tmp/{{dcc}}-mcp.sock`)
- **Any OS:** Message queue (ZMQ, RabbitMQ)

### Option C: Headless CLI Only
If the DCC has no interactive plugin system, just support headless mode:
- User runs: `{{dcc}} --batch --mcp-server`
- No addon, just bootstrap script that launches DCC in scripting mode
- All commands execute in headless batch context

**Examples:** Some rendering engines, some data processors.

### Option D: Reduce Scope to Read-Only
If DCC truly has no scripting:
- Scaffold MCP as read-only (queries only)
- Tools: `list_*`, `get_*`, `search_*`
- No mutations, no state changes
- Still useful for inspection, data export

---

## Failure Mode 2: DCC Has No Plugin System

**Problem:** No addon/plugin architecture. Can't load code at startup.

**Examples:** Some legacy tools, some single-purpose apps.

**Solutions:**

### Option A: User Manual Startup
User must manually start MCP server in the DCC:

```python
# User runs this in DCC's script console:
import {{dcc}}_mcp
server = {{dcc}}_mcp.start()
# Server now listening on port {{PORT}}
```

**from MCP side:** No change. Claude Code just tells user "Start the server manually."

**Addon changes:** Remove plugin/startup code. Just provide a standalone `start()` function.

### Option B: Wrapper Script/Launcher
Create a launcher that:
1. Starts DCC
2. Injects code (via stdin, script args, or environment)
3. Starts MCP server

```bash
# {{dcc}}-mcp-launcher.sh
#!/bin/bash
{{DCC_BIN}} --script - <<'EOF'
import {{dcc}}_mcp
{{dcc}}_mcp.start()
EOF
```

### Option C: Scripted Batch Mode Only
If no interactive mode, operate headless only:

```bash
{{dcc}} --batch --script mcp_server.py
```

User launches this once at session start; MCP server runs for the session.

---

## Failure Mode 3: DCC Has No Main Thread / Not Thread-Safe

**Problem:** DCC API is async or doesn't support main-thread marshalling.

**Examples:** Some cloud/SaaS tools, some web-based editors.

**Solutions:**

### Option A: Async/Await Pattern
If DCC supports async APIs, use them:

```python
# addon.py (async version)

async def _handle_create_node(params):
    """Async handler."""
    dcc = _get_dcc()
    node = await dcc.create_node_async(params["class"])
    return {"status": "ok", "result": {"name": node.name}}

# Server processes async handlers:
async def _process_command(data):
    handler = COMMANDS.get(data["type"])
    result = await handler(params)
    return result
```

### Option B: Queue with Timeout
If sync but not main-thread-safe, use queue with generous timeout:

```python
# addon.py
def serve_forever(self):
    """Process queue with longer timeout for slow DCCs."""
    while self._running:
        try:
            func, args, result_q = self._main_queue.get(timeout=5.0)  # ← longer timeout
            result = func(*args)
            result_q.put(result)
        except queue.Empty:
            continue
```

### Option C: Fire-and-Forget + Polling
If DCC can't block for results, use polling:

```python
# addon.py sends command ID, doesn't wait for result

def _handle_create_node(params):
    cmd_id = uuid.uuid4()
    # Schedule async execution
    schedule_for_later(cmd_id, lambda: dcc.create_node(...))
    # Return immediately with ID
    return {"status": "pending", "command_id": cmd_id}

# Client polls for result:
# → send_command("get_command_result", {"command_id": "..."})
# → returns {"status": "ok", "result": {...}} or {"status": "pending"}
```

---

## Failure Mode 4: DCC Has Minimal or Slow API

**Problem:** DCC API is slow, incomplete, or only exposes high-level operations.

**Examples:** Some SaaS tools, some specialized renderers.

**Solutions:**

### Option A: Reduce Tool Scope
Not every DCC needs 20+ tools. Scaffold minimal:

```python
# Minimal tool set for constrained DCCs:
tools/
  core.py       ← ping, status, version
  scene.py      ← get_scene_info, list_objects (if possible)
  
# Omit:
# - create/delete (if API doesn't expose it)
# - modify (if API is read-only or high-level only)
# - connect (if no node graph)
```

**Mock parity:** Same 2-4 tools, same mock handlers. Tests still pass.

### Option B: Wrapper Functions
If API is high-level, wrap it into lower-level operations:

```python
# Example: DCC only exposes "render(scene_name)"
# Wrap it into get/set operations:

def get_render_settings() -> dict:
    """Expose scene parameters as dict."""
    return dcc.get_scene_metadata()

def set_render_settings(settings: dict) -> dict:
    """Apply scene parameters."""
    for key, value in settings.items():
        dcc.set_scene_param(key, value)
    return {"status": "ok"}
```

### Option C: Document Limitations
If DCC truly can't do something, document it:

```markdown
# {{PRETTY_DCC}}-MCP Limitations

This MCP exposes read-only operations because {{PRETTY_DCC}}'s API does not support mutations.

Supported:
- list_objects()
- get_object_info()
- get_scene_metadata()

Not supported:
- create_object (API limitation)
- modify_object (API limitation)
- delete_object (API limitation)

Workaround: Use {{PRETTY_DCC}}'s native UI for mutations; use MCP for querying.
```

---

## Failure Mode 5: DCC Has No Telemetry / Event System

**Problem:** Can't push real-time notifications. No callbacks, no observer pattern.

**Examples:** Some stateless web tools, some legacy CLIs.

**Solutions:**

### Option A: Skip Event System
Just don't implement `events.py`. Optional feature.

```python
# server.py (no event registration)
from {{dcc}}mcp import events  # Optional
# Only register if DCC supports callbacks:
if hasattr(dcc, 'register_callback'):
    events.register(server)
else:
    log.info("Event system not supported by this DCC")
```

### Option B: Polling-Based Events
If no callbacks, Claude can poll:

```python
# tools/polling.py
@mcp.tool(annotations={"readOnlyHint": True})
def get_scene_changes(since_timestamp: str) -> dict:
    """Get changes since timestamp (polling-based event system)."""
    changes = dcc.get_changes_since(since_timestamp)
    return {"changes": changes, "timestamp": datetime.now().isoformat()}

# Claude calls this periodically instead of receiving push events
```

### Option C: Batch Queries
Expose state snapshots Claude can compare:

```python
@mcp.tool(annotations={"readOnlyHint": True})
def snapshot_scene() -> dict:
    """Get full scene state (for comparison-based change detection)."""
    return {
        "objects": dcc.list_objects(),
        "parameters": dcc.get_all_parameters(),
        "timestamp": datetime.now().isoformat()
    }
```

---

## Failure Mode 6: DCC Runs Remotely / Cloud-Based

**Problem:** DCC is not on local machine. Can't install addon locally.

**Examples:** Cloud renderers, SaaS tools (Figma, Blender Cloud, etc.).

**Solutions:**

### Option A: API Bridge
If DCC has HTTP/REST API, use that:

```python
# addon.py (cloud version)
class CloudMCPBridge:
    def __init__(self, api_endpoint, api_key):
        self.endpoint = api_endpoint  # https://dcc.cloud/api
        self.api_key = api_key
    
    def _handle_create_node(self, params):
        response = requests.post(
            f"{self.endpoint}/nodes",
            json=params,
            headers={"Authorization": f"Bearer {self.api_key}"}
        )
        return response.json()
```

**Setup:** User provides API endpoint + API key instead of local addon.

### Option B: Authenticate + Forward
MCP server authenticates with cloud DCC and forwards commands:

```python
# server.py (cloud version)
class CloudMCPServer:
    def __init__(self, api_endpoint, user_token):
        self.api = CloudAPI(api_endpoint, user_token)
    
    def _process_command(self, data):
        # Forward to cloud
        return self.api.execute_command(data["type"], data["params"])
```

**Claude's role:** Handle authentication, error codes, rate limiting (all standard HTTP stuff).

### Option C: WebSocket for Real-Time
If cloud DCC supports WebSocket, use that instead of TCP:

```python
# connection.py (WebSocket variant)
import asyncio
import websockets

async def connect(self):
    self.ws = await websockets.connect(f"wss://{self.host}:{self.port}")

async def send_command(self, cmd_type, params):
    await self.ws.send(json.dumps({"type": cmd_type, "params": params}))
    response = await self.ws.recv()
    return json.loads(response)
```

---

## Failure Mode 7: DCC is Proprietary but Has HTTP API

**Problem:** DCC has no Python, but exposes REST or HTTP interface.

**Examples:** Some commercial software, some game engines.

**Solutions:**

### Option A: HTTP Bridge (Simplest)
Replace Python addon with HTTP client:

```python
# addon.py (HTTP variant)
class HTTPMCPBridge:
    def __init__(self, dcc_http_port):
        self.base_url = f"http://localhost:{dcc_http_port}"
    
    def _handle_create_node(self, params):
        response = requests.post(
            f"{self.base_url}/api/nodes",
            json=params
        )
        return response.json()

def start():
    bridge = HTTPMCPBridge(dcc_http_port=8080)
    # Wrap bridge to match MCP socket protocol
    mcp_server = MCPSocketServer(bridge)
    mcp_server.start()
```

**Claude's role:** Adapt socket protocol layer to HTTP client calls. Not a big deal.

---

## Failure Mode Checklist

When scaffolding a new DCC, ask:

```
Does your DCC have...
☐ Python (or equivalent scripting language)?
  If no → Use HTTP/REST bridge (Failure Mode 1, Option A)

☐ Plugin/addon system?
  If no → Use manual startup or launcher script (Failure Mode 2)

☐ Main thread or thread-safe API?
  If no → Use async/await or polling (Failure Mode 3)

☐ Rich API (not read-only)?
  If no → Reduce tool scope or document limitations (Failure Mode 4)

☐ Event callbacks or telemetry?
  If no → Use polling or skip events (Failure Mode 5)

☐ Runs locally?
  If no → Use cloud API bridge (Failure Mode 6)

☐ HTTP/REST API?
  If yes → Use HTTP bridge (Failure Mode 7, Option A)
```

If you check all boxes → Standard scaffold (this framework).
If you check none → MCP probably doesn't make sense for this DCC.
If you check some → Mix and match solutions above.

---

## When MCP Doesn't Make Sense

Some tools are **not good candidates for MCP**, no matter what:

- **CLI-only tools** with no API → Read-only snapshot tool at best
- **Single-command tools** (convert image format, transcode video) → MCP overkill
- **Stateless tools** with no scene/document concept → No state to manipulate
- **Tools that must run isolated** (VFX renderer) → Better as subprocess, not MCP

**In these cases:** Tell Claude Code "This DCC is not a good fit for MCP because [reason]. Consider [alternative] instead."

---

## The Philosophy

The framework doesn't pre-fill addon skeletons for specific DCCs because:
1. It defeats the purpose of being generalized
2. Every DCC's plugin system is different
3. Claude Code can learn the pattern and adapt it

What the framework **does** provide:
- **Patterns** (queue-based, timer-based, event-based dispatch)
- **Failure modes** (what to do if DCC doesn't fit)
- **Reference implementations** (Nuke, Blender, Houdini as examples)
- **Guidance** (Domain Knowledge Checklist)

Claude Code figures out the rest. And it's good at it.
