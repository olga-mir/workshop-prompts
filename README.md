
# ADK Workshop

In this workshop we use Blogger Agent from `adk-samples` repository and make it run locally with Ollama models and RAG.

ADK Samples repo: `git@github.com:google/adk-samples.git` needs to be cloned locally
Prompts repo: `todo` this is auxillary repo that contains scripts and prompts to follow along the steps. The files are either scripts (ending in `.sh`) or prompts (ending in `.md`) - you can either follow along the instructions or let your AI assistant execute on that prompt.

## Steps

1. **Copy sample ADK agent**

Copy the folder to a fresh folder (`prompts/step1-copy-sample-agent-files.sh`)

```
cp -r ${PATH_FROM}/adk-samples/python/agents/blog-writer  ${PATH_TO}/workshop-adk-ollama
```

2. **Replace project files**

Step description is in `step2-prompt-replace-project-files.md`.
