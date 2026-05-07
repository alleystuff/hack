# Azure AI Agent Orchestration Research

**Date:** 2026-05-04  
**Status:** Complete  
**Topic:** Azure AI Agent pipeline using MCP servers (Office 365, Autodesk PLM, Teams)

---

## Research Questions

1. Azure AI Agent Service (now Microsoft Foundry Agent Service) — overview, MCP tool support, SDK, Python setup
2. Azure OpenAI Assistants API — differences from Foundry Agent Service
3. Semantic Kernel — MCP server integration
4. LangChain / LangGraph on Azure — MCP tool usage
5. Recommended Microsoft approach for MCP-based agent orchestration on Azure
6. Configuring an agent with multiple MCP servers (Office MCP + Teams MCP)
7. Authentication flow for OAuth-protected MCP servers
8. Required Azure services/resources
9. Official Microsoft sample repos
10. Agent system prompt for email-to-PLM workflow
11. Cost model — Foundry Agent Service vs direct Azure OpenAI Assistants

Additional topics: Microsoft AutoGen, Copilot Studio, Azure Logic Apps, Hackathon starters

---

## 1. Microsoft Foundry Agent Service (formerly Azure AI Agent Service)

### What it is

**Microsoft Foundry Agent Service** is the official, fully managed platform for building, deploying, and scaling AI agents on Azure. It is GA as of 2026. The previous branding "Azure AI Agent Service" redirects to this. It lives under **Azure AI Foundry** (portal at https://ai.azure.com).

An agent combines:
- **Model** — from the Foundry model catalog (GPT-4o, GPT-4.1, Llama, DeepSeek, Grok, etc.)
- **Instructions** — system prompt defining goals, behavior, constraints
- **Tools** — built-in (web search, code interpreter, file search, Azure AI Search, SharePoint) or custom (MCP, OpenAPI, Azure Functions, Agent-to-Agent)

### Three Agent Types

| Type | Code Required | Hosting | Orchestration |
|------|--------------|---------|---------------|
| Prompt agents | No | Fully managed | Single agent |
| Workflow agents (preview) | No (YAML optional) | Fully managed | Multi-agent, branching |
| Hosted agents (preview) | Yes | Container-based | Custom logic (LangGraph, AutoGen, custom) |

### SDK — `azure-ai-projects`

The primary SDK is **`azure-ai-projects`** (Python and C#/.NET). It wraps the Foundry Responses API.

```bash
pip install azure-ai-projects azure-identity openai
```

The SDK exposes `AIProjectClient` which provides:
- `project.agents.create_version()` — create/update agent definitions
- `project.get_openai_client()` — get an OpenAI client scoped to the project
- `openai.responses.create()` — invoke agent with a message
- `openai.conversations.create()` — create a persistent conversation thread

### Python Quickstart — Agent with Web Search

```python
from azure.identity import DefaultAzureCredential
from azure.ai.projects import AIProjectClient
from azure.ai.projects.models import PromptAgentDefinition, WebSearchTool

PROJECT_ENDPOINT = "https://<resource_name>.ai.azure.com/api/projects/<project_name>"

project = AIProjectClient(
    endpoint=PROJECT_ENDPOINT,
    credential=DefaultAzureCredential(),
)
openai = project.get_openai_client()

agent = project.agents.create_version(
    agent_name="my-agent",
    definition=PromptAgentDefinition(
        model="gpt-4.1-mini",
        instructions="You are a helpful assistant.",
        tools=[WebSearchTool()],
    ),
)

response = openai.responses.create(
    input="What are the latest updates to Microsoft Foundry?",
    extra_body={"agent_reference": {"name": agent.name, "type": "agent_reference"}},
)
print(response.output_text)
```

### Key URLs

- Overview: https://learn.microsoft.com/en-us/azure/foundry/agents/overview
- Environment Setup: https://learn.microsoft.com/en-us/azure/foundry/agents/environment-setup
- Tool Catalog: https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/tool-catalog
- MCP Connection: https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/tools/model-context-protocol
- MCP Auth: https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/mcp-authentication
- Foundry Samples GitHub: https://github.com/microsoft-foundry/foundry-samples

---

## 2. Azure OpenAI Assistants API vs Foundry Agent Service

### Azure OpenAI Assistants API

- The **legacy** approach: directly calls the Azure OpenAI Assistants v2 API
- No native MCP server support — tools are restricted to built-in types (code_interpreter, file_search, function_calling)
- Lower-level control: you manage threads, runs, messages manually
- Backed only by Azure OpenAI models (GPT-4o, GPT-4.1, etc.)
- Uses the `openai` Python SDK directly with an `AzureOpenAI` client

### Foundry Agent Service (Recommended)

- **Supersedes** the Assistants API for new projects
- Adds first-class MCP server support, Azure AI Search, SharePoint, Bing, Azure Functions
- The **Basic Setup** is Assistants API-compatible (same underlying thread/run model)
- Adds support for non-OpenAI models (Llama, DeepSeek, Grok)
- Adds Toolbox (versioned, shareable tool bundles)
- Manages Entra identity, OAuth passthrough, RBAC natively

### When to use each

| Use Case | Recommendation |
|----------|---------------|
| MCP server tools | **Foundry Agent Service** (only option) |
| Multi-model (non-OpenAI) | Foundry Agent Service |
| Simple single-model chatbot (existing code) | Azure OpenAI Assistants API is fine |
| Enterprise identity + RBAC + VNet | Foundry Agent Service |
| New projects in 2026 | **Foundry Agent Service** |

---

## 3. Connecting MCP Servers to Foundry Agent Service

### How it works

1. You bring a **remote MCP server endpoint** (HTTPS URL with streamable HTTP transport)
2. You create an `MCPTool` on the agent, providing `server_url`, `server_label`, authentication credentials via a project connection
3. When the agent decides to call an MCP tool, Foundry sends the request to your MCP server endpoint, injects auth headers, and returns the result to the model
4. Tool calls with `require_approval="always"` (the default) require your application to explicitly approve before execution

### Python Code — Agent with Single MCP Server

```python
import json
from azure.identity import DefaultAzureCredential
from azure.ai.projects import AIProjectClient
from azure.ai.projects.models import PromptAgentDefinition, MCPTool
from openai.types.responses.response_input_param import McpApprovalResponse

PROJECT_ENDPOINT = "https://<resource_name>.ai.azure.com/api/projects/<project_name>"
MCP_CONNECTION_NAME = "my-mcp-connection"  # project connection storing credentials

project = AIProjectClient(
    endpoint=PROJECT_ENDPOINT,
    credential=DefaultAzureCredential(),
)
openai = project.get_openai_client()

tool = MCPTool(
    server_label="my-api",
    server_url="https://my-mcp-server.azurewebsites.net/runtime/webhooks/mcp",
    require_approval="never",       # or "always" for human-in-the-loop
    project_connection_id=MCP_CONNECTION_NAME,
    allowed_tools=["read_emails", "post_to_teams"],  # optional allowlist
)

agent = project.agents.create_version(
    agent_name="email-plm-agent",
    definition=PromptAgentDefinition(
        model="gpt-4.1",
        instructions="You are an agent that reads emails and updates PLM records.",
        tools=[tool],
    ),
)

conversation = openai.conversations.create()
response = openai.responses.create(
    conversation=conversation.id,
    input="Check my inbox for new PLM change orders",
    extra_body={"agent_reference": {"name": agent.name, "type": "agent_reference"}},
)
print(response.output_text)
```

### Python Code — Agent with Multiple MCP Servers

```python
from azure.ai.projects.models import PromptAgentDefinition, MCPTool

office_mcp_tool = MCPTool(
    server_label="office365",
    server_url="https://graph.microsoft.com/mcp",   # Agent365 MCP endpoint (Frontier)
    require_approval="never",
    project_connection_id="office365-oauth-connection",
)

teams_mcp_tool = MCPTool(
    server_label="teams",
    server_url="https://graph.microsoft.com/teams-mcp",  # Teams MCP endpoint (Frontier)
    require_approval="never",
    project_connection_id="teams-oauth-connection",
)

autodesk_mcp_tool = MCPTool(
    server_label="autodesk-plm",
    server_url="https://my-autodesk-mcp.azurewebsites.net/mcp",
    require_approval="always",   # require approval for PLM writes
    project_connection_id="autodesk-api-key-connection",
)

agent = project.agents.create_version(
    agent_name="email-to-plm-agent",
    definition=PromptAgentDefinition(
        model="gpt-4.1",
        instructions=SYSTEM_PROMPT,
        tools=[office_mcp_tool, teams_mcp_tool, autodesk_mcp_tool],
    ),
)
```

### Known Limitations

- Max 128 tools per agent
- Non-streaming MCP tool call timeout: 100 seconds
- Private MCP endpoints require Standard Setup + VNet
- Agent Service only accepts **remote** MCP endpoints (no local stdio)
- Custom MCP servers must use **streamable HTTP** transport when hosted on Azure Functions

---

## 4. Agent 365 MCP Servers (Microsoft 365 Frontier)

### What they are

Microsoft provides first-party MCP servers for Microsoft 365 services under the "Frontier" program. These are only available to **Frontier tenants** (Microsoft adoption program at https://adoption.microsoft.com/en-us/copilot/frontier-program/).

### Available Frontier MCP Servers

| MCP Server | Permission Scope |
|------------|----------------|
| Microsoft Outlook Mail MCP Server | `McpServers.Mail.All` |
| Microsoft Outlook Calendar MCP Server | `McpServers.Calendar.All` |
| Microsoft Teams MCP Server | `McpServers.Teams.All` |
| Microsoft 365 User Profile MCP Server | `McpServers.Me.All` |
| Microsoft SharePoint and OneDrive MCP Server | `McpServers.OneDriveSharepoint.All` |
| Microsoft SharePoint Lists MCP Server | `McpServers.SharepointLists.All` |
| Microsoft Word MCP Server | `McpServers.Word.All` |
| Microsoft 365 Copilot (Search) MCP Server | `McpServers.CopilotMCP.All` |
| Microsoft 365 Admin Center MCP Server | `McpServers.M365Admin.All` |
| Microsoft Dataverse MCP Server | `McpServers.Dataverse.All` |

These use the App ID `ea9ffc3e-8a23-4a7d-836d-234d7c7565c1` in Entra.

### Authentication Setup for Agent365 MCP

1. Register a Microsoft Entra app (get client_id + client_secret)
2. Grant API permissions for Agent365 Tools: `ea9ffc3e-8a23-4a7d-836d-234d7c7565c1/{permission}`
3. Grant admin consent for the tenant
4. In Foundry portal: Build > Tools > Custom > MCP, configure with OAuth Identity Passthrough:
   - Auth URL: `https://login.microsoftonline.com/{tenantId}/oauth2/v2.0/authorize`
   - Token URL: `https://login.microsoftonline.com/{tenantId}/oauth2/v2.0/token`
   - Refresh URL: `https://login.microsoftonline.com/{tenantId}/oauth2/v2.0/token`
   - Scopes: `ea9ffc3e-8a23-4a7d-836d-234d7c7565c1/McpServers.Mail.All`
5. Add the redirect URL from Foundry back to your Entra app

**Note for non-Frontier tenants:** If you do not have Frontier access, you must build your own Office MCP server using Microsoft Graph API and host it on Azure Functions or Container Apps.

---

## 5. MCP Authentication Options

### Summary Table

| Method | Use Case | Per-User? |
|--------|----------|-----------|
| Key-based (API key / Bearer token) | Third-party APIs (Autodesk PLM) | No |
| Microsoft Entra — Agent Identity (preview) | Agent has its own service principal | No |
| Microsoft Entra — Project Managed Identity (preview) | All agents in project share identity | No |
| OAuth Identity Passthrough | User's identity passed through (Office MCP, Teams MCP) | Yes |
| Unauthenticated | Public/internal VNet MCP servers | No |

### OAuth Identity Passthrough Flow (for Office/Teams MCP)

1. User sends a message to the agent
2. Agent tries to call the Office MCP tool
3. Foundry returns `oauth_consent_request` with a `consent_link` URL
4. Your application surfaces the link to the user
5. User authorizes in browser (single sign-in; consent stored per project-tool pair)
6. Submit a follow-up response using `previous_response_id`
7. Agent continues and calls the MCP tool with the user's token

```python
# Step 1: Initial call
response = client.responses.create(
    input=user_message,
    extra_body={"agent_reference": {"name": agent.name, "type": "agent_reference"}},
)

# Step 2: Check for consent request
for item in response.output:
    if item.type == "oauth_consent_request":
        consent_link = item.consent_link
        # Surface consent_link to the user

# Step 3: After user consents, re-submit
response2 = client.responses.create(
    previous_response_id=response.id,
    input=user_message,
    extra_body={"agent_reference": {"name": agent.name, "type": "agent_reference"}},
)
```

### Key-based Auth (for Autodesk PLM MCP)

Store credentials in a Foundry project connection:
```python
from azure.ai.projects.models import MCPTool

tool = MCPTool(
    server_label="autodesk-plm",
    server_url="https://my-autodesk-mcp.azurewebsites.net/mcp",
    require_approval="always",
    project_connection_id="autodesk-api-key-connection",
    # The connection stores: header name="Authorization", value="Bearer <autodesk_token>"
)
```

---

## 6. Hosting a Custom MCP Server (e.g., Autodesk PLM MCP)

Since Autodesk Fusion/Manage does not have a native MCP server, you must build one.

### Option A: Azure Functions (Recommended for simple APIs)

```bash
# Scaffold with azd
azd init --template self-hosted-mcp-scaffold-python
azd up
```

Sample repo: https://github.com/Azure-Samples/mcp-sdk-functions-hosting-python

Key requirements:
- Uses streamable HTTP transport
- `host.json` configures the custom handler
- Supports OBO (On-Behalf-Of) flow for delegated auth
- Functions MCP webhook endpoint: `/runtime/webhooks/mcp`

```python
# server.py — minimal MCP server with FastMCP
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("Autodesk PLM")

@mcp.tool()
async def get_change_orders(status: str = "pending") -> list:
    """Get Autodesk PLM change orders by status"""
    # Call Autodesk Manage API
    ...

@mcp.tool()
async def update_change_order(order_id: str, status: str, notes: str) -> dict:
    """Update an Autodesk PLM change order"""
    ...

if __name__ == "__main__":
    mcp.run(transport="http")
```

### Option B: Azure Container Apps

```bash
# For more complex servers or those with OS-level dependencies
docker build -t mcp-autodesk-plm .
az containerapp create \
  --name mcp-autodesk-plm \
  --resource-group <rg> \
  --image mcp-autodesk-plm \
  --ingress external
```

Container transport: HTTP POST/GET. Internal-only ingress for private MCP in VNet.

Sample repo: https://github.com/Azure-Samples/mcp-container-ts

---

## 7. Required Azure Services/Resources

### Basic Setup (Quick Start)

| Resource | Purpose |
|----------|---------|
| Azure Subscription | Billing |
| Microsoft Foundry Account | Top-level AI resource |
| Foundry Project | Isolated workspace for agents, files, tools |
| Azure OpenAI model deployment (GPT-4.1 or GPT-4.1-mini) | Agent's reasoning model |
| Microsoft Entra App Registration | OAuth for Office/Teams MCP auth |

### Standard Setup (Production)

All Basic resources plus:

| Resource | Purpose |
|----------|---------|
| Azure Storage Account | Files, conversation history |
| Azure Cosmos DB | Thread state, conversation persistence |
| Azure AI Search | Vector store for file search |
| Azure Functions or Container Apps | Hosting custom MCP servers |
| Azure Key Vault | Secrets management (managed by Foundry, or BYOK) |
| Azure Monitor / Application Insights | Tracing, metrics |

### Standard Setup + VNet (Enterprise)

All Standard resources plus:

| Resource | Purpose |
|----------|---------|
| Azure Virtual Network | Private networking |
| Dedicated MCP subnet (Container Apps delegation) | Private MCP server endpoints |
| Private Endpoints | Private connectivity to storage, Cosmos, Search |

**Deploy template:**
- Basic: "Deploy to Azure" button at https://learn.microsoft.com/en-us/azure/foundry/agents/environment-setup
- Standard: Same page, Standard option
- Infrastructure Bicep: https://github.com/microsoft-foundry/foundry-samples/tree/main/infrastructure/infrastructure-setup-bicep

### RBAC Roles Required

| Action | Role |
|--------|------|
| Create Account + Project | Azure AI Account Owner (subscription scope) |
| Assign RBAC for BYO resources | Role Based Access Control Administrator |
| Create and run agents | Azure AI User |

---

## 8. Semantic Kernel MCP Integration

### Overview

**Semantic Kernel** (SK) is Microsoft's open-source SDK for AI orchestration (C#, Python, Java). It supports adding MCP servers as plugins via `MCPStdioPlugin` or `MCPSsePlugin`/`MCPStreamableHttpPlugin`.

### Python Setup

```bash
pip install semantic-kernel[mcp]
```

### Local MCP Server (stdio — e.g., docker/uvx)

```python
import os
from semantic_kernel import Kernel
from semantic_kernel.connectors.mcp import MCPStdioPlugin

async def main():
    async with MCPStdioPlugin(
        name="AutodeskPLM",
        description="Autodesk PLM Plugin",
        command="docker",
        args=["run", "-i", "--rm", "my-autodesk-plm-mcp"],
        env={"AUTODESK_API_KEY": os.getenv("AUTODESK_API_KEY")},
    ) as plm_plugin:
        kernel = Kernel()
        kernel.add_plugin(plm_plugin)
        # Use with ChatCompletionAgent or kernel.invoke()
```

### Remote MCP Server (SSE/HTTP)

```python
from semantic_kernel import Kernel
from semantic_kernel.connectors.mcp import MCPSsePlugin

async def main():
    async with MCPSsePlugin(
        name="AutodeskPLM",
        description="Autodesk PLM Plugin",
        url="https://my-autodesk-mcp.azurewebsites.net/mcp",
    ) as plm_plugin:
        kernel = Kernel()
        kernel.add_plugin(plm_plugin)
```

### Notes

- SK does **not** have built-in Office 365 / Teams MCP support — you bring your own MCP server URL
- SK is better for code-first orchestration with fine-grained control
- Works with Azure OpenAI via `AzureChatCompletion` connector
- Docs: https://learn.microsoft.com/en-us/semantic-kernel/concepts/plugins/adding-mcp-plugins

---

## 9. LangChain / LangGraph with MCP

### `langchain-mcp-adapters` Package

```bash
pip install langchain-mcp-adapters langgraph "langchain[openai]"
```

Package: https://github.com/langchain-ai/langchain-mcp-adapters (v0.2.2, March 2026)

### Single MCP Server

```python
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client
from langchain_mcp_adapters.tools import load_mcp_tools
from langchain.agents import create_agent

server_params = StdioServerParameters(
    command="python", args=["/path/to/plm_server.py"]
)

async with stdio_client(server_params) as (read, write):
    async with ClientSession(read, write) as session:
        await session.initialize()
        tools = await load_mcp_tools(session)
        agent = create_agent("openai:gpt-4.1", tools)
        response = await agent.ainvoke({"messages": "Get pending change orders"})
```

### Multiple MCP Servers

```python
from langchain_mcp_adapters.client import MultiServerMCPClient
from langchain.agents import create_agent

client = MultiServerMCPClient({
    "office365": {
        "url": "https://my-office365-mcp.azurewebsites.net/mcp",
        "transport": "http",
        "headers": {"Authorization": "Bearer <token>"},
    },
    "teams": {
        "url": "https://my-teams-mcp.azurewebsites.net/mcp",
        "transport": "http",
        "headers": {"Authorization": "Bearer <token>"},
    },
    "autodesk-plm": {
        "url": "https://my-autodesk-mcp.azurewebsites.net/mcp",
        "transport": "http",
        "headers": {"Authorization": "Bearer <autodesk_token>"},
    },
})

tools = await client.get_tools()
agent = create_agent("openai:gpt-4.1", tools)
```

### LangGraph StateGraph with MCP

```python
from langchain_mcp_adapters.client import MultiServerMCPClient
from langgraph.graph import StateGraph, MessagesState, START
from langgraph.prebuilt import ToolNode, tools_condition
from langchain.chat_models import init_chat_model

model = init_chat_model("azure_openai:gpt-4.1")
client = MultiServerMCPClient({...})
tools = await client.get_tools()

def call_model(state: MessagesState):
    response = model.bind_tools(tools).invoke(state["messages"])
    return {"messages": response}

builder = StateGraph(MessagesState)
builder.add_node(call_model)
builder.add_node(ToolNode(tools))
builder.add_edge(START, "call_model")
builder.add_conditional_edges("call_model", tools_condition)
builder.add_edge("tools", "call_model")
graph = builder.compile()
```

### Notes

- LangGraph supports hosted agent deployment via LangGraph API Server / LangGraph Platform
- Use `"azure_openai:gpt-4.1"` as model identifier when using Azure OpenAI
- LangChain MCP adapters are **framework-level** — no built-in Entra OAuth support; you manage tokens yourself
- Transport supports: `stdio`, `sse`, `http` (streamable HTTP)

---

## 10. Microsoft AutoGen with MCP

### Overview

**AutoGen** (v0.7.5) is Microsoft's open-source multi-agent framework. MCP support is via `autogen-ext[mcp]`.

```bash
pip install "autogen-agentchat" "autogen-ext[openai,mcp]"
```

### `McpWorkbench` — Recommended Pattern

```python
import asyncio
from autogen_ext.models.openai import AzureOpenAIChatCompletionClient
from autogen_ext.tools.mcp import McpWorkbench, SseServerParams
from autogen_agentchat.agents import AssistantAgent
from autogen_agentchat.ui import Console

async def main():
    model_client = AzureOpenAIChatCompletionClient(
        model="gpt-4.1",
        azure_endpoint="https://<resource>.openai.azure.com/",
        api_version="2025-01-01-preview",
        azure_deployment="gpt-41",
    )

    # Connect to Autodesk PLM MCP server
    plm_params = SseServerParams(
        url="https://my-autodesk-mcp.azurewebsites.net/mcp",
        headers={"Authorization": "Bearer <autodesk_token>"},
        timeout=30,
    )

    async with McpWorkbench(server_params=plm_params) as mcp:
        agent = AssistantAgent(
            "plm_agent",
            model_client=model_client,
            workbench=mcp,
            reflect_on_tool_use=True,
        )
        await Console(agent.run_stream(task="Get all pending ECOs from Autodesk PLM"))

asyncio.run(main())
```

### Multi-MCP via Multiple Workbenches / Tools

AutoGen's `AssistantAgent` supports a `workbench` (single MCP server) or `tools` list (multiple `SseMcpToolAdapter` instances):

```python
from autogen_ext.tools.mcp import mcp_server_tools, SseServerParams

office_tools = await mcp_server_tools(SseServerParams(url="...", headers={...}))
teams_tools = await mcp_server_tools(SseServerParams(url="...", headers={...}))
plm_tools = await mcp_server_tools(SseServerParams(url="...", headers={...}))

agent = AssistantAgent(
    "orchestrator",
    model_client=model_client,
    tools=[*office_tools, *teams_tools, *plm_tools],
)
```

### Key Classes

| Class | Transport | Use Case |
|-------|-----------|----------|
| `StdioMcpToolAdapter` | stdio | Local CLI-based MCP servers |
| `SseMcpToolAdapter` | SSE | Remote HTTP/SSE MCP servers |
| `StreamableHttpMcpToolAdapter` | Streamable HTTP | Remote streamable HTTP MCP servers |
| `McpWorkbench` | All three | Higher-level wrapper; supports resources/prompts too |
| `mcp_server_tools()` | All three | Factory: returns all tools from a server |

- Docs: https://microsoft.github.io/autogen/stable/reference/python/autogen_ext.tools.mcp.html
- AutoGen home: https://microsoft.github.io/autogen/stable/

---

## 11. Recommended Approach: Decision Matrix

| Scenario | Recommended Framework |
|----------|----------------------|
| **Hackathon / quick prototype** | Microsoft Foundry Agent Service (prompt agent, portal) |
| **Production, Microsoft-managed infra, MCP tools** | **Microsoft Foundry Agent Service** (azure-ai-projects SDK) |
| **Multi-agent coordination** | Foundry Workflow agents OR AutoGen OR LangGraph as hosted agent |
| **C#/.NET team** | Semantic Kernel + MCPSsePlugin |
| **Python-first, LLM-agnostic** | AutoGen (McpWorkbench) or LangGraph (langchain-mcp-adapters) |
| **No-code / low-code** | Microsoft Copilot Studio |
| **Enterprise integration (no AI)** | Azure Logic Apps + Power Automate |

### Microsoft's Stated Recommendation (2026)

Per the official docs: **Microsoft Foundry Agent Service** is the recommended approach for MCP-based agent orchestration on Azure. It is the only option with:
- Native Entra identity integration for MCP
- Built-in OAuth identity passthrough for Agent365 MCP servers
- Managed toolbox versioning
- Azure RBAC, VNet isolation, content safety guardrails
- First-party support for Teams/Office365 MCP servers (Frontier program)

---

## 12. Agent System Prompt — Email-to-PLM Workflow

```
You are an intelligent enterprise workflow agent for [Company Name].

Your primary workflow:
1. Read new emails from the user's Outlook inbox using the office365 MCP server
2. Identify emails related to engineering change requests, PLM updates, or part number queries
3. Extract relevant information: change order numbers, part numbers, requestor names, urgency
4. Look up or update the corresponding records in Autodesk PLM using the autodesk-plm MCP server
5. Post a summary notification to the designated Microsoft Teams channel using the teams MCP server

Rules:
- Only process emails with subjects containing keywords: "ECO", "ECR", "change order", "PLM", "BOM"
- Before updating any PLM record, summarize the change and ask for confirmation unless explicitly told to auto-approve
- When posting to Teams, use the #engineering-updates channel
- Never share credentials or personal data outside the approved MCP servers
- If a PLM record cannot be found, escalate via Teams to @engineering-leads

Available tools:
- office365: read emails, list inbox, get email body
- teams: post message, mention user, create meeting
- autodesk-plm: get_change_orders, update_change_order, get_part_number, search_bom

Current date: {current_date}
User context: {user_name} ({user_email})
```

---

## 13. Cost Model

### Microsoft Foundry Agent Service

- **No additional charge** for the Foundry Agent Service runtime itself
- You pay for:
  - **Model tokens**: Azure OpenAI token pricing (input + output tokens per model)
  - **Azure OpenAI model deployment**: pay-as-you-go or provisioned throughput
  - **Supporting resources**: Azure Storage, Cosmos DB, AI Search, Functions, Container Apps (Standard setup only)
  - **File search**: charged as Azure AI Search (if using Standard setup BYO Search)

### Azure OpenAI Pricing Reference (as of 2026)

| Model | Input (per 1M tokens) | Output (per 1M tokens) |
|-------|----------------------|----------------------|
| GPT-4.1 | ~$2.00 | ~$8.00 |
| GPT-4.1-mini | ~$0.40 | ~$1.60 |
| GPT-4o | ~$2.50 | ~$10.00 |

*(Check https://azure.microsoft.com/en-us/pricing/details/cognitive-services/openai-service/ for current prices)*

### Direct Azure OpenAI Assistants API vs Foundry Agent Service

- **Identical token costs** — both use the same Azure OpenAI model deployments
- Foundry Agent Service **adds no per-call fee** for the agent orchestration layer
- Foundry Standard Setup has costs for Cosmos DB threads storage (typically < $10/month for moderate usage)
- **Key difference**: Foundry gives more capabilities (MCP, Toolbox, VNet) at the same token cost

### Azure Logic Apps (for comparison)

- Standard Plan: ~$0.000016/action step
- Connector calls (Office 365, Teams): included in Standard Plan
- Much cheaper than AI agents for pure workflow automation, but no reasoning capability

---

## 14. Microsoft Copilot Studio (No-Code Alternative)

### What it is

**Microsoft Copilot Studio** (https://copilotstudio.microsoft.com) is a low-code/no-code graphical tool for building agents and agent flows. It is built on Azure OpenAI and Power Platform connectors.

### Capabilities

- Visual conversation flow builder
- 1000+ prebuilt Power Platform connectors (including Office 365 Mail, Teams, and custom HTTP)
- Agent flows (Power Automate-based, triggered by agent or schedule)
- Can extend Microsoft 365 Copilot
- Publish to Teams, websites, custom apps

### MCP Support in Copilot Studio

- Copilot Studio now supports connecting to MCP servers as tools (in preview as of 2026)
- Configure via the "Tools" section in the agent builder
- OAuth connectors available for Office 365 Mail, Calendar, Teams natively — no custom MCP needed for M365

### Comparison for This Workflow

| Aspect | Copilot Studio | Foundry Agent Service |
|--------|---------------|----------------------|
| Technical expertise | Low (no-code) | Medium (Python SDK) |
| MCP server support | Preview (limited) | GA, full SDK control |
| Custom Autodesk PLM | Via custom HTTP connector or MCP | Native MCPTool |
| Office 365 integration | Native connectors (no Frontier needed) | Requires Frontier MCP or custom server |
| Teams integration | Native connector | Requires Teams MCP (Frontier) or custom |
| Enterprise governance | Power Platform DLP + Entra | Azure RBAC + Foundry policies |
| Cost | Per-message (consumption-based) + capacity | Azure OpenAI tokens |

**Recommendation**: Use Copilot Studio if the team lacks Python experience. Use Foundry Agent Service if you need full MCP control, custom Autodesk PLM integration, or enterprise-grade Azure networking.

---

## 15. Azure Logic Apps (Non-AI Alternative)

### When to consider

If the workflow is deterministic (no reasoning required):
- Read email → parse → call Autodesk API → post Teams notification

Logic Apps can do all of this with prebuilt connectors:
- Office 365 Outlook connector (read mail, trigger on new email)
- HTTP connector (call Autodesk Manage REST API)
- Microsoft Teams connector (post adaptive card)

### Cost comparison

Logic Apps Standard: ~$0.000016/action, very cheap for high volume
AI agent: $2–10 per 1M tokens, meaningful cost at scale

### When to add AI

Add Foundry Agent Service **on top of** Logic Apps when you need:
- Natural language interpretation of email content
- Ambiguous field mapping (e.g., "part number 12345 or 12345-A?")
- Reasoning about which PLM fields to update
- Escalation decision-making

---

## 16. Key GitHub Repositories & Official Samples

| Repository | Description | URL |
|------------|-------------|-----|
| microsoft-foundry/foundry-samples | Official Foundry documentation samples (Python, Bicep, C#) | https://github.com/microsoft-foundry/foundry-samples |
| Azure-Samples/mcp-sdk-functions-hosting-python | Host MCP servers on Azure Functions (Python) | https://github.com/Azure-Samples/mcp-sdk-functions-hosting-python |
| Azure-Samples/mcp-sdk-functions-hosting-dotnet | Host MCP servers on Azure Functions (.NET) | https://github.com/Azure-Samples/mcp-sdk-functions-hosting-dotnet |
| Azure-Samples/mcp-container-ts | Host MCP servers on Azure Container Apps (TypeScript) | https://github.com/Azure-Samples/mcp-container-ts |
| langchain-ai/langchain-mcp-adapters | LangChain/LangGraph MCP adapters | https://github.com/langchain-ai/langchain-mcp-adapters |
| microsoft/autogen | Microsoft AutoGen framework | https://github.com/microsoft/autogen |
| microsoft-foundry/foundry-samples (infra) | Bicep templates for all agent setups (basic/standard/VNet) | https://github.com/microsoft-foundry/foundry-samples/tree/main/infrastructure/infrastructure-setup-bicep |

### No Specific Office MCP + Teams MCP Hackathon Sample Found

As of 2026-05-04, there is no single public sample combining all three (Office MCP + Teams MCP + Autodesk PLM MCP). The Agent 365 MCP servers (Outlook Mail, Teams) are Frontier-gated. However, the building blocks are all documented and available.

**Suggested approach for hackathon:**
1. Use `foundry-samples` for the Foundry agent setup skeleton
2. Use `mcp-sdk-functions-hosting-python` to scaffold the Autodesk PLM MCP server
3. For Office/Teams MCP: either apply for Frontier access or build minimal Graph-API-backed MCP servers using the functions hosting template

---

## 17. Architecture Diagram Description

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Microsoft Foundry Agent Service                   │
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────────┐ │
│  │                   Prompt Agent (GPT-4.1)                        │ │
│  │  Instructions: Email-to-PLM workflow system prompt              │ │
│  │                                                                  │ │
│  │  Tools:                                                          │ │
│  │  ├── MCPTool: office365 (Agent365 Mail MCP - Frontier)          │ │
│  │  │    auth: OAuth Identity Passthrough → Entra App              │ │
│  │  ├── MCPTool: teams (Agent365 Teams MCP - Frontier)             │ │
│  │  │    auth: OAuth Identity Passthrough → Entra App              │ │
│  │  └── MCPTool: autodesk-plm (Custom MCP on Azure Functions)      │ │
│  │       auth: Key-based → Autodesk API key in project connection  │ │
│  └─────────────────────────────────────────────────────────────────┘ │
│                                                                       │
│  Identity: Managed Identity + Per-user OAuth passthrough             │
│  Storage: Cosmos DB (threads) + Blob (files) + AI Search (vectors)  │
│  Observability: Application Insights + Agent tracing                 │
└─────────────────────────────────────────────────────────────────────┘
            │                    │                    │
            ▼                    ▼                    ▼
  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────────┐
  │  Agent 365 Mail  │  │ Agent 365 Teams  │  │  Azure Functions     │
  │  MCP Server      │  │ MCP Server       │  │  (Custom MCP)        │
  │  (Frontier)      │  │  (Frontier)      │  │                      │
  │  → Graph API     │  │  → Teams API     │  │  → Autodesk Manage   │
  │    Office 365    │  │                  │  │    REST API          │
  └──────────────────┘  └──────────────────┘  └──────────────────────┘
```

---

## 18. Environment Setup Checklist

### Azure Resources to Create

```bash
# 1. Deploy Foundry environment (Basic setup — fastest for hackathon)
# Use "Deploy to Azure" button at:
# https://learn.microsoft.com/en-us/azure/foundry/agents/environment-setup
# Or via Bicep:
cd foundry-samples/infrastructure/infrastructure-setup-bicep/01-basic-agent-setup
az deployment sub create --location eastus --template-file main.bicep

# 2. Install Python packages
pip install azure-ai-projects azure-identity openai

# 3. Set environment variables
export PROJECT_ENDPOINT="https://<resource>.ai.azure.com/api/projects/<project>"
export AZURE_CLIENT_ID="<managed-identity-or-service-principal-id>"

# 4. Deploy Autodesk PLM MCP server
git clone https://github.com/Azure-Samples/mcp-sdk-functions-hosting-python
# Add Autodesk PLM tools to server.py
azd up

# 5. Register project connections in Foundry portal
# Build > Tools > Connections > + New Connection
# For Autodesk: Custom/API Key
# For Office365: OAuth Identity Passthrough (Frontier required)
```

---

## Summary & Recommendations

### For This Email-to-PLM Workflow Specifically

1. **Primary recommendation**: **Microsoft Foundry Agent Service** with `azure-ai-projects` SDK
   - Use `MCPTool` with OAuth passthrough for Office365/Teams (requires Frontier tenant)
   - Build Autodesk PLM MCP server on Azure Functions (use `mcp-sdk-functions-hosting-python` template)
   - Key auth: API key for Autodesk, OAuth passthrough for Office/Teams

2. **Alternative if not on Frontier**: Build minimal Office 365 + Teams MCP servers wrapping Microsoft Graph API, host on Azure Functions, use key-based auth or Entra managed identity

3. **If team is Python-heavy and wants flexibility**: AutoGen with `McpWorkbench` or LangGraph with `langchain-mcp-adapters` deployed as a **Foundry Hosted Agent** (container)

4. **If team wants no-code**: Microsoft Copilot Studio + Power Automate flows + native Office 365/Teams connectors (no custom MCP needed for M365 integration)

5. **For pure automation (no AI reasoning needed)**: Azure Logic Apps with built-in connectors

### Key Clarifying Questions (User Input Needed)

1. Is the tenant enrolled in the **Frontier program**? This determines whether Agent 365 MCP servers (Outlook Mail, Teams) are available directly
2. What Autodesk product is used — **Fusion Manage** (PLM) or **Vault** (PDM)? Each has different APIs
3. Is human approval required before PLM updates (for `require_approval` configuration)?
4. What is the target deployment scale — single user, team, or enterprise-wide?
5. Is the team comfortable with Python SDK, or is a no-code approach preferred?

---

## References

- Microsoft Foundry Agent Service Overview: https://learn.microsoft.com/en-us/azure/foundry/agents/overview
- Connect to MCP Servers: https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/tools/model-context-protocol
- MCP Authentication: https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/mcp-authentication
- Tool Catalog: https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/tool-catalog
- Environment Setup: https://learn.microsoft.com/en-us/azure/foundry/agents/environment-setup
- Limits & Quotas: https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/limits-quotas-regions
- Semantic Kernel MCP Plugins: https://learn.microsoft.com/en-us/semantic-kernel/concepts/plugins/adding-mcp-plugins
- AutoGen MCP Reference: https://microsoft.github.io/autogen/stable/reference/python/autogen_ext.tools.mcp.html
- AutoGen Home: https://microsoft.github.io/autogen/stable/
- LangChain MCP Adapters GitHub: https://github.com/langchain-ai/langchain-mcp-adapters
- LangChain MCP Adapters PyPI: https://pypi.org/project/langchain-mcp-adapters/
- Foundry Samples GitHub: https://github.com/microsoft-foundry/foundry-samples
- MCP on Azure Functions (Python): https://github.com/Azure-Samples/mcp-sdk-functions-hosting-python
- MCP on Azure Container Apps (TS): https://github.com/Azure-Samples/mcp-container-ts
- Copilot Studio Overview: https://learn.microsoft.com/en-us/microsoft-copilot-studio/overview
- Frontier Program: https://adoption.microsoft.com/en-us/copilot/frontier-program/
- MCP Security Best Practices (Microsoft): https://techcommunity.microsoft.com/blog/microsoft-security-blog/understanding-and-mitigating-security-risks-in-mcp-implementations/4404667
- Integrate MCP Tools with Azure AI Agents (Training): https://learn.microsoft.com/en-us/training/modules/connect-agent-to-mcp-tools/
- Foundry Infrastructure Bicep Templates: https://github.com/microsoft-foundry/foundry-samples/tree/main/infrastructure/infrastructure-setup-bicep

