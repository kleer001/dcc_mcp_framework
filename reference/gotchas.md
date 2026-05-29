# Gotchas & Edge Cases

## PySide2 vs PySide6

Different Nuke versions use different PySide versions:
- **Nuke 15** and earlier: PySide2
- **Nuke 16+**: PySide6

Import both with a fallback:

```python
try:
    from PySide6.QtCore import QTimer, Signal
except ImportError:
    from PySide2.QtCore import QTimer, Signal
```

Same applies to Houdini and other Qt-based DCCs.

Avoid PySide2-only or PySide6-only code in shared addon templates.

## Lazy DCC Imports

The addon module must be **importable outside the DCC**. Use lazy imports:

```python
def _get_nuke():
    """Import nuke lazily so the module can be imported for testing."""
    import nuke
    return nuke

def _handle_get_script_info(params):
    nuke = _get_nuke()
    return {"status": "ok", "result": {"name": nuke.root().name()}}
```

**Bad:**
```python
import nuke  # Fails if nuke not installed; blocks testing
```

This is critical for mock testing and CI/CD.

## SO_REUSEADDR

When the addon server starts, bind the socket with `SO_REUSEADDR`:

```python
server_sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
server_sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
server_sock.bind(("127.0.0.1", self.port))
```

Without this, if the server crashes and restarts quickly, you get `"Address already in use"` error (the port remains in TIME_WAIT state for ~30s).

## Daemon Threads & Cleanup

Socket listener threads should be daemon threads:

```python
self._thread = threading.Thread(target=self._serve, daemon=True)
```

This ensures:
- The DCC can exit even if the socket thread is stuck
- No need to join() on shutdown

**Bad:**
```python
self._thread = threading.Thread(target=self._serve)  # Non-daemon; may hang DCC shutdown
```

## Reconnect Logic

The client connection should use exponential backoff:

```python
last_error = None
for attempt in range(retries):
    try:
        self.sock = socket.socket(...)
        self.sock.connect((host, port))
        return
    except OSError as e:
        last_error = e
        delay = base_delay * (2 ** attempt)  # 1s, 2s, 4s, 8s
        time.sleep(delay)
raise ConnectionError(f"Failed after {retries} attempts: {last_error}")
```

This is critical for:
- Headless launch (DCC takes time to start)
- Network hiccups
- User accidentally closing the addon panel

## FastMCP vs mcp[cli] SDK Notes

### FastMCP (Standalone)

```python
from fastmcp import FastMCP
mcp = FastMCP("MyServer")

@mcp.tool()
def my_tool(...): ...

mcp.run()  # Starts stdio server
```

- **Single file** — Can be in any Python environment
- **Port agnostic** — Doesn't care about server port (uses stdio)
- **Simpler** — Fewer dependencies

### mcp[cli] (Official Anthropic SDK)

```python
from mcp.server.fastmcp import FastMCP
mcp = FastMCP("MyServer")

@mcp.tool()
def my_tool(...): ...

if __name__ == "__main__":
    mcp.run()  # Starts stdio server
```

- **Official SDK** — Blessed by Anthropic
- **More integrations** — Better IDE support, official examples
- **Heavier** — More dependencies, more structured

Both expose the same MCP interface (stdio JSON-RPC). The difference is internal.

**Template strategy:** Scaffold skill offers both variants; user chooses during setup.

## Mock Parity

Every tool must have a mock handler:

**Real addon:**
```python
def _handle_create_node(params):
    nuke = _get_nuke()
    node = nuke.createNode(params["node_class"])
    return {"status": "ok", "result": {"name": node.name()}}
```

**Mock:**
```python
def _cmd_create_node(self, params):
    name = f"{params['node_class']}{self._next_id}"
    self._next_id += 1
    self.nodes[name] = {"class": params["node_class"]}
    return {"status": "ok", "result": {"name": name}}
```

If a handler exists in the addon but not in mock, tests will fail in mock mode:

```python
def test_create_node():
    conn = connection_fixture  # Uses mock
    result = conn.send_command("create_node", {"node_class": "Grade"})
    assert result["name"] == "Grade1"  # Fails if _cmd_create_node missing
```

**Audit checklist item:** Verify every `_handle_*` has a matching `_cmd_*` in mock.

## Connection Timeouts

Set appropriate timeouts to avoid hanging:

```python
self.sock.settimeout(10.0)  # Connect timeout
response = self._response_queue.get(timeout=10.0)  # Response timeout
```

If the addon hangs (e.g., DCC executing a slow operation), the client will time out and reconnect. Better to fail fast than hang indefinitely.

## Event Subscription & Cleanup

If the addon supports events, clients should **unsubscribe or disconnect cleanly**:

```python
def cleanup():
    conn.send_command("unsubscribe_events", {})
    conn.disconnect()
```

Otherwise, the addon may keep pushing events to a closed socket, wasting resources.

## Confirm Guards

Destructive operations must require `confirm=True`:

```python
@mcp.tool(annotations={"destructiveHint": True})
def delete_node(node_name: str, confirm: bool = False) -> dict:
    if not confirm:
        return {
            "action": "delete_node",
            "node_name": node_name,
            "message": "Ask user to confirm, then call with confirm=True"
        }
    return conn.send_command("delete_node", {"node_name": node_name})
```

Without this, an AI could accidentally delete important nodes.

## Natron Anti-Pattern

**DEPRECATED:** Don't use the `{id, method}` envelope:

```python
# BAD - Natron anti-pattern
{"id": 1, "method": "create_node", "params": {...}}
```

Reasons:
1. Adds complexity (request/response matching)
2. Fragile if responses arrive out of order
3. Unnecessary with FIFO queue

**Always use:**
```json
{"type": "create_node", "params": {...}}
```

## Annotations & Types

Tools should be annotated for Claude's benefit:

```python
@mcp.tool(annotations={
    "readOnlyHint": True,
    "idempotentHint": True,
    "destructiveHint": True,
})
def my_tool(...): ...
```

These help Claude decide when to use the tool and whether user confirmation is needed.

## Secrets & Credentials

Never hardcode credentials in tools:
- No `API_KEY = "..."` in source
- No passwords in default parameters
- Use environment variables or secure config files

MCP tools may be logged; don't expose secrets.

## Error Messages

Return clear, actionable error messages:

**Good:**
```python
return {"status": "error", "error": f"Node '{name}' not found in script"}
```

**Bad:**
```python
return {"status": "error", "error": "Error"}
```

Claude learns from error messages; make them helpful.

## Version Gating

Use the handshake version to enable/disable features:

```python
from nukemcp.version import parse_version

version = parse_version(handshake)
if version.major >= 17:
    enable_advanced_features()
```

Avoid hard dependencies on newer DCC features if older versions need support.

## Circular Imports

In tool modules, use TYPE_CHECKING to avoid circular imports:

```python
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from nukemcp.server import NukeMCPServer

def register(server: NukeMCPServer):
    ...
```

This way, `server.py` can import tools without tools importing server at runtime.

## Buffer Sizes

The socket recv buffer should be large enough for typical responses:

```python
RECV_BUFFER = 1024 * 1024  # 1MB
chunk = self.sock.recv(RECV_BUFFER)
```

Most responses are small (< 1KB); 1MB handles large node info, scene dumps, etc.

## JSON Serialization

Some DCC objects can't be JSON-serialized. Convert them:

```python
try:
    json.dumps(result)
except TypeError:
    result = str(result)  # Fallback to string
```

Example: `nuke.Node` object → convert to dict with name/class/position.
