# DCC MCP Framework

A mother-repo framework that provides **reusable templates and 4 Claude skills** for creating and maintaining Model Context Protocol (MCP) servers for Digital Content Creation tools.

This is **not itself a runnable MCP server**. Instead, it provides the architecture, templates, and automation to build new DCC MCP servers quickly and correctly.

## What It Does

Four convergent DCC MCP implementations (Nuke, Blender, Houdini, Natron) show a unified two-process architecture that the framework captures:

```
Claude (AI client)
  ↓ stdio, JSON-RPC
MCP Server (Python, FastMCP or mcp[cli])
  ↓ TCP newline-JSON
In-DCC Addon (TCP socket server, runs in DCC's Python)
  ↓ calls
DCC Python API (nuke / bpy / hou / NatronEngine)
```

The framework packages this as:
- **Reusable templates** with placeholder tokens (for connection, addon, mock, tests, config)
- **Reference documentation** (architecture, protocol, threading, gotchas, advanced features)
- **4 Claude skills** (scaffold, add-domain, add-feature, audit)

## Installation

Install via Claude Code:

```bash
/plugin install https://github.com/kleer001/dcc_mcp_framework.git
```

Once installed, use the skills in any Claude conversation:

```
/scaffold-dcc-mcp     # Create a new DCC MCP repo
/add-tool-domain      # Add a new tool domain to an existing repo
/add-advanced-feature # Add events, RAG, memory, etc. to a repo
/audit-dcc-mcp        # Review an existing repo for framework conformance
```

## Using the Skills

### 1. Scaffold a New DCC MCP

To create a new MCP server for a DCC you're supporting (e.g., Cinema4D):

```
@claude I want to create a new MCP server for Cinema4D.
```

The skill will:
- Ask you to identify the SDK (FastMCP or mcp[cli]), Python API module, GUI marshalling, etc.
- Instantiate templates with your choices
- Generate starter tool domains (graph, script, etc.)
- Verify the scaffold with `pytest` + `ruff` (mock mode, no C4D required)
- Return a ready-to-clone repo structure

### 2. Add a Tool Domain

To add a new set of tools (e.g., lighting, rendering, tracking) to an existing repo:

```
@claude I want to add a "lighting" domain to my blender-mcp repo.
```

The skill will:
- Read your existing repo structure
- Match conventions (connection, addon handlers, mock)
- Generate `tools/lighting.py` with annotated MCP tools
- Add addon handlers and mock handlers
- Wire into `server.py`
- Add tests
- Verify everything passes

### 3. Add an Advanced Feature

To add opt-in capabilities like events, offline RAG doc search, persistent memory, etc.:

```
@claude I want to add bidirectional events to my nuke-mcp repo.
```

The skill will:
- Copy the relevant template module (e.g., `templates/advanced/events.py`)
- Wire event subscriptions into the addon
- Connect the event reader thread
- Add tests
- Verify the changes

### 4. Audit an Existing Repo

To review an existing DCC MCP repo against the framework's best practices:

```
@claude Audit my nuke-mcp repo for conformance.
```

The skill will:
- Walk the `reference/audit-checklist.md`
- Check protocol shape, thread-safety, mock parity, confirm guards, annotations, tests, etc.
- Report findings with file:line pointers
- Offer fixes that reference the framework templates

## Architecture Overview

Two-process design with queue + timer marshalling (thread-safe DCC access):

**MCP Server** (stdio, runs in Claude's context):
- Talks to addon via TCP socket
- Handles tool calls
- Supports mock mode (no DCC binary required)

**Addon** (TCP socket server, runs inside DCC GUI):
- Listens for JSON commands
- Marshals them to DCC's main thread (GUI thread is not thread-safe)
- Returns structured JSON responses
- Can push events back to the server

**Connection** (both sides):
- Newline-delimited JSON framing
- Exponential-backoff retry on connect
- Handshake with version/variant info
- Reader thread separating responses from events
- Timeout + reconnect on broken pipe

**Mock Mode**:
- Simulates the addon's TCP behavior
- Maintains minimal in-memory state
- Every tool has a mock handler (mock parity)
- Enables offline development and testing

## Reference Documentation

See `reference/` for:
- **architecture.md** — Two-process diagram, responsibilities, lifecycle
- **protocol.md** — Envelope spec, framing, handshake, error handling
- **threading.md** — GUI marshalling per DCC; headless serve_forever
- **gotchas.md** — PySide2/6, lazy imports, SO_REUSEADDR, daemon threads, etc.
- **advanced-features.md** — One section per feature (events, RAG, memory, discovery, plugins, undo, recipes)
- **audit-checklist.md** — The framework conformance checklist

## Templates

The `templates/` directory holds:

- `shared/` — SDK-agnostic core (connection.py, mock.py, addon, conftest, etc.)
- `fastmcp/` — FastMCP-specific boilerplate (pyproject.toml, server.py, exemplar tools)
- `mcp_cli/` — mcp[cli]-specific boilerplate (for official Anthropic SDK)
- `advanced/` — Copy-on-demand modules (events, RAG, memory, discovery, plugins)

Each template uses placeholder tokens (`{{DCC}}`, `{{dcc}}`, `{{PORT}}`, etc.) that the scaffold skill substitutes.

## For Framework Maintainers

This repo is **not a Python package**. It's a plugin that ships templates + docs + skills.

- Do **not** add `main.py`, `tests/`, or application code to the root
- Templates use `.tmpl` suffix so they're not collected by linters
- Ground truth: reverse-engineered from `nuke-mcp`, `blender-mcp`, `houdini-mcp`, `natron-mcp`

To update templates:

1. Find the relevant file in a reference repo (e.g., `nuke-mcp/src/nukemcp/connection.py`)
2. Extract the DCC-specific parts
3. Replace with placeholder tokens (e.g., `NukeConnection` → `{{DCC}}Connection`)
4. Save to the appropriate template location with `.tmpl` suffix
5. Document the tokens in `templates/PLACEHOLDERS.md`

## License

MIT
