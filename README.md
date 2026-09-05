# 🔍 Sherlock — Recon & Discovery Agent

Sherlock is the **automated reconnaissance agent** in the [KATE](https://kate-gateway.rxfr6l.usa-e2.cloudhub.io) documentation network. Its job is to scan every corner of your integration estate — Anypoint Platform, GitHub, and GitBooks — and produce a structured asset catalog with Linear tracking tasks for every discovered application.

---

## What Sherlock Does

Given a trigger message (e.g. *"scan all apps"* or *"document order-api in Sandbox"*), Sherlock runs four sequential steps:

```
Trigger → Anypoint Scan → GitHub Scan → GitBooks Scan → Linear Task Creation → Done
```

### Step 1 — Anypoint Platform Scan
Calls the Anypoint Platform MCP to discover:
- **Deployed applications** across Sandbox, Production, and Development environments (`list_applications`)
- **API Manager instances** registered in the org (`list_api_instances`)
- **Exchange assets** related to the discovered apps (`find_assets`)

### Step 2 — GitHub Scan
For each app found in Anypoint, searches GitHub for matching Mule repositories:
- Locates source repositories by app name
- Reads `src/main/mule/` for Mule XML flows
- Reads `src/main/resources/dwl/` for DataWeave transformations
- Reads `README.md` for project documentation summaries

> If GitHub is unreachable or returns no results, Sherlock falls back to an empty result set and continues — it never blocks the pipeline on a GitHub failure.

### Step 3 — GitBooks Scan
Searches existing GitBooks documentation for each discovered app:
- Dynamically resolves the GitBooks organization ID
- Lists all published documentation sites
- Uses full-text search (`searchOrganizationContent`) to find pages matching each app name
- Reads matched pages to extract summaries and links

### Step 4 — Linear Task Creation
Creates one Linear issue per discovered application:
- **Title:** `[KATE] Document: <App Name>`
- **Description:** Markdown summary with Anypoint app details, GitHub repo links, GitBooks doc links, API versions, and environments
- **Priority:** Medium (3)
- Adds individual finding comments per API, repo, and doc page found

Returns the complete **asset catalog** with Linear task IDs for all downstream KATE agents.

---

## Architecture

```
sherlock-network/
├── brokers/
│   └── sherlock.agent          # AgentScript definition (AGENTFABRIC 1.1)
├── agent-network.yaml          # Agent network configuration
├── exchange.json               # Anypoint Exchange metadata
└── README.md                   # This file
```

Sherlock is defined as a single **AgentScript broker** (`sherlockAgentBroker`) exposed over the A2A protocol. It is deployed to CloudHub 2.0 and fronted by the KATE API Gateway.

### MCP Connections

| Connection | Purpose |
|---|---|
| `sherlockAnypointDxConnection` | List applications and APIs from Anypoint Platform |
| `sherlockAnypointApisCatalogConnection` | Search Exchange and view API details |
| `githubConnection` | Search repositories and read file contents |
| `gitbooksConnection` | List sites, search content, and read documentation pages |
| `linearConnection` | Create issues and post comments |

### Graph Nodes

| Node | Type | Description |
|---|---|---|
| `scanStartNotification` | echo | Sends "Sherlock is on the case..." status update |
| `anypointScan` | orchestrator | Discovers apps, APIs, and Exchange assets |
| `anypointScanRouter` | router | Routes on `status == "success"` |
| `githubScan` | orchestrator | Finds Mule repos and reads source files |
| `githubScanRouter` | router | Routes on `status == "success"` |
| `gitbooksScan` | orchestrator | Searches existing documentation |
| `gitbooksScanRouter` | router | Routes on `status == "success"` |
| `linearCreation` | orchestrator | Creates Linear tasks and comments |
| `linearCreationRouter` | router | Routes on `status == "success"` |
| `discoveryRouter` | router | Final routing to success or error |
| `discoverySuccess` | echo | Emits `TASK_STATE_COMPLETED` with full catalog |
| `discoveryError` | echo | Emits `TASK_STATE_FAILED` with partial data |

---

## Invoking Sherlock

Sherlock exposes an A2A endpoint. Send an HTTP POST with `A2A-Version: 1.0`:

```
POST OMNI_GATEWAY_URL/sherlockAgentBroker
A2A-Version: 1.0
Content-Type: application/json

{
    "jsonrpc": "2.0",
    "id": "req-123",
    "method": "SendStreamingMessage",
    "params": {
        "message": {
            "messageId": "msg-123",
            "role": "ROLE_USER",
            "parts": [
                {
                    "text": "Let's identify assets for salesforce-system-api deployed in Sandbox"
                }
            ]
        }
    }
} 
```

**Scoping examples:**

| Message | Sherlock behaviour |
|---|---|
| `"Scan all apps"` | Scans every app across all environments |
| `"Document order-api"` | Focuses on apps matching `order-api` |
| `"Scan Sandbox"` | Scans all apps in the Sandbox environment only |
| `"Document payment-service in Production"` | Targets `payment-service` in Production |

---

## Output

Sherlock streams A2A `TASK_STATE_WORKING` status updates as each step completes, then finishes with `TASK_STATE_COMPLETED` containing the full asset catalog:

```
🔍 Sherlock is on the case. Scanning Anypoint Platform, GitHub, and GitBooks...
✅ Anypoint Platform MCP scan complete. Found 3 apps and 5 APIs. Proceeding to GitHub scan...
✅ GitHub MCP scan complete. Found 2 repositories. Proceeding to GitBooks scan...
✅ GitBooks MCP scan complete. Found 1 documentation pages. Proceeding to Linear task creation...
✅ Linear MCP tasks created. 3 tasks created for team: KATE. Finalizing...
✅ Sherlock completed reconnaissance. Discovered 3 apps. Linear tasks created.
```

---

## LLM

Sherlock uses **`gpt-5.6-luna`** via the `omniLLM` connection (`llm://llmConnection`) routed through the KATE gateway.

---