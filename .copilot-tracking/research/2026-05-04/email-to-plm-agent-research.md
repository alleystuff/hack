<!-- markdownlint-disable-file -->
# Task Research: AI Agent — Email-to-PLM Autodesk Integration

An AI Agent that reads live Outlook emails via Microsoft Graph API, reasons about which Autodesk PLM design changes are required based on email content, writes the updated design records to local CSV files, and posts a change summary to a Microsoft Teams channel. The agent loop runs **locally** and uses an Azure AI Foundry GPT-4o deployment as the LLM endpoint. All three MCP servers run as local subprocesses managed by the `mcp` Python client — no ngrok or remote hosting required.

## Task Implementation Requests

* Read live Outlook emails from an active M365 account using the `@softeria/ms-365-mcp-server` MCP server and Microsoft Graph API
* Read Autodesk PLM design records from local CSV files via a custom Python MCP server
* Agent reasons about which design changes are needed based on the live email contents (no hardcoded rules)
* Update the PLM design records in local CSV files based on the agent's reasoning
* Send a Microsoft Teams channel message summarizing the changes made
* Agent loop runs locally; GPT-4o is called via an Azure AI Foundry model deployment endpoint
* All three MCP servers run as local subprocesses managed by the `mcp` Python client library

## Scope and Success Criteria

* Scope: End-to-end AI agent pipeline — live Outlook email in (Graph API) → local agent reasons (Foundry LLM endpoint) → CSV PLM design updated → Teams notification out. All PLM data is local CSV. Agent loop and all MCP servers run on developer machine.
* Assumptions:
  * Active Microsoft 365 account with Outlook mailbox and mock PLM-related emails seeded
  * Azure AD app registration (public client) for Graph API delegated permissions
  * Azure AI Foundry project with GPT-4o model deployment (used as LLM API endpoint only)
  * Microsoft Teams tenant with an "Engineering" team and "PLM Updates" channel
  * No Autodesk Fusion Manage API or subscription required — PLM data stays in local CSV
  * No ngrok or remote hosting required — all MCP servers are local subprocesses
* Success Criteria:
  * Agent reads unprocessed/unread emails from a live Outlook inbox via Graph API
  * Agent reads matching PLM design records from local CSV files
  * Agent reasons about required design changes from live email content (no hardcoded rules)
  * Updated PLM design records written back to local CSV files
  * Teams message posted with markdown summary of changes made
  * All integration touch points demonstrably working end-to-end

## Outline

1. System Dependencies
2. Integration Map — How Systems Connect
3. System Architecture and Data Flow
4. Component: Outlook MCP Server (`@softeria/ms-365-mcp-server`) + Microsoft Graph API
5. Component: PLM Data MCP Server (Python FastMCP — CSV read/write)
6. Component: Microsoft Teams MCP Server (`@floriscornel/teams-mcp`)
7. Component: Local Agent Loop + Azure AI Foundry LLM Endpoint (GPT-4o)
8. PLM CSV Data Schema
9. Agent Reasoning Approach
10. Agent System Prompt
11. Selected Approach with Full Implementation Details
12. Alternatives Considered

## Research Executed

### External Research

* **`@softeria/ms-365-mcp-server`** (v0.99.1, 685 GitHub stars — recommended Outlook MCP):
  * npm: `@softeria/ms-365-mcp-server`
  * GitHub: https://github.com/Softeria/ms-365-mcp-server
  * 200+ tools covering the full Graph API; use `--preset mail` to restrict to email tools only
  * Auth: device code flow (default, ideal for CLI), BYOT (Bring Your Own Token), or OAuth auth-code (HTTP mode)
  * Key permissions: `Mail.Read`, `offline_access`, `User.Read`
  * Run `npx @softeria/ms-365-mcp-server --login` once; tokens cached in OS credential store or file
  * Supports `$filter=isRead eq false`, `$orderby=receivedDateTime desc`, `$select=...` via Graph query params

* **Microsoft Graph API — Email Reading:**
  * Endpoint: `GET /me/mailFolders/inbox/messages?$filter=isRead eq false&$orderby=receivedDateTime desc&$top=50`
  * `Mail.ReadBasic` — metadata only (no body); `Mail.Read` — full body content (required for agent reasoning)
  * Device code flow requires Azure AD public client app registration ("Allow public client flows" enabled)
  * `@odata.nextLink` pattern for pagination; do NOT manipulate `$skip` manually
  * Filtering works on: `isRead`, `from/emailAddress/address`, `receivedDateTime`, `startswith(subject,...)`

* **Azure AI Foundry Agent Service** (`azure-ai-projects>=2.0.0`, GA as of 2026):
  * Replaces raw Azure OpenAI chat completions loop; service handles tool orchestration server-side
  * `MCPTool(server_label=..., server_url=..., require_approval=...)` — native MCP support, no `mcp` Python bridge needed
  * Two MCP servers = two `MCPTool` instances in `tools=[]` array
  * **Critical constraint:** Foundry only accepts remote MCP endpoints — local stdio servers must be tunneled publicly
  * Hackathon mitigation: `ngrok http 8000` exposes local Python FastMCP server over HTTPS instantly
  * Conversation API (`openai.conversations.create()` + `openai.responses.create()`) manages multi-turn history server-side
  * v2.x is incompatible with v1.x (deprecated `create_thread` / `create_run` pattern)
  * Entra ID auth via `DefaultAzureCredential` or API key via `AzureKeyCredential`

* **`@floriscornel/teams-mcp`** (v1.0.0, 97 GitHub stars):
  * `send_channel_message` confirmed with Markdown, importance levels, and `@mention` support
  * Delegated auth via device code flow; tokens cached at `~/.teams-mcp-token-cache.json`
  * 26 tools total: `list_teams`, `list_channels`, `send_channel_message`, etc.

* **Python `mcp` library (FastMCP) — PLM CSV MCP Server:**
  * `pip install mcp[cli]`; `FastMCP` with `@mcp.tool()` decorators
  * Server must be started as an HTTP server (`--port 8000`) rather than stdio so ngrok can tunnel it
  * `csv.DictReader` / `csv.DictWriter` for PLM record read/write

### Project Conventions

* Greenfield project — Python preferred
* **Three MCP servers:** Outlook MCP (`@softeria/ms-365-mcp-server`) + PLM Data MCP (Python FastMCP, local + ngrok) + Teams MCP (`@floriscornel/teams-mcp`)
* Live Outlook M365 account required for email reading
* No Autodesk Fusion Manage API or APS credentials — PLM data stays in local CSV
* Azure AI Foundry project required for agent orchestration (replaces direct Azure OpenAI)

---

## Key Discoveries

### Critical Findings

1. **Live Outlook via `@softeria/ms-365-mcp-server`** — The only mature community MCP server for Outlook/M365 email (685 stars, 209 releases, v0.99.1). Uses device code flow; authenticate once, tokens cached. Use `--preset mail` to reduce context overhead. Supports full Graph OData filtering (`isRead eq false`, date range, sender, subject prefix).

2. **`Mail.Read` is the required Graph permission** — `Mail.ReadBasic` only exposes metadata, not email body. Agent reasoning requires full body content, so `Mail.Read` (delegated) is the minimum viable permission.

3. **Azure AI Foundry replaces direct Azure OpenAI** — Foundry Agent Service (v2.x, GA 2026) provides native `MCPTool` support. No `mcp` Python bridge library needed. Two `MCPTool` instances attach the PLM CSV server and the Teams server. Built-in tracing, portal observability, and managed multi-turn conversation state.

4. **Foundry requires remote MCP endpoints** — Local Python FastMCP servers running on `localhost` are not reachable by the Foundry cloud runtime. Hackathon solution: `ngrok http 8000` provides an instant public HTTPS tunnel. Production path: Azure Container Apps.

5. **PLM data stays in local CSV** — No Autodesk Fusion Manage API or subscription required. The custom Python FastMCP server exposes CSV read/write operations as MCP tools. Foundry calls these tools via the ngrok tunnel URL.

6. **Teams channel messaging requires delegated auth** — `ChannelMessage.Send` is only available as a delegated (user-context) permission. Authenticate once with `npx @floriscornel/teams-mcp authenticate`; tokens auto-refreshed.

7. **Three MCP servers, one agent** — Outlook MCP (read emails) + PLM CSV MCP (read/write design records) + Teams MCP (send notification). All three are `MCPTool` entries in the Foundry agent definition.

8. **Agent reasoning is the core value** — GPT-4o receives live email body + PLM design record fields in one context window and reasons about which field changes are warranted. No hardcoded status mappings.

### System Dependencies

| Dependency | Purpose | Auth / Credentials |
|---|---|---|
| **Microsoft 365 account (Outlook)** | Live email source; agent reads inbox via Graph API | Active M365 tenant + user account with seeded PLM-related emails |
| **Azure AD app registration (public client)** | Grants delegated `Mail.Read` permission for Graph API access | Client ID + tenant ID (no secret — public client) |
| **`@softeria/ms-365-mcp-server`** (npm) | Outlook MCP server — exposes Graph email tools to Foundry | Device code flow at setup; tokens cached in `~/.config/ms365-mcp/` |
| **Microsoft Graph API** | Backend for Outlook email reading (`/me/mailFolders/inbox/messages`) | Delegated OAuth token from Azure AD via device code |
| **Azure AI Foundry project** | GPT-4o LLM endpoint — agent loop calls it for model inference only | Foundry project endpoint + API key or `DefaultAzureCredential` |
| **GPT-4o model deployment** (Foundry) | Language model for reasoning about email → PLM changes | Part of Foundry project |
| **Python FastMCP server** (local subprocess) | PLM CSV MCP server — exposes design record read/write tools | No external auth; local process managed by `mcp` Python client |
| **`@floriscornel/teams-mcp`** (npm) | Teams MCP server — sends change summary to channel | Device code flow at setup; `ChannelMessage.Send` delegated permission |
| **Microsoft Teams tenant** | Notification output channel ("Engineering" team, "PLM Updates" channel) | Same M365 tenant as Outlook account |
| **Local CSV files** | PLM design records storage (`ecn_records.csv`, `cr_records.csv`, `part_records.csv`) | Filesystem only — no external credentials |

### Integration Map — How Systems Connect

```
┌──────────────────────────────────────────────────────┐
│  Outlook Mailbox (M365 cloud)                           │
│  Emails seeded with ECN/CR/Part identifiers            │
└────────────────────────┬─────────────────────────────┘
                         │ Microsoft Graph API
                         │ (Mail.Read delegated)
                         ▼
┌──────────────────────────────────────────────────────┐
│  DEVELOPER MACHINE                                     │
│                                                        │
│  agent/main.py  (local Python process)                 │
│    ├─ mcp client ─► @softeria/ms-365-mcp-server (subprocess) │
│    ├─ mcp client ─► plm-data-mcp/server.py  (subprocess) │
│    ├─ mcp client ─► @floriscornel/teams-mcp  (subprocess) │
│    └─ openai SDK ─► Azure AI Foundry GPT-4o (HTTPS out)   │
│                                                        │
│  All MCP servers are stdio subprocesses — no ngrok     │
└────────────────────────┬─────────────────────────────┘
              │ model inference (HTTPS)
              ▼
┌──────────────────────────────────────────────────────┐
│  Azure AI Foundry (cloud)                              │
│  GPT-4o model deployment                               │
│  (LLM endpoint only — no agent runtime used)           │
└──────────────────────────────────────────────────────┘
```

### PLM CSV Data Model

Emails are read live from Outlook via Graph API — no `emails.csv` file is used. PLM design records remain in local CSV files:

* `ecn_records.csv` — ECN design change records; mutable fields: `status`, `priority`, `approver`, `comments`
* `cr_records.csv` — Change Request records linked to ECNs; mutable fields: `status`, `comments`
* `part_records.csv` — Part master records; mutable field: `status` (e.g. Obsolete)
* Multi-value columns use pipe separator: `affected_parts=100-4821|100-4822`
* In-memory dict holds all pending writes during a session; flushed to CSV on each `update_design_record` call

### Agent Reasoning Approach

The agent receives the full email body and the matching PLM design record fields together in a single LLM context window. It reasons about:
* What the email is requesting (approval, status change, obsolescence alert, digest)
* Which specific fields on which design record should change
* What the new field values should be and why
* What summary to write to Teams

This reasoning step is not rule-based — the LLM infers the action from context, making it robust to varied email formats.

### Complete Tool Call Sequence (8 Steps)

```
1. list-mail-messages(filter=isRead eq false)  → read unread emails from live Outlook inbox (Outlook MCP)
2. get-mail-message(id)                        → get full email body for each unread message (Outlook MCP)
3. search_design_record(identifier)            → find ECN/CR/Part row from local CSV files (PLM CSV MCP)
4. [LLM reasons about changes]                 → no tool call; pure GPT-4o reasoning step
5. update_design_record(id, record_type, ...)  → write updated fields back to CSV (PLM CSV MCP)
6. mark_email_read(message_id)                 → mark email as read in Outlook via Graph API (Outlook MCP)
7. list_teams() + list_channels()              → find Engineering team + PLM Updates channel (Teams MCP)
8. send_channel_message(teamId, channelId, md) → post markdown change summary (Teams MCP)
```

### Agent System Prompt (ready-to-use)

See Section 8 below for the complete system prompt.

---

## Technical Scenarios

### Selected Approach: Local Agent Loop + Azure AI Foundry LLM Endpoint + Three Local MCP Servers

**Architecture:**

```
sequenceDiagram
    autonumber
    participant User as User / Script
    participant Agent as Local Agent Loop<br/>(agent/main.py + mcp Python client)
    participant LLM as Azure AI Foundry<br/>(GPT-4o endpoint, HTTPS)
    participant OutlookMCP as Outlook MCP Server<br/>(@softeria/ms-365-mcp-server, subprocess)
    participant PLM as PLM CSV MCP Server<br/>(plm-data-mcp/server.py, subprocess)
    participant Files as Local CSV Files<br/>(plm-data-mcp/data/*.csv)
    participant Teams as Teams MCP Server<br/>(@floriscornel/teams-mcp, subprocess)
    participant TeamsAPI as Microsoft Teams<br/>(Graph API)

    User->>Agent: python agent/main.py
    Note over Agent: mcp client starts all 3 MCP servers as local subprocesses
    Agent->>OutlookMCP: list-mail-messages($filter=isRead eq false)
    OutlookMCP->>GraphMail: GET /me/mailFolders/inbox/messages?$filter=isRead eq false
    GraphMail-->>OutlookMCP: unread email list (id, subject, sender, receivedDateTime)
    OutlookMCP-->>Agent: unread email summaries

    loop For each unread email
        Agent->>OutlookMCP: get-mail-message(messageId)
        OutlookMCP->>GraphMail: GET /me/messages/{id}?$select=body,subject,from,...
        GraphMail-->>OutlookMCP: full message with body content
        OutlookMCP-->>Agent: email with full body text

        Note over Agent,LLM: Agent sends email+record to GPT-4o (Foundry HTTPS)
        Agent->>LLM: chat.completions.create(tools=[...], messages=[...])
        LLM-->>Agent: tool_call: search_design_record(identifier)

        Agent->>PLM: search_design_record(identifier)
        PLM->>Files: search ecn_records.csv / cr_records.csv / part_records.csv
        Files-->>PLM: matching design record row
        PLM-->>Agent: design record fields

        Note over Agent,LLM: GPT-4o reasons about required design changes
        Agent->>LLM: tool result injected; next tool_call
        LLM-->>Agent: tool_call: update_design_record(...)

        Agent->>PLM: update_design_record(id, record_type, fields)
        PLM->>Files: write updated row to CSV
        Files-->>PLM: write confirmed
        PLM-->>Agent: update result

        Agent->>OutlookMCP: mark-message-read(messageId)
        OutlookMCP->>GraphMail: PATCH /me/messages/{id} {isRead: true}
        GraphMail-->>OutlookMCP: 200 OK
        OutlookMCP-->>Agent: confirmed
    end

    Agent->>Teams: list_teams()
    Teams->>TeamsAPI: GET /me/joinedTeams
    TeamsAPI-->>Teams: team list
    Teams-->>Agent: team IDs

    Agent->>Teams: list_channels(teamId)
    Teams-->>Agent: channel IDs

    Agent->>Teams: send_channel_message(teamId, channelId, markdownSummary)
    Teams->>TeamsAPI: POST /teams/{id}/channels/{id}/messages
    TeamsAPI-->>Teams: 201 Created
    Teams-->>Agent: success

    Agent->>User: print run summary
```

**Requirements:**

* Python 3.10+
* Node.js 20+ (for Outlook MCP and Teams MCP npm servers)
* Azure AI Foundry project with GPT-4o model deployment (LLM endpoint only)
* Azure AD public client app registration (tenant ID + client ID; `Mail.Read`, `offline_access`, `User.Read`)
* Active Microsoft 365 account with Outlook inbox containing mock PLM emails
* Microsoft Teams tenant with "Engineering" team and "PLM Updates" channel
* **No ngrok or remote hosting required**
* No Autodesk account required

**File Tree:**

```text
hack/
├── agent/
│   ├── main.py                         # Local agent loop: mcp client + openai SDK
│   ├── agent_prompt.py                 # System prompt definition
│   ├── tool_executor.py               # Dispatches GPT-4o tool calls to mcp sessions
│   └── requirements.txt               # mcp[cli], openai, azure-identity, python-dotenv
├── plm-data-mcp/
│   ├── server.py                       # FastMCP stdio server: search_design_record,
│   │                                   #   update_design_record
│   ├── csv_store.py                    # CSV read/write helper (csv.DictReader/DictWriter)
│   ├── models.py                       # Pydantic result models
│   ├── requirements.txt               # mcp[cli], pydantic
│   └── data/
│       ├── ecn_records.csv             # ECN / design change records
│       ├── cr_records.csv              # Change Request records
│       └── part_records.csv            # Part master records
├── mcp_servers.json                    # MCP subprocess config (command, args, env per server)
└── .env.example                        # Azure AI Foundry + M365 auth variables
```

**MCP Subprocess Config (mcp_servers.json):**

Defines how the `mcp` Python client starts each server as a local subprocess:

```json
{
  "outlook-mail": {
    "command": "npx",
    "args": ["-y", "@softeria/ms-365-mcp-server", "--preset", "mail"]
  },
  "plm-csv-data": {
    "command": "python",
    "args": ["plm-data-mcp/server.py"],
    "env": { "PLM_DATA_DIR": "plm-data-mcp/data" }
  },
  "teams-notify": {
    "command": "npx",
    "args": ["-y", "@floriscornel/teams-mcp@latest"]
  }
}
```

**Environment Variables (.env.example):**

```bash
# ─────────────────────────────────────────────────
# Azure AI Foundry — LLM endpoint only
# ─────────────────────────────────────────────────
AZURE_OPENAI_ENDPOINT=https://<your-resource>.openai.azure.com/
AZURE_OPENAI_API_KEY=<your-api-key>
AZURE_OPENAI_DEPLOYMENT=gpt-4o

# ─────────────────────────────────────────────────
# Azure AD — public client app registration
# Authenticate once: npx @softeria/ms-365-mcp-server --login
# Token cached at: ~/.config/ms365-mcp/.token-cache.json
# ─────────────────────────────────────────────────
AZURE_AD_CLIENT_ID=<app-registration-client-id>
AZURE_AD_TENANT_ID=<your-tenant-id>

# ─────────────────────────────────────────────────
# PLM CSV MCP Server
# ─────────────────────────────────────────────────
PLM_DATA_DIR=plm-data-mcp/data

# ─────────────────────────────────────────────────
# Microsoft Teams — @floriscornel/teams-mcp
# Authenticate once: npx @floriscornel/teams-mcp authenticate
# Token cached at: ~/.teams-mcp-token-cache.json
# ─────────────────────────────────────────────────
TEAMS_TEAM_NAME=Engineering
TEAMS_CHANNEL_NAME=PLM Updates
```

**Local Agent Loop (agent/main.py — key snippet):**

```python
import asyncio, json, os
from openai import AzureOpenAI
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client
from dotenv import load_dotenv
from agent_prompt import SYSTEM_PROMPT

load_dotenv()

client = AzureOpenAI(
    azure_endpoint=os.environ["AZURE_OPENAI_ENDPOINT"],
    api_key=os.environ["AZURE_OPENAI_API_KEY"],
    api_version="2024-12-01-preview",
)

SERVERS = json.load(open("mcp_servers.json"))

async def run():
    sessions = {}
    all_tools = []

    # Start all MCP servers as local subprocesses
    for name, cfg in SERVERS.items():
        params = StdioServerParameters(command=cfg["command"], args=cfg["args"], env=cfg.get("env"))
        read, write = await stdio_client(params).__aenter__()
        session = ClientSession(read, write)
        await session.initialize()
        sessions[name] = session
        tools = await session.list_tools()
        for t in tools.tools:
            all_tools.append({"type": "function", "function": {"name": f"{name}__{t.name}", "description": t.description, "parameters": t.inputSchema}})

    messages = [{"role": "system", "content": SYSTEM_PROMPT}]
    messages.append({"role": "user", "content": "Process unread PLM emails and update design records."})

    # Agent loop
    while True:
        response = client.chat.completions.create(
            model=os.environ["AZURE_OPENAI_DEPLOYMENT"],
            messages=messages,
            tools=all_tools,
        )
        msg = response.choices[0].message
        messages.append(msg)
        if not msg.tool_calls:
            print(msg.content)
            break
        for tc in msg.tool_calls:
            server_name, tool_name = tc.function.name.split("__", 1)
            result = await sessions[server_name].call_tool(tool_name, json.loads(tc.function.arguments))
            messages.append({"role": "tool", "tool_call_id": tc.id, "content": str(result.content)})

asyncio.run(run())
```

---

## Agent System Prompt

```
You are an Autodesk Engineering PLM Agent. Your job is to:

1. READ all unread emails from the Outlook inbox using the list-mail-messages tool
   (filter: isRead eq false, ordered by receivedDateTime desc).
   These are live Outlook engineering emails from active M365 accounts.

2. For each unread email:
   a. GET the full email body using get-mail-message(messageId).
   b. IDENTIFY any PLM record identifiers mentioned in the body:
      - ECN numbers (pattern: ECN-YYYY-NNNN)
      - Change Request numbers (pattern: CR-YYYY-NNNN)
      - Part numbers (pattern: NNN-NNNN where NNN is 100–600)
   c. RETRIEVE the matching design record using search_design_record.
   d. REASON about what changes to the design record are warranted based on
      the email content and the current record state. Consider:
      - What is the email asking or notifying about?
      - What field(s) on the design record should change, and to what value?
      - Is this actionable (update required) or informational (notify only)?
   e. If actionable: UPDATE the design record using update_design_record.
   f. MARK the email as read using mark-message-read(messageId) after processing.

3. After processing all emails, NOTIFY the Engineering team on Microsoft Teams:
   - Use list_teams then list_channels to find the "PLM Updates" channel
     in the "Engineering" team.
   - Post a single markdown summary message covering all changes made.
   - For each change: include ECN/record number, previous status, new status,
     reason (from email), and affected parts.
   - Use importance "high" only for URGENT items.

REASONING RULES:
- Do not apply hardcoded status mappings — reason from the email text each time.
- If the email is ambiguous, state your uncertainty in the Teams message
  and do NOT update the record.
- If no matching design record is found, include "Record not found" in the
  Teams notification.
- For digest/summary emails referencing multiple ECNs, report the overview
  without updating records unless a specific action is clearly requested.
- Always explain your reasoning for each update in the Teams message.
- Skip emails that are clearly unrelated to PLM changes (newsletters, meeting invites).

CURRENT DATE: {current_date}
ENGINEERING TEAM: resolve at runtime via list_teams
PLM CHANNEL: PLM Updates
```

---

## Mock Sample Data (Seed Emails in Outlook)

**Emails are sent to a live Outlook inbox** (active M365 account). Seed the inbox by sending 4 emails to the account before running the agent. PLM design records are stored as local CSV files in `plm-data-mcp/data/`.

### Seed Emails to Send to Outlook Inbox

| # | Subject | Key Identifiers in Body | Expected Agent Action |
|---|---|---|---|
| 1 | ECN-2024-0342: Engineering Change Notice Requires Your Approval | ECN-2024-0342, CR-2024-0287, 100-4821, 100-4822 | Update ECN status to Pending Approval |
| 2 | [PLM Notification] Change Request CR-2024-0301 Status Updated | CR-2024-0301, ECN-2024-0338 | Mirror status update on linked records |
| 3 | URGENT: Part 300-1204 obsoleted — downstream impact on ECN-2024-0355 | ECN-2024-0355, 300-1204, 300-1205 | Set ECN On Hold, alert Teams with high importance |
| 4 | [Weekly Digest] Open ECNs Awaiting Action — 4 items | ECN-2024-0339, ECN-2024-0342, ECN-2024-0350, ECN-2024-0355 | Compile overview only, no record updates |

### ecn_records.csv — Sample Autodesk PLM Design Records / ECNs (3 rows)

Columns: `id, workspace_id, number, title, status, priority, originator, approver, change_reason, associated_cr, implementation_date, affected_parts, plm_url`

| id | number | title | status |
|---|---|---|---|
| 1452 | ECN-2024-0342 | Redesign of Heat Exchanger Bracket Assembly | In Review |
| 1478 | ECN-2024-0355 | Seal Assembly Sub-Frame Component Update | In Review |
| 1461 | ECN-2024-0338 | Turbine Blade Profile Tolerance Update | In Design |

### cr_records.csv — Change Requests (2 rows)

Columns: `id, workspace_id, number, title, status, priority, description, requested_by, linked_ecn, plm_url`

### part_records.csv — Parts / Design Data (3 rows)

Columns: `id, workspace_id, number, title, revision, status, material, weight_kg, drawing_number, pending_ecn, plm_url`

Includes Part 300-1204 with `status=Obsolete` (drives the urgency scenario in email 3).

### CSV Design Notes

* Multi-value columns (e.g., `affected_parts`) use pipe-separated values: `100-4821|100-4822`
* The PLM CSV MCP server loads all CSVs at startup using Python `csv.DictReader`; updates are written immediately on each `update_design_record` tool call
* The Outlook inbox serves as the email source — no `emails.csv` file exists in this architecture

---

## Alternatives Considered

### Alternative 1: Azure AI Foundry Agent Service runtime with MCPTool (Not selected for hackathon)

* Use `azure-ai-projects>=2.0.0` `MCPTool` class where Foundry cloud runtime calls MCP servers directly
* **Not selected because:** Foundry Agent Service only accepts remote HTTPS MCP endpoints — all three local MCP servers would require public tunneling (ngrok or Azure Container Apps), adding infrastructure friction. The local agent loop approach achieves the same result with fewer moving parts and is easier to debug during a hackathon. Foundry Agent Service is the recommended production path.

### Alternative 2: Direct Azure OpenAI with `mcp` Python client bridge (same approach, different LLM endpoint)

* Use Azure OpenAI directly (not via Foundry project) with `chat.completions.create(tools=...)` and the `mcp` Python client
* **Not meaningfully different from selected approach:** Foundry model deployments expose the same Azure OpenAI-compatible endpoint. The selected approach already uses the `openai` SDK with `AzureOpenAI` client; the Foundry project endpoint is the URL. No functional difference.

### Alternative 3: Live Autodesk Fusion Manage API (Rejected)

* Call the Fusion Manage REST API v3 directly with APS 2-legged OAuth for PLM records
* **Rejected because:** Requires paid Autodesk Fusion Manage subscription; no public demo tenant available. Local CSV provides equivalent design data fidelity without any Autodesk account dependency.

### Alternative 4: Semantic Kernel with MCP Plugins (Not selected)

* `pip install semantic-kernel[mcp]` + `MCPSsePlugin` for tool integration
* **Not selected because:** More boilerplate than `azure-ai-projects` v2.x Foundry SDK. No advantage for this three-server setup. Foundry SDK is the recommended Microsoft path for Azure-hosted agents.

### Alternative 5: Copilot Studio No-Code (Rejected)

* Microsoft Copilot Studio with file connectors and Teams output
* **Rejected because:** Cannot demonstrate programmatic CSV I/O or custom reasoning logic; not code-demonstrable for a hackathon judged on technical depth.

### Alternative 6: Azure Container Apps for MCP server hosting (Deferred to production)

* Deploy the Python FastMCP server to Azure Container Apps with a stable public HTTPS URL
* **Deferred:** Correct production path when switching to Foundry Agent Service runtime, but unnecessary for the local agent loop approach selected for hackathon.

---

## Potential Next Research

* Research `@softeria/ms-365-mcp-server` stdio mode behaviour and token cache path
  * Reasoning: Confirm exact startup behaviour when launched as a subprocess by the `mcp` Python client (no `--login` prompt after initial auth); verify token cache persists across subprocess restarts
  * Reference: https://github.com/Softeria/ms-365-mcp-server

* Investigate Microsoft Graph Webhooks for real-time email triggers (post-hackathon)
  * Reasoning: Replace polling (`isRead eq false`) with push notifications for production; agent triggers immediately on new email
  * Reference: https://learn.microsoft.com/en-us/graph/webhooks

* Research Adaptive Cards for richer Teams notification formatting
  * Reasoning: Adaptive Cards are more visually structured than markdown for PLM summaries with action buttons
  * Reference: https://adaptivecards.io/

* Migrate PLM CSV data store to SQLite for concurrent-safe writes
  * Reasoning: CSV writes are not concurrent-safe; SQLite provides transactions with zero infrastructure overhead

* Migrate to live Autodesk Fusion Manage API (post-hackathon)
  * Reasoning: Swap CSV read/write in `csv_store.py` for APS REST API calls; MCP tool interface stays identical
  * Reference: https://help.autodesk.com/view/PLM/ENU/?guid=FLC_RestAPI_Read_Me_First_html

* Research Azure AI Foundry MCP approval flow for production safety
  * Reasoning: Use `require_approval="always"` on PLM update tools so a human confirms design changes before write
  * Reference: https://learn.microsoft.com/en-us/azure/ai-services/agents/how-to/tools/mcp
