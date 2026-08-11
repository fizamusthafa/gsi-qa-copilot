# Setup — Part B: Power Platform / Copilot Studio (the quick part)

Once [Part A (Azure AI Foundry)](SETUP-FOUNDRY.md) is done, this side is fast: **import one
solution, set one secret, fill in three values, connect two tools, publish.** No code, roughly
**15 minutes**.

> Have these from Part A ready: `TENANT_ID`, `CLIENT_ID`, `CLIENT_SECRET` (value), and your Foundry
> Responses URL.

**At a glance**

| Step | What | ~Time |
|------|------|-------|
| B1 | Import the solution | 3–5 min |
| B2 | Set the secret environment variable | 1 min |
| B3 | Fill in 3 values in the workflow | 2 min |
| B4 | Connect Azure DevOps MCP + GitHub | 3 min |
| B5 | Publish & test | 3 min |

---

## B0. Access you need for this part

- **Environment Maker** + **System Customizer** (or System Administrator) on the target Power
  Platform environment — required to import a solution.
- A **Copilot Studio** license, and Copilot Studio enabled on the environment.
- Rights to **create connections** for the connectors below (some tenants restrict this).
- **Premium** Power Platform licensing (the workflow uses the **HTTP** action and premium
  connectors).
- Your tenant's **DLP policy must allow** the connectors this solution uses — see
  [DLP requirements](#dlp-requirements) before importing, or the workflows will be blocked.

---

## DLP requirements

> Read this **before** importing. Data Loss Prevention (DLP) policies are the most common reason the
> "Calling Foundry Agent" workflow fails to run even when everything else is correct.

Power Platform DLP sorts every connector into **Business**, **Non-Business**, or **Blocked**. Two
rules matter here:

1. A single workflow **cannot combine connectors from different groups** (e.g., one Business + one
   Non-Business). All connectors a workflow uses must be in the **same** group.
2. A **Blocked** connector cannot be used at all.

Make sure your tenant/environment DLP policy places **all** of these in the **same, non-blocked**
group:

| Connector | Used by | Notes |
|-----------|---------|-------|
| **HTTP** | Calling Foundry Agent workflow | ⚠️ **Most common blocker.** The generic **HTTP** connector is blocked by default in many tenants. It must be **allowed** and in the same group as the others. |
| **Microsoft Dataverse** (`shared_commondataserviceforapps`) | All workflows / tables | |
| **Azure DevOps** (`shared_visualstudioteamservices`) | ADO Return Sync, build triggers | |
| **Azure DevOps MCP** (`shared_adomcpserver`) | Agent tool | |
| **GitHub** (`shared_github`) | Agent tool | |
| **Agent / Copilot** connectors (`shared_agentnode`) | Agent ↔ workflow calls | |

Also confirm with your Power Platform admin that:
- **Copilot Studio** and **generative AI** features are enabled for the environment.
- Adding **MCP tools / custom connectors** to agents is permitted.
- **Connector action-level** DLP rules don't specifically block the HTTP action or these connectors.
- Any **tenant isolation** policy allows the cross-service calls (to `*.services.ai.azure.com`).

> If DLP can't be changed to allow the generic HTTP connector, an alternative is to front the Foundry
> call with a custom connector that's approved in your tenant — but the simplest path is to allow
> HTTP in the same DLP group as Dataverse.

---

## B1. Import the solution

1. **[make.powerapps.com](https://make.powerapps.com)** → pick the target **environment**
   (top-right environment picker).
2. **Solutions → Import solution → Browse** → select
   [`solution/QACopilot_1_0_0_3.zip`](../solution/QACopilot_1_0_0_3.zip) → **Next**.
3. **Connections / connection references** — the wizard lists the connectors used:
   - **Microsoft Dataverse**
   - **Azure DevOps MCP** (`shared_adomcpserver`)
   - **GitHub** (`shared_github`)
   - **Agent / Copilot** connectors
   For each, pick an existing connection or click **+ New connection**, authenticate, then return and
   **Refresh**.
4. **Environment variable** — when prompted for **Foundry Agent Client Secret**
   (`crbcf_FoundryAgentClientSecret`), paste your `CLIENT_SECRET` value from Part A.
5. Click **Import** and wait for it to finish.

> If the import doesn't prompt for the environment variable, set it in B2.

---

## B2. Set the secret environment variable

**Solutions → QA Copilot → Environment variables → Foundry Agent Client Secret → set Current
Value** = your `CLIENT_SECRET`.

Why it's stored here (and not in the workflow):
- The workflow references the secret as `@parameters('crbcf_FoundryAgentClientSecret')`. Because the
  workflow **and** the environment variable are in the **same solution**, this binding survives
  designer saves. You set the secret **once** and never paste it into the workflow.
- **To rotate**: just update this value. No workflow edit needed.
- **Never** type the raw secret into the HTTP action's `secret` field — Copilot Studio masks it to
  `******` on save, which breaks auth with `AADSTS7000215: Invalid client secret provided`.

---

## B3. Fill in your Foundry details in the workflow

The workflow ships with **placeholders** (identifiers were removed for public sharing). Replace them:

1. **Solutions → QA Copilot → Workflows → Calling Foundry Agent → Edit.**
2. Open the **HTTP** action and set:

   | Field | Value |
   |-------|-------|
   | **URI** | your full Foundry Responses URL from Part A |
   | **Authentication → Tenant** | `<TENANT_ID>` |
   | **Authentication → Client ID** | `<CLIENT_ID>` |
   | **Authentication → Audience** | `https://ai.azure.com` (already set) |
   | **Authentication → Secret** | keep `@parameters('crbcf_FoundryAgentClientSecret')` |

3. **Save**, then confirm the workflow is **turned on**.

### About the request body (already configured — no change needed)
The HTTP body is an expression that builds valid JSON and wraps the inputs with an instruction:

```
@setProperty(json('{}'), 'input',
  concat('You are given a test case ... convert ... commit ... trigger the pipeline.',
         decodeUriComponent('%0A%0A'),
         'Target application URL: ', triggerOutputs()?['body/text_1'],
         decodeUriComponent('%0A%0A'),
         'Test case:', decodeUriComponent('%0A%0A'),
         triggerOutputs()?['body/text']))
```

- `text` (label **Test cases**) and `text_1` (label **web link**) are the trigger inputs the agent
  passes in.
- Using `setProperty(json('{}'), …)` (not hand-typed `{ "input": "..." }`) guarantees newlines and
  quotes in the test case are correctly JSON-escaped. Hand-typed JSON fails with
  `invalid_payload: '0x0A' is invalid within a JSON string` when the test case contains line breaks.

---

## B4. Connect the MCP tools & other workflows

1. **Copilot Studio → QA Copilot → Tools:**
   - **Azure DevOps MCP** — connect an account with access to your ADO project.
   - **GitHub MCP** — connect your GitHub account/app.
2. **Workflows** — make sure **ADO Return Sync** and **Foundry back to Copilot Studio** are
   **turned on** and their connections are authorized (Azure DevOps / Dataverse).

---

## B5. Publish and validate

1. **Copilot Studio → QA Copilot → Publish.**
2. In the **Test** pane, drive a QA workflow that ends in a test-execution hand-off. The agent calls
   the workflow → the workflow calls Foundry → the Foundry agent generates/commits/queues the test.
3. Validate the run in **Power Automate → Calling Foundry Agent → Run history** (HTTP action should
   be **Succeeded / 200**) and in **Azure DevOps** (new branch + pipeline run).

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
| Workflow won't save / "connector not allowed" / flow suspended | **DLP** blocks a connector or mixes groups | Put HTTP + Dataverse + ADO + GitHub + Agent connectors in the same, non-blocked DLP group (see [DLP requirements](#dlp-requirements)) |
| `AADSTS7000215: Invalid client secret provided` | Secret field was overwritten with `******` on a designer save, or the secret **ID** was used instead of the **value** | Reset the environment variable value; keep the workflow's Secret bound to `@parameters('crbcf_FoundryAgentClientSecret')` |
| `The workflow parameter 'crbcf_FoundryAgentClientSecret' is not found` | The workflow's `parameters` binding was stripped, or the env var isn't in the same solution | Ensure the env var **and** the workflow are in the QA Copilot solution; re-open/save the workflow so the binding is restored |
| HTTP **403** `agents/write` denied | App lacks data-plane role on Foundry, or RBAC hasn't propagated | Assign **Azure AI User** to the app on the Foundry account (Part A); wait 5–10 min |
| `invalid_payload: '0x0A' is invalid within a JSON string` | Test case newlines injected into hand-typed JSON | Use the `setProperty(json('{}'), 'input', …)` body expression (already in this solution) |
| Agent replies "no BDD scenarios / nothing to run" | The agent received empty or non-actionable input | Confirm the trigger inputs are populated (the agent must pass `text`/`text_1`); the workflow's instruction wrapper handles tabular/markdown test cases |
| Workflow can't be tested standalone from an API call | The "When an agent calls the workflow" trigger only receives inputs when invoked **by the agent** | Test end-to-end from the QA Copilot agent, not by calling the workflow's run endpoint directly |

---

## Security notes

- This repository is **sanitized**: no tenant ID, client ID, Foundry endpoint, or secret is included.
  Everything environment-specific is a placeholder you fill in.
- Never commit a real client secret, PAT, or connection credential to source control.
- Test data handled by the agent must be **synthetic/masked** — never real personal data.
