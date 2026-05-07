---
applyTo: '.copilot-tracking/changes/2026-05-04/email-to-plm-agent-changes.md'
---
<!-- markdownlint-disable-file -->
# Implementation Plan: Email-to-PLM Agent

## Overview

Build a greenfield AI agent that reads unread engineering emails via Office 365 MCP, searches and updates Autodesk Fusion Manage PLM records, and posts a markdown summary to a Microsoft Teams channel — orchestrated by an Azure OpenAI GPT-4o agentic loop with three stdio MCP servers.

## Objectives

### User Requirements

* Read user emails via Office MCP server — Source: User request, conversation context
* Search for related Autodesk PLM data based on email contents — Source: User request
* Update Autodesk PLM data based on email contents — Source: User request
* Send a Teams channel message summarizing the change — Source: User request
* Use mock/sample emails and Autodesk PLM data for demonstration — Source: User request
* All integration touch points working as expected — Source: User success criteria

### Derived Objectives

* Build a custom FastMCP Python server wrapping Autodesk Fusion Manage REST API v3 — Derived from: No existing Autodesk PLM MCP server exists (research finding DR-04, autodesk-plm-research.md)
* Provide a Flask mock server for Fusion Manage API — Derived from: No public Autodesk demo tenant available; paid subscription required for real Fusion Manage (autodesk-plm-research.md Section 8)
* Use community MCP servers (@softeria/ms-365-mcp-server, @floriscornel/teams-mcp) — Derived from: Official Agent365 MCP servers require Microsoft Frontier program tenant access (log DD-01, azure-ai-agent-research.md)
* Orchestrate with direct Azure OpenAI + `mcp` Python client (stdio) instead of Foundry Agent Service MCPTool — Derived from: Community MCP servers run as stdio; Foundry MCPTool requires HTTP/SSE endpoints (log DD-01, DD-02)

## Context Summary

### Project Files

* /Users/alimurad/Desktop/projects/hack/ — workspace root; greenfield project, no existing source files

### References

* .copilot-tracking/research/2026-05-04/email-to-plm-agent-research.md — primary research, selected approach, 9-step tool call sequence, system prompt, mock data overview
* .copilot-tracking/research/subagents/2026-05-04/integration-architecture-research.md — complete Python code for all files, mock data JSON, Flask server, Teams notification format, error handling patterns
* .copilot-tracking/research/subagents/2026-05-04/autodesk-plm-research.md — Fusion Manage REST API v3 endpoints, JSON:API PATCH format, APS 2-legged OAuth, data model
* .copilot-tracking/research/subagents/2026-05-04/office-mcp-email-research.md — @softeria/ms-365-mcp-server tool schemas, auth flows (BYOT, device code), Graph API permissions
* .copilot-tracking/research/subagents/2026-05-04/teams-mcp-research.md — @floriscornel/teams-mcp tool schemas (send_channel_message, list_teams, list_channels), permissions
* .copilot-tracking/research/subagents/2026-05-04/azure-ai-agent-research.md — Foundry Agent Service overview, MCP integration, Azure OpenAI SDK patterns
* https://github.com/Softeria/ms-365-mcp-server — M365 MCP server source and docs
* https://github.com/floriscornel/teams-mcp — Teams MCP server source and docs
* https://help.autodesk.com/view/PLM/ENU/?guid=FLC_RestAPI_Read_Me_First_html — Fusion Manage REST API v3 reference
* https://learn.microsoft.com/en-us/azure/foundry/agents/overview — Foundry Agent Service (future migration path)

### Standards References

* .copilot-tracking/plans/logs/2026-05-04/email-to-plm-agent-log.md — implementation paths, discrepancy log, follow-on work items

## Implementation Checklist

### [ ] Implementation Phase 1: Project Scaffolding

<!-- parallelizable: false -->

* [ ] Step 1.1: Create directory structure (agent/, autodesk-plm-mcp/, autodesk-plm-mcp/mock_data/)
  * Details: .copilot-tracking/details/2026-05-04/email-to-plm-agent-details.md (Lines 23-44)
* [ ] Step 1.2: Create .env.example with all required environment variable placeholders
  * Details: .copilot-tracking/details/2026-05-04/email-to-plm-agent-details.md (Lines 46-92)
* [ ] Step 1.3: Create mcp_config.json with all 3 MCP server definitions
  * Details: .copilot-tracking/details/2026-05-04/email-to-plm-agent-details.md (Lines 94-137)
* [ ] Step 1.4: Validate scaffold
  * Details: .copilot-tracking/details/2026-05-04/email-to-plm-agent-details.md (Lines 139-143)

---

### [ ] Implementation Phase 2: Mock Data and Mock PLM Server

<!-- parallelizable: true -->

* [ ] Step 2.1: Create autodesk-plm-mcp/mock_data/emails.json (4 sample Autodesk engineering emails)
  * Details: .copilot-tracking/details/2026-05-04/email-to-plm-agent-details.md (Lines 149-269)
* [ ] Step 2.2: Create autodesk-plm-mcp/mock_data/ecn_records.json (3 ECN records)
  * Details: .copilot-tracking/details/2026-05-04/email-to-plm-agent-details.md (Lines 271-376)
* [ ] Step 2.3: Create autodesk-plm-mcp/mock_data/cr_records.json (2 CR records)
  * Details: .copilot-tracking/details/2026-05-04/email-to-plm-agent-details.md (Lines 378-435)
* [ ] Step 2.4: Create autodesk-plm-mcp/mock_data/part_records.json (3 part records including obsolete 300-1204)
  * Details: .copilot-tracking/details/2026-05-04/email-to-plm-agent-details.md (Lines 437-508)
* [ ] Step 2.5: Create autodesk-plm-mcp/mock_server.py (Flask REST API mimicking Fusion Manage v3)
  * Details: .copilot-tracking/details/2026-05-04/email-to-plm-agent-details.md (Lines 510-669)
* [ ] Step 2.6: Validate — start mock server, run health check, verify search endpoint
  * Details: .copilot-tracking/details/2026-05-04/email-to-plm-agent-details.md (Lines 671-683)

---

### [ ] Implementation Phase 3: Custom Autodesk PLM MCP Server

<!-- parallelizable: true -->

* [ ] Step 3.1: Create autodesk-plm-mcp/models.py (Pydantic models: PLMSearchResult, PLMUpdateResult, PLMRecord)
  * Details: .copilot-tracking/details/2026-05-04/email-to-plm-agent-details.md (Lines 689-734)
* [ ] Step 3.2: Create autodesk-plm-mcp/plm_client.py (APS OAuth 2-legged HTTP client with mock mode and X-user-id header for Fusion Manage)
  * Details: .copilot-tracking/details/2026-05-04/email-to-plm-agent-details.md (Lines 736-893)
* [ ] Step 3.3: Create autodesk-plm-mcp/server.py (FastMCP server: search_plm_record, get_plm_record, update_plm_record)
  * Details: .copilot-tracking/details/2026-05-04/email-to-plm-agent-details.md (Lines 887-1108)
* [ ] Step 3.4: Create autodesk-plm-mcp/requirements.txt
  * Details: .copilot-tracking/details/2026-05-04/email-to-plm-agent-details.md (Lines 1110-1128)
* [ ] Step 3.5: Validate — pip install, syntax check, confirm 3 tools registered
  * Details: .copilot-tracking/details/2026-05-04/email-to-plm-agent-details.md (Lines 1130-1140)

---

### [ ] Implementation Phase 4: Agent Orchestration

<!-- parallelizable: true -->

* [ ] Step 4.1: Create agent/agent_prompt.py (system prompt with workflow rules, current date)
  * Details: .copilot-tracking/details/2026-05-04/email-to-plm-agent-details.md (Lines 1146-1211)
* [ ] Step 4.2: Create agent/main.py (Azure OpenAI + mcp Python client agentic loop)
  * Details: .copilot-tracking/details/2026-05-04/email-to-plm-agent-details.md (Lines 1213-1455)
* [ ] Step 4.3: Create agent/requirements.txt (mcp[cli], openai, python-dotenv, azure-identity)
  * Details: .copilot-tracking/details/2026-05-04/email-to-plm-agent-details.md (Lines 1457-1476)
* [ ] Step 4.4: Validate — pip install, syntax check, confirm agent/main.py importable
  * Details: .copilot-tracking/details/2026-05-04/email-to-plm-agent-details.md (Lines 1478-1487)

---

### [ ] Implementation Phase 5: Integration Wiring and Auth Setup

<!-- parallelizable: false -->

* [ ] Step 5.1: Install npm MCP servers globally and pip packages
  * Details: .copilot-tracking/details/2026-05-04/email-to-plm-agent-details.md (Lines 1493-1510)
* [ ] Step 5.2: Obtain Microsoft 365 delegated OAuth token via device code flow
  * Details: .copilot-tracking/details/2026-05-04/email-to-plm-agent-details.md (Lines 1512-1543)
* [ ] Step 5.3: Obtain Microsoft Teams delegated OAuth token via device code flow
  * Details: .copilot-tracking/details/2026-05-04/email-to-plm-agent-details.md (Lines 1545-1569)
* [ ] Step 5.4: Configure .env, seed 4 sample emails into M365 inbox (unread), and start mock PLM server
  * Details: .copilot-tracking/details/2026-05-04/email-to-plm-agent-details.md (Lines 1571-1600)
* [ ] Step 5.5: Run end-to-end demo and verify all 9 tool calls execute
  * Details: .copilot-tracking/details/2026-05-04/email-to-plm-agent-details.md (Lines 1595-1641)

---

### [ ] Implementation Phase 6: Final Validation

<!-- parallelizable: false -->

* [ ] Step 6.1: Run full project validation (py_compile all .py files, validate all JSON files)
  * Details: .copilot-tracking/details/2026-05-04/email-to-plm-agent-details.md (Lines 1647-1672)
* [ ] Step 6.2: Fix minor validation issues (import paths, Python version compatibility, JSON syntax)
  * Details: .copilot-tracking/details/2026-05-04/email-to-plm-agent-details.md (Lines 1674-1683)
* [ ] Step 6.3: Report blocking issues with root cause and next steps
  * Details: .copilot-tracking/details/2026-05-04/email-to-plm-agent-details.md (Lines 1685-1696)

## Planning Log

See .copilot-tracking/plans/logs/2026-05-04/email-to-plm-agent-log.md for:
* DD-01: Orchestration layer deviation (Foundry Agent Service → direct Azure OpenAI + mcp client)
* DD-02: No Azure Foundry project resource required
* DR-01 through DR-04: Unaddressed research items (webhooks, Adaptive Cards, APS free tenant, Foundry transport)
* IP-01 through IP-04: All implementation paths considered with selection rationale
* WI-01 through WI-05: Suggested follow-on work items

## Dependencies

* Python 3.10+ (for `str | None` union syntax and `mcp` library)
* Node.js 20+ (for npx MCP servers)
* Azure OpenAI resource with GPT-4o deployment (any Azure subscription; agent runs locally connecting to Azure OpenAI — satisfies Azure-hosted model requirement; full Azure-hosted orchestrator is WI-01)
* Microsoft 365 tenant with mailbox (any M365 license tier); 4 sample emails must be seeded into inbox before demo run
* Microsoft Teams tenant with an "Engineering" team and "PLM Updates" channel
* `pip install mcp[cli] openai python-dotenv flask httpx pydantic azure-identity`
* `npm install -g @softeria/ms-365-mcp-server @floriscornel/teams-mcp`
* Autodesk APS app credentials optional when USE_MOCK_PLM=true; `APS_USER_ID` required for real Fusion Manage API

## Success Criteria

* All 8 Python source files pass `python -m py_compile` — Traces to: User success criteria (all integration touch points working)
* All 4 JSON mock data files parse without error — Traces to: User requirement (mock/sample data provided)
* Mock PLM server returns ECN-2024-0342 record at `GET /api/v3/workspaces/57/items?filter[number]=ECN-2024-0342` — Traces to: User requirement (search related Autodesk PLM data)
* `python agent/main.py` executes all 9 tool calls (3 M365 + 3 PLM + 3 Teams) — Traces to: User success criteria (all integration touch points working)
* Microsoft Teams "PLM Updates" channel receives a markdown message with ECN number, title, new status, and affected parts — Traces to: User requirement (send Teams message with summary of change)
* ECN-2024-0342 mock PLM record shows STATUS=Pending Approval after agent run — Traces to: User requirement (updates Autodesk PLM data based on email contents)
* All three MCP servers (M365, PLM, Teams) are demonstrably invoked with tool call logs visible in terminal output — Traces to: User success criteria (all integration touch points working as expected)
