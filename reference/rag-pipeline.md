# RAG Pipeline: Tightly Coupling Documentation to MCP Workflow

The RAG (Retrieval-Augmented Generation) pipeline is the **backbone for real work**. It lets Claude access official {{PRETTY_DCC}} documentation without hallucinating, all offline, tightly integrated into the MCP workflow.

## Why RAG Matters

Claude is powerful but has limitations:
- **Hallucination risk** — Invents plausible-sounding (but wrong) API syntax
- **Knowledge cutoff** — Doesn't know docs newer than training data
- **DCC-specific quirks** — Edge cases not in public training data

RAG solves this by:
- **Embedding official docs** — Vector embeddings of real documentation
- **Semantic search** — "How do I create a {{NODE_NOUN}}?" finds the right section
- **Offline operation** — No external API calls, no latency, no hallucination risk
- **Tight integration** — `search_docs(query)` is just another MCP tool Claude uses

## The Pipeline

### 1. Ingestion: Download Documentation

```bash
python scripts/ingest_docs.py --source local --path /path/to/nuke-docs
python scripts/ingest_docs.py --source github --repo foundry/nuke-docs
python scripts/ingest_docs.py --source web --url https://docs.foundry.com/nuke
```

Three sources supported:
- **Local** — Copy docs from disk (fastest, recommended)
- **GitHub** — Download from GitHub repository (automatic)
- **Web** — Scrape website (requires `requests` + `beautifulsoup4`)

Output: Markdown files in `~/.{{dcc}}mcp/docs/`

### 2. Indexing: Build Embeddings

```bash
python -m {{dcc}}mcp.rag --build
```

Generates vector embeddings using Claude API (requires `ANTHROPIC_API_KEY`):
- Embeds each document chunk
- Saves index to `~/.{{dcc}}mcp/docs/index.json`
- ~5-10 minutes for 1000 docs (one-time cost)

Output: Pre-built index, ready for semantic search

### 3. Search: Use in MCP Workflow

Claude can now search docs while working:

```python
# In conversation, Claude uses:
search_docs("How do I set up proxy rendering in Nuke?")
```

Returns:
```json
{
  "query": "How do I set up proxy rendering in Nuke?",
  "results": [
    {
      "title": "Proxy Settings",
      "content": "...[relevant section]...",
      "score": 0.92
    },
    ...
  ]
}
```

## Architecture

### Three Components

**1. Ingest Script** (`scripts/ingest_docs.py`)
- Downloads/copies docs from various sources
- Cleans up and normalizes markdown
- Outputs to `~/.{{dcc}}mcp/docs/`
- No AI involved; just file operations

**2. RAG Module** (`{{dcc}}mcp/rag.py`)
- Loads pre-built embeddings index
- Implements semantic search (or falls back to keyword search)
- Exposes `search_docs()` MCP tool
- No external calls during operation (fully offline)

**3. MCP Tool** (`search_docs(query, limit=5)`)
- Available in Claude during conversation
- Returns ranked list of relevant docs
- Claude references them in responses

### Embedding Strategy

```
Docs (markdown files)
  ↓
  [Ingest script - no AI]
  ↓
Cleaned markdown
  ↓
  [RAG build process - uses Anthropic API]
  ↓
Vector embeddings (saved to disk)
  ↓
  [At runtime - fully offline]
  ↓
search_docs(query) → semantic search → results
```

Key insight: **Expensive embedding work is one-time (during `--build`). Runtime search is fast and offline.**

## Fallback Strategy

If embeddings aren't available (e.g., Anthropic API issues):

1. **With embeddings**: Semantic search (understands meaning)
2. **Without embeddings**: Keyword search (simple substring matching)

Both work; semantic is better but keyword search is 100% offline and requires no API.

```python
# In rag.py
if HAS_EMBEDDINGS and self.embeddings:
    return self._semantic_search(query, limit)  # Vector similarity
else:
    return self._keyword_search(query, limit)   # Keyword fallback
```

## Hooks & Automation

### Hook 1: Post-Install Ingest

When user installs repo:

```bash
pip install -e .
python scripts/ingest_docs.py --source local --path /path/to/{{DCC}}/docs
python -m {{dcc}}mcp.rag --build
```

Could be automated in `setup.py` or a post-install hook.

### Hook 2: Periodic Updates

Keep docs fresh:

```bash
# Cron job or scheduled task
python scripts/ingest_docs.py --source github --repo owner/{{dcc}}-docs
python -m {{dcc}}mcp.rag --build
```

### Hook 3: Documentation Versioning

Store version info with embeddings:

```json
{
  "version": "17.0v1",
  "timestamp": "2025-05-29T00:00:00Z",
  "docs": {...},
  "embeddings": {...}
}
```

Claude can then say: "Based on {{PRETTY_DCC}} 17.0 documentation..."

## Integration with MCP Server

### Server Registration

In `server.py`:

```python
from {{dcc}}mcp import rag

# Initialize on startup
rag.init(docs_dir="~/.{{dcc}}mcp/docs")

# Register tool
rag.register(server)  # Adds search_docs() tool
```

### Tool Usage by Claude

During conversation:

```
Claude: "Let me search the {{PRETTY_DCC}} docs for {{NODE_NOUN}} creation patterns."

[Claude calls: search_docs("create {{NODE_NOUN}} best practices")]

Response: [{title: "Creating {{NODE_NOUN}}s", content: "...", score: 0.95}, ...]

Claude: "Based on the official docs, here's how to create a {{NODE_NOUN}}:"
```

## Configuration

### docs.yaml

Store in repo root:

```yaml
docs:
  version: "17.0v1"  # Match {{DCC}} version
  sources:
    - type: github
      repo: foundry/nuke-docs
      branch: main
      path: docs
  
  # Optional: exclude patterns
  exclude:
    - "**/api_reference/**"  # Too technical
    - "**/deprecated/**"     # Old versions

search:
  embedding_model: "claude-3-5-sonnet-20241022"  # Or whatever
  max_chunk_size: 8000      # Tokens per embedding
  similarity_threshold: 0.5 # Minimum score to return
```

Then:

```bash
python scripts/ingest_docs.py --config docs.yaml
python -m {{dcc}}mcp.rag --build
```

## Example: Real Work Session

**User:** "Help me set up a complex compositing pipeline in {{PRETTY_DCC}}"

**Claude:**
```
Let me check the {{PRETTY_DCC}} documentation for best practices on {{NODE_NOUN}} composition.
```

**[Claude searches docs]**

```python
search_docs("{{NODE_NOUN}} composition pipeline best practices")
```

**[Returns relevant docs]**

**Claude:** "Based on {{PRETTY_DCC}} official documentation, here's the recommended pattern:
1. Create Read nodes for inputs
2. Chain processing {{NODE_NOUN}}s in dependency order
3. Use Merge {{NODE_NOUN}}s for combining
4. Apply color correction at the end

Let me create this structure for you..."

**Result:** Claude works *with authority*, citing official docs, not guessing.

## Performance Characteristics

| Operation | Time | Notes |
|-----------|------|-------|
| Ingest docs | ~5-10 min | One-time, no AI |
| Build embeddings | ~5-10 min | One-time, uses API |
| Search query | <100ms | Fully offline |
| Claude response | 2-5s | Same as normal |

## Troubleshooting

### "No index found"
```bash
python scripts/ingest_docs.py --source github --repo owner/{{dcc}}-docs
python -m {{dcc}}mcp.rag --build
```

### "Embedding failed"
Check `ANTHROPIC_API_KEY`:
```bash
echo $ANTHROPIC_API_KEY
python -c "import anthropic; print(anthropic.Anthropic().api_key)"
```

### "Search returns nothing"
- Check docs were ingested: `python scripts/ingest_docs.py --list`
- Try simpler query: `search_docs("{{NODE_NOUN}}")` vs `search_docs("how do I create a {{NODE_NOUN}}")`
- System will fall back to keyword search automatically

## Extensibility

### Custom Document Processors

Extend `ingest_docs.py` to handle:
- PDF extraction
- API reference parsing
- Video transcripts
- Community forum posts

### Custom Embeddings

Replace Anthropic embeddings with:
- Local embeddings (ollama, sentence-transformers)
- Cheaper providers (Cohere, Together)
- Fine-tuned models on {{DCC}} terminology

### Hybrid Search

Combine embeddings + keyword search:
- Semantic: find conceptually related sections
- Keyword: find exact API calls

## Ground Truth

This pattern is proven in:
- **nuke-mcp** — Official Nuke documentation embedded
- **houdini-mcp** — SideFX docs + tutorials
- Both shipped RAG-enabled repos to production

The pipeline has moved from "nice to have" to **essential infrastructure for AI-assisted DCC work**.
