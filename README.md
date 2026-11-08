# Cyolo Microsoft Sentinel Solutions

This repository contains the Cyolo data connector solutions for Microsoft Sentinel. They let you ingest Cyolo Zero-Trust Access logs into your Sentinel workspace for security monitoring, threat detection, and incident investigation.

The connectors are built on Microsoft's **Codeless Connector Platform (CCP)** using the **CCF (Codeless Connector Framework)** file layout, and use the Cyolo REST API to pull logs on a scheduled basis.

## Solutions

This repository ships two solutions, each following the [CCF solution structure](https://github.com/Azure/Azure-Sentinel/tree/master/Solutions/DomainTools%20CCF):

| Solution | Log type | API endpoint | Custom table |
|----------|----------|--------------|--------------|
| **Cyolo Activity** | User access events, authentication, sessions | `/v1/activity_log` | `CyoloActivity_CL` |
| **Cyolo Audit** | Admin actions, configuration changes, compliance | `/v1/audit_log` | `CyoloAudit_CL` |

## Repository Structure

Each solution follows the CCF layout, with the CCF Data Connectors folder containing only the four required files:

```
Solutions/
├── Cyolo Activity/
│   ├── Data Connectors/
│   │   └── CyoloActivityLogs_ccf/
│   │       ├── CyoloActivityLogs_ConnectorDefinition.json
│   │       ├── CyoloActivityLogs_DCR.json
│   │       ├── CyoloActivityLogs_PollingConfig.json
│   │       └── CyoloActivityLogs_Table.json
│   ├── Data/
│   │   └── Solution_CyoloActivity.json
│   ├── SolutionMetadata.json
│   └── ReleaseNotes.md
│
└── Cyolo Audit/
    ├── Data Connectors/
    │   └── CyoloAuditLogs_ccf/
    │       ├── CyoloAuditLogs_ConnectorDefinition.json
    │       ├── CyoloAuditLogs_DCR.json
    │       ├── CyoloAuditLogs_PollingConfig.json
    │       └── CyoloAuditLogs_Table.json
    ├── Data/
    │   └── Solution_CyoloAudit.json
    ├── SolutionMetadata.json
    └── ReleaseNotes.md
```

> The `Package/` folder (with `mainTemplate.json`, `CreateUiDefinition.json`, and `3.x.x.zip`) is generated automatically by the V3 packaging tool and is not committed here. `SolutionMetadata.json`, the `Data/` solution file, and `ReleaseNotes.md` are authored by Cyolo before packaging.

## Prerequisites

Before you start, you'll need:

1. An Azure subscription.
2. **An existing Log Analytics workspace with Microsoft Sentinel enabled** (required).
3. Cyolo API credentials (Key ID and Secret) with log access permissions.

### Get Cyolo API credentials

1. Log in to the Cyolo Admin Console with an Admin user.
2. Go to **Identities** > **API Keys** and create (or reuse) a key.
3. Under **Roles** > **Admin**, assign the **Log Admin** role to the key.
4. Copy the API Key ID and Secret.

## Building the packaged solution

These solutions use Microsoft's V3 packaging tool. The tool combines the Connector Definition, DCR, Polling Config, and Table JSON files into an ARM template and generates the `Package` folder.

1. Clone the [Azure-Sentinel](https://github.com/Azure/Azure-Sentinel) repository and copy a solution folder (e.g. `Solutions/Cyolo Activity`) into it.
2. Run the [V3 packaging tool](https://github.com/Azure/Azure-Sentinel/tree/master/Tools/Create-Azure-Sentinel-Solution/V3). It generates `mainTemplate.json`, `CreateUiDefinition.json`, and `3.x.x.zip` under `Package/`.
3. See the [CCF README](https://github.com/Azure/Azure-Sentinel/blob/master/Tools/Create-Azure-Sentinel-Solution/V3/CCF_README.md) for CCF best practices.

## Validating the connector

After packaging, validate the CCF Data Connector by deploying `mainTemplate.json` via **Deploy a custom template** in the Azure portal. Once deployment completes:

1. Go to the **Data Connectors** tab in Microsoft Sentinel and select the Cyolo CCF Data Connector.
2. Click **Open Connector Page**.
3. Enter your API URL and credentials, then select **Connect**.

If the connection is successful, the connector deploys the required Data Collection Rule (DCR), Data Collection Endpoint (DCE), and custom table, and its status changes to **Connected**. Ingestion then lands in the custom table, which you can query from the **Logs** section in Microsoft Sentinel.

## Sample Queries

**Get recent activity events**

```kusto
CyoloActivity_CL
| take 10
```

**Count events by kind**

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

**Recent admin/configuration changes (audit)**

```kusto
CyoloAudit_CL
| project TimeGenerated, subjectName, Action, objectKind, objectName, Result
| order by TimeGenerated desc
```
