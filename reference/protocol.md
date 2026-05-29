# DCC MCP Protocol Specification

## Overview

The protocol is **newline-delimited JSON** over TCP. All messages are complete JSON objects terminated by `\n`.

- **Framing** — Newline (`\n`) separator
- **Serialization** — JSON (UTF-8)
- **Order** — Handshake, then command/response pairs and events
- **Async** — Responses and events can arrive in any order (reader thread sorts them)

## Connection Handshake

When a client connects to the addon socket, the addon **immediately sends** a handshake message:

```json
{"type":"handshake","{{dcc}}_version":"17.0v1","variant":"NukeX","pid":12345}
```

Fields:
- `type` — Always `"handshake"`
- `{{dcc}}_version` — DCC version string (e.g., "17.0v1", "3.4.2", "20.1")
- `variant` — Optional variant (e.g., "NukeX", "NukeStudio", "Blender Cycles", "Houdini FX")
- `pid` — Process ID of the DCC (useful for debugging, launching headless instances)

The client reads this handshake **before sending any commands**. If handshake parsing fails or socket closes before handshake, connection fails.

## Command/Response Pairs

After handshake, client sends commands; addon sends responses.

### Request Format

```json
{"type":"<command_name>","params":{...}}
```

Fields:
- `type` — Command name (e.g., `"create_node"`, `"get_script_info"`, `"delete_node"`)
- `params` — Object with command-specific parameters (can be empty `{}`)

Example:
```json
{"type":"create_node","params":{"node_class":"Grade","name":"grade1","knobs":{"input":0.5}}}
```

### Response Format (Success)

```json
{"status":"ok","result":{...}}
```

Fields:
- `status` — Always `"ok"` on success
- `result` — Any JSON-serializable value (dict, list, string, number, null)

Example:
```json
{"status":"ok","result":{"name":"grade1","class":"Grade","xpos":100,"ypos":50}}
```

### Response Format (Error)

```json
{"status":"error","error":"<error message>"}
```

Fields:
- `status` — Always `"error"`
- `error` — Human-readable error string

Example:
```json
{"status":"error","error":"Node 'grade1' not found"}
```

**Important:** The response includes **only** `error`, not `result`. Clients must check `status` before reading `result` or `error`.

## Event Messages

The addon can push **unsolicited event messages** to the client (e.g., when a user creates a node in the GUI).

```json
{"type":"event","event_type":"<event_type>","data":{...}}
```

Fields:
- `type` — Always `"event"`
- `event_type` — Event type (e.g., `"node_created"`, `"knob_changed"`, `"script_saved"`)
- `data` — Event-specific payload

Example:
```json
{"type":"event","event_type":"node_created","data":{"name":"grade1","class":"Grade"}}
```

Events are **optional**. Addon can send them only if client has subscribed via `subscribe_events(event_types=[...])`.

## Message Ordering & Reader Thread

The **reader thread** reads all incoming messages and routes them:

1. If `type == "event"` → pass to event handler callback
2. Otherwise → put in response queue (matched to the request that triggered it)

Responses are **matched to requests** by arrival order (FIFO). The first response received is for the first command sent.

## Timeouts

- **Connect timeout** — Typically 10 seconds
- **Response timeout** — Typically 10 seconds per command
- **Handshake timeout** — Same as response timeout

If a response doesn't arrive within the timeout, raise `ConnectionError`. The connection is considered broken and should be re-established.

## Anti-Pattern: The Natron Envelope

**Do NOT use this pattern:**

```json
{"id":1,"method":"create_node","params":{...}}
```

This `{id,method}` envelope was used in `natron-mcp` but is **deprecated**. The `id` field adds complexity (request/response matching) that is unnecessary with a FIFO response queue.

**Use the pattern above** (simple `{type,params}` envelope) for all new code.

## Reconnection & Broken Pipe

If the client detects a broken socket (recv returns empty, send raises OSError):

1. Close the socket
2. Wait exponential backoff (2s, 4s, 8s, 16s)
3. Try to reconnect
4. Read handshake again
5. Retry the command

The addon does **not** persist state across client disconnects. All state is in memory (nodes dict, connections, frame range, etc.).

## Connection Settings

- **Host** — Typically `localhost` (127.0.0.1); rarely remote
- **Port** — Default `{{PORT}}` (e.g., 54321 for Nuke); configurable
- **SO_REUSEADDR** — Set on server socket to allow quick restart (`socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)`)
- **TCP_NODELAY** — Not set; defaults are acceptable for this use case
- **Buffer size** — Typically 1MB for recv; sufficient for large responses

## Framing Error Handling

If the client receives invalid JSON:

```json
{"type":"foo",invalid json
```

The client should:
1. Log the error
2. Skip the incomplete message
3. Try to read the next message

However, if framing is corrupted (missing newlines, binary garbage), the connection is probably broken. Better to reconnect.

## Command Naming Convention

Command names use `snake_case` (e.g., `create_node`, `get_script_info`, `execute_python`, `find_nodes_by_type`).

## Annotations for Tool Safety

Tool definitions can include metadata annotations:

```python
@mcp.tool(annotations={
    "readOnlyHint": True,           # Does not modify state
    "idempotentHint": True,         # Safe to call multiple times
    "destructiveHint": True,        # Destructive; requires confirm=True
})
def delete_node(node_name: str, confirm: bool = False) -> dict:
    ...
```

These are hints to Claude; the tool itself enforces safety (e.g., `confirm` parameter).

## Extensibility

- **New commands** — Add `_handle_<command_name>` to addon and register in tools
- **New events** — Add to `EVENTS` dict or on-demand subscription
- **Variants** — Use handshake `variant` field to branch behavior (e.g., NukeX-specific features)
- **Version gating** — Use `{{dcc}}_version` to enable/disable features

See `reference/advanced-features.md` for advanced usage (events, RAG, memory, etc.).
