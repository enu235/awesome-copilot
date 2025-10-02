---
mode: 'agent'
description: 'Quick-start Azure AI Foundry Hub and single Project deployment with basic private endpoints'
---

# Azure AI Foundry Quick Start

Deploy a minimal Azure AI Foundry environment (Hub + 1 Project) with private endpoints for rapid prototyping and testing.

## What This Does

Creates a basic Azure AI Foundry setup:
- 1 Azure AI Foundry Hub (Cognitive Services account with AIServices kind)
- 1 Project (child resource under Hub)
- Storage Account with private endpoints (blob + file)
- Key Vault with private endpoint
- VNet with 2 subnets
- 4 Private DNS zones
- Basic RBAC assignments

**Deployment Model**: Cognitive Services (simple, AI-first)
**Environment**: Development/Testing
**Estimated Cost**: ~$50/month
**Deployment Time**: ~30 minutes

## Prerequisites

- Azure subscription with Owner or Contributor rights
- Azure CLI installed or use Cloud Shell
- Target resource group (will be created if doesn't exist)

## Parameters

**Required**:
- `workloadName`: Short name for resources (e.g., "ai-demo") [2-8 characters]
- `location`: Azure region (e.g., "eastus", "westus2")
- `resourceGroupName`: Target resource group name

**Optional**:
- `projectName`: Project name (default: `${workloadName}-project`)
- `vnetCidr`: VNet CIDR (default: "10.0.0.0/16")

## Deployment Steps

### 1. Gather Parameters

Prompt user for required parameters if not provided:

```
Please provide the following:

1. Workload Name (2-8 chars, e.g., "aidemo"): _______
2. Azure Region (e.g., "eastus"): _______
3. Resource Group Name: _______

Optional:
4. VNet CIDR [10.0.0.0/16]: _______
```

### 2. Generate Minimal Bicep Template

Create `quickstart.bicep` with minimal configuration:

```bicep
targetScope = 'resourceGroup'

param workloadName string
param location string = resourceGroup().location
param projectName string = '${workloadName}-project'

var uniqueSuffix = substring(uniqueString(resourceGroup().id), 0, 4)
var hubName = '${workloadName}-hub-${uniqueSuffix}'
var storageName = toLower('${workloadName}st${uniqueSuffix}')
var kvName = '${workloadName}-kv-${uniqueSuffix}'
var vnetName = '${workloadName}-vnet'

// VNet
resource vnet 'Microsoft.Network/virtualNetworks@2023-11-01' = {
  name: vnetName
  location: location
  properties: {
    addressSpace: { addressPrefixes: ['10.0.0.0/16'] }
    subnets: [
      {
        name: 'hub-subnet'
        properties: {
          addressPrefix: '10.0.1.0/24'
          privateEndpointNetworkPolicies: 'Disabled'
        }
      }
      {
        name: 'services-subnet'
        properties: {
          addressPrefix: '10.0.2.0/24'
          privateEndpointNetworkPolicies: 'Disabled'
        }
      }
    ]
  }
}

// Storage
resource storage 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: storageName
  location: location
  kind: 'StorageV2'
  sku: { name: 'Standard_LRS' }
  properties: {
    publicNetworkAccess: 'Disabled'
    minimumTlsVersion: 'TLS1_2'
  }
}

// Key Vault
resource keyVault 'Microsoft.KeyVault/vaults@2023-07-01' = {
  name: kvName
  location: location
  properties: {
    sku: { family: 'A', name: 'standard' }
    tenantId: tenant().tenantId
    publicNetworkAccess: 'Disabled'
    enableRbacAuthorization: true
  }
}

// AI Foundry Hub
resource hub 'Microsoft.CognitiveServices/accounts@2024-04-01-preview' = {
  name: hubName
  location: location
  kind: 'AIServices'
  sku: { name: 'S0' }
  identity: { type: 'SystemAssigned' }
  properties: {
    publicNetworkAccess: 'Disabled'
    allowProjectManagement: true
    networkAcls: {
      defaultAction: 'Deny'
    }
  }
}

// AI Project
resource project 'Microsoft.CognitiveServices/accounts/projects@2024-04-01-preview' = {
  parent: hub
  name: projectName
  location: location
  identity: { type: 'SystemAssigned' }
  properties: {
    displayName: projectName
  }
}

// Private DNS Zones
resource cogServicesDnsZone 'Microsoft.Network/privateDnsZones@2020-06-01' = {
  name: 'privatelink.cognitiveservices.azure.com'
  location: 'global'
}

resource blobDnsZone 'Microsoft.Network/privateDnsZones@2020-06-01' = {
  name: 'privatelink.blob.core.windows.net'
  location: 'global'
}

// ... (DNS zone VNet links and private endpoints would follow)

output hubId string = hub.id
output projectId string = project.id
```

### 3. Deploy to Azure

```bash
# Create resource group
az group create --name <rg-name> --location <location>

# Deploy template
az deployment group create \
  --resource-group <rg-name> \
  --template-file quickstart.bicep \
  --parameters workloadName=<workload> location=<location>
```

### 4. Configure RBAC

```bash
# Get managed identity IDs
HUB_MI=$(az cognitiveservices account show -g <rg> -n <hub-name> --query identity.principalId -o tsv)
PROJECT_MI=$(az cognitiveservices account show -g <rg> -n <hub-name> --query identity.principalId -o tsv)

# Assign permissions
az role assignment create --assignee $HUB_MI --role "Storage Blob Data Contributor" --scope <storage-id>
az role assignment create --assignee $HUB_MI --role "Key Vault Secrets User" --scope <kv-id>
```

### 5. Verify Deployment

```bash
# Check Hub status
az cognitiveservices account show -g <rg> -n <hub-name>

# Access portal
echo "Access: https://ai.azure.com"
echo "Project: <project-name>"
```

## Output

```
✅ Quick Start Complete!

Resources Created:
- Hub: <hub-name>
- Project: <project-name>
- Storage: <storage-name>
- Key Vault: <kv-name>
- VNet: <vnet-name>
- Private Endpoints: 3

Access:
Portal: https://ai.azure.com
Project: <project-name>

Estimated Cost: ~$50/month

Next Steps:
1. Browse model catalog
2. Deploy a model (GPT-4, etc.)
3. Create a prompt flow
```

## Cleanup

```bash
az group delete --name <rg-name> --yes --no-wait
```

## Notes

- This is a minimal deployment for quick testing
- For production, use the full workflow chatmodes (Plan → Validate → Review → Deploy → Confirm)
- Private endpoint DNS may take 5-10 minutes to propagate
- Default SKUs are cost-optimized (not production-ready)
