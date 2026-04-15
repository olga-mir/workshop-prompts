
# ADK Workshop

Make the ADK Blogger Agent run locally with Ollama models and RAG.

## Goal

Start with an agent from Google provided open source collection of sample agents: [`adk-samples`](https://github.com/google/adk-samples)

Select an agent as a base for your own project, copy it to your local folder and follow the steps provided here in your local project.

Most of the steps are defined as prompts that can be fed into your coding assitant of choice (Claude Code, Gemini CLI, etc) or you can read them and follow along step by step yourself to gain deep understanding and actually learn the tech.

[`workshop-adk-ollama`](https://github.com/olga-mir/workshop-adk-ollama) is a reference repo that was created by following these steps. You can use it as a reference. It also contains demos running project locally and in cloud in GCP Vertex AI)

## Setup

Clone [`adk-samples`](https://github.com/google/adk-samples) locally to get started.

## Steps

1. **Copy sample ADK agent**

Create a clean empty folder for your project. We take the sample code and modify it to suit our needs.

Take copy command from [`step1-copy-sample-agent-files.md`](step1-copy-sample-agent-files.md) to copy the blog-writer agent to your working directory. Replace the placeholders with your paths.

2. **Replace project files**

Follow [`step2-prompt-replace-project-files.md`](step2-prompt-replace-project-files.md).

3. **Model switch**

Switch to local Ollama models via LiteLLM adapter in [`step3-prompt-litellm-demoswitch.md`](step3-prompt-litellm-demoswitch.md).

4. **Multi-agent handovers**

Enable multi-agent turns in [`step4-prompt-restore-multiagent-handovers.md`](step4-prompt-restore-multiagent-handovers.md).

5. **RAG**

Implement local RAG [`step5-prompt-local-rag-integration.md`](step5-prompt-local-rag-integration.md)

## Demos

Demos are available in the `demo` folder in the workshop repository - workshop-adk-ollama.
