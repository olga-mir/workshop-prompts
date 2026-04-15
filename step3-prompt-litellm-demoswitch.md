# Integration Prompt: Enable Ollama + Gemini Support in ADK Agent

Your target: Modify the adk-samples code to support both local Ollama and Vertex/Gemini models using a clean, maintainable approach.

## Prerequisites (Already Completed)
- You have `.env.ollama` and `.env.gemini` files in `blog-writer/blogger_agent/` folder
- Both files are in the correct location and structure is correct

## Implementation Steps

### 1. Create/Update `config.py` with DEMO_MODE Switching

Add a demo mode switch at the top of your config file:

```python
import os
from dotenv import load_dotenv

# Set this to "local" or "vertex" - controls which .env file loads
DEMO_MODE = os.getenv("DEMO_MODE", "local")

# Load the appropriate .env file based on DEMO_MODE
if DEMO_MODE == "local":
    env_file = os.path.join(os.path.dirname(__file__), ".env.ollama")
    load_dotenv(dotenv_path=env_file)
    print("🚀 Running in LOCAL mode with Ollama")
else:
    env_file = os.path.join(os.path.dirname(__file__), ".env.gemini")
    load_dotenv(dotenv_path=env_file)
    print("☁️ Running in VERTEX mode with Gemini")
```

### 2. Add Model Wrapper Function with LiteLLM

Create a function that wraps models appropriately based on the mode:

```python
def get_model_wrapper(model_name: str):
    """
    Returns the appropriate model wrapper based on DEMO_MODE.
    - For local mode: wraps in LiteLlm for Ollama compatibility
    - For vertex mode: returns the model name directly for Gemini

    Args:
        model_name (str): The model identifier from environment

    Returns:
        LiteLlm wrapped model for local mode, or model name string for vertex mode
    """
    if DEMO_MODE == "local":
        from google.adk.models.lite_llm import LiteLlm
        return LiteLlm(model_name)
    else:
        # For Vertex/Gemini, return the model name directly
        return model_name
```

### 3. Update Agent Definitions

When initializing your agent with a model, use the wrapper:

```python
from .config import config, get_model_wrapper

your_agent = Agent(
    name="your_agent_name",
    model=get_model_wrapper(config.worker_model),  # ← Use wrapper instead of direct model name
    # ... rest of agent configuration
)
```

### 4. Fix Search Tool (Replace Problematic Tool)

Wrap the search tool with explicit type hints to avoid type-hint resolution issues:

```python
from langchain_community.tools import DuckDuckGoSearchRun

def search(query: str) -> str:
    """Search the web using DuckDuckGo.

    This wrapper exists because DuckDuckGoSearchRun has type hints that
    reference 'uuid' which causes issues with Google ADK's FunctionTool
    when analyzing type hints.
    """
    search_tool = DuckDuckGoSearchRun()
    return search_tool.run(query)
```

Use this wrapped function in your FunctionTools instead of the raw DuckDuckGo tool.

### 5. Update Your Credentials & Environment Configuration

Ensure your config.py handles GCP credentials gracefully:

```python
import google.auth

try:
    _, project_id = google.auth.default()
except Exception:
    project_id = os.environ.get("GOOGLE_CLOUD_PROJECT", "placeholder-project")

os.environ.setdefault("GOOGLE_CLOUD_PROJECT", project_id)
os.environ["GOOGLE_CLOUD_LOCATION"] = "global"
os.environ.setdefault("GOOGLE_GENAI_USE_VERTEXAI", "True")
```

## Environment Setup

### For Local Mode (.env.ollama):
```
DEMO_MODE=local
WORKER_MODEL=ollama/gemma4:e4b
CRITIC_MODEL=ollama/gemma4:e4b
EMBEDDING_MODEL=ollama/mxbai-embed-large:335m
OLLAMA_API_BASE=http://localhost:11434
```

### For Vertex Mode (.env.gemini):
```
DEMO_MODE=vertex
WORKER_MODEL=gemini-2.5-flash
CRITIC_MODEL=gemini-2.5-flash
EMBEDDING_MODEL=text-embedding-004
GOOGLE_CLOUD_PROJECT=your-project-id
GOOGLE_CLOUD_LOCATION=us-central1
```

## Switching Between Modes

**Local (Ollama):**
```bash
DEMO_MODE=local uv run adk web blog-writer
```

**Vertex (Gemini):**
```bash
DEMO_MODE=vertex uv run adk web blog-writer
```

## Key Principles

- **Single Source of Truth**: Use `get_model_wrapper()` for all model initialization
- **Tool Wrapping**: Wrap problematic tools with explicit type hints to avoid ADK parsing issues
- **Environment Isolation**: Load the correct .env file based on DEMO_MODE
- **Graceful Degradation**: Handle missing GCP credentials gracefully for local development

## Testing Checklist

- [ ] Local mode: Ollama running and models pulled locally
- [ ] Agent initializes with wrapped model successfully
- [ ] Tools can be called without type-hint resolution errors
- [ ] Vertex mode: GCP credentials configured
- [ ] Environment variables load correctly for each mode
