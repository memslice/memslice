# LanceDB vs ChromaDB Analysis

> Comparison for Memslice's vector database choice.

## Gemini's Analysis Accuracy

| Claim | Verdict | Notes |
|-------|---------|-------|
| LanceDB is embedded/serverless | ✅ **Accurate** | True in-process, no background service |
| LanceDB is Rust-based | ✅ **Accurate** | Built in Rust for speed and safety |
| LanceDB stores data as files | ✅ **Accurate** | `.lance` folders, Arrow-based |
| ChromaDB has heavy dependencies | ✅ **Accurate** | Requires ONNXRuntime, causes [known conflicts](https://github.com/chroma-core/chroma/issues/1627) with Python 3.12+ |
| ChromaDB needs Docker for stability | ⚠️ **Partially** | Has embedded mode, but [client-server is recommended](https://cookbook.chromadb.dev/core/install/) for production |
| LanceDB is faster | ✅ **Accurate** | [100x faster than Parquet](https://medium.com/@patricklenert/vector-databases-lance-vs-chroma-cc8d124372e9), used by Midjourney |
| ChromaDB's DX is simpler | ✅ **Accurate** | Better LangChain integration, more tutorials |

## Head-to-Head Comparison

| Aspect | LanceDB | ChromaDB |
|--------|---------|----------|
| **Language** | Rust | C++ (hnswlib) + Python |
| **Storage Format** | Lance (evolved Parquet) | Parquet + hnswlib |
| **Dependencies** | Lightweight | Heavy (ONNXRuntime, tokenizers, grpcio, etc.) |
| **Cold Start** | Milliseconds | Slower (loads embedding models) |
| **Multimodal** | Native (text, images, video) | Primarily text/embeddings |
| **GPU Support** | SIMD + GPU acceleration | Limited |
| **Data Versioning** | Built-in automatic versioning | Not native |
| **Pandas/Polars** | Native Arrow integration | Requires conversion |
| **Community** | Smaller but growing | Larger, LangChain favorite |
| **Market Rank** | #5 in vector DBs | #8 in vector DBs |

## Architecture Comparison

### LanceDB

- **Server Model**: Embedded, serverless - database tightly coupled with application code
- **Implementation**: Rust from the ground up for speed, storage security, and low resource utilization
- **Storage**: Lance data format (evolution of Parquet) with fragments for efficient querying
- **Performance**: Vector queries 100x faster than Parquet, 1B vectors in <100ms on MacBook

### ChromaDB

- **Server Model**: Uses Clickhouse internally + hnswlib for vector search
- **Implementation**: C++ (via hnswlib) with Python bindings
- **Storage**: Parquet format with column-oriented compression
- **Performance**: Fast for prototyping, may slow under heavy loads

## Dependency Analysis

### LanceDB

```bash
pip install lancedb
# Minimal dependencies, lightweight installation
```

### ChromaDB

```bash
pip install chromadb
# Heavy dependencies including:
# - onnxruntime>=1.14.1 (causes Python 3.12+ conflicts)
# - tokenizers
# - grpcio
# - httpx
# - numpy
# - bcrypt
# - build
```

**Known Issues:**
- [ONNXRuntime conflicts with Python 3.12+](https://github.com/chroma-core/chroma/issues/1627)
- [CrewAI dependency issues](https://github.com/crewAIInc/crewAI/issues/679)
- Thin client (`chromadb-client`) available but loses default embedding function

## For Memslice: LanceDB is the Right Choice

**Recommendation: Choose LanceDB**

### Alignment with Memslice Goals

1. **Local-First Philosophy**
   - LanceDB: True embedded, data is just files
   - ChromaDB: Embedded mode exists but feels like "local server simulation"

2. **Dependency Hell Avoidance**
   - LanceDB: `pip install lancedb` - minimal deps
   - ChromaDB: ONNXRuntime conflicts are well-documented pain

3. **Consistency with Stack**
   - SQLite → file-based relational
   - NetworkX → file-based graph (JSON)
   - LanceDB → file-based vectors (Lance)
   - All three: no background processes, portable folder

4. **Performance for Personal Use**
   - LanceDB: 1B vectors in <100ms on MacBook
   - More than enough for personal memory system

5. **Multimodal Future**
   - If you later want to store code screenshots, diagrams, etc.
   - LanceDB handles this natively

### When to Choose ChromaDB Instead

- Heavily invested in LangChain ecosystem
- Need maximum community support/tutorials
- Want compatibility with claude-mem codebase (which uses ChromaDB)

## Code Comparison

### LanceDB (Pythonic)

```python
import lancedb

# Connect = just a folder path
db = lancedb.connect("./vectors")

# Data as dict list or pandas df
data = [
    {"vector": [1.3, 1.4], "text": "Document", "source": "my_source"},
]

# Auto schema inference
table = db.create_table("memory", data)

# Chainable search API
results = table.search([1.3, 1.4]).limit(5).to_pandas()
```

### ChromaDB (More Verbose)

```python
import chromadb

# Even local mode has Settings complexity
client = chromadb.PersistentClient(path="./chroma_db")
collection = client.get_or_create_collection(name="memory")

# Must manually manage IDs
collection.add(
    documents=["Document"],
    metadatas=[{"source": "my_source"}],
    ids=["id1"]  # Required manual ID management
)

# Query
results = collection.query(query_texts=["search"], n_results=5)
```

## Sources

- [Vector Databases: Lance vs Chroma - Medium](https://medium.com/@patricklenert/vector-databases-lance-vs-chroma-cc8d124372e9)
- [Chroma vs LanceDB 2025 - PeerSpot](https://www.peerspot.com/products/comparisons/chroma_vs_lancedb)
- [Chroma vs LanceDB - Zilliz](https://zilliz.com/comparison/chroma-vs-lancedb)
- [LanceDB vs ChromaDB - Restack](https://www.restack.io/p/lancedb-knowledge-lancedb-vs-chromadb-cat-ai)
- [ChromaDB Installation - Cookbook](https://cookbook.chromadb.dev/core/install/)
- [ChromaDB ONNXRuntime Issues](https://github.com/chroma-core/chroma/issues/1627)
- [5 Powerful Vector Database Tools for 2025](https://cybergarden.au/blog/5-powerful-vector-database-tools-2025)
