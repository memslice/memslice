# Claude-Mem Analysis

> Analysis of [thedotmack/claude-mem](https://github.com/thedotmack/claude-mem) - A Claude Code plugin providing persistent memory across sessions.

## Overview

Claude-mem is a **production-grade memory system**, not a simple JSON notepad as initially assumed. It uses SQLite with full-text search, ChromaDB for vector search, and Claude Agent SDK for AI-powered observation synthesis.

## Project Structure

```
claude-mem/
├── src/
│   ├── hooks/          # TypeScript lifecycle hooks
│   ├── services/       # Core business logic (worker, database, search, sync)
│   ├── servers/        # MCP server implementation
│   ├── ui/             # React viewer interface
│   ├── sdk/            # Claude Agent SDK integration
│   └── types/          # Database and TypeScript type definitions
├── plugin/             # Built artifacts and plugin manifests
└── migrations/         # SQLite schema migrations
```

**Data Location:**
- Database: `~/.claude-mem/claude-mem.db`
- Vector DB: `~/.claude-mem/vector-db/`
- Settings: `~/.claude-mem/settings.json`

## Architecture

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────────────┐
│  Claude Desktop │────▶│   MCP Server     │────▶│   Worker Service        │
│                 │     │   (stdio)        │     │   (localhost:37777)     │
└─────────────────┘     └──────────────────┘     └───────────┬─────────────┘
                                                             │
                        ┌────────────────────────────────────┼────────────────────────────────────┐
                        │                                    │                                    │
                        ▼                                    ▼                                    ▼
              ┌─────────────────┐                  ┌─────────────────┐                  ┌─────────────────┐
              │     SQLite      │                  │    ChromaDB     │                  │  Claude Agent   │
              │  (FTS5 search)  │                  │ (vector search) │                  │      SDK        │
              └─────────────────┘                  └─────────────────┘                  └─────────────────┘
```

## Data Storage (SQLite + ChromaDB)

### SQLite Database Schema

**Core Tables:**
- `sessions` - Session metadata and archiving info
- `memories` - Compressed memory chunks with hierarchical fields
- `observations` - Extracted observations with types
- `session_summaries` - Structured summaries (request/investigated/learned/completed/next_steps)
- `transcript_events` - Raw conversation events
- `user_prompts` - Individual user prompts with numbering
- `observations_fts` - FTS5 virtual table for semantic search

**Database Configuration:**
```typescript
// Optimized SQLite settings
PRAGMA journal_mode = WAL    // Write-Ahead Logging for concurrency
PRAGMA synchronous = NORMAL
PRAGMA mmap_size = 256MB     // Memory-mapped I/O
PRAGMA cache_size = 10000    // 10,000 pages cache
```

### Observation Structure

```typescript
interface ParsedObservation {
  type: string;           // decision|bugfix|feature|refactor|discovery|change
  title: string | null;
  subtitle: string | null;
  facts: string[];        // JSON array
  narrative: string | null;
  concepts: string[];     // JSON array (tags)
  files_read: string[];
  files_modified: string[];
}
```

## MCP Tools Exposed

### Search & Retrieval Tools

| Tool | Description |
|------|-------------|
| `search()` | Full-text search with filters (query, type, concepts, files, date range) |
| `timeline()` | Timeline context with anchor-based retrieval |
| `get_recent_context()` | Recent observations with optional filters |
| `get_context_timeline()` | Timeline around specific observation ID |

### Data Access Tools

| Tool | Description |
|------|-------------|
| `get_observation(id)` | Fetch single observation by ID |
| `get_observations(ids)` | Batch fetch multiple observations |
| `get_session(id)` | Fetch session metadata |
| `get_prompt(id)` | Fetch user prompt by ID |

### Utility Tools

| Tool | Description |
|------|-------------|
| `get_schema(tool_name)` | Returns parameter schema for tools |
| `help()` | Detailed documentation |

## Session Lifecycle

```
SessionStart (context-hook)     → Injects relevant past observations
        ↓
PostToolUse (save-hook)         → Captures tool usage observations
        ↓
Stop (summary-hook)             → Triggers session summary generation
        ↓
SessionEnd (cleanup-hook)       → Marks session complete
```

## Search Capabilities

**Hybrid Search Approach:**
- **Full-Text Search (FTS5)** - SQLite native keyword matching
- **Vector Search (ChromaDB)** - Semantic similarity search
- **Combined scoring** - Results merged and ranked from both sources

**Search Parameters:**
- `query` - Full-text search string
- `type` - Filter by observation type
- `concepts` - Comma-separated tags
- `files` - File path filters
- `dateStart`/`dateEnd` - Date range
- `limit`, `offset` - Pagination
- `project` - Project name filter

## Key Technologies

| Component | Technology |
|-----------|------------|
| Database | SQLite3 (Bun native) with FTS5 |
| Vector Store | ChromaDB |
| AI Synthesis | @anthropic-ai/claude-agent-sdk |
| API Server | Express.js (Worker on :37777) |
| MCP Protocol | stdio JSON-RPC |
| Frontend | React (viewer UI) |

## Data Flow

### Remember (Save) Flow

```
PostToolUse Hook (fire-and-forget)
    ↓
POST /api/sessions/observations
    ↓
SessionManager (queue tool observations)
    ↓
SDKAgent (async Claude synthesis)
    ↓
parseObservations() (XML parsing)
    ↓
SQLite (persisted) + ChromaDB (indexed)
```

### Recall (Search) Flow

```
MCP: search(query, filters)
    ↓
Worker: /api/search?query=...
    ↓
SearchManager.search()
    ↓
[Parallel] ChromaDB.query() + SQLite FTS5
    ↓
[Merge] Hybrid results with ranking
    ↓
Return to Claude via MCP
```

## Comparison: Initial Assumptions vs Reality

| Initial Assumption | Actual Implementation |
|--------------------|----------------------|
| JSON file storage | SQLite + ChromaDB |
| Key-value pairs | Structured schema with 6+ migrations |
| Load entire file into memory | Paginated queries with offset/limit |
| Mutex file access | WAL mode for concurrent access |
| Simple keyword search | Hybrid vector + FTS5 search |
| "Notepad plugin" | Production-grade system with AI synthesis |

## What Memslice Can Learn

**Already implemented in claude-mem:**
- SQLite for metadata/logs
- Vector search (ChromaDB)
- MCP protocol integration
- Session lifecycle management
- Observation typing and tagging

**Potential differentiators for Memslice:**
- **NetworkX for graph relationships** - claude-mem lacks explicit entity-relationship graphs
- **LanceDB** - Lighter than ChromaDB (no Python dependency)
- **Cross-tool support** - claude-mem is Claude-specific; Memslice aims for universal service
- **Simpler architecture** - claude-mem has complex Worker/MCP separation

## Conclusion

Claude-mem is far more sophisticated than a "JSON notepad". It's a well-architected memory system with:
- Proper database design with migrations
- Hybrid search (vector + keyword)
- AI-powered observation synthesis
- Concurrent access support
- Privacy controls

The main opportunity for Memslice is in providing a **simpler, lighter-weight alternative** with **graph-based relationships** and **universal tool support** beyond just Claude.
