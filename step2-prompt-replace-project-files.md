
Now we need to adapt the ADK sample agent that we copied to our needs. We will remove the python dependency management files from inside the folder and instead create our project dependencies at the repo root.

```
git rm -rf blog-writer/pyproject.toml
git rm -rf blog-writer/requirements.txt
git rm -rf blog-writer/uv.lock
```

Add following files from the workshop repo at the repo root.

```
.gitignore
pyproject.toml
```

Also add .env.example and .env.ollama in `blog-writer/blogger_agent/`

The content should be (for both):
```
# Model configuration for local development with Ollama
# Ensure Ollama is running and the model is pulled:
# ollama pull gemma4:e4b
# ollama pull mxbai-embed-large:335m
WORKER_MODEL=ollama/gemma4:e4b
CRITIC_MODEL=ollama/gemma4:e4b
EMBEDDING_MODEL=ollama/mxbai-embed-large:335m
```
