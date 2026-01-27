# Cyolo Microsoft Sentinel Solution

This repository contains the Cyolo data connector solution for Microsoft Sentinel. It allows you to ingest Cyolo Zero-Trust Access logs into your Sentinel workspace for security monitoring, threat detection, and incident investigation.

The connector is built on Microsoft's Codeless Connector Platform (CCP) and uses the Cyolo REST API to pull activity logs on a scheduled basis.

## What's Included

- **Data Connectors** - REST API pollers that fetch logs from Cyolo and push them to Sentinel
- **Custom Tables** - Pre-defined table schemas for storing Cyolo events
- **Data Collection Rules** - Transformation rules that parse and normalize incoming data
- **Workbooks** - Dashboards for visualizing access patterns and activity trends

## Repository Structure

```
Solutions/
├── Cyolo/                              # Main solution (activity logs)
│   ├── Data Connectors/
│   │   ├── CyoloDataConnector.json
│   │   ├── CyoloDataConnectorDefinition.json
│   │   ├── DCR.json
│   │   └── table.json
│   ├── Workbooks/
│   │   └── CyoloAccessInsights.json
│   ├── Metadata/
│   │   └── SolutionMetadata.json
│   ├── CreateUiDefinition.json
│   └── CyoloSolution.json
│
└── CyoloSentinelSolution/              # Extended solution (multiple log types)
    ├── Data Connectors/
    │   └── CyoloSentinel_ccp/
    │       ├── DataConnectorDefinition.json
    │       ├── PollingConfig.json
    │       ├── table_analytics.json
    │       ├── table_audit.json
    │       └── table_system.json
    └── Package/
        ├── mainTemplate.json
        └── createUiDefinition.json
```

## Data Tables

The solution can create the following custom tables in your Log Analytics workspace (depending on which log streams you enable):

| Table | Description | Enable Option |
|-------|-------------|---------------|
| `CyoloActivity_CL` | User access events, authentication, sessions | Basic solution |
| `CyoloSentinelAnalyticsTable_CL` | Behavioral analytics data | `enableAnalyticsLogs` |
| `CyoloSentinelAuditTable_CL` | Admin and configuration changes | `enableAuditLogs` |
| `CyoloSentinelSystemTable_CL` | System health and operational events | `enableSystemLogs` |

> **Note:** Only the tables for enabled log streams will be created during deployment.

## Prerequisites

Before you start, you'll need:

1. An Azure subscription
2. **An existing Log Analytics workspace with Microsoft Sentinel enabled** (required)
3. Cyolo API credentials (Key ID and Secret) with log access permissions

> **Important:** You must create your Log Analytics workspace and enable Microsoft Sentinel on it **before** deploying this solution.

### Creating a Workspace and Enabling Sentinel

If you don't have a workspace yet:

**Create Log Analytics Workspace:**
```bash
az monitor log-analytics workspace create \
  --resource-group <your-resource-group> \
  --workspace-name <your-workspace-name> \
  --location <region> \
  --sku PerGB2018
```

**Enable Microsoft Sentinel:**
1. Go to Azure Portal > Search for "Microsoft Sentinel"
2. Click "Add Microsoft Sentinel to a workspace"
3. Select your workspace
4. Click "Add"

To get API credentials from Cyolo:

1. Log into the Cyolo Admin Console
2. Go to **Identities** > **API Keys**
3. Create a new key or use an existing one
4. Make sure the key has the **Log Admin** role assigned under **Roles** > **Admin**

## Installation

### Selecting Log Streams

The **CyoloSentinelSolution** supports selective log stream ingestion. **Log stream selection happens during ARM template deployment**, not in the Sentinel UI.

**Available Log Streams:**
- **Analytics Logs** - Behavioral analytics, user activity patterns, and access trends
- **Audit Logs** - Administrative actions, configuration changes, and compliance events  
- **System Logs** - System health, performance metrics, and operational diagnostics

**Benefits:**
- **Reduce costs** by ingesting only the logs you need
- **Focus monitoring** on specific security areas
- **Optimize performance** with smaller data volumes

**How It Works:**
- During deployment, you set boolean parameters (`enableAnalyticsLogs`, `enableAuditLogs`, `enableSystemLogs`) to `true` or `false`
- Each enabled log stream creates a separate data connector and custom table in Sentinel
- After deployment, you configure API credentials for each enabled connector in the Sentinel UI

> 📖 **For detailed guidance on log stream selection, see [LOG_STREAM_SELECTION_GUIDE.md](LOG_STREAM_SELECTION_GUIDE.md)**

### Quick Deploy

**Prerequisites:** You must have an existing Log Analytics workspace with Microsoft Sentinel enabled.

**One-Click Deploy (Recommended):**

Click the button below to deploy directly to Azure Portal with the full configuration UI:

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fcyolosecurity%2Fcyolo-sentinel%2Fmain%2FSolutions%2FCyoloSentinelSolution%2FPackage%2FmainTemplate.json/createUIDefinitionUri/https%3A%2F%2Fraw.githubusercontent.com%2Fcyolosecurity%2Fcyolo-sentinel%2Fmain%2FSolutions%2FCyoloSentinelSolution%2FPackage%2FcreateUiDefinition.json)

This will open Azure Portal with a guided deployment form where you can:
- Select your existing workspace
- Choose which log streams to enable with checkboxes
- Provide your Cyolo API credentials

---

**Alternative: Manual Deploy**

If the button doesn't work or you prefer manual deployment:

1. **Download the template**
   - Get `Solutions/CyoloSentinelSolution/Package/mainTemplate.json` from this repository

2. **Go to Azure Portal Custom Deployment**
   - Navigate to: https://portal.azure.com/#create/Microsoft.Template
   - Click **"Build your own template in the editor"**
   - Paste the contents of mainTemplate.json
   - Click **"Save"**

3. **Fill in Basic Settings**
   - **Subscription**: Your Azure subscription
   - **Resource group**: The resource group containing your Sentinel workspace
   - **Region**: Your Azure region
   - **Workspace**: Your existing Log Analytics workspace name
   - **Workspace Location**: Same region as your workspace

4. **🎯 Select Log Streams (Deployment-Time Configuration)**
   
   In the deployment form, you'll see these boolean parameters - set each to `true` or `false`:
   
   - **Enable Analytics Logs**: 
     - `true` = Creates CyoloSentinelAnalyticsTable_CL and connector for behavioral analytics
     - `false` = Skip Analytics logs
   
   - **Enable Audit Logs**: 
     - `true` = Creates CyoloSentinelAuditTable_CL and connector for admin/config changes
     - `false` = Skip Audit logs
   
   - **Enable System Logs**: 
     - `true` = Creates CyoloSentinelSystemTable_CL and connector for system health
     - `false` = Skip System logs
   
   > **Important**: These selections happen **during deployment**. Each enabled log stream creates a separate data connector in Sentinel. To change later, you must redeploy.

5. **Provide Cyolo API Credentials**
   - **Api Url**: `https://console.YOUR-TENANT.cyolo.io`
   - **Api Id**: Your Cyolo API Key ID
   - **Api Token**: Your Cyolo API Secret Token

6. **Review and Create**
   - Click **"Review + create"**
   - Review all parameters
   - Click **"Create"**

**After Deployment:**
- Go to Microsoft Sentinel → Data connectors
- You'll see separate connectors for each enabled log stream:
  - CyoloAnalytics (if enabled)
  - CyoloAudit (if enabled)
  - CyoloSystem (if enabled)
- Each connector will have its configuration page where you verify API settings

### Manual Deployment via CLI

If you prefer to deploy step by step, here's the process.

**1. Create your Log Analytics workspace and enable Sentinel**

```bash
# Create workspace
az monitor log-analytics workspace create \
  --resource-group <your-resource-group> \
  --workspace-name <your-workspace> \
  --location <region> \
  --sku PerGB2018

# Enable Sentinel (via Azure Portal or separate template)
```

**2. Deploy the solution**

```bash
az deployment group create \
  --resource-group <your-resource-group> \
  --template-file Solutions/CyoloSentinelSolution/Package/mainTemplate.json \
  --parameters \
    workspace="<your-workspace>" \
    workspace-location="<region>" \
    enableAnalyticsLogs=true \
    enableAuditLogs=true \
    enableSystemLogs=true
```

### Legacy Step-by-Step Deployment

If you need more control, you can deploy components individually:

**1. Create the custom tables**

```bash
az deployment group create \
  --resource-group <your-resource-group> \
  --template-file Solutions/Cyolo/Data\ Connectors/table.json \
  --parameters workspaceName="<your-workspace>"
```

**2. Create a Data Collection Endpoint**

```bash
az monitor data-collection endpoint create \
  --resource-group <your-resource-group> \
  --location <region> \
  --name Cyolo-DCE \
  --public-network-access Enabled
```

**3. Create the Data Collection Rule**

```bash
az monitor data-collection rule create \
  --resource-group <your-resource-group> \
  --location <region> \
  --name Cyolo-DCR \
  --rule-file Solutions/Cyolo/Data\ Connectors/DCR.json
```

**4. Deploy the connector definition**

```bash
az deployment group create \
  --resource-group <your-resource-group> \
  --template-file Solutions/Cyolo/Data\ Connectors/CyoloDataConnectorDefinition.json \
  --parameters workspace="<your-workspace>" \
               workspace-location="<region>"
```

**5. Deploy the data connector**

```bash
az deployment group create \
  --resource-group <your-resource-group> \
  --template-file Solutions/Cyolo/Data\ Connectors/CyoloDataConnector.json \
  --parameters workspace="<your-workspace>" \
               workspace-location="<region>" \
               username="<cyolo-api-key-id>" \
               password="<cyolo-api-secret>" \
               dataCollectionEndpoint="<your-dce-url>" \
               dataCollectionRuleImmutableId="<dcr-immutable-id>"
```

## Sample Queries

Once data starts flowing in, you can query it using KQL. Here are some examples:

**Get recent events**

```kusto
CyoloActivity_CL
| take 10
```

**Count events by type**

```kusto
CyoloActivity_CL
| summarize count() by eventKind
| order by count_ desc
```

**Find failed authentication attempts**

```kusto
CyoloActivity_CL
| where Result == "failure" and Action contains "auth"
| project TimeGenerated, subjectName, remoteAddress, countryCode, Action
| order by TimeGenerated desc
```

**See which applications are accessed most**

```kusto
CyoloActivity_CL
| where objectKind == "application"
| summarize AccessCount = count() by objectName
| top 10 by AccessCount
```

**Activity by country**

```kusto
CyoloActivity_CL
| summarize EventCount = count() by countryCode
| order by EventCount desc
```
