# Defender for Cloud → ServiceNow Risk Enrichment (Logic App)

This Logic App enriches ServiceNow incidents that originate from **Microsoft Defender for Cloud** with the **contextual risk level** of the referenced recommendation (assessment). It pulls incidents from the last 24 hours from ServiceNow, extracts the **full assessment `id`**, queries **Azure Resource Graph (ARG)** for that exact item, and updates the incident with the `risk_level` (Critical/High/Medium/Low or Unknown).

- Azure Resource Graph is invoked via the **ARM endpoint** and returns results under a `data` array—this is the documented REST contract.
- The data is queried from the **`securityresources`** table which surfaces Microsoft Defender for Cloud entities (assessments, subassessments, etc.). Filtering on **`id =~ '<full id>'`** targets the per‑resource assessment instance.
- The HTTP action authenticates with **Managed Identity** and uses the **audience** `https://management.azure.com`.

---

## Prerequisites

1. An Azure subscription and a resource group where you can deploy.  
2. Permission to **create a Logic App** and **create a Microsoft.Web/connection** (ServiceNow).  
3. Permission to assign **Reader** on the subscription(s) you will query with ARG (for the Logic App’s managed identity). ARG only returns what the caller can see (RBAC‑scoped).   
4. Your **ServiceNow** instance details and credentials to authorize the connection after deployment.
5. Microsoft Defender for Cloud enabled on the subscription(s) you will query with the Defender CSPM plan (which includes risk prioritization).
6. Built-in integration between Microsoft Defender for Cloud and ServiceNow set up, so that incidents are created in ServiceNow from Defender recommendations (see [Integrate with ServiceNow](https://learn.microsoft.com/en-us/azure/defender-for-cloud/integrate-with-servicenow)).
7. Existing ServiceNow incidents created from Microsoft Defender for Cloud recommendations to validate the enrichment.

---

## What this template deploys

- **Logic App (Consumption)** with **system‑assigned Managed Identity**.  
- **ServiceNow connection** (Microsoft.Web/connections). After deployment, you’ll authorize it in the portal.  
- Logic App **parameters**:
  - `arg_subscriptions`: array of subscription IDs for the ARG query.
  - `$connections`: automatically points the workflow to the ServiceNow connection you deploy.

---

## Deploy
You can deploy this ARM template by clicking on the blue **Deploy to Azure** button below, via the Azure Portal or Azure CLI.

### Option A — ARM Template

<a href="https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Feimerperalta%2FMDC-ServiceNow-Risk-Enrichment%2Frefs%2Fheads%2Fmain%2Fazuredeploy.json" target="_blank">
    <img src="https://aka.ms/deploytoazurebutton"/>
</a>

### Option B — Azure Portal
1. Go to **Deploy a custom template** in the Azure portal.  
2. Upload `azuredeploy.json`.  
3. Fill parameters:
   - **logicAppName**: e.g., `mdc-snow-risk-enrichment`
   - **location**: your region
   - **serviceNowConnectionName**: e.g., `service-now`
   - **arg_subscriptions**: e.g., `["<your-subscription-id>"]`
4. Click **Review + create** → **Create**.

### Option C — Azure CLI
```bash
# Set variables
RG=<your-resource-group> # e.g. myResourceGroup
LOC=<region> # e.g. eastus
APP=mdc-snow-risk-enrichment
CONN=service-now
SUBS='["<your-subscription-id>"]'

az deployment group create \
  --resource-group $RG \
  --template-file azuredeploy.json \
  --parameters logicAppName=$APP location=$LOC serviceNowConnectionName=$CONN arg_subscriptions="$SUBS"
```