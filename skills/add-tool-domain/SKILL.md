# add-tool-domain Skill

## Description

Add a new tool domain module (lighting, rendering, tracking, etc.) to an existing DCC MCP server. Use when the user wants to add tools/commands/a domain to an existing `*-mcp` repo.

## Trigger Patterns

- "Add a lighting domain to my blender-mcp"
- "I want to create a rendering tools module for nuke-mcp"
- "Let's add tracking tools to houdini-mcp"

Do NOT use when scaffolding a new repo (use `scaffold-dcc-mcp`).

## Workflow

### 1. Understand the Repo

Read:
- `tools/_helpers.py` — Extract helper functions (e.g., `send()`, `require_confirm()`)
- One existing tool domain (e.g., `tools/graph.py`) — Match conventions
- `reference/protocol.md` — Understand envelope and command structure
- Addon code — Understand handler pattern and API calls

### 2. Define Tools

Interview the user:
- **Domain name** → `lighting`, `rendering`, `tracking`, etc. (becomes `tools/lighting.py`)
- **Tool list** → Commands to expose (e.g., `create_light`, `set_light_intensity`, `get_lights`)
- **Read-only vs destructive** → Mark each tool

Document each tool:
- Name: `create_light`
- Params: `name: str`, `type: str`
- Returns: `{"name": "...", "type": "..."}`
- Annotation: read-only or destructive?

### 3. Generate `tools/<domain>.py`

Pattern (from reference/protocol.md):

```python
"""Lighting tools."""

def register(server):
    mcp = server.mcp
    conn = server.connection
    
    @mcp.tool(annotations={"readOnlyHint": True})
    def get_lights() -> dict:
        """Get all lights in the scene."""
        return conn.send_command("get_lights")
    
    @mcp.tool(annotations={"destructiveHint": True})
    def delete_light(name: str, confirm: bool = False) -> dict:
        """Delete a light."""
        if not confirm:
            return {"message": "Ask user, then call with confirm=True"}
        return conn.send_command("delete_light", {"name": name})
```

For each tool:
- Read-only → `annotations={"readOnlyHint": True}`
- Destructive → `annotations={"destructiveHint": True}` + `confirm=False` param
- Document return shape

### 4. Add Addon Handlers

For each command in the domain, add addon handler (sketch):

```python
def _handle_get_lights(params: dict) -> dict:
    dcc = _get_dcc()
    lights = [l.name() for l in dcc.get_lights()]
    return {"status": "ok", "result": {"lights": lights}}

def _handle_delete_light(params: dict) -> dict:
    dcc = _get_dcc()
    name = params["name"]
    light = dcc.get_light(name)
    if not light:
        return {"status": "error", "error": f"Light '{name}' not found"}
    dcc.delete(light)
    return {"status": "ok", "result": {"deleted": name}}
```

Add to `COMMANDS` dispatch table in addon.

### 5. Add Mock Handlers

For each addon handler, add mock handler:

```python
def _cmd_get_lights(self, params: dict) -> dict:
    lights = [{"name": n, "type": self.lights[n]} for n in self.lights]
    return {"status": "ok", "result": {"lights": lights}}

def _cmd_delete_light(self, params: dict) -> dict:
    name = params["name"]
    if name not in self.lights:
        return {"status": "error", "error": f"Light '{name}' not found"}
    del self.lights[name]
    return {"status": "ok", "result": {"deleted": name}}
```

Initialize mock state fields (e.g., `self.lights = {}`).

### 6. Wire Into `server.py`

Add import and registration:

```python
from {{dcc}}mcp.tools import lighting

# In build_server():
lighting.register(server)
```

### 7. Add Tests

Create `tests/test_lighting.py`:

```python
import pytest

def test_get_lights(connection):
    result = connection.send_command("get_lights")
    assert result["lights"] is not None

def test_delete_light_requires_confirm(connection):
    result = connection.send_command("create_light", {"name": "light1"})
    # Try to delete without confirm
    result = connection.send_command("delete_light", {"name": "light1", "confirm": False})
    assert "confirm=True" in result.get("message", "")

def test_delete_light_with_confirm(connection):
    connection.send_command("create_light", {"name": "light1"})
    result = connection.send_command("delete_light", {"name": "light1", "confirm": True})
    assert result["deleted"] == "light1"
```

### 8. Verify

```bash
pytest tests/test_lighting.py -v
ruff check tools/lighting.py
```

All tests pass in mock mode.

## Checklist

- [ ] Tool definitions documented (name, params, return, annotation)
- [ ] Addon handlers added (one `_handle_*` per command)
- [ ] Mock handlers added (one `_cmd_*` per handler)
- [ ] Mock state updated (e.g., `self.lights = {}`)
- [ ] Import + register in `server.py`
- [ ] Tests written and pass
- [ ] Ruff clean
- [ ] Annotations correct (readOnlyHint, destructiveHint)

## Related Skills

- `scaffold-dcc-mcp` — Created this repo
- `add-advanced-feature` — Add events, RAG, etc.
- `audit-dcc-mcp` — Review for conformance
