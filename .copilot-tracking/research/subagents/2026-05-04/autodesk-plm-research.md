# Autodesk PLM and Design Data APIs Research

**Date:** 2026-05-04  
**Status:** Complete  
**Topic:** Autodesk PLM and Design Data APIs for AI Agent Pipeline (email-to-PLM-record update)

---

## Table of Contents

1. [Autodesk Fusion Manage (formerly PLM 360)](#1-autodesk-fusion-manage-formerly-plm-360)
2. [Autodesk Platform Services (APS) Overview](#2-autodesk-platform-services-aps-overview)
3. [Authentication — OAuth 2-Legged vs 3-Legged](#3-authentication--oauth-2-legged-vs-3-legged)
4. [Fusion Manage REST API — Key Endpoints](#4-fusion-manage-rest-api--key-endpoints)
5. [Data Model in Fusion Manage](#5-data-model-in-fusion-manage)
6. [Searching PLM Records (by Part Number, ECN, Project Name)](#6-searching-plm-records-by-part-number-ecn-project-name)
7. [Updating PLM Records via REST](#7-updating-plm-records-via-rest)
8. [Sample Dataset / Demo Tenant](#8-sample-dataset--demo-tenant)
9. [GitHub Samples and Reference Code](#9-github-samples-and-reference-code)
10. [AI Agent — Identifying PLM Records from Email](#10-ai-agent--identifying-plm-records-from-email)
11. [Python and Node.js SDKs](#11-python-and-nodejs-sdks)
12. [Mock / Sample Data Structure for Demo](#12-mock--sample-data-structure-for-demo)
13. [Autodesk Vault (On-Premise) API vs Fusion Manage](#13-autodesk-vault-on-premise-api-vs-fusion-manage)
14. [MCP Servers for Autodesk APIs](#14-mcp-servers-for-autodesk-apis)
15. [ACC (Autodesk Construction Cloud) Data Connector](#15-acc-autodesk-construction-cloud-data-connector)
16. [Recommended Architecture for the AI Agent Pipeline](#16-recommended-architecture-for-the-ai-agent-pipeline)

---

## 1. Autodesk Fusion Manage (formerly PLM 360)

### What Is It?

Autodesk Fusion Manage is a **cloud-based Product Lifecycle Management (PLM)** platform that connects people, processes, and data across the product development lifecycle. It was previously known as **Autodesk PLM 360** and is part of the broader **Autodesk Fusion industry cloud** for manufacturing.

Key capabilities:
- Manages product records, change orders, BOMs (Bills of Materials), quality processes, supplier collaboration
- Configurable workspaces (equivalent to "modules") such as: Items, Change Orders, Change Requests, ECNs, Projects, Quality Records
- Workflow engine for lifecycle state transitions (Draft → In Review → Released → Obsolete)
- Role-based access control
- Integration points with ERP, PDM, and CRM via REST API

### Product vs Platform Context

| Product | Description |
|---|---|
| Autodesk Fusion Manage | Cloud PLM (the PLM-specific product) |
| Autodesk Fusion Operations | Production floor tracking / MES |
| Autodesk Platform Services (APS) | Developer API layer for all Autodesk cloud products |
| Autodesk Fusion (CAD) | CAD/CAM/CAE tool — separate from Fusion Manage |

**Official docs home:** https://help.autodesk.com/view/PLM/ENU/?guid=FLC_RestAPI_Read_Me_First_html

---

## 2. Autodesk Platform Services (APS) Overview

**APS** (formerly **Autodesk Forge**) is the developer platform for all Autodesk cloud APIs. It provides unified authentication (OAuth 2.0) and hosts documentation/SDK resources.

**Portal:** https://aps.autodesk.com/

### Key APIs Relevant to This Pipeline

| API | Purpose | Relevance |
|---|---|---|
| **Fusion Manage API** | PLM record CRUD, workflow, BOM | **Primary** |
| **Data Management API** | File storage, projects, hubs | Secondary — design files |
| **Manufacturing Data Model API v3** | GraphQL API for Fusion BOM/parts data | Secondary — granular part data |
| **Model Derivative API** | Translate 3D models to viewable formats | Optional — design visualization |
| **Webhooks API** | Push notifications on record/file changes | For real-time triggers |
| **ACC / Forma APIs** | Construction project data | Alternative data source |
| **Vault Data API** | On-premise Vault (PDM) integration | Alternative for on-prem |

**API catalog:** https://aps.autodesk.com/developer/documentation (browse by industry)

### Manufacturing Data Model API (GraphQL)

The Manufacturing Data Model API v3 is now GA and provides a **GraphQL endpoint** to access granular Fusion part/BOM data:
- Access BOMs, integrate ERP, augment the data model with custom properties
- More granular than file-based Data Management API
- Documentation: https://aps.autodesk.com/en/docs/mfgdataapi/v3/reference/graphqlendpoint/

---

## 3. Authentication — OAuth 2-Legged vs 3-Legged

All Autodesk APS APIs use **OAuth 2.0** via this endpoint:

```
POST https://developer.api.autodesk.com/authentication/v2/token
```

### 2-Legged OAuth (Client Credentials)

Used for **server-to-server** automation without user interaction. Recommended for AI agent pipelines.

**Flow:**
1. Register an APS app at https://aps.autodesk.com/myapps/ — get `CLIENT_ID` and `CLIENT_SECRET`
2. Encode `CLIENT_ID:CLIENT_SECRET` as Base64
3. POST to token endpoint:

```bash
curl -X POST 'https://developer.api.autodesk.com/authentication/v2/token' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -H 'Accept: application/json' \
  -H 'Authorization: Basic <BASE64_ENCODED_ID_SECRET>' \
  -d 'grant_type=client_credentials' \
  -d 'scope=data:read data:write'
```

**Response:**
```json
{
  "token_type": "Bearer",
  "expires_in": 1799,
  "access_token": "eyJhbGci..."
}
```

**Fusion Manage additional requirement for 2-legged:**
- The APS app must be added to Fusion Manage's **Allowed List** via: Administration → General Settings → Security Settings
- Every API call must include an `X-user-id` header (a valid user email to impersonate)

### 3-Legged OAuth (Authorization Code)

For user-interactive flows. Required for some Fusion Manage operations where user context matters.
- Tutorial: https://aps.autodesk.com/en/docs/oauth/v2/tutorials/get-3-legged-token

### Scopes

| Scope | Description |
|---|---|
| `data:read` | Read all end user data |
| `data:write` | Create, update, delete data |
| `data:create` | Create data |
| `data:search` | Search data |
| `account:read` | Read product/service account data |
| `account:write` | Update product/service account data |
| `openid` | Required for id_token generation |

**For an AI agent reading/writing PLM records:** Use `data:read data:write` scopes.

---

## 4. Fusion Manage REST API — Key Endpoints

**Base URL pattern:** `https://{tenant_name}.autodeskplm360.net/api/v3/`

All requests require:
```
Authorization: Bearer {access_token}
X-user-id: {user_email}        # For 2-legged auth
Content-Type: application/json
Accept: application/json
```

### List All Workspaces

```bash
GET https://{tenant}.autodeskplm360.net/api/v3/workspaces?unlimited=true
```

**Example response structure:**
```json
{
  "items": [
    {
      "id": 9,
      "title": "Change Orders",
      "type": "/api/v3/workspace-types/7",
      "__self__": "/api/v3/workspaces/9"
    },
    {
      "id": 1,
      "title": "Items",
      "__self__": "/api/v3/workspaces/1"
    }
  ]
}
```

### List Items in a Workspace

```bash
GET https://{tenant}.autodeskplm360.net/api/v3/workspaces/{workspaceId}/items
```

With `If-Modified-Since` header for incremental polling:
```bash
GET https://{tenant}.autodeskplm360.net/api/v3/workspaces/1/items
If-Modified-Since: Wed, 29 May 2024 07:00:00 GMT
```

### Get a Single Item (Full Details)

```bash
GET https://{tenant}.autodeskplm360.net/api/v3/workspaces/{workspaceId}/items/{itemId}
```

For plain-text paragraph fields (avoids HTML encoding):
```bash
GET https://{tenant}.autodeskplm360.net/api/v3/workspaces/1/items/1001?paragraph-content=text/plain
```

### Bulk Item Details (Reduces API Calls)

```bash
GET https://{tenant}.autodeskplm360.net/api/v3/workspaces/9/items/
Accept: application/vnd.autodesk.plm.items.bulk+json
```

Returns full item details for all items (up to 100 per page).

### Create a New Item

```bash
POST https://{tenant}.autodeskplm360.net/api/v3/workspaces/{workspaceId}/items
```

### Update an Item's Fields (PATCH)

```bash
PATCH https://{tenant}.autodeskplm360.net/api/v3/workspaces/{workspaceId}/items/{itemId}
```

Partial updates are supported — only provide the fields to change.

### Workflow Transitions (Change State)

```bash
GET  https://{tenant}.autodeskplm360.net/api/v3/workspaces/{workspaceId}/items/{itemId}/workflows/{workflowId}
GET  https://{tenant}.autodeskplm360.net/api/v3/workspaces/{workspaceId}/items/{itemId}/workflows/{workflowId}/history
POST https://{tenant}.autodeskplm360.net/api/v3/workspaces/{workspaceId}/items/{itemId}/workflows/{workflowId}/transitions
```

### BOM Endpoints

```bash
GET https://{tenant}.autodeskplm360.net/api/v3/workspaces/{workspaceId}/items/{itemId}/bom
GET https://{tenant}.autodeskplm360.net/api/v3/workspaces/{workspaceId}/items/{itemId}/versions
GET https://{tenant}.autodeskplm360.net/api/v3/workspaces/{workspaceId}/items/{itemId}/versions/{versionId}
```

### Full Resource Map

```
/api/v3/workspaces
/api/v3/workspaces/{workspaceId}
/api/v3/workspaces/{workspaceId}/items
/api/v3/workspaces/{workspaceId}/items/{itemId}
/api/v3/workspaces/{workspaceId}/items/{itemId}/bom
/api/v3/workspaces/{workspaceId}/items/{itemId}/versions
/api/v3/workspaces/{workspaceId}/items/{itemId}/versions/{versionId}
/api/v3/workspaces/{workspaceId}/items/{itemId}/workflows/{workflowId}
/api/v3/workspaces/{workspaceId}/items/{itemId}/workflows/{workflowId}/history
/api/v3/workspaces/{workspaceId}/items/{itemId}/workflows/{workflowId}/transitions
/api/v3/workspaces/{workspaceId}/workflows/{workflowId}/states
/api/v3/workspaces/{workspaceId}/workflows/{workflowId}/states/{stateId}
/api/v3/search-results
```

**Advanced endpoints:**
```
/api/v3/workspaces/{workspaceId}/items/{itemId}/views/{viewId}/sections
/api/v3/workspaces/{workspaceId}/items/{itemId}/views/{viewId}/fields/{fieldId}
/api/v3/workspaces/{workspaceId}/items/{itemId}/audit
/api/v3/workspaces/{workspaceId}/items/{itemId}/owners
```

**Official endpoint listing:** https://help.autodesk.com/view/PLM/ENU/?guid=FLC_RestAPI_Resource_Endpoints_html

---

## 5. Data Model in Fusion Manage

### Conceptual Hierarchy

```
Tenant (e.g., "mycompany")
  └── Workspace (e.g., "Change Orders", "Items", "Quality Records")
        └── Item (a single record, e.g., ECN-00234)
              └── View (layout of sections for a particular user role)
                    └── Section (logical grouping of fields, e.g., "General Info")
                          └── Field (a single data element, e.g., "Part Number", "Status")
              └── Workflow (state machine, e.g., "Draft → In Review → Released")
                    └── State (e.g., "Draft", "Released")
                    └── Transition (e.g., "Submit for Review")
              └── BOM (Bill of Materials — child parts/assemblies)
              └── Versions (revision history)
```

### Item JSON Structure (bulk endpoint response excerpt)

```json
{
  "__self__": "/api/v3/workspaces/9/items/134615",
  "urn": "urn:adsk.plm:tenant.workspace.item:TENANTNAME.9.134615",
  "workspace": {
    "link": "/api/v3/workspaces/9",
    "title": "Change Orders"
  },
  "title": "Some Generic Item - ECN-000001",
  "deleted": false,
  "latestRelease": true,
  "workingVersion": true,
  "itemLocked": true,
  "workflowReference": false,
  "currentState": {
    "link": "/api/v3/workspaces/9/workflows/1/states/33",
    "title": "Released"
  },
  "lifecycle": {
    "title": "Unreleased"
  },
  "bom": { "link": "/api/v3/workspaces/9/items/134615/bom" },
  "sectionsLink": "/api/v3/workspaces/9/items/134615/views/1/sections",
  "sections": [
    {
      "link": "/api/v3/workspaces/9/items/134615/views/1/sections/466",
      "title": "General Information",
      "fields": [
        {
          "__self__": "/api/v3/workspaces/9/items/134615/views/1/fields/PART_NUMBER",
          "urn": "urn:adsk.plm:tenant.workspace.item.view.field:TENANTNAME.9.134615.1.PART_NUMBER",
          "title": "Part Number",
          "type": {
            "title": "Single Line Text"
          },
          "value": "PN-1042-A",
          "defaultValue": ""
        },
        {
          "__self__": "/api/v3/workspaces/9/items/134615/views/1/fields/STATUS",
          "title": "Status",
          "type": { "title": "Single Line Text" },
          "value": "In Review"
        }
      ]
    }
  ]
}
```

### Field Types Supported

- Single Line Text
- Paragraph (rich text / plain text)
- Number
- Date
- Checkbox
- Dropdown / Single-Select
- Multi-Select
- User Reference
- Item Reference (link to another PLM item)

### URN Pattern

All objects in Fusion Manage use **URNs** for identification:
```
urn:adsk.plm:tenant.workspace.item:TENANTNAME.{workspaceId}.{itemId}
urn:adsk.plm:tenant.workspace.item.view.field:TENANTNAME.{wsId}.{itemId}.{viewId}.{fieldName}
```

---

## 6. Searching PLM Records (by Part Number, ECN, Project Name)

### Global Full-Text Search

```bash
GET https://{tenant}.autodeskplm360.net/api/v3/search-results?limit=100&offset=0&page=1&query=ECN-00234&revision=1
```

**Revision parameter options:**
- `revision=1` — Latest version only (including Unreleased), plus non-revision-controlled items
- `revision=2` — All revisions
- `revision=3` — Working versions (including Unreleased) + non-revision-controlled

**With bulk item details in response:**
```bash
GET https://{tenant}.autodeskplm360.net/api/v3/search-results?query=PN-1042&revision=1
Accept: application/vnd.autodesk.plm.items.bulk+json
```

**Response:**
```json
{
  "__self__": "/api/v3/search-results",
  "offset": 0,
  "limit": 100,
  "totalCount": 3,
  "items": [ ... ]
}
```

> Note: Response is GZIP compressed — set `Accept-Encoding: gzip` in client.

**Search docs:** https://help.autodesk.com/view/PLM/ENU/?guid=FLC_RestAPI_Advanced_Functionalities_search_endpoints_html

### Workspace-Level Grid Search (Filtered)

```bash
GET https://{tenant}.autodeskplm360.net/api/v3/workspaces/{workspaceId}/views/{viewId}/grid?filter=...
```

### Incremental Search (Poll for Changes)

```bash
GET https://{tenant}.autodeskplm360.net/api/v3/workspaces/1/items/
If-Modified-Since: Wed, 01 Jan 2025 00:00:00 GMT
```

Returns only items modified after the given datetime — ideal for the agent to detect PLM records that need attention.

---

## 7. Updating PLM Records via REST

### Pattern: Read → Identify Field → Patch

**Step 1:** Search for the record by part number or ECN number:
```bash
GET /api/v3/search-results?query=ECN-00234&revision=1
```

**Step 2:** Retrieve the item details to see current field values and their `__self__` URIs:
```bash
GET /api/v3/workspaces/{wsId}/items/{itemId}
```

**Step 3:** Update specific fields via PATCH (partial update support):
```bash
PATCH https://{tenant}.autodeskplm360.net/api/v3/workspaces/{wsId}/items/{itemId}
Content-Type: application/json

{
  "sections": [
    {
      "link": "/api/v3/workspaces/9/items/134615/views/1/sections/466",
      "fields": [
        {
          "__self__": "/api/v3/workspaces/9/items/134615/views/1/fields/STATUS",
          "value": "Approved"
        },
        {
          "__self__": "/api/v3/workspaces/9/items/134615/views/1/fields/NOTES",
          "value": "Updated via email agent on 2026-05-04"
        }
      ]
    }
  ]
}
```

**Step 4:** Trigger a workflow transition if needed:
```bash
POST https://{tenant}.autodeskplm360.net/api/v3/workspaces/{wsId}/items/{itemId}/workflows/{workflowId}/transitions
Content-Type: application/json

{
  "transition": {
    "link": "/api/v3/workspaces/9/workflows/1/transitions/42"
  }
}
```

**Partial update docs:** https://help.autodesk.com/view/PLM/ENU/?guid=FLC_RestAPI_Advanced_Functionalities_Item_details_endpoints_items_partial_updates_html

---

## 8. Sample Dataset / Demo Tenant

### Getting Access

1. **APS Free Tier** — Sign up at https://aps.autodesk.com/ for API access. The free tier includes access to APS APIs, but Fusion Manage requires a separate subscription (not part of the free tier).

2. **Fusion Manage Trial** — Autodesk offers trials for Fusion 360 + Manage. Check: https://www.autodesk.com/products/fusion-360/plm (requires Autodesk account/login).

3. **Autodesk Education** — Academic licenses available; may include Fusion Manage.

### No Public Mock Tenant

There is **no publicly available mock/demo tenant** for Fusion Manage API. Access requires:
- A paid Fusion Manage subscription, OR
- An Autodesk Developer account with access to a tenant (request via Autodesk Developer Network)

### Alternative: Autodesk ADN (Developer Network)

- https://aps.autodesk.com/adn
- ADN members can get access to sandbox tenants

### Generating Mock Data for a Demo

Since live API access is gated, the best approach for demos is to **mock the Fusion Manage REST API** locally. See Section 12 for mock data structure.

---

## 9. GitHub Samples and Reference Code

### Official APS GitHub Organization

**URL:** https://github.com/autodesk-platform-services (223 repositories)

#### Directly Relevant Repositories

| Repository | Language | Description |
|---|---|---|
| [aps-sdk-node](https://github.com/autodesk-platform-services/aps-sdk-node) | TypeScript | Official APS SDK for Node.js |
| [aps-sdk-net](https://github.com/autodesk-platform-services/aps-sdk-net) | C# | Official APS SDK for .NET |
| [aps-sdk-openapi](https://github.com/autodesk-platform-services/aps-sdk-openapi) | YAML | OpenAPI specs for all APS APIs |
| [vault-data-api-samples](https://github.com/autodesk-platform-services/vault-data-api-samples) | JS/C# | Vault on-premise REST API samples |
| [skills](https://github.com/autodesk-platform-services/skills) | Markdown | AI agent skills incl. MCP server generator |
| [aps-tutorial-postman](https://github.com/autodesk-platform-services/aps-tutorial-postman) | TypeScript | Postman collections for APS APIs |
| [llmstxt](https://github.com/autodesk-platform-services/llmstxt) | — | LLMs.txt files for APS docs (AI-optimized) |
| [aps-data-explorer](https://github.com/autodesk-platform-services/aps-data-explorer) | HTML | GraphQL query explorer for APS APIs |
| [aps-dx-tools-python](https://github.com/autodesk-platform-services/aps-dx-tools-python) | Python | Data Exchange CLI tool (Python) |

#### Note on Fusion Manage Specific Samples

A search for dedicated Fusion Manage REST API sample repos returned **0 results** on GitHub. The Fusion Manage API documentation samples are embedded in the help docs, not in separate repos.

### Autodesk Official Tutorials

- https://aps.autodesk.com/tutorials (Node.js and .NET guided tutorials)
- https://get-started.aps.autodesk.com/ — Getting started portal

### LLMs.txt for AI Agents

Autodesk publishes `llms.txt` files to make APS documentation AI-optimized:
- https://github.com/autodesk-platform-services/llmstxt (47 stars)
- These can be fed directly to LLM context for accurate API calls

---

## 10. AI Agent — Identifying PLM Records from Email

### Recommended Identification Strategy

When an AI agent processes an inbound email about a PLM record, it should use the following multi-signal extraction approach:

#### Signal 1 — Part Number (PN)
- Pattern: alphanumeric codes like `PN-1042-A`, `PART-10423`, `REV-B`
- Regex: `\b[A-Z]{2,4}-\d{3,6}(-[A-Z0-9]+)?\b`
- PLM Search: `GET /api/v3/search-results?query=PN-1042-A`

#### Signal 2 — ECN / ECO Number (Engineering Change Notice/Order)
- Pattern: `ECN-00234`, `ECO-2024-042`, `CR-001`
- Regex: `\b(ECN|ECO|CR|DCO)-\d{3,6}\b`
- PLM Search: `GET /api/v3/search-results?query=ECN-00234`

#### Signal 3 — Project Name
- Extracted via NLP/LLM
- Search: `GET /api/v3/search-results?query={project+name}`

#### Signal 4 — Item Title / Descriptor
- Extracted from email subject or body
- Search against item descriptors

### Disambiguation Logic

```
1. Extract identifiers from email (PN, ECN, project name) using regex + LLM
2. Search Fusion Manage for each identifier
3. If unique match → proceed to update
4. If multiple matches → present choices (or filter by workspace type)
5. If no match → flag for human review
```

### Confidence Scoring

| Match Type | Confidence |
|---|---|
| Exact ECN number match | High (0.95) |
| Exact part number match, single result | High (0.90) |
| Part number with multiple results | Medium (0.60) |
| Project name (fuzzy) | Low (0.40) |
| No structured identifier, only description | Very Low (0.20) |

---

## 11. Python and Node.js SDKs

### Python

**No official APS Python SDK** is currently published/maintained by Autodesk Platform Services.

**Options:**
1. **Direct HTTP calls** using `requests` or `httpx` — recommended for a custom agent
2. **Legacy `forge-api-python-client`** on PyPI — this is a community/legacy package from the Forge era, may not cover Fusion Manage v3
3. **APS-generated client from OpenAPI spec** — use https://github.com/autodesk-platform-services/aps-sdk-openapi with OpenAPI Generator to generate a Python client

**Recommended Python approach:**
```python
import requests

def get_2legged_token(client_id: str, client_secret: str) -> str:
    import base64
    credentials = base64.b64encode(f"{client_id}:{client_secret}".encode()).decode()
    response = requests.post(
        "https://developer.api.autodesk.com/authentication/v2/token",
        headers={
            "Authorization": f"Basic {credentials}",
            "Content-Type": "application/x-www-form-urlencoded",
        },
        data={"grant_type": "client_credentials", "scope": "data:read data:write"},
    )
    return response.json()["access_token"]

def search_plm_records(token: str, tenant: str, query: str) -> list:
    resp = requests.get(
        f"https://{tenant}.autodeskplm360.net/api/v3/search-results",
        headers={
            "Authorization": f"Bearer {token}",
            "X-user-id": "service-account@company.com",
            "Accept": "application/vnd.autodesk.plm.items.bulk+json",
        },
        params={"query": query, "revision": 1, "limit": 10},
    )
    return resp.json().get("items", [])
```

### Node.js / TypeScript

**Official SDK:** https://github.com/autodesk-platform-services/aps-sdk-node (TypeScript, 38 stars)

```bash
npm install @aps_sdk/authentication @aps_sdk/data-management
```

Note: The official SDK covers Authentication, Data Management, Model Derivative, OSS, Webhooks, and ACC. It does **not** include a dedicated Fusion Manage module — direct HTTP calls are required for Fusion Manage v3.

### .NET

**Official SDK:** https://github.com/autodesk-platform-services/aps-sdk-net (C#, 36 stars)
Available as NuGet packages: `Autodesk.Authentication`, `Autodesk.DataManagement`, `Autodesk.ModelDerivative`, etc.

---

## 12. Mock / Sample Data Structure for Demo

Since live Fusion Manage access requires a subscription, use this mock data structure to build and test the AI agent pipeline.

### Mock Workspace Definitions

```json
{
  "workspaces": [
    { "id": 1, "title": "Items (Parts)", "shortName": "ITEMS" },
    { "id": 9, "title": "Change Orders", "shortName": "CO" },
    { "id": 12, "title": "Engineering Change Notices", "shortName": "ECN" },
    { "id": 15, "title": "Quality Non-Conformances", "shortName": "NC" },
    { "id": 22, "title": "Projects", "shortName": "PROJ" }
  ]
}
```

### Mock Item (Part Record)

```json
{
  "__self__": "/api/v3/workspaces/1/items/1001",
  "urn": "urn:adsk.plm:tenant.workspace.item:DEMO.1.1001",
  "title": "Hydraulic Pump Assembly - PN-1042-A",
  "deleted": false,
  "latestRelease": true,
  "workingVersion": false,
  "itemLocked": false,
  "currentState": {
    "title": "Released",
    "link": "/api/v3/workspaces/1/workflows/1/states/10"
  },
  "sections": [
    {
      "title": "General Information",
      "fields": [
        { "title": "Part Number", "value": "PN-1042-A", "type": { "title": "Single Line Text" }},
        { "title": "Description", "value": "Hydraulic Pump Assembly, 2000 PSI, Aluminum Body", "type": { "title": "Paragraph" }},
        { "title": "Revision", "value": "B", "type": { "title": "Single Line Text" }},
        { "title": "Status", "value": "Released", "type": { "title": "Single Line Text" }},
        { "title": "Material", "value": "6061 Aluminum", "type": { "title": "Single Line Text" }},
        { "title": "Weight (kg)", "value": "2.35", "type": { "title": "Number" }},
        { "title": "Owner", "value": "jane.doe@company.com", "type": { "title": "Single Line Text" }}
      ]
    },
    {
      "title": "Lifecycle",
      "fields": [
        { "title": "Created Date", "value": "2024-03-15", "type": { "title": "Date" }},
        { "title": "Last Modified", "value": "2025-11-02", "type": { "title": "Date" }},
        { "title": "Released Date", "value": "2025-01-20", "type": { "title": "Date" }}
      ]
    }
  ]
}
```

### Mock Engineering Change Notice (ECN)

```json
{
  "__self__": "/api/v3/workspaces/12/items/5042",
  "urn": "urn:adsk.plm:tenant.workspace.item:DEMO.12.5042",
  "title": "ECN-00234 - Update Pump Assembly Material Spec",
  "currentState": { "title": "In Review" },
  "sections": [
    {
      "title": "Change Details",
      "fields": [
        { "title": "ECN Number", "value": "ECN-00234", "type": { "title": "Single Line Text" }},
        { "title": "Title", "value": "Update Pump Assembly Material Spec", "type": { "title": "Single Line Text" }},
        { "title": "Reason for Change", "value": "Supplier discontinuation of 6061 alloy variant. Switching to 7075-T6.", "type": { "title": "Paragraph" }},
        { "title": "Priority", "value": "High", "type": { "title": "Single Line Text" }},
        { "title": "Affected Parts", "value": "PN-1042-A, PN-1043-B", "type": { "title": "Paragraph" }},
        { "title": "Requestor", "value": "john.smith@company.com", "type": { "title": "Single Line Text" }},
        { "title": "Implementation Date", "value": "2026-06-01", "type": { "title": "Date" }},
        { "title": "Approval Status", "value": "Pending", "type": { "title": "Single Line Text" }}
      ]
    }
  ],
  "workflow": {
    "link": "/api/v3/workspaces/12/items/5042/workflows/3",
    "currentState": "In Review",
    "availableTransitions": [
      { "id": 42, "title": "Approve", "targetState": "Approved" },
      { "id": 43, "title": "Reject", "targetState": "Rejected" },
      { "id": 44, "title": "Request More Info", "targetState": "On Hold" }
    ]
  }
}
```

### Mock Python Helper: Local Mock Server

For demos without live API access, spin up a lightweight mock server:

```python
# mock_plm_server.py
from flask import Flask, jsonify, request

app = Flask(__name__)

MOCK_ITEMS = {
    "9": {
        "134615": {
            "title": "ECN-00234 - Update Pump Assembly",
            "currentState": {"title": "In Review"},
            "sections": [
                {
                    "title": "Change Details",
                    "fields": [
                        {"title": "ECN Number", "value": "ECN-00234"},
                        {"title": "Priority", "value": "High"},
                        {"title": "Approval Status", "value": "Pending"}
                    ]
                }
            ]
        }
    }
}

@app.route("/api/v3/search-results")
def search():
    query = request.args.get("query", "")
    results = [
        item for ws_items in MOCK_ITEMS.values()
        for item in ws_items.values()
        if query.lower() in item["title"].lower()
    ]
    return jsonify({"items": results, "totalCount": len(results)})

@app.route("/api/v3/workspaces/<ws_id>/items/<item_id>")
def get_item(ws_id, item_id):
    return jsonify(MOCK_ITEMS.get(ws_id, {}).get(item_id, {}))

if __name__ == "__main__":
    app.run(port=5000)
```

---

## 13. Autodesk Vault (On-Premise) API vs Fusion Manage

### Overview Comparison

| Feature | Autodesk Vault | Autodesk Fusion Manage |
|---|---|---|
| Deployment | On-premise (Windows Server) | Cloud (SaaS) |
| Primary Focus | PDM (Product Data Management) — CAD file management | PLM (Product Lifecycle Management) — process management |
| API Type | REST API v2 (via Vault Data API) | REST API v3 |
| Auth | Vault token OR Autodesk ID 3-legged via Vault Gateway | APS 2-legged or 3-legged OAuth |
| Access URL | `http://{server}/AutodeskDM/Services/api/vault/v2/` OR `https://{id}.vg.autodesk.com/...` | `https://{tenant}.autodeskplm360.net/api/v3/` |
| OpenAPI Spec | `http://{server}/AutodeskDM/Services/api/vault/v2/openapi-spec.yml` | Embedded in help docs |
| Integration | Can connect to cloud via Vault Gateway | Native cloud |
| Best For | Companies with on-prem IT; heavy CAD data | Pure cloud PLM workflows |

### Vault Data API

**Base URL:** `http://{server-url}/AutodeskDM/Services/api/vault/v2/`

**Example — Get user by ID using Vault token:**
```bash
curl -X GET http://<server-url>/AutodeskDM/Services/api/vault/v2/users/9 \
  -H "Accept: application/json" \
  -H "Authorization: V:ce676ccb-01dd-41c0-8466-fb47701f5263"
```

**Example — Using APS 3-legged token via Vault Gateway:**
```bash
curl -X GET https://******.vg.autodesk.com/AutodeskDM/Services/api/vault/v2/users/9 \
  -H "Accept: application/json" \
  -H "Authorization: Bearer eyJhbGci..."
```

**GitHub samples:** https://github.com/autodesk-platform-services/vault-data-api-samples

### Which to Use for the AI Agent Pipeline?

- **Choose Fusion Manage** if the org is cloud-first or already using Fusion Manage.
- **Choose Vault API** if the org has on-premise Vault and cannot migrate.
- Both support REST; Fusion Manage has a richer PLM data model. Vault is primarily a PDM (files) system.

---

## 14. MCP Servers for Autodesk APIs

### Official Autodesk MCP Servers (as of May 2026)

**Documentation hub:** https://help.autodesk.com/view/ADSKMCP/ENU/

| MCP Server | Type | Description |
|---|---|---|
| **Autodesk Product Help MCP** | Remote | Direct AI access to 110+ Autodesk product docs, 10 languages |
| **Autodesk Fusion MCP** | Local (STDIO) | Executes actions against a live running Fusion Desktop session |
| **Autodesk Fusion Data MCP** | Remote | Tools to invite Fusion collaborators, manage projects, data management |
| **InfoWorks Hydraulic Modeling MCP** | Remote | Hydraulic modeling databases (coming soon) |

**Marketplace for community MCPs:** https://aps.autodesk.com/design-and-make-marketplace

### APS MCP Server Generator Skill

Autodesk provides an AI agent **skill** to scaffold custom MCP servers that integrate with APS:

**GitHub:** https://github.com/autodesk-platform-services/skills/tree/main/skills/aps-mcp-server-gen

**Supports:**
- Languages: Node.js/TypeScript, .NET/C#, Python
- Transport: STDIO (local) or Streamable HTTP (cloud)
- Auth patterns: 2-legged OAuth, Secure Service Accounts (SSA), 3-legged OAuth

**Install:**
```bash
npx skills add --project autodesk-platform-services/skills --skill aps-mcp-server-gen
```

### Community MCP Servers

- **AUTOM8LABS MCP Connector** — Connects Claude/Cursor to Autodesk Revit
- **Sentinel QC with MCP** — Revit model quality checking via AI
- Others listed in the marketplace

### Key Architecture Note

From the APS MCP skill rules:
> "Never implement token passthrough. The MCP server must authenticate with APS itself — never accept APS tokens from the MCP client."
> "Always cache tokens. Check expiry before every APS call; refresh proactively (≥60s before expiry)."

### No Fusion Manage–Specific MCP Server Found

As of research date, there is **no official or community MCP server specifically for Fusion Manage PLM records**. Building one using the `aps-mcp-server-gen` skill + direct Fusion Manage v3 API calls is the recommended approach.

---

## 15. ACC (Autodesk Construction Cloud) Data Connector

**Note:** ACC (now called **Forma** in the API docs) is primarily for **construction projects**, not manufacturing PLM. However, it may be relevant if the AI agent pipeline also processes construction project emails.

**Official docs:** https://aps.autodesk.com/en/docs/acc/v1/overview/

### Data Connector API (Relevant for Bulk Data Extraction)

The **Data Connector API** retrieves bulk data from Forma/ACC services for BI and analytics:

```bash
# Submit a data extraction request
POST https://developer.api.autodesk.com/construction/dataconnector/v2/projects/{projectId}/requests

# Check job status
GET https://developer.api.autodesk.com/construction/dataconnector/v2/projects/{projectId}/requests/{requestId}/jobs

# Download extracted data
GET https://developer.api.autodesk.com/construction/dataconnector/v2/projects/{projectId}/jobs/{jobId}/data/{name}
```

### Potential Use in Pipeline

- If emails reference **Forma projects** (RFIs, Issues, Submittals), ACC APIs are more appropriate than Fusion Manage
- For construction engineers monitoring project status via email → ACC Issues API, RFIs API
- **ACC Issues API** supports PATCH for updates, similar pattern to Fusion Manage

---

## 16. Recommended Architecture for the AI Agent Pipeline

### Pipeline Overview

```
Email Ingestion (IMAP/Graph API)
        │
        ▼
Email Parser + Entity Extractor (LLM — GPT-4o / Claude)
        │  Extracts: ECN#, PN#, project name, intent (update/approve/query)
        ▼
PLM Record Lookup (Fusion Manage Search API)
        │  GET /api/v3/search-results?query={extracted_identifier}
        ▼
Confidence Scorer
        │  High confidence → auto-proceed
        │  Low confidence → queue for human review
        ▼
PLM Record Update (Fusion Manage PATCH / Workflow Transition)
        │  PATCH /api/v3/workspaces/{wsId}/items/{itemId}
        │  POST  /api/v3/workspaces/{wsId}/items/{itemId}/workflows/{wfId}/transitions
        ▼
Confirmation Email + Audit Log
```

### Authentication Strategy

For the AI agent backend:
- Use **2-legged OAuth** (client_credentials)
- Add app to Fusion Manage **Security Settings → Allowed List**
- Use a dedicated **service account email** for `X-user-id`
- Cache the token; refresh 60s before expiry

### Key Implementation Decisions

| Decision | Recommendation |
|---|---|
| Live API vs Mock | Start with mock server (Section 12); swap to live when access available |
| Python vs Node.js | Python recommended for AI agent (LLM libraries, asyncio) |
| Direct HTTP vs SDK | Direct `httpx` calls — no official Python SDK for Fusion Manage |
| MCP server | Yes — scaffold with aps-mcp-server-gen skill for Claude integration |
| Search strategy | Full-text search first; workspace-filtered grid search for precision |
| Error handling | Wrap all PLM calls; rate limits apply per APS Free Tier quotas |

---

## References and Key URLs

| Resource | URL |
|---|---|
| Fusion Manage API docs (main) | https://help.autodesk.com/view/PLM/ENU/?guid=FLC_RestAPI_Read_Me_First_html |
| Resource Endpoints index | https://help.autodesk.com/view/PLM/ENU/?guid=FLC_RestAPI_Resource_Endpoints_html |
| Search Endpoint | https://help.autodesk.com/view/PLM/ENU/?guid=FLC_RestAPI_Advanced_Functionalities_search_endpoints_html |
| Item Details Endpoint | https://help.autodesk.com/view/PLM/ENU/?guid=FLC_RestAPI_Advanced_Functionalities_item_details_endpoints_html |
| 2-Legged OAuth Tutorial | https://help.autodesk.com/view/PLM/ENU/?guid=FLC_RestAPI_v3_API_2_legged_Tutorial_html |
| 3-Legged OAuth Tutorial | https://help.autodesk.com/view/PLM/ENU/?guid=FLC_RestAPI_v3_API_3_legged_Tutorial_html |
| APS Developer Portal | https://aps.autodesk.com/ |
| APS OAuth Scopes | https://aps.autodesk.com/en/docs/oauth/v2/developers_guide/scopes |
| APS 2-Legged Token Tutorial | https://aps.autodesk.com/en/docs/oauth/v2/tutorials/get-2-legged-token/ |
| APS GitHub org | https://github.com/autodesk-platform-services |
| APS SDK for Node.js | https://github.com/autodesk-platform-services/aps-sdk-node |
| APS SDK for .NET | https://github.com/autodesk-platform-services/aps-sdk-net |
| APS OpenAPI specs | https://github.com/autodesk-platform-services/aps-sdk-openapi |
| APS MCP Skills repo | https://github.com/autodesk-platform-services/skills |
| Vault Data API samples | https://github.com/autodesk-platform-services/vault-data-api-samples |
| APS LLMs.txt files | https://github.com/autodesk-platform-services/llmstxt |
| Autodesk MCP servers | https://help.autodesk.com/view/ADSKMCP/ENU/ |
| Design and Make Marketplace | https://aps.autodesk.com/design-and-make-marketplace |
| Manufacturing Data Model API | https://aps.autodesk.com/manufacturing-data-model-api |
| ACC/Forma APIs | https://aps.autodesk.com/en/docs/acc/v1/overview/ |
| APS Tutorials | https://aps.autodesk.com/tutorials |
| APS Getting Started | https://get-started.aps.autodesk.com/ |

---

## Clarifying Questions Requiring User Input

1. **Does the target org use Autodesk Fusion Manage (cloud) or Autodesk Vault (on-premise)?** This determines the API endpoint and auth mechanism.

2. **Is there an existing Fusion Manage tenant available for development/testing?** If not, mock data is necessary for the demo phase.

3. **What PLM workspace types are most relevant?** (Items/Parts, Change Orders, ECNs, Quality records, Projects?)

4. **What is the email source?** (Microsoft 365 Graph API, Gmail API, IMAP?) This affects the email ingestion component.

5. **What is the desired update action?** (Update a field value? Approve a workflow step? Add a comment? Create a new item?)

6. **Is the Manufacturing Data Model API (GraphQL) needed for BOM/part hierarchy data**, or is the Fusion Manage REST API sufficient?
