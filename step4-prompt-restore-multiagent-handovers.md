# Step 4 Prompt: Restore Multi-Agent Handovers and Rich Interactions

**Objective:** Restore the rich multi-agent workflow that enables proper agent handovers and collaborative interactions, replacing the current one-shot behavior.

**Status:** Implementing against reference implementation in `/Users/olga/repos/agentic-rag-intro-workshop/`

---

## Problem Statement

The current implementation has degraded from a rich multi-agent system with proper handovers to a boring one-shot process, even with Gemini. The agents list other agents as tools but don't properly transfer work between them. This step restores the proper architecture.

---

## Prerequisites

- Step 3 implementation complete (DEMO_MODE switching)
- Reference implementation available at `/Users/olga/repos/agentic-rag-intro-workshop/`
- Current implementation at `/Users/olga/dev/workshop-clean-room/workshop-adk-ollama/`

---

## Implementation Steps

### 1. Add UUID Import for Type Hint Resolution

**File:** `blog-writer/blogger_agent/agent.py`

Add at the top of imports:
```python
import uuid  # noqa: F401 - Required for type hint resolution in Google ADK
```

**Why:** The uuid import (even though unused) ensures type hints resolve correctly when ADK analyzes function signatures. Without it, tools with uuid references may fail silently.

**Verification:** The import should appear before other imports but doesn't need to be used.

---

### 2. Implement Wrapped Search Function

**File:** `blog-writer/blogger_agent/tools.py`

Add this function (import DuckDuckGoSearchRun at the top):

```python
from langchain_community.tools import DuckDuckGoSearchRun

def search(query: str) -> str:
    """Search the web using DuckDuckGo.

    This wrapper function exists because DuckDuckGoSearchRun has type hints
    that reference 'uuid', but uuid is not imported in langchain_community's namespace.
    When Google ADK's FunctionTool tries to analyze type hints using typing.get_type_hints(),
    it fails with "NameError: name 'uuid' is not defined".

    By wrapping it with explicit type hints, we avoid this issue during tool declaration.
    """
    search_tool = DuckDuckGoSearchRun()
    return search_tool.run(query)
```

**Why:** This wrapping avoids the uuid type hint resolution issue that breaks when using DuckDuckGoSearchRun directly.

**Note:** Ensure uuid is imported at the top of tools.py:
```python
import uuid
```

---

### 3. Implement RAG Search Function (Optional but Recommended)

**File:** `blog-writer/blogger_agent/tools.py`

Add this function:

```python
def search_internal_data(query: str) -> str:
    """Search the internal technical knowledge base.
    
    This searches your knowledge base for relevant technical information.
    Use this to find internal articles, documentation, or technical references.
    
    Args:
        query: Search query to find relevant information
        
    Returns:
        Relevant information from the knowledge base, or message if nothing found
    """
    # Placeholder implementation - returns a message
    # In production, this would connect to a RAG system
    return f"Knowledge base search for '{query}' - feature available in production setup with RAG corpus"
```

**Why:** Gives the main agent a way to search internal knowledge before delegating to sub-agents.

---

### 4. Update Main Agent Instruction

**File:** `blog-writer/blogger_agent/agent.py`

Replace the entire `instruction` parameter with:

```python
instruction=f"""
You are a technical blogging assistant. Help users create technical blog posts.

IMPORTANT: You have TWO types of capabilities:

TOOLS (direct function calls - use these directly):
- analyze_codebase(directory): Analyze code structure from a directory
- search_internal_data(query): Search technical knowledge base
- search(query): Search the web for information
- save_blog_post_to_file(blog_post, filename): Save the final blog post

SUB-AGENTS (use transfer_to_agent() to delegate work):
- robust_blog_planner: Creates and refines blog outlines
- robust_blog_writer: Writes blog content
- blog_editor: Reviews and edits blog posts
- social_media_writer: Creates social media content

WORKFLOW:
1. Gather any needed information using tools (analyze_codebase, search_internal_data, search)
2. Create an outline by calling transfer_to_agent("robust_blog_planner")
3. Refine the outline with user feedback
4. Write the post by calling transfer_to_agent("robust_blog_writer")
5. Edit as needed by calling transfer_to_agent("blog_editor") with user feedback
6. Ask if user wants social content → transfer_to_agent("social_media_writer")
7. Save final post using save_blog_post_to_file(blog_post, filename)

DO NOT try to call any agent as a tool. DO NOT mention internal agent names.
ONLY use the tools and sub-agents listed above.

Current date: {datetime.datetime.now().strftime("%Y-%m-%d")}
"""
```

**Key Changes:**
- Explicitly lists "TWO types of capabilities"
- Uses directive language with `transfer_to_agent()`
- Shows numbered workflow with agent handovers
- Clear separation of tools vs sub-agents
- Includes timestamp for context

---

### 5. Update Main Agent Tools List

**File:** `blog-writer/blogger_agent/agent.py`

Replace the `tools` parameter:

```python
from .tools import analyze_codebase, save_blog_post_to_file, search_internal_data, search

# In the Agent definition:
tools=[
    FunctionTool(save_blog_post_to_file),
    FunctionTool(analyze_codebase),
    FunctionTool(search_internal_data),  # ← ADD THIS
    FunctionTool(search),                 # ← ADD THIS
],
```

**Why:** Gives the main agent tools to gather information before delegating to sub-agents.

---

### 6. Update blog_planner Sub-Agent

**File:** `blog-writer/blogger_agent/sub_agents/blog_planner.py`

Make these changes:

```python
import uuid  # noqa: F401 - Required for type hint resolution in Google ADK

from ..tools import search  # ← Change this import

blog_planner = Agent(
    model=get_model_wrapper(config.worker_model),
    name="blog_planner",
    description="Generates a blog post outline.",
    instruction="""
    You are a technical content strategist. Your job is to create a blog post outline.
    The outline should be well-structured and easy to follow.
    It should include a title, an introduction, a main body with several sections, and a conclusion.
    If a codebase is provided, the outline should include sections for code snippets and technical deep dives.
    The codebase context will be available in the `codebase_context` state key.
    Use the information in the `codebase_context` to generate a specific and accurate outline.
    Use the search tool to find relevant information and examples to support your writing.
    Your final output should be a blog post outline in Markdown format.
    """,
    tools=[search],  # ← Change from google_search to wrapped search
    output_key="blog_outline",
    after_agent_callback=suppress_output_callback,
)
```

**Changes:**
- Add uuid import
- Import wrapped `search` from tools instead of `google_search`
- Use `tools=[search]` instead of `tools=[google_search]`

---

### 7. Update blog_writer Sub-Agent

**File:** `blog-writer/blogger_agent/sub_agents/blog_writer.py`

Make these changes:

```python
import uuid  # noqa: F401 - Required for type hint resolution in Google ADK

from ..tools import search  # ← Change this import

blog_writer = Agent(
    model=get_model_wrapper(config.critic_model),
    name="blog_writer",
    description="Writes a technical blog post.",
    instruction="""
    You are an expert technical writer, crafting articles for a sophisticated audience similar to that of 'Towards Data Science' and 'freeCodeCamp'.
    Your task is to write a high-quality, in-depth technical blog post based on the provided outline and codebase summary.
    The article must be well-written, authoritative, and engaging for a technical audience.
    - Assume your readers are familiar with programming concepts and software development.
    - Dive deep into the technical details. Explain the 'how' and 'why' behind the code.
    - Use code snippets extensively to illustrate your points.
    - Use the search tool to find relevant information and examples to support your writing.
    - The codebase context will be available in the `codebase_context` state key.
    The final output must be a complete blog post in Markdown format. Do not wrap the output in a code block.
    """,
    tools=[search],  # ← Change from google_search to wrapped search
    output_key="blog_post",
    after_agent_callback=suppress_output_callback,
)
```

**Changes:**
- Add uuid import
- Import wrapped `search` from tools instead of `google_search`
- Use `tools=[search]` instead of `tools=[google_search]`

---

### 8. Verification Checklist

After implementing all changes:

- [ ] `uuid` import added to:
  - `blog-writer/blogger_agent/agent.py`
  - `blog-writer/blogger_agent/sub_agents/blog_planner.py`
  - `blog-writer/blogger_agent/sub_agents/blog_writer.py`
  - `blog-writer/blogger_agent/tools.py`

- [ ] `search()` function implemented in `tools.py` with:
  - Proper docstring explaining uuid type hint handling
  - DuckDuckGoSearchRun import
  - Correct function signature: `def search(query: str) -> str:`

- [ ] `search_internal_data()` function implemented in `tools.py`

- [ ] Main agent instruction in `agent.py` contains:
  - "TWO types of capabilities" section
  - Explicit `transfer_to_agent()` mentions
  - Numbered workflow with agent handovers
  - Clear distinction between tools and sub-agents

- [ ] Main agent tools list includes:
  - `search_internal_data`
  - `search`
  - `analyze_codebase`
  - `save_blog_post_to_file`

- [ ] Sub-agents (blog_planner, blog_writer) updated with:
  - uuid import
  - Import of wrapped `search` function
  - `tools=[search]` instead of `tools=[google_search]`

- [ ] Build succeeds: `DEMO_MODE=vertex uv run adk web blog-writer`

- [ ] Test interaction shows:
  - Agent mentions "Let me create an outline..." (calls blog_planner)
  - Agent mentions "Now let me write..." (calls blog_writer)  
  - Agent mentions "Let me refine..." (calls blog_editor)
  - Actual agent handovers occurring, not one-shot responses

---

## Testing Checklist

### Test with Gemini (Vertex Mode)
```bash
export DEMO_MODE=vertex
uv run adk web blog-writer
```

Expected behavior:
- [ ] Agent asks for topic
- [ ] Agent calls `robust_blog_planner` and shows outline
- [ ] Waits for user feedback
- [ ] Agent calls `robust_blog_writer` and writes blog post
- [ ] Multiple iterations as user provides feedback
- [ ] Agent offers social media generation

### Test with Ollama (Local Mode)
```bash
export DEMO_MODE=local
uv run adk web blog-writer
```

Expected behavior:
- [ ] Same multi-agent workflow as Gemini
- [ ] Should show proper agent handovers
- [ ] May be slower due to local model

---

## Key Principles

1. **Explicit Over Implicit:** Always mention `transfer_to_agent()` explicitly in instructions
2. **Two Capabilities:** Clearly separate tools from sub-agents
3. **Type Hint Resolution:** Always import uuid even if unused
4. **Search Integration:** Provide information gathering tools for agents
5. **Wrapped Tools:** Wrap problematic third-party tools to avoid type hint issues
6. **Numbered Workflows:** Show clear sequence of agent handovers

---

## Troubleshooting

### Problem: Agent still doesn't transfer to sub-agents
- **Check:** Does the instruction mention `transfer_to_agent()`? 
- **Check:** Are tools properly registered with FunctionTool?
- **Check:** Is uuid imported?

### Problem: Type hint errors with search tool
- **Check:** Is uuid imported in tools.py?
- **Check:** Is the search function properly wrapped?
- **Check:** Are sub-agents importing wrapped search, not google_search?

### Problem: Search tool not working
- **Check:** Is DuckDuckGoSearchRun installed? (`pip list | grep langchain`)
- **Check:** Is search function exported from tools.py?
- **Check:** Are FunctionTools properly wrapping it?

---

## Files to Modify

1. `blog-writer/blogger_agent/agent.py` - Main agent
2. `blog-writer/blogger_agent/tools.py` - Add search and search_internal_data
3. `blog-writer/blogger_agent/sub_agents/blog_planner.py` - Update search tool
4. `blog-writer/blogger_agent/sub_agents/blog_writer.py` - Update search tool

---

## Success Criteria

✅ Application builds and starts without errors
✅ Agent properly transfers to sub-agents (visible in conversation)
✅ Search tools work for both local and Vertex modes
✅ Multiple-turn interactions with agent coordination
✅ User can provide feedback and agents iterate
✅ Rich, engaging workflow instead of one-shot responses

---

## References

- Reference implementation: `/Users/olga/repos/agentic-rag-intro-workshop/`
- Detailed comparison: `AGENT_INSTRUCTION_COMPARISON.md`
- Previous steps: Step 1-3 implementations
