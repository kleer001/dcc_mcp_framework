# CI/CD and Infrastructure Patterns

Production DCC MCP repos require automated testing, linting, and coverage enforcement. This document covers the infrastructure patterns used in nuke-mcp, blender-mcp, and houdini-mcp.

## GitHub Actions Workflow

All production repos use GitHub Actions with `uv` (fast Python package manager):

```yaml
# .github/workflows/test.yml
name: Tests

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ["3.10", "3.11", "3.12"]

    steps:
      - uses: actions/checkout@v4

      # Install uv package manager
      - uses: astral-sh/setup-uv@v2

      # Run linting BEFORE tests
      - name: Lint with ruff
        run: |
          uv pip install ruff
          ruff check .
          ruff format --check .

      # Install test dependencies
      - name: Install dependencies
        run: uv pip install -e ".[dev]"

      # Run tests (mock mode by default)
      - name: Run tests (mock mode)
        run: |
          uv pip install pytest pytest-cov
          pytest -m "not integration" --cov={{dcc}}mcp --cov-fail-under=70

      # Upload coverage to Codecov
      - name: Upload coverage
        uses: codecov/codecov-action@v3
```

**Key points:**
- **uv**: Fast, deterministic, caches wheels
- **Matrix**: Test Python 3.10, 3.11, 3.12 (covers LTS and current)
- **Ruff first**: Catch style/lint errors before running tests
- **70% coverage**: Enforced via `--cov-fail-under=70`
- **Mock mode**: Default; integration tests require special setup

---

## Local Development Workflow

### Initial Setup

```bash
# Clone repo
git clone https://github.com/user/{{dcc}}-mcp.git
cd {{dcc}}-mcp

# Install with dev dependencies
pip install -e ".[dev]"
# Or with uv:
uv pip install -e ".[dev]"
```

### Before Committing

```bash
# Run linting
ruff check .
ruff format .

# Run tests (mock mode)
pytest -m "not integration"

# If all pass, commit
git add -A
git commit -m "feat: add new tool"
```

### pyproject.toml Structure

```toml
[project]
name = "{{dcc}}-mcp"
version = "0.1.0"
description = "MCP server for {{PRETTY_DCC}}"
requires-python = ">=3.10"
dependencies = [
    "mcp>=0.1.0",  # Or fastmcp>=2.14
]

[project.optional-dependencies]
dev = [
    "pytest>=7.0",
    "pytest-cov>=4.0",
    "ruff>=0.1.0",
]

[tool.pytest.ini_options]
markers = [
    "unit: unit tests (no connection)",
    "mock: mock tests (mock connection, no DCC)",
    "integration: integration tests (requires DCC)",
]
testpaths = ["tests"]

[tool.ruff]
line-length = 100
target-version = "py310"
select = ["E", "F", "W", "I"]  # Errors, undefined names, warnings, imports

[tool.ruff.per-file-ignores]
"__init__.py" = ["F401"]  # Allow unused imports in __init__
```

---

## Pytest Marks and Organization

Tests are organized into tiers (Unit → Mock → Headless → Real → Multi-version):

```python
# tests/test_connection.py
import pytest

@pytest.mark.unit
def test_connection_init():
    """Unit test: no connection needed."""
    conn = Connection()
    assert conn.host == "localhost"

@pytest.mark.mock
def test_send_command_mock(connection_mock):
    """Mock test: uses mock server, no DCC."""
    result = connection_mock.send_command("ping", {})
    assert result["status"] == "ok"

@pytest.mark.integration
def test_send_command_real(connection_real):
    """Integration test: requires real DCC."""
    result = connection_real.send_command("ping", {})
    assert result["status"] == "ok"
```

**Running by tier:**
```bash
# Run only mock tests (default, no DCC required)
pytest -m "not integration"

# Run only integration tests (requires DCC)
pytest -m integration

# Run everything
pytest

# Run specific test
pytest tests/test_connection.py::test_send_command_mock
```

---

## Coverage Requirements

Maintain **70% code coverage** (enforced in CI):

```bash
# Generate coverage report
pytest --cov={{dcc}}mcp --cov-report=html

# Check coverage threshold
pytest --cov={{dcc}}mcp --cov-fail-under=70
```

**What counts toward coverage:**
- ✅ Mock tests (executed in CI)
- ✅ Unit tests (no external deps)
- ❌ Integration tests (skipped in CI; assumed to pass if mock passes)

**Coverage by module:**
- `src/{{dcc}}mcp/server.py` — 90%+ (core logic)
- `src/{{dcc}}mcp/connection.py` — 85%+ (protocol handling)
- `src/{{dcc}}mcp/addon.py` — 70%+ (DCC-specific, hard to test without DCC)
- `src/{{dcc}}mcp/mock.py` — 95%+ (pure logic, no DCC)
- `src/{{dcc}}mcp/tools/*.py` — 80%+ (tool handlers)

---

## Pre-Commit Hooks (Optional)

For faster feedback before `git push`:

**.git/hooks/pre-commit:**
```bash
#!/bin/bash
set -e

echo "Running ruff..."
ruff check .
ruff format . --check

echo "Running pytest..."
pytest -m "not integration" -q

echo "✓ All checks passed"
```

Make executable: `chmod +x .git/hooks/pre-commit`

---

## Release Checklist

Before tagging a release:

```bash
# 1. Update version
# In pyproject.toml: version = "0.2.0"

# 2. Update CHANGELOG
# Add entry under "## [0.2.0] - 2026-05-29"

# 3. Run full test suite
pytest  # All tests, including integration

# 4. Check coverage
pytest --cov={{dcc}}mcp --cov-fail-under=70

# 5. Verify linting
ruff check .
ruff format --check .

# 6. Commit version bump
git add pyproject.toml CHANGELOG.md
git commit -m "chore: bump version to 0.2.0"

# 7. Create tag
git tag -a v0.2.0 -m "Release 0.2.0"

# 8. Push
git push origin main --tags
```

---

## Multi-Version Testing

For DCCs that have multiple major versions (e.g., Nuke 16, 17, 18):

```yaml
# .github/workflows/test-multiversion.yml
name: Multi-Version Tests

on:
  push:
    branches: [main]
  schedule:
    - cron: "0 2 * * 0"  # Weekly on Sunday

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        {{dcc}}-version: ["16", "17", "18"]
    
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v2
      
      - name: Install {{DCC}} {{${{ matrix.{{dcc}}-version }}}}
        run: |
          # DCC-specific installation
          ./scripts/install_{{dcc}}_${{ matrix.{{dcc}}-version }}.sh

      - name: Run integration tests
        run: |
          pytest -m integration --cov={{dcc}}mcp
```

---

## Troubleshooting CI Failures

### "ruff check failed"
```bash
# Auto-fix formatting
ruff format .

# Check remaining issues
ruff check . --fix
```

### "Coverage below 70%"
```bash
# See what's not covered
pytest --cov={{dcc}}mcp --cov-report=term-missing

# Add tests for missing lines
# Focus on high-value code (server.py, connection.py)
```

### "Test timeout"
```bash
# Some tests may hang (e.g., socket reads with no timeout)
# Set pytest timeout:
pytest --timeout=30  # 30 second timeout per test
```

**In pyproject.toml:**
```toml
[tool.pytest.ini_options]
timeout = 30
```

### "Integration test failed"
```bash
# Integration tests require DCC running
# Check: Is DCC installed?
pytest -m "not integration"  # Skip integration tests
```

---

## References

- nuke-mcp: `.github/workflows/test.yml`
- pyproject.toml structure: `nuke-mcp/pyproject.toml`
- Coverage config: `tests/conftest.py` with `pytest-cov`

