<!-- markdownlint-disable-file -->
# Implementation Details: Email-to-PLM Agent

## Context Reference

Sources:
* .copilot-tracking/research/2026-05-04/email-to-plm-agent-research.md — primary research, selected approach, mock data schemas, 9-step tool call sequence, agent system prompt
* .copilot-tracking/research/subagents/2026-05-04/integration-architecture-research.md — full code for PLM MCP server, mock data JSON, Flask mock server, Teams notification format
* .copilot-tracking/research/subagents/2026-05-04/autodesk-plm-research.md — Fusion Manage API endpoints, data model, JSON:API PATCH format, OAuth 2-legged flow
* .copilot-tracking/research/subagents/2026-05-04/office-mcp-email-research.md — M365 MCP server tool schemas (list-mail-messages, get-mail-message, update-mail-message)
* .copilot-tracking/research/subagents/2026-05-04/teams-mcp-research.md — Teams MCP server tools (send_channel_message, list_teams, list_channels), auth
* .copilot-tracking/research/subagents/2026-05-04/azure-ai-agent-research.md — Foundry Agent Service, Azure OpenAI orchestration patterns

Selected approach: Direct Azure OpenAI (GPT-4o) + `mcp` Python client managing three stdio MCP servers.
See .copilot-tracking/plans/logs/2026-05-04/email-to-plm-agent-log.md (DD-01, DD-02) for deviation rationale.

---

## Implementation Phase 1: Project Scaffolding

<!-- parallelizable: false -->

### Step 1.1: Create directory structure

Create all project directories. All subsequent steps write files into these directories.

```bash
mkdir -p hack/agent
mkdir -p hack/autodesk-plm-mcp/mock_data
```

Files created by this step:
* hack/agent/ — agent orchestration code
* hack/autodesk-plm-mcp/ — custom PLM MCP server + mock server
* hack/autodesk-plm-mcp/mock_data/ — static JSON mock records

Success criteria:
* Both directories exist
* No files yet; just empty directories

Dependencies:
* None — first step

---

### Step 1.2: Create .env.example

Create `/Users/alimurad/Desktop/projects/hack/.env.example` with content:

```bash
# ─────────────────────────────────────────────────
# Azure OpenAI
# ─────────────────────────────────────────────────
AZURE_OPENAI_ENDPOINT=https://<your-resource>.openai.azure.com/
AZURE_OPENAI_API_KEY=<your-azure-openai-key>
AZURE_OPENAI_DEPLOYMENT=gpt-4o

# ─────────────────────────────────────────────────
# Microsoft 365 — @softeria/ms-365-mcp-server
# Get token via: npx @softeria/ms-365-mcp-server --login --preset mail
# Copy token from OS keychain after login
# ─────────────────────────────────────────────────
MS365_OAUTH_TOKEN=<graph-api-delegated-token>

# ─────────────────────────────────────────────────
# Autodesk Platform Services
# Register app at: https://aps.autodesk.com/myapps/
# For mock mode: set USE_MOCK_PLM=true; APS_* values ignored
# APS_USER_ID: required for real Fusion Manage calls (2-legged impersonation)
# ─────────────────────────────────────────────────
APS_CLIENT_ID=<your-aps-client-id>
APS_CLIENT_SECRET=<your-aps-client-secret>
APS_PLM_TENANT_URL=https://acme.autodeskplm360.net
APS_USER_ID=<your-user-email@tenant.com>
USE_MOCK_PLM=true
MOCK_PLM_PORT=5001

# ─────────────────────────────────────────────────
# Microsoft Teams — @floriscornel/teams-mcp
# Get token via device code: npx @floriscornel/teams-mcp --login
# ─────────────────────────────────────────────────
TEAMS_AUTH_TOKEN=<graph-api-delegated-token>
TEAMS_TEAM_NAME=Engineering
TEAMS_CHANNEL_NAME=PLM Updates
```

Success criteria:
* File exists at hack/.env.example
* Copy to hack/.env and fill in real values before running

Dependencies:
* Step 1.1 complete

---

### Step 1.3: Create mcp_config.json

Create `/Users/alimurad/Desktop/projects/hack/mcp_config.json` with content:

```json
{
  "mcpServers": {
    "ms365-mail": {
      "command": "npx",
      "args": ["-y", "@softeria/ms-365-mcp-server", "--preset", "mail", "--org-mode"],
      "env": {
        "MS365_MCP_OAUTH_TOKEN": "${MS365_OAUTH_TOKEN}"
      }
    },
    "autodesk-plm": {
      "command": "python",
      "args": ["autodesk-plm-mcp/server.py"],
      "env": {
        "APS_PLM_TENANT_URL": "${APS_PLM_TENANT_URL}",
        "APS_CLIENT_ID": "${APS_CLIENT_ID}",
        "APS_CLIENT_SECRET": "${APS_CLIENT_SECRET}",
        "USE_MOCK_PLM": "${USE_MOCK_PLM}",
        "MOCK_PLM_PORT": "${MOCK_PLM_PORT}"
      }
    },
    "teams-mcp": {
      "command": "npx",
      "args": ["-y", "@floriscornel/teams-mcp@latest"],
      "env": {
        "AUTH_TOKEN": "${TEAMS_AUTH_TOKEN}"
      }
    }
  }
}
```

Success criteria:
* File exists at hack/mcp_config.json
* Valid JSON

Dependencies:
* Step 1.1 complete

---

### Step 1.4: Validate phase changes

Run: `cd /Users/alimurad/Desktop/projects/hack && ls -la agent/ autodesk-plm-mcp/ .env.example mcp_config.json`

---

## Implementation Phase 2: Mock Data and Mock PLM Server

<!-- parallelizable: true -->

### Step 2.1: Create mock_data/emails.json

Create `/Users/alimurad/Desktop/projects/hack/autodesk-plm-mcp/mock_data/emails.json` with 4 realistic Autodesk engineering scenario emails:

```json
[
  {
    "id": "AAMkAGI2ZTlmNjEw",
    "subject": "ECN-2024-0342: Engineering Change Notice Requires Your Approval",
    "from": {
      "emailAddress": {
        "name": "Sarah Chen",
        "address": "sarah.chen@acme-engineering.com"
      }
    },
    "toRecipients": [
      {
        "emailAddress": {
          "name": "James Whitfield",
          "address": "j.whitfield@acme-engineering.com"
        }
      }
    ],
    "receivedDateTime": "2024-11-15T09:23:00Z",
    "isRead": false,
    "importance": "high",
    "body": {
      "contentType": "html",
      "content": "<p>Hi James,</p><p>ECN <strong>ECN-2024-0342</strong> - <em>Redesign of Heat Exchanger Bracket Assembly</em> has been routed to you for final approval.</p><p><strong>Change Summary:</strong></p><ul><li>Part Number: <strong>100-4821</strong> (Heat Exchanger Bracket - Rev B to Rev C)</li><li>Part Number: <strong>100-4822</strong> (Mounting Flange - Rev A to Rev B)</li><li>Reason: Field failure reports indicate fatigue cracking at weld joint under thermal cycling</li><li>Estimated Impact: Medium - affects 3 active assemblies</li></ul><p>PLM Workspace: ECN | Item ID: 1452<br/>Change Request: <strong>CR-2024-0287</strong></p><p>Please review and approve by EOD Friday, 2024-11-22.</p><p>Regards,<br/>Sarah Chen<br/>Senior Mechanical Engineer</p>"
    },
    "categories": ["Engineering", "PLM", "Approval Required"]
  },
  {
    "id": "AAMkAGI2ZTlmNjEy",
    "subject": "[PLM Notification] Change Request CR-2024-0301 Status Updated",
    "from": {
      "emailAddress": {
        "name": "Fusion Manage System",
        "address": "plm-notifications@acme-engineering.com"
      }
    },
    "toRecipients": [
      {
        "emailAddress": {
          "name": "PLM Engineering Team",
          "address": "plm-team@acme-engineering.com"
        }
      }
    ],
    "receivedDateTime": "2024-11-15T11:45:00Z",
    "isRead": false,
    "importance": "normal",
    "body": {
      "contentType": "text",
      "content": "The following Change Request has been updated in Fusion Manage:\n\nChange Request: CR-2024-0301\nTitle: Tolerance Adjustment for Turbine Blade Profile\nPrevious Status: In Design\nNew Status: Under Review\nUpdated By: Marcus Okafor (m.okafor@acme-engineering.com)\nTimestamp: 2024-11-15 11:40 UTC\n\nAffected Parts:\n- 200-0891 (Turbine Blade Stage 1 - Rev D)\n- 200-0892 (Blade Root Attachment - Rev B)\n\nAssociated ECN: ECN-2024-0338\n\nReview deadline: 2024-11-29\nAccess the record: https://acme.autodeskplm360.net/workspace/changeorders/CR-2024-0301\n\nThis is an automated notification from Fusion Manage."
    },
    "categories": ["PLM", "Change Request", "Automated"]
  },
  {
    "id": "AAMkAGI2ZTlmNjEz",
    "subject": "URGENT: Part 300-1204 obsoleted — downstream impact on ECN-2024-0355",
    "from": {
      "emailAddress": {
        "name": "David Reinholt",
        "address": "d.reinholt@acme-engineering.com"
      }
    },
    "toRecipients": [
      {
        "emailAddress": {
          "name": "ECN Review Board",
          "address": "ecn-board@acme-engineering.com"
        }
      }
    ],
    "receivedDateTime": "2024-11-15T14:10:00Z",
    "isRead": false,
    "importance": "high",
    "body": {
      "contentType": "html",
      "content": "<p>Team,</p><p>Part <strong>300-1204</strong> (Seal Assembly Sub-Frame) has been marked <strong>Obsolete</strong> in Fusion Manage due to supplier discontinuation (Supplier: FormTech Industries, SKU: FT-8832).</p><p>This part is referenced in <strong>ECN-2024-0355</strong> as a retained component. The ECN must be updated to either:</p><ol><li>Substitute with approved alternate <strong>300-1205</strong> (FormTech FT-8833 compatible)</li><li>Issue a new CR for re-design if no alternate is suitable</li></ol><p>Current ECN-2024-0355 status: <strong>In Review</strong><br/>Assigned Engineer: Lisa Park (l.park@acme-engineering.com)<br/>ECN workspace item ID: 1478</p><p>Please update the PLM record and notify the project lead immediately.</p><p>— David Reinholt, Supply Chain Engineering</p>"
    },
    "categories": ["URGENT", "Parts", "ECN", "Supply Chain"]
  },
  {
    "id": "AAMkAGI2ZTlmNjE0",
    "subject": "[Weekly Digest] Open ECNs Awaiting Action — 4 items",
    "from": {
      "emailAddress": {
        "name": "Fusion Manage Digest",
        "address": "digest@acme-engineering.com"
      }
    },
    "toRecipients": [
      {
        "emailAddress": {
          "name": "Engineering Managers",
          "address": "eng-managers@acme-engineering.com"
        }
      }
    ],
    "receivedDateTime": "2024-11-15T07:00:00Z",
    "isRead": false,
    "importance": "normal",
    "body": {
      "contentType": "text",
      "content": "Weekly PLM Digest - 2024-11-11 to 2024-11-15\n\nOPEN ECNs AWAITING ACTION:\n\n1. ECN-2024-0339 | Cooling Fan Bracket Redesign | Status: Pending Approval | Owner: Sarah Chen | Due: 2024-11-18\n2. ECN-2024-0342 | Heat Exchanger Bracket Assembly | Status: In Review | Owner: James Whitfield | Due: 2024-11-22\n3. ECN-2024-0350 | Motor Controller Housing | Status: Draft | Owner: Carlos Mendez | Due: 2024-11-25\n4. ECN-2024-0355 | Seal Assembly Sub-Frame | Status: In Review | Owner: Lisa Park | Due: 2024-11-20\n\nTOTAL OPEN CHANGE REQUESTS: 7\nHIGH PRIORITY ITEMS: 2 (ECN-2024-0339, ECN-2024-0355)\n\nFull report: https://acme.autodeskplm360.net/reports/weekly-digest"
    },
    "categories": ["PLM", "Digest", "Management"]
  }
]
```

Success criteria:
* Valid JSON array with 4 email objects
* Each email has id, subject, from, receivedDateTime, isRead=false, body, importance, categories

Dependencies:
* Step 1.1 (mock_data directory exists)

---

### Step 2.2: Create mock_data/ecn_records.json

Create `/Users/alimurad/Desktop/projects/hack/autodesk-plm-mcp/mock_data/ecn_records.json`:

```json
[
  {
    "id": "1452",
    "workspaceId": 57,
    "number": "ECN-2024-0342",
    "title": "Redesign of Heat Exchanger Bracket Assembly",
    "status": "In Review",
    "createdDate": "2024-11-01T08:00:00Z",
    "modifiedDate": "2024-11-15T09:30:00Z",
    "fields": {
      "ECN_NUMBER": { "value": "ECN-2024-0342", "label": "ECN Number", "type": "text" },
      "TITLE": { "value": "Redesign of Heat Exchanger Bracket Assembly", "label": "Title", "type": "text" },
      "STATUS": {
        "value": "In Review",
        "label": "Status",
        "allowedValues": ["Draft", "In Review", "Pending Approval", "Approved", "Released", "Rejected", "On Hold", "Obsolete"],
        "type": "picklist"
      },
      "PRIORITY": { "value": "Medium", "label": "Priority", "allowedValues": ["Low", "Medium", "High", "Critical"], "type": "picklist" },
      "CHANGE_REASON": { "value": "Field failure reports indicate fatigue cracking at weld joint under thermal cycling. Root cause analysis (RCA-2024-112) confirmed design deficiency at T-joint.", "label": "Reason for Change", "type": "textarea" },
      "ORIGINATOR": { "value": "sarah.chen@acme-engineering.com", "label": "Originator", "type": "user" },
      "APPROVER": { "value": "j.whitfield@acme-engineering.com", "label": "Approver", "type": "user" },
      "CATEGORY": { "value": "Design - Safety", "label": "Change Category", "type": "picklist" },
      "IMPLEMENTATION_DATE": { "value": "2024-12-01", "label": "Target Implementation Date", "type": "date" },
      "ASSOCIATED_CR": { "value": "CR-2024-0287", "label": "Associated Change Request", "type": "reference" },
      "COMMENTS": { "value": "", "label": "Comments", "type": "textarea" }
    },
    "affectedItems": [
      { "partNumber": "100-4821", "partTitle": "Heat Exchanger Bracket", "fromRevision": "B", "toRevision": "C", "disposition": "Replace" },
      { "partNumber": "100-4822", "partTitle": "Mounting Flange", "fromRevision": "A", "toRevision": "B", "disposition": "Replace" }
    ],
    "workflow": {
      "currentStep": "Engineering Review",
      "steps": ["Draft", "Initiator Review", "Engineering Review", "Approval", "Released"]
    },
    "links": {
      "plmUrl": "https://acme.autodeskplm360.net/workspace/ecn/1452"
    }
  },
  {
    "id": "1478",
    "workspaceId": 57,
    "number": "ECN-2024-0355",
    "title": "Seal Assembly Sub-Frame Component Update",
    "status": "In Review",
    "createdDate": "2024-10-28T10:00:00Z",
    "modifiedDate": "2024-11-14T16:45:00Z",
    "fields": {
      "ECN_NUMBER": { "value": "ECN-2024-0355", "label": "ECN Number", "type": "text" },
      "TITLE": { "value": "Seal Assembly Sub-Frame Component Update", "label": "Title", "type": "text" },
      "STATUS": { "value": "In Review", "label": "Status", "allowedValues": ["Draft", "In Review", "Pending Approval", "Approved", "Released", "Rejected", "On Hold", "Obsolete"], "type": "picklist" },
      "PRIORITY": { "value": "High", "label": "Priority", "type": "picklist" },
      "CHANGE_REASON": { "value": "Component redesign to improve seal retention under high-pressure cycles.", "label": "Reason for Change", "type": "textarea" },
      "ORIGINATOR": { "value": "d.reinholt@acme-engineering.com", "label": "Originator", "type": "user" },
      "APPROVER": { "value": "l.park@acme-engineering.com", "label": "Approver", "type": "user" },
      "COMMENTS": { "value": "", "label": "Comments", "type": "textarea" }
    },
    "affectedItems": [
      { "partNumber": "300-1204", "partTitle": "Seal Assembly Sub-Frame", "fromRevision": "A", "toRevision": "B", "disposition": "Replace" }
    ],
    "workflow": {
      "currentStep": "Engineering Review",
      "steps": ["Draft", "Initiator Review", "Engineering Review", "Approval", "Released"]
    },
    "links": {
      "plmUrl": "https://acme.autodeskplm360.net/workspace/ecn/1478"
    }
  },
  {
    "id": "1461",
    "workspaceId": 57,
    "number": "ECN-2024-0338",
    "title": "Turbine Blade Profile Tolerance Update",
    "status": "In Design",
    "createdDate": "2024-10-20T09:00:00Z",
    "modifiedDate": "2024-11-12T11:00:00Z",
    "fields": {
      "ECN_NUMBER": { "value": "ECN-2024-0338", "label": "ECN Number", "type": "text" },
      "TITLE": { "value": "Turbine Blade Profile Tolerance Update", "label": "Title", "type": "text" },
      "STATUS": { "value": "In Design", "label": "Status", "type": "picklist" },
      "PRIORITY": { "value": "Medium", "label": "Priority", "type": "picklist" },
      "ASSOCIATED_CR": { "value": "CR-2024-0301", "label": "Associated Change Request", "type": "reference" },
      "COMMENTS": { "value": "", "label": "Comments", "type": "textarea" }
    },
    "affectedItems": [
      { "partNumber": "200-0891", "partTitle": "Turbine Blade Stage 1", "fromRevision": "D", "toRevision": "E", "disposition": "Replace" }
    ],
    "workflow": { "currentStep": "Engineering Design", "steps": ["Draft", "Engineering Design", "Engineering Review", "Approval", "Released"] },
    "links": { "plmUrl": "https://acme.autodeskplm360.net/workspace/ecn/1461" }
  }
]
```

Success criteria:
* Valid JSON array with 3 ECN records
* Each record has id, workspaceId=57, number, status, fields (including STATUS with allowedValues), affectedItems

Dependencies:
* Step 1.1 complete

---

### Step 2.3: Create mock_data/cr_records.json

Create `/Users/alimurad/Desktop/projects/hack/autodesk-plm-mcp/mock_data/cr_records.json`:

```json
[
  {
    "id": "892",
    "workspaceId": 28,
    "number": "CR-2024-0287",
    "title": "Heat Exchanger Bracket Fatigue Fix",
    "status": "Under Review",
    "createdDate": "2024-10-15T09:00:00Z",
    "modifiedDate": "2024-11-08T14:30:00Z",
    "fields": {
      "CR_NUMBER": { "value": "CR-2024-0287", "label": "CR Number", "type": "text" },
      "TITLE": { "value": "Heat Exchanger Bracket Fatigue Fix", "label": "Title", "type": "text" },
      "STATUS": { "value": "Under Review", "label": "Status", "allowedValues": ["Submitted", "In Design", "Under Review", "Approved", "Rejected", "Closed"], "type": "picklist" },
      "PRIORITY": { "value": "High", "label": "Priority", "type": "picklist" },
      "DESCRIPTION": { "value": "During commissioning test cycle TC-4882, bracket assembly showed micro-cracking at weld joint after 500 thermal cycles (target: 10,000). FEA model updated to reflect actual operating loads.", "label": "Description", "type": "textarea" },
      "REQUESTED_BY": { "value": "sarah.chen@acme-engineering.com", "label": "Requested By", "type": "user" },
      "CATEGORY": { "value": "Safety", "label": "Category", "type": "picklist" },
      "LINKED_ECN": { "value": "ECN-2024-0342", "label": "Linked ECN", "type": "reference" },
      "COMMENTS": { "value": "", "label": "Comments", "type": "textarea" }
    },
    "links": { "plmUrl": "https://acme.autodeskplm360.net/workspace/changerequests/892" }
  },
  {
    "id": "905",
    "workspaceId": 28,
    "number": "CR-2024-0301",
    "title": "Tolerance Adjustment for Turbine Blade Profile",
    "status": "Under Review",
    "createdDate": "2024-10-28T11:00:00Z",
    "modifiedDate": "2024-11-15T11:40:00Z",
    "fields": {
      "CR_NUMBER": { "value": "CR-2024-0301", "label": "CR Number", "type": "text" },
      "TITLE": { "value": "Tolerance Adjustment for Turbine Blade Profile", "label": "Title", "type": "text" },
      "STATUS": { "value": "Under Review", "label": "Status", "allowedValues": ["Submitted", "In Design", "Under Review", "Approved", "Rejected", "Closed"], "type": "picklist" },
      "PRIORITY": { "value": "Medium", "label": "Priority", "type": "picklist" },
      "DESCRIPTION": { "value": "Tighten tolerance on blade tip profile from +/-0.5mm to +/-0.2mm to reduce turbulence. Affects manufacturing fixture tooling.", "label": "Description", "type": "textarea" },
      "REQUESTED_BY": { "value": "m.okafor@acme-engineering.com", "label": "Requested By", "type": "user" },
      "LINKED_ECN": { "value": "ECN-2024-0338", "label": "Linked ECN", "type": "reference" },
      "COMMENTS": { "value": "", "label": "Comments", "type": "textarea" }
    },
    "links": { "plmUrl": "https://acme.autodeskplm360.net/workspace/changerequests/905" }
  }
]
```

Success criteria:
* Valid JSON array with 2 CR records
* Each has workspaceId=28, status, fields, links

Dependencies:
* Step 1.1 complete

---

### Step 2.4: Create mock_data/part_records.json

Create `/Users/alimurad/Desktop/projects/hack/autodesk-plm-mcp/mock_data/part_records.json`:

```json
[
  {
    "id": "3391",
    "workspaceId": 14,
    "number": "100-4821",
    "title": "Heat Exchanger Bracket",
    "revision": "B",
    "status": "Released",
    "fields": {
      "PART_NUMBER": { "value": "100-4821", "label": "Part Number", "type": "text" },
      "DESCRIPTION": { "value": "Structural bracket for heat exchanger mounting on Frame Sub-Assembly A", "label": "Description", "type": "text" },
      "REVISION": { "value": "B", "label": "Revision", "type": "text" },
      "STATUS": { "value": "Released", "label": "Status", "allowedValues": ["In Design", "Released", "Obsolete", "On Hold"], "type": "picklist" },
      "MATERIAL": { "value": "304 Stainless Steel", "label": "Material", "type": "text" },
      "WEIGHT_KG": { "value": 0.842, "label": "Weight (kg)", "type": "number" },
      "DRAWING_NUMBER": { "value": "DRG-100-4821-B", "label": "Drawing Number", "type": "text" },
      "COMMENTS": { "value": "", "label": "Comments", "type": "textarea" }
    },
    "pendingChanges": [{ "ecnNumber": "ECN-2024-0342", "targetRevision": "C", "status": "In Review" }],
    "links": { "plmUrl": "https://acme.autodeskplm360.net/workspace/parts/3391" }
  },
  {
    "id": "3392",
    "workspaceId": 14,
    "number": "100-4822",
    "title": "Mounting Flange",
    "revision": "A",
    "status": "Released",
    "fields": {
      "PART_NUMBER": { "value": "100-4822", "label": "Part Number", "type": "text" },
      "DESCRIPTION": { "value": "Mounting flange for heat exchanger bracket assembly", "label": "Description", "type": "text" },
      "REVISION": { "value": "A", "label": "Revision", "type": "text" },
      "STATUS": { "value": "Released", "label": "Status", "allowedValues": ["In Design", "Released", "Obsolete", "On Hold"], "type": "picklist" },
      "MATERIAL": { "value": "Mild Steel A36", "label": "Material", "type": "text" },
      "COMMENTS": { "value": "", "label": "Comments", "type": "textarea" }
    },
    "pendingChanges": [{ "ecnNumber": "ECN-2024-0342", "targetRevision": "B", "status": "In Review" }],
    "links": { "plmUrl": "https://acme.autodeskplm360.net/workspace/parts/3392" }
  },
  {
    "id": "4201",
    "workspaceId": 14,
    "number": "300-1204",
    "title": "Seal Assembly Sub-Frame",
    "revision": "A",
    "status": "Obsolete",
    "fields": {
      "PART_NUMBER": { "value": "300-1204", "label": "Part Number", "type": "text" },
      "DESCRIPTION": { "value": "Sub-frame for seal assembly. OBSOLETE — supplier FormTech discontinued SKU FT-8832.", "label": "Description", "type": "text" },
      "REVISION": { "value": "A", "label": "Revision", "type": "text" },
      "STATUS": { "value": "Obsolete", "label": "Status", "allowedValues": ["In Design", "Released", "Obsolete", "On Hold"], "type": "picklist" },
      "COMMENTS": { "value": "Supplier FormTech Industries discontinued SKU FT-8832 as of 2024-11-01. Alternate: 300-1205", "label": "Comments", "type": "textarea" }
    },
    "pendingChanges": [],
    "links": { "plmUrl": "https://acme.autodeskplm360.net/workspace/parts/4201" }
  }
]
```

Success criteria:
* Valid JSON array with 3 part records
* Includes Part 300-1204 with status Obsolete (for email 3 scenario)

Dependencies:
* Step 1.1 complete

---

### Step 2.5: Create autodesk-plm-mcp/mock_server.py

Create `/Users/alimurad/Desktop/projects/hack/autodesk-plm-mcp/mock_server.py`:

```python
"""
Mock Autodesk Fusion Manage REST API server.
Simulates the PLM 360 API v3 for demo purposes without a Fusion Manage subscription.

Run: python autodesk-plm-mcp/mock_server.py
Default port: 5001 (set MOCK_PLM_PORT env var to override)
"""
import json
import os
from pathlib import Path

from flask import Flask, jsonify, request, abort

app = Flask(__name__)

DATA_DIR = Path(__file__).parent / "mock_data"

# Workspace ID to metadata mapping
WORKSPACES = {
    57: {"id": 57, "name": "Engineering Change Notices", "shortName": "ECN", "itemCount": 3},
    42: {"id": 42, "name": "Change Orders", "shortName": "CO", "itemCount": 0},
    14: {"id": 14, "name": "Parts", "shortName": "PARTS", "itemCount": 3},
    28: {"id": 28, "name": "Change Requests", "shortName": "CR", "itemCount": 2},
}

WORKSPACE_FILES = {
    57: "ecn_records.json",
    28: "cr_records.json",
    14: "part_records.json",
}

# In-memory store for PATCH updates applied during the session
_updates: dict[str, dict] = {}


def load_records(filename: str) -> list:
    path = DATA_DIR / filename
    with open(path) as f:
        return json.load(f)


def apply_updates(record: dict, ws_id: int) -> dict:
    key = f"{ws_id}:{record['id']}"
    if key not in _updates:
        return record
    record = dict(record)
    record["fields"] = dict(record.get("fields", {}))
    for field_id, field_data in _updates[key].get("fields", {}).items():
        if field_id in record["fields"]:
            record["fields"][field_id] = dict(record["fields"][field_id])
            record["fields"][field_id]["value"] = field_data.get("value")
        else:
            record["fields"][field_id] = field_data
    if "status" in _updates[key]:
        record["status"] = _updates[key]["status"]
    return record


@app.route("/api/v3/workspaces", methods=["GET"])
def list_workspaces():
    return jsonify({"data": list(WORKSPACES.values())})


@app.route("/api/v3/workspaces/<int:ws_id>/items", methods=["GET"])
def list_items(ws_id: int):
    if ws_id not in WORKSPACES:
        abort(404, description=f"Workspace {ws_id} not found")
    filename = WORKSPACE_FILES.get(ws_id)
    if not filename:
        return jsonify({"data": [], "meta": {"total": 0}})

    records = load_records(filename)
    records = [apply_updates(r, ws_id) for r in records]

    # Apply filter[number] query param (case-insensitive)
    number_filter = request.args.get("filter[number]")
    if number_filter:
        records = [r for r in records if r.get("number", "").upper() == number_filter.upper()]

    return jsonify({"data": records, "meta": {"total": len(records)}})


@app.route("/api/v3/workspaces/<int:ws_id>/items/<string:item_id>", methods=["GET"])
def get_item(ws_id: int, item_id: str):
    if ws_id not in WORKSPACES:
        abort(404)
    filename = WORKSPACE_FILES.get(ws_id)
    if not filename:
        abort(404)
    records = load_records(filename)
    record = next((r for r in records if r["id"] == item_id), None)
    if not record:
        abort(404, description=f"Item {item_id} not found in workspace {ws_id}")
    return jsonify({"data": apply_updates(record, ws_id)})


@app.route("/api/v3/workspaces/<int:ws_id>/items/<string:item_id>", methods=["PATCH"])
def update_item(ws_id: int, item_id: str):
    if ws_id not in WORKSPACES:
        abort(404)
    payload = request.get_json(silent=True)
    if not payload or "data" not in payload:
        abort(400, description="Request body must contain JSON:API 'data' object")

    attributes = payload["data"].get("attributes", {})
    fields = attributes.get("fields", {})

    key = f"{ws_id}:{item_id}"
    if key not in _updates:
        _updates[key] = {"fields": {}}

    for field_id, field_data in fields.items():
        _updates[key]["fields"][field_id] = field_data
        if field_id == "STATUS":
            _updates[key]["status"] = field_data.get("value")

    print(f"[MockPLM] PATCH workspace={ws_id} item={item_id} fields={list(fields.keys())}")
    if "STATUS" in fields:
        print(f"[MockPLM]   STATUS → {fields['STATUS'].get('value')}")

    return jsonify({
        "data": {
            "id": item_id,
            "type": "items",
            "attributes": {
                "message": f"Item {item_id} in workspace {ws_id} updated successfully",
                "updatedFields": list(fields.keys()),
            }
        }
    }), 200


@app.route("/api/v3/health", methods=["GET"])
def health():
    return jsonify({"status": "ok", "server": "Mock Fusion Manage API v3"})


if __name__ == "__main__":
    port = int(os.environ.get("MOCK_PLM_PORT", 5001))
    print(f"[MockPLM] Mock Autodesk Fusion Manage API running at http://localhost:{port}")
    print(f"[MockPLM] Data directory: {DATA_DIR}")
    print(f"[MockPLM] Workspaces: ECN(57), CR(28), Parts(14), CO(42)")
    app.run(host="0.0.0.0", port=port, debug=False)
```

Success criteria:
* Flask server starts on port 5001
* `GET /api/v3/workspaces` returns 4 workspaces
* `GET /api/v3/workspaces/57/items?filter[number]=ECN-2024-0342` returns ECN record
* `PATCH /api/v3/workspaces/57/items/1452` persists field update in memory

Dependencies:
* Steps 2.1–2.4 complete (mock_data JSON files)

---

### Step 2.6: Validate phase changes

```bash
cd /Users/alimurad/Desktop/projects/hack
pip install flask
python autodesk-plm-mcp/mock_server.py &
curl http://localhost:5001/api/v3/health
curl "http://localhost:5001/api/v3/workspaces/57/items?filter%5Bnumber%5D=ECN-2024-0342"
```

Expected: health returns `{"status": "ok"}`, items query returns ECN-2024-0342 record.

---

## Implementation Phase 3: Custom Autodesk PLM MCP Server

<!-- parallelizable: true -->

### Step 3.1: Create autodesk-plm-mcp/models.py

Create `/Users/alimurad/Desktop/projects/hack/autodesk-plm-mcp/models.py`:

```python
"""Pydantic data models for Autodesk PLM MCP server tool return types."""
from typing import Any
from pydantic import BaseModel


class PLMSearchResult(BaseModel):
    found: bool
    workspace: str
    identifier: str
    records: list[dict[str, Any]]
    total: int = 0
    message: str = ""


class PLMUpdateResult(BaseModel):
    success: bool
    workspace: str
    item_id: str
    updated_fields: list[str]
    message: str


class PLMRecord(BaseModel):
    id: str
    workspace_id: int | None = None
    number: str | None = None
    title: str | None = None
    status: str | None = None
    fields: dict[str, Any] = {}
    affected_items: list[dict[str, Any]] = []
    workflow: dict[str, Any] = {}
    links: dict[str, str] = {}
```

Success criteria:
* File is valid Python, importable without error

Dependencies:
* Step 1.1 complete

---

### Step 3.2: Create autodesk-plm-mcp/plm_client.py

Create `/Users/alimurad/Desktop/projects/hack/autodesk-plm-mcp/plm_client.py`:

```python
"""
Autodesk Fusion Manage REST API client.
Supports both real APS 2-legged OAuth and mock server mode.

API reference: https://help.autodesk.com/view/PLM/ENU/?guid=FLC_RestAPI_Read_Me_First_html
APS auth: https://aps.autodesk.com/en/docs/oauth/v2/reference/http/gettoken-POST/
"""
import os
import time
from typing import Any

import httpx


class FusionManageClient:
    """
    Client for Autodesk Fusion Manage (PLM 360) REST API v3.

    In mock mode (USE_MOCK_PLM=true), skips APS OAuth and connects
    directly to the local Flask mock server.
    """

    APS_AUTH_URL = "https://developer.api.autodesk.com/authentication/v2/token"
    JSONAPI_CONTENT_TYPE = "application/vnd.api+json"

    def __init__(
        self,
        tenant_url: str,
        client_id: str,
        client_secret: str,
        mock_mode: bool = False,
    ):
        self.tenant_url = tenant_url.rstrip("/")
        self.client_id = client_id
        self.client_secret = client_secret
        self.mock_mode = mock_mode
        self._token: str | None = None
        self._token_expiry: float = 0

    def _get_token(self) -> str:
        """Obtain or refresh APS OAuth 2.0 access token (2-legged / client credentials)."""
        if self.mock_mode:
            return "mock-token"  # mock server accepts any bearer value
        if self._token and time.time() < self._token_expiry - 60:
            return self._token

        resp = httpx.post(
            self.APS_AUTH_URL,
            data={
                "grant_type": "client_credentials",
                "scope": "data:read data:write",
                "client_id": self.client_id,
                "client_secret": self.client_secret,
            },
            headers={"Content-Type": "application/x-www-form-urlencoded"},
            timeout=30,
        )
        resp.raise_for_status()
        payload = resp.json()
        self._token = payload["access_token"]
        self._token_expiry = time.time() + payload.get("expires_in", 3600)
        return self._token

    def _headers(self) -> dict[str, str]:
        headers = {
            "Authorization": f"Bearer {self._get_token()}",
            "Content-Type": self.JSONAPI_CONTENT_TYPE,
            "Accept": self.JSONAPI_CONTENT_TYPE,
        }
        # Fusion Manage requires X-user-id on every call (2-legged impersonation)
        user_id = os.environ.get("APS_USER_ID", "")
        if user_id:
            headers["X-user-id"] = user_id
        return headers

    def list_workspaces(self) -> list[dict]:
        """List all workspaces in the tenant."""
        resp = httpx.get(
            f"{self.tenant_url}/api/v3/workspaces",
            headers=self._headers(),
            timeout=30,
        )
        resp.raise_for_status()
        return resp.json().get("data", [])

    def search_items(self, workspace_id: int, identifier: str) -> list[dict]:
        """Search PLM workspace items by number/identifier (case-insensitive)."""
        resp = httpx.get(
            f"{self.tenant_url}/api/v3/workspaces/{workspace_id}/items",
            params={"offset": 0, "limit": 5, "filter[number]": identifier},
            headers=self._headers(),
            timeout=30,
        )
        resp.raise_for_status()
        return resp.json().get("data", [])

    def get_item(self, workspace_id: int, item_id: str) -> dict:
        """Get a specific PLM item by its workspace and item ID."""
        resp = httpx.get(
            f"{self.tenant_url}/api/v3/workspaces/{workspace_id}/items/{item_id}",
            headers=self._headers(),
            timeout=30,
        )
        resp.raise_for_status()
        return resp.json().get("data", {})

    def update_item(
        self,
        workspace_id: int,
        item_id: str,
        fields: dict[str, Any],
        comment: str | None = None,
    ) -> dict:
        """
        Update fields on a PLM item using JSON:API PATCH format.

        fields: dict of { FIELD_ID: new_value_string }
        """
        attributes: dict[str, Any] = {
            "fields": {field_id: {"value": value} for field_id, value in fields.items()}
        }
        if comment:
            attributes["comment"] = comment

        payload = {
            "data": {
                "type": "items",
                "id": item_id,
                "attributes": attributes,
            }
        }

        resp = httpx.patch(
            f"{self.tenant_url}/api/v3/workspaces/{workspace_id}/items/{item_id}",
            json=payload,
            headers=self._headers(),
            timeout=30,
        )
        resp.raise_for_status()
        return {"success": True, "message": f"Item {item_id} updated successfully"}
```

Success criteria:
* `FusionManageClient(tenant_url="http://localhost:5001", client_id="mock", client_secret="mock", mock_mode=True)` instantiates without error
* `client.search_items(57, "ECN-2024-0342")` returns list when mock server is running

Dependencies:
* Step 1.1 complete

---

### Step 3.3: Create autodesk-plm-mcp/server.py

Create `/Users/alimurad/Desktop/projects/hack/autodesk-plm-mcp/server.py`:

```python
"""
Autodesk Fusion Manage PLM MCP Server.
FastMCP-based server exposing PLM search and update tools to AI agents.

Exposes 3 tools:
  - search_plm_record: Search for a PLM item by ECN/CR/Part number
  - get_plm_record: Retrieve full details of a PLM item by ID
  - update_plm_record: Update one or more fields on a PLM item

Usage (stdio, for agent/main.py):
  python autodesk-plm-mcp/server.py

Environment variables:
  APS_PLM_TENANT_URL   - Fusion Manage tenant URL or http://localhost:5001 for mock
  APS_CLIENT_ID        - APS app client ID (ignored in mock mode)
  APS_CLIENT_SECRET    - APS app client secret (ignored in mock mode)
  USE_MOCK_PLM         - Set to 'true' to skip APS OAuth and use mock server
"""
import os
import sys

# Ensure project root is on path when invoked as subprocess
sys.path.insert(0, os.path.dirname(os.path.dirname(os.path.abspath(__file__))))

from mcp.server.fastmcp import FastMCP
from autodesk_plm_mcp.plm_client import FusionManageClient
from autodesk_plm_mcp.models import PLMSearchResult, PLMUpdateResult, PLMRecord

mcp = FastMCP(name="autodesk-plm-mcp")

# Workspace name → integer ID mapping (standard Fusion Manage defaults; override per tenant)
WORKSPACE_MAP: dict[str, int] = {
    "ECN": 57,
    "CR": 28,
    "PARTS": 14,
    "CO": 42,
    "CHANGE_ORDERS": 42,
    "CHANGE_REQUESTS": 28,
}

# Fields permitted for agent updates per workspace (whitelist — security boundary)
ALLOWED_FIELDS: dict[str, list[str]] = {
    "ECN": ["STATUS", "PRIORITY", "APPROVER", "COMMENTS", "IMPLEMENTATION_DATE"],
    "CR": ["STATUS", "PRIORITY", "DESCRIPTION", "COMMENTS"],
    "PARTS": ["STATUS", "COMMENTS"],
    "CO": ["STATUS", "APPROVER", "COMMENTS"],
    "CHANGE_ORDERS": ["STATUS", "APPROVER", "COMMENTS"],
    "CHANGE_REQUESTS": ["STATUS", "PRIORITY", "DESCRIPTION", "COMMENTS"],
}


def _client() -> FusionManageClient:
    mock_mode = os.environ.get("USE_MOCK_PLM", "false").lower() == "true"
    tenant_url = os.environ.get("APS_PLM_TENANT_URL", "http://localhost:5001")
    if mock_mode:
        tenant_url = f"http://localhost:{os.environ.get('MOCK_PLM_PORT', '5001')}"
    return FusionManageClient(
        tenant_url=tenant_url,
        client_id=os.environ.get("APS_CLIENT_ID", "mock"),
        client_secret=os.environ.get("APS_CLIENT_SECRET", "mock"),
        mock_mode=mock_mode,
    )


@mcp.tool()
def search_plm_record(workspace: str, identifier: str) -> PLMSearchResult:
    """
    Search Autodesk Fusion Manage for a PLM record by its identifier.

    Args:
        workspace: Workspace name — one of: ECN, CR, PARTS, CO
        identifier: The record number to search for (e.g. 'ECN-2024-0342', 'CR-2024-0287', '100-4821')

    Returns:
        PLMSearchResult with found=True and records list, or found=False if not found
    """
    ws_key = workspace.upper()
    ws_id = WORKSPACE_MAP.get(ws_key)
    if ws_id is None:
        return PLMSearchResult(
            found=False,
            workspace=workspace,
            identifier=identifier,
            records=[],
            message=f"Unknown workspace '{workspace}'. Valid values: {list(WORKSPACE_MAP.keys())}",
        )

    try:
        records = _client().search_items(workspace_id=ws_id, identifier=identifier)
    except Exception as e:
        return PLMSearchResult(
            found=False,
            workspace=workspace,
            identifier=identifier,
            records=[],
            message=f"PLM API error: {e}",
        )

    return PLMSearchResult(
        found=bool(records),
        workspace=workspace,
        identifier=identifier,
        records=records,
        total=len(records),
        message=f"Found {len(records)} record(s) for '{identifier}' in workspace '{workspace}'",
    )


@mcp.tool()
def get_plm_record(workspace: str, item_id: str) -> PLMRecord:
    """
    Retrieve the full details of a PLM record by its item ID.

    Args:
        workspace: Workspace name (ECN, CR, PARTS, CO)
        item_id: The numeric item ID string (e.g. '1452')

    Returns:
        PLMRecord with all fields, workflow state, and affected items
    """
    ws_key = workspace.upper()
    ws_id = WORKSPACE_MAP.get(ws_key)
    if ws_id is None:
        raise ValueError(f"Unknown workspace '{workspace}'")

    item = _client().get_item(workspace_id=ws_id, item_id=item_id)
    return PLMRecord(
        id=item.get("id", item_id),
        workspace_id=ws_id,
        number=item.get("number"),
        title=item.get("title"),
        status=item.get("status"),
        fields=item.get("fields", {}),
        affected_items=item.get("affectedItems", []),
        workflow=item.get("workflow", {}),
        links=item.get("links", {}),
    )


@mcp.tool()
def update_plm_record(
    workspace: str,
    item_id: str,
    fields: dict[str, str],
    comment: str | None = None,
) -> PLMUpdateResult:
    """
    Update one or more fields on a Fusion Manage PLM record.

    Args:
        workspace: Workspace name (ECN, CR, PARTS, CO)
        item_id: The numeric item ID (e.g. '1452')
        fields: Dict of FIELD_ID to new string value (e.g. {"STATUS": "Pending Approval"})
        comment: Optional comment to record in the item's activity log

    Returns:
        PLMUpdateResult with success status and list of updated field names
    """
    ws_key = workspace.upper()
    ws_id = WORKSPACE_MAP.get(ws_key)
    if ws_id is None:
        raise ValueError(f"Unknown workspace '{workspace}'")

    allowed = ALLOWED_FIELDS.get(ws_key, [])
    unauthorized = [f for f in fields if f not in allowed]
    if unauthorized:
        raise ValueError(
            f"Field(s) {unauthorized} are not permitted for update in workspace '{workspace}'. "
            f"Allowed fields: {allowed}"
        )

    try:
        _client().update_item(
            workspace_id=ws_id,
            item_id=item_id,
            fields=fields,
            comment=comment,
        )
        return PLMUpdateResult(
            success=True,
            workspace=workspace,
            item_id=item_id,
            updated_fields=list(fields.keys()),
            message=f"Successfully updated item {item_id} in workspace '{workspace}'",
        )
    except Exception as e:
        return PLMUpdateResult(
            success=False,
            workspace=workspace,
            item_id=item_id,
            updated_fields=[],
            message=f"Update failed: {e}",
        )


if __name__ == "__main__":
    mcp.run(transport="stdio")
```

Note: The module import paths use `autodesk_plm_mcp` package name. Create `autodesk-plm-mcp/__init__.py` (empty) and ensure the parent directory is on `sys.path`. Alternatively, flatten the import to relative in the same directory:

Simplified import version (no package prefix needed if run from `autodesk-plm-mcp/` dir):
Replace the import block with:
```python
from plm_client import FusionManageClient
from models import PLMSearchResult, PLMUpdateResult, PLMRecord
```
And run as: `python server.py` from within `autodesk-plm-mcp/` directory.

Success criteria:
* `python autodesk-plm-mcp/server.py` starts without error and waits for MCP stdio messages
* The 3 tools are registered: `search_plm_record`, `get_plm_record`, `update_plm_record`

Dependencies:
* Steps 3.1, 3.2 complete (models.py, plm_client.py)

---

### Step 3.4: Create autodesk-plm-mcp/requirements.txt

Create `/Users/alimurad/Desktop/projects/hack/autodesk-plm-mcp/requirements.txt`:

```
mcp[cli]>=1.6.0
httpx>=0.27.0
pydantic>=2.0.0
python-dotenv>=1.0.0
flask>=3.0.0
```

Success criteria:
* `pip install -r autodesk-plm-mcp/requirements.txt` succeeds

Dependencies:
* Step 1.1 complete

---

### Step 3.5: Validate phase changes

```bash
cd /Users/alimurad/Desktop/projects/hack
pip install -r autodesk-plm-mcp/requirements.txt
python -c "from autodesk_plm_mcp.models import PLMSearchResult; print('models OK')"
# Or if using flat imports from within the directory:
cd autodesk-plm-mcp && python -c "from models import PLMSearchResult; print('models OK')" && cd ..
```

---

## Implementation Phase 4: Agent Orchestration

<!-- parallelizable: true -->

### Step 4.1: Create agent/agent_prompt.py

Create `/Users/alimurad/Desktop/projects/hack/agent/agent_prompt.py`:

```python
"""System prompt for the Autodesk Engineering PLM Agent."""
from datetime import date

SYSTEM_PROMPT = f"""You are an Autodesk Engineering PLM Agent. Your job is to:

1. READ unprocessed engineering emails from the inbox that contain references to
   Autodesk Fusion Manage PLM records (ECNs, Change Requests, part numbers).

2. EXTRACT PLM identifiers from each email:
   - ECN numbers (pattern: ECN-YYYY-NNNN, e.g. ECN-2024-0342)
   - Change Request numbers (pattern: CR-YYYY-NNNN, e.g. CR-2024-0287)
   - Part numbers (pattern: NNN-NNNN where NNN is 100-600, e.g. 100-4821)

3. LOOK UP each identifier in Fusion Manage PLM using the search_plm_record tool.

4. DETERMINE the appropriate action based on the email content:
   - Approval requests → update ECN status to "Pending Approval"
   - Status change notifications → verify and confirm the status in PLM
   - Obsolescence alerts → set affected ECN status to "On Hold" and add a comment explaining the obsolete part
   - Digest summaries → compile an overview but do NOT update records

5. UPDATE PLM records using update_plm_record ONLY when:
   - An approval email explicitly routes an ECN for approval
   - A status change email reflects a confirmed change to mirror in PLM
   - A part obsolescence alert impacts an open ECN
   NEVER update records when uncertain — flag for human review instead.

6. NOTIFY the Engineering team on Microsoft Teams:
   - Find the "Engineering" team using list_teams
   - Find the "PLM Updates" channel using list_channels
   - Post a markdown summary using send_channel_message
   - Include: ECN number, title, new status, affected parts, source email sender/date
   - Tag the approver with an @mention when status reaches "Pending Approval"
   - Use importance "high" only for urgent items (URGENT in subject or critical priority)
   - Always include the Fusion Manage deep link from the record's links.plmUrl field

7. MARK each processed email as read using update-mail-message after handling it.

RULES:
- Process only emails with isRead=false
- If no PLM identifiers found in an email, skip it silently
- If a PLM record is not found, include "Record not found" in the Teams notification
- If an email references multiple ECNs, process each one independently
- For digest emails, post a single summary Teams message for all open ECNs mentioned
- Log each action in your final response summary

CURRENT DATE: {date.today().isoformat()}
PLM TENANT URL: acme.autodeskplm360.net (or mock at localhost:5001)
ENGINEERING TEAM: discover at runtime using list_teams
PLM CHANNEL: PLM Updates
"""
```

Success criteria:
* File is valid Python, importable
* `SYSTEM_PROMPT` is a string with current date interpolated

Dependencies:
* Step 1.1 complete

---

### Step 4.2: Create agent/main.py

Create `/Users/alimurad/Desktop/projects/hack/agent/main.py`:

```python
"""
AI Agent Orchestration: Email → PLM → Teams

Connects three MCP servers as stdio subprocesses and runs an agentic loop
using Azure OpenAI GPT-4o with function calling.

MCP Servers:
  1. @softeria/ms-365-mcp-server (Office 365 email read)
  2. autodesk-plm-mcp/server.py  (Autodesk Fusion Manage PLM)
  3. @floriscornel/teams-mcp      (Microsoft Teams notifications)

Prerequisites:
  - cp .env.example .env && fill in values
  - python autodesk-plm-mcp/mock_server.py &   (if USE_MOCK_PLM=true)
  - pip install -r agent/requirements.txt
  - npm install -g @softeria/ms-365-mcp-server @floriscornel/teams-mcp

Run:
  cd /path/to/hack && python agent/main.py
"""
import asyncio
import json
import os
import sys
from pathlib import Path

# Ensure project root is on path
sys.path.insert(0, str(Path(__file__).parent.parent))

from dotenv import load_dotenv
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client
from openai import AzureOpenAI

load_dotenv()

from agent.agent_prompt import SYSTEM_PROMPT

# ── Azure OpenAI configuration ──────────────────────────────────────────────
AZURE_OAI_ENDPOINT = os.environ["AZURE_OPENAI_ENDPOINT"]
AZURE_OAI_KEY = os.environ["AZURE_OPENAI_API_KEY"]
AZURE_OAI_DEPLOYMENT = os.environ.get("AZURE_OPENAI_DEPLOYMENT", "gpt-4o")

# ── MCP server parameters ────────────────────────────────────────────────────
MS365_PARAMS = StdioServerParameters(
    command="npx",
    args=["-y", "@softeria/ms-365-mcp-server", "--preset", "mail", "--org-mode"],
    env={
        **os.environ,
        "MS365_MCP_OAUTH_TOKEN": os.environ.get("MS365_OAUTH_TOKEN", ""),
    },
)

PLM_PARAMS = StdioServerParameters(
    command="python",
    args=[str(Path(__file__).parent.parent / "autodesk-plm-mcp" / "server.py")],
    env={
        **os.environ,
        "APS_PLM_TENANT_URL": os.environ.get(
            "APS_PLM_TENANT_URL",
            f"http://localhost:{os.environ.get('MOCK_PLM_PORT', '5001')}",
        ),
        "APS_CLIENT_ID": os.environ.get("APS_CLIENT_ID", "mock"),
        "APS_CLIENT_SECRET": os.environ.get("APS_CLIENT_SECRET", "mock"),
        "USE_MOCK_PLM": os.environ.get("USE_MOCK_PLM", "true"),
        "MOCK_PLM_PORT": os.environ.get("MOCK_PLM_PORT", "5001"),
    },
)

TEAMS_PARAMS = StdioServerParameters(
    command="npx",
    args=["-y", "@floriscornel/teams-mcp@latest"],
    env={
        **os.environ,
        "AUTH_TOKEN": os.environ.get("TEAMS_AUTH_TOKEN", ""),
    },
)


def _mcp_tool_to_openai(prefix: str, tool) -> dict:
    """Convert an MCP tool definition to an OpenAI function definition."""
    return {
        "type": "function",
        "function": {
            "name": f"{prefix}__{tool.name}",
            "description": tool.description or f"MCP tool: {tool.name}",
            "parameters": tool.inputSchema or {"type": "object", "properties": {}},
        },
    }


async def gather_tools(
    sessions: list[tuple[ClientSession, str]]
) -> tuple[list[dict], dict[str, tuple[ClientSession, str]]]:
    """Gather all tools from all MCP sessions. Returns (openai_tools, name_to_session_map)."""
    all_tools: list[dict] = []
    session_map: dict[str, tuple[ClientSession, str]] = {}

    for session, prefix in sessions:
        resp = await session.list_tools()
        for tool in resp.tools:
            qualified_name = f"{prefix}__{tool.name}"
            all_tools.append(_mcp_tool_to_openai(prefix, tool))
            session_map[qualified_name] = (session, tool.name)

    print(f"[Agent] Loaded {len(all_tools)} tools from {len(sessions)} MCP servers")
    return all_tools, session_map


async def execute_tool_call(
    session_map: dict[str, tuple[ClientSession, str]],
    tool_name: str,
    tool_args: dict,
) -> str:
    """Execute a single tool call via its MCP session and return the text result."""
    entry = session_map.get(tool_name)
    if entry is None:
        return json.dumps({"error": f"Unknown tool: {tool_name}"})

    session, original_name = entry
    result = await session.call_tool(original_name, tool_args)

    if not result.content:
        return json.dumps({"result": "empty"})
    return result.content[0].text if hasattr(result.content[0], "text") else str(result.content[0])


async def run_agent() -> None:
    """Main agentic loop: connect MCP servers, gather tools, run LLM tool-call loop."""
    print("[Agent] Connecting to MCP servers...")

    async with (
        stdio_client(MS365_PARAMS) as (m365_r, m365_w),
        stdio_client(PLM_PARAMS) as (plm_r, plm_w),
        stdio_client(TEAMS_PARAMS) as (teams_r, teams_w),
    ):
        async with (
            ClientSession(m365_r, m365_w) as m365_session,
            ClientSession(plm_r, plm_w) as plm_session,
            ClientSession(teams_r, teams_w) as teams_session,
        ):
            await m365_session.initialize()
            await plm_session.initialize()
            await teams_session.initialize()
            print("[Agent] All MCP servers initialized")

            sessions = [
                (m365_session, "ms365"),
                (plm_session, "plm"),
                (teams_session, "teams"),
            ]
            all_tools, session_map = await gather_tools(sessions)

            # ── Azure OpenAI client ──────────────────────────────────────────
            oai_client = AzureOpenAI(
                azure_endpoint=AZURE_OAI_ENDPOINT,
                api_key=AZURE_OAI_KEY,
                api_version="2024-12-01-preview",
            )

            messages: list[dict] = [
                {"role": "system", "content": SYSTEM_PROMPT},
                {
                    "role": "user",
                    "content": (
                        "Process all unread PLM-related engineering emails in the inbox. "
                        "For each relevant email: look up the referenced Autodesk Fusion Manage "
                        "record, apply the appropriate PLM update, then post a summary message "
                        "to the Engineering team's PLM Updates channel on Microsoft Teams."
                    ),
                },
            ]

            print("[Agent] Starting agentic run...\n")
            iteration = 0
            max_iterations = 30  # safety cap

            while iteration < max_iterations:
                iteration += 1
                response = oai_client.chat.completions.create(
                    model=AZURE_OAI_DEPLOYMENT,
                    messages=messages,
                    tools=all_tools,
                    tool_choice="auto",
                )

                choice = response.choices[0]
                msg = choice.message
                messages.append(msg.model_dump(exclude_none=True))

                if choice.finish_reason == "stop":
                    print("\n" + "═" * 60)
                    print("AGENT RUN COMPLETE")
                    print("═" * 60)
                    print(msg.content or "(no final message)")
                    break

                if choice.finish_reason == "tool_calls" and msg.tool_calls:
                    tool_results = []
                    for tc in msg.tool_calls:
                        tool_name = tc.function.name
                        try:
                            tool_args = json.loads(tc.function.arguments)
                        except json.JSONDecodeError:
                            tool_args = {}

                        print(f"\n→ Tool: {tool_name}")
                        print(f"  Args: {json.dumps(tool_args, indent=2)[:300]}")

                        result_text = await execute_tool_call(session_map, tool_name, tool_args)
                        print(f"  Result: {result_text[:200]}...")

                        tool_results.append({
                            "role": "tool",
                            "tool_call_id": tc.id,
                            "content": result_text,
                        })
                    messages.extend(tool_results)
                else:
                    print(f"[Agent] Unexpected finish_reason: {choice.finish_reason}")
                    break

            if iteration >= max_iterations:
                print("[Agent] Warning: reached max iterations without completion")


if __name__ == "__main__":
    asyncio.run(run_agent())
```

Success criteria:
* File is valid Python
* `python agent/main.py` starts and prints "[Agent] Connecting to MCP servers..."

Dependencies:
* Step 4.1 complete (agent_prompt.py), Step 3.3 complete (server.py)

---

### Step 4.3: Create agent/requirements.txt

Create `/Users/alimurad/Desktop/projects/hack/agent/requirements.txt`:

```
mcp[cli]>=1.6.0
openai>=1.35.0
python-dotenv>=1.0.0
azure-identity>=1.17.0
```

Note: `azure-identity` is included for future migration to Foundry Agent Service (WI-01), though not used in the current implementation which uses API key auth.

Success criteria:
* `pip install -r agent/requirements.txt` succeeds

Dependencies:
* Step 1.1 complete

---

### Step 4.4: Validate phase changes

```bash
cd /Users/alimurad/Desktop/projects/hack
pip install -r agent/requirements.txt
python -c "from agent.agent_prompt import SYSTEM_PROMPT; print('prompt OK:', SYSTEM_PROMPT[:60])"
python -c "import agent.main; print('main OK')"
```

---

## Implementation Phase 5: Integration Wiring and Auth Setup

<!-- parallelizable: false -->

### Step 5.1: Install npm MCP servers and Python packages

```bash
# Install npm MCP servers globally
npm install -g @softeria/ms-365-mcp-server @floriscornel/teams-mcp

# Install all Python packages
pip install -r autodesk-plm-mcp/requirements.txt -r agent/requirements.txt
```

Success criteria:
* `npx @softeria/ms-365-mcp-server --version` succeeds
* `npx @floriscornel/teams-mcp --version` (or --help) succeeds

Dependencies:
* Node.js 20+ installed, Python 3.11+ installed

---

### Step 5.2: Obtain Microsoft 365 OAuth token

Run the device code login flow for the M365 MCP server. This is a one-time interactive step.

```bash
# Triggers browser-based device code authentication
# After login, token is cached in OS keychain
npx @softeria/ms-365-mcp-server --login --preset mail --org-mode
```

After login, extract the cached token for use in .env:
```bash
# The server caches tokens via MSAL; for BYOT (pipeline mode), retrieve with:
npx @softeria/ms-365-mcp-server --get-token --preset mail
```

Copy the token value to `.env` as `MS365_OAUTH_TOKEN=<token>`.

Required Graph API permissions (configure in Azure AD app registration):
* `Mail.Read` (delegated)
* `Mail.ReadWrite` (delegated — for marking emails as read)
* `User.Read` (delegated)
* `offline_access` (for token refresh)

Success criteria:
* Token obtained and set in .env
* `MS365_OAUTH_TOKEN` is non-empty in .env

Dependencies:
* Microsoft 365 tenant with an Azure AD app registration (or use Softeria's default app registration via device code)

---

### Step 5.3: Obtain Microsoft Teams OAuth token

```bash
# Triggers device code authentication for Teams
npx @floriscornel/teams-mcp --login
```

Copy the token to `.env` as `TEAMS_AUTH_TOKEN=<token>`.

Required Graph API permissions (configure in Azure AD app registration):
* `ChannelMessage.Send` (delegated)
* `Team.ReadBasic.All` (delegated)
* `Channel.ReadBasic.All` (delegated)
* `User.Read` (delegated)

Ensure a "PLM Updates" channel exists in an "Engineering" team in the tenant.

Success criteria:
* Token obtained and set in .env
* "PLM Updates" channel exists in Teams (create manually if not present)

Dependencies:
* Microsoft Teams tenant with at least one team and one channel

---

### Step 5.4: Configure .env, seed sample emails, and start mock PLM server

Copy .env.example to .env and fill in credentials:
```bash
cp .env.example .env
# Edit .env: set AZURE_OPENAI_ENDPOINT, AZURE_OPENAI_API_KEY, MS365_OAUTH_TOKEN, TEAMS_AUTH_TOKEN
# For mock mode: USE_MOCK_PLM=true (no APS credentials needed)
```

**Seed sample emails into the M365 inbox (required for demo):**

Because `@softeria/ms-365-mcp-server` reads from the real Microsoft Graph API, the 4 sample emails in `mock_data/emails.json` must exist as real emails in the inbox. Seed them by sending (or drafting + sending) the following emails to the mailbox associated with MS365_OAUTH_TOKEN:

1. Subject: `ECN-2024-0342: Engineering Change Notice Requires Your Approval` — paste the HTML body from `mock_data/emails.json[0].body.content`
2. Subject: `[PLM Notification] Change Request CR-2024-0301 Status Updated` — paste text body from `emails.json[1]`
3. Subject: `URGENT: Part 300-1204 obsoleted — downstream impact on ECN-2024-0355` — paste HTML body from `emails.json[2]`
4. Subject: `[Weekly Digest] Open ECNs Awaiting Action — 4 items` — paste text body from `emails.json[3]`

Leave all 4 emails **unread** in the inbox before running the agent.

Start the mock PLM server in a separate terminal:
```bash
python autodesk-plm-mcp/mock_server.py
# Verify: curl http://localhost:5001/api/v3/health → {"status":"ok"}
```

Success criteria:
* .env has all required values
* 4 sample emails are unread in the M365 inbox
* Mock PLM server responds to health check at `http://localhost:5001/api/v3/health`

Dependencies:
* Steps 5.1–5.3 complete

---

### Step 5.5: Run end-to-end demo

```bash
cd /Users/alimurad/Desktop/projects/hack
python agent/main.py
```

Expected output sequence:
```
[Agent] Connecting to MCP servers...
[Agent] All MCP servers initialized
[Agent] Loaded N tools from 3 MCP servers
[Agent] Starting agentic run...

→ Tool: ms365__list-mail-messages
  Args: {"folderId": "inbox", ...}
  Result: [{"id": "AAMk...", "subject": "ECN-2024-0342...

→ Tool: plm__search_plm_record
  Args: {"workspace": "ECN", "identifier": "ECN-2024-0342"}
  Result: {"found": true, "records": [...

→ Tool: plm__update_plm_record
  Args: {"workspace": "ECN", "item_id": "1452", "fields": {"STATUS": "Pending Approval"}}
  Result: {"success": true ...

→ Tool: teams__list_teams
→ Tool: teams__list_channels
→ Tool: teams__send_channel_message

══════════════════════════════════════════════
AGENT RUN COMPLETE
══════════════════════════════════════════════
Processed 1 PLM-related email:
- ECN-2024-0342 status updated to "Pending Approval"
- Teams notification sent to Engineering / PLM Updates
```

Success criteria:
* All 9 tool calls execute without error
* ECN-2024-0342 record shows STATUS=Pending Approval in mock server
* Teams channel receives a markdown summary message

Dependencies:
* All Phase 1–4 steps complete, .env filled, mock server running

---

## Implementation Phase 6: Final Validation

<!-- parallelizable: false -->

### Step 6.1: Run full project validation

```bash
cd /Users/alimurad/Desktop/projects/hack

# Verify Python syntax on all project files
python -m py_compile autodesk-plm-mcp/models.py
python -m py_compile autodesk-plm-mcp/plm_client.py
python -m py_compile autodesk-plm-mcp/server.py
python -m py_compile autodesk-plm-mcp/mock_server.py
python -m py_compile agent/agent_prompt.py
python -m py_compile agent/main.py

# Validate JSON files
python -c "import json; json.load(open('autodesk-plm-mcp/mock_data/emails.json')); print('emails.json OK')"
python -c "import json; json.load(open('autodesk-plm-mcp/mock_data/ecn_records.json')); print('ecn_records.json OK')"
python -c "import json; json.load(open('autodesk-plm-mcp/mock_data/cr_records.json')); print('cr_records.json OK')"
python -c "import json; json.load(open('autodesk-plm-mcp/mock_data/part_records.json')); print('part_records.json OK')"
python -c "import json; json.load(open('mcp_config.json')); print('mcp_config.json OK')"
```

Success criteria:
* All `py_compile` calls exit with code 0
* All JSON files parse without error

---

### Step 6.2: Fix minor validation issues

Address any syntax errors, import path issues, or JSON malformation found in Step 6.1.

Common issues:
* Import paths: use relative imports in `server.py` (`from models import...`, `from plm_client import...`) when running from `autodesk-plm-mcp/` directory
* Python version: ensure `str | None` syntax requires Python 3.10+; use `Optional[str]` for 3.9 compatibility
* JSON trailing commas: not valid JSON — verify with `python -m json.tool <file>`

---

### Step 6.3: Report blocking issues

If end-to-end run (Step 5.5) fails at a specific tool call:

* M365 `list-mail-messages` fails with 401: MS365_OAUTH_TOKEN is expired — re-run `npx @softeria/ms-365-mcp-server --login`
* PLM `search_plm_record` fails with ConnectionError: mock server is not running — start `python autodesk-plm-mcp/mock_server.py`
* Teams `send_channel_message` fails with 403: `ChannelMessage.Send` permission missing from Azure AD app registration
* Teams `list_teams` returns empty: TEAMS_AUTH_TOKEN is expired or user has no joined teams

---

## Dependencies

* Python 3.10+
* Node.js 20+
* Azure OpenAI resource with GPT-4o deployment
* Microsoft 365 tenant (any M365 license — E1, E3, E5, or Business)
* Microsoft Teams tenant with "Engineering" team and "PLM Updates" channel
* Azure AD app registration for M365/Teams OAuth (or device code flow with default app)
* Autodesk APS app registration (optional — skip if `USE_MOCK_PLM=true`)

## Success Criteria

* All 6 Python source files pass `py_compile`
* All 4 JSON mock data files are valid JSON
* Mock PLM server responds at `http://localhost:5001/api/v3/health`
* `python agent/main.py` executes all 9 tool calls: 3 M365 + 3 PLM + 3 Teams
* Teams "PLM Updates" channel receives a markdown message with ECN-2024-0342 summary
* ECN-2024-0342 status updated to "Pending Approval" in mock PLM server
