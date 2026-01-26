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

The **CyoloSentinelSolution** supports selective log stream ingestion. During deployment, you can choose which log types to enable:

- **Analytics Logs** - Behavioral analytics, user activity patterns, and access trends
- **Audit Logs** - Administrative actions, configuration changes, and compliance events  
- **System Logs** - System health, performance metrics, and operational diagnostics

This allows you to:
- **Reduce costs** by ingesting only the logs you need
- **Focus monitoring** on specific security areas
- **Optimize performance** with smaller data volumes

All log streams are enabled by default, but you can customize this during deployment using the checkboxes in the Azure Portal UI.

### Quick Deploy

**Prerequisites:** You must have an existing Log Analytics workspace with Microsoft Sentinel enabled.

**Deploy the Enhanced Solution:**
1. Download `Solutions/CyoloSentinelSolution/Package/mainTemplate.json` from this repository
2. Go to [Azure Portal Custom Deployment](https://portal.azure.com/#create/Microsoft.Template)
3. Click "Build your own template in the editor"
4. Paste the contents of the JSON file and click Save
5. Fill in the parameters:
   - Select your existing workspace
   - Choose which log streams to enable (Analytics, Audit, System)
   - Provide your Cyolo API credentials (URL, Key ID, Token)
6. Review and Create

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
