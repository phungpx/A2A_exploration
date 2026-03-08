# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Install dependencies
uv sync

# Run an individual agent server
uv run agents/a2a_policy_agent.py
uv run agents/a2a_provider_agent.py
uv run agents/a2a_research_agent.py
uv run agents/a2a_healthcare_agent.py

# Run the MCP server (usually launched automatically via stdio by ProviderAgent)
uv run agents/mcp_server.py
```

No test suite is configured. Python 3.13 is required.

## Architecture

This project implements a multi-agent healthcare assistant system using the **Agent-to-Agent (A2A) protocol** — a standard for inter-agent communication over HTTP/JSON-RPC.

### Agent topology

```
Healthcare Orchestrator (a2a_healthcare_agent.py)
├── PolicyAgent     (a2a_policy_agent.py)    — insurance PDF RAG
├── ProviderAgent   (a2a_provider_agent.py)  — doctor lookup via MCP
└── ResearchAgent   (a2a_research_agent.py)  — web search via Google ADK
```

Each specialist agent runs as a standalone HTTP server with an **AgentCard** (name, description, skills, URL) published at `GET /`. The orchestrator discovers them at startup by calling `check_agent_exists()` and wraps each one as a `HandoffTool`.

### Three different A2A integration patterns

| Agent | SDK used | Pattern |
|---|---|---|
| Policy, Provider | `a2a-sdk` | `AgentExecutor` subclass → `A2AStarletteApplication` → uvicorn |
| Research | `google-adk` | `LlmAgent` wrapped with `to_a2a()` utility |
| Orchestrator | `beeai-framework` | `RequirementAgent` registered with `A2AServer`; connects to remote agents via `A2AAgent` + `HandoffTool` |

### MCP server

`agents/mcp_server.py` exposes a `list_doctors` FastMCP tool over stdio transport. It reads from `data/doctors.json` and is launched as a subprocess by `ProviderAgent` via `langchain-mcp-adapters`.

### Authentication

`agents/helpers.py::authenticate()` uses a GCP service account key to obtain an impersonated credential (2-hour lifetime). It sets `GOOGLE_CLOUD_PROJECT` and optionally `GOOGLE_CLOUD_LOCATION`.

### Required data files (not in repo)

- `data/2026AnthemgHIPSBC.pdf` — insurance policy PDF loaded by `PolicyAgent`
- `data/doctors.json` — doctor records array loaded by `mcp_server.py`

### Required environment variables (`.env`)

| Variable | Used by |
|---|---|
| `GOOGLE_APPLICATION_CREDENTIALS` | `helpers.authenticate()` |
| `GOOGLE_VERTEX_BASE_URL` | `agents.py` (ProviderAgent), `a2a_healthcare_agent.py` |
| `DLAI_GOOGLE_IAM_ENDPOINT` | `helpers.authenticate()` (optional) |
| `AGENT_HOST` | all agents |
| `POLICY_AGENT_PORT` | policy agent + orchestrator |
| `RESEARCH_AGENT_PORT` | research agent + orchestrator |
| `PROVIDER_AGENT_PORT` | provider agent + orchestrator |
| `HEALTHCARE_AGENT_PORT` | orchestrator |

### Key import note

`a2a_policy_agent.py` and `a2a_provider_agent.py` import from `.agents` (relative package import), while `a2a_research_agent.py` imports `helpers` directly. Run them from the `agents/` directory or ensure `agents/` is on the Python path.
