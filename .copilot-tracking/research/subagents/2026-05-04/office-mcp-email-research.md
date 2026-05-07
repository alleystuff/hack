# Microsoft Office MCP Server — Email Reading Research

**Research Date:** 2026-05-04  
**Status:** Complete  
**Researcher:** Subagent  

---

## Research Questions

1. What is the official Microsoft Office MCP server? (GitHub repo, npm package, Azure deployment)
2. What MCP tools/endpoints exist for reading emails (list messages, get message body, search)?
3. What authentication model does it use (OAuth 2.0, delegated vs. app permissions, MS Graph scopes)?
4. How do you configure it as an MCP server in an AI agent (config format, env variables)?
5. Rate limits, message size limits, notable constraints?
6. Sample tool call for reading an email (input schema, output schema)?
7. Known working examples or tutorials?
8. Required Microsoft Graph API permissions (Mail.Read, Mail.ReadAll, etc.)?
9. Alternative Office 365 / Outlook email MCP servers?

---

## Executive Summary

Two distinct options exist for reading Microsoft 365 / Outlook email via MCP:

1. **Official Microsoft Agent365 Mail MCP** — Cloud-hosted at `agent365.svc.cloud.microsoft`, part of Microsoft's internal Agent365 platform. Requires a Microsoft 365 tenant ID. Source code is behind Microsoft SSO and not publicly accessible; tool schemas are undocumented publicly. Likely requires an M365 Copilot subscription or equivalent.

2. **Community `@softeria/ms-365-mcp-server`** — Open-source (MIT), TypeScript, 675 stars, v0.96.0 as of 2026-05-04. Provides 200+ tools covering the full Microsoft Graph API surface including comprehensive email reading tools. Fully documented with known schemas, multiple auth flows, and active maintenance. **This is the practical choice for an AI agent pipeline today.**

---

## Q1: Official Microsoft Office MCP Server

### Primary Official Server — Microsoft 365 Mail (Agent365 Platform)

| Field | Value |
|---|---|
| Name | Microsoft 365 Mail |
| Platform | Agent365 MCP Platform |
| Type | REMOTE (cloud-hosted by Microsoft) |
| Endpoint | `https://agent365.svc.cloud.microsoft/agents/tenants/{tenant_id}/servers/mcp_MailTools` |
| Source repo | `https://github.com/bap-microsoft/MCP-Platform` (SSO-protected, requires Microsoft employee login) |
| Catalog | `https://github.com/microsoft/mcp` (public listing of all official Microsoft MCP servers) |
| License | Microsoft internal; not open source |

**VS Code Install Config:**
```json
{
  "type": "http",
  "url": "https://agent365.svc.cloud.microsoft/agents/tenants/${input:tenant_id}/servers/mcp_MailTools"
}
```
*(Requires `tenant_id` input — Microsoft Entra tenant GUID)*

**VS Code install URL (one-click):**
```
https://vscode.dev/redirect/mcp/install?name=agent365-mailtools&config=%7B%22type%22%3A%22http%22%2C%22url%22%3A%22https%3A//agent365.svc.cloud.microsoft/agents/tenants/%24%7Binput%3Atenant_id%7D/servers/mcp_MailTools%22%7D&inputs=%5B%7B%22id%22%3A%22tenant_id%22%2C%22type%22%3A%22promptString%22%2C%22description%22%3A%22Microsoft%20Entra%20tenant%20ID%20(GUID)%22%7D%5D
```

**Description (from official catalog):**
> "Email tools for creating, sending, replying, updating, deleting, and searching messages. Integrates with Microsoft Graph Mail APIs."

**Limitation:** Exact tool names and schemas are not publicly documented. The source repo requires Microsoft SSO to access.

### Other Official Agent365 MCP Servers

| Server ID | Description |
|---|---|
| `mcp_M365Copilot` | Searches across emails, docs, chats, and SharePoint sites |
| `mcp_CalendarTools` | Calendar integration |
| `mcp_TeamsServer` | Teams messages and channels |
| `mcp_MeServer` | User profile information |
| `mcp_ODSPRemoteServer` | OneDrive and SharePoint |

All share the same base URL pattern: `https://agent365.svc.cloud.microsoft/agents/tenants/{tenant_id}/servers/{server_id}`

---

## Q2: Email Reading Tools

### Official Agent365 Mail MCP Tools (partial, inferred from description)

The official server description lists: creating, sending, replying, updating, deleting, and **searching** messages. Exact tool names are not publicly available.

### Community `@softeria/ms-365-mcp-server` — Complete Email Tool List

**Source:** GitHub `Softeria/ms-365-mcp-server` (v0.96.0, 2026-05-04)
**npm:** `@softeria/ms-365-mcp-server`
**Tools defined declaratively in:** `src/endpoints.json`

#### Personal/Delegated Mode (default)

| Tool Name | Description |
|---|---|
| `list-mail-messages` | List messages in inbox; supports `$search` KQL syntax, `$filter`, `$top`, `$skip`, `$orderby`, pagination |
| `list-mail-folders` | List all mail folders |
| `list-mail-folder-messages` | List messages in a specific folder by folder ID |
| `get-mail-message` | Get a specific message by `messageId`; returns body (HTML format), properties, and relationships |
| `list-mail-attachments` | List attachments on a specific message |
| `get-mail-attachment` | Get a specific attachment |
| `send-mail` | Send a new email |
| `create-draft-email` | Create a draft email |
| `delete-mail-message` | Delete a message |
| `move-mail-message` | Move a message to a different folder |
| `add-mail-attachment` | Add an attachment to a draft |

#### Org/Work Mode (requires `--org-mode` flag)

| Tool Name | Description |
|---|---|
| `list-shared-mailbox-messages` | List messages in a shared mailbox |
| `list-shared-mailbox-folder-messages` | List messages in a folder of a shared mailbox |
| `get-shared-mailbox-message` | Get a specific message from a shared mailbox |
| `send-shared-mailbox-mail` | Send email from a shared mailbox |

#### Cross-Service Search

| Tool Name | Description |
|---|---|
| `search-query` | Microsoft Search API — searches across emails, files, sites, Teams messages |

---

## Q3: Authentication Model

### Official Agent365 Mail MCP

- **Auth type:** OAuth 2.0 delegated (user context)
- **Identity provider:** Microsoft Entra ID (Azure AD)
- **Required input:** Tenant ID (GUID) in the endpoint URL
- **Client handling:** Authentication is handled by the MCP client connecting to the remote endpoint; the user signs in via their Microsoft 365 account
- **License requirement:** Likely requires Microsoft 365 Copilot license (not confirmed via public docs)

### Community `@softeria/ms-365-mcp-server` — Three Authentication Methods

#### Method 1: Device Code Flow (Default — stdio mode)

```bash
# Step 1: Login
npx @softeria/ms-365-mcp-server --login
# Provides URL + code — open in browser and authenticate

# Step 2: Verify
npx @softeria/ms-365-mcp-server --verify-login
```

- Tokens cached in OS credential store (keytar) with file fallback
- Token cache persists across sessions

**Persistent token storage (recommended for agents):**
```bash
export MS365_MCP_TOKEN_CACHE_PATH="$HOME/.config/ms365-mcp/.token-cache.json"
export MS365_MCP_SELECTED_ACCOUNT_PATH="$HOME/.config/ms365-mcp/.selected-account.json"
```

#### Method 2: OAuth 2.0 Authorization Code Flow (HTTP mode only)

```bash
npx @softeria/ms-365-mcp-server --http 3000
```

- Advertises OAuth capabilities via MCP discovery protocol
- Provides endpoints at `/auth/authorize`, `/auth/token`, `/auth/metadata`
- All MCP requests require `Authorization: Bearer <token>`
- MCP clients handle the OAuth flow automatically

**Azure AD App Registration Setup:**
1. Azure Portal → Azure Active Directory → App registrations → New registration
2. Configure redirect URIs (Web platform):
   - `http://localhost:6274/oauth/callback`
   - `http://localhost:6274/oauth/callback/debug`
   - `http://localhost:3000/callback` (optional)
3. Set environment variables:

```bash
MS365_MCP_CLIENT_ID=your-azure-ad-app-client-id
MS365_MCP_CLIENT_SECRET=your-secret   # Optional for public apps
MS365_MCP_TENANT_ID=common
```

#### Method 3: Bring Your Own Token (BYOT — for agent pipelines)

```bash
MS365_MCP_OAUTH_TOKEN=your_oauth_token npx @softeria/ms-365-mcp-server
```

- Bypasses all interactive auth flows
- Ideal when the pipeline manages Microsoft OAuth tokens externally
- **No token refresh** — token lifecycle is fully external responsibility
- Best for server-to-server or daemon agent scenarios with pre-obtained tokens

---

## Q4: Configuration Format for AI Agents

### Official Agent365 Mail MCP — VS Code `mcp.json`

```json
{
  "servers": {
    "agent365-mailtools": {
      "type": "http",
      "url": "https://agent365.svc.cloud.microsoft/agents/tenants/${input:tenant_id}/servers/mcp_MailTools"
    }
  },
  "inputs": [
    {
      "id": "tenant_id",
      "type": "promptString",
      "description": "Microsoft Entra tenant ID (GUID)"
    }
  ]
}
```

### Community Server — Claude Desktop Config

**Personal account (MSA):**
```json
{
  "mcpServers": {
    "ms365": {
      "command": "npx",
      "args": ["-y", "@softeria/ms-365-mcp-server"]
    }
  }
}
```

**Work/school account:**
```json
{
  "mcpServers": {
    "ms365": {
      "command": "npx",
      "args": ["-y", "@softeria/ms-365-mcp-server", "--org-mode"]
    }
  }
}
```

**Email-only preset (reduces tool count from 90+ to ~10):**
```json
{
  "mcpServers": {
    "ms365": {
      "command": "npx",
      "args": ["-y", "@softeria/ms-365-mcp-server", "--preset", "mail"]
    }
  }
}
```

**Read-only mode (safe for agent pipelines):**
```json
{
  "mcpServers": {
    "ms365": {
      "command": "npx",
      "args": ["-y", "@softeria/ms-365-mcp-server", "--read-only", "--preset", "mail"]
    }
  }
}
```

**Token-efficient TOON output format (30-60% fewer tokens):**
```json
{
  "mcpServers": {
    "ms365": {
      "command": "npx",
      "args": ["-y", "@softeria/ms-365-mcp-server", "--preset", "mail", "--toon"]
    }
  }
}
```

**HTTP/Streamable mode for web-accessible agent:**
```bash
npx @softeria/ms-365-mcp-server --http 3000
# MCP endpoint: http://localhost:3000/mcp
```

### Community Server — Claude Code CLI

```bash
# Personal account
claude mcp add ms365 -- npx -y @softeria/ms-365-mcp-server

# Work/school account (macOS/Linux)
claude mcp add ms365 -- npx -y @softeria/ms-365-mcp-server --org-mode

# Work/school account (Windows)
claude mcp add ms365 -s user -- cmd /c "npx -y @softeria/ms-365-mcp-server --org-mode"
```

### Key Environment Variables

| Variable | Purpose |
|---|---|
| `MS365_MCP_CLIENT_ID` | Custom Azure app client ID (overrides built-in app) |
| `MS365_MCP_CLIENT_SECRET` | Client secret (optional for public apps) |
| `MS365_MCP_TENANT_ID` | Tenant ID (defaults to `common` for multi-tenant) |
| `MS365_MCP_OAUTH_TOKEN` | Pre-existing Bearer token (BYOT method) |
| `MS365_MCP_ORG_MODE=true` | Enable org/work mode (alternative to `--org-mode`) |
| `READ_ONLY=true` | Enable read-only mode |
| `ENABLED_TOOLS` | Regex filter for tools (e.g., `mail` to enable only mail tools) |
| `MS365_MCP_OUTPUT_FORMAT=toon` | Enable TOON format (30-60% token reduction) |
| `MS365_MCP_MAX_TOP=15` | Cap list result size (prevents oversized responses) |
| `MS365_MCP_BODY_FORMAT=html` | Return email bodies as HTML (default: text) |
| `MS365_MCP_TOKEN_CACHE_PATH` | Custom MSAL token cache file path |
| `MS365_MCP_SELECTED_ACCOUNT_PATH` | Custom selected account metadata file path |
| `MS365_MCP_CLOUD_TYPE=global\|china` | Cloud environment |
| `MS365_MCP_KEYVAULT_URL` | Azure Key Vault URL for secret management |

### CLI Commands

```bash
npx @softeria/ms-365-mcp-server --login             # Authenticate
npx @softeria/ms-365-mcp-server --logout            # Clear credentials
npx @softeria/ms-365-mcp-server --verify-login      # Verify auth status
npx @softeria/ms-365-mcp-server --list-accounts     # List logged-in accounts
npx @softeria/ms-365-mcp-server --list-permissions  # Show required Graph permissions
npx @softeria/ms-365-mcp-server --list-presets      # Show available tool presets
npx @softeria/ms-365-mcp-server --preset mail       # Start with mail tools only
npx @softeria/ms-365-mcp-server --read-only         # Read-only mode
npx @softeria/ms-365-mcp-server --discovery         # Dynamic tool loading (2 tools initially)
```

---

## Q5: Rate Limits and Constraints

### Microsoft Graph — Outlook Service Limits (Official)

Source: `https://learn.microsoft.com/en-us/graph/throttling-limits#outlook-service-limits`

**Per app+mailbox combination:**

| Limit | Scope |
|---|---|
| 10,000 API requests per 10-minute period | Per app ID + mailbox |
| 4 concurrent requests | Per app ID + mailbox |
| 150 MB upload (PATCH, POST, PUT) per 5-minute period | Per app ID + mailbox |

**Global Graph limit:** 130,000 requests per 10 seconds (all services combined)

**Important:** Exceeding the limit for one mailbox does **not** affect the app's ability to access other mailboxes. Throttled responses return HTTP `429 Too Many Requests` with a `Retry-After` header.

**Covered resources under Outlook limits:**
- `message`, `mailFolder`, `mailSearchFolder`, `messageRule`, `attachment`
- `event`, `calendar`, `calendarGroup`, `place`
- `contact`, `contactFolder`
- `person`, `outlookCategory`, `outlookTask`

### Community Server Constraints

| Constraint | Detail |
|---|---|
| `MS365_MCP_MAX_TOP=<n>` | Hard cap on `$top` list results (prevents token overflow) |
| Shared mailbox tools | Require `--org-mode` flag; delegated `Mail.Read.Shared` scope needed |
| Body format | Default is plain text; set `MS365_MCP_BODY_FORMAT=html` for HTML |
| Token cache | Tokens lost on npm update unless `MS365_MCP_TOKEN_CACHE_PATH` is set |
| HTTP mode | Requires OAuth token; device code flow not available in HTTP mode (unless `--enable-auth-tools`) |

### Message Size Limits (Microsoft Graph)

| Method | Max message size |
|---|---|
| REST API (JSON) | 4 MB |
| MIME upload | 150 MB |
| Batch requests | Up to 4 Outlook requests processed in parallel per batch |

---

## Q6: Sample Tool Calls

### `list-mail-messages` — Input Schema

All parameters are **optional** (no required parameters):

| Parameter | Required | Description |
|---|---|---|
| `includeHiddenMessages` | No | Include hidden messages |
| `top` | No | Max number of items to return (`$top`) |
| `skip` | No | Skip first n items (`$skip`) |
| `search` | No | KQL search query — **must be wrapped in double quotes**: `$search="from:john@example.com"` |
| `filter` | No | OData filter expression (`$filter`) |
| `count` | No | Include count of items |
| `orderby` | No | Sort order (`$orderby`) |
| `select` | No | Properties to return (`$select`) |
| `expand` | No | Expand related entities (`$expand`) |
| `fetchAllPages` | No | Auto-paginate through all result pages |
| `includeHeaders` | No | Include response headers (e.g., ETag) |
| `excludeResponse` | No | Return only success/failure, not full response body |

**KQL search syntax (for `search` parameter):**
```
$search="from:john@example.com"
$search="subject:meeting AND hasAttachments:true"
$search="body:urgent AND received>=2024-01-01"
$search="from:john AND importance:high"
$search="to:alice@company.com AND subject:invoice"
```

**KQL properties:** `from:`, `subject:`, `body:`, `to:`, `cc:`, `bcc:`, `attachment:`, `hasAttachments:`, `importance:`, `received:`, `sent:`

**Multi-account tool call:**
```json
{
  "tool": "list-mail-messages",
  "arguments": {
    "account": "work@company.com",
    "top": 20,
    "search": "subject:project update",
    "select": "id,subject,from,receivedDateTime,isRead"
  }
}
```

### `get-mail-message` — Input Schema

| Parameter | Required | Description |
|---|---|---|
| `messageId` | **Yes** | The message ID (path parameter) |
| `select` | No | Properties to return (`$select`) |
| `expand` | No | Expand related entities (e.g., `event` to get associated calendar event) |
| `fetchAllPages` | No | Auto-paginate |
| `includeHeaders` | No | Include response headers |
| `excludeResponse` | No | Return only success/failure |

**Notes:**
- Returns event message bodies in **HTML format** only
- Use `$expand=event` to get associated calendar event in attendee's calendar

**Sample call:**
```json
{
  "tool": "get-mail-message",
  "arguments": {
    "messageId": "AAMkAGI2TG93AAA=",
    "select": "id,subject,from,body,receivedDateTime,toRecipients,ccRecipients",
    "account": "work@company.com"
  }
}
```

### `list-mail-folder-messages` — Sample Call

```json
{
  "tool": "list-mail-folder-messages",
  "arguments": {
    "folderId": "Inbox",
    "top": 10,
    "filter": "isRead eq false",
    "orderby": "receivedDateTime desc"
  }
}
```

### Typical Agent Pipeline Pattern

```json
// Step 1: List recent unread emails
{
  "tool": "list-mail-messages",
  "arguments": {
    "top": 10,
    "filter": "isRead eq false",
    "orderby": "receivedDateTime desc",
    "select": "id,subject,from,receivedDateTime,bodyPreview"
  }
}

// Step 2: Get full body of a specific message
{
  "tool": "get-mail-message",
  "arguments": {
    "messageId": "<id from step 1>",
    "select": "id,subject,from,body,toRecipients,ccRecipients,attachments"
  }
}
```

---

## Q7: Known Working Examples and Tutorials

### Softeria ms-365-mcp-server Resources

| Resource | URL |
|---|---|
| GitHub repo | `https://github.com/Softeria/ms-365-mcp-server` |
| npm package | `https://www.npmjs.com/package/@softeria/ms-365-mcp-server` |
| GitHub Issues | `https://github.com/softeria/ms-365-mcp-server/issues` |
| GitHub Discussions | `https://github.com/softeria/ms-365-mcp-server/discussions` |
| Discord community | `https://discord.gg/WvGVNScrAZ` |
| Deployment guide | `https://github.com/Softeria/ms-365-mcp-server/blob/main/docs/deployment.md` |
| Azure Container Apps example | `examples/azure-container-apps/` (Bicep templates in repo) |

### Deployment Options

The `docs/deployment.md` covers: Docker, Azure Container Apps with managed identity, Azure App Service, Azure AD app registration, reverse proxy configuration.

### Admin/Daemon Companion Server

For **application permissions** (no user sign-in required) — daemon/service scenarios:
- `https://github.com/okapi-ca/ms-365-admin-mcp-server`
- Uses client credentials flow (client ID + secret, no user required)
- Covers security alerts, audit logs, service health, usage reports

---

## Q8: Microsoft Graph API Permissions

### Email Reading — Minimum Required Permissions

| Permission | Type | Admin Consent | Description |
|---|---|---|---|
| `Mail.Read` | Delegated | No | Read signed-in user's mailbox |
| `Mail.Read` | Application | **Yes** | Read all users' mailboxes |
| `Mail.ReadBasic` | Delegated | No | Read without body/attachments (headers, metadata only) |
| `Mail.ReadBasic.All` | Application | **Yes** | Read basic mail properties across all mailboxes |

### Read + Write Permissions

| Permission | Type | Admin Consent |
|---|---|---|
| `Mail.ReadWrite` | Delegated | No |
| `Mail.ReadWrite` | Application | **Yes** |
| `Mail.Read.Shared` | Delegated | No |
| `Mail.ReadWrite.Shared` | Delegated | No |
| `Mail-Advanced.ReadWrite` | Delegated | **Yes** |

### Supporting Permissions (commonly needed)

| Permission | Purpose |
|---|---|
| `MailboxSettings.Read` | Read mailbox settings (auto-reply, timezone, etc.) |
| `offline_access` | Required for refresh tokens in long-running agents |
| `User.Read` | Required for basic sign-in |
| `openid` | OpenID Connect — required for token identity |

### Permission GUIDs (for Azure AD app registration)

| Permission | GUID |
|---|---|
| `Mail.Read` (Delegated) | `570282fd-fa5c-430d-a7fd-fc8dc98a9dca` |
| `Mail.Read` (Application) | `810c84a8-4a9e-49e6-bf7d-12d183f40d01` |
| `Mail.ReadBasic` (Delegated) | `a4b8392a-d8d1-4954-a029-8e668a39a170` |
| `Mail.ReadBasic.All` (Application) | `6be147d2-6849-4788-ba4e-31ed09d0eb73` |

### Application Access Policy

Admins can configure application access policies to restrict an app's `Mail.Read` **application permission** to specific mailboxes only — preventing access to all mailboxes in the org.

Reference: `https://learn.microsoft.com/en-us/graph/auth-limit-mailbox-access`

---

## Q9: Alternative and Community MCP Servers

### Primary Alternative: `@softeria/ms-365-mcp-server`

| Property | Value |
|---|---|
| GitHub | `https://github.com/Softeria/ms-365-mcp-server` |
| npm | `@softeria/ms-365-mcp-server` |
| Stars | 675 (as of 2026-05-04) |
| Forks | 257 |
| Version | v0.96.0 |
| Contributors | 48 |
| License | MIT |
| Language | TypeScript (82.9%) |
| Transport | stdio (default) or Streamable HTTP (`--http`) |
| Tools | 200+ (90+ email/calendar/files/Teams) |

**Strengths for AI agent pipelines:**
- `--preset mail` reduces tool count to ~10 for focused email use
- `--read-only` prevents accidental writes
- `--toon` output format reduces token usage 30-60% for list operations
- `MS365_MCP_MAX_TOP` prevents runaway large responses
- `MS365_MCP_OAUTH_TOKEN` supports BYOT for external token management
- Multi-account support via `account` parameter in every tool call
- Dynamic tool discovery (`--discovery`) starts with only 2 tools
- Production deployment to Azure Container Apps with managed identity

### Admin/Daemon Companion

| Property | Value |
|---|---|
| GitHub | `https://github.com/okapi-ca/ms-365-admin-mcp-server` |
| Purpose | App-only permissions (no user sign-in) |
| Auth | Client credentials flow |
| Covers | Security alerts, audit logs, service health, usage reports |

### Other Notable Microsoft-Adjacent MCP Servers

| Name | Stars | Use Case |
|---|---|---|
| `floriscornel/teams-mcp` | 750 | Teams-focused MCP (channels, chats, members) — MIT |
| `imoon/mcp-teams` | ~1 | Teams messenger bridge |
| Official `mcp_M365Copilot` | N/A | Searches across emails + docs + sites (cloud-hosted) |

---

## Recommended Configuration for AI Agent Email Pipeline

For a **production AI agent pipeline** reading emails:

```json
{
  "mcpServers": {
    "ms365-mail": {
      "command": "npx",
      "args": [
        "-y",
        "@softeria/ms-365-mcp-server",
        "--preset", "mail",
        "--read-only",
        "--toon"
      ],
      "env": {
        "MS365_MCP_OAUTH_TOKEN": "${MS365_TOKEN}",
        "MS365_MCP_MAX_TOP": "20",
        "MS365_MCP_TOKEN_CACHE_PATH": "/persistent/.ms365-token-cache.json"
      }
    }
  }
}
```

Or for daemon mode using BYOT from an external token provider:
```bash
MS365_MCP_OAUTH_TOKEN="$TOKEN" \
MS365_MCP_MAX_TOP=20 \
MS365_MCP_OUTPUT_FORMAT=toon \
READ_ONLY=true \
npx @softeria/ms-365-mcp-server --preset mail
```

---

## Key References

| Resource | URL |
|---|---|
| Microsoft MCP server catalog (official) | `https://github.com/microsoft/mcp` |
| Softeria ms-365-mcp-server GitHub | `https://github.com/Softeria/ms-365-mcp-server` |
| Microsoft Graph Mail API docs | `https://learn.microsoft.com/en-us/graph/api/resources/mail-api-overview` |
| Microsoft Graph throttling limits | `https://learn.microsoft.com/en-us/graph/throttling-limits#outlook-service-limits` |
| Microsoft Graph auth (delegated) | `https://learn.microsoft.com/en-us/graph/auth-v2-user` |
| Microsoft Graph permissions reference | `https://learn.microsoft.com/en-us/graph/permissions-reference#mail-permissions` |
| Limit mailbox access (app permissions) | `https://learn.microsoft.com/en-us/graph/auth-limit-mailbox-access` |
| MCP Protocol spec | `https://modelcontextprotocol.io/` |
| Glama.ai tool schema viewer | `https://glama.ai/mcp/servers/Softeria/ms-365-mcp-server/schema` |
| list-mail-messages tool details | `https://glama.ai/mcp/servers/Softeria/ms-365-mcp-server/tools/list-mail-messages` |
| get-mail-message tool details | `https://glama.ai/mcp/servers/Softeria/ms-365-mcp-server/tools/get-mail-message` |

---

## Findings (In-Progress)

_Placeholders — updating progressively_

