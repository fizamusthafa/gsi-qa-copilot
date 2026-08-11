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
> placeholders. You must supply your own — see [Setup](#setup-two-parts).

---

## What's in the box

The solution contains everything below. For **just showing the agent working**, you only need the
**core** pieces — the rest is included so you have a foundation to grow into, but you can safely
ignore (or not wire up) the optional pieces for a quick demo. See
[What you actually need for a demo](#what-you-actually-need-for-a-demo).

### Core — needed to demo the agent + Foundry hand-off

| Component | Type | Purpose |
|-----------|------|---------|
| **QA Copilot** | Copilot Studio agent (bot) | The orchestrator the user talks to |
| 7 Skills | Inline agent skills | `requirements-analysis`, `scenario-design`, `test-case-design`, `suite-optimization`, `test-data-design`, `rca-method`, `quality-narration` |
| **Calling Foundry Agent** | Workflow (Copilot Studio) | Sends a test case + target URL to the Foundry hosted agent (`test-executor`) via HTTP |
| `crbcf_FoundryAgentClientSecret` | Environment variable | Holds the Foundry app client secret used by the workflow |

### Optional — richer implementation (great for later; skippable for a hackathon)

| Component | Type | Purpose | Why it's optional for a demo |
|-----------|------|---------|------------------------------|
| **Azure DevOps MCP** | MCP tool | Reads/writes Azure DevOps work items & QA artifacts | Recommended for the full workflow; a basic agent demo runs without it |
| **GitHub MCP** | MCP tool | QA-relevant repo context (issues, PRs, diffs) | Only needed if you want GitHub grounding |
| **ADO Return Sync** | Workflow | Syncs test results back from Azure DevOps | Part of the return path; not needed to *trigger* a run |
| **Foundry back to Copilot Studio** | Workflow | Returns Foundry output to the agent | Return path; not needed to *trigger* a run |
| 6 Dataverse tables | Tables | `mre_userstory`, `mre_testcase`, `mre_testdataset`, `mre_runrecord`, `mre_reqfinding`, `mre_defectrca` | Persistence/data model for a full solution; the agent conversation + hand-off works without storing to them |
| Canvas / code app | App | Supporting UI | A front-end for the data model; not needed to demo the agent itself |

### How it fits together

```
User ──▶ QA Copilot (Copilot Studio)
             │  skills: analyze / design / RCA / narrate
             ├──▶ Azure DevOps MCP  (read/write work items)   [optional]
             ├──▶ GitHub MCP        (repo context)            [optional]
             └──▶ "Calling Foundry Agent" workflow            [core]
                        │  HTTP POST { input: <test case + URL> }
                        ▼
                  Azure AI Foundry hosted agent (test-executor / "QAFoundry")
                        │  generates Playwright test, commits to Azure DevOps,
                        │  triggers BDD-Test-Execution pipeline
                        ▼
                  Results ──▶ ADO ──▶ "ADO Return Sync" / "Foundry back to Copilot Studio"  [optional]
```

The Foundry hosted agent itself (its Python source, tools, and pipeline) is **not** part of this
solution — it lives in Azure AI Foundry and Azure DevOps. This solution is the **Copilot Studio +
Power Platform orchestration layer** that calls it.

---

## What you actually need for a demo

The solution zip imports as a single unit (all components come in together — that's fine and does no
harm). The point is what you have to **configure and use**:

- **To show the agent + test-execution hand-off:** the QA Copilot agent, its skills, the
  **Calling Foundry Agent** workflow, and the secret environment variable. That's it.
- **The Dataverse tables and the canvas/code app** are there for a fuller, data-backed
  implementation. Wiring them up end-to-end is a **future exercise** — for a hackathon with limited
  time, you can leave them unused. They don't need any configuration to demo the agent.
- **The return-path workflows and GitHub MCP** are also optional for a first demo; add them when you
  want the full round-trip.

> TL;DR — for a fast hackathon demo, focus on **agent + Calling Foundry Agent workflow + secret**.
> Everything else can wait.

---

## Prerequisites

You need access to the following in **your** tenant. The Azure/Foundry items are the heavier,
one-time groundwork; the Power Platform items are quick.

**Azure / Foundry (one-time platform groundwork — [Part A](docs/SETUP-FOUNDRY.md))**
1. **Azure AI Foundry** project with a **deployed hosted agent** that exposes the OpenAI-compatible
   `responses` endpoint (reference solution used an agent named `test-executor`).
2. **Microsoft Entra app registration** (service principal) + a **client secret**, for the workflow
   to authenticate to Foundry.
3. Permission to **assign roles** (Owner / User Access Administrator) on the Foundry resource, to
   grant the app **Azure AI User**.
4. Foundry endpoint **reachable from Power Platform** (public network access / IP exception).

**Power Platform / Copilot Studio (the quick part — [Part B](docs/SETUP-POWER-PLATFORM.md))**
5. A **Power Platform environment** with **Dataverse**, and **Copilot Studio** enabled.
6. **Environment Maker + System Customizer** (or admin) to import the solution, and rights to
   **create connections**.
7. **Premium** Power Platform licensing (the workflow uses the **HTTP** action + premium connectors).
8. **DLP policy that allows** the connectors used — most importantly the generic **HTTP** connector,
   in the **same group** as Dataverse / Azure DevOps / GitHub / Agent connectors. This is the #1
   thing admins need to unblock. Details in
   [Part B → DLP requirements](docs/SETUP-POWER-PLATFORM.md#dlp-requirements).

**Optional (only if you use those components)**
9. **Azure DevOps** org/project (execution + return-sync) and a **GitHub** account/app (GitHub MCP).

---

## Setup (two parts)

Setup is split so you can see how small the Copilot Studio side is once the platform groundwork is
done:

- **[Part A — Azure AI Foundry](docs/SETUP-FOUNDRY.md)** — the one-time platform groundwork (app
  registration, secret, RBAC, confirm the agent endpoint, network). The more involved half.
- **[Part B — Power Platform / Copilot Studio](docs/SETUP-POWER-PLATFORM.md)** — the quick half:
  import the solution, set one secret, fill in three values, connect tools, publish. ~15 minutes, no
  code.

> Do Part A first (or confirm it already exists), then Part B.

---

## Credentials & the secret

**No secret value is included in this repository.** The solution ships the environment-variable
*definition* `crbcf_FoundryAgentClientSecret` but **not** its value.

- The workflow references the secret as `@parameters('crbcf_FoundryAgentClientSecret')`. Because the
  workflow **and** the environment variable are in the **same solution**, this binding is preserved
  across designer saves — so you set the secret **once** (in the environment variable) and never
  paste it into the workflow.
- **To rotate** the secret: update the environment variable value under
  **Solutions → QA Copilot → Environment variables**. No workflow edit required.
- **Do not** put the raw secret directly in the HTTP action's `secret` field — Copilot Studio masks
  that field on save (writes `******`), which breaks authentication with
  `AADSTS7000215: Invalid client secret provided`.

> For higher security you can back the environment variable with **Azure Key Vault** (a *secret*
> environment variable) instead of storing the value in Dataverse. See
> [Part B → Optional: Key Vault](docs/SETUP-POWER-PLATFORM.md#optional-back-the-secret-with-azure-key-vault).

---

## Repository layout

```
qa-copilot-solution/
├── README.md                         # this file
├── solution/
│   └── QACopilot_1_0_0_3.zip         # sanitized, unmanaged Power Platform solution
└── docs/
    ├── SETUP-FOUNDRY.md              # Part A — Azure AI Foundry (platform groundwork)
    └── SETUP-POWER-PLATFORM.md       # Part B — Power Platform / Copilot Studio (the quick part)
```

## Notes & limitations

- Exported as **unmanaged** (`Managed=0`) so you can customize. For locked production deployment,
  export a **managed** version from your own environment after configuring.
- The publisher prefix is `mre` (Munich Re QA). You can keep it or repackage under your own prefix.
- The agent instructions reference QA workflows and a hand-off agent named "QAFoundry"; adjust the
  agent instructions to your terminology if desired.
- Environment-specific IDs were removed for public sharing; the solution will import but **will not
  call Foundry until you complete [Part B → B3](docs/SETUP-POWER-PLATFORM.md#b3-fill-in-your-foundry-details-in-the-workflow)**.
