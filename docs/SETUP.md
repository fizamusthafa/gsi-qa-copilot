# QA Copilot — Detailed Setup Guide

This guide walks through everything needed to import and run the QA Copilot solution in your own
tenant. Read the top-level [`README.md`](../README.md) first for the overview.

---

## 0. Checklist of what you'll need

| # | Item | Where |
|---|------|-------|
| 1 | Power Platform environment with Dataverse | Power Platform Admin Center |
| 2 | Copilot Studio enabled | make.powerapps.com / copilotstudio.microsoft.com |
| 3 | Azure AI Foundry project + deployed hosted agent | Azure AI Foundry |
| 4 | Entra app registration + client secret | Entra ID → App registrations |
| 5 | Azure DevOps org/project (optional, for execution) | dev.azure.com |
| 6 | GitHub account/app (optional, for GitHub MCP) | github.com |
| 7 | Permission to assign roles on the Foundry resource | Azure RBAC |

---

## 1. Entra app registration (service principal) + secret

The **Calling Foundry Agent** flow authenticates to Azure AI Foundry using **client-credentials**
(OAuth 2.0 client credentials via the flow's `ActiveDirectoryOAuth` block).

1. **Entra ID → App registrations → New registration.**
   - Name: e.g. `qa-copilot-foundry-caller`.
   - Supported account types: **Single tenant** is sufficient.
   - No redirect URI needed.
2. Copy the **Application (client) ID** and the **Directory (tenant) ID** from the Overview page.
3. **Certificates & secrets → New client secret.**
   - Set an expiry that matches your rotation policy.
   - **Copy the secret _value_ immediately** (you cannot read it again later).
   - ⚠️ Use the secret **value**, not the secret **ID**. Using the ID causes
     `AADSTS7000215: Invalid client secret provided`.

Keep these three values handy:
- `TENANT_ID`
- `CLIENT_ID`
- `CLIENT_SECRET` (value)

---

## 2. Grant the app data-plane access to Foundry

Invoking a hosted agent requires **data-plane** permission on the Foundry (Azure AI Services /
Cognitive Services) account — control-plane roles like *Contributor* are **not** enough.

### Portal
1. Azure Portal → your **Azure AI Foundry** (Cognitive Services / AI Services) account.
2. **Access control (IAM) → Add role assignment.**
3. Role: **Azure AI User** (also shown as **Foundry User**).
4. Assign access to: **User, group, or service principal** → select your app registration.
5. Save. **Wait 5–10 minutes** for RBAC to propagate before testing.

### Azure CLI (equivalent)
```bash
az role assignment create \
  --assignee <CLIENT_ID> \
  --role "Azure AI User" \
  --scope "/subscriptions/<SUB_ID>/resourceGroups/<RG>/providers/Microsoft.CognitiveServices/accounts/<FOUNDRY_ACCOUNT>"
```

> Symptom if this step is missing/not propagated: the flow authenticates but the agent call returns
> **HTTP 403** — `does not have permissions for Microsoft.CognitiveServices/accounts/AIServices/agents/write`.

---

## 3. Confirm your Foundry hosted agent endpoint

Your agent must expose the **Responses** protocol. The invocation URL pattern is:

```
https://<FOUNDRY_RESOURCE>.services.ai.azure.com/api/projects/<PROJECT>/agents/<AGENT_NAME>/endpoint/protocols/openai/responses?api-version=2025-11-15-preview
```

- `<FOUNDRY_RESOURCE>` — your Foundry account name (the subdomain of `*.services.ai.azure.com`).
- `<PROJECT>` — your Foundry project name.
- `<AGENT_NAME>` — the hosted agent to invoke (this solution used `test-executor`).
- The endpoint routes to the agent's `@latest` version unless you pin a version.

**Quick manual test** (using your own Azure login token, to confirm the agent works before wiring
the flow):
```bash
TOKEN=$(az account get-access-token --resource https://ai.azure.com --query accessToken -o tsv)
curl -X POST \
  "https://<FOUNDRY_RESOURCE>.services.ai.azure.com/api/projects/<PROJECT>/agents/<AGENT_NAME>/endpoint/protocols/openai/responses?api-version=2025-11-15-preview" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{ "input": "Say OK" }'
```
A `status: completed` response with an assistant message confirms the endpoint.

---

## 4. Import the solution

1. **[make.powerapps.com](https://make.powerapps.com)** → pick the target **environment**
   (top-right environment picker).
2. **Solutions → Import solution → Browse** → select
   [`solution/QACopilot_1_0_0_3.zip`](../solution/QACopilot_1_0_0_3.zip) → **Next**.
3. **Connections / connection references** — the import wizard lists the connectors used:
   - **Microsoft Dataverse**
   - **Azure DevOps MCP** (`shared_adomcpserver`)
   - **GitHub** (`shared_github`)
   - **Agent** / Copilot connectors
   For each, pick an existing connection or click **+ New connection**, authenticate, then return and
   **Refresh**.
4. **Environment variable** — when prompted for **Foundry Agent Client Secret**
   (`crbcf_FoundryAgentClientSecret`), paste your `CLIENT_SECRET` value.
5. Click **Import** and wait for it to finish.

> If the import doesn't prompt for the environment variable, set it afterwards:
> **Solutions → QA Copilot → open `Foundry Agent Client Secret` → set Current Value.**

---

## 5. Configure the "Calling Foundry Agent" flow

The flow ships with **placeholders** (identifiers were removed for public sharing). Update them:

1. **Solutions → QA Copilot → Cloud flows → Calling Foundry Agent → Edit.**
2. Open the **HTTP** action and replace:

   | Field | Value |
   |-------|-------|
   | **URI** | your full Responses URL from step 3 |
   | **Authentication → Tenant** | `<TENANT_ID>` |
   | **Authentication → Client ID** | `<CLIENT_ID>` |
   | **Authentication → Audience** | `https://ai.azure.com` |
   | **Authentication → Secret** | keep `@parameters('crbcf_FoundryAgentClientSecret')` |

3. **Do not** type the raw secret into the Secret field. Leave the `@parameters(...)` expression so
   it reads from the environment variable. (Copilot Studio masks a literal secret to `******` on
   save, which breaks auth.)
4. **Save**, then confirm the flow is **turned on**.

### About the request body
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
  `invalid_payload: '0x0A' is invalid within a JSON string` when the test case has line breaks.

---

## 6. Configure MCP tools & other flows

1. **Copilot Studio → QA Copilot → Tools:**
   - **Azure DevOps MCP** — connect an account with access to your ADO project.
   - **GitHub MCP** — connect your GitHub account/app.
2. **Cloud flows** — make sure **ADO Return Sync** and **Foundry back to Copilot Studio** are
   **turned on** and their connections are authorized (Azure DevOps / Dataverse).

---

## 7. Publish and validate

1. **Copilot Studio → QA Copilot → Publish.**
2. In the **Test** pane, drive a QA workflow that ends in a test-execution hand-off. The agent calls
   the flow → the flow calls Foundry → the Foundry agent generates/commits/queues the test.
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
   `subscription / resource group / vault / secret name`, and point the flow's `secret` at it.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `AADSTS7000215: Invalid client secret provided` | Secret field was overwritten with `******` on a designer save, or the secret **ID** was used instead of the **value** | Reset the environment variable value; keep the flow's Secret bound to `@parameters('crbcf_FoundryAgentClientSecret')` |
| `The workflow parameter 'crbcf_FoundryAgentClientSecret' is not found` | The flow's `parameters` binding was stripped, or the env var isn't in the same solution | Ensure the env var **and** the flow are in the QA Copilot solution; re-open/save the flow so the binding is restored |
| HTTP **403** `agents/write` denied | App lacks data-plane role on Foundry, or RBAC hasn't propagated | Assign **Azure AI User** to the app on the Foundry account; wait 5–10 min |
| `invalid_payload: '0x0A' is invalid within a JSON string` | Test case contains newlines injected into hand-typed JSON | Use the `setProperty(json('{}'), 'input', …)` body expression (already in this solution) |
| Agent replies "no BDD scenarios / nothing to run" | The agent received empty or non-actionable input | Confirm the trigger inputs are populated (the agent must pass `text`/`text_1`); the flow's instruction wrapper handles tabular/markdown test cases |
| Flow can't be tested standalone from an API call | The "When an agent calls the flow" trigger only receives inputs when invoked **by the agent** | Test end-to-end from the QA Copilot agent, not by calling the flow's run endpoint directly |

---

## Security notes

- This repository is **sanitized**: no tenant ID, client ID, Foundry endpoint, or secret is
  included. Everything environment-specific is a placeholder you fill in.
- Never commit a real client secret, PAT, or connection credential to source control.
- Test data handled by the agent must be **synthetic/masked** — never real personal data.
