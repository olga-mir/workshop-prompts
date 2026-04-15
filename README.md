
# ADK Workshop

Make the ADK Blogger Agent run locally with Ollama models and RAG.

## Setup

Clone [`adk-samples`](https://github.com/google/adk-samples) locally to get started.

## Steps

1. **Copy sample ADK agent**

Run [`step1-copy-sample-agent-files.sh`](step1-copy-sample-agent-files.sh) to copy the blog-writer agent to a fresh folder.

2. **Replace project files**

Follow [`step2-prompt-replace-project-files.md`](step2-prompt-replace-project-files.md).

3. **Model switch**

Switch to local Ollama models via LiteLLM adapter in [`step3-prompt-litellm-demoswitch.md`](step3-prompt-litellm-demoswitch.md).

4. **Multi-agent handovers**

Enable multi-agent turns in [`step4-prompt-restore-multiagent-handovers.md`](step4-prompt-restore-multiagent-handovers.md).

## Demos

Demos are available in the `demo` folder in the workshop repository.
