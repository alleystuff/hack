# Outlook MCP + Microsoft Graph API — Research Report

**Date:** 2026-05-07
**Scope:** Outlook MCP servers, Microsoft Graph email API, Teams MCP server confirmation

---

## Table of Contents

1. [Outlook MCP Server Options](#1-outlook-mcp-server-options)
2. [Microsoft Graph API — Email Reading](#2-microsoft-graph-api--email-reading)
3. [Teams MCP Server Confirmation](#3-teams-mcp-server-confirmation)
4. [Recommendation Summary](#4-recommendation-summary)
5. [Authentication Flow Recommendation](#5-authentication-flow-recommendation)
6. [Python Code Snippet — Getting a Graph API Token](#6-python-code-snippet--getting-a-graph-api-token)
7. [Blockers and Limitations](#7-blockers-and-limitations)

---

## 1. Outlook MCP Server Options

### 1.1 `@softeria/ms-365-mcp-server` ✅ RECOMMENDED

| Attribute | Details |
|---|---|
| **Package** | `@softeria/ms-365-mcp-server` (npm) |
| **Install command** | `npx @softeria/ms-365-mcp-server` |
| **GitHub repo** | https://github.com/Softeria/ms-365-mcp-server |
| **Stars** | 685 ⭐ |
| **Forks** | 263 |
| **Latest version** | v0.99.1 (released 2026-05-06, actively maintained — 209 releases total) |
| **Language** | TypeScript (83.9%) / JavaScript (16%) |
| **License** | MIT |

#### Authentication Methods

Three supported methods:

1. **Device Code Flow (default)** — User is presented a code and URL in the terminal. They visit the URL in any browser, enter the code, and authenticate. The server polls for the token. Tokens are cached in the OS credential store (`keytar`) or file-based fallback.
   ```bash
   npx @softeria/ms-365-mcp-server --login
   ```

2. **OAuth Authorization Code Flow (HTTP mode only)** — For web-based clients via `--http` flag. Requires an Azure AD app registration with redirect URIs configured. Suitable for production deployments.

3. **Bring Your Own Token (BYOT)** — Pass a pre-existing Microsoft Graph access token directly:
   ```bash
   MS365_MCP_OAUTH_TOKEN=your_oauth_token npx @softeria/ms-365-mcp-server
   ```

#### Email/Outlook Tools Exposed

The server exposes **200+ tools** covering the full Graph API surface, defined declaratively in `src/endpoints.json`. Mail-specific tools are accessible via the `--preset mail` flag. Key email-related tools include:

| Tool | Description |
|---|---|
| `list-mail-messages` | List messages in mailbox (supports `$filter`, `$top`, `$select`) |
| `get-mail-message` | Get a specific message by ID |
| `list-shared-mailbox-messages` | List messages in a shared mailbox (org mode) |
| `search` (via Microsoft Search API) | Full-text search across mailbox |
| mark-read operations | Implied via PATCH operations on message `isRead` property |

Use `--preset mail` to load only mail tools, reducing context overhead:
```bash
npx @softeria/ms-365-mcp-server --preset mail
```

List exact permissions required for the mail preset:
```bash
npx @softeria/ms-365-mcp-server --preset mail --list-permissions
```

#### Required Graph API Permissions

Permissions are **requested dynamically** based on which tools are loaded. For mail operations, the expected required permissions are:

| Permission | Purpose | Type |
|---|---|---|
| `Mail.ReadBasic` | Read mail metadata (least privileged) | Delegated |
| `Mail.Read` | Read full message content including body | Delegated |
| `Mail.ReadWrite` | Read and modify messages | Delegated |
| `Mail.Read.Shared` | Read shared mailboxes | Delegated (org mode) |
| `Mail.Send.Shared` | Send from shared mailboxes | Delegated (org mode) |
| `offline_access` | Required for refresh token (long-lived sessions) | Delegated |

Run `--list-permissions` to get the exact set for your configuration.

#### Filtering Support

Yes — the server passes OData query parameters directly to the Graph API. The LLM can instruct the server to apply:
- `$filter=isRead eq false` — unread emails
- `$filter=from/emailAddress/address eq 'sender@example.com'` — specific sender
- `$filter=receivedDateTime ge 2026-05-01` — date range
- `$filter=contains(subject, 'keyword')` — subject keyword search
- `$top=50` — page size control
- `@odata.nextLink` — automatic pagination

#### Limitations

- **Node.js >= 20 recommended** (14+ may work with dependency warnings)
- **Organization mode** (Teams, SharePoint, shared mailboxes) requires `--org-mode` flag at startup — cannot be switched at runtime
- **Token persistence risk**: Default file-based token cache is relative to the installed package directory. Tokens can be lost on `npm` reinstall/update. Mitigation:
  ```bash
  export MS365_MCP_TOKEN_CACHE_PATH="$HOME/.config/ms365-mcp/.token-cache.json"
  export MS365_MCP_SELECTED_ACCOUNT_PATH="$HOME/.config/ms365-mcp/.selected-account.json"
  ```
- **HTTP mode only** supports the OAuth Authorization Code Flow; stdio mode uses device code
- **BYOT mode** does not handle token refresh — lifecycle management is the caller's responsibility
- **Email bodies** are returned as plain text by default; use `MS365_MCP_BODY_FORMAT=html` for HTML

---

### 1.2 Official Microsoft MCP Servers (`@modelcontextprotocol/*`)

**No official Microsoft 365 / Outlook MCP server exists.** The `modelcontextprotocol/servers` repository (85.2k ⭐) is maintained by Anthropic and contains only reference/demo servers:

- `@modelcontextprotocol/server-memory`
- `@modelcontextprotocol/server-filesystem`
- `@modelcontextprotocol/server-fetch`
- `@modelcontextprotocol/server-git`
- `@modelcontextprotocol/server-everything`

The third-party server list was retired from this repo in favor of the [MCP Registry](https://registry.modelcontextprotocol.io/). No `@modelcontextprotocol/server-outlook` exists.

---

### 1.3 `ms-365-admin-mcp-server` (Companion Server)

| Attribute | Details |
|---|---|
| **GitHub** | https://github.com/okapi-ca/ms-365-admin-mcp-server |
| **Purpose** | Admin/daemon scenarios using **application permissions** (client credentials flow) |
| **Covers** | Security alerts, audit logs, service health, usage reports |
| **Auth** | App-only (no user delegation) — requires tenant admin consent |
| **Mail access** | Not designed for user mail reading; focused on admin reporting |

This server complements `@softeria/ms-365-mcp-server` for headless/service scenarios but is **not suitable for delegated email reading**.

---

### 1.4 Community Landscape Summary

A search of `glama.ai/mcp/servers?q=outlook` and `mcp.so` returned no other significant Outlook-specific MCP servers with meaningful adoption. The Softeria server (`ms-365-mcp-server`) is the dominant community solution for Microsoft 365 MCP integration.

---

## 2. Microsoft Graph API — Email Reading

### 2.1 Core Endpoints

```http
# All messages in mailbox
GET /me/messages
GET /users/{id | userPrincipalName}/messages

# Messages in a specific folder (recommended for inbox)
GET /me/mailFolders/inbox/messages
GET /me/mailFolders/{folderId}/messages
```

### 2.2 Required Permissions

| Scenario | Least Privileged | Higher Privilege |
|---|---|---|
| Delegated (work/school) | `Mail.ReadBasic` | `Mail.Read`, `Mail.ReadWrite` |
| Delegated (personal MSA) | `Mail.ReadBasic` | `Mail.Read`, `Mail.ReadWrite` |
| Application (app-only) | `Mail.ReadBasic.All` | `Mail.Read`, `Mail.ReadWrite` |

**`Mail.ReadBasic` vs `Mail.Read`:**
- `Mail.ReadBasic` — Read mail metadata (sender, subject, date, recipients, read status) but **not the body content**
- `Mail.Read` — Read full message content including body, attachments metadata
- For an email-reading agent that needs to understand email content, `Mail.Read` is required

### 2.3 OData Query Parameters

| Parameter | Purpose | Example |
|---|---|---|
| `$select` | Return only specified fields | `$select=sender,subject,isRead,receivedDateTime,body` |
| `$filter` | Filter the result set | See examples below |
| `$orderby` | Sort results | `$orderby=receivedDateTime desc` |
| `$top` | Page size (1–1000, default 10) | `$top=50` |
| `$search` | Keyword search across mail | `$search="project kickoff"` |

**Important constraint:** When using `$filter` and `$orderby` together, properties in `$orderby` must also appear in `$filter` in the same order, and before any other filter properties.

### 2.4 `$filter` Examples for Email

```http
# Unread emails in inbox
GET /me/mailFolders/inbox/messages?$filter=isRead eq false

# Emails from a specific sender
GET /me/messages?$filter=from/emailAddress/address eq 'boss@example.com'

# Emails received in a date range
GET /me/mailFolders/inbox/messages?$filter=receivedDateTime ge 2026-05-01 and receivedDateTime lt 2026-05-08

# Unread emails from a specific sender (combined)
GET /me/mailFolders/inbox/messages?$filter=isRead eq false and from/emailAddress/address eq 'sender@example.com'

# Subject starts with a keyword
GET /me/messages?$filter=startswith(subject, 'Action Required')

# Unread emails ordered by date (note: isRead must appear in $filter when used in $orderby)
GET /me/mailFolders/inbox/messages?$filter=isRead eq false&$orderby=receivedDateTime desc&$top=20
```

### 2.5 Pagination with `@odata.nextLink`

The default page size is 10. Responses include `@odata.nextLink` when more pages exist:

```json
{
  "@odata.context": "https://graph.microsoft.com/v1.0/$metadata#users/.../messages",
  "@odata.nextLink": "https://graph.microsoft.com/v1.0/me/messages?$top=10&$skip=10",
  "value": [ ... ]
}
```

Follow `@odata.nextLink` directly (do **not** manually extract and manipulate `$skip`) until the response contains no `@odata.nextLink`.

### 2.6 Device Code Flow for CLI/Agent Authentication

Device code flow is the recommended approach for CLI scripts and agents where there is no web server to receive redirect URIs.

**Flow:**
1. App requests device code from `https://login.microsoftonline.com/{tenant}/oauth2/v2.0/devicecode`
2. Microsoft returns a `device_code`, `user_code`, and `verification_uri`
3. User navigates to `verification_uri` in any browser and enters `user_code`
4. App polls `https://login.microsoftonline.com/{tenant}/oauth2/v2.0/token` until the user completes sign-in
5. App receives `access_token` and `refresh_token`
6. App uses `access_token` in `Authorization: Bearer <token>` header for Graph API calls
7. When token expires, app uses `refresh_token` to get a new `access_token`

**Azure AD App Registration requirements for device code flow:**
- Register app in Azure Portal > Azure Active Directory > App registrations
- Set application type to "Public client/native (mobile & desktop)"
- Enable "Allow public client flows" under Authentication
- Add `https://login.microsoftonline.com/common/oauth2/nativeclient` as a redirect URI
- No client secret required (public client)

---

## 3. Teams MCP Server Confirmation

### `@floriscornel/teams-mcp`

| Attribute | Details |
|---|---|
| **Package** | `@floriscornel/teams-mcp` (npm) |
| **Install** | `npx @floriscornel/teams-mcp@latest` |
| **GitHub** | https://github.com/floriscornel/teams-mcp |
| **Stars** | 97 ⭐ |
| **Forks** | 46 |
| **Latest version** | v1.0.0 (released ~2 months ago from 2026-05-07) |
| **Language** | TypeScript (97.4%) |
| **License** | MIT |

### `send_channel_message` — Confirmed ✅

The `send_channel_message` tool is present and documented. It supports:
- `teamId` and `channelId` parameters
- `message` content (plain text or Markdown)
- `format` parameter: `"text"` (default) or `"markdown"` (converted to sanitized HTML)
- `importance` levels: `normal`, `high`, `urgent`
- `mentions` array for `@mention` support
- `imageUrl` for inline image attachments (URL or base64)

Example call:
```json
{
  "teamId": "team-id",
  "channelId": "channel-id",
  "message": "**Update:** Task completed successfully",
  "format": "markdown",
  "importance": "high",
  "mentions": [
    { "mention": "alice.smith", "userId": "00000000-0000-0000-0000-000000000000" }
  ]
}
```

### Authentication — Delegated Auth via Device Code Flow ✅

- Uses **OAuth 2.0 device code authentication** via Microsoft Graph (delegated permissions)
- Tokens are cached at `~/.teams-mcp-token-cache.json` and auto-refreshed
- Also supports `AUTH_TOKEN` environment variable for pre-issued Graph JWT tokens

Authenticate:
```bash
npx @floriscornel/teams-mcp@latest authenticate        # Full access
npx @floriscornel/teams-mcp@latest authenticate --read-only  # Read-only
```

### Required Permissions (Full Mode)

| Permission | Purpose |
|---|---|
| `User.Read` | Read user profile |
| `User.ReadBasic.All` | Read basic user info for search/mentions |
| `Team.ReadBasic.All` | Read team information |
| `Channel.ReadBasic.All` | Read channel information |
| `ChannelMessage.Read.All` | Read channel messages |
| `ChannelMessage.Send` | Send channel messages and replies |
| `ChannelMessage.ReadWrite` | Edit and delete channel messages |
| `Chat.ReadWrite` | Create/manage chats, send/edit/delete chat messages |
| `TeamMember.Read.All` | Read team members |
| `Files.ReadWrite.All` | File uploads to channels (SharePoint) and chats (OneDrive) |

### Recent Changes (past 3 months)

- **v1.0.0** — Stable release (2 months ago)
- Enhanced read-only mode with scoped permissions and improved documentation
- Added file upload support for large files (>4 MB) via resumable upload sessions
- Added Markdown formatting support with HTML sanitization (XSS prevention)
- Added `fetchAll` pagination via `@odata.nextLink` for chat message history
- Added `get_my_mentions` and advanced KQL search via Microsoft Search API
- LLM-friendly HTML-to-Markdown conversion for retrieved messages

### All Available Tools (26 total)

**Read tools (16):** `auth_status`, `get_current_user`, `search_users`, `get_user`, `list_teams`, `list_channels`, `get_channel_messages`, `get_channel_message_replies`, `list_team_members`, `search_users_for_mentions`, `download_message_hosted_content`, `list_chats`, `get_chat_messages`, `download_chat_hosted_content`, `search_messages`, `get_my_mentions`

**Write tools (10):** `send_channel_message`, `reply_to_channel_message`, `update_channel_message`, `delete_channel_message`, `send_file_to_channel`, `send_chat_message`, `create_chat`, `update_chat_message`, `delete_chat_message`, `send_file_to_chat`

---

## 4. Recommendation Summary

### Best Outlook MCP Server: `@softeria/ms-365-mcp-server`

**Justification:**
- Only mature, actively maintained MCP server for Outlook/M365 email (685 stars, 209 releases)
- 200+ tools covering the full Graph API surface — not limited to email
- Supports device code flow natively (ideal for CLI/agent use cases)
- Supports BYOT (Bring Your Own Token) for integration into existing auth pipelines
- Read-only mode for safe, non-destructive operations
- Mail preset (`--preset mail`) for lean deployments
- Dynamic tool discovery (`--discovery`) to reduce LLM context overhead
- Multi-account support for multi-tenant scenarios
- Active community (48 contributors, Discord support)

**Claude Desktop configuration:**
```json
{
  "mcpServers": {
    "ms365-mail": {
      "command": "npx",
      "args": ["-y", "@softeria/ms-365-mcp-server", "--preset", "mail", "--read-only"]
    }
  }
}
```

### Complete List of Required Graph API Permissions

For an email-reading agent using `@softeria/ms-365-mcp-server` with the `mail` preset:

| Permission | Why Needed |
|---|---|
| `Mail.ReadBasic` | Minimum — read mail metadata |
| `Mail.Read` | Read full email body content |
| `offline_access` | Refresh token for long-lived sessions |
| `User.Read` | Get signed-in user profile |

For shared mailbox access (org mode):
- Add `Mail.Read.Shared`

For write operations (mark as read, move, delete):
- Upgrade to `Mail.ReadWrite`

---

## 5. Authentication Flow Recommendation

**Recommended: Device Code Flow via MSAL Python**

Device code flow is ideal for CLI scripts and agents because:
- No web server or redirect URI needed
- User authenticates in a separate browser session (can be on any device)
- Tokens are cached and auto-refreshed
- Supported natively by MSAL (Microsoft Authentication Library)

**Steps:**
1. Register an Azure AD app as a **public client** (no client secret)
2. Enable "Allow public client flows" in Authentication settings
3. Request `Mail.Read offline_access User.Read` scopes
4. Use MSAL's `DeviceCodeCredential` or `PublicClientApplication` with device flow
5. Cache the token for subsequent runs

---

## 6. Python Code Snippet — Getting a Graph API Token

### Option A: Using MSAL (direct device code flow)

```python
"""
Get a Microsoft Graph API access token using device code flow.
Requires: pip install msal requests
Azure AD App Registration: Public client, no client secret.
"""

import json
import os
import msal
import requests

# Azure AD app registration values
CLIENT_ID = os.environ.get("AZURE_CLIENT_ID", "YOUR_APP_CLIENT_ID")
TENANT_ID = os.environ.get("AZURE_TENANT_ID", "common")  # Use "common" for multi-tenant
SCOPES = ["Mail.Read", "User.Read", "offline_access"]

AUTHORITY = f"https://login.microsoftonline.com/{TENANT_ID}"
TOKEN_CACHE_FILE = os.path.expanduser("~/.graph-token-cache.json")


def load_cache() -> msal.SerializableTokenCache:
    cache = msal.SerializableTokenCache()
    if os.path.exists(TOKEN_CACHE_FILE):
        with open(TOKEN_CACHE_FILE, "r") as f:
            cache.deserialize(f.read())
    return cache


def save_cache(cache: msal.SerializableTokenCache) -> None:
    if cache.has_state_changed:
        with open(TOKEN_CACHE_FILE, "w") as f:
            f.write(cache.serialize())
        os.chmod(TOKEN_CACHE_FILE, 0o600)  # Restrict file permissions


def get_access_token() -> str:
    cache = load_cache()
    app = msal.PublicClientApplication(
        client_id=CLIENT_ID,
        authority=AUTHORITY,
        token_cache=cache,
    )

    # Try to get a token silently from cache first
    accounts = app.get_accounts()
    if accounts:
        result = app.acquire_token_silent(SCOPES, account=accounts[0])
        if result and "access_token" in result:
            save_cache(cache)
            return result["access_token"]

    # Fall back to device code flow
    flow = app.initiate_device_flow(scopes=SCOPES)
    if "user_code" not in flow:
        raise ValueError(f"Failed to create device flow: {flow.get('error_description')}")

    print(f"\n{flow['message']}\n")  # Prints: "To sign in, visit https://... and enter code XXXXX"
    result = app.acquire_token_by_device_flow(flow)  # Blocks until user authenticates

    if "access_token" not in result:
        raise ValueError(f"Auth failed: {result.get('error_description')}")

    save_cache(cache)
    return result["access_token"]


def list_unread_emails(access_token: str, top: int = 20) -> list[dict]:
    """List unread emails from the inbox."""
    headers = {"Authorization": f"Bearer {access_token}"}
    params = {
        "$filter": "isRead eq false",
        "$select": "id,subject,from,receivedDateTime,isRead,bodyPreview",
        "$orderby": "receivedDateTime desc",
        "$top": top,
    }
    url = "https://graph.microsoft.com/v1.0/me/mailFolders/inbox/messages"
    emails = []

    while url:
        response = requests.get(url, headers=headers, params=params if emails == [] else None)
        response.raise_for_status()
        data = response.json()
        emails.extend(data.get("value", []))
        url = data.get("@odata.nextLink")  # Follow pagination

    return emails


if __name__ == "__main__":
    token = get_access_token()
    emails = list_unread_emails(token)
    print(f"\nFound {len(emails)} unread email(s):\n")
    for email in emails:
        sender = email["from"]["emailAddress"]["address"]
        print(f"  [{email['receivedDateTime'][:10]}] {sender}: {email['subject']}")
```

### Option B: Using `azure-identity` (DefaultAzureCredential / device code)

```python
"""
Alternative using azure-identity for Graph API token acquisition.
Requires: pip install azure-identity requests
"""

import os
import requests
from azure.identity import DeviceCodeCredential

CLIENT_ID = os.environ.get("AZURE_CLIENT_ID", "YOUR_APP_CLIENT_ID")
TENANT_ID = os.environ.get("AZURE_TENANT_ID", "common")
SCOPES = ["https://graph.microsoft.com/Mail.Read"]

credential = DeviceCodeCredential(client_id=CLIENT_ID, tenant_id=TENANT_ID)

def get_graph_token() -> str:
    token = credential.get_token(*SCOPES)
    return token.token

token = get_graph_token()
response = requests.get(
    "https://graph.microsoft.com/v1.0/me/mailFolders/inbox/messages",
    headers={"Authorization": f"Bearer {token}"},
    params={"$filter": "isRead eq false", "$top": "10", "$select": "subject,from,receivedDateTime"},
)
response.raise_for_status()
for msg in response.json().get("value", []):
    print(msg["subject"], "-", msg["from"]["emailAddress"]["address"])
```

**Install dependencies:**
```bash
pip install msal requests azure-identity
```

---

## 7. Blockers and Limitations

### Potential Blockers

| Blocker | Severity | Mitigation |
|---|---|---|
| **Azure AD app registration required** | Medium | Register a free public client app in Azure Portal; takes ~5 minutes |
| **Organizational tenant — admin consent** | Medium–High | `Mail.Read` is a low-risk scope but some tenants require admin pre-approval |
| **Device code flow requires browser** | Low | User opens URL in any browser (can be on different machine) |
| **Token cache persistence** | Low | Use custom `MS365_MCP_TOKEN_CACHE_PATH` to avoid loss on npm reinstall |
| **BYOT token expiry (1 hour)** | Medium | Token refresh must be handled externally; use device code flow for simplicity |

### Known Limitations of `@softeria/ms-365-mcp-server`

- `--org-mode` cannot be added after server starts — must be set at launch
- No official Microsoft-published MCP server for Outlook; Softeria is community-maintained
- File-based token storage (`0600` permissions) — secure for single-user machines, not for shared systems
- Email bodies returned as plain text by default; HTML mode is opt-in via env var
- `$top` is capped server-side via `MS365_MCP_MAX_TOP` env var (configurable)
- TOON output format is experimental (may change in future releases)
- LLM token usage can be high with 200+ tools loaded — use `--preset mail` or `--discovery`

### Known Limitations of `@floriscornel/teams-mcp`

- Lower star count (97 vs 685) — less battle-tested than ms-365-mcp-server
- Write tools disabled in read-only mode (use full mode for `send_channel_message`)
- File uploads >4 MB require resumable session support (handled automatically)
- Token stored at `~/.teams-mcp-token-cache.json` — not suitable for multi-user environments
- No support for Microsoft 365 China (21Vianet) cloud (unlike ms-365-mcp-server)

### No Official Microsoft Outlook MCP Server

Microsoft has not published an official MCP server for Outlook or Microsoft 365 email. All options are community-maintained. The Softeria server is the de facto standard with the best adoption metrics.

---

*Research compiled 2026-05-07. Sources: GitHub repos (Softeria/ms-365-mcp-server, floriscornel/teams-mcp, modelcontextprotocol/servers), Microsoft Learn (Graph API docs, auth-v2-user, filter-query-parameter), mcp.so, glama.ai.*
