# Thread Safety & GUI Marshalling

## The Problem

DCC Python APIs are **not thread-safe**. You cannot call nuke, bpy, hou, etc. from arbitrary threads. All API calls must run on the DCC's main (GUI) thread.

The socket server runs on a **background thread** (to avoid blocking the GUI while waiting for network I/O). Therefore, when a command arrives, we must **marshal the handler to the GUI thread**, execute it there, and return the result to the background socket thread.

## The Pattern: Queue + Timer / Executor

Every DCC has a different mechanism for scheduling work on the main thread:

### Nuke: `nuke.executeInMainThread()`

```python
def _run_in_nuke(func, *args):
    """Call func on Nuke's main thread."""
    nuke = _get_nuke()
    return nuke.executeInMainThread(func, args=args)
```

Usage:
```python
result = _run_in_nuke(_handle_create_node, params)
```

- **Synchronous** — Blocks until func returns
- **No timer needed** — Nuke's executeInMainThread handles scheduling

### Blender: `bpy.app.timers`

Blender's timer system runs callbacks on the main thread at regular intervals:

```python
def _run_in_blender(func, *args):
    """Queue func to run on Blender's main thread."""
    result_queue = queue.Queue()
    
    def callback():
        try:
            result = func(*args)
        except Exception as e:
            result = e
        result_queue.put(result)
        return None  # Don't reschedule
    
    bpy.app.timers.register(callback)
    result = result_queue.get(timeout=10)
    if isinstance(result, Exception):
        raise result
    return result
```

- **Asynchronous queuing** — Timer callback runs at 60 Hz (roughly)
- **Timer needed** — Must register a callback; socket thread blocks on queue.get()

### Houdini: `QtCore.QTimer` + Queue

Houdini uses PySide's QTimer:

```python
def _run_in_houdini(func, *args):
    """Queue func to run on Houdini's main thread."""
    result_queue = queue.Queue()
    
    def callback():
        try:
            result = func(*args)
        except Exception as e:
            result = e
        result_queue.put(result)
    
    QtCore.QTimer.singleShot(0, callback)
    result = result_queue.get(timeout=10)
    if isinstance(result, Exception):
        raise result
    return result
```

- **Qt event loop** — Houdini runs a Qt event loop; singleShot(0) schedules immediately
- **Synchronous wait** — Socket thread blocks until result appears in queue

### Natron: `QtCore.QTimer` (same as Houdini)

Natron is also Qt-based; same pattern as Houdini.

## The Headless Case: `serve_forever()`

In **headless mode**, the DCC has no GUI. The main thread is free. Instead of using a timer, the addon calls `serve_forever()`, which drains the command queue on the main thread:

```python
def serve_forever(self):
    """Run the server with main-thread command dispatch (for headless mode)."""
    self._main_queue = queue.Queue()
    self.start()  # Background thread listens on socket
    try:
        while self._running:
            try:
                func, args, result_q = self._main_queue.get(timeout=0.1)
            except queue.Empty:
                continue
            try:
                result = func(*args)
            except Exception as e:
                result = e
            result_q.put(result)
    finally:
        self.stop()
```

- **Main thread drains queue** — No timer needed; main thread loops
- **Socket thread queues commands** — Via `self._main_queue.put((func, args, result_q))`
- **Result returned** — Via result_q

Usage:
```python
nuke_server = NukeMCPServer(port=54321)
nuke_server.serve_forever()  # Blocks; use in headless Nuke launched by MCP
```

## Implementation Checklist

When adding a new DCC:

1. **Identify the main-thread scheduling idiom:**
   - Is there a function like `executeInMainThread()`?
   - Is there a timer/callback system (bpy.app.timers, QtCore.QTimer)?
   - Can the addon run `serve_forever()` on the main thread (headless)?

2. **Implement `_run_in_dcc(func, *args)` helper:**
   - If synchronous (`executeInMainThread`): call directly, return result
   - If async (timer): queue callback, block on result_queue, return result
   - For headless: check if `self._main_queue` is set; use it if available

3. **Update addon startup:**
   - GUI mode: `server.start()` (background thread, use timer/executeInMainThread)
   - Headless mode: `server.serve_forever()` (main thread, queue drain loop)

4. **Test both paths:**
   - GUI: `python -m dcc` (or launch GUI) → `pytest -m "not integration"` (mock mode)
   - Headless: `python main.py --headless` → `pytest -m "not integration"` (mock mode)

## Daemon Threads

**Important:** The socket listener thread should be a **daemon thread**:

```python
self._thread = threading.Thread(target=self._serve, daemon=True)
self._thread.start()
```

This ensures:
- If the main thread exits, the daemon thread is killed automatically
- No need to explicitly `join()` in cleanup
- DCC can quit cleanly even if socket thread is stuck

## Event Handlers & Callbacks

When the addon supports events (e.g., `subscribe_events(["node_created"])`):

1. DCC fires callback (e.g., `nuke.addOnCreate(callback)`)
2. Callback runs on DCC's main thread (automatically)
3. Callback calls `_push_event(...)` to send event to client
4. `_push_event()` writes to socket (should be thread-safe; socket module handles it)

No marshalling needed for event callbacks; they're already on the main thread.

## Lazy DCC Import

To allow testing without the DCC installed:

```python
def _get_dcc():
    """Import DCC lazily so this module can be imported outside the DCC."""
    import nuke
    return nuke
```

This way, `import nuke_mcp_addon` succeeds even if `nuke` is not installed (as long as `_get_dcc()` is not called).

## Socket Thread Safety

The socket operations (send/recv) are **thread-safe** in Python's socket module. However:

- Only **one thread** should read from the socket (the reader thread)
- Only **one thread** should write to the socket, or use a lock

The addon handles this by having the socket listener thread (`_serve`) call handlers, which queue results.

## Performance Considerations

- **Round-trip latency** — Command queued, waits for timer tick (e.g., 16ms for 60Hz), executes, returns. Adds ~16ms per command.
- **Headless is faster** — No timer tick; main thread drains queue immediately.
- **Mock is instant** — No DCC, no marshalling; sync execution.

For most use cases (interactive AI), this latency is acceptable.
