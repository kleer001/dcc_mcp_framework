# Integration Testing Tiers: Unit → Mock → Headless → Real → Multi-Version

Testing a DCC MCP server requires different levels of fidelity. This document explains the five-tier test hierarchy used in production repos.

## The Five Tiers

```
┌─────────────────────────────────────────────────────┐
│ Tier 5: Multi-Version Tests                         │
│ (Multiple DCC versions, multiple platforms, slow)  │
└─────────────────────────────────────────────────────┘
                          ↑
┌─────────────────────────────────────────────────────┐
│ Tier 4: Real Tests (Requires DCC with GUI)         │
│ (Actual DCC instance, real API calls, 5-10min)    │
└─────────────────────────────────────────────────────┘
                          ↑
┌─────────────────────────────────────────────────────┐
│ Tier 3: Headless Tests (DCC in headless mode)      │
│ (No GUI, real DCC API, 1-5min)                     │
└─────────────────────────────────────────────────────┘
                          ↑
┌─────────────────────────────────────────────────────┐
│ Tier 2: Mock Tests (No DCC, mock connection)       │
│ (Fast, offline, deterministic, 10-30s, DEFAULT)   │
└─────────────────────────────────────────────────────┘
                          ↑
┌─────────────────────────────────────────────────────┐
│ Tier 1: Unit Tests (Pure logic, no connection)     │
│ (Fastest, no dependencies, <1s)                    │
└─────────────────────────────────────────────────────┘
```

Each tier builds on the previous one. Most development uses Tiers 1-2 (fast). CI runs Tier 2. Nightly runs Tier 3-5.

---

## Tier 1: Unit Tests

**Purpose:** Test pure business logic without any connection or external dependencies.

**Requirements:**
- No `Connection` object
- No mock server
- No {{DCC}} API
- Just functions, classes, utilities

**Example:**
```python
import pytest
from {{dcc}}mcp.helpers import parse_node_info

@pytest.mark.unit
def test_parse_node_info():
    """Test node info parsing (pure logic)."""
    info_str = "MyNode [Read v1.0]"
    result = parse_node_info(info_str)
    assert result["name"] == "MyNode"
    assert result["class"] == "Read"
    assert result["version"] == "1.0"
```

**Fixtures:** None (pure Python)

**Run:** `pytest -m unit`

**Speed:** <1 second

---

## Tier 2: Mock Tests

**Purpose:** Test MCP protocol and command handling without requiring {{DCC}}.

**Requirements:**
- Mock socket server (in-process)
- Mock {{DCC}} API (simulated state)
- No real {{DCC}} binary required
- Deterministic responses

**Example:**
```python
import pytest
from {{dcc}}mcp.connection import {{DCC}}Connection
from {{dcc}}mcp.mock import Mock{{DCC}}Server

@pytest.mark.mock
def test_create_node_mock(connection_mock):
    """Test create_node via mock server."""
    result = connection_mock.send_command("create_node", {
        "class": "Read",
        "name": "my_read"
    })
    assert result["status"] == "ok"
    assert result["result"]["name"] == "my_read"

@pytest.mark.mock
def test_delete_node_requires_confirm(connection_mock):
    """Test destructive operation requires confirm=True."""
    # First attempt: no confirm → returns action request
    result = connection_mock.send_command("delete_node", {
        "name": "my_node",
        "confirm": False
    })
    assert result["action"] == "delete_node"
    assert "Confirm deletion" in result["message"]
    
    # Second attempt: with confirm → succeeds
    result = connection_mock.send_command("delete_node", {
        "name": "my_node",
        "confirm": True
    })
    assert result["status"] == "ok"
```

**Fixtures:** `connection_mock` (from conftest.py)
```python
@pytest.fixture
def connection_mock():
    """In-process mock connection for testing."""
    server = Mock{{DCC}}Server()
    server.start()
    conn = {{DCC}}Connection(port={{PORT}})
    conn.connect()
    yield conn
    conn.disconnect()
    server.stop()
```

**Run:** `pytest -m "not integration"` (default)

**Speed:** 10-30 seconds for full suite

**Default:** All test suites should have Tier 2 tests; Tier 2 is the baseline.

---

## Tier 3: Headless Tests

**Purpose:** Test against a real {{DCC}} instance running in headless mode (no GUI).

**Requirements:**
- Real {{DCC}} binary installed
- Launched in headless mode (no X11/Wayland)
- Real {{DCC}} Python API
- Real file I/O (scene files, caches)

**Example:**
```python
import pytest

@pytest.mark.integration
def test_create_node_real(connection_real):
    """Test against real {{DCC}} in headless mode."""
    result = connection_real.send_command("create_node", {
        "class": "Read",
        "name": "real_read",
        "path": "/tmp/test.exr"
    })
    assert result["status"] == "ok"
    
    # Verify node actually exists (not just mocked)
    scene_info = connection_real.send_command("get_scene_info", {})
    assert any(n["name"] == "real_read" for n in scene_info["nodes"])
```

**Fixtures:** `connection_real` (from conftest.py)
```python
@pytest.fixture(scope="session")
def headless_server():
    """Launch {{DCC}} headless server once per session."""
    import subprocess
    proc = subprocess.Popen([
        "{{dcc}}", "--headless", "--mcp-server"
    ])
    time.sleep(5)  # Wait for startup
    yield
    proc.terminate()

@pytest.fixture
def connection_real(headless_server):
    """Connect to real headless {{DCC}} instance."""
    conn = {{DCC}}Connection(port={{PORT}})
    conn.connect()
    yield conn
    conn.disconnect()
```

**Launch script:** `scripts/launch_headless.sh`
```bash
#!/bin/bash
# Launch {{DCC}} in headless mode with MCP addon loaded
export HEADLESS=1
{{dcc}}-path/bin/{{dcc}} --headless --python \
  -c "import {{dcc}}_mcp_addon; {{dcc}}_mcp_addon.start()"
```

**Run:** `pytest -m integration --headless`

**Speed:** 1-5 minutes (DCC startup overhead)

**When to use:** Before committing; check that code works against real {{DCC}} API

---

## Tier 4: Real Tests (with GUI)

**Purpose:** Test with {{DCC}} running in normal GUI mode.

**Requirements:**
- Real {{DCC}} binary with display (X11, Wayland, or macOS)
- User interactions (dialog boxes, UI events)
- Real {{DCC}} GUI event loop

**Example:**
```python
import pytest

@pytest.mark.integration
@pytest.mark.gui
def test_undo_push_in_gui(connection_gui):
    """Test undo/redo via real GUI."""
    # Execute command
    result = connection_gui.send_command("create_node", {
        "class": "Read",
        "name": "test_read"
    })
    assert result["status"] == "ok"
    
    # Undo via GUI
    # (DCC-specific; e.g., nuke.Undo() context)
    undo_result = connection_gui.send_command("undo", {})
    assert undo_result["status"] == "ok"
    
    # Node should be gone
    scene = connection_gui.send_command("get_scene_info", {})
    assert not any(n["name"] == "test_read" for n in scene["nodes"])
```

**Fixtures:** `connection_gui` (requires display)
```python
@pytest.fixture(scope="session")
def gui_server():
    """Launch {{DCC}} in GUI mode."""
    import subprocess
    proc = subprocess.Popen([
        "{{dcc}}", "--mcp-server"  # No --headless
    ])
    time.sleep(10)  # Wait for GUI startup
    yield
    proc.terminate()
```

**Run:** `pytest -m gui` (requires display, rarely used in CI)

**Speed:** 5-10 minutes

**When to use:** Manual testing before release; CI rarely runs this

---

## Tier 5: Multi-Version Tests

**Purpose:** Verify code works across multiple {{DCC}} versions.

**Requirements:**
- Multiple {{DCC}} versions installed (e.g., 16, 17, 18 for Nuke)
- Test matrix per version
- Nightly/weekly scheduled run

**Example (GitHub Actions):**
```yaml
# .github/workflows/multi-version.yml
name: Multi-Version Tests
on:
  schedule:
    - cron: "0 2 * * 0"  # Weekly Sunday 2am UTC

jobs:
  test:
    strategy:
      matrix:
        {{dcc}}-version: ["16", "17", "18"]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Install {{DCC}} ${{ matrix.{{dcc}}-version }}
        run: |
          ./scripts/install_{{dcc}}_${{ matrix.{{dcc}}-version }}.sh
      
      - name: Run integration tests
        run: pytest -m integration
```

**Example (manual):**
```bash
# Test against Nuke 16
/opt/Nuke16.0v1/libnuke.so pytest -m integration
# Test against Nuke 17
/opt/Nuke17.0v1/libnuke.so pytest -m integration
# etc.
```

**Run:** `pytest -m multi-version` (scheduled, slow)

**Speed:** 30+ minutes (depends on number of versions)

**When to use:** Before release; scheduled nightly runs

---

## Test Pyramid (Recommended Coverage)

```
              │       Tier 5 (1-2 tests)
              │       Multi-version
              │  ┌────────────────────┐
              │  │                    │
              │  │  Tier 4 (3-5 tests)
              │  │  GUI-specific
              │  ├────────────────────┤
              │  │                    │
              │  │ Tier 3 (10-20 tests)
              │  │ Headless integration
              │  ├────────────────────┤
              │  │                    │
              │  │ Tier 2 (50+ tests) │  ← MOST of your tests
              │  │ Mock (DEFAULT)     │
              │  │                    │
              │  │ Tier 1 (10-20)     │
              └──┴────────────────────┘
```

**Guidelines:**
- **70% of tests:** Tier 2 (mock) — fast, deterministic, CI-safe
- **20% of tests:** Tier 3 (headless) — verify real DCC API
- **5% of tests:** Tier 4 (GUI) — verify GUI-specific behavior
- **5% of tests:** Tier 1 (unit) — edge cases, utilities
- **Tier 5:** Not counted in daily CI; scheduled weekly

**In numbers:**
- 100 tests total
- 70 mock (Tier 2)
- 15 headless (Tier 3)
- 5 GUI (Tier 4)
- 10 unit (Tier 1)
- Tier 5: separate nightly job

---

## Pytest Configuration

**In pyproject.toml:**
```toml
[tool.pytest.ini_options]
markers = [
    "unit: unit tests (pure logic, no connection)",
    "mock: mock tests (mock server, no DCC)",
    "integration: integration tests (requires DCC)",
    "gui: GUI-specific tests (headless won't work)",
    "multi_version: multi-version matrix tests",
]
testpaths = ["tests"]
timeout = 30

[tool.pytest.ini_options.addopts]
# Default: run Tier 1 + Tier 2 only (fastest)
addopts = "-m 'not integration'"
```

**Usage:**
```bash
# Tier 1 + 2 (DEFAULT - what CI runs)
pytest -m "not integration"

# Tier 3 (headless)
pytest -m "integration and not gui"

# Tier 4 (GUI)
pytest -m gui

# Tier 5 (multi-version)
pytest -m multi_version

# All
pytest
```

---

## Conftest Structure

**tests/conftest.py:**
```python
import pytest
import queue
from {{dcc}}mcp.connection import {{DCC}}Connection
from {{dcc}}mcp.mock import Mock{{DCC}}Server

# Tier 1: unit tests (no fixtures needed)

# Tier 2: mock tests
@pytest.fixture
def connection_mock():
    """In-process mock server."""
    server = Mock{{DCC}}Server()
    server.start()
    conn = {{DCC}}Connection()
    conn.connect()
    yield conn
    conn.disconnect()
    server.stop()

# Tier 3: headless tests
@pytest.fixture(scope="session")
def headless_proc():
    """Start {{DCC}} headless once per test session."""
    # Launch {{DCC}} headless
    yield
    # Cleanup

@pytest.fixture
def connection_real(headless_proc):
    """Connect to real headless {{DCC}}."""
    conn = {{DCC}}Connection()
    conn.connect()
    yield conn
    conn.disconnect()

# Tier 4: GUI tests
@pytest.fixture(scope="session")
def gui_proc():
    """Start {{DCC}} with GUI."""
    # Launch {{DCC}} normal
    yield
    # Cleanup

@pytest.fixture
def connection_gui(gui_proc):
    """Connect to GUI {{DCC}}."""
    conn = {{DCC}}Connection()
    conn.connect()
    yield conn
    conn.disconnect()
```

---

## Continuous Integration Strategy

| Tier | When | Duration | Gating |
|------|------|----------|--------|
| 1+2 | Every commit (default) | 10-30s | Yes (blocks merge) |
| 3 | Nightly headless job | 5-10min | Warning only |
| 4 | Manual before release | 5-10min | N/A |
| 5 | Weekly multi-version | 30+min | Warning only |

**In GitHub Actions:**
```yaml
# Fast job (blocks PR)
- name: Unit + Mock Tests
  run: pytest -m "not integration"

# Slow jobs (informational)
- name: Headless Tests (nightly)
  if: github.event_name == 'schedule'
  run: pytest -m integration
```

---

## References

- Blender-mcp: `tests/` directory (tier 1-4 examples)
- Nuke-mcp: `.github/workflows/test.yml` (CI setup)
- Houdini-mcp: `tests/conftest.py` (fixture setup)

