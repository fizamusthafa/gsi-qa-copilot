# QA Copilot – Power Platform Solution

QA Copilot is a Microsoft Copilot Studio agent that acts as a QA engineering assistant. It analyzes
requirements, designs test scenarios / cases / data, performs defect & build RCA, narrates quality
metrics, integrates with **Azure DevOps**, and (optionally) hands off test execution to an **Azure AI
Foundry** hosted agent that generates and runs Playwright tests.

This repository packages the solution as an **unmanaged Power Platform solution** (`.zip`) that you
import into your own environment, plus the setup you need to run it in **your** tenant.

> ✅ **You can set up and test the Copilot Studio + Azure DevOps experience today — no Azure AI
> Foundry access required.** Foundry is a **separate, optional track** for a single workflow, and you
> can add it later. See [Setup](#setup).

> ⚠️ **This export is sanitized.** All environment-specific values (tenant ID, app registration
> client ID, Foundry endpoint) and the client **secret** have been removed and replaced with
> placeholders.

---

## What's in the box

The whole solution imports as one unit. What matters is which parts you can **use now** versus which
need the separate Foundry track.

### Works now — Copilot Studio + Azure DevOps (no Foundry)

| Component | Type | Purpose |
|-----------|------|---------|
| **QA Copilot** | Copilot Studio agent (bot) | The orchestrator the user talks to |
| 7 Skills | Inline agent skills | `requirements-analysis`, `scenario-design`, `test-case-design`, `suite-optimization`, `test-data-design`, `rca-method`, `quality-narration` |
| **Azure DevOps MCP** | MCP tool | Reads/writes Azure DevOps work items & QA artifacts |
| **Foundry back to Copilot Studio** | Workflow | **When an Azure DevOps build completes → calls the QA Copilot agent.** One of the core use cases. *(Does not call Foundry despite the name.)* |
| **ADO Return Sync** | Workflow | When an Azure DevOps work item is updated → syncs to Dataverse |
| GitHub MCP | MCP tool | QA-relevant repo context (issues/PRs/diffs). Optional |
| 6 Dataverse tables | Tables | `mre_userstory`, `mre_testcase`, `mre_testdataset`, `mre_runrecord`, `mre_reqfinding`, `mre_defectrca`. Imported; optional to use for a quick demo |
| Canvas / code app | App | Supporting UI for the data model. Optional to use |

### Needs Azure AI Foundry — add later (separate track)

| Component | Type | Purpose |
|-----------|------|---------|
| **Calling Foundry Agent** | Workflow | Agent → Foundry hosted agent over HTTP, to generate & run Playwright tests |
| `crbcf_FoundryAgentClientSecret` | Environment variable | Client secret used **only** by the Calling Foundry Agent workflow |

### How it fits together

```
User ──▶ QA Copilot (Copilot Studio)
             │  skills: analyze / design / RCA / narrate
             ├──▶ Azure DevOps MCP  (read/write work items)          [now]
             ├──▶ GitHub MCP        (repo context)                   [now, optional]
             │
             │   Azure DevOps ──(work item updated)──▶ "ADO Return Sync" ──▶ Dataverse   [now]
             │   Azure DevOps ──(build completes)────▶ "Foundry back to Copilot Studio"
             │                                              └──▶ calls the agent          [now]
             │
             └──▶ "Calling Foundry Agent" workflow ──HTTP──▶ Azure AI Foundry agent       [later]
                        generates Playwright test, commits to Azure DevOps, triggers pipeline
```

The Foundry hosted agent itself (its Python source, tools, pipeline) is **not** part of this solution
— it lives in Azure AI Foundry and Azure DevOps. This solution is the **Copilot Studio + Power
Platform orchestration layer**.

---

## What you actually need for a demo

- **To demo now (no Foundry):** the QA Copilot agent + skills, the **Azure DevOps MCP** tool, and the
  two Azure DevOps workflows — **Foundry back to Copilot Studio** (build completes → agent) and
  **ADO Return Sync** (work item updated → Dataverse).
- **The Dataverse tables and canvas/code app** import along with the solution but need **no
  configuration** to demo the agent. Fully wiring them is a **future exercise** — fine to leave
  unused for a hackathon.
- **The Calling Foundry Agent workflow** stays turned off until you have Foundry access; add it via
  the separate Foundry track without touching anything else.

---

## Prerequisites

### For the part you're doing now — Power Platform / Copilot Studio + Azure DevOps
1. A **Power Platform environment** with **Dataverse**, and **Copilot Studio** enabled.
2. **Environment Maker + System Customizer** (or admin) to import the solution, and rights to
   **create connections**.
3. **Premium** Power Platform licensing (premium connectors are used).
4. An **Azure DevOps** organization/project you can connect to (the two workflows + the Azure DevOps
   MCP tool).
5. **DLP policy that allows** the connectors used now — **Dataverse, Azure DevOps, Azure DevOps MCP,
   GitHub, Agent** — all in the **same, non-blocked** group. *(The generic **HTTP** connector is only
   needed later for the Foundry workflow.)* Details:
   [Power Platform → DLP requirements](docs/SETUP-POWER-PLATFORM.md#dlp-requirements).
6. Optional: a **GitHub** account/app for the GitHub MCP tool.

### For later, only when you enable the Foundry workflow (separate track)
- Azure AI Foundry project + deployed hosted agent, an **Entra app registration + client secret**,
  the **Azure AI User** role on the Foundry resource, and the **HTTP** connector allowed by DLP.
  Everything Foundry-related is isolated in **[docs/SETUP-FOUNDRY.md](docs/SETUP-FOUNDRY.md)**.

---

## Setup

- **👉 Start here: [Power Platform / Copilot Studio setup](docs/SETUP-POWER-PLATFORM.md)** — import the
  solution, connect Azure DevOps + the agent tools, turn on the two Azure DevOps workflows, publish,
  and test. **~15 minutes, no code, no Foundry needed.**
- **Later / separate: [Azure AI Foundry track](docs/SETUP-FOUNDRY.md)** — a self-contained guide to
  enable the single **Calling Foundry Agent** workflow once you have Foundry access. Independent of
  everything above.

---

## Credentials & the secret (Foundry workflow only)

The client secret is used **only** by the Calling Foundry Agent workflow, so you only need it when
you do the Foundry track. **No secret value is included in this repository** — the solution ships the
environment-variable *definition* `crbcf_FoundryAgentClientSecret` but not its value.

- The workflow references the secret as `@parameters('crbcf_FoundryAgentClientSecret')`. Because the
  workflow and the environment variable are in the **same solution**, this binding survives designer
  saves — set the secret **once** in the environment variable, never in the workflow.
- **Rotate** by updating the environment variable value under
  **Solutions → QA Copilot → Environment variables**. No workflow edit required.
- **Do not** put the raw secret in the HTTP action's `secret` field — Copilot Studio masks it to
  `******` on save, breaking auth with `AADSTS7000215: Invalid client secret provided`.

> Optional: back it with **Azure Key Vault** — see
> [Power Platform → Optional: Key Vault](docs/SETUP-POWER-PLATFORM.md#optional-back-the-secret-with-azure-key-vault).

---

## Repository layout

```
qa-copilot-solution/
├── README.md                         # this file
├── solution/
│   └── QACopilot_1_0_0_3.zip         # sanitized, unmanaged Power Platform solution
└── docs/
    ├── SETUP-POWER-PLATFORM.md       # 👉 start here — Copilot Studio + Azure DevOps (no Foundry)
    └── SETUP-FOUNDRY.md              # separate/later — the one Foundry workflow
```

## Notes & limitations

- Exported as **unmanaged** (`Managed=0`) so you can customize. For locked production deployment,
  export a **managed** version from your own environment after configuring.
- The publisher prefix is `mre` (Munich Re QA). Keep it or repackage under your own prefix.
- "Foundry back to Copilot Studio" is triggered by an **Azure DevOps build completing** and calls the
  agent — it works without Foundry; in the full loop that build is what Foundry's pipeline triggers.
- Environment-specific IDs were removed for public sharing; the solution imports and the Copilot
  Studio + Azure DevOps parts run once you complete the Power Platform setup. The Foundry workflow
  stays off until you complete the Foundry track.
