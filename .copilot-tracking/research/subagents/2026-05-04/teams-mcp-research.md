# Microsoft Teams MCP Server Research

**Date:** 2026-05-04  
**Status:** Complete  
**Topic:** Microsoft Teams MCP Server — posting messages to Teams channels for AI agent pipelines

---

## Research Questions

1. What is the official Microsoft Teams MCP server? Source/documentation?
2. What MCP tools/endpoints does it expose for posting channel messages?
3. What authentication model does it use?
4. How do you configure it as an MCP server in an AI agent?
5. What Microsoft Graph or Bot Framework permissions are required?
6. What does a sample tool call look like?
7. Alternatives: incoming webhooks, Graph API direct calls, Adaptive Cards?
8. Known working examples or tutorials?

Also: Microsoft Teams Toolkit, Power Automate connectors, official vs. community MCP servers.

---

## Q1. Official Microsoft Teams MCP Server

### Summary

There is **no standalone public "Microsoft Teams MCP Server" GitHub repo** open to the public.
The official Microsoft Teams MCP server is part of Microsoft's **Agent365 MCP platform**,
hosted as a remote (cloud) server. Its source code lives in a Microsoft-internal enterprise
GitHub organization (`bap-microsoft/MCP-Platform`) that requires Microsoft Enterprise SSO to access.

### Official Sources

| Name | Details |
|---|---|
| Central Microsoft MCP catalog | https://github.com/microsoft/mcp |
| Teams MCP source (enterprise SSO required) | https://github.com/bap-microsoft/MCP-Platform/tree/main/src/Services/WebApi/MCPServers/FirstParty/CodeBased/mcp_TeamsServer |
| Agent365 platform | `https://agent365.svc.cloud.microsoft/agents/tenants/{tenant_id}/servers/mcp_TeamsServer` |

### Description (from microsoft/mcp catalog)

> "Manage Microsoft Teams chats, channels, users, and messages via Graph API.
> Features server-side filtering, pagination, and token optimization."

- **Category:** PRODUCTIVITY
- **Type:** REMOTE (cloud-hosted, not local)
- **Endpoint:** `https://agent365.svc.cloud.microsoft/agents/tenants/{tenant_id}/servers/mcp_TeamsServer`

### VS Code Install Link

```
https://vscode.dev/redirect/mcp/install?name=agent365-teamsserver&config={"type":"http","url":"https://agent365.svc.cloud.microsoft/agents/tenants/${input:tenant_id}/servers/mcp_TeamsServer"}&inputs=[{"id":"tenant_id","type":"promptString","description":"Microsoft Entra tenant ID (GUID)"}]
```

### Other Official Microsoft MCP Servers in the Agent365 Platform

All hosted at `https://agent365.svc.cloud.microsoft/agents/tenants/{tenant_id}/servers/...`:

| Server | Path Suffix | Description |
|---|---|---|
| Microsoft Teams | `mcp_TeamsServer` | Chats, channels, users, messages |
| Microsoft 365 Calendar | `mcp_CalendarTools` | Calendar events, availability |
| Microsoft 365 Mail | `mcp_MailTools` | Email read/send |
| Microsoft 365 Copilot Chat | `mcp_M365Copilot` | Search M365 content |
| OneDrive & SharePoint | `mcp_ODSPRemoteServer` | Files |
| SharePoint Lists | `mcp_SharePointListsTools` | Lists |
| Microsoft Word | `mcp_WordServer` | Document management |
| Microsoft Admin Center | `mcp_AdminTools` | Admin operations |
| Microsoft 365 User | `mcp_MeServer` | User profile, org chart |

---

## Q2. MCP Tools Exposed for Posting Messages

### Official Agent365 Teams MCP (inferred from description)

The official server "manages Teams chats, channels, users, and messages via Graph API."
The exact tool names are not publicly documented (source is behind enterprise SSO),
but based on Graph API capabilities and the description, it likely exposes tools for:
- Listing teams, channels, chats
- Reading channel/chat messages
- Sending messages to channels and chats
- Managing users/members

### Community Server: `floriscornel/teams-mcp` (Best Documented)

npm: `@floriscornel/teams-mcp`  
GitHub: https://github.com/floriscornel/teams-mcp  
Stars: 95 | Forks: 43 | Weekly visitors: ~653

**Full tool list (26 tools total):**

**Write tools (disabled in read-only mode):**

| Tool | Description |
|---|---|
| `send_channel_message` | Send a message to a team channel |
| `reply_to_channel_message` | Reply to an existing channel thread |
| `update_channel_message` | Edit a previously sent channel message or reply |
| `delete_channel_message` | Soft delete a channel message or reply |
| `send_file_to_channel` | Upload a local file and send it as a message to a channel |
| `send_chat_message` | Send a message to a chat |
| `create_chat` | Create a new 1:1 or group chat |
| `update_chat_message` | Edit a previously sent chat message |
| `delete_chat_message` | Soft delete a chat message |
| `send_file_to_chat` | Upload a local file and send it to a chat |

**Read tools:**

| Tool | Description |
|---|---|
| `auth_status` | Check current authentication status |
| `get_current_user` | Get authenticated user information |
| `search_users` | Search for users by name or email |
| `get_user` | Get detailed user information by ID or email |
| `list_teams` | List user's joined teams |
| `list_channels` | List channels in a specific team |
| `get_channel_messages` | Retrieve messages from a team channel |
| `get_channel_message_replies` | Get replies to a specific channel message |
| `list_team_members` | List members of a specific team |
| `search_users_for_mentions` | Search for team members to @mention |
| `download_message_hosted_content` | Download hosted content from channel messages |
| `list_chats` | List user's chats (1:1 and group) |
| `get_chat_messages` | Retrieve messages from a specific chat |
| `download_chat_hosted_content` | Download hosted content from chat messages |
| `search_messages` | Search across all Teams messages using KQL syntax |
| `get_my_mentions` | Find recent messages mentioning the current user |

### Community Server: `SurgeEnterpriseAI/teams-mcp-server`

GitHub: https://github.com/SurgeEnterpriseAI/teams-mcp-server  
11 tools, multi-tenant OAuth, STDIO + HTTP/SSE transport:

| Tool | Description |
|---|---|
| `teams_list_teams` | List all your Teams |
| `teams_list_channels` | List channels in a team |
| `teams_read_channel_messages` | Read recent channel messages |
| `teams_send_channel_message` | Send a message to a channel |
| `teams_reply_to_message` | Reply in a channel thread |
| `teams_read_message_replies` | Read thread replies |
| `teams_list_chats` | List your DMs and group chats |
| `teams_read_chat_messages` | Read messages from a chat |
| `teams_send_chat_message` | Send a direct/group chat message |
| `teams_search_messages` | Search messages across all Teams |
| `teams_get_profile` | Get your Microsoft 365 profile |

### Community Server: `FRESHSK/mcp_teams`

GitHub: https://github.com/FRESHSK/mcp_teams  
8 tools, MSAL device-code auth, JavaScript:

| Tool | Description |
|---|---|
| `list_chats` | View recent chats |
| `list_chat_messages` | Read message history of a chat |
| `send_chat_message` | Send a message to a chat |
| `create_chat` | Create a new chat |
| `list_teams` | View joined teams |
| `list_channels` | View channels within a team |
| `send_channel_message` | Post a message to a channel |
| `create_team` | Create a new Standard team |

---

## Q3. Authentication Model

### Official Agent365 Teams MCP (Remote)

- **Auth:** Microsoft Entra ID (Azure AD) OAuth 2.0
- **Flow:** Delegated (on-behalf-of user), tenant-specific
- **Scoping:** Tenant ID is required in the server URL; the MCP client handles the OAuth token exchange
- **Consent:** Requires Microsoft 365 Copilot license or appropriate tenant licensing

### Community Servers (`floriscornel/teams-mcp`)

- **Primary flow:** OAuth 2.0 **device code flow** via MSAL (`@azure/msal-node`)
- **Token storage:** Cached locally at `~/.teams-mcp-token-cache.json` and `~/.msgraph-mcp-auth.json`
- **Refresh:** Refresh tokens are automatically renewed; no hourly re-login
- **Alternative:** Pre-inject a Microsoft Graph JWT via `AUTH_TOKEN` env var
  - Token must target `https://graph.microsoft.com`
  - Useful for server-side automation / CI/CD pipelines

### Azure AD App Registration (required for community servers)

1. Go to **Azure Portal > App registrations > New registration**
2. Supported account types: "Accounts in any organizational directory (Any Entra ID tenant - Multitenant)" for multi-tenant or single-tenant
3. Redirect URI: `http://localhost` (public client / native app) for device code flow; or Web for OAuth code flow
4. Copy **Application (client) ID** and **Tenant ID**
5. Add API Permissions → Microsoft Graph → **Delegated**

---

## Q4. Configuration as MCP Server

### Official Agent365 Teams MCP (HTTP/Remote)

VS Code `settings.json` or MCP client config:

```json
{
  "mcp": {
    "servers": {
      "agent365-teamsserver": {
        "type": "http",
        "url": "https://agent365.svc.cloud.microsoft/agents/tenants/YOUR_TENANT_ID/servers/mcp_TeamsServer"
      }
    }
  }
}
```

### `floriscornel/teams-mcp` (STDIO, most common)

**Standard config (Claude Desktop, Cursor, VS Code):**

```json
{
  "mcpServers": {
    "teams-mcp": {
      "command": "npx",
      "args": ["-y", "@floriscornel/teams-mcp@latest"]
    }
  }
}
```

**With pre-issued token (CI/CD, server-side pipelines):**

```json
{
  "mcpServers": {
    "teams-mcp": {
      "command": "npx",
      "args": ["-y", "@floriscornel/teams-mcp@latest"],
      "env": {
        "AUTH_TOKEN": "<jwt-for-https://graph.microsoft.com>"
      }
    }
  }
}
```

**Read-only mode:**

```json
{
  "mcpServers": {
    "teams-mcp": {
      "command": "npx",
      "args": ["-y", "@floriscornel/teams-mcp@latest"],
      "env": {
        "TEAMS_MCP_READ_ONLY": "true"
      }
    }
  }
}
```

**Authentication one-time setup:**

```bash
# Authenticate interactively (opens browser/device login)
npx @floriscornel/teams-mcp@latest authenticate

# Read-only mode
npx @floriscornel/teams-mcp@latest authenticate --read-only

# Check status
npx @floriscornel/teams-mcp@latest check

# Logout / clear tokens
npx @floriscornel/teams-mcp@latest logout
```

### `SurgeEnterpriseAI/teams-mcp-server` (STDIO local or HTTP/SSE remote)

**STDIO (local):**

```json
{
  "mcpServers": {
    "teams": {
      "command": "node",
      "args": ["/path/to/teams-mcp-server/dist/index.js"],
      "cwd": "/path/to/teams-mcp-server"
    }
  }
}
```

**Environment variables:**

```env
TEAMS_CLIENT_ID=your_azure_app_client_id
TEAMS_CLIENT_SECRET=your_azure_app_client_secret
TEAMS_AUTHORITY=common       # or specific tenant ID
TRANSPORT_MODE=auto          # auto | stdio | http
PORT=3000
BASE_URL=http://localhost:3000
REDIRECT_URI=http://localhost:3000/auth/callback
TOKEN_STORE_PATH=./.data/tokens
```

---

## Q5. Required Microsoft Graph Permissions

### Minimum for sending channel messages

| Permission | Type | Purpose |
|---|---|---|
| `ChannelMessage.Send` | Delegated | Send messages to channels |
| `Team.ReadBasic.All` | Delegated | List and read teams |
| `Channel.ReadBasic.All` | Delegated | List and read channels |
| `User.Read` | Delegated | Read own profile |

### Full read/write access (floriscornel full mode)

| Permission | Type | Purpose |
|---|---|---|
| `User.Read` | Delegated | Read user profile |
| `User.ReadBasic.All` | Delegated | Read basic user info |
| `Team.ReadBasic.All` | Delegated | List teams |
| `Channel.ReadBasic.All` | Delegated | List channels |
| `ChannelMessage.Read.All` | Delegated | Read channel messages |
| `ChannelMessage.Send` | Delegated | Send channel messages/replies |
| `ChannelMessage.ReadWrite` | Delegated | Edit and delete channel messages |
| `Chat.ReadWrite` | Delegated | Create chats, send/edit/delete chat messages |
| `TeamMember.Read.All` | Delegated | Read team members |
| `Files.ReadWrite.All` | Delegated | File uploads to channels/chats |

### Important note on application permissions

**Application permissions** (non-delegated) for `ChannelMessage.Send` are **not available**.
The only application permission is `Teamwork.Migrate.All`, which is limited to message migration scenarios only.
For normal bot/agent use cases, you **must use delegated permissions** (user context).

### API throttle limits

- `POST /teams/{id}/channels/{id}/messages`: max 10 messages per 10 seconds per conversation
- Rate limiting applies: use exponential backoff on HTTP 429 responses

---

## Q6. Sample Tool Call to Send a Channel Message

### Using `floriscornel/teams-mcp` — `send_channel_message` tool

**Input schema:**

```json
{
  "teamId": "fbe2bf47-16c8-47cf-b4a5-4b9b187c508b",
  "channelId": "19:4a95f7d8db4c4e7fae857bcebe0623e6@thread.tacv2",
  "message": "**Build succeeded!** Deployment to staging is complete.",
  "format": "markdown",
  "importance": "high",
  "mentions": [
    {
      "mention": "alex.chen",
      "userId": "00000000-0000-0000-0000-000000000000"
    }
  ]
}
```

**Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `teamId` | string | yes | Team GUID |
| `channelId` | string | yes | Channel thread ID |
| `message` | string | yes | Message content |
| `format` | "text" \| "markdown" | no | Default: "text" |
| `importance` | "normal" \| "high" \| "urgent" | no | Default: "normal" |
| `mentions` | array | no | Users to @mention |
| `imageUrl` | string | no | Inline image URL or base64 |

**Underlying Graph API call:**

```http
POST https://graph.microsoft.com/v1.0/teams/{team-id}/channels/{channel-id}/messages
Content-type: application/json
Authorization: Bearer {token}

{
  "body": {
    "contentType": "html",
    "content": "<strong>Build succeeded!</strong> Deployment to staging is complete."
  },
  "importance": "high"
}
```

**Response (HTTP 201 Created):**

```json
{
  "id": "1616990032035",
  "replyToId": null,
  "messageType": "message",
  "createdDateTime": "2021-03-29T03:53:52.035Z",
  "body": {
    "contentType": "html",
    "content": "<strong>Build succeeded!</strong> Deployment to staging is complete."
  },
  "channelIdentity": {
    "teamId": "fbe2bf47-16c8-47cf-b4a5-4b9b187c508b",
    "channelId": "19:4a95f7d8db4c4e7fae857bcebe0623e6@thread.tacv2"
  },
  "webUrl": "https://teams.microsoft.com/l/message/..."
}
```

### Using `SurgeEnterpriseAI/teams-mcp-server` — `teams_send_channel_message` tool

```json
{
  "teamId": "fbe2bf47-16c8-47cf-b4a5-4b9b187c508b",
  "channelId": "19:4a95f7d8db4c4e7fae857bcebe0623e6@thread.tacv2",
  "message": "Hello from an AI agent pipeline!"
}
```

---

## Q7. Alternatives for Posting to Teams Channels

### A. Teams Incoming Webhooks (via Power Automate Workflows)

**Status:** Office 365 Connectors are **deprecated** as of 2025–2026. New connectors cannot be created. The replacement is **Power Automate Workflows**.

**Modern approach (2026):**

1. In Teams, go to a channel → More options (...) → **Workflows**
2. Select template: "Send webhook alerts to a channel" or build from scratch
3. Use trigger: "When a Teams webhook request is received"
4. Copy the generated webhook URL
5. HTTP POST to that URL:

```http
POST https://xxxxx.webhook.office.com/webhookb2/...
Content-Type: application/json

{
  "type": "message",
  "attachments": [
    {
      "contentType": "application/vnd.microsoft.card.adaptive",
      "content": {
        "type": "AdaptiveCard",
        "body": [{ "type": "TextBlock", "text": "Hello from webhook!" }],
        "$schema": "http://adaptivecards.io/schemas/adaptive-card.json",
        "version": "1.0"
      }
    }
  ]
}
```

**Limitations:**
- Messages are 28 KB max
- Rate limit: 4 requests/second
- Workflows are linked to a user (not the channel); orphan risk if owner leaves
- Private channel posting support is still in development for flow bots

### B. Microsoft Graph API Direct Calls

Best for application-level integration. Requires registered Azure AD app.

**Endpoint:**
```
POST https://graph.microsoft.com/v1.0/teams/{team-id}/channels/{channel-id}/messages
```

**Simple text message:**
```json
{
  "body": { "content": "Hello World" }
}
```

**HTML with @mention:**
```json
{
  "body": {
    "contentType": "html",
    "content": "Hello <at id=\"0\">Jane Smith</at>"
  },
  "mentions": [
    {
      "id": 0,
      "mentionText": "Jane Smith",
      "mentioned": {
        "user": {
          "displayName": "Jane Smith",
          "id": "ef1c916a-...",
          "userIdentityType": "aadUser"
        }
      }
    }
  ]
}
```

**Graph API docs:** https://learn.microsoft.com/en-us/graph/api/chatmessage-post

### C. Adaptive Cards via Graph API

Graph API supports Adaptive Card attachments in channel messages:

```json
{
  "body": {
    "contentType": "html",
    "content": "<attachment id=\"74d20c7f34aa4a7fb74e2b30004247c5\"></attachment>"
  },
  "attachments": [
    {
      "id": "74d20c7f34aa4a7fb74e2b30004247c5",
      "contentType": "application/vnd.microsoft.card.adaptive",
      "content": "{\"type\":\"AdaptiveCard\",\"version\":\"1.2\",\"body\":[{\"type\":\"TextBlock\",\"text\":\"Hello!\"}]}"
    }
  ]
}
```

Card types supported:
- `application/vnd.microsoft.card.adaptive` — Adaptive Cards v1.2+
- `application/vnd.microsoft.card.thumbnail` — Thumbnail cards
- `application/vnd.microsoft.teams.messaging-announcementBanner` — Announcement

### D. Bot Framework / Proactive Messaging

For production agents/bots:
- Register a bot in Azure Bot Service
- Use `BotFrameworkAdapter` or Teams SDK (`microsoft/teams-sdk`)
- Send proactive messages to channels using conversation references
- Supports all card types, OAuth, and SSO
- More complex setup but production-grade

**Teams SDK:** https://github.com/microsoft/teams-sdk  
(Formerly Teams AI SDK, now renamed Teams SDK)

---

## Q8. Known Working Examples and Tutorials

### Official Microsoft

- Graph API post message docs: https://learn.microsoft.com/en-us/graph/api/chatmessage-post
- Incoming webhook docs: https://learn.microsoft.com/en-us/microsoftteams/platform/webhooks-and-connectors/how-to/add-incoming-webhook
- Teams SDK (bot framework): https://github.com/microsoft/teams-sdk
- Graph Explorer (interactive testing): https://developer.microsoft.com/en-us/graph/graph-explorer

### Community MCP Servers (Working Examples)

1. **`@floriscornel/teams-mcp`** (most popular, 95 stars, 43 forks)
   - GitHub: https://github.com/floriscornel/teams-mcp
   - npm: https://www.npmjs.com/package/@floriscornel/teams-mcp
   - Version: 1.0.0 (March 2026)
   - Weekly visitors: ~653

2. **`SurgeEnterpriseAI/teams-mcp-server`**
   - GitHub: https://github.com/SurgeEnterpriseAI/teams-mcp-server
   - Multi-tenant, STDIO + HTTP/SSE, 11 tools

3. **`FRESHSK/mcp_teams`**
   - GitHub: https://github.com/FRESHSK/mcp_teams
   - JavaScript, simpler implementation, device code flow

4. **Incoming webhook sample code (C#):**
   - https://github.com/OfficeDev/Microsoft-Teams-Samples/blob/main/samples/incoming-webhook/csharp/IncomingWebhook/Controllers/CardController.cs

---

## Microsoft Teams Toolkit (M365 Agents Toolkit)

The **Microsoft 365 Agents Toolkit** (formerly Teams Toolkit) is a VS Code extension for *building* Teams apps and agents — it is **not** an MCP server for posting messages. However, it includes an MCP server for developer workflows:

- **Purpose:** Build, test, deploy Teams apps and M365 Copilot agents
- **MCP Server:** Yes, it exposes an MCP server for AI assistants to help *developers* scaffold and deploy Teams apps
- **Repository:** https://github.com/OfficeDev/microsoft-365-agents-toolkit/
- **VS Code Extension:** `TeamsDevApp.ms-teams-vscode-extension`
- **Listed in:** https://github.com/microsoft/mcp catalog as "Microsoft 365 Agents Toolkit"

This is a developer tooling MCP server, not a messaging integration.

---

## Power Automate Connectors

Power Automate has a **Microsoft Teams connector** with actions including:
- Post a message to a channel
- Post a card to a channel
- Send message to a channel
- Create a channel
- List channels and teams

This is **not an MCP server** but can be triggered via HTTP (webhooks) or integrated with Logic Apps.
The new Power Automate webhook trigger ("When a Teams webhook request is received") is the recommended replacement for deprecated Office 365 Connectors.

---

## Official vs. Community Summary

| | Official (Agent365) | floriscornel/teams-mcp | SurgeEnterpriseAI | FRESHSK |
|---|---|---|---|---|
| Maintained by | Microsoft | Community | Community | Community |
| Source visible | No (enterprise SSO) | Yes | Yes | Yes |
| Type | Remote (cloud) | Local (STDIO) | Local+Remote | Local |
| Auth | Entra ID OAuth | MSAL device code / JWT | MSAL OAuth | MSAL device code |
| Requires M365 license | Yes (Copilot) | M365 account | M365 account | M365 account |
| Channel message send | Yes (inferred) | Yes (`send_channel_message`) | Yes (`teams_send_channel_message`) | Yes (`send_channel_message`) |
| Stars/maturity | N/A | 95 stars, v1.0 | 1 star, v1.0 | 1 star, initial |
| npm package | No | `@floriscornel/teams-mcp` | No npm | No npm |

---

## Key Configuration Reference

### Quick-start for AI agent pipeline (recommended path)

**Best option for most AI agent pipelines: `@floriscornel/teams-mcp`**

```bash
# 1. Authenticate once
npx @floriscornel/teams-mcp@latest authenticate

# 2. Add to your MCP config
```

```json
{
  "mcpServers": {
    "teams-mcp": {
      "command": "npx",
      "args": ["-y", "@floriscornel/teams-mcp@latest"]
    }
  }
}
```

**For automated/headless pipelines (pre-issued token):**

```json
{
  "mcpServers": {
    "teams-mcp": {
      "command": "npx",
      "args": ["-y", "@floriscornel/teams-mcp@latest"],
      "env": {
        "AUTH_TOKEN": "<Microsoft Graph JWT>"
      }
    }
  }
}
```

Obtain a Graph token for automation:
```bash
# Via Azure CLI (delegated)
az account get-access-token --resource https://graph.microsoft.com --query accessToken -o tsv

# Or use MSAL confidential client (app credentials) for daemon/service scenarios
```

### For official Microsoft Teams MCP (VS Code with M365 Copilot license)

```json
{
  "mcp": {
    "servers": {
      "teams": {
        "type": "http",
        "url": "https://agent365.svc.cloud.microsoft/agents/tenants/YOUR_TENANT_GUID/servers/mcp_TeamsServer"
      }
    }
  }
}
```

---

## Clarifying Questions (unresolved through research)

1. The exact tool names and input/output schemas for the official Agent365 `mcp_TeamsServer` are not publicly documented (source behind enterprise SSO). Clarification needed from Microsoft documentation or internal access.
2. What Microsoft 365 license tier is required to use the official Agent365 MCP servers? The catalog says "REMOTE" with the agent365 endpoint, implying it may require Microsoft 365 Copilot licensing.
3. Does the official server support `ChannelMessage.Send` application permissions (non-delegated), or is it also delegated-only?

