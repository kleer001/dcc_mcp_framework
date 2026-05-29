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

### Easy: Plugin Install (Recommended)

In Claude Code, run:

```bash
/plugin install https://github.com/kleer001/dcc_mcp_framework.git
```

Then immediately use the skills in any Claude conversation:

```
/scaffold-dcc-mcp     # Create a new DCC MCP repo
/add-tool-domain      # Add a new tool domain to an existing repo
/add-advanced-feature # Add events, RAG, memory, etc. to a repo
/audit-dcc-mcp        # Review an existing repo for framework conformance
```

No dependencies. No setup. The skills handle everything.

### Manual: Clone and Use Locally

If you want to browse the framework files, templates, and docs before using the skills:

```bash
git clone https://github.com/kleer001/dcc_mcp_framework.git
cd dcc_mcp_framework
```

Then explore:
- `reference/` — Read docs to understand the architecture
- `templates/` — See the boilerplate structure
- `skills/*/SKILL.md` — Understand what each skill does

To use the skills, either:
1. Copy the repo path and paste it into Claude Code's plugin installer, or
2. Start a Claude conversation and use the skills directly (they access the repo remotely)

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

## Directory Structure

```
dcc_mcp_framework/
├── reference/              # Single source of truth documentation
│   ├── architecture.md     # Two-process design, component responsibilities
│   ├── protocol.md         # Socket envelope spec, handshake, error handling
│   ├── threading.md        # GUI marshalling patterns (queue, timer, event-loop)
│   ├── gotchas.md          # PySide2/6, lazy imports, SO_REUSEADDR, etc.
│   ├── advanced-features.md # Events, RAG, memory, discovery, plugins, undo
│   ├── audit-checklist.md  # Conformance checklist for audit-dcc-mcp skill
│   ├── failure-modes.md    # What if your DCC doesn't have Python API, plugins, etc.
│   ├── domain-knowledge-checklist.md # Detailed research guide for DCC-specific implementation
│   ├── headless-patterns.md # Queue vs timer vs event patterns per DCC
│   ├── resource-cleanup.md # Socket/thread shutdown patterns
│   ├── infrastructure.md   # CI/CD, coverage, pytest marks
│   └── testing-tiers.md    # Unit→Mock→Headless→Real→Multi-version hierarchy
│
├── templates/              # Token-substitution boilerplate
│   ├── PLACEHOLDERS.md     # {{DCC}}, {{dcc}}, {{PORT}}, etc. definitions
│   ├── shared/             # SDK-agnostic (copy these)
│   │   ├── connection.py   # Socket client/server, newline-JSON framing
│   │   ├── mock.py         # Simulates addon for offline testing
│   │   ├── addon.py.tmpl   # Addon scaffold (DCC-specific: you implement)
│   │   ├── conftest.py     # Pytest fixtures (connection, mock mode)
│   │   ├── tools/__init__.py
│   │   ├── tools/_helpers.py # send(), require_confirm(), etc.
│   │   └── mcp.json.tmpl   # Claude Code config
│   ├── fastmcp/            # FastMCP SDK variant
│   │   ├── pyproject.toml.tmpl
│   │   ├── server.py.tmpl  # MCP server entry point
│   │   └── tools/graph.py.tmpl # Exemplar tool domain
│   ├── mcp_cli/            # mcp[cli] SDK variant (official Anthropic SDK)
│   │   ├── pyproject.toml.tmpl
│   │   ├── server.py.tmpl
│   │   └── tools/graph.py.tmpl
│   └── advanced/           # Copy-on-demand (opt-in features)
│       ├── events.py.tmpl
│       ├── memory.py.tmpl
│       ├── discovery.py.tmpl
│       ├── plugins.py.tmpl
│       ├── undo.py.tmpl
│       ├── scripts/bootstrap.sh.tmpl # Setup wizard
│       └── scripts/ingest_docs.py.tmpl # RAG doc ingestion
│
├── skills/                 # Claude Code skill definitions
│   ├── scaffold-dcc-mcp/SKILL.md    # Create new repo
│   ├── add-tool-domain/SKILL.md     # Add tools/lighting, tools/rendering, etc.
│   ├── add-advanced-feature/SKILL.md # Add events, RAG, memory, etc.
│   └── audit-dcc-mcp/SKILL.md       # Review repo for conformance
│
└── CLAUDE.md              # Project instructions (what this repo is/isn't)
```

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

## Reference Implementations

These production repos show how the framework applies to real DCCs:

- **nuke-mcp** — Polished reference; demonstrates queue-based marshalling, events, RAG, memory, discovery
- **blender-mcp** — Timer-based marshalling with bpy.app.timers; clean helpers
- **houdini-mcp** — Event-loop marshalling; headless launch; RAG on Houdini docs
- **natron-mcp** — Minimal implementation; teaches what *not* to do (wrong envelope)

If you're building an MCP for a specific DCC, study the reference closest to your DCC's threading model.

## Effort Estimate

| Task | Time | Notes |
|------|------|-------|
| **Scaffold** (create new DCC MCP) | 1-2 hours | Templates + skills do most of the work |
| **Add tool domain** (lighting, rendering, etc.) | 1-2 hours per domain | Depends on DCC API complexity |
| **Add advanced feature** (events, RAG, etc.) | 2-4 hours | Copy template + wire into server |
| **Full production repo** | 1-2 weeks | With comprehensive tools, tests, docs |

Without the framework, estimate **4-6 weeks** for a polished, production-ready MCP.

## How to Use This in Practice

### Scenario 1: Building a New MCP for Cinema4D

1. **Start:** Run `/scaffold-dcc-mcp` in Claude Code
2. **Claude asks:** SDK choice, C4D API module, GUI marshalling method
3. **Result:** Ready-to-test repo structure with mock tests passing
4. **Next:** `/add-tool-domain` to add modeling, rendering tools
5. **Optional:** `/add-advanced-feature` to add events, RAG, memory

### Scenario 2: Reviewing Existing Repo for Best Practices

1. **Start:** Run `/audit-dcc-mcp` pointing to your repo
2. **Claude walks:** The conformance checklist, finds issues
3. **Result:** Detailed report with file:line pointers and framework references
4. **Next:** Claude can auto-fix issues or guide you through fixes

### Scenario 3: Understanding the Architecture

1. **Read:** `reference/architecture.md` for two-process overview
2. **Read:** `reference/threading.md` for your DCC's marshalling pattern
3. **Read:** `reference/domain-knowledge-checklist.md` for DCC-specific research guide
4. **Check:** `reference/failure-modes.md` if your DCC has constraints
5. **Study:** Code in a reference repo (e.g., nuke-mcp) that matches your DCC

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

Issues and PRs welcome. Before proposing changes:

- Read `CLAUDE.md` (project instructions)
- Check if it's a framework gap (missing template, unclear docs) or a generated-repo issue (belongs in a DCC-specific repo)
- Reference the ground-truth files in production repos (nuke-mcp, blender-mcp, etc.)
- Test your changes against the audit-dcc-mcp conformance checklist

## Support

- **Framework questions:** Open an issue on this repo
- **Generated repo issues:** Ask Claude Code to run `/audit-dcc-mcp` on your repo, fix findings
- **DCC-specific questions:** Consult your DCC's Python API docs + `reference/domain-knowledge-checklist.md`

## License

MIT
