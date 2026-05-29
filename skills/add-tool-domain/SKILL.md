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

### 2.5. Adapt Tool Implementations (DCC-API Mapping)

Before implementing addon handlers, understand your DCC's Python API for the domain:

**Example: Lighting Domain**

| Operation | Nuke | Blender | Houdini | Maya |
|-----------|------|---------|---------|------|
| **Get all lights** | `[n for n in nuke.allNodes() if n.Class()=="Light"]` | `[o for o in bpy.data.objects if o.type=="LIGHT"]` | `[n for n in hou.node("/obj").children() if n.type().name()=="light"]` | `cmds.ls(type="light")` |
| **Create light** | `nuke.createNode("Light")` | `bpy.ops.object.light_add()` | `hou.node("/obj").createNode("light")` | `cmds.createNode("light")` |
| **Set intensity** | `node["intensity"].setValue(v)` | `obj.data.energy = v` | `node.parm("intensity").set(v)` | `cmds.setAttr("light.intensity", v)` |
| **Get position** | `node["xpos"]()` (2D coords) | `obj.location[:]` (3D) | `node.parmTuple("t")[:]` (3D) | `cmds.xform("light", q=True, t=True)` |

**Implementation pattern:**

1. Choose a reference tool domain from an existing repo that matches your DCC
   - Example: Using nuke-mcp/tools/lighting.py as reference for your Blender MCP
2. Read through the tool signatures and return shapes
3. Adapt addon handlers to call your DCC's API instead of Nuke's
4. Keep mock handlers unchanged (they're DCC-independent)

**Example: Blender lighting adapted from Nuke pattern**
```python
# Reference (from nuke-mcp/tools/lighting.py):
def get_lights() -> dict:
    result = conn.send_command("get_lights")
    return result

# Addon implementation (blender-specific):
def _handle_get_lights(params: dict) -> dict:
    dcc = _get_bpy()
    lights = [
        {"name": obj.name, "type": obj.data.type}
        for obj in dcc.data.objects
        if obj.type == "LIGHT"
    ]
    return {"status": "ok", "result": {"lights": lights}}

# Mock implementation (unchanged):
def _cmd_get_lights(self, params: dict) -> dict:
    lights = [{"name": n, "type": self.lights[n]} for n in self.lights]
    return {"status": "ok", "result": {"lights": lights}}
```

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
