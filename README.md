# DCC MCP Framework

A mother-repo framework that provides **reusable templates and 4 Claude skills** for creating and maintaining Model Context Protocol (MCP) servers for Digital Content Creation tools.

This is **not itself a runnable MCP server**. Instead, it provides the architecture, templates, and automation to build new DCC MCP servers quickly and correctly.

## What It Is

A reverse-engineered framework from four production DCC MCP implementations (Nuke, Blender, Houdini, Natron) that captures the unified two-process architecture:

```
Claude (AI client)
  ↓ stdio, JSON-RPC
MCP Server (Python, FastMCP or mcp[cli])
  ↓ TCP newline-JSON
In-DCC Addon (TCP socket server, runs in DCC's Python)
  ↓ calls
DCC Python API (nuke / bpy / hou / NatronEngine)
```

The framework provides:
- **15 reusable templates** (connection, addon, mock, tests, config; advanced features: events, RAG, memory, discovery, plugins, undo, bootstrap)
- **13 reference documentation files** (architecture, protocol, threading, gotchas, advanced features, testing tiers, headless patterns, failure modes, domain knowledge checklist)
- **4 Claude skills** (scaffold new repos, add tool domains, add advanced features, audit existing repos)
- **Production patterns** (thread-safe queue + timer marshalling, mock parity, destructive op guards, resource cleanup, CI/CD infrastructure)

## What It Is NOT

This repository is **not**:
- A runnable MCP server (it's templates + docs, not executable code)
- A pre-built MCP server for every DCC (you scaffold one for your DCC)
- A Python package to `pip install` (it's a Claude Code plugin)
- An MCP client or Claude integration library
- A replacement for DCC documentation (it guides you to integrate your DCC's docs via RAG)
- Pre-filled with DCC-specific code (it teaches the pattern so Claude can generate it)
- Something you clone and run directly (you use it via Claude Code skills)

## Installation

In Claude Code, run:

```bash
/plugin install https://github.com/kleer001/dcc_mcp_framework.git
```

Then use the skills in any Claude conversation:

```
/scaffold-dcc-mcp     # Create a new DCC MCP repo
/add-tool-domain      # Add tools (lighting, rendering, etc.) to existing repo
/add-advanced-feature # Add events, RAG, memory, discovery, plugins, undo
/audit-dcc-mcp        # Review repo for conformance to framework
```

No dependencies. No setup.

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

## What the Framework Provides vs. What You Implement

### Framework Provides
- ✅ Socket protocol (newline-delimited JSON)
- ✅ Main-thread marshalling patterns (queue + timer, async/await, event loops)
- ✅ Mock adapter for offline testing
- ✅ Tool registration and metadata (annotations, confirm guards)
- ✅ Error handling contract (status/error envelope)
- ✅ Resource cleanup (socket close, thread shutdown)
- ✅ Advanced features (events, RAG, memory, discovery, plugins, undo)

### You Must Implement (DCC-Specific)
- 🔧 Addon socket server (in DCC's scripting language)
- 🔧 GUI marshalling (schedule callbacks on DCC main thread)
- 🔧 Tool handlers (call DCC APIs: nuke.createNode(), bpy.ops.mesh.add(), etc.)
- 🔧 Command dispatch (route JSON commands to handlers)
- 🔧 Event callbacks (register DCC callbacks, push events to server)
- 🔧 Undo integration (call DCC's undo API before mutating state)

The **framework does not pre-fill addon code** because every DCC's plugin system is different. Instead, it teaches the patterns so Claude can help you adapt them.

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

## Key Documentation

In `reference/`:
- **architecture.md** — Two-process design, component responsibilities
- **protocol.md** — Socket envelope, handshake, error contract
- **threading.md** — GUI marshalling patterns (queue, timer, event-loop)
- **gotchas.md** — DCC-specific pitfalls and solutions
- **failure-modes.md** — When your DCC doesn't have Python, plugins, etc.
- **domain-knowledge-checklist.md** — What to research and implement for your DCC
- **advanced-features.md** — Events, RAG, memory, discovery, plugins, undo

## Reference Implementations

These production repos show how the framework applies to real DCCs:

- **nuke-mcp** — Queue-based marshalling, events, RAG, memory, discovery
- **blender-mcp** — Timer-based marshalling, clean helpers
- **houdini-mcp** — Event-loop marshalling, headless launch, RAG
- **natron-mcp** — Teaches what *not* to do (wrong envelope pattern)

Study the reference closest to your DCC's threading model.

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

## Contributing

Before proposing changes, read `CLAUDE.md` for project instructions. Ground truth is reverse-engineered from production repos (nuke-mcp, blender-mcp, houdini-mcp, natron-mcp).

## Support

- **Framework issues:** Open an issue on this repo
- **Generated repo conformance:** Use `/audit-dcc-mcp` skill to identify gaps
- **DCC-specific implementation:** Consult your DCC's Python docs + `reference/domain-knowledge-checklist.md`

## License

MIT
