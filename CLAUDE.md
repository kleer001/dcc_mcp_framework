# DCC MCP Framework

A mother-repo framework that exposes **4 Claude skills** for building and maintaining Model Context Protocol (MCP) servers for Digital Content Creation tools (Nuke, Blender, Houdini, Maya, Cinema4D, Krita, Resolve, etc.).

This repo is **not a runnable MCP server itself**. It's a framework that provides:

- **Templates** — SDK-agnostic and SDK-specific (FastMCP vs mcp[cli]) boilerplate that you substitute placeholders into
- **Reference documentation** — Single source of truth for architecture, protocol, threading, and advanced features
- **Skills** — 4 Claude workflow automation entries (scaffold, add-domain, add-feature, audit)

## The Four Skills

1. **scaffold-dcc-mcp** — Create a new DCC MCP repo from templates (one-shot scaffolding)
2. **add-tool-domain** — Add a tool domain (lighting, rendering, tracking, etc.) to an existing repo
3. **add-advanced-feature** — Add opt-in features (events, RAG, memory, headless launch, plugin discovery, etc.)
4. **audit-dcc-mcp** — Review an existing repo for conformance to the framework blueprint

## Installation

Install the plugin via Claude Code:
```
/plugin install https://github.com/kleer001/dcc_mcp_framework.git
```

Then use any of the 4 skills:
```
/scaffold-dcc-mcp  # Create a new DCC MCP repo
/add-tool-domain   # Add a domain to an existing repo
/add-advanced-feature  # Add a feature
/audit-dcc-mcp     # Audit a repo
```

## Key Design Decisions

- **Template strategy**: Hybrid copy + token-substitution for the near-identical core (connection, addon, mock, conftest, pyproject); skills *generate* thin DCC-specific modules from exemplars.
- **SDK support**: Ship two variants — `fastmcp>=2.14` (standalone) and `mcp[cli]>=1.4` (official SDK). Scaffold skill lets the user choose.
- **Mock parity**: Every tool must have a mock handler. Tests run offline with no DCC binary required.
- **Destructive ops**: Guarded by `confirm=True` + `require_confirm()` helper. Annotated with `destructiveHint`.
- **Thread-safety**: DCC APIs are not thread-safe. Socket runs on background thread; commands marshalled to DCC main thread via queue + timer, with Event to return result. Headless mode drains queue on main thread via `serve_forever()`.

## Reference Docs

- `reference/architecture.md` — Two-process diagram, component responsibilities, lifecycle
- `reference/protocol.md` — Envelope spec (newline-delimited JSON), framing, handshake, error contract
- `reference/threading.md` — GUI marshalling per DCC (bpy.app.timers / QtCore.QTimer / nuke.executeInMainThread); headless serve_forever
- `reference/gotchas.md` — PySide6/2, lazy import, SO_REUSEADDR, daemon threads, reconnect, mcp[cli]<->fastmcp notes
- `reference/advanced-features.md` — One section per: events, RAG, memory, discovery, plugins, undo, recipes
- `reference/audit-checklist.md` — The conformance checklist that audit-dcc-mcp walks

## Templates

Located in `templates/`:

- `shared/` — SDK-agnostic (one copy): connection.py, mock.py, tools/__init__.py, _helpers.py, addon, conftest, test_connection, mcp.json
- `fastmcp/` — FastMCP variant: pyproject.toml, server.py, tools/graph.py exemplar
- `mcp_cli/` — mcp[cli] variant: pyproject.toml, server.py, tools/graph.py exemplar
- `advanced/` — Copy-on-demand (stdlib-only): events.py, rag.py, memory.py, discovery.py, plugins.py

## Placeholder Tokens

Every template uses these substitutable tokens (documented in `templates/PLACEHOLDERS.md`):

- `{{DCC}}` — PascalCase (Nuke, Blender, Houdini)
- `{{dcc}}` — package stem (nuke, blender, houdini)
- `{{PRETTY_DCC}}` — Display name (Foundry Nuke, Blender, SideFX Houdini)
- `{{PORT}}` — Default TCP port (default 54321)
- `{{API_MOD}}` — Python API module (nuke, bpy, hou, NatronEngine)
- `{{GUI_MARSHAL}}` — Main-thread dispatch idiom (nuke.executeInMainThread, bpy.app.timers, etc.)
- `{{SCENE_NOUN}}` — Scene/comp/script
- `{{NODE_NOUN}}` — node/object/point

## Ground Truth

The convergent architecture is reverse-engineered from four production DCC MCP repos:

- `nuke-mcp` — Most polished; primary template source
- `blender-mcp` — Clean helpers, queue+timer addon
- `houdini-mcp` — Headless launch, RAG, events
- `natron-mcp` — Minimal; teaches the anti-pattern (`{id,method}` envelope) to *avoid*

## No Framework Code

This repo's own structure is minimal. Do not:
- Add a `main.py`, `tests/`, or other code to the framework itself
- Try to run the framework as a Python app (it's not one)
- Lint or import `.tmpl` files (they're templates, not code)

The framework's job is **templates + docs + skills**, not executable code.

## Branching & PRs

- Develop on feature branches
- Atomic commits: `feat`, `fix`, `docs`, `refactor`, `test`, `chore`
- Push to `origin/<branch>` and open PR against `main`
