# Setup — Azure AI Foundry track (separate; only for the one Foundry workflow)

> 🟢 **You do NOT need any of this to test the Copilot Studio part.**
> The Copilot Studio agent, its skills, and the two **Azure DevOps–driven** workflows
> (**ADO Return Sync** and **Foundry back to Copilot Studio**) all work **without** Azure AI Foundry.
> See **[Power Platform / Copilot Studio setup](SETUP-POWER-PLATFORM.md)** — that's the part to do now.
>
> This Foundry track only enables the single **Calling Foundry Agent** workflow, which sends a test
> case to a Foundry hosted agent for execution. Come back to this **when you have Foundry access**;
> it's an independent track you can sort out separately.

This is a **one-time platform setup** on the Azure side — Azure AI Foundry, an Entra app
registration, and Azure RBAC. It's fully decoupled from the Copilot Studio setup.

> If your Foundry agent and app registration already exist, you just need `TENANT_ID`, `CLIENT_ID`,
> `CLIENT_SECRET`, and the Responses URL, then wire the one workflow per
> [Power Platform → "When you get Foundry access"](SETUP-POWER-PLATFORM.md#when-you-get-foundry-access-enable-the-calling-foundry-agent-workflow).

**What you produce here (needed only for the Calling Foundry Agent workflow):**

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

Keep `TENANT_ID`, `CLIENT_ID`, and `CLIENT_SECRET` (value) to wire the workflow later.

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
to wire the workflow later.

---

## A4. Network reachability

Power Platform calls the Foundry endpoint over the public internet from Microsoft-managed IPs.

- The Foundry endpoint (`*.services.ai.azure.com`) must be **reachable from Power Platform**.
- If your Foundry account has **public network access disabled** or is locked to a private
  endpoint / IP allow-list, the workflow's HTTP call will fail. Either allow public access or add the
  appropriate exception so Power Platform outbound traffic can reach it.

---

## Next

Once you have `TENANT_ID`, `CLIENT_ID`, `CLIENT_SECRET`, and the Responses URL, enable the single
Foundry workflow by following
**[Power Platform → "When you get Foundry access"](SETUP-POWER-PLATFORM.md#when-you-get-foundry-access-enable-the-calling-foundry-agent-workflow)**.
Everything else in Copilot Studio works without this track.
