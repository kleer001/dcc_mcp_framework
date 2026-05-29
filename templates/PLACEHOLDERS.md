# Template Placeholder Tokens

All template files use the following substitution tokens. The scaffold-dcc-mcp skill performs literal string substitution during instantiation.

## Core Identity Tokens

| Token | Example | Usage | Notes |
|-------|---------|-------|-------|
| `{{DCC}}` | `Nuke`, `Blender`, `Houdini` | PascalCase, used for class names, module names | Single word; use in `{{DCC}}Connection`, `{{DCC}}MCPServer` |
| `{{dcc}}` | `nuke`, `blender`, `houdini` | Lowercase, used for package name, file names | Single word; used in `src/{{dcc}}mcp/` |
| `{{PRETTY_DCC}}` | `Foundry Nuke`, `Blender`, `SideFX Houdini` | Display name with vendor | Full marketing name |
| `{{API_MOD}}` | `nuke`, `bpy`, `hou`, `NatronEngine` | Python API module to import | Exact module name that users import |

## Configuration Tokens

| Token | Example | Usage | Notes |
|-------|---------|-------|-------|
| `{{PORT}}` | `54321`, `9876`, `55555` | Default TCP port for addon socket | Choose a free port; document it |
| `{{GUI_MARSHAL}}` | `nuke.executeInMainThread(func, args=args)`, `bpy.app.timers.register(callback)` | How to run a function on the main thread | DCC-specific; see reference/threading.md |
| `{{SCENE_NOUN}}` | `script`, `scene`, `composition` | What users call a saved project | Affects documentation wording |
| `{{NODE_NOUN}}` | `node`, `object`, `operator` | What the DCC calls a graph element | Affects error messages and docs |

## Examples

### Nuke

```
{{DCC}} = Nuke
{{dcc}} = nuke
{{PRETTY_DCC}} = Foundry Nuke
{{API_MOD}} = nuke
{{PORT}} = 54321
{{GUI_MARSHAL}} = nuke.executeInMainThread(func, args=args)
{{SCENE_NOUN}} = script
{{NODE_NOUN}} = node
```

### Blender

```
{{DCC}} = Blender
{{dcc}} = blender
{{PRETTY_DCC}} = Blender
{{API_MOD}} = bpy
{{PORT}} = 9333
{{GUI_MARSHAL}} = bpy.app.timers.register(callback)  # callback returns None to unschedule
{{SCENE_NOUN}} = scene
{{NODE_NOUN}} = object
```

### Houdini

```
{{DCC}} = Houdini
{{dcc}} = houdini
{{PRETTY_DCC}} = SideFX Houdini
{{API_MOD}} = hou
{{PORT}} = 9876
{{GUI_MARSHAL}} = QtCore.QTimer.singleShot(0, callback)
{{SCENE_NOUN}} = hip file
{{NODE_NOUN}} = node
```

### Cinema4D (Hypothetical)

```
{{DCC}} = Cinema4D
{{dcc}} = cinema4d
{{PRETTY_DCC}} = Maxon Cinema 4D
{{API_MOD}} = c4d
{{PORT}} = 54322
{{GUI_MARSHAL}} = c4d.threading.RunAsync(func)
{{SCENE_NOUN}} = document
{{NODE_NOUN}} = object
```

## Substitution in Key Files

### pyproject.toml

```toml
[project]
name = "{{dcc}}mcp"
description = "MCP server for {{PRETTY_DCC}}"
```

### connection.py

```python
class {{DCC}}Connection:
    """Manages a TCP socket connection to the {{PRETTY_DCC}} addon."""
    DEFAULT_PORT = {{PORT}}
```

### mock.py

```python
class Mock{{DCC}}State:
    def __init__(self, {{dcc}}_version: str = "1.0"):
        self.{{dcc}}_version = {{dcc}}_version
```

### addon (.py)

```python
def _handshake_data() -> dict:
    {{API_MOD}} = _get_{{dcc}}()
    return {
        "type": "handshake",
        "{{dcc}}_version": {{API_MOD}}.__version__,
        "pid": __import__("os").getpid(),
    }
```

### server.py

```python
from {{dcc}}mcp.connection import {{DCC}}Connection

mcp = FastMCP(
    "{{DCC}}MCP",
    instructions=f"You are connected to {{PRETTY_DCC}}. Use the available tools...",
)
```

### tools/graph.py

```python
@mcp.tool(annotations={"readOnlyHint": True})
def get_scene_info() -> dict:
    """Get information about the current {{PRETTY_DCC}} {{SCENE_NOUN}}.
    
    Returns {{SCENE_NOUN}} name, frame range, and {{NODE_NOUN}} count.
    """
    return conn.send_command("get_scene_info")
```

## Case Sensitivity

- **DCC** — Always PascalCase. `{{DCC}}Connection`, not `{{Dcc}}Connection`
- **dcc** — Always lowercase. `{{dcc}}mcp`, not `{{Dcc}}mcp`
- **Paths** — Use lowercase: `src/{{dcc}}mcp/`, `{{dcc}}_addon/`

## Special Cases

### Multi-Word DCCs

If the DCC name is multi-word (e.g., "Cinema 4D"), use:
- `{{DCC}}` = `Cinema4D` (no space, run together)
- `{{dcc}}` = `cinema4d` (lowercase, no space)
- `{{PRETTY_DCC}}` = `Maxon Cinema 4D` (full name with vendor)

### Version String Format

- **Nuke**: `"17.0v1"` (major.minor + letter code)
- **Blender**: `"3.4.2"` (semantic versioning)
- **Houdini**: `"20.1.123"` (major.minor.patch)

No standard; use the DCC's native format. Document in pyproject.toml example.

## Testing Placeholders

When testing the scaffold skill, use a fictional DCC (e.g., "foo") with dummy values:

```
{{DCC}} = Foo
{{dcc}} = foo
{{PRETTY_DCC}} = Fictional Foo DCC
{{API_MOD}} = foo
{{PORT}} = 55555
{{GUI_MARSHAL}} = foo.run_on_main_thread(func)
{{SCENE_NOUN}} = scene
{{NODE_NOUN}} = object
```

This allows testing the scaffold pipeline without needing a real DCC installed.

## No Partial Substitution

**Important:** Do not partially substitute tokens. Either:
1. All tokens are substituted (normal case)
2. No tokens are substituted (reference docs, this file)

Do NOT create a file with some tokens substituted and others remaining (e.g., `{{DCC}}Connection` in a .tmpl file should not be partially filled).

## File Naming

Template files use `.tmpl` suffix to avoid linting as code:
- `connection.py.tmpl` (not `connection.py`)
- `server.py.tmpl` (not `server.py`)
- `mcp.json.tmpl` (not `mcp.json`)

On substitution, the `.tmpl` suffix is removed.
