# Memslice - Product Requirements Document

> The universal memory layer for AI coding agents.

## Executive Summary

**Memslice** is an open-source memory layer for agentic coding tools. It provides persistent, structured memory that works across Claude Code, Codex CLI, Gemini CLI, Cursor, and any MCP-compatible or REST-based AI tool.

**Vision:** Your AI coding assistant finally remembers—your codebase, your preferences, your past decisions—across every session and every tool.

### Strategy

| Aspect | Decision |
|--------|----------|
| **Audience** | Developers using agentic coding tools |
| **Model** | Open source (MIT) |
| **Goal** | Community adoption, GitHub stars, ecosystem influence |
| **Monetization** | Not a priority (possible future: hosted/enterprise) |

### Quick Start

```bash
uv tool install memslice && memslice start -d
```

---

## Problem Statement

### The Pain Point

Agentic coding tools (Claude Code, Codex, Cursor, Gemini CLI) are powerful but **amnesiac**:

- Every new session starts from zero context
- Tool-specific memory doesn't transfer (Cursor memories don't help Claude Code)
- Project context must be re-explained constantly
- Coding preferences are forgotten ("use pnpm", "prefer functional style")
- Past debugging sessions and decisions are lost

### Root Cause

Each AI coding tool builds its own memory (or none at all):

| Tool | Memory | Problem |
|------|--------|---------|
| Claude Code | CLAUDE.md files | Tool-specific, manual |
| Cursor | .cursorrules | Tool-specific |
| Codex | None built-in | Starts fresh every time |
| Gemini CLI | None built-in | Starts fresh every time |

There's no **universal memory layer** that works across all tools.

### Developer Impact

Developers waste time on:

- Writing and maintaining CLAUDE.md / .cursorrules for each tool
- Re-explaining project architecture in new sessions
- Repeating coding preferences ("use TypeScript strict mode", "prefer composition")
- Losing context from past debugging sessions

---

## Solution Overview

### Core Concept: Four-Layer Memory Model

Inspired by computer memory hierarchy (registers → cache → RAM → disk):

| Layer           | Name            | Content                                        | Storage                  |
| --------------- | --------------- | ---------------------------------------------- | ------------------------ |
| **Global**      | User Profile    | Preferences, coding style, communication style | SQLite `config` table    |
| **Long-term**   | Knowledge Base  | API docs, code snippets, reference materials   | LanceDB (vectors)        |
| **Medium-term** | Project State   | Current project status, TODOs, known bugs      | NetworkX (graph)         |
| **Short-term**  | Session Context | Recent conversation, current task state        | SQLite `chat_logs` table |

### Architecture: Sidecar Model

Memslice runs as an **independent local service** that AI tools connect to:

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  Claude Code    │     │  Custom Agent   │     │   IDE Plugin    │
└────────┬────────┘     └────────┬────────┘     └────────┬────────┘
         │                       │                       │
         └───────────────────────┼───────────────────────┘
                                 │
                    ┌────────────▼────────────┐
                    │    Memslice Server      │
                    │    localhost:8999       │
                    │  ┌─────────────────┐    │
                    │  │   REST API      │    │
                    │  │   MCP Server    │    │
                    │  └────────┬────────┘    │
                    │           │             │
                    │  ┌────────▼────────┐    │
                    │  │   Core Engine   │    │
                    │  │  recall/store   │    │
                    │  └────────┬────────┘    │
                    │           │             │
                    │  ┌────────▼────────┐    │
                    │  │ LanceDB│NetworkX│    │
                    │  │ SQLite │        │    │
                    │  └─────────────────┘    │
                    └─────────────────────────┘
```

---

## Design Philosophy

### Core Principles

1. **Universal** - Works with ANY agentic coding tool, not locked to one ecosystem
2. **Local-First** - All data on local disk, no cloud dependency, your data stays yours
3. **Lightweight** - No Docker, no Java, no heavy dependencies—just Python
4. **Zero-Config** - `uv tool install memslice && memslice start -d` should just work
5. **Portable** - Copy `~/.memslice/` to migrate everything
6. **Open Source** - MIT license, community-driven, no vendor lock-in

### What We're NOT Building

- ❌ A LangChain/LangGraph wrapper (too heavy, unnecessary abstraction)
- ❌ A cloud service (local-first, your data stays yours)
- ❌ A tool-specific plugin (must work with ALL agentic coding tools)
- ❌ Another AI wrapper/chatbot (we're infrastructure, not interface)
- ❌ A monetization-first product (open source, community-first)

---

## Technical Architecture

### Technology Stack

| Component      | Choice         | Rationale                                                        |
| -------------- | -------------- | ---------------------------------------------------------------- |
| **Vector DB**  | LanceDB        | Embedded, Rust-based, no dependencies, millisecond cold start    |
| **Graph DB**   | NetworkX       | Pure Python, file-based (JSON), perfect for entity relationships |
| **Meta DB**    | SQLite         | Built-in Python, WAL mode for concurrency, battle-tested         |
| **Embeddings** | FastEmbed      | Local CPU inference, no PyTorch required                         |
| **API Server** | FastAPI        | Modern, fast, automatic OpenAPI docs                             |
| **MCP Server** | Python MCP SDK | Native Claude Desktop integration                                |

### Why LanceDB over ChromaDB?

See [lancedb-vs-chromadb.md](./lancedb-vs-chromadb.md) for detailed comparison.

Key reasons:

- True embedded (no background process)
- Minimal dependencies (no ONNXRuntime conflicts)
- Consistent with SQLite/NetworkX file-based philosophy
- Native multimodal support for future expansion

### Data Storage Layout

```
~/.memslice/                    # Or project-local ./data/
├── memslice.db                 # SQLite (config, logs, sessions)
├── graph.json                  # NetworkX (entities, relationships)
└── vectors/                    # LanceDB (embeddings)
    └── knowledge.lance/
```

### Memory Mapping

| Memory Type     | Storage            | Content Examples                                 |
| --------------- | ------------------ | ------------------------------------------------ |
| **Global**      | SQLite `config`    | "I prefer Python type hints", "Always use PEP8"  |
| **Long-term**   | LanceDB            | API documentation, code snippets, past solutions |
| **Medium-term** | NetworkX           | Project → uses → Python, Bug → blocks → Feature  |
| **Short-term**  | SQLite `chat_logs` | Last 10 conversation turns                       |

---

## API Design

### REST API Endpoints

```
POST /recall
  Body: { "query": string, "project"?: string, "limit"?: number }
  Returns: { "context": string, "sources": Source[] }

POST /memorize
  Body: { "text": string, "tags"?: string[], "type"?: "snippet"|"doc"|"decision" }
  Returns: { "id": string, "status": "ok" }

POST /link
  Body: { "from": string, "to": string, "relation": string }
  Returns: { "status": "ok" }

GET /projects
  Returns: { "projects": Project[] }

GET /project/:name
  Returns: { "status": string, "todos": string[], "context": string }

PUT /project/:name
  Body: { "status"?: string, "todos"?: string[] }
  Returns: { "status": "ok" }

DELETE /memory/:id
  Returns: { "status": "ok" }
```

### MCP Tools

For Claude Desktop integration:

| Tool                      | Description                           |
| ------------------------- | ------------------------------------- |
| `memslice_recall`         | Retrieve relevant context for a query |
| `memslice_remember`       | Store new information                 |
| `memslice_project_status` | Get/update current project state      |
| `memslice_link`           | Create entity relationships           |

---

## Supported Integrations

Memslice works with any tool that supports MCP or REST APIs:

| Tool | Integration | Status |
|------|-------------|--------|
| **Claude Code** | MCP Server | 🎯 Primary |
| **Cursor** | MCP Server | 🎯 Primary |
| **Codex CLI** | REST API | 🎯 Primary |
| **Gemini CLI** | REST API | 🎯 Primary |
| **Windsurf** | MCP Server | Planned |
| **Aider** | REST API | Planned |
| **Custom Agents** | REST API | ✅ Works |

### Integration Examples

**Claude Code / Cursor (MCP):**
```json
{
  "mcpServers": {
    "memslice": {
      "command": "memslice",
      "args": ["mcp"]
    }
  }
}
```

**Codex / Gemini CLI (REST):**
```bash
# Start the server
memslice start

# Tools can call the API
curl localhost:8999/recall -d '{"query": "auth implementation"}'
```

---

## Core Workflows

### Recall Flow (Query → Context)

```
User Query: "Fix the auth bug"
         │
         ▼
┌─────────────────────────────────────────────────┐
│ 1. Entity Extraction                            │
│    Identify: "auth", "bug" → Project: Auth-API  │
└─────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────┐
│ 2. Parallel Retrieval                           │
│    • LanceDB: semantic search "auth bug"        │
│    • NetworkX: Auth-API node → status, deps     │
│    • SQLite: recent conversation context        │
└─────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────┐
│ 3. Context Assembly                             │
│    [Project State]: Auth-API, status=debugging  │
│    [Related Docs]: JWT validation snippet       │
│    [Recent]: "Tried refresh token approach"     │
└─────────────────────────────────────────────────┘
```

### Memorize Flow (Response → Storage)

```
AI Response: "Fixed by adding token expiry check"
         │
         ▼
┌─────────────────────────────────────────────────┐
│ 1. Classification                               │
│    Type: bugfix, Project: Auth-API              │
└─────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────┐
│ 2. Processing                                   │
│    • Chunk text → embed → LanceDB              │
│    • Extract entities → NetworkX edges          │
│    • Log interaction → SQLite                   │
└─────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────┐
│ 3. Graph Update                                 │
│    Auth-API.status = "fixed"                    │
│    Auth-API → solved_by → "token expiry check"  │
└─────────────────────────────────────────────────┘
```

---

## Differentiation

### vs. Tool-Specific Memory

| Aspect | CLAUDE.md | .cursorrules | Memslice |
|--------|-----------|--------------|----------|
| **Scope** | Claude Code only | Cursor only | All tools |
| **Format** | Manual markdown | Manual text | Automatic + structured |
| **Semantic Search** | ❌ | ❌ | ✅ Vector search |
| **Relationships** | ❌ | ❌ | ✅ Graph (NetworkX) |
| **Cross-Project** | ❌ | ❌ | ✅ Shared memory |

### vs. claude-mem

See [claude-mem-analysis.md](./claude-mem-analysis.md) for detailed analysis.

| Aspect            | claude-mem                     | Memslice                    |
| ----------------- | ------------------------------ | --------------------------- |
| **Storage**       | SQLite + ChromaDB              | SQLite + LanceDB + NetworkX |
| **Graph Support** | No explicit entity graph       | NetworkX for relationships  |
| **Dependencies**  | Heavy (ChromaDB → ONNXRuntime) | Lightweight (LanceDB)       |
| **Target**        | Claude-only                    | Universal (any AI tool)     |
| **Search**        | Hybrid (vector + FTS)          | Hybrid + graph traversal    |

### Key Differentiators

1. **Universal** - One memory layer for all your coding tools
2. **Graph Relationships** - NetworkX enables "Project A uses Library B" queries
3. **Lighter Stack** - LanceDB vs ChromaDB reduces dependency hell
4. **Open Source** - MIT license, community-driven

---

## Project Structure

```
memslice/
├── data/                       # Default data directory
│   ├── memslice.db
│   ├── graph.json
│   └── vectors/
├── src/
│   ├── __init__.py
│   ├── core.py                 # Memslice main class
│   ├── storage/
│   │   ├── __init__.py
│   │   ├── vector.py           # LanceDB wrapper
│   │   ├── graph.py            # NetworkX wrapper
│   │   └── meta.py             # SQLite wrapper
│   ├── api/
│   │   ├── __init__.py
│   │   ├── server.py           # FastAPI app
│   │   └── mcp.py              # MCP server
│   └── utils/
│       ├── __init__.py
│       ├── chunker.py          # Text splitting
│       └── extractor.py        # Entity extraction
├── scripts/
│   ├── start.py                # memslice start
│   └── ingest.py               # Bulk document ingestion
├── tests/
├── requirements.txt
├── pyproject.toml
└── README.md
```

---

## Implementation Phases

### Phase 1: Core Engine (MVP)

**Goal:** Basic recall/memorize working locally

- [ ] SQLite schema and migrations
- [ ] LanceDB integration with FastEmbed
- [ ] NetworkX graph persistence
- [ ] Core `Memslice` class with `recall()` and `memorize()`
- [ ] Basic CLI for testing

**Deliverable:** `python -m memslice recall "auth bug"` returns context

### Phase 2: REST API

**Goal:** HTTP interface for custom agents

- [ ] FastAPI server with endpoints
- [ ] Project management routes
- [ ] OpenAPI documentation
- [ ] Basic authentication (optional)

**Deliverable:** `curl localhost:8999/recall -d '{"query":"auth"}'` works

### Phase 3: MCP Integration

**Goal:** Claude Desktop can use Memslice

- [ ] MCP server implementation
- [ ] Tool definitions (recall, remember, project_status)
- [ ] Installation instructions for Claude Desktop

**Deliverable:** Claude Desktop shows Memslice tools

### Phase 4: Intelligence Layer

**Goal:** Smarter entity extraction and linking

- [ ] LLM-powered entity extraction
- [ ] Automatic relationship inference
- [ ] Memory consolidation (merge similar memories)
- [ ] Importance scoring

**Deliverable:** Memslice automatically builds knowledge graph from conversations

---

## Success Metrics

### Technical Requirements

- [ ] Cold start < 500ms
- [ ] Recall latency < 100ms for 100K memories
- [ ] Support concurrent access (multiple tools simultaneously)
- [ ] Data portable (copy folder = full migration)
- [ ] Works offline (no internet required)
- [ ] Memory usage < 200MB idle

### Developer Experience

- [ ] One-command install: `uv tool install memslice`
- [ ] One-command start: `memslice start -d`
- [ ] Works with Claude Code out of the box
- [ ] Clear, copy-paste integration docs for each tool
- [ ] Helpful error messages

### Open Source Goals

- [ ] Comprehensive README with GIFs/demos
- [ ] GitHub stars (vanity metric, but indicates reach)
- [ ] Community contributions (PRs, issues, discussions)
- [ ] Mentions in AI coding tool communities
- [ ] Integration into popular dotfiles / starter configs

---

## Project Identity

### The Problem

AI tools typically use directory path as project identifier:

```
/Users/yumin/ventures/my-project  → has memories
              ↓ rename
/Users/yumin/ventures/better-name → memories lost!
```

### Solution: `.memslice.yml` Config File

Each project gets a **committed** config file with a stable UUID and optional settings:

```
my-project/
├── .memslice.yml          # Project identity + config
├── .git/
└── src/
```

**Minimal:**

```yaml
id: proj_a1b2c3d4
```

**Full example:**

```yaml
# .memslice.yml

id: proj_a1b2c3d4

# Optional metadata
name: "My Awesome Project"
tags:
  - backend
  - python

# Patterns to exclude from memory ingestion
ignore:
  - "*.log"
  - "node_modules/**"
  - ".env*"

# Related projects (for cross-project context)
related:
  - proj_xyz789 # shared library
  - proj_def456 # frontend counterpart

# Auto-capture settings
auto_memorize:
  decisions: true # capture architectural decisions
  todos: true # track TODOs
  errors: false # don't store error logs
```

### Design Principles

| Component       | Location       | Shared?       | Purpose                    |
| --------------- | -------------- | ------------- | -------------------------- |
| `.memslice.yml` | Project root   | ✅ Committed  | Project identity + config  |
| `~/.memslice/`  | Home directory | ❌ Local only | Personal memories database |

### Workflow

```
# Person A creates project
→ Memslice generates .memslice.yml with id: "proj_abc"
→ Person A commits .memslice.yml to git

# Person B clones repo
→ Sees existing .memslice.yml with id "proj_abc"
→ Checks local ~/.memslice/ DB → no memories for this ID
→ Starts fresh, but uses same ID

# Person A on new machine
→ Clones repo → sees "proj_abc" in .memslice.yml
→ Syncs personal ~/.memslice/ DB (optional)
→ All memories reconnect automatically
```

### Benefits

- **Rename-proof** - Directory name changes, UUID stays
- **Move-proof** - Move project anywhere, UUID stays
- **Clone-friendly** - Team shares ID, not memories
- **Multi-machine** - Sync your DB, memories follow you

### Identity Resolution Order

1. Look for `.memslice.yml` in project root → use existing ID
2. Not found? Generate new UUID → create `.memslice.yml`
3. Prompt user to commit `.memslice.yml` to version control

---

## Open Questions

- **Entity Extraction** - Use LLM (expensive) or rule-based (limited)?

  - LLM-first
  - user can choose to use rule-base, like regex

- **Memory Decay** - Should old memories fade or persist forever?

  - GREAT QUESTION
  - TBD

- **Multi-user** - Is this ever multi-user or always personal?

  - .memslice.yml is shared, config is shared
  - ~/.memslice/ directory is personal
  - so, no, always personal

- **Sync** - Optional cloud backup/sync across machines?

  - v2

- **Conflict Resolution** - What if someone changes `.memslice.yml` ID manually?
  - User's choice, user's burden

---

## References

- [Design Discussion with Gemini](./discuss-with-geminie.md)
- [claude-mem Analysis](./claude-mem-analysis.md)
- [LanceDB vs ChromaDB](./lancedb-vs-chromadb.md)
- [MCP Protocol](https://modelcontextprotocol.io/)
- [LanceDB Docs](https://lancedb.github.io/lancedb/)
- [NetworkX Docs](https://networkx.org/)
