# Codelab Troubleshooting Guides

This repository contains troubleshooting guides for Google Cloud codelabs. Each guide helps participants diagnose and fix common issues encountered while following a codelab tutorial.

## Repository Structure

Each codelab has its own directory named after the codelab slug, containing a `TROUBLESHOOTING_GUIDE.md`:

```
codelab-troubleshoot-guide/
├── README.md
├── <codelab-slug>/
│   └── TROUBLESHOOTING_GUIDE.md
└── ...
```

## Available Guides

| Codelab | Troubleshooting Guide |
| ------- | --------------------- |
| [Building AI Agents with ADK: The Foundation](https://codelabs.developers.google.com/devsite/codelabs/build-agents-with-adk-foundation) | [Guide](build-agents-with-adk-foundation/TROUBLESHOOTING_GUIDE.md) |
| [Building AI Agents with ADK: Empowering with Tools](https://codelabs.developers.google.com/devsite/codelabs/build-agents-with-adk-empowering-with-tools) | [Guide](build-agents-with-adk-empowering-with-tools/TROUBLESHOOTING_GUIDE.md) |
| [Building Persistent AI Agents with ADK and CloudSQL](https://codelabs.developers.google.com/persistent-adk-cloudsql) | [Guide](persistent-adk-cloudsql/TROUBLESHOOTING_GUIDE.md) |
| [Deploy, Manage, and Observe ADK Agent on Cloud Run](https://codelabs.developers.google.com/deploy-manage-observe-adk-cloud-run) | [Guide](deploy-manage-observe-adk-cloud-run/TROUBLESHOOTING_GUIDE.md) |
| [Database as a Tool: Agentic RAG with ADK, MCP Toolbox, and Cloud SQL](https://codelabs.developers.google.com/agentic-rag-toolbox-cloudsql) | [Guide](agentic-rag-toolbox-cloudsql/TROUBLESHOOTING_GUIDE.md) |

## Adding a New Guide

1. Create a directory using the codelab slug as the name (lowercase kebab-case, e.g., `deploy-flask-cloudrun`)
2. Add a `TROUBLESHOOTING_GUIDE.md` inside that directory
3. Update the **Available Guides** table in this README
