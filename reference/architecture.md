# DCC MCP Architecture

## Two-Process Design

The convergent DCC MCP architecture separates concerns into two processes communicating via TCP newline-delimited JSON:

```
┌─────────────┐
│   Claude    │
│   (AI)      │
└──────┬──────┘
       │ stdio JSON-RPC
       │
┌──────▼──────────────────┐
│  MCP Server             │
│  (Python, FastMCP)      │
│  - Tool handlers        │
│  - Connection mgmt      │
│  - Event routing        │
│  - Mock mode support    │
└──────┬──────────────────┘
       │ TCP newline-JSON
       │ (localhost:{{PORT}})
┌──────▼──────────────────┐
│  In-DCC Addon           │
│  (TCP Socket Server)    │
│  - Command dispatch     │
│  - Queue + timer        │
│  - Main-thread marshal  │
│  - Event push           │
└──────┬──────────────────┘
       │ direct API calls
       │
┌──────▼──────────────────┐
│  DCC Python API         │
│  (nuke/bpy/hou/etc.)    │
│  - GUI thread unsafe    │
│  - Requires marshalling │
└─────────────────────────┘
```

## Component Responsibilities

### MCP Server

- **Connection management** — Connect with exponential-backoff retry, read handshake, manage reconnects
- **Tool registration** — Register annotated tools (e.g., `create_node`, `delete_node` with `confirm=True`)
- **Command dispatch** — Route tool calls to addon via `send_command(type, params)`
- **Mock mode** — When `--mock` is used, spins up a mock socket server instead of connecting to a real addon
- **Event handling** — Receives pushed events from addon (e.g., `node_created`, `knob_changed`) and routes to handlers

### Addon (In-DCC)

- **Socket server** — Listens on `localhost:{{PORT}}` for JSON commands
- **Handshake** — On client connect, sends `{"type":"handshake","{{dcc}}_version":"...","variant":"...","pid":...}`
- **Command dispatch** — Routes `{"type":"cmd_name","params":{...}}` to handler function `_handle_cmd_name(params)`
- **Main-thread marshalling** — Queues handler calls to the DCC's main thread (GUI thread is not thread-safe)
- **Response format** — Returns `{"status":"ok"|"error","result":{...},"error":"..."}`
- **Event push** — When events occur in the DCC (node created, script saved), sends `{"type":"event","event_type":"...","data":{...}}`

### Connection Class

- **Framing** — Read/write newline-delimited JSON
- **Reader thread** — Separate background thread that reads all incoming messages and routes:
  - Responses (`status` + `result`/`error`) → response queue
  - Events (`type=="event"`) → event handler callback
- **Timeout handling** — Raises on timeout waiting for response
- **Reconnect** — On socket close, connection goes down; client must retry

## Lifecycle

### Startup (GUI Mode)

1. User starts DCC (Nuke, Blender, Houdini, etc.)
2. DCC loads addon on init (e.g., `nuke_mcp_addon.start()`)
3. Addon creates `{{DCC}}MCPServer` and calls `server.start()` (background thread)
4. Addon socket server listens on `localhost:{{PORT}}`
5. MCP client connects to addon, reads handshake (version, variant, PID)
6. MCP client opens reader thread to separate responses from events
7. Claude can now issue tool calls → server sends commands → addon dispatches

### Startup (Headless Mode)

1. MCP client launches DCC subprocess in headless mode (no GUI)
2. Addon starts and calls `server.serve_forever()` instead of `server.start()`
3. `serve_forever()` spawns socket listener in background thread
4. Main thread drains the command queue (GUI calls run sequentially on main thread)
5. Same connection protocol as GUI mode; no timer needed (main thread handles the queue)

### Startup (Mock Mode)

1. MCP client calls `build_server(mock=True)`
2. Server spawns `MockNukeServer` (simulates addon without real DCC)
3. Mock server listens on same port, responds with mock data
4. Connection protocol identical to real addon
5. Enables offline development, testing, CI/CD

## Envelope Specification

All messages are newline-delimited JSON objects.

### Client → Addon (Command)
```json
{"type":"create_node","params":{"node_class":"Grade","name":"grade1"}}
```

### Addon → Client (Response)
```json
{"status":"ok","result":{"name":"grade1","class":"Grade"}}
```

or on error:
```json
{"status":"error","error":"Node 'grade1' not found"}
```

### Addon → Client (Event)
```json
{"type":"event","event_type":"node_created","data":{"name":"grade1","class":"Grade"}}
```

## Thread Safety

**Problem:** DCC Python APIs (nuke, bpy, hou) are NOT thread-safe. Calls must run on the DCC's main (GUI) thread.

**Solution:** Queue + timer/executor pattern:

1. Socket reader runs on background thread
2. When a command arrives, it queues the handler function
3. GUI thread (via timer) or main thread (via serve_forever) drains the queue
4. Handler executes on GUI/main thread (safe)
5. Result returned via Event or result queue

### GUI Thread (bpy.app.timers, QtCore.QTimer, nuke.executeInMainThread)

Each DCC has a different idiom for scheduling work on the main thread:

- **Nuke** — `nuke.executeInMainThread(func, args)`
- **Blender** — `bpy.app.timers.register(callback)` (callback returns float for reschedule)
- **Houdini** — `QtCore.QTimer.singleShot()` or `hou.scheduleCallback()`
- **Natron** — Similar Qt timer approach

The adapter schedules a periodic callback that drains the command queue one item at a time.

### Headless Thread (serve_forever)

In headless mode, the main thread is free (no GUI). `serve_forever()` calls a queue-draining loop on the main thread:

```python
while running:
    func, args = queue.get(timeout=0.1)  # Block briefly
    result = func(*args)
    result_queue.put(result)
```

No timer needed; no separate GUI thread.

## Event Flow

Optional: addon can push events back to the client. Example lifecycle:

1. Client calls `subscribe_events(event_types=["node_created","node_deleted"])`
2. Addon registers callbacks with the DCC (e.g., `nuke.addOnCreate()`)
3. User creates a node in the GUI
4. DCC fires the callback
5. Callback calls `_push_event("node_created", {...})`
6. Addon sends newline-delimited JSON event to socket
7. Client's reader thread routes to event handler
8. Event handler processes (e.g., logs, updates state, pushes to Claude)

## Mock State

The `MockState` class maintains minimal scene state:
- Nodes dict: `{node_name: {class, xpos, ypos, knobs}}`
- Connections list: `[{output, input, input_index}]`
- Frame range, FPS, colorspace, script name, proxy mode

Commands update state (e.g., `create_node` adds to nodes dict), enabling sequential commands to see coherent state. No file I/O, no actual rendering; tests run instantly offline.

## Extensibility

The architecture supports:

- **Tool domains** — Multiple `tools/domain.py` modules, each with `register(server)` function
- **Annotations** — `@mcp.tool(annotations={"readOnlyHint": True, "destructiveHint": True})`
- **Confirm guards** — Destructive ops require `confirm=True` to proceed
- **Advanced features** — Events, RAG, memory, discovery, plugins as pluggable modules
- **Version gating** — Version/variant from handshake; tools can branch on feature availability

See `reference/advanced-features.md` for plugin details.
