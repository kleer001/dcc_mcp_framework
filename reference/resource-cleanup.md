# Resource Cleanup Patterns

When the MCP server shuts down—whether normally or due to an exception—resources must be cleaned up properly. Improper cleanup can leave sockets open, threads hanging, or timers running. This document covers the patterns used in production MCPs.

## The Challenge

The {{DCC}} MCP server holds several resources:
1. **Socket** — TCP connection listening on a port
2. **Background thread** — Socket listener thread (daemon)
3. **DCC-specific resources** — Timers (Blender), event handlers (Houdini), undo state
4. **File handles** — Logs, temp files

When shutting down:
- Close socket first (stops accepting connections)
- Join threads with timeout (prevent hang)
- Cancel timers/handlers
- Flush and close logs
- Clean temp files

Failure to do this can cause:
- Port remains bound (can't restart server)
- Thread hangs (process won't exit)
- Zombie resources in DCC
- Incomplete logs

---

## Pattern 1: Socket Close with OSError Handling

Sockets can raise `OSError` when closed twice or if already closed by the OS. Always catch this:

```python
def _close_socket(self):
    """Close socket safely."""
    if self._socket:
        try:
            self._socket.close()
        except OSError as e:
            # Already closed or other OS-level error; ignore
            log.debug(f"Socket close error (expected): {e}")
        self._socket = None
```

**Why OSError?**
- `socket.close()` on a closed socket raises `OSError`
- The OS may have already closed it (client disconnect)
- Double-close should not crash the server

**Good:**
```python
try:
    self._socket.close()
except OSError:
    pass  # Expected if already closed
```

**Bad:**
```python
self._socket.close()  # Will raise if already closed
```

---

## Pattern 2: Thread Join with Timeout

Threads must be joined to ensure they finish cleanly. But a thread hang will block shutdown forever. Always use a timeout:

```python
def _stop(self):
    """Stop server and wait for threads."""
    self._running = False
    
    # Give thread time to finish
    if self._listener_thread and self._listener_thread.is_alive():
        self._listener_thread.join(timeout=2.0)
        if self._listener_thread.is_alive():
            log.warning("Listener thread did not stop; continuing anyway")
```

**Why timeout?**
- If the thread is blocked on a socket read, it won't exit immediately
- Without timeout, `join()` blocks forever
- 2.0 second timeout is conventional (long enough to be safe, short enough to not hang)

**Good:**
```python
thread.join(timeout=2.0)  # Wait up to 2 seconds
if thread.is_alive():
    log.warning("Thread still running after timeout")
```

**Bad:**
```python
thread.join()  # Will block forever if thread is hung
```

---

## Pattern 3: try/finally for Guaranteed Cleanup

Cleanup must happen even if an exception occurs. Always use `try/finally`:

```python
def serve_forever(self):
    """Main thread drains command queue."""
    self.start()  # Start background listener
    try:
        while self._running:
            try:
                func, args, result_q = self._main_queue.get(timeout=0.1)
                result = func(*args)
                result_q.put(result)
            except queue.Empty:
                continue
            except Exception as e:
                log.error(f"Command failed: {e}")
                # Attempt to notify caller
    finally:
        # Cleanup ALWAYS happens, even if exception above
        log.info("Shutting down...")
        self._close_socket()
        self._stop_threads()
        log.info("Shutdown complete")
```

**Key points:**
- `finally` block ALWAYS executes
- Even if an exception is raised in the try block
- Use it for cleanup that must happen

---

## Pattern 4: Cleanup Sequence

Resources should be cleaned up in reverse order of creation:

```
1. Create socket → Bind and listen
2. Create thread → Start listener
3. Create handlers/timers → Register with DCC

Shutdown (reverse):
3. Cancel handlers/timers
2. Join thread with timeout
1. Close socket
```

**Example (Blender):**
```python
def cleanup(self):
    """Cleanup in reverse order of creation."""
    # 1. Cancel Blender timer
    if bpy.app.timers.is_registered(self._poll_queue):
        bpy.app.timers.unregister(self._poll_queue)
        log.info("Timer unregistered")
    
    # 2. Signal thread to stop and wait
    self._running = False
    if self._listener_thread:
        self._listener_thread.join(timeout=2.0)
        log.info("Listener thread joined")
    
    # 3. Close socket
    try:
        self._socket.close()
        log.info("Socket closed")
    except OSError:
        pass
```

---

## Pattern 5: Shutdown Signal

Use a flag to signal threads to stop gracefully:

```python
class {{DCC}}MCPServer:
    def __init__(self):
        self._running = True
        self._listener_thread = None

    def start(self):
        """Start background listener."""
        self._listener_thread = threading.Thread(target=self._socket_listen, daemon=True)
        self._listener_thread.start()

    def _socket_listen(self):
        """Background thread: listen for commands while _running."""
        while self._running:
            try:
                # Socket read with timeout so we check _running periodically
                data = self._socket.recv(1024, timeout=0.5)
                if data:
                    self._process_command(data)
            except socket.timeout:
                continue  # Check _running flag
            except Exception as e:
                if self._running:  # Only log if not shutting down
                    log.error(f"Socket error: {e}")
                break

    def stop(self):
        """Signal thread to stop."""
        self._running = False
        self._listener_thread.join(timeout=2.0)
```

**Why timeout on socket reads?**
- Without it, `recv()` blocks forever on the background thread
- When `_running = False` is set, the thread is stuck on `recv()`
- Timeout (0.5s) lets the thread check `_running` periodically
- It then exits the while loop and joins cleanly

---

## Pattern 6: Context Manager for Guaranteed Cleanup

For simpler cases, use a context manager:

```python
class {{DCC}}MCPServer:
    def __enter__(self):
        self.start()
        return self
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        self.stop()
        return False  # Don't suppress exceptions

# Usage:
with {{DCC}}MCPServer() as server:
    server.serve_forever()
# Automatically cleaned up
```

---

## Testing Cleanup

Test that resources are actually cleaned up:

```python
def test_cleanup_closes_socket():
    server = {{DCC}}MCPServer()
    server.start()
    assert not server._socket.fileno() == -1  # Socket is open
    
    server.stop()
    
    # Socket should be closed
    with pytest.raises(OSError):
        server._socket.fileno()  # Closed sockets raise OSError

def test_cleanup_joins_thread():
    server = {{DCC}}MCPServer()
    server.start()
    assert server._listener_thread.is_alive()
    
    server.stop()
    
    # Thread should have exited
    server._listener_thread.join(timeout=1.0)
    assert not server._listener_thread.is_alive()

def test_cleanup_on_exception():
    """Verify cleanup happens even if serve_forever() raises."""
    server = {{DCC}}MCPServer()
    
    def mock_handler(*args):
        raise ValueError("Simulated error")
    
    server._main_queue.put((mock_handler, (), queue.Queue()))
    
    # This should raise but still cleanup
    with pytest.raises(ValueError):
        server.serve_forever()
    
    # Verify cleanup happened despite exception
    assert not server._listener_thread.is_alive()
```

---

## Checklist for Your DCC

When implementing cleanup for {{PRETTY_DCC}}:

- [ ] Socket close catches `OSError`
- [ ] Thread join uses `timeout=2.0`
- [ ] Cleanup in `finally` block
- [ ] Reverse order: handlers → threads → socket
- [ ] `_running` flag signals graceful shutdown
- [ ] Socket reads use timeout (check `_running` periodically)
- [ ] Tests verify socket closed
- [ ] Tests verify threads joined
- [ ] Tests verify cleanup on exception
- [ ] Logs show when/why shutdown happened

---

## Common Issues

### "Address already in use" on restart
**Cause:** Socket not closed properly  
**Fix:** Use `socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)` and always close socket

```python
self._socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
self._socket.bind(("localhost", {{PORT}}))
```

### Process hangs on shutdown
**Cause:** Thread not joined or timeout too short  
**Fix:** Ensure `_running` flag + socket timeout + join with timeout

```python
self._running = False  # Signal thread
self._listener_thread.join(timeout=2.0)  # Wait
```

### Incomplete logs
**Cause:** Logs not flushed before exit  
**Fix:** Flush handlers before closing

```python
import logging
for handler in logging.root.handlers:
    handler.flush()
```

### Zombie DCC processes
**Cause:** Threads not joined, process exits before cleanup  
**Fix:** Use `daemon=False` for threads that MUST finish, or ensure join before exit

```python
# For critical cleanup threads:
self._cleanup_thread = threading.Thread(target=self._cleanup, daemon=False)
```

---

## References

- Nuke: `nuke_addon/nuke_mcp_addon.py:750-825` — Full cleanup example
- Blender: `blender_addon/server.py:60-82` — Timer cleanup
- All: Connection.py — Socket SO_REUSEADDR setup

