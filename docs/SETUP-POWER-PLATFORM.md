# Setup — Power Platform / Copilot Studio (do this now — no Foundry needed)

**You can set up and test the whole Copilot Studio experience today without Azure AI Foundry
access.** The agent, its skills, and the two **Azure DevOps–driven** workflows all run without
Foundry. Only one workflow (**Calling Foundry Agent**) needs Foundry, and it's cleanly separated into
a [later section](#when-you-get-foundry-access-enable-the-calling-foundry-agent-workflow).

No code, roughly **15 minutes**.

**What works now vs later**

| Component | Works now (no Foundry)? |
|-----------|:-----------------------:|
| QA Copilot agent + 7 skills | ✅ |
| **Azure DevOps MCP** tool | ✅ |
| GitHub MCP tool | ✅ (optional) |
| **Foundry back to Copilot Studio** workflow (build completes → calls the agent) | ✅ |
| **ADO Return Sync** workflow (work item updated → Dataverse) | ✅ |
| Dataverse tables + canvas/code app | ✅ (imported; optional to use) |
| **Calling Foundry Agent** workflow (agent → Foundry over HTTP) | ⛔ needs Foundry — [wire later](#when-you-get-foundry-access-enable-the-calling-foundry-agent-workflow) |

> Despite its name, **Foundry back to Copilot Studio** does not call Foundry. It's triggered by an
> **Azure DevOps build completing** and calls back into the Copilot Studio **Agent**. In the full
> loop that build is what Foundry's pipeline kicks off, but you can test this workflow now with any
> Azure DevOps build.

---

## B0. Access you need for this part

- **Environment Maker** + **System Customizer** (or System Administrator) on the target Power
  Platform environment — required to import a solution.
- A **Copilot Studio** license, and Copilot Studio enabled on the environment.
- Rights to **create connections** for the connectors below (some tenants restrict this).
- **Premium** Power Platform licensing (premium connectors are used).
- An **Azure DevOps** organization/project you can connect to (the two workflows and the Azure DevOps
  MCP tool use it).
- Your tenant's **DLP policy must allow** the connectors used — see
  [DLP requirements](#dlp-requirements) before importing.

> You do **not** need Azure AI Foundry, an Entra app registration, or the client secret for anything
> in this guide except the optional [Foundry workflow section](#when-you-get-foundry-access-enable-the-calling-foundry-agent-workflow).

---

## DLP requirements

> Read this **before** importing. Data Loss Prevention (DLP) policies are the most common reason a
> workflow fails to run even when everything else is correct.

Power Platform DLP sorts every connector into **Business**, **Non-Business**, or **Blocked**. Two
rules matter:

1. A single workflow **cannot combine connectors from different groups**. All connectors a workflow
   uses must be in the **same** group.
2. A **Blocked** connector cannot be used at all.

**For the part you're doing now (no Foundry),** make sure these are in the **same, non-blocked**
group:

| Connector | Used by |
|-----------|---------|
| **Microsoft Dataverse** (`shared_commondataserviceforapps`) | ADO Return Sync, tables |
| **Azure DevOps** (`shared_visualstudioteamservices`) | ADO Return Sync, Foundry-back workflow |
| **Azure DevOps MCP** (`shared_adomcpserver`) | Agent tool |
| **GitHub** (`shared_github`) | Agent tool (optional) |
| **Agent / Copilot** (`shared_agentnode`) | Foundry-back workflow, agent ↔ workflow calls |

**Only when you add Foundry later** you additionally need:

| Connector | Used by |
|-----------|---------|
| **HTTP** | Calling Foundry Agent workflow (⚠️ often blocked by default; allow it and place it in the same group) |

Also confirm with your Power Platform admin that:
- **Copilot Studio** and **generative AI** features are enabled for the environment.
- Adding **MCP tools / custom connectors** to agents is permitted.
- Any **tenant isolation** policy allows the connections you create.

---

## B1. Import the solution

1. **[make.powerapps.com](https://make.powerapps.com)** → pick the target **environment**
   (top-right environment picker).
2. **Solutions → Import solution → Browse** → select
   [`solution/QACopilot_1_0_0_3.zip`](../solution/QACopilot_1_0_0_3.zip) → **Next**.
3. **Connections / connection references** — the wizard lists the connectors used:
   - **Microsoft Dataverse**
   - **Azure DevOps** (`shared_visualstudioteamservices`)
   - **Azure DevOps MCP** (`shared_adomcpserver`)
   - **GitHub** (`shared_github`)
   - **Agent / Copilot** (`shared_agentnode`)
   For each, pick an existing connection or click **+ New connection**, authenticate, then return and
   **Refresh**.
4. **Environment variable** — if prompted for **Foundry Agent Client Secret**
   (`crbcf_FoundryAgentClientSecret`), you can **leave it blank for now** (it's only used by the
   Foundry workflow). Set it later when you do the [Foundry section](#when-you-get-foundry-access-enable-the-calling-foundry-agent-workflow).
5. Click **Import** and wait for it to finish.

> The whole solution imports as one unit (tables, app, all three workflows). That's fine — unused
> components just sit there and need no configuration. See the README's *"What you actually need for
> a demo."*

---

## B2. Connect the Azure DevOps workflows

Both of these work **without Foundry**:

1. **Solutions → QA Copilot → Workflows.**
2. **ADO Return Sync** — open it, make sure its **Azure DevOps** and **Dataverse** connections are
   authorized, point the **"When a work item is updated"** trigger at your ADO organization/project,
   and **turn it on**.
3. **Foundry back to Copilot Studio** — open it, authorize its **Azure DevOps** and **Agent**
   connections, point the **"When a build completes"** trigger at your ADO
   organization/project/pipeline, and **turn it on**. This is the workflow that reports a completed
   build back into the QA Copilot agent.

---

## B3. Connect the MCP tools

**Copilot Studio → QA Copilot → Tools:**
- **Azure DevOps MCP** — connect an account with access to your ADO project.
- **GitHub MCP** — connect your GitHub account/app (optional).

---

## B4. Publish and test (no Foundry required)

1. **Copilot Studio → QA Copilot → Publish.**
2. In the **Test** pane, exercise the agent's QA skills (requirements analysis, scenario/test-case
   design, RCA, quality narration) grounded in your Azure DevOps work items via the Azure DevOps MCP.
3. Test the **Foundry back to Copilot Studio** use case: complete a build in your Azure DevOps
   pipeline and confirm the workflow fires and the agent is invoked
   (**Power Automate → Foundry back to Copilot Studio → Run history**).
4. Test **ADO Return Sync**: update a work item and confirm the workflow runs and reads/writes
   Dataverse as expected.

That's the Copilot Studio experience validated end-to-end — no Foundry needed.

---

## When you get Foundry access: enable the "Calling Foundry Agent" workflow

Do this **only after** completing the separate **[Azure AI Foundry track](SETUP-FOUNDRY.md)** (app
registration, client secret, RBAC, and the Responses URL). It's fully independent of everything
above.

1. **Set the secret** — **Solutions → QA Copilot → Environment variables → Foundry Agent Client
   Secret** → set **Current Value** = your `CLIENT_SECRET`.
   - Keep the secret here, not in the workflow. The workflow references it as
     `@parameters('crbcf_FoundryAgentClientSecret')`; because the env var and workflow are in the
     same solution, this binding survives designer saves. Rotate by updating this value only.
   - **Never** type the raw secret into the HTTP action's `secret` field — Copilot Studio masks it to
     `******` on save, breaking auth with `AADSTS7000215: Invalid client secret provided`.
2. **Fill in your Foundry details** — **Workflows → Calling Foundry Agent → Edit → HTTP** action:

   | Field | Value |
   |-------|-------|
   | **URI** | your Foundry Responses URL |
   | **Authentication → Tenant** | `<TENANT_ID>` |
   | **Authentication → Client ID** | `<CLIENT_ID>` |
   | **Authentication → Audience** | `https://ai.azure.com` (already set) |
   | **Authentication → Secret** | keep `@parameters('crbcf_FoundryAgentClientSecret')` |

3. Ensure the **HTTP** connector is allowed by DLP (see [DLP requirements](#dlp-requirements)).
4. **Save** and **turn on** the workflow.

### About the request body (already configured — no change needed)
```
@setProperty(json('{}'), 'input',
  concat('You are given a test case ... convert ... commit ... trigger the pipeline.',
         decodeUriComponent('%0A%0A'),
         'Target application URL: ', triggerOutputs()?['body/text_1'],
         decodeUriComponent('%0A%0A'),
         'Test case:', decodeUriComponent('%0A%0A'),
         triggerOutputs()?['body/text']))
```
- `text` (**Test cases**) and `text_1` (**web link**) are the trigger inputs the agent passes in.
- `setProperty(json('{}'), …)` (not hand-typed `{ "input": "..." }`) guarantees newlines/quotes are
  JSON-escaped. Hand-typed JSON fails with `invalid_payload: '0x0A' is invalid within a JSON string`
  when the test case has line breaks.

---

## Optional: back the secret with Azure Key Vault

For a *secret* environment variable backed by Key Vault instead of a plain-text value in Dataverse:

1. Store the client secret in a Key Vault.
2. Grant the **Power Platform** first-party service principal **Key Vault Secrets User** on that
   secret (RBAC vaults) or an equivalent access policy.
3. Ensure the vault allows Power Platform network access (a fully **public-network-disabled** vault
   with no exception will block retrieval).
4. Create the environment variable as type **Secret** referencing
   `subscription / resource group / vault / secret name`, and point the workflow's `secret` at it.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| Workflow won't save / "connector not allowed" / flow suspended | **DLP** blocks a connector or mixes groups | Put the connectors this workflow uses in the same, non-blocked DLP group (see [DLP requirements](#dlp-requirements)) |
| Foundry-back / ADO Return Sync workflow doesn't trigger | Trigger not pointed at your ADO org/project, or connection unauthorized | Re-open the workflow, fix the Azure DevOps trigger target, authorize the connection, turn it on |
| `AADSTS7000215: Invalid client secret provided` *(Foundry workflow only)* | Secret overwritten with `******` on a designer save, or the secret **ID** used instead of the **value** | Reset the environment variable value; keep the Secret bound to `@parameters('crbcf_FoundryAgentClientSecret')` |
| `The workflow parameter 'crbcf_FoundryAgentClientSecret' is not found` *(Foundry workflow only)* | The `parameters` binding was stripped, or the env var isn't in the same solution | Ensure the env var **and** the workflow are in the QA Copilot solution; re-open/save the workflow |
| HTTP **403** `agents/write` denied *(Foundry workflow only)* | App lacks data-plane role on Foundry, or RBAC hasn't propagated | Assign **Azure AI User** to the app on the Foundry account (see [Foundry track](SETUP-FOUNDRY.md)); wait 5–10 min |
| `invalid_payload: '0x0A' is invalid within a JSON string` *(Foundry workflow only)* | Test case newlines injected into hand-typed JSON | Use the `setProperty(json('{}'), 'input', …)` body expression (already in this solution) |

---

## Security notes

- This repository is **sanitized**: no tenant ID, client ID, Foundry endpoint, or secret is included.
  Everything environment-specific is a placeholder you fill in.
- Never commit a real client secret, PAT, or connection credential to source control.
- Test data handled by the agent must be **synthetic/masked** — never real personal data.
