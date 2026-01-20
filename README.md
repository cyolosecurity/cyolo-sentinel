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

The solution creates the following custom tables in your Log Analytics workspace:

| Table | Description |
|-------|-------------|
| `CyoloActivity_CL` | User access events, authentication, sessions |
| `CyoloSentinelAnalyticsTable_CL` | Behavioral analytics data |
| `CyoloSentinelAuditTable_CL` | Admin and configuration changes |
| `CyoloSentinelSystemTable_CL` | System health and operational events |

## Prerequisites

Before you start, you'll need:

1. An Azure subscription with a Log Analytics workspace that has Sentinel enabled
2. A Data Collection Endpoint (DCE) in the same region as your workspace
3. Cyolo API credentials (Key ID and Secret) with log access permissions

To get API credentials from Cyolo:

1. Log into the Cyolo Admin Console
2. Go to **Identities** > **API Keys**
3. Create a new key or use an existing one
4. Make sure the key has the **Log Admin** role assigned under **Roles** > **Admin**

## Installation

### Quick Deploy

You can deploy the solution directly from the Azure Portal by clicking the button below:

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fcdn.jsdelivr.net%2Fgh%2Fcyolosecurity%2Fcyolo-sentinel%40main%2FSolutions%2FCyolo%2FCyoloSolution.json)

> **Note:** If you encounter a CORS error with the deploy button, you can manually deploy by:
> 1. Download `Solutions/Cyolo/CyoloSolution.json` from this repository
> 2. Go to [Azure Portal Custom Deployment](https://portal.azure.com/#create/Microsoft.Template)
> 3. Click "Build your own template in the editor"
> 4. Paste the contents of the JSON file and click Save

### Manual Deployment via CLI

If you prefer to deploy step by step, here's the process:

**1. Create the custom table**

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
