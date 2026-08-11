# QA Copilot – Power Platform Solution

QA Copilot is a Microsoft Copilot Studio agent that acts as a QA engineering assistant. It analyzes
requirements, designs test scenarios / cases / data, performs defect & build RCA, narrates quality
metrics, and **hands off test execution to an Azure AI Foundry hosted agent** ("QAFoundry") that
generates and runs Playwright tests against a target application and reports results back through
Azure DevOps.

This repository packages the solution as an **unmanaged Power Platform solution** (`.zip`) that you
import into your own environment, plus the setup you need to make it run in **your** tenant.

> ⚠️ **This export is sanitized.** All environment-specific values (tenant ID, app registration
> client ID, Foundry endpoint) and the client **secret** have been removed and replaced with
> placeholders. You must supply your own — see [Setup](#setup).

---

## What's in the box

| Component | Type | Purpose |
|-----------|------|---------|
| **QA Copilot** | Copilot Studio agent (bot) | The orchestrator the user talks to |
| 7 Skills | Inline agent skills | `requirements-analysis`, `scenario-design`, `test-case-design`, `suite-optimization`, `test-data-design`, `rca-method`, `quality-narration` |
| **Azure DevOps MCP** | MCP tool | Reads/writes Azure DevOps work items & QA artifacts |
| **GitHub MCP** | MCP tool | QA-relevant repo context (issues, PRs, diffs) |
| **Calling Foundry Agent** | Cloud flow (Power Automate) | Sends a test case + target URL to the Foundry hosted agent (`test-executor`) via HTTP |
| **ADO Return Sync** | Cloud flow | Syncs results back from Azure DevOps |
| **Foundry back to Copilot Studio** | Cloud flow | Returns Foundry output to the agent |
| 6 Dataverse tables | Tables | `mre_userstory`, `mre_testcase`, `mre_testdataset`, `mre_runrecord`, `mre_reqfinding`, `mre_defectrca` |
| Canvas/code app | App | Supporting UI |
| `crbcf_FoundryAgentClientSecret` | Environment variable | Holds the Foundry app client secret used by the flow |

### How it fits together

```
User ──▶ QA Copilot (Copilot Studio)
             │  skills: analyze / design / RCA / narrate
             ├──▶ Azure DevOps MCP  (read/write work items)
             ├──▶ GitHub MCP        (repo context)
             └──▶ "Calling Foundry Agent" flow
                        │  HTTP POST { input: <test case + URL> }
                        ▼
                  Azure AI Foundry hosted agent (test-executor / "QAFoundry")
                        │  generates Playwright test, commits to Azure DevOps,
                        │  triggers BDD-Test-Execution pipeline
                        ▼
                  Results ──▶ ADO ──▶ "ADO Return Sync" / "Foundry back to Copilot Studio"
```

The Foundry hosted agent itself (its Python source, tools, and pipeline) is **not** part of this
solution — it lives in Azure AI Foundry and Azure DevOps. This solution is the **Copilot Studio +
Power Platform orchestration layer** that calls it.

---

## Prerequisites

You need access to all of the following in **your** tenant:

1. **Power Platform environment** with **Dataverse** enabled, and the **Power Platform / Copilot
   Studio** licenses/permissions to import solutions and create agents.
2. **Microsoft Copilot Studio** enabled in that environment.
3. **Azure AI Foundry** project with a **deployed hosted agent** that exposes the OpenAI-compatible
   `responses` endpoint (this solution was built against an agent named `test-executor`). See
   [Foundry agent requirements](#the-foundry-hosted-agent).
4. **Microsoft Entra app registration** (service principal) that the flow uses to authenticate to
   Foundry, **plus a client secret**.
5. **Azure DevOps** organization/project (used by the Foundry agent to commit tests and run the
   pipeline) — only required if you use the execution hand-off and return-sync flows.
6. A **GitHub** account/app if you want the GitHub MCP tool.
7. Permissions to **create role assignments** on the Foundry resource (to grant the app access).

---

## Setup

Follow these in order. Full detail is in [`docs/SETUP.md`](docs/SETUP.md); this is the summary.

### 1. Create the Entra app registration + secret

The "Calling Foundry Agent" flow authenticates to Foundry with **client credentials**
(`ActiveDirectoryOAuth`).

1. In **Entra ID → App registrations → New registration**, create an app (single tenant is fine).
2. Under **Certificates & secrets**, create a **client secret** and copy the **value** (not the ID).
3. Note the **Application (client) ID** and your **Directory (tenant) ID**.

### 2. Grant the app access to your Foundry agent

The app's service principal needs **data-plane** access to invoke the agent.

- Assign the **Azure AI User** (a.k.a. **Foundry User**) role to the app's service principal, scoped
  to your Azure AI Foundry (Cognitive Services) account.
- RBAC can take a few minutes to propagate.

### 3. Import the solution

1. Go to **[make.powerapps.com](https://make.powerapps.com)** → select your environment →
   **Solutions → Import solution**.
2. Upload [`solution/QACopilot_1_0_0_3.zip`](solution/QACopilot_1_0_0_3.zip).
3. When prompted, either **create new connections** or map **connection references** for:
   - Microsoft Dataverse
   - Azure DevOps MCP (`shared_adomcpserver`)
   - GitHub (`shared_github`)
   - Agent / Copilot connectors
4. When prompted for the **environment variable** `Foundry Agent Client Secret`
   (`crbcf_FoundryAgentClientSecret`), paste the **client secret value** from step 1.
   - If not prompted, set it after import under **Solutions → QA Copilot → Environment variables**.

### 4. Point the flow at *your* Foundry agent

The exported flow contains **placeholders** you must replace. Open the **Calling Foundry Agent**
flow → **HTTP** action and set:

| Field | Replace placeholder with |
|-------|--------------------------|
| **URI** | `https://<YOUR_FOUNDRY_RESOURCE>.services.ai.azure.com/api/projects/<YOUR_PROJECT>/agents/<YOUR_AGENT_NAME>/endpoint/protocols/openai/responses?api-version=2025-11-15-preview` |
| **Authentication → Tenant** | `<YOUR_AZURE_TENANT_ID>` |
| **Authentication → Client ID** | `<YOUR_APP_CLIENT_ID>` |
| **Authentication → Audience** | `https://ai.azure.com` (already set) |
| **Authentication → Secret** | leave as `@parameters('crbcf_FoundryAgentClientSecret')` |

> Keep the **secret** field bound to the environment variable — do **not** paste the raw secret into
> the flow. See [Credentials & the secret](#credentials--the-secret).

### 5. Configure the MCP tool connections

In Copilot Studio, open the **QA Copilot** agent → **Tools** and finish authenticating:
- **Azure DevOps MCP** – connect with an account that can read your ADO project.
- **GitHub MCP** – connect your GitHub account/app.

### 6. Publish & test

1. **Publish** the QA Copilot agent in Copilot Studio.
2. Ensure all three flows are **turned on**.
3. In the agent's **Test** pane, ask it to analyze a work item and hand off a test case for
   execution. The agent calls the flow, which calls the Foundry agent.

---

## Credentials & the secret

**No secret value is included in this repository.** The solution ships the environment-variable
*definition* `crbcf_FoundryAgentClientSecret` but **not** its value.

- The flow references the secret as `@parameters('crbcf_FoundryAgentClientSecret')`. Because the flow
  **and** the environment variable are in the **same solution**, this binding is preserved across
  designer saves — so you set the secret **once** (in the environment variable) and never paste it
  into the flow.
- **To rotate** the secret: update the environment variable value under
  **Solutions → QA Copilot → Environment variables**. No flow edit required.
- **Do not** put the raw secret directly in the HTTP action's `secret` field — Copilot Studio masks
  that field on save (writes `******`), which breaks authentication with
  `AADSTS7000215: Invalid client secret provided`.

> For higher security you can back the environment variable with **Azure Key Vault** (a *secret*
> environment variable) instead of storing the value in Dataverse. This requires the Key Vault to
> allow Power Platform network access and the Power Platform service principal to have
> **Key Vault Secrets User** on the secret. See `docs/SETUP.md`.

---

## The Foundry hosted agent

This solution **calls** a Foundry hosted agent; it does not deploy one. Your agent must:

- Be a **hosted** Foundry agent exposing the **Responses** protocol at
  `…/agents/<name>/endpoint/protocols/openai/responses`.
- Accept a body of `{ "input": "<text>" }`.
- (For the execution use-case) be able to commit to Azure DevOps and trigger a pipeline — i.e. have
  the equivalent of the `test-executor` agent's tools and an `AZURE_DEVOPS_PAT`.

The flow wraps whatever the agent passes (`text` = test case, `text_1` = target URL) with an
instruction telling the Foundry agent to **convert the test case into an executable Playwright test,
commit it, and trigger the pipeline** — so plain tabular/markdown test cases work, not just Gherkin.

---

## Repository layout

```
qa-copilot-solution/
├── README.md                       # this file
├── solution/
│   └── QACopilot_1_0_0_3.zip       # sanitized, unmanaged Power Platform solution
└── docs/
    └── SETUP.md                    # detailed, step-by-step setup guide
```

## Notes & limitations

- Exported as **unmanaged** (`Managed=0`) so you can customize. For locked production deployment,
  export a **managed** version from your own environment after configuring.
- The publisher prefix is `mre` (Munich Re QA). You can keep it or repackage under your own prefix.
- The agent instructions reference QA workflows and a hand-off agent named "QAFoundry"; adjust the
  agent instructions to your terminology if desired.
- Environment-specific IDs were removed for public sharing; the solution will import but **will not
  call Foundry until you complete step 4**.
