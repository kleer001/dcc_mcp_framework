# Headless Patterns: DCC-Specific Main-Thread Dispatch

When running a DCC without a GUI (headless mode), the MCP socket server still needs to execute commands on the DCC's main thread. Different DCCs provide different threading APIs; this document covers the three most common patterns.

## The Problem

The MCP socket server runs on a background thread (daemon thread). But DCC APIs are **not thread-safe**—all calls must execute on the main thread. This creates a coordination problem: how do we safely dispatch socket commands from the background thread to the DCC main thread and return results?

## Three Patterns

### Pattern 1: Queue-Based Dispatch (Nuke)

**Used by:** Nuke  
**Threading model:** Nuke has no built-in main-thread callback system; instead, user code manually polls the event queue.

**How it works:**
```
[Background thread]
  └─ Socket listener accepts commands
     └─ Pushes (func, args, result_queue) into main_queue

[Main thread]
  └─ serve_forever() loop:
     1. Get item from main_queue (blocking, timeout 0.1s)
     2. Execute func(*args) on main thread
     3. Put result into result_queue
     4. Repeat
```

**Advantages:**
- No external dependencies (pure stdlib threading)
- Deterministic: main thread stays in control
- Works in both headless and GUI modes

**Disadvantages:**
- Requires explicit serve_forever() loop in user code
- Manual polling; not event-driven

**Implementation (addon.py):**
```python
import queue
import threading

class {{DCC}}MCPServer:
    def __init__(self):
        self._main_queue = queue.Queue()
        self._listener_thread = None
        self._running = False

    def start(self):
        """Start background socket listener."""
        self._running = True
        self._listener_thread = threading.Thread(target=self._socket_listen, daemon=True)
        self._listener_thread.start()

    def serve_forever(self):
        """Main thread drains command queue (blocking)."""
        self._main_queue = queue.Queue()
        self.start()  # Start background listener
        while self._running:
            try:
                func, args, result_q = self._main_queue.get(timeout=0.1)
                result = func(*args)
                result_q.put(result)
            except queue.Empty:
                continue
            except Exception as e:
                result_q.put({"status": "error", "error": str(e)})

    def _socket_listen(self):
        """Background thread: accept socket commands, queue for main thread."""
        # Listen for commands, parse JSON, create work items
        # Push (handler_func, params, result_q) to self._main_queue
```

**Headless launch:**
```bash
python -c "
import {{dcc}}_mcp_addon
server = {{dcc}}_mcp_addon.{{DCC}}MCPServer()
server.serve_forever()
"
```

---

### Pattern 2: Timer-Based Callback (Blender)

**Used by:** Blender  
**Threading model:** Blender provides `bpy.app.timers` for scheduling callbacks on the main thread.

**How it works:**
```
[Background thread]
  └─ Socket listener accepts commands
     └─ Pushes (func, args, result_queue) into command_queue

[Main thread]
  └─ bpy.app.timers.register(poll_queue)
     └─ Every ~0.01s, poll_queue() wakes up:
        1. Get item from command_queue (non-blocking)
        2. Execute func(*args) on main thread
        3. Put result into result_queue
        4. Return 0.01 (reschedule in 10ms)
```

**Advantages:**
- Blender handles rescheduling; no explicit loop
- Integrates with Blender's event loop (draws while commands execute)
- Works in GUI mode (no blocking)

**Disadvantages:**
- Requires Blender's bpy module (headless mode needs special setup)
- Latency: commands execute up to 10ms later
- Harder to test without Blender running

**Implementation (addon.py):**
```python
import bpy
import queue

class BlenderMCPServer:
    def __init__(self):
        self._command_queue = queue.Queue()

    def start(self):
        """Start background socket listener + main thread timer."""
        self._listener_thread = threading.Thread(target=self._socket_listen, daemon=True)
        self._listener_thread.start()
        # Register timer callback
        bpy.app.timers.register(self._poll_queue)

    def _poll_queue(self):
        """Called every ~10ms by Blender's event loop."""
        try:
            func, args, result_q = self._command_queue.get_nowait()
            result = func(*args)
            result_q.put(result)
        except queue.Empty:
            pass
        except Exception as e:
            result_q.put({"status": "error", "error": str(e)})
        return 0.01  # Reschedule in 10ms
```

**Headless launch:**
```bash
# Blender headless requires special setup
blender --background --python-use-system-env --python - <<'EOF'
import bpy
import blender_mcp_addon
server = blender_mcp_addon.BlenderMCPServer()
server.start()
# Keep running
import time
while True:
    time.sleep(0.1)
EOF
```

---

### Pattern 3: Event-Loop Integration (Houdini)

**Used by:** Houdini  
**Threading model:** Houdini provides `hou.session.inBackground` to detect headless mode and `HOM` event-driven API.

**How it works:**
```
[Background thread]
  └─ Socket listener accepts commands
     └─ Pushes (func, args, result_queue) into event_queue

[Main thread (Houdini HOM)]
  └─ hou.session registers event callback
     └─ When command event fires:
        1. Get item from event_queue
        2. Execute func(*args) in HOM context
        3. Put result into result_queue
```

**Advantages:**
- Houdini handles event dispatch natively
- Works in both GUI and headless modes (same code path)
- Integrates with Houdini's undo/redo system

**Disadvantages:**
- Houdini-specific; not portable to other DCCs
- Requires understanding Houdini's HOM event model
- Complex for users unfamiliar with Houdini

**Implementation (addon.py):**
```python
import hou
import queue

class HoudiniMCPServer:
    def __init__(self):
        self._command_queue = queue.Queue()

    def start(self):
        """Start background listener + Houdini event registration."""
        self._listener_thread = threading.Thread(target=self._socket_listen, daemon=True)
        self._listener_thread.start()
        # Register Houdini event callback (pseudo-code)
        hou.session.addEventListener(self._on_mcp_command)

    def _on_mcp_command(self):
        """Called when command event fires."""
        while not self._command_queue.empty():
            try:
                func, args, result_q = self._command_queue.get_nowait()
                result = func(*args)
                result_q.put(result)
            except queue.Empty:
                break
            except Exception as e:
                result_q.put({"status": "error", "error": str(e)})
```

**Headless launch:**
```bash
hython -c "
import hou
import houdini_mcp_addon
server = houdini_mcp_addon.HoudiniMCPServer()
server.start()
# Houdini's event loop keeps running
"
```

---

## Comparison Table

| Aspect | Queue (Nuke) | Timer (Blender) | Event (Houdini) |
|--------|--------------|-----------------|-----------------|
| **Setup complexity** | Simple | Medium | Complex |
| **Latency** | <1ms | 10ms | <1ms |
| **Headless support** | Native | Requires setup | Native |
| **GUI mode** | Works (block user) | Works (non-blocking) | Works |
| **Testing ease** | Easy (mock queue) | Medium (mock bpy) | Hard (mock HOM) |
| **Dependencies** | Stdlib only | bpy module | hou module |
| **Integration** | Manual loop | Auto-rescheduled | Event-driven |

---

## Choosing a Pattern for Your DCC

1. **Does your DCC have a built-in main-thread callback or timer system?**
   - **Yes** → Use timer-based (Pattern 2) or event-based (Pattern 3)
   - **No** → Use queue-based (Pattern 1)

2. **Do you need to support both headless AND GUI modes?**
   - **Yes** → Test both; Pattern 1 works for both naturally
   - **GUI only** → Timer/event patterns are cleaner

3. **What's the DCC's Python API threading model?**
   - **Not thread-safe, requires main thread** → All patterns apply
   - **Partially thread-safe (async API exists)** → Consider async patterns (out of scope here)

4. **Is low latency critical?**
   - **Yes** → Avoid timer-based (10ms jitter); prefer queue or event
   - **No** → Timer-based is simpler in GUI mode

---

## Testing Each Pattern

### Queue-Based (Mock)
```python
def test_queue_dispatch():
    server = {{DCC}}MCPServer()
    # Mock: push work item
    result_q = queue.Queue()
    server._main_queue.put((lambda x: x*2, (5,), result_q))
    # Execute one iteration
    func, args, rq = server._main_queue.get(timeout=1)
    result = func(*args)
    rq.put(result)
    # Verify
    assert rq.get() == 10
```

### Timer-Based (Mock bpy)
```python
def test_timer_callback(mocker):
    mock_bpy = mocker.MagicMock()
    server = BlenderMCPServer()
    server._command_queue.put((lambda x: x*2, (5,), result_q))
    # Simulate timer firing
    server._poll_queue()
    assert result_q.get() == 10
```

### Event-Based (Mock HOM)
```python
def test_event_callback(mocker):
    mock_hou = mocker.MagicMock()
    server = HoudiniMCPServer()
    server._command_queue.put((lambda x: x*2, (5,), result_q))
    # Simulate event firing
    server._on_mcp_command()
    assert result_q.get() == 10
```

---

## References

- Nuke: `nuke_addon/nuke_mcp_addon.py` — serve_forever() implementation
- Blender: `blender_addon/server.py` — bpy.app.timers pattern
- Houdini: `houdini_addon/server.py` — HOM event integration

