# Memslice MVP: Local Codebase Memory

## Problem Statement

LLMs confidently hallucinate about code they haven't read. They rely on "parametric memory" (training weights) which is:
- **Lossy** - compressed, incomplete
- **Outdated** - frozen at training time
- **Probabilistic** - guesses based on patterns, not facts

The solution: force grounding through RAG with actual source code.

## MVP Scope

**One sentence:** Index the user's local codebase and answer questions with real code citations.

### What We Build

```
Local codebase → chunk files → embed → LanceDB → RAG retrieval → grounded answers
```

| Component | Description | Effort |
|-----------|-------------|--------|
| `ingest_directory` | Recursively read files, chunk, embed, store | 1-2 days |
| Simple chunking | Split by file or double-newline (no AST) | Hours |
| LanceDB storage | Store chunks with embeddings + metadata | Already designed |
| `query` tool | Retrieve relevant chunks, inject into context | 1 day |

### What We Skip (For Now)

- AST-based parsing (use naive text splitting)
- Graph relationships between code entities
- Git/GitHub integration (user has code locally)
- Multi-repo support
- Sophisticated re-ranking

## Demo Flow

```
User: "Ingest my ./src folder"
Memslice: Indexed 47 files, 312 chunks

User: "How does error handling work in this project?"
Memslice: [Retrieves 5 relevant chunks from actual code]
         "Based on src/utils/errors.ts:23, you use a custom
          ErrorHandler class that wraps all async operations..."
```

## Why This First?

1. **Immediate value** - Solves the hallucination problem directly
2. **Low complexity** - No external dependencies (git, network)
3. **Proves the architecture** - LanceDB + embeddings + RAG working end-to-end
4. **Same value prop as Cursor** - But as an MCP tool for any Claude client

## Success Criteria

- [ ] User can ingest a local directory
- [ ] User can query and get answers citing real code
- [ ] Answers include file paths and line numbers
- [ ] No hallucination about code structure

## Next Iterations (Post-MVP)

1. **Better chunking** - Tree-sitter for language-aware splitting
2. **Code-specific embeddings** - Models trained on code, not just text
3. **Repo ingestion** - Clone from GitHub URLs
4. **Relationship graph** - Track imports, calls, inheritance
5. **Incremental updates** - Watch for file changes, re-index

---

*This is the weekend project that proves Memslice prevents hallucination. Ship this first.*
