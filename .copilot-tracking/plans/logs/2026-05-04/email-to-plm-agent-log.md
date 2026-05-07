<!-- markdownlint-disable-file -->
# Planning Log: Email-to-PLM Agent

## Discrepancy Log

### Unaddressed Research Items

* DR-01: Microsoft Graph Webhooks for real-time email triggers
  * Source: .copilot-tracking/research/2026-05-04/email-to-plm-agent-research.md (Potential Next Research)
  * Reason: Timer-based polling is sufficient for hackathon demo; webhooks add Graph subscription management complexity
  * Impact: low

* DR-02: Adaptive Cards for Teams notification formatting
  * Source: .copilot-tracking/research/2026-05-04/email-to-plm-agent-research.md (Potential Next Research)
  * Reason: Markdown formatting is supported by `@floriscornel/teams-mcp` `send_channel_message` and is sufficient for demo
  * Impact: low

* DR-03: Autodesk Webhooks API for event-driven PLM change detection
  * Source: .copilot-tracking/research/2026-05-04/email-to-plm-agent-research.md (Potential Next Research)
  * Reason: Out of scope for initial demo — agent reads email as the trigger, not PLM events
  * Impact: low

* DR-04: Foundry Agent Service MCPTool HTTP/SSE transport requirement
  * Source: .copilot-tracking/research/subagents/2026-05-04/azure-ai-agent-research.md (Section 1)
  * Reason: Research notes that Foundry MCPTool expects HTTP endpoints, but community MCP servers run as stdio. This plan uses direct Azure OpenAI function-calling with `mcp` Python client (stdio), which is simpler and demo-ready. Foundry Agent Service remains the production path documented as IP-02.
  * Impact: medium — selected approach (IP-01) is demo-correct; Foundry Agent Service is documented as follow-on

* DR-05: Azure-hosted agent orchestration not implemented
  * Source: .copilot-tracking/research/2026-05-04/email-to-plm-agent-research.md (Scope — "Azure-hosted agent orchestration"); User requirement 7 ("Azure Instance for AI Agent")
  * Reason: DD-02 accounts for the Foundry Agent Service deviation, but `agent/main.py` runs as a local Python process with no Azure deployment steps (no Azure Functions, Container Apps, or equivalent). Using Azure OpenAI as the model backend satisfies the "Azure" label partially, but the orchestrator itself is not Azure-hosted. WI-01 (Foundry Agent Service migration) would fully address this requirement post-demo.
  * Impact: medium — demo uses Azure OpenAI (cloud model) but agent execution is local; the "Azure-hosted" user requirement is only partially satisfied for hackathon purposes

* DR-06: No mock M365 email endpoint — sample emails require a real M365 inbox
  * Source: User requirement ("Use mock/sample emails and Autodesk PLM data"); .copilot-tracking/research/2026-05-04/email-to-plm-agent-research.md (Scope — "Mock/sample data acceptable for demo")
  * Reason: `emails.json` is created as reference/documentation data but is never served by a mock email endpoint. `@softeria/ms-365-mcp-server` reads from the real Microsoft Graph API inbox. For Step 5.5 to succeed with the sample identifiers (ECN-2024-0342, CR-2024-0301, ECN-2024-0355), a real M365 tenant must have those exact emails in its inbox. No plan step addresses seeding or injecting the sample emails into an M365 mailbox. Implementers must manually draft the 4 sample emails before running the agent.
  * Impact: high — blocks end-to-end demo if the implementer does not have an M365 tenant with the sample email subjects/identifiers; the mock/sample email user requirement is only satisfied for the PLM side (Flask mock server), not the email-read side

### Plan Deviations from Research

* DD-01: Orchestration layer is direct Azure OpenAI + `mcp` Python library, not Foundry Agent Service SDK
  * Research recommends: `azure-ai-projects` SDK with `PromptAgentDefinition` + `MCPTool`
  * Plan implements: `openai` Python SDK (Azure OpenAI) + `mcp` Python client (stdio subprocess management)
  * Rationale: Community MCP servers (`@softeria`, `@floriscornel`) run as stdio processes. Foundry Agent Service `MCPTool` requires HTTP/SSE endpoints. Converting both npm servers to HTTP mode adds complexity that is unnecessary for a hackathon demo. The `mcp` Python client natively manages stdio subprocesses and exposes tools as OpenAI function definitions, achieving the same result without an Azure Foundry project resource.

* DD-02: No Azure Foundry project resource required
  * Research recommends: Azure AI Foundry project + `AIProjectClient`
  * Plan implements: Azure OpenAI resource only (endpoint + key)
  * Rationale: Reduces Azure resource prerequisites to a single Azure OpenAI deployment. Any GPT-4o-capable Azure subscription works. The architecture still uses Azure OpenAI (same underlying model) and is cloud-hosted. Foundry Agent Service can replace this layer post-hackathon with minimal code changes.

* DD-03: `plm_client._headers()` omits `X-user-id` header required by Fusion Manage production API
  * Research recommends: Every Fusion Manage API call requires an `X-user-id` header containing the impersonated user email (.copilot-tracking/research/2026-05-04/email-to-plm-agent-research.md — Key Discoveries, Autodesk Platform Services section: "Extra Fusion Manage requirement: `X-user-id` header with impersonated user email on every call")
  * Plan implements: `_headers()` in `plm_client.py` returns only `Authorization`, `Content-Type`, and `Accept` — `X-user-id` is absent. No `APS_USER_ID` env variable is defined in `.env.example`.
  * Rationale: The Flask mock server ignores this header so the demo functions correctly in `USE_MOCK_PLM=true` mode. However, when switching to a real Fusion Manage tenant (WI-04), all API calls will return 401/403 until `X-user-id` is added to `_headers()` and sourced from a new `APS_USER_ID` environment variable.

---

## Implementation Paths Considered

### Selected: IP-01 — Direct Azure OpenAI + MCP Python Client (stdio)

* Approach: Three MCP servers run as stdio subprocesses managed by the Python `mcp` client library. Tools are extracted from each server at runtime and presented to Azure OpenAI GPT-4o as function definitions. The agent loop calls the LLM, executes tool calls via the MCP client, and feeds results back until completion.
* Rationale: Works with community MCP servers (stdio) without requiring HTTP/SSE conversion. No Azure Foundry project needed. Single Python entry point (`agent/main.py`). Demo-ready in under 1 hour of setup.
* Evidence: .copilot-tracking/research/subagents/2026-05-04/integration-architecture-research.md (Section 1, Section 8)

### IP-02: Microsoft Foundry Agent Service + HTTP MCP Servers (Production Path)

* Approach: Deploy all three MCP servers as HTTP/SSE endpoints (Azure Functions or Container Apps). Register them via `MCPTool` in `PromptAgentDefinition`. Use `azure-ai-projects` SDK for orchestration. Uses Managed Identity for credentials.
* Trade-offs: Fully managed, scalable, supports Entra Managed Identity, aligns with Microsoft production architecture. Requires Azure Foundry project resource, HTTP conversion for npm MCP servers (or community HTTP mode flags), and additional Azure resource provisioning.
* Rejection rationale: Requires more Azure setup time. npm MCP servers need `--http` mode testing. Not needed to demonstrate integration touch points for hackathon demo. Recommend as WI-01 follow-on.

### IP-03: Semantic Kernel + MCP Plugins

* Approach: `pip install semantic-kernel[mcp]` + `MCPSsePlugin` per server. Kernel manages tool dispatch.
* Trade-offs: Better fit for C# teams. Python SK MCP support is less mature than `mcp` client library. More boilerplate than direct MCP client approach.
* Rejection rationale: `mcp` Python client is simpler and language-native. No SK advantage for this project.

### IP-04: Official Agent365 MCP Servers (Microsoft Frontier)

* Approach: Use `mcp_MailTools` + `mcp_TeamsServer` at `agent365.svc.cloud.microsoft`
* Trade-offs: Official Microsoft-supported, enterprise-grade, integrates with Entra. Requires Frontier program tenant enrollment. Tool schemas not publicly documented. Cannot verify tool behavior without Frontier access.
* Rejection rationale: Frontier access gates this path entirely for hackathon demo.

---

## Suggested Follow-On Work

* WI-01: Migrate to Microsoft Foundry Agent Service (production path) — Convert MCP servers to HTTP/SSE transport, register via `MCPTool` in `PromptAgentDefinition`, add Entra Managed Identity. (priority: high)
  * Source: .copilot-tracking/research/subagents/2026-05-04/azure-ai-agent-research.md (Section 1)
  * Dependency: Working demo (this plan) complete first

* WI-02: Add Microsoft Graph change notifications (webhooks) to replace email polling — Subscribe to mailbox change notifications via Graph API, trigger agent on new email rather than polling. (priority: medium)
  * Source: .copilot-tracking/research/2026-05-04/email-to-plm-agent-research.md (Potential Next Research)
  * Dependency: WI-01 (Foundry Agent Service supports webhook triggers natively)

* WI-03: Replace markdown Teams messages with Adaptive Cards — Use Bot Framework or Graph API to post Adaptive Cards with structured PLM record data, status badges, and action buttons. (priority: low)
  * Source: .copilot-tracking/research/subagents/2026-05-04/teams-mcp-research.md (Q7 Alternatives)
  * Dependency: None

* WI-04: Connect to real Autodesk Fusion Manage tenant — Obtain APS app credentials, configure 2-legged OAuth, test against live Fusion Manage API. Auto-discover workspace IDs via `GET /api/v3/workspaces`. (priority: high for post-demo)
  * Source: .copilot-tracking/research/subagents/2026-05-04/autodesk-plm-research.md (Section 3)
  * Dependency: Autodesk APS app registration at https://aps.autodesk.com/myapps/

* WI-05: Add human-in-the-loop approval gate for PLM writes — Before `update_plm_record` executes, pause and request confirmation via Teams reply or email. (priority: medium)
  * Source: .copilot-tracking/research/subagents/2026-05-04/azure-ai-agent-research.md (Clarifying Questions)
  * Dependency: WI-01

---

## Validation Results

* Validated: 2026-05-04
* Status: **Gaps Found**
* Line-number cross-references: All 27 plan→details references verified accurate ✅
* User requirements coverage: 6 of 6 addressed (1 partial — Azure-hosted orchestrator, DR-05)
* New DR entries added: 2 (DR-05, DR-06)
* New DD entries added: 1 (DD-03)

### Severity-Ordered Findings

#### High — End-to-End Demo Blocker

**DR-06 · Mock M365 email endpoint not provided**

`emails.json` is created as reference data but is never served by a mock email endpoint. `@softeria/ms-365-mcp-server` reads from the real Microsoft Graph API inbox. Without a real M365 tenant that has the exact sample emails (ECN-2024-0342, CR-2024-0301, ECN-2024-0355 identifiers in subject/body), Step 5.5 cannot complete the email-read leg.

* Evidence: Plan creates `autodesk-plm-mcp/mock_data/emails.json` (Step 2.1) but all Phase 5 steps require a real M365 OAuth token and real inbox contents. No step seeds the sample emails.
* Remediation: Add a prerequisite step (before Step 5.5) instructing the implementer to manually draft the 4 sample emails into their M365 inbox using the subjects/identifiers from `emails.json`. Document this clearly in a README or demo guide.

---

#### Medium — Partial Requirement / Blocks Production Path

**DR-05 · Azure-hosted orchestrator not planned**

User requirement 7 specifies "Azure Instance for AI Agent (Azure-hosted)." `agent/main.py` runs as a local Python process; no Azure deployment step is included in the plan. Azure OpenAI as the model backend provides partial Azure integration but does not make the agent process "Azure-hosted."

* Evidence: All 6 implementation phases write files and run commands locally. DD-02 documents the Foundry deviation but does not address orchestrator hosting.
* Remediation: Add a note to `agent/main.py` that it can be wrapped in an Azure Function HTTP trigger as a near-term hosting option. Full resolution via WI-01 (Foundry Agent Service migration).

**DD-03 · `X-user-id` header missing from `plm_client._headers()`**

Research documents that every Fusion Manage API call requires a `X-user-id` header with an impersonated user email. `plm_client.py` omits this header and no `APS_USER_ID` env variable is defined. The Flask mock server does not enforce this header, so the hackathon demo succeeds. Real Fusion Manage API will return 401/403 for all requests without it.

* Evidence: Research — "Extra Fusion Manage requirement: `X-user-id` header with impersonated user email on every call." Plan `_headers()` returns `Authorization`, `Content-Type`, `Accept` only.
* Remediation: One-line fix — add `"X-user-id": os.getenv("APS_USER_ID", "")` to `_headers()` in `plm_client.py`; add `APS_USER_ID=<user-email>` to `.env.example`.

---

#### Minor — Documentation / Data Accuracy

**Phases 3 and 4 marked `parallelizable: true` with intra-phase file dependencies**

Phase 3 `server.py` imports from `models.py` and `plm_client.py`; Phase 4 `main.py` imports `agent_prompt.py`. If an implementation agent executes steps in parallel it may write importing files before dependencies exist. Files can be written correctly in any order (content is known), but the label is potentially misleading for automated agents.

* Remediation: No code change required. Clarify in a comment that `parallelizable: true` refers to file-write order only; imports are resolved at Python runtime, not write-time.

**ECN-2024-0338 STATUS value "In Design" not in standard ECN picklist**

`ecn_records.json` ECN-2024-0338 has `STATUS: "In Design"` which is absent from the allowedValues list defined on ECN-2024-0342 (`["Draft", "In Review", "Pending Approval", "Approved", "Released", "Rejected", "On Hold", "Obsolete"]`). "In Design" is a CR-workspace status value. The mock server or agent may behave unexpectedly when processing email 2 (CR-2024-0301 status notification) if it attempts to mirror the CR "In Design" status to this ECN record.

* Remediation: Change ECN-2024-0338's STATUS to `"In Review"` in `ecn_records.json`, matching the standard ECN workflow state. Email 2 describes a CR status change, not an ECN status change, so the agent's system prompt rules should prevent an ECN update in this case anyway.

---

### Clarifying Questions

1. **M365 tenant availability**: Does the implementer have an M365 tenant with a mailbox they can send test emails to? If not, a mock email-seeding step must be added to Phase 5 before the demo is executable end-to-end.
2. **Azure hosting intent**: Should the agent process be deployable to Azure as a first-class deliverable for the hackathon (e.g., Azure Function wrapper), or is local execution + Azure OpenAI sufficient to satisfy the "Azure-hosted" requirement?
3. **Real Autodesk APS credentials**: Is there any possibility of obtaining a trial Fusion Manage tenant (see aps.autodesk.com/pricing)? If so, the `X-user-id` fix (DD-03) and `APS_USER_ID` env variable should be promoted to Phase 5 before running against a real API.
