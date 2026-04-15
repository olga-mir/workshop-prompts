# Step 5 Prompt: Local RAG Integration with LlamaIndex

**Objective:** Integrate local Retrieval-Augmented Generation (RAG) using LlamaIndex and Ollama embeddings, enabling agents to search and reference documents from a knowledge base.

**Status:** Ready for Implementation

**Target Models:** Ollama (local mode) - Documents stored locally in `internal-data/` folder

---

## Prerequisites

- Step 1-4 implementation complete (DEMO_MODE switching, multi-agent handovers)
- Ollama running with embedding model `mxbai-embed-large:335m` pulled
- LlamaIndex dependencies already in pyproject.toml
- `search_internal_data()` stub already in tools.py

---

## Overview

This step implements local RAG that:
- Reads documents from `internal-data/` folder (participant-provided files)
- Generates embeddings using Ollama embedding model
- Creates a vector index using LlamaIndex
- Enables agents to search knowledge base for relevant information
- Returns matching document chunks to agents

---

## Implementation Steps

### 1. Create `rag.py` Module

**File:** `blog-writer/blogger_agent/rag.py`

This is the core RAG implementation. Create with the following structure:

```python
# Imports
import os
from dotenv import load_dotenv
from llama_index.core import VectorStoreIndex, SimpleDirectoryReader, Settings
from llama_index.core.node_parser import SimpleNodeParser
from llama_index.embeddings.ollama import OllamaEmbedding

# Load environment variables
# (Load from .env.ollama for DEMO_MODE=local)

# 1. Embedding Model Function
def _get_embedding_model() -> OllamaEmbedding:
    """Get configured embedding model from environment."""
    # Read EMBEDDING_MODEL from env
    # Remove "ollama/" prefix if present
    # Return OllamaEmbedding with model_name

# 2. Build Index Function
def _build_rag_index():
    """Build vector index from internal-data directory."""
    # Check if internal-data/ exists
    # Load all documents using SimpleDirectoryReader
    # Configure embedding model
    # Split documents into chunks (256 tokens, 20 overlap)
    # Create and return VectorStoreIndex

# 3. Lazy Initialization
_rag_index = None

def _get_rag_index() -> VectorStoreIndex | None:
    """Get or create RAG index (lazy initialization)."""
    # Create index on first call
    # Cache for subsequent calls
    # Handle exceptions gracefully

# 4. Main Search Function
def search_internal_data(query: str) -> str:
    """Search internal knowledge base.
    
    Main function called by agents.
    - Gets the RAG index
    - Uses retriever to find relevant nodes
    - Combines and returns results as string
    """
```

**Key Details:**
- Load environment from `.env.ollama` (for local mode)
- Embedding model: Ollama (from EMBEDDING_MODEL env var)
- Chunk size: 256 tokens with 20 token overlap
- Lazy initialization: Index created on first search
- Error handling: Returns helpful message if no docs found

---

### 2. Create `internal-data/` Directory with Sample Files

**Location:** Project root → `internal-data/`

**Structure:**
```
internal-data/
├── document1.md (participant provides)
├── document2.md (participant provides)
└── ... (more markdown files)
```

**For Testing:**
Copy sample markdown files from reference:
- `mastering_network_timeouts_and_retries_in_go__*.md`
- `retries_strategies_in_distributed_systems.md`

Or create simple test files with:
- Technical content
- Multiple sections
- Keyword-rich content for retrieval testing

**Note:** Participants can replace these with their own files

---

### 3. Update `tools.py` to Use Real RAG

**File:** `blog-writer/blogger_agent/tools.py`

**Current State:**
```python
def search_internal_data(query: str) -> str:
    return f"Knowledge base search for '{query}' - placeholder"
```

**Change To:**
```python
from .rag import search_internal_data as rag_search_internal_data

# Replace placeholder with import
search_internal_data = rag_search_internal_data
```

**Why:**
- Points to actual RAG implementation instead of placeholder
- Maintains same function signature for agent tools
- Enables `search_internal_data()` tool in agents

---

### 4. Verify Dependencies

**Check `pyproject.toml` includes:**
```toml
dependencies = [
    ...
    "llama-index-core>=0.14.20",
    "llama-index-embeddings-ollama>=0.9.0",
    ...
]
```

**These should already be present from reference project**

---

## Testing Checklist

### Pre-Test Setup
- [ ] Ollama running: `ollama serve`
- [ ] Embedding model pulled: `ollama pull mxbai-embed-large:335m`
- [ ] `internal-data/` folder exists with sample .md files
- [ ] `rag.py` created in `blogger_agent/`
- [ ] `tools.py` updated to import from rag.py

### Build Test
- [ ] Build succeeds: `DEMO_MODE=local uv run adk web blog-writer`
- [ ] No import errors
- [ ] No type hint errors
- [ ] Server starts on http://127.0.0.1:8000

### Functional Test (Manual)
1. Start server: `DEMO_MODE=local uv run adk web blog-writer`
2. Open http://127.0.0.1:8000
3. Ask agent: "Write a blog about network timeouts and retries"
4. Observe:
   - Agent calls `search_internal_data()`
   - Retrieval returns relevant document chunks
   - Agent incorporates findings into blog post
   - Multi-agent workflow continues (outline → write → edit)

### Expected Behavior
- ✅ First request takes longer (index building with Ollama embeddings)
- ✅ Subsequent requests faster (cached index)
- ✅ Relevant documents returned for searches
- ✅ Agent uses search results in generated content
- ✅ Agent workflow continues with multi-agent handovers

---

## Key Implementation Details

### LlamaIndex Components
- **SimpleDirectoryReader**: Loads all files from directory
- **SimpleNodeParser**: Splits documents into chunks
- **VectorStoreIndex**: Creates in-memory vector index
- **OllamaEmbedding**: Generates embeddings using local Ollama

### Chunk Configuration
```
chunk_size=256      # 256 tokens per chunk
chunk_overlap=20    # 20 tokens overlap between chunks
```

**Why these values:**
- 256 tokens ≈ ~1000 chars, fits in context window
- 20 token overlap provides continuity
- Balances granularity and context

### Lazy Initialization Pattern
```python
_rag_index = None  # Cache variable

def _get_rag_index():
    global _rag_index
    if _rag_index is None:
        _rag_index = _build_rag_index()
    return _rag_index
```

**Benefits:**
- Index built only on first search (not on import)
- Cached for subsequent searches
- Graceful handling if no documents found

---

## Troubleshooting

### Issue: "No relevant documents found"
**Possible Causes:**
- No files in `internal-data/` folder
- Query too specific for document content
- Embedding model not returning good matches

**Solutions:**
- Verify files exist: `ls internal-data/`
- Try broader search queries
- Check embedding model is running: `curl http://localhost:11434/api/tags`
- Inspect document content manually

### Issue: "Internal knowledge base is not available"
**Possible Causes:**
- `internal-data/` folder doesn't exist
- Folder exists but is empty
- RAG index initialization failed

**Solutions:**
- Create folder: `mkdir internal-data`
- Add markdown files to folder
- Check logs for initialization errors
- Verify permissions: `ls -la internal-data/`

### Issue: Import errors (ModuleNotFoundError)
**Possible Causes:**
- `rag.py` not created
- Wrong import path in `tools.py`
- LlamaIndex not installed

**Solutions:**
- Verify `rag.py` exists: `ls blog-writer/blogger_agent/rag.py`
- Check import in tools.py: `from .rag import search_internal_data`
- Reinstall: `uv sync`

### Issue: Ollama embedding not working
**Possible Causes:**
- Ollama not running
- Embedding model not pulled
- Connection to localhost:11434 fails

**Solutions:**
- Start Ollama: `ollama serve`
- Pull model: `ollama pull mxbai-embed-large:335m`
- Test connection: `curl http://localhost:11434/api/tags`
- Check .env.ollama has EMBEDDING_MODEL set

---

## How Participants Use This

### For Workshop Participants

1. **Prepare Documents:**
   - Collect technical documents (markdown preferred)
   - Save to `internal-data/` folder
   - Filename examples: `my-article.md`, `guide.md`

2. **Run Agent:**
   ```bash
   DEMO_MODE=local uv run adk web blog-writer
   ```

3. **Query Knowledge Base:**
   - Ask agent: "Based on my documents, write about..."
   - Agent searches knowledge base
   - Results incorporated into generated content

### For Cloud Deployment

Note: This step covers local RAG only. Cloud RAG (Vertex AI) is future step.

---

## Files Modified/Created

| File | Status | Change |
|------|--------|--------|
| `blog-writer/blogger_agent/rag.py` | NEW | LlamaIndex RAG implementation |
| `blog-writer/blogger_agent/tools.py` | MODIFIED | Import from rag.py |
| `internal-data/` | NEW | Directory for user documents |
| `internal-data/*.md` | NEW | Sample markdown files |

---

## Success Criteria

✅ **Implementation Complete When:**
1. `rag.py` created with all functions
2. `internal-data/` folder exists with sample files
3. `tools.py` imports `search_internal_data` from rag.py
4. Build succeeds: `DEMO_MODE=local uv run adk web blog-writer`
5. No import or type hint errors
6. Server starts without errors

✅ **Functional Success When:**
1. Agent can call `search_internal_data()`
2. Retrieval returns relevant document snippets
3. Multiple searches return consistent results
4. Agent incorporates results into blog content

---

## Key Principles

1. **Lazy Initialization** - Index only built on first use
2. **Graceful Degradation** - Works without documents, returns helpful message
3. **Local First** - No cloud dependencies for local RAG
4. **Participant Flexibility** - Users provide their own documents
5. **Multi-Agent Integration** - Search results fed to agents for creativity

---

## References

- **LlamaIndex Docs**: https://docs.llamaindex.ai/
- **Ollama Embeddings**: https://docs.llamaindex.ai/integrations/embeddings/ollama/
- **Reference Implementation**: `/Users/olga/repos/agentic-rag-intro-workshop/blog-writer/blogger_agent/rag.py`
- **Sample Documents**: `/Users/olga/repos/agentic-rag-intro-workshop/internal-data/`

---

## Next Steps (Step 5b - Future)

After local RAG is working, next steps can include:
- Cloud RAG with Vertex AI (create_corpus.py)
- Hybrid RAG (local + cloud)
- Search optimization
- Caching strategies
- Document preprocessing

---

## Quick Copy-Paste: Complete rag.py

See implementation example in reference:
`/Users/olga/repos/agentic-rag-intro-workshop/blog-writer/blogger_agent/rag.py`

Key sections:
- Imports (llama_index, ollama)
- `_get_embedding_model()` function
- `_build_rag_index()` function
- Lazy initialization with global `_rag_index`
- `search_internal_data()` main function

---

**Status**: Ready to implement - all prerequisites met, dependencies available, reference implementation tested
