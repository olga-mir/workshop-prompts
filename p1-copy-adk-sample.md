
We are building an ADK agent based on the adk-sample repository. The first step is to take sample repo as is and copy it to working repo.

The user should provide path to `adk-sample` local repo and  where they want to copy. After your copy it will look like this:

```
blog-writer
├── adk_web.png
├── agent_explainer.md
├── blogger_agent
│   ├── __init__.py
│   ├── agent_utils.py
│   ├── agent.py
│   ├── config.py
│   ├── sub_agents
│   │   ├── __init__.py
│   │   ├── blog_editor.py
│   │   ├── blog_planner.py
│   │   ├── blog_writer.py
│   │   └── social_media_writer.py
│   ├── tools.py
│   └── validation_checkers.py
├── eval
│   ├── data
│   │   ├── blog_eval.test.json
│   │   └── test_config.json
│   ├── README.md
│   └── test_eval.py
├── README.md
├── requirements.txt
└── tests
    ├── README.md
    └── test_agent.py
```
