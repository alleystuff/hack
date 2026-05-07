# Azure AI Foundry Agent Service — Deep Research

**Date:** 2026-05-07  
**Status:** Complete  
**Sources:** Microsoft Learn docs (Apr–May 2026), azure-sdk-for-python v2.1.0 README, official Python samples

---

## Table of Contents

1. [Azure AI Foundry Agent Service Overview](#1-azure-ai-foundry-agent-service-overview)
2. [Python SDK: `azure-ai-projects`](#2-python-sdk-azure-ai-projects)
3. [Agent Lifecycle: Conversations, Responses, and Approvals](#3-agent-lifecycle-conversations-responses-and-approvals)
4. [Tool System Overview](#4-tool-system-overview)
5. [MCP Integration — Native Support](#5-mcp-integration--native-support)
6. [Complete Python Code Snippets](#6-complete-python-code-snippets)
7. [Required Azure Resources](#7-required-azure-resources)
8. [Environment Variables](#8-environment-variables)
9. [Comparison: Foundry Agent SDK vs Direct Azure OpenAI + mcp Client](#9-comparison-foundry-agent-sdk-vs-direct-azure-openai--mcp-client)
10. [Blockers and Limitations for Hackathon](#10-blockers-and-limitations-for-hackathon)
11. [Recommendation](#11-recommendation)

---

## 1. Azure AI Foundry Agent Service Overview

### What is Microsoft Foundry Agent Service?

**Microsoft Foundry Agent Service** (formerly Azure AI Agent Service / Azure AI Projects) is a fully managed, enterprise-grade platform for building, deploying, and scaling AI agents. As of May 2026, it is generally available (GA) with some preview features.

An agent combines:
- **Model** — a model from the Foundry model catalog (GPT-4o, GPT-5-mini, Llama, DeepSeek, etc.)
- **Instructions** — system prompt defining agent behavior
- **Tools** — built-in or custom capabilities (web search, code interpreter, file search, MCP servers, function calling, OpenAPI, etc.)

The service handles hosting, scaling, identity, observability, and enterprise security.

### How it Differs from Raw Azure OpenAI

| Aspect | Azure OpenAI (raw) | Azure AI Foundry Agent Service |
|---|---|---|
| API abstraction | Low-level: manage prompts, history, tool loop yourself | High-level: agent definitions, versioning, managed execution |
| Tool orchestration | Manual: call tools, inject results, loop | Managed: service calls tools, handles multi-step reasoning |
| MCP support | None natively; need `mcp` Python client as bridge | **Native**: `MCPTool` class, approval flow built-in |
| State management | Stateless completions; you manage history | Conversations managed server-side |
| Model deployment | Use `azure_deployment` directly | Use deployment name via `FOUNDRY_MODEL_NAME` |
| Authentication | API key or Entra ID via `openai` client | Entra ID (Foundry-managed) or API key via `AIProjectClient` |
| Observability | Manual | Built-in: tracing, Application Insights, dashboards |
| Versioning | None | Agent versions with rollback |
| Cost | Model tokens only | Model tokens + agent service (token pricing same underlying OpenAI models) |

### Agent Types (May 2026)

| Type | Description | Code Required |
|---|---|---|
| **Prompt agents** | Defined via instructions + tools. Fully managed execution. | No (portal) or minimal SDK code |
| **Workflow agents** (preview) | Multi-agent orchestration, visual YAML builder | Optional YAML |
| **Hosted agents** (preview) | Container-based; bring your own framework (LangGraph, etc.) | Yes |

**For a hackathon demo, Prompt agents are the right choice** — fastest to build, supports all tools including MCP.

### Naming History

- **Azure AI Projects** (classic, v1.x) → deprecated March 31, 2027
- **Microsoft Foundry Agent Service** (new, v2.x) → GA as of 2026
- The Python package name stays `azure-ai-projects` but v2.0.0+ targets the new API

---

## 2. Python SDK: `azure-ai-projects`

### Installation

```bash
pip install "azure-ai-projects>=2.0.0"
pip install azure-identity  # for DefaultAzureCredential
pip install python-dotenv   # for .env file support
```

> **Critical:** Azure AI Projects 2.x is **incompatible** with 1.x. The old `create_thread` / `create_run` / `add_message` API no longer applies. See the [migration guide](https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/migrate).

Verify version:
```bash
pip show azure-ai-projects
# Should show: Version: 2.1.0 (or higher)
```

### Key Classes (v2.x)

| Class / Method | Purpose |
|---|---|
| `AIProjectClient` | Main entry point; requires `endpoint` + `credential` |
| `AIProjectClient.agents` | Create, version, delete agents |
| `AIProjectClient.get_openai_client()` | Returns an OpenAI client pre-authenticated to your Foundry project |
| `PromptAgentDefinition` | Agent definition: model, instructions, tools |
| `MCPTool` | Attach an MCP server as a tool |
| `WebSearchTool` | Built-in web search |
| `FileSearchTool` | Built-in file/vector search |
| `CodeInterpreterTool` | Built-in sandboxed Python execution |
| `FunctionTool` | Custom function calling |
| `OpenApiTool` | Connect via OpenAPI spec |
| `openai_client.conversations` | Create/manage conversation threads |
| `openai_client.responses` | Send messages, get responses (with tool calls handled server-side) |

### Authentication

Entra ID (recommended):
```python
from azure.identity import DefaultAzureCredential
from azure.ai.projects import AIProjectClient

project = AIProjectClient(
    endpoint=os.environ["FOUNDRY_PROJECT_ENDPOINT"],
    credential=DefaultAzureCredential(),
)
```

API key alternative (simpler for hackathon):
```python
from azure.core.credentials import AzureKeyCredential

project = AIProjectClient(
    endpoint=os.environ["FOUNDRY_PROJECT_ENDPOINT"],
    credential=AzureKeyCredential(os.environ["FOUNDRY_API_KEY"]),
)
```

**Endpoint format:**
```
https://<your-ai-services-account-name>.services.ai.azure.com/api/projects/<your-project-name>
```
Find it in: Foundry portal → your project → Overview page → "Project endpoint".

---

## 3. Agent Lifecycle: Conversations, Responses, and Approvals

The new v2.x API does **not** use the classic `threads / runs / messages` pattern from OpenAI Assistants. Instead it uses the **Responses API** combined with **Conversations**.

### Concept Map

```
                        ┌────────────────────────────────────────┐
                        │       Foundry Agent Service            │
                        │                                        │
  User message ──►  responses.create()  ──►  Agent (model+tools) │
                        │                         │              │
                        │   [tool calls managed    │              │
                        │    server-side for       │              │
                        │    built-in tools]       │              │
                        │                         ▼              │
                        │              MCP approval_request?     │
                        │                 ↓ (if require_approval) │
  App handles approval ◄──── response.output items ─────────────┘
  App sends approval ──► responses.create(input=approval_responses,
                                          previous_response_id=...)
```

### Conversation (thread)

A conversation maintains message history across multiple turns:
```python
openai = project.get_openai_client()
conversation = openai.conversations.create()
# conversation.id is passed on every turn
```

### Sending a message and getting a response

```python
response = openai.responses.create(
    conversation=conversation.id,
    input="Your question here",
    extra_body={"agent_reference": {"name": agent_name, "type": "agent_reference"}},
)
print(response.output_text)
```

### MCP approval loop

When `require_approval="always"` is set on `MCPTool`, the response contains `mcp_approval_request` items that must be approved before execution continues:

```python
# First call: may return approval requests
response = openai.responses.create(...)

# Check for pending approvals
input_list = []
for item in response.output:
    if item.type == "mcp_approval_request" and item.id:
        input_list.append(McpApprovalResponse(
            type="mcp_approval_response",
            approve=True,   # or False to reject
            approval_request_id=item.id,
        ))

# Second call: submit approvals
if input_list:
    response = openai.responses.create(
        input=input_list,
        previous_response_id=response.id,
        extra_body={"agent_reference": {"name": agent_name, "type": "agent_reference"}},
    )
print(response.output_text)
```

---

## 4. Tool System Overview

### Built-in Tools

These are preconfigured; service handles execution:

| Tool | Class | Notes |
|---|---|---|
| Web Search | `WebSearchTool()` | Real-time web with citations |
| Code Interpreter | `CodeInterpreterTool()` | Sandboxed Python |
| File Search | `FileSearchTool(vector_store_ids=[...])` | Vector search over uploaded files |
| Azure AI Search | — | Requires connection |
| Function Calling | `FunctionTool(functions=[...])` | App executes, returns result |

### Custom / External Tools

| Tool | Class | Notes |
|---|---|---|
| MCP | `MCPTool(server_url=..., server_label=...)` | Remote MCP endpoint required |
| OpenAPI | `OpenApiTool(openapi=...)` | OpenAPI 3.0/3.1 spec |
| Agent-to-Agent | — | A2A protocol (preview) |

### Tool Invocation Control

```python
# Force a specific tool
run = project_client.agents.runs.create_and_process(
    thread_id=thread.id,
    agent_id=agent.id,
    tool_choice={"type": "bing_grounding"},
)
```

Use agent `instructions` to guide tool selection semantically.

---

## 5. MCP Integration — Native Support

### Key Finding

**Azure AI Foundry Agent Service has native MCP support.** You do **not** need the `mcp` Python library as a bridge. The `MCPTool` class in `azure-ai-projects>=2.0.0` connects directly to remote MCP servers.

### Architecture: How Foundry Connects to MCP

```
Your App
   │
   ▼
AIProjectClient ─► Foundry Agent Service (cloud-managed)
                        │
                        ▼
                   MCPTool config:
                   - server_url: "https://your-mcp-server.com/mcp"
                   - server_label: "my-tool"
                   - require_approval: "always" | "never"
                   - project_connection_id: (for auth)
                        │
                        ▼ (Foundry calls your MCP server)
                   Your Remote MCP Server
                   (must be internet-accessible or on private VNet)
```

### Critical Constraint: Remote Endpoint Required

**The Foundry Agent Service runtime only accepts remote MCP server endpoints.** If you have a local MCP server (e.g., a subprocess), you must host it publicly first.

Options for exposing local MCP servers:
1. **Azure Container Apps** — Preferred; supports any language, UVX/NPX
2. **Azure Functions** (MCP webhook endpoint `/runtime/webhooks/mcp`) — Key-auth only; Python, Node.js, Java, .NET
3. **ngrok or similar tunneling** — For hackathon demos only (not production)

### MCPTool Parameters

```python
from azure.ai.projects.models import MCPTool

tool = MCPTool(
    server_label="my-server",          # unique label per agent (required)
    server_url="https://...",           # remote MCP endpoint URL (required)
    require_approval="always",          # "always" | "never" | {"never": [...]} | {"always": [...]}
    allowed_tools=["tool1", "tool2"],   # optional allowlist; defaults to all
    project_connection_id="conn-name",  # connection storing auth credentials (optional)
)
```

### Authentication Options for MCP

| Method | How to Set Up |
|---|---|
| **No auth** | Omit `project_connection_id`; server must be public |
| **API key / Bearer token** | Create project connection with custom key `Authorization: Bearer <token>` |
| **Microsoft Entra (managed identity)** | Use agent's or project's managed identity; no secrets |
| **OAuth passthrough (OBO)** | Per-user auth; Foundry generates consent link |

To create a connection (key-based):
1. Foundry portal → your project → Settings → Connections
2. Create connection type: Custom keys
3. Key = `Authorization`, Value = `Bearer <your-api-key>`
4. Copy the resource ID as `project_connection_id`

### Structured Inputs for MCP (dynamic server at runtime)

MCP server URL, label, and headers can be overridden per-request via structured inputs:

```json
{
  "tools": [{"type": "mcp", "server_label": "{{mcp_label}}", "server_url": "{{mcp_url}}"}],
  "structured_inputs": {
    "mcp_label": {"description": "MCP server label", "required": true, "schema": {"type": "string"}},
    "mcp_url":   {"description": "MCP server URL",   "required": true, "schema": {"type": "string"}}
  }
}
```

---

## 6. Complete Python Code Snippets

### Snippet 1: Minimal Agent with MCP Tool (unauthenticated public MCP server)

```python
"""
Minimal Foundry Agent with an MCP tool.
Requires:
  pip install "azure-ai-projects>=2.0.0" azure-identity python-dotenv
"""
import os
from dotenv import load_dotenv
from azure.identity import DefaultAzureCredential
from azure.ai.projects import AIProjectClient
from azure.ai.projects.models import PromptAgentDefinition, MCPTool
from openai.types.responses.response_input_param import McpApprovalResponse, ResponseInputParam

load_dotenv()

PROJECT_ENDPOINT = os.environ["FOUNDRY_PROJECT_ENDPOINT"]
MODEL_NAME = os.environ["FOUNDRY_MODEL_NAME"]        # e.g., "gpt-4o" or "gpt-5-mini"
MCP_SERVER_URL = os.environ["MCP_SERVER_URL"]         # e.g., "https://your-mcp-server.azurecontainerapps.io/mcp"
AGENT_NAME = "hackathon-agent"

with (
    DefaultAzureCredential() as credential,
    AIProjectClient(endpoint=PROJECT_ENDPOINT, credential=credential) as project,
    project.get_openai_client() as openai,
):
    # 1. Define the MCP tool
    mcp_tool = MCPTool(
        server_label="my-mcp-server",
        server_url=MCP_SERVER_URL,
        require_approval="never",   # auto-approve for demos; use "always" in prod
    )

    # 2. Create (or update) the agent version
    agent = project.agents.create_version(
        agent_name=AGENT_NAME,
        definition=PromptAgentDefinition(
            model=MODEL_NAME,
            instructions=(
                "You are a helpful assistant. Use the available MCP tools "
                "to answer questions accurately."
            ),
            tools=[mcp_tool],
        ),
    )
    print(f"Agent created: id={agent.id}, version={agent.version}")

    # 3. Start a conversation (thread)
    conversation = openai.conversations.create()
    print(f"Conversation id: {conversation.id}")

    # 4. Send a message — built-in tools are invoked server-side automatically
    response = openai.responses.create(
        conversation=conversation.id,
        input="What can you help me with today?",
        extra_body={"agent_reference": {"name": AGENT_NAME, "type": "agent_reference"}},
    )
    print(f"Response: {response.output_text}")

    # 5. Multi-turn: follow-up in same conversation
    response = openai.responses.create(
        conversation=conversation.id,
        input="Great, tell me more about X.",
        extra_body={"agent_reference": {"name": AGENT_NAME, "type": "agent_reference"}},
    )
    print(f"Response: {response.output_text}")

    # 6. Cleanup
    project.agents.delete_version(agent_name=agent.name, agent_version=agent.version)
    print("Agent version deleted.")
```

### Snippet 2: Agent with Two MCP Tools + Approval Flow

```python
"""
Agent with two MCP servers (e.g., data server + action server) and approval loop.
"""
import os, json
from dotenv import load_dotenv
from azure.identity import DefaultAzureCredential
from azure.ai.projects import AIProjectClient
from azure.ai.projects.models import PromptAgentDefinition, MCPTool
from openai.types.responses.response_input_param import McpApprovalResponse, ResponseInputParam

load_dotenv()

PROJECT_ENDPOINT = os.environ["FOUNDRY_PROJECT_ENDPOINT"]
MODEL_NAME = os.environ["FOUNDRY_MODEL_NAME"]

def run_agent_turn(openai, agent_name, conversation_id, user_input, auto_approve=False):
    """Send a message and handle MCP approval loop."""
    response = openai.responses.create(
        conversation=conversation_id,
        input=user_input,
        extra_body={"agent_reference": {"name": agent_name, "type": "agent_reference"}},
    )

    # Handle MCP approval requests
    while True:
        approval_inputs: ResponseInputParam = []
        for item in response.output:
            if item.type == "mcp_approval_request" and item.id:
                tool_name = getattr(item, "name", "<unknown>")
                args = getattr(item, "arguments", None)
                print(f"MCP approval requested: server={item.server_label}, tool={tool_name}")
                print(f"  Arguments: {json.dumps(args, indent=2, default=str)}")

                approve = True if auto_approve else (
                    input("  Approve? (y/N): ").strip().lower() == "y"
                )
                approval_inputs.append(McpApprovalResponse(
                    type="mcp_approval_response",
                    approve=approve,
                    approval_request_id=item.id,
                ))

        if not approval_inputs:
            break   # No more approvals needed

        response = openai.responses.create(
            input=approval_inputs,
            previous_response_id=response.id,
            extra_body={"agent_reference": {"name": agent_name, "type": "agent_reference"}},
        )

    return response.output_text


with (
    DefaultAzureCredential() as credential,
    AIProjectClient(endpoint=PROJECT_ENDPOINT, credential=credential) as project,
    project.get_openai_client() as openai,
):
    # Two MCP tools: data tool (read-only, auto-approve) and action tool (requires approval)
    data_tool = MCPTool(
        server_label="data-server",
        server_url=os.environ["DATA_MCP_SERVER_URL"],
        require_approval={"never": ["list_items", "get_item", "search"]},  # safe reads auto-approved
    )
    action_tool = MCPTool(
        server_label="action-server",
        server_url=os.environ["ACTION_MCP_SERVER_URL"],
        project_connection_id=os.environ.get("ACTION_MCP_CONNECTION_ID"),
        require_approval="always",   # all writes require explicit approval
    )

    agent = project.agents.create_version(
        agent_name="dual-mcp-agent",
        definition=PromptAgentDefinition(
            model=MODEL_NAME,
            instructions=(
                "You have access to two tool servers: "
                "'data-server' for reading data and 'action-server' for taking actions. "
                "Always read before you act. Explain what you plan to do before invoking "
                "action-server tools."
            ),
            tools=[data_tool, action_tool],
        ),
    )
    print(f"Agent: {agent.name} v{agent.version}")

    conversation = openai.conversations.create()

    # Chat loop
    while True:
        user_input = input("\nYou: ").strip()
        if user_input.lower() in ("exit", "quit"):
            break
        answer = run_agent_turn(openai, agent.name, conversation.id, user_input, auto_approve=False)
        print(f"\nAgent: {answer}")

    project.agents.delete_version(agent_name=agent.name, agent_version=agent.version)
```

### Snippet 3: Agent with FileSearch + CodeInterpreter (no MCP)

```python
"""
Classic Foundry agent with built-in tools only (no MCP).
"""
import os
from azure.identity import DefaultAzureCredential
from azure.ai.projects import AIProjectClient
from azure.ai.projects.models import PromptAgentDefinition, CodeInterpreterTool, FileSearchTool

PROJECT_ENDPOINT = os.environ["FOUNDRY_PROJECT_ENDPOINT"]
MODEL_NAME = os.environ["FOUNDRY_MODEL_NAME"]
VECTOR_STORE_ID = os.environ.get("VECTOR_STORE_ID", "")

with (
    DefaultAzureCredential() as credential,
    AIProjectClient(endpoint=PROJECT_ENDPOINT, credential=credential) as project,
    project.get_openai_client() as openai,
):
    tools = [CodeInterpreterTool()]
    if VECTOR_STORE_ID:
        tools.append(FileSearchTool(vector_store_ids=[VECTOR_STORE_ID]))

    agent = project.agents.create_version(
        agent_name="analysis-agent",
        definition=PromptAgentDefinition(
            model=MODEL_NAME,
            instructions="You are a data analyst. Use code interpreter to run calculations.",
            tools=tools,
        ),
    )

    conversation = openai.conversations.create()

    response = openai.responses.create(
        conversation=conversation.id,
        input="Calculate the mean of [1, 5, 3, 8, 2] using Python.",
        extra_body={"agent_reference": {"name": agent.name, "type": "agent_reference"}},
    )
    print(response.output_text)

    project.agents.delete_version(agent_name=agent.name, agent_version=agent.version)
```

---

## 7. Required Azure Resources

### Minimum Setup for Hackathon

| Resource | Purpose | Notes |
|---|---|---|
| **Azure AI Foundry Account** (Azure AI Services) | Root container for projects | Created automatically via Foundry portal |
| **Foundry Project** | Workspace; holds agents, connections, deployments | Create via portal: `ai.azure.com` |
| **Model Deployment** | e.g., `gpt-4o` or `gpt-5-mini` | Deploy via Foundry portal → Models + Endpoints |
| **MCP Server (remote)** | Your MCP server hosted publicly | Azure Container Apps, Azure Functions, or ngrok tunnel |

### Optional (for auth to MCP servers)

| Resource | Purpose |
|---|---|
| **Project Connection** (Custom keys) | Store API keys / Bearer tokens for MCP server auth |
| **Managed Identity** | For production Entra-based MCP auth |

### Setup Path (fastest for hackathon)

1. Go to [ai.azure.com](https://ai.azure.com)
2. Click "Create an agent" → this provisions: Foundry Account + Project + deploys `gpt-4o` automatically
3. In the project, go to Models + Endpoints → note your deployment name
4. Copy the Project Endpoint URL from the Overview page
5. Run `az login` locally for DefaultAzureCredential to work

---

## 8. Environment Variables

```bash
# .env file for hackathon

# Foundry Project endpoint (from Foundry portal → Project Overview)
FOUNDRY_PROJECT_ENDPOINT=https://<account-name>.services.ai.azure.com/api/projects/<project-name>

# Model deployment name (from Foundry portal → Models + Endpoints → Name column)
FOUNDRY_MODEL_NAME=gpt-4o

# Optional: API key authentication alternative to az login
# FOUNDRY_API_KEY=<project-api-key-from-portal>

# MCP server URLs (your deployed MCP servers)
MCP_SERVER_URL=https://<your-mcp-server>.azurecontainerapps.io/mcp
DATA_MCP_SERVER_URL=https://<data-mcp-server>.azurecontainerapps.io/mcp
ACTION_MCP_SERVER_URL=https://<action-mcp-server>.azurecontainerapps.io/mcp

# Optional: Connection resource IDs for authenticated MCP servers
# MCP_PROJECT_CONNECTION_ID=<connection-resource-id-from-portal>
# ACTION_MCP_CONNECTION_ID=<connection-resource-id-from-portal>

# Optional: Tracing
# AZURE_EXPERIMENTAL_ENABLE_GENAI_TRACING=true
# OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT=true
```

### How to Reference GPT-4o

In Azure AI Foundry, you reference models by their **deployment name**, not the canonical model name. When you deploy `gpt-4o` from the model catalog, you assign it a name (e.g., `"gpt-4o"` or `"my-gpt4o"`). This deployment name is what goes in `FOUNDRY_MODEL_NAME` and into `PromptAgentDefinition(model=...)`.

This differs from raw Azure OpenAI where you use `azure_deployment=` parameter. In Foundry, the SDK abstracts this through the project endpoint + deployment name.

---

## 9. Comparison: Foundry Agent SDK vs Direct Azure OpenAI + mcp Python Client

### Option A: Azure AI Foundry Agent SDK (azure-ai-projects v2.x)

**How it works:** `AIProjectClient` → `MCPTool` → Foundry manages calling your MCP server → agent responds

**Pros:**
- Native MCP support — no conversion layer needed
- Foundry manages the multi-turn loop, tool dispatch, and retries
- Built-in approval flow for MCP tool calls
- Conversation (thread) state managed server-side
- Enterprise features: agent versioning, rollback, tracing, monitoring
- Can combine MCP + built-in tools (file search, code interpreter, etc.) seamlessly
- Toolbox feature: bundle multiple MCP servers + built-in tools into one endpoint

**Cons:**
- MCP server must be hosted remotely (cannot use local subprocess directly)
- Approval-based flow adds a round-trip for `require_approval="always"` tools
- Less control over the exact agentic loop
- Requires Azure subscription + Foundry project setup (5–10 minutes)
- v2.x API is relatively new; some docs still reference the deprecated v1.x

### Option B: Direct Azure OpenAI + `mcp` Python Client (manual bridge)

**How it works:** `openai.chat.completions.create()` → detect tool calls → call `mcp.Client` locally → inject results → loop

**Pros:**
- Full control over every step of the agent loop
- MCP servers can run as local subprocesses (no hosting required)
- Simpler dependency tree if you already use `openai` and `mcp` packages
- Works with any OpenAI-compatible endpoint (not just Foundry)
- Easier to debug; you see every message

**Cons:**
- Must manually convert MCP tool schemas to OpenAI function-calling format
- Must implement the tool-call loop yourself (fetch completions → check `finish_reason` → execute tools → re-send)
- No built-in approval flow
- No server-side conversation state
- No agent versioning or enterprise features
- More code to write and maintain

### Hackathon Decision Matrix

| Criterion | Foundry Agent SDK | Direct AzureOAI + mcp |
|---|---|---|
| Setup time | Medium (need Azure project + hosted MCP) | Low (local servers, minimal Azure setup) |
| MCP integration | Native, clean API | Manual conversion + loop |
| Control over loop | Low (managed) | High |
| Demo polish | High (enterprise-grade) | Medium |
| 2 MCP servers | `tools=[mcp1, mcp2]` — trivial | Manual bridge for each |
| Local MCP dev | Needs hosting (ngrok ok for demo) | Runs locally without hosting |
| Lines of code | ~50–80 lines | ~120–200 lines |
| Cost | Same model token cost | Same model token cost |
| Latency | Slightly higher (managed service overhead) | Lower (direct API calls) |

---

## 10. Blockers and Limitations for Hackathon

### Critical Blockers

1. **Remote MCP server required.** If your MCP servers run as local stdio subprocesses, Foundry cannot reach them directly. You must either:
   - Deploy to Azure Container Apps (a few minutes with Docker)
   - Use ngrok to tunnel localhost to a public HTTPS URL (quick hack for demos)
   - Use Azure Functions MCP webhook endpoint

2. **API version mismatch (v1.x vs v2.x).** Many tutorials and Stack Overflow answers still reference the old `create_thread` / `create_run` API (v1.x). Make sure to use `azure-ai-projects>=2.0.0`. The new API uses `openai.responses.create()` with `extra_body={"agent_reference": ...}`.

3. **MCP timeout: 100 seconds.** Non-streaming MCP tool calls time out after 100 seconds. Ensure your MCP server responds within this limit.

### Potential Issues

4. **Approval round-trip latency.** With `require_approval="always"`, every MCP tool call requires a second API round-trip. For a hackathon demo with live audience, consider `require_approval="never"` or selective approval (`{"never": ["safe_tool_name"]}`).

5. **"Invalid tool schema" errors.** Foundry rejects MCP tool definitions that use `anyOf`, `allOf`, or parameters accepting multiple types. Fix your MCP server's tool definitions to use simple, concrete schemas.

6. **Private MCP requires Standard Agent Setup.** Public endpoints (via ngrok or Container Apps public ingress) work with Basic Agent Setup. Private VNet deployment is more complex.

7. **Model deployment name.** GPT-4o must be deployed in your Foundry project. The name you give the deployment (not "gpt-4o" as a global identifier) is what goes in `FOUNDRY_MODEL_NAME`. Check "Models + Endpoints" in the Foundry portal.

8. **RBAC permissions.** The user creating agents needs the **Azure AI User** role at the project level. The user creating the project needs **Azure AI Account Owner** or Contributor/Owner at subscription scope.

9. **Region availability.** Not all tools are available in all regions. Check [tool support by region and model](https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/tool-best-practice#tool-support-by-region-and-model).

---

## 11. Recommendation

### For a hackathon demo with 2 MCP servers

**Use the Azure AI Foundry Agent SDK (azure-ai-projects v2.x).**

**Rationale:**

- The MCP integration is first-class: `tools=[MCPTool(...), MCPTool(...)]` — two lines of code to attach both MCP servers. No conversion, no manual loop.
- Demo quality is higher: enterprise branding, built-in tracing in Foundry portal, agent versioning.
- Less code to write and debug under time pressure.
- The approval flow adds demo interactivity (judges can see tool calls being approved in real time).

**MCP server hosting for hackathon:** Use ngrok or Cloudflare Tunnel to expose local MCP servers as public HTTPS endpoints. This avoids needing to deploy to Azure Container Apps under time pressure.

```bash
# Quick tunnel for demo (not for production)
ngrok http 8000   # exposes localhost:8000 as https://<random>.ngrok.io
# Then: MCP_SERVER_URL=https://<random>.ngrok.io/mcp
```

**If you need maximum control or local-only MCP servers:** Use Direct Azure OpenAI + `mcp` Python client. But expect to write the tool-call loop and schema conversion yourself.

### Suggested Tech Stack for Hackathon

```
azure-ai-projects>=2.0.0   # Foundry Agent SDK
azure-identity             # DefaultAzureCredential
python-dotenv              # .env loading
openai                     # Bundled dependency (responses API types)

# Optional for local MCP server hosting:
fastapi                    # HTTP server for MCP
uvicorn                    # ASGI server
mcp                        # MCP server framework (for building your MCP server)
ngrok / cloudflare tunnel  # For exposing local MCP to Foundry
```

---

## References

- [Microsoft Foundry Agent Service Overview](https://learn.microsoft.com/en-us/azure/foundry/agents/overview) — Apr 28, 2026
- [Agent Tools Catalog](https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/tool-catalog) — Apr 23, 2026
- [Connect Agents to MCP Servers](https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/tools/model-context-protocol) — Apr 23, 2026
- [azure-ai-projects Python SDK README v2.1.0](https://learn.microsoft.com/en-us/python/api/overview/azure/ai-projects-readme) — Apr 21, 2026
- [Microsoft Foundry Quickstart (code)](https://learn.microsoft.com/en-us/azure/foundry/quickstarts/get-started-code) — Mar 24, 2026
- [GitHub: azure-ai-projects samples/agents/tools](https://github.com/Azure/azure-sdk-for-python/tree/main/sdk/ai/azure-ai-projects/samples/agents/tools) — v2.1.0
- [sample_agent_mcp.py](https://raw.githubusercontent.com/Azure/azure-sdk-for-python/main/sdk/ai/azure-ai-projects/samples/agents/tools/sample_agent_mcp.py) — official MCP sample
- [sample_agent_mcp_with_project_connection.py](https://raw.githubusercontent.com/Azure/azure-sdk-for-python/main/sdk/ai/azure-ai-projects/samples/agents/tools/sample_agent_mcp_with_project_connection.py) — MCP with auth
