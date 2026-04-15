# Adapting ADK Samples to a Custom Project Structure

This guide documents how the Agentic RAG workshop was adapted from the [Google ADK Blogger Agent Sample](https://github.com/google/adk-samples/tree/main/python/agents/blog-writer) to support local execution with Ollama models and a consolidated project structure.

## Overview of Changes

The adaptation involved three main areas:
1. **Project Structure**: Consolidating dependencies into a single `pyproject.toml` at the root level
2. **Local Model Support**: Adding LiteLLM to enable local Ollama model execution
3. **Search Tool Adaptation**: Replacing Google Search with DuckDuckGo wrapped in a custom tool
4. **RAG Integration**: Adding LlamaIndex for local vector-based retrieval augmented generation

---

## 1. Project Structure Consolidation

### Original Structure (adk-samples)
```
blog-writer/
├── pyproject.toml        (project-specific dependencies)
├── blogger_agent/
│   └── ...
└── tests/
```

### Adapted Structure
```
.
├── pyproject.toml        (root-level, all dependencies)
├── README.md
├── docs/
│   └── adapting-adk-samples.md
├── blog-writer/
│   ├── README.md         (simplified, references root)
│   └── blogger_agent/
│       └── ...
└── internal-data/        (RAG knowledge base)
```

### Why This Matters
- **Single dependency source**: All packages managed at root level via `uv sync`
- **Simplified setup**: No nested virtual environments or dual dependency management
- **Clearer structure**: `blog-writer/` is clearly a subdirectory of the workshop project

### Implementation Steps

1. **Consolidate `pyproject.toml`**:
   - Merge `blog-writer/pyproject.toml` into root `pyproject.toml`
   - Keep all tool configurations (ruff, mypy, pytest, etc.)
   - Adjust Python version requirement to be compatible (e.g., `>=3.10,<3.13`)

2. **Remove nested files**:
   - Delete `blog-writer/pyproject.toml`
   - Delete `blog-writer/uv.lock`
   - Update `blog-writer/README.md` to reference root README

3. **Update imports** (if needed):
   - Adjust `[tool.ruff.lint.isort]` to use correct `known-first-party` paths
   - For this project: `known-first-party = ["blogger_agent"]` (unchanged)

---

## 2. Adding Local Ollama Model Support

### Dependencies to Add

```toml
[project]
dependencies = [
    # ... existing dependencies ...
    "litellm>=1.82.6",
    "google-adk>=1.28.1",  # Ensure version supports LiteLlm
    # ... other dependencies ...
]
```

### Key Files Modified

#### A. Sub-Agents (all agents in `blogger_agent/sub_agents/`)

**Pattern**: Wrap config models with `LiteLlm` class

**Before**:
```python
from google.adk.agents import Agent

agent = Agent(
    model=config.worker_model,  # e.g., "gemini-2.5-flash"
    # ... rest of config
)
```

**After**:
```python
from google.adk.agents import Agent
from google.adk.models.lite_llm import LiteLlm
import uuid  # noqa: F401 - Required for type hint resolution in Google ADK

agent = Agent(
    model=LiteLlm(config.worker_model),  # Wraps model string for LiteLLM
    # ... rest of config
)
```

**Why**:
- `LiteLlm` allows ADK to use non-Vertex AI models
- `uuid` import required by Google ADK for type hint resolution
- Applies to all agents: `blog_planner.py`, `blog_writer.py`, `blog_editor.py`, `social_media_writer.py`

#### B. Config File (`config.py`)

**Key Settings**:
- For **local Ollama**: `WORKER_MODEL=ollama/gemma4:e4b` (format: `ollama/<model-name>`)
- For **Vertex AI**: `WORKER_MODEL=gemini-2.5-flash`

**Configuration Loading**:
```python
@dataclass
class ResearchConfiguration:
    worker_model: str = os.environ.get("WORKER_MODEL", "gemini-2.5-flash")
    critic_model: str = os.environ.get("CRITIC_MODEL", "gemini-2.5-pro")
    embedding_model: str = os.environ.get("EMBEDDING_MODEL", "gemini-embedding-2-preview")
```

**For Local Ollama**:
Create `.env` in `blog-writer/blogger_agent/`:
```env
WORKER_MODEL=ollama/gemma4:e4b
CRITIC_MODEL=ollama/gemma4:e4b
EMBEDDING_MODEL=ollama/mxbai-embed-large:335m
```

---

## 3. Search Tool Adaptation

### Original (Google Search)
```python
from google.adk.tools import google_search

agent = Agent(
    tools=[google_search],  # Built-in Google Search tool
    # ...
)
```

### Adapted (DuckDuckGo with Wrapper)

#### Why Custom Wrapper?
The `DuckDuckGoSearchRun` from LangChain has type hints referencing `uuid`, but uuid isn't in the langchain namespace. When Google ADK analyzes the tool's type hints, it fails. A wrapper with explicit type hints solves this.

#### Implementation in `tools.py`:

```python
import uuid
from langchain_community.tools import DuckDuckGoSearchRun

def search(query: str) -> str:
    """Search the web using DuckDuckGo.

    This wrapper function exists because DuckDuckGoSearchRun.run() has type hints
    that reference 'uuid', but uuid is not imported in langchain_community's namespace.
    When Google ADK's FunctionTool tries to analyze this function's type hints using
    typing.get_type_hints(), it fails with "NameError: name 'uuid' is not defined".

    By wrapping it with explicit type hints, we avoid this issue during tool declaration.
    """
    search_tool = DuckDuckGoSearchRun()
    return search_tool.run(query)
```

#### Usage in Agents:

```python
from ..tools import search

agent = Agent(
    tools=[search],  # Use wrapped search() instead of google_search
    # ...
)
```

#### Dependencies:
```toml
dependencies = [
    "langchain-community>=0.4.1",
    "ddgs",  # DuckDuckGo search backend
    # ...
]
```

---

## 4. RAG Integration with LlamaIndex

### Adding RAG Capability

#### Dependencies:
```toml
dependencies = [
    "llama-index-core>=0.14.20",
    "llama-index-embeddings-ollama>=0.9.0",
    # ... other dependencies ...
]
```

#### Implementation (`rag.py`):

```python
from llama_index.core import VectorStoreIndex, SimpleDirectoryReader

def _initialize_rag():
    """Initialize the RAG index from documents in the internal-data directory."""
    data_dir = os.path.join(os.path.dirname(__file__), "..", "..", "internal-data")
    if not os.path.exists(data_dir):
        return None

    documents = SimpleDirectoryReader(data_dir).load_data()
    if not documents:
        return None

    index = VectorStoreIndex.from_documents(documents)
    return index.as_query_engine()

_query_engine = _initialize_rag()

def search_internal_data(query: str) -> str:
    """Search the internal knowledge base using LlamaIndex RAG."""
    if _query_engine is None:
        return "Internal knowledge base is not available."

    response = _query_engine.query(query)
    return str(response)
```

#### Expose as Tool:

In `tools.py`:
```python
from .rag import search_internal_data

# search_internal_data is now available for export
```

In `agent.py`:
```python
from .tools import search_internal_data
from google.adk.tools import FunctionTool

agent = Agent(
    tools=[
        FunctionTool(search_internal_data),
        FunctionTool(analyze_codebase),
        # ... other tools
    ],
    # ...
)
```

#### Data Storage:
- Place `.md` or `.txt` files in `internal-data/` directory
- LlamaIndex will automatically index them on first load
- Supports hierarchical directory structures

---

## 5. Configuration Management

### Environment Variables

Create `.env` in `blog-writer/blogger_agent/`:

**For Local Ollama Development**:
```env
# Local Ollama models
WORKER_MODEL=ollama/gemma4:e4b
CRITIC_MODEL=ollama/gemma4:e4b
EMBEDDING_MODEL=ollama/mxbai-embed-large:335m

# GCP (optional, for Stage 2 deployment)
# GOOGLE_CLOUD_PROJECT=your-project-id
# GCS_BUCKET=your-bucket
# RAG_CORPUS=your-corpus-id
```

**For Vertex AI Deployment** (Stage 2):
```env
WORKER_MODEL=gemini-2.5-flash
CRITIC_MODEL=gemini-2.5-pro
EMBEDDING_MODEL=gemini-embedding-2-preview

GOOGLE_CLOUD_PROJECT=your-project-id
GCS_BUCKET=your-bucket
RAG_CORPUS=your-corpus-id
```

### Why `.env` in `blogger_agent/` Directory?
The `config.py` loads from this location specifically to avoid conflicts with `gemini-cli` which automatically reads `.env` from the repository root.

---

## 6. Prerequisites and Setup

### System Requirements
- Python 3.10+
- [uv](https://github.com/astral-sh/uv) package manager
- [Ollama](https://ollama.com/) (for local development)

### Local Ollama Setup
```bash
# Install Ollama from https://ollama.com/

# Start Ollama server (in separate terminal)
ollama serve

# In another terminal, pull required models
ollama pull gemma4:e4b
ollama pull mxbai-embed-large:335m
```

### Project Setup
```bash
# Root level setup
uv sync

# Run the agent
uv run adk web blog-writer/blogger_agent
```

**Important**: Always use `uv run adk` instead of `adk` directly. Running `adk` directly can cause it to pick up packages from the global ADK venv instead of the project's virtual environment.

---

## 7. Testing Ollama Integration

### Verify Ollama is Running
```bash
curl -X POST http://localhost:11434/api/embed \
  -H "Content-Type: application/json" \
  -d '{
    "model": "mxbai-embed-large:335m",
    "input": "test"
  }'
```

### Test Agent with Ollama
```bash
uv run adk web blog-writer/blogger_agent
# Navigate to http://localhost:8000 and test with a query
```

---

## 8. Migration Checklist

When adapting another ADK sample to this pattern:

- [ ] Consolidate `pyproject.toml` files into single root file
- [ ] Remove nested `pyproject.toml` and `uv.lock` files
- [ ] Add `litellm>=1.82.6` to dependencies
- [ ] Wrap all agent `model=` definitions with `LiteLlm()`
- [ ] Add `import uuid` to files using LiteLlm
- [ ] Replace Google Search with wrapped search tool from `langchain_community`
- [ ] Create wrapper function for search tool (to avoid uuid type hint issues)
- [ ] Update `.env` configuration for local models
- [ ] Test with `uv run adk web` before deploying
- [ ] Document any custom tools added
- [ ] Update sub-agent instructions to reference new tools

---

## 9. Troubleshooting

### "uuid is not defined" Error
**Cause**: Using `DuckDuckGoSearchRun` directly as a tool
**Fix**: Wrap it in a function with explicit type hints (see Search Tool Adaptation section)

### "Module not found" with `uv run adk`
**Cause**: Dependencies not synced or running `adk` directly
**Fix**:
```bash
# Ensure dependencies are synced
uv sync

# Always use uv run
uv run adk web blog-writer/blogger_agent
```

### Ollama Models Not Loading
**Cause**: Ollama server not running or models not pulled
**Fix**:
```bash
# Check Ollama is running
curl http://localhost:11434/api/tags

# Pull missing models
ollama pull gemma4:e4b
ollama pull mxbai-embed-large:335m
```

### RAG Not Finding Documents
**Cause**: `internal-data/` directory empty or in wrong location
**Fix**:
```bash
# Verify directory exists at project root
ls -la internal-data/

# Add markdown files
cp your-documents.md internal-data/
```

---

## References

- [Google ADK Documentation](https://cloud.google.com/docs/agent-engine)
- [LiteLLM Documentation](https://docs.litellm.ai/)
- [LlamaIndex Documentation](https://docs.llamaindex.ai/)
- [ADK Blogger Sample](https://github.com/google/adk-samples/tree/main/python/agents/blog-writer)
- [Ollama Documentation](https://ollama.ai/)
