# Setup — Part A: Azure AI Foundry (platform groundwork)

This is the **one-time platform setup** on the Azure side. It's the more involved half, because it
touches Azure AI Foundry, Entra app registrations, and Azure RBAC. Once this is done, the
[Power Platform / Copilot Studio side](SETUP-POWER-PLATFORM.md) is quick.

> You only do this once per environment. If your Foundry agent and app registration already exist,
> you can skip straight to [Part B](SETUP-POWER-PLATFORM.md).

**What you produce here (carry these into Part B):**

| Value | Example |
|-------|---------|
| `TENANT_ID` | `00000000-0000-0000-0000-000000000000` |
| `CLIENT_ID` (app registration) | `11111111-1111-1111-1111-111111111111` |
| `CLIENT_SECRET` (value) | *(kept in the environment variable, never in the workflow)* |
| Foundry Responses URL | `https://<resource>.services.ai.azure.com/api/projects/<project>/agents/<agent>/endpoint/protocols/openai/responses?api-version=2025-11-15-preview` |

---

## A0. Access you need for this part

- **Azure subscription** rights to your Azure AI Foundry (Cognitive Services / AI Services) account.
- **Owner** or **User Access Administrator** on that Foundry account (to create a role assignment).
- Rights to **register an Entra application** (or an admin who will do it and grant consent).
- A **deployed hosted agent** in your Foundry project (see [A3](#a3-confirm-your-foundry-hosted-agent)).

---

## A1. Create the Entra app registration + secret

The **Calling Foundry Agent** workflow authenticates to Foundry using **client credentials**
(OAuth 2.0, via the workflow's `ActiveDirectoryOAuth` block).

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

Keep `TENANT_ID`, `CLIENT_ID`, and `CLIENT_SECRET` (value) for Part B.

---

## A2. Grant the app data-plane access to Foundry

Invoking a hosted agent requires **data-plane** permission on the Foundry account — control-plane
roles like *Contributor* are **not** enough.

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

> Symptom if this step is missing/not propagated: the workflow authenticates but the agent call
> returns **HTTP 403** —
> `does not have permissions for Microsoft.CognitiveServices/accounts/AIServices/agents/write`.

---

## A3. Confirm your Foundry hosted agent

This solution **calls** a Foundry hosted agent; it does not deploy one. Your agent must:

- Be a **hosted** Foundry agent exposing the **Responses** protocol.
- Accept a body of `{ "input": "<text>" }`.
- (For the execution use-case) be able to commit to Azure DevOps and trigger a pipeline — i.e. have
  the equivalent of the reference `test-executor` agent's tools and an `AZURE_DEVOPS_PAT`.

The invocation URL pattern is:

```
https://<FOUNDRY_RESOURCE>.services.ai.azure.com/api/projects/<PROJECT>/agents/<AGENT_NAME>/endpoint/protocols/openai/responses?api-version=2025-11-15-preview
```

- `<FOUNDRY_RESOURCE>` — your Foundry account name (subdomain of `*.services.ai.azure.com`).
- `<PROJECT>` — your Foundry project name.
- `<AGENT_NAME>` — the hosted agent to invoke (reference solution used `test-executor`).
- The endpoint routes to the agent's `@latest` version unless you pin a version.

**Quick manual test** (uses *your* Azure login token, to confirm the agent works before wiring the
workflow):
```bash
TOKEN=$(az account get-access-token --resource https://ai.azure.com --query accessToken -o tsv)
curl -X POST \
  "https://<FOUNDRY_RESOURCE>.services.ai.azure.com/api/projects/<PROJECT>/agents/<AGENT_NAME>/endpoint/protocols/openai/responses?api-version=2025-11-15-preview" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{ "input": "Say OK" }'
```
A `status: completed` response with an assistant message confirms the endpoint. Record the full URL
for Part B.

---

## A4. Network reachability

Power Platform calls the Foundry endpoint over the public internet from Microsoft-managed IPs.

- The Foundry endpoint (`*.services.ai.azure.com`) must be **reachable from Power Platform**.
- If your Foundry account has **public network access disabled** or is locked to a private
  endpoint / IP allow-list, the workflow's HTTP call will fail. Either allow public access or add the
  appropriate exception so Power Platform outbound traffic can reach it.
- The same applies to Azure DevOps if you use the execution hand-off / return-sync workflows.

---

## Next

Platform groundwork done. Continue to **[Part B: Power Platform / Copilot Studio](SETUP-POWER-PLATFORM.md)** —
the quick part.
