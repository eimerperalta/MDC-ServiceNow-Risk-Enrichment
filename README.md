# Defender for Cloud → ServiceNow Risk Enrichment (Logic App)

This Logic App enriches ServiceNow incidents that originate from **Microsoft Defender for Cloud** with the **contextual risk level** of the referenced recommendation (assessment). It pulls the last-24h incidents from ServiceNow, extracts the **full assessment `id`**, queries **Azure Resource Graph (ARG)** for that exact item, and updates the incident with the `risk_level` (High/Medium/Low or Unknown).

- Azure Resource Graph is invoked via the **ARM endpoint** and returns results under a `data` array—this is the documented REST contract. [1](https://learn.microsoft.com/en-us/rest/api/azureresourcegraph/resourcegraph/resources/resources?view=rest-azureresourcegraph-resourcegraph-2024-04-01)  
- The data is queried from the **`securityresources`** table which surfaces Microsoft Defender for Cloud entities (assessments, subassessments, etc.). Filtering on **`id =~ '<full id>'`** targets the per‑resource assessment instance. [2](https://learn.microsoft.com/en-us/azure/defender-for-cloud/resource-graph-samples)[3](https://learn.microsoft.com/en-us/azure/governance/resource-graph/reference/supported-tables-resources)  
- The HTTP action authenticates with **Managed Identity** and uses the **audience** `https://management.azure.com`. [4](https://learn.microsoft.com/en-us/azure/logic-apps/authenticate-with-managed-identity)

---

## Prerequisites

1. An Azure subscription and a resource group where you can deploy.  
2. Permission to **create a Logic App** and **create a Microsoft.Web/connection** (ServiceNow).  
3. Permission to assign **Reader** on the subscription(s) you will query with ARG (for the Logic App’s managed identity). ARG only returns what the caller can see (RBAC‑scoped). [1](https://learn.microsoft.com/en-us/rest/api/azureresourcegraph/resourcegraph/resources/resources?view=rest-azureresourcegraph-resourcegraph-2024-04-01)  
4. Your **ServiceNow** instance details and credentials to authorize the connection after deployment.

---

## What this template deploys

- **Logic App (Consumption)** with **system‑assigned Managed Identity**.  
- **ServiceNow connection** (Microsoft.Web/connections). After deployment, you’ll authorize it in the portal.  
- Logic App **parameters**:
  - `arg_subscriptions`: array of subscription IDs for the ARG query.
  - `$connections`: automatically points the workflow to the ServiceNow connection you deploy.

---

## Deploy

### Option A — Azure Portal
1. Go to **Deploy a custom template** in the Azure portal.  
2. Upload `template.json`.  
3. Fill parameters:
   - **logicAppName**: e.g., `mdc-snow-risk-enrichment`
   - **location**: your region
   - **serviceNowConnectionName**: e.g., `service-now`
   - **arg_subscriptions**: e.g., `["<your-subscription-id>"]`
4. Click **Review + create** → **Create**.

### Option B — Azure CLI
```bash
# Set variables
RG=<your-resource-group>
LOC=<region> # e.g. eastus
APP=mdc-snow-risk-enrichment
CONN=service-now
SUBS='["<your-subscription-id>"]'

az deployment group create \
  --resource-group $RG \
  --template-file template.json \
  --parameters logicAppName=$APP location=$LOC serviceNowConnectionName=$CONN arg_subscriptions="$SUBS"
```