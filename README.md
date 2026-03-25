# Defender for Cloud → ServiceNow Risk Enrichment (Logic App)

This ARM template deploys a **Logic App (Consumption)** that automatically enriches **ServiceNow** incidents originating from **Microsoft Defender for Cloud** with the **contextual risk level** of the referenced recommendation.

Every day, the Logic App:

1. Pulls active incidents opened in the last 24 hours from ServiceNow.
2. Extracts the **assessment ID** from each incident description.
3. Queries **Azure Resource Graph (ARG)** for the matching assessment's risk level.
4. Updates the ServiceNow incident with a comment: `Risk level (MDC): Critical | High | Medium | Low | Unknown`.

### How it works

| Component | Detail |
|---|---|
| **ARG endpoint** | `https://management.azure.com/providers/Microsoft.ResourceGraph/resources?api-version=2024-04-01` (ARM REST API) |
| **ARG table** | `securityresources` — surfaces Defender for Cloud assessments, sub-assessments, etc. |
| **Query filter** | `id =~ '<full assessment id>'` — targets the per-resource assessment instance |
| **Authentication** | System-assigned **Managed Identity** with audience `https://management.azure.com` |
| **Retry policy** | Exponential back-off: up to 4 retries, 10 s initial / 1 min max interval |

---

## What this template deploys

| Resource | Type | Notes |
|---|---|---|
| Logic App (Consumption) | `Microsoft.Logic/workflows` | System-assigned Managed Identity enabled |

> **Note:** This template does **not** create a ServiceNow API connection. You must create one beforehand and supply its resource ID as a parameter (see [Pre-Deployment Checklist](#pre-deployment-checklist)).

---

## Prerequisites

| # | Requirement | Details |
|---|---|---|
| 1 | **Azure subscription & resource group** | Where the Logic App will be deployed. |
| 2 | **Contributor** role | Permission to create Logic Apps and API connections (`Microsoft.Web/connections`). |
| 3 | **User Access Administrator** role | To assign the **Security Reader** role to the Logic App's managed identity on target subscriptions. ARG only returns data the caller can see (RBAC-scoped). |
| 4 | **ServiceNow instance** | Instance URL and credentials to authorize the API connection after deployment. |
| 5 | **Microsoft Defender for Cloud** | Enabled on the target subscription(s) with the **Defender CSPM** plan (includes risk prioritization). |
| 6 | **Defender ↔ ServiceNow integration** | Incidents must already be created in ServiceNow from Defender recommendations. See [Connect ServiceNow to Defender for Cloud](https://learn.microsoft.com/en-us/azure/defender-for-cloud/connect-servicenow). |
| 7 | **Existing ServiceNow incidents** | At least one Defender-created incident to validate the enrichment after deployment. |

---

## Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `Logic App name` | String | `Update-SNow-Incident-With-Risk` | Name for the Logic App resource. Must be unique within the resource group. |
| `location` | String | Resource group location | Azure region for the Logic App. **Must match** the region of your ServiceNow API connection. |
| `arg_subscriptions` | Array | Current subscription | Subscription IDs that ARG will query for Defender for Cloud assessment data. |
| `recurrence_timezone` | String | `UTC` | Time zone for the daily recurrence trigger. Allowed values: `UTC`, `Eastern Standard Time`, `Central Standard Time`, `Mountain Standard Time`, `Pacific Standard Time`, `GMT Standard Time`, `W. Europe Standard Time`, `AUS Eastern Standard Time`, `Tokyo Standard Time`, `India Standard Time`, `China Standard Time`, `Arab Standard Time`, `E. Africa Standard Time`, `SA Pacific Standard Time`, `SE Asia Standard Time`. |
| `SNow_API_connection_resource_ID` | String | *(required)* | Full resource ID of an **existing** ServiceNow API connection. Format: `/subscriptions/{subscriptionId}/resourceGroups/{resourceGroup}/providers/Microsoft.Web/connections/{connectionName}` |

---

## Pre-Deployment Checklist

Complete steps 1–3 **before** deploying. Steps 4–6 are performed **after** deployment.

| Step | When | Action |
|---|---|---|
| 1 | Before | **Create a ServiceNow API connection** in Azure: Portal → Logic Apps → API Connections → Add → ServiceNow. Note its full resource ID — you will need it for the `SNow_API_connection_resource_ID` parameter. |
| 2 | Before | **Identify the Azure region** to deploy into. It must match the region of the ServiceNow API connection. |
| 3 | Before | **List all subscription IDs** where Defender for Cloud assessments exist. These go into the `arg_subscriptions` parameter. |
| 4 | After | **Grant Security Reader** to the Logic App's system-assigned managed identity on each subscription in `arg_subscriptions`: Portal → Subscription → Access control (IAM) → Add role assignment. |
| 5 | After | **Authorize the ServiceNow API connection**: Portal → Logic App → API connections → service-now-1 → Edit API connection → Authorize → Save. |
| 6 | After | **Run a manual test** and verify in ServiceNow that incidents are updated with the correct `Risk level (MDC):` comment. |

---

## Deploy

### Option A — Deploy to Azure (one-click)

<a href="https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Feimerperalta%2FMDC-ServiceNow-Risk-Enrichment%2Frefs%2Fheads%2Fmain%2Fazuredeploy.json" target="_blank">
    <img src="https://aka.ms/deploytoazurebutton"/>
</a>

### Option B — Azure Portal (manual upload)

1. Go to **Deploy a custom template** in the Azure portal.
2. Click **Build your own template in the editor** and upload `azuredeploy.json`.
3. Fill in the parameters:
   - **Logic App name** — e.g., `Update-SNow-Incident-With-Risk`
   - **location** — e.g., `eastus`
   - **arg_subscriptions** — e.g., `["xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"]`
   - **recurrence_timezone** — e.g., `UTC`
   - **SNow_API_connection_resource_ID** — the full resource ID noted in step 1 of the checklist
4. Click **Review + create** → **Create**.

### Option C — Azure CLI

```bash
# Set variables
RG="<your-resource-group>"
LOC="<region>"                    # e.g. eastus — must match the ServiceNow connection region
APP="Update-SNow-Incident-With-Risk"
SUBS='["<subscription-id>"]'      # e.g. '["xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"]'
SNOW_CONN="/subscriptions/<sub-id>/resourceGroups/<rg>/providers/Microsoft.Web/connections/<conn-name>"

az deployment group create \
  --resource-group "$RG" \
  --template-file azuredeploy.json \
  --parameters \
    "Logic App name=$APP" \
    "location=$LOC" \
    "arg_subscriptions=$SUBS" \
    "SNow_API_connection_resource_ID=$SNOW_CONN"
```

---