---
title: Data Landing Zones
description: Learn about cloud-scale analytics architecture data landing zones in Azure that connect to your data management landing zone via virtual network peering or private endpoints.
author: mboswell
ms.author: mboswell
ms.date: 02/14/2025
ms.topic: conceptual
ms.custom: e2e-data-management, think-tank
---

# Data landing zones

Data landing zones are connected to your [data management landing zone](./data-management-landing-zone.md) by virtual network peering or private endpoints. Each data landing zone is considered a [landing zone](../../../ready/landing-zone/index.md) related to Azure landing zone architecture.

> [!IMPORTANT]
> Before you provision a data landing zone, ensure that your DevOps and continuous integration and continuous delivery (CI/CD) operating model is in place and that a data management landing zone is deployed.

Each data landing zone has several layers that enable agility for the service data integrations and data applications it contains. You can deploy a new data landing zone with a standard set of services that allow the data landing zone to ingest and analyze data.

The following table shows the structure of a typical Azure subscription associated with a data landing zone.

| Layer | Required | Resource groups |
|---|---|---|
| [Platform services layer](#platform-services) | Yes | <ul> <li> [Network](#networking) </li> <li> [Security](#security-and-monitoring) </li> </ul> |
| [Core services](#core-services) | Yes | <ul> <li> [Storage](#storage) </li> <li> [Shared integration runtimes (IRs)](#shared-irs) </li> <li> [Management](#management) </li> <li> [External storage](#external-storage) </li> <li> [Data ingestion](#data-ingestion) </li> <li> [Shared applications](#shared-applications) </li> </ul> |
| [Data application](#data-application) | Optional | <ul> <li> [Data application](#data-application-resource-group) (one or more) </li> </ul> |
| [Reporting and visualization](#reporting-and-visualization) | Optional | <ul> <li> [Reporting and visualization](#reporting-and-visualization) </li> </ul> |

> [!NOTE]
> The core services layer is marked as required, but not all resource groups and services included in this article might be necessary for your data landing zone.

## Data landing zone architecture

The following data landing zone architecture illustrates the layers, their resource groups, and the services that each resource group contains. The architecture provides an overview of all groups and roles associated with your data landing zone and the extent of their access to your control and data planes. The architecture also illustrates how each layer aligns with the operating model responsibilities.

:::image type="content" source="../images/data-landing-zone.png" alt-text="Diagram of the data landing zone architecture." border="false" lightbox="../images/data-landing-zone.png":::

> [!TIP]
> Before you deploy a data landing zone, make sure to [consider the number of initial data landing zones that you want to deploy](../../cloud-scale-analytics/architectures/scale-architectures.md).

## Platform services

The platform services layer includes services required to enable connectivity and observability to your data landing zone within the context of cloud-scale analytics. The following table lists the recommended resource groups.

| Resource group | Required | Description |
|----------------|----------|-------------|
| `network-rg` | Yes | Networking |
| `security-rg` | Yes | Security and monitoring |

### Networking

The network resource group contains connectivity services, including [Azure Virtual Network](/azure/virtual-network/virtual-networks-overview), [network security groups](/azure/virtual-network/network-security-groups-overview), and [route tables](/azure/virtual-network/virtual-networks-udr-overview). All these services are deployed into a single resource group.

The virtual network of your data landing zone is [automatically peered with your data management landing zone's virtual network](../eslz-network-topology-and-connectivity.md) and your [connectivity subscription's virtual network](../../../ready/landing-zone/index.md).

### Security and monitoring

The security and monitoring resource group includes [Azure Monitor](/azure/azure-monitor/overview) and [Microsoft Defender for Cloud](/azure/defender-for-cloud/defender-for-cloud-introduction) to collect service telemetry, define monitoring criteria and alerts, and apply policies and scanning to services.

## Core services

The core services layer includes foundational services required to enable your data landing zone within the context of cloud-scale analytics. The following table lists the resource groups that provide the standard suite of available services in every data landing zone that you deploy.

| Resource group | Required | Description |
|----------------|----------|-------------|
| `storage-rg` | Yes | Data lake services |
| `runtimes-rg` | Yes | Shared IRs |
| `mgmt-rg` | Yes | CI/CD agents |
| `external-data-rg` | Yes | External data storage |
| `data-ingestion-rg` | Optional | Shared data ingestion services |
| `shared-applications-rg` | Optional | Shared applications (Azure Databricks) |

### Storage

The previous diagram shows three [Azure Data Lake Storage Gen2](/azure/storage/blobs/data-lake-storage-introduction) accounts provisioned in a single data lake services resource group. Data transformed at different stages is saved in one of your data landing zone's data lakes. The data is available for consumption by your analytics, data science, and visualization teams.

[!INCLUDE [data-lake-layers](../../cloud-scale-analytics/includes/data-lake-layers.md)]

> [!NOTE]
> In the previous diagram, each data landing zone has three data lake storage accounts. Depending on your requirements, you can choose to consolidate your raw, enriched, and curated layers into one storage account and maintain another storage account called *workspace* for data consumers to bring in other useful data products.

For more information, see:

- [Overview of Azure Data Lake Storage for cloud-scale analytics](../best-practices/data-lake-overview.md)
- [Data standardization](../../cloud-scale-analytics/architectures/data-standardization.md)
- [Data lake zones and containers](../../cloud-scale-analytics/best-practices/data-lake-zones.md)
- [Key considerations for Data Lake Storage](../best-practices/data-lake-key-considerations.md)
- [Access control and data lake configurations in Data Lake Storage](../best-practices/data-lake-access.md)

### Shared IRs

Azure Data Factory pipelines use IRs to securely access data sources in peered or isolated networks. Shared IRs should be deployed to a virtual machine (VM) or Azure Virtual Machine Scale Sets in the shared IR resource group.

To enable the shared resource group:

- Create at least one Azure Data Factory instance in your data landing zone's shared integration resource group. Use it only for linking the shared self-hosted IR, not for data pipelines.

- [Create and configure a self-hosted IR](/azure/data-factory/create-self-hosted-integration-runtime) on the VM.

- Associate the self-hosted IR with Azure data factories in your data landing zones.

- Use PowerShell scripts to [periodically update the self-hosted IR](/azure/data-factory/self-hosted-integration-runtime-automation-scripts).

> [!NOTE]
> The deployment describes a single VM deployment that has a self-hosted IR. You can associate a self-hosted IR with multiple VMs on-premises or in Azure. These machines are called nodes. You can have up to four nodes associated with a self-hosted IR. The benefits of having multiple nodes include:
>
> - Higher availability of the self-hosted IR so that it's no longer the single point of failure in your data application or in the orchestration of cloud data integration.
>
> - Improved performance and throughput during data movement between on-premises and cloud data services. For more information, see [Copy activity performance and scalability guide](/azure/data-factory/copy-activity-performance).
>
> You can associate multiple nodes by installing the self-hosted IR software from [Microsoft Download Center](https://www.microsoft.com/download/details.aspx?id=39717). Then register it by using either of the authentication keys obtained from the **New-AzDataFactoryV2IntegrationRuntimeKey** cmdlet, as described in the [tutorial](/azure/data-factory/tutorial-hybrid-copy-powershell).
>
> For more information, see [Azure Data Factory high availability and scalability](/azure/data-factory/create-self-hosted-integration-runtime#high-availability-and-scalability).

Make sure to deploy shared IRs as close to the data source as possible. You can deploy the IRs in a data landing zone, into non-Microsoft clouds, or into a private cloud if the VM has connectivity to the required data sources.

### Management

CI/CD agents run on VMs and help deploy artifacts from the source code repository, including data applications and changes to the data landing zone.

For more information, see [Azure Pipelines agents](/azure/devops/pipelines/agents/agents).

### External storage

Partner data publishers need to land data in your platform so that your data application teams can pull it into their data lakes. You can also have internal or external data sources that can't support the connectivity or authentication requirements enforced across the rest of the data landing zones. The recommended approach is to use a separate storage account to receive data. Then use a shared IR or similar ingestion process to bring it into your processing pipeline.

The data application teams request the storage blobs. These requests get approved by the data landing zone operations team. Data should be deleted from its source storage blob after it's ingested into the raw data storage.

> [!IMPORTANT]
> Because Azure Storage blobs are provisioned on an *as-needed* basis, you should initially deploy an empty storage services resource group in each data landing zone.

### Data ingestion

This resource group is optional and doesn't prevent you from deploying your landing zone. It applies if you have, or are developing, a data-agnostic ingestion engine that automatically ingests data based on registered metadata. This feature includes connection strings, paths for data transfer, and ingestion schedules.

The ingestion and processing resource group has key services for this kind of framework.

Deploy an Azure SQL Database instance to hold metadata that Azure Data Factory uses. Provision an Azure key vault to store secrets related to automated ingestion services. These secrets can include:

- Azure Data Factory metastore credentials.
- Service principal credentials for your automated ingestion process.

For more information, see [Data agnostic ingestion engine](../best-practices/automated-ingestion-pattern.md).

The following table describes services in this resource group.

| Service | Required | Guidelines |
|---------|----------|------------|
| Azure Data Factory | Yes | Azure Data Factory is your orchestration engine for data-agnostic ingestion. |
| Azure SQL Database | Yes | SQL Database is the metastore for Azure Data Factory. |
| Azure Event Hubs or Azure IoT Hub | Optional | Event Hubs or IoT Hub can provide real-time streaming to event hubs, plus batch and streaming processing via an Azure Databricks engineering workspace. |
| Azure Databricks | Optional | You can deploy Azure Databricks to use with your data-agnostic ingestion engine. |

### Shared applications

Use this optional resource group when you need to have a set of shared services made available to all the teams building data applications in this data landing zone. Use cases include:

- An Azure Databricks workspace used as a shared metastore for all other Databricks workspaces created in the same data landing zone or region.

> [!NOTE]
> Azure Databricks uses Unity Catalog to govern access and visibility to metastores across Databricks workspaces. Unity Catalog is enabled at a tenant level, but metastores are aligned with Azure regions. This setup means that all Unity Catalog-enabled Databricks workspaces in a given Azure region must register with the same metastore. For more information, see [Unity Catalog best practices](/azure/databricks/data-governance/unity-catalog/best-practices).

To integrate Azure Databricks, follow cloud-scale analytics best practices. For more information, see [Secure access to Azure Data Lake Gen2 from Azure Databricks](https://github.com/hurtn/datalake-ADLS-access-patterns-with-Databricks/blob/master/readme.md) and [Azure Databricks best practices](https://github.com/Azure/AzureDatabricksBestPractices/blob/master/toc.md).

## Data application

Each data landing zone can have multiple data applications. You can create these applications by ingesting data from various sources. You can also create data applications from other data applications within the same data landing zone or from other data landing zones. The creation of data applications is subject to data steward approval.

### Data application resource group

Your data application resource group includes all the services required to make the data application. For example, an Azure Database is required for MySQL, which is used by a visualization tool. Data must be ingested and transformed before it goes into that MySQL database. In this case, you can deploy Azure Database for MySQL and Azure Data Factory into the data application resource group.

> [!TIP]
> If you decide not to implement a data-agnostic engine for single ingestion from operational sources, or if complex connections aren't supported in your data-agnostic engine, develop a source-aligned [data application](../../cloud-scale-analytics/architectures/data-application-source-aligned.md).

## Reporting and visualization

You can use visualization and reporting tools within Fabric workspaces, which are similar to Power BI workspaces. This feature allows you to avoid deploying unique resources within your data landing zone. You can include a resource group to deploy Fabric capacity, VMs for data gateways, or other necessary data services to deliver your data application to the user.

## Next step

> [!div class="nextstepaction"]
> [Cloud-scale analytics data products in Azure](data-landing-zone-data-products.md)
