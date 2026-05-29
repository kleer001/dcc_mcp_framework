# audit-dcc-mcp Skill

## Description

Audit and improve an existing DCC MCP server against the framework blueprint — protocol conformance, thread-safety, mock parity, confirm-guards on destructive ops, test coverage.

## Trigger Patterns

- "Audit my nuke-mcp for framework conformance"
- "Check if my foo-mcp follows best practices"
- "Review blender-mcp against the framework"
- "Is my houdini-mcp compliant?"

## Workflow

### 1. Read Checklist

Reference:
- `reference/audit-checklist.md` — Complete checklist with all items

Understand the categories:
1. Protocol conformance (envelope shape, framing, handshake)
2. Thread safety (reader thread, main-thread marshalling, daemon threads)
3. Lazy imports (addon importable outside DCC)
4. Confirm guards (destructive ops require confirm=True)
5. Mock parity (every handler has mock)
6. Test coverage (conftest, test_connection, test_<domain>)
7. Tools & annotations
8. Packaging (pyproject.toml, mcp.json, README)
9. Advanced features (events, RAG, memory, etc.)
10. Edge cases (PySide2/6, JSON serialization, version gating)
11. Repo structure (matches template layout)

### 2. Walk Against Target Repo

For each checklist item:

**Protocol Conformance:**
```
- [ ] Envelope shape
  $ grep -r '"type"' src/*/server.py tools/
  $ grep -r '"id"' src/*/server.py  # Should find NONE (anti-pattern)
```

**Thread Safety:**
```
- [ ] Reader thread in connection.py
  $ grep -A 5 "_reader_loop" src/*/connection.py
  $ grep "_response_queue" src/*/connection.py
```

**Mock Parity:**
```
- [ ] Count handlers vs mock commands
  $ grep "def _handle_" addon/*.py | wc -l
  $ grep "def _cmd_" src/*/mock.py | wc -l
  # Should match
```

**Tests:**
```
- [ ] Run mock tests
  $ pytest -m "not integration"
  # Should pass
```

**Ruff:**
```
- [ ] Code style
  $ ruff check .
  # Should pass
```

### 3. Document Findings

For each finding:
- **Severity** — Critical, Major, Minor
- **Category** — Protocol, Thread-safety, Tests, etc.
- **File:Line** — Exact location
- **Description** — What's wrong
- **Recommendation** — How to fix

Example finding:

```markdown
### CRITICAL: Natron Anti-Pattern Envelope (protocol)

File: `src/foomcp/connection.py`, line 45

The response envelope uses `{"id": N, "method": "...", "params": {...}}` instead of the standard `{"type": "...", "params": {...}}`.

Recommendation:
Replace with standard envelope:
- In connection.py: Replace parsing logic with: `msg = {"type": ..., "params": ...}`
- Use template: `templates/shared/src/{{dcc}}mcp/connection.py.tmpl`
```

### 4. Scoring

**Tally findings by severity:**

- **0 critical** + **0-1 major** → **PASS** ✓
- **0 critical** + **2-3 major** → **NEAR PASS** ⚠
- **0-1 critical** + **4+ major** → **FIX RECOMMENDED** 🔧
- **2+ critical** → **REFACTOR** 🚨

Print score at top of report.

### 5. Offer Fixes

For each finding, propose a fix:

**Option 1: Template Replacement**
```
Replace src/{{dcc}}mcp/connection.py with templates/shared/src/{{dcc}}mcp/connection.py.tmpl
- Substitute {{DCC}}, {{dcc}}, {{PORT}} per .scaffold.json
```

**Option 2: Code Patch**
```python
# In addon.py, line 500, change:
# OLD:
if nuke.GUI:

# NEW:
if _get_nuke().GUI:
```

**Option 3: Add Missing File**
```
Create tests/test_memory.py (copy from nuke-mcp/tests/test_memory.py)
Update server.py: add `from {{dcc}}mcp import memory; memory.register(server)`
```

### 6. Generate Report

Print a Markdown report:

```markdown
# Audit Report: foo-mcp

**Status:** PASS ✓

**Date:** 2026-05-29
**Auditor:** audit-dcc-mcp skill
**Framework:** https://github.com/kleer001/dcc_mcp_framework

## Summary

- Checked: 42/42 items
- Critical issues: 0
- Major issues: 0
- Minor issues: 0
- **Result:** Fully conformant to framework blueprint

## Findings

(None)

## Strengths

- Clean protocol implementation
- Full mock parity
- Comprehensive test coverage (92%)
- Well-structured tool domains
- Clear thread-safety design

## Recommendations

1. Consider adding event support (optional, nice-to-have)
2. Add README section on architecture for new contributors
3. Tag v0.1.0 release when stable

---

Framework: https://github.com/kleer001/dcc_mcp_framework
Reference: reference/audit-checklist.md
```

### 7. Interactive Fixes (Optional)

If the user wants fixes:

1. Ask which issues to fix
2. Read relevant template or example repo
3. Generate replacement code
4. Verify (pytest, ruff)
5. Commit with message: `fix: <issue> per audit-dcc-mcp`

## Example: Audit nuke-mcp (Known Good)

Expected result:

```markdown
# Audit Report: nuke-mcp

**Status:** PASS ✓

- All 42 checklist items: ✓
- Critical: 0
- Major: 0
- Minor: 0

**Findings:** None

**Conclusion:** nuke-mcp is the reference implementation for the framework.
```

## Example: Audit Broken Repo

Repo with deliberate issues:

```markdown
# Audit Report: broken-mcp

**Status:** FIX RECOMMENDED 🔧

## Findings

### CRITICAL
1. **Natron Anti-Pattern Envelope** (protocol, connection.py:45)
   - Uses `{"id":1,"method":"cmd"}` instead of `{"type":"cmd","params":{}}`
   - Fix: Replace with templates/shared/src/{{dcc}}mcp/connection.py.tmpl

2. **Missing Confirm Guard** (tools/graph.py:50)
   - `delete_node(name)` allows deletion without confirmation
   - Fix: Add `confirm: bool = False` parameter + check
   ```python
   if not confirm:
       return {"message": "Confirm deletion", "requires_confirmation": True}
   ```

### MAJOR
3. **Mock Parity Missing** (mock.py:12)
   - Addon has 15 handlers but mock has only 8 commands
   - Missing: create_light, delete_light, set_material, execute_python
   - Fix: Add _cmd_* handlers to match _handle_* in addon

4. **No Integration Test Markers** (tests/test_graph.py:1)
   - Tests that require real DCC not marked @pytest.mark.integration
   - Fix: Add markers to tests that call nuke.allNodes() etc.

### MINOR
5. **Incomplete README** (README.md:10)
   - Missing "Advanced Features" section
   - Fix: Add section describing events, RAG, memory support

---

**Total fixes needed:** 5 (2 critical, 2 major, 1 minor)
**Estimated time:** ~2 hours
**Recommended action:** Address critical & major issues; minor can wait
```

## Checklist

- [ ] Read `reference/audit-checklist.md` completely
- [ ] Clone/access target repo
- [ ] Walk each checklist category
- [ ] Document all findings (severity, file:line, description)
- [ ] Propose fixes (template replacement, code patch, add file)
- [ ] Score: PASS / NEAR PASS / FIX RECOMMENDED / REFACTOR
- [ ] Generate Markdown report
- [ ] (Optional) Apply fixes and verify (pytest + ruff)

## Related Skills

- `scaffold-dcc-mcp` — Created target repo
- `add-tool-domain` — Added domains to target repo
- `add-advanced-feature` — Added features to target repo
