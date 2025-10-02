---
description: 'Comprehensive Azure Private Endpoint configuration patterns for AI services, storage, networking, and supporting infrastructure'
applyTo: '**/*.bicep, **/*.tf, **/.aif-planning-files/**'
---

# Azure Private Endpoints for AI Infrastructure

## Overview

Private Endpoints provide secure, private connectivity to Azure services over a private IP address in your VNet. This eliminates exposure to the public internet and ensures traffic stays on the Microsoft backbone network.

## Private Endpoint Fundamentals

### How Private Endpoints Work

1. **Private IP Assignment**: PE gets IP from your VNet subnet
2. **DNS Resolution**: Azure Private DNS Zone maps service FQDN to private IP
3. **Network Isolation**: Traffic flows over Microsoft backbone, not internet
4. **NSG Support**: Network Security Groups control PE traffic

### Key Components

```
VNet (10.0.0.0/16)
  └─ Subnet (10.0.1.0/24)
      └─ Private Endpoint (10.0.1.4)
          ├─ Network Interface (NIC)
          ├─ Private DNS Zone Link
          └─ Connection to Azure Service
```

## Azure AI Services Private Endpoints

### Azure AI Foundry Hub (Cognitive Services Model)

**Private Endpoint Type**: `Microsoft.CognitiveServices/accounts` (for AIServices kind)

**Sub-Resources**:
- `account`: AI Foundry Hub endpoint

**Required DNS Zones**:
- `privatelink.cognitiveservices.azure.com`
- `privatelink.openai.azure.com`

**Bicep Example**:
```bicep
resource aiHubPrivateEndpoint 'Microsoft.Network/privateEndpoints@2023-11-01' = {
  name: '${hubName}-pe'
  location: location
  properties: {
    subnet: {
      id: subnetId
    }
    privateLinkServiceConnections: [
      {
        name: '${hubName}-pe-connection'
        properties: {
          privateLinkServiceId: aiHub.id
          groupIds: ['account']
        }
      }
    ]
  }
}

resource aiHubDnsZoneGroup 'Microsoft.Network/privateEndpoints/privateDnsZoneGroups@2023-11-01' = {
  parent: aiHubPrivateEndpoint
  name: 'default'
  properties: {
    privateDnsZoneConfigs: [
      {
        name: 'cognitive-services'
        properties: {
          privateDnsZoneId: cognitiveServicesDnsZone.id
        }
      }
    ]
  }
}
```

**Important**: Projects (child resources) inherit the Hub's private endpoint. No separate PE required for projects.

### Azure AI Foundry Hub (ML Workspaces Model)

**Private Endpoint Type**: `Microsoft.MachineLearningServices/workspaces`

**Sub-Resources**:
- `amlworkspace`: Primary workspace endpoint

**Required DNS Zones**:
- `privatelink.api.azureml.ms`
- `privatelink.notebooks.azure.net`
- `privatelink.cert.api.azureml.ms`
- `privatelink.ml.azure.com`

**Bicep Example**:
```bicep
resource hubPrivateEndpoint 'Microsoft.Network/privateEndpoints@2023-11-01' = {
  name: '${hubName}-pe'
  location: location
  properties: {
    subnet: {
      id: subnetId
    }
    privateLinkServiceConnections: [
      {
        name: '${hubName}-pe-connection'
        properties: {
          privateLinkServiceId: hub.id
          groupIds: ['amlworkspace']
        }
      }
    ]
  }
}

resource hubDnsZoneGroup 'Microsoft.Network/privateEndpoints/privateDnsZoneGroups@2023-11-01' = {
  parent: hubPrivateEndpoint
  name: 'default'
  properties: {
    privateDnsZoneConfigs: [
      {
        name: 'amlworkspace'
        properties: {
          privateDnsZoneId: mlApiDnsZone.id
        }
      }
    ]
  }
}
```

### Azure AI Services (Cognitive Services)

**Private Endpoint Type**: `Microsoft.CognitiveServices/accounts`

**Sub-Resources**:
- `account`: All Cognitive Services endpoints

**Required DNS Zones**:
- `privatelink.cognitiveservices.azure.com`
- `privatelink.openai.azure.com` (for Azure OpenAI)

**Bicep Example**:
```bicep
resource aiServicesPrivateEndpoint 'Microsoft.Network/privateEndpoints@2023-11-01' = {
  name: '${aiServicesName}-pe'
  location: location
  properties: {
    subnet: {
      id: subnetId
    }
    privateLinkServiceConnections: [
      {
        name: '${aiServicesName}-pe-connection'
        properties: {
          privateLinkServiceId: aiServices.id
          groupIds: ['account']
        }
      }
    ]
  }
}

resource aiServicesDnsZoneGroup 'Microsoft.Network/privateEndpoints/privateDnsZoneGroups@2023-11-01' = {
  parent: aiServicesPrivateEndpoint
  name: 'default'
  properties: {
    privateDnsZoneConfigs: [
      {
        name: 'cognitive-services'
        properties: {
          privateDnsZoneId: cognitiveServicesDnsZone.id
        }
      }
    ]
  }
}
```

### Azure OpenAI Service

**Private Endpoint Type**: `Microsoft.CognitiveServices/accounts`

**Sub-Resources**:
- `account`: OpenAI API endpoint

**Required DNS Zones**:
- `privatelink.openai.azure.com`
- `privatelink.cognitiveservices.azure.com`

**Important**: Link both DNS zones even though only one is primary

## Supporting Services Private Endpoints

### Azure Storage Account

**Private Endpoint Type**: `Microsoft.Storage/storageAccounts`

**Sub-Resources** (requires separate PE for each):
- `blob`: Blob storage endpoint
- `file`: File share endpoint
- `queue`: Queue storage endpoint
- `table`: Table storage endpoint
- `dfs`: Data Lake Gen2 endpoint (if enabled)

**Required DNS Zones**:
- `privatelink.blob.core.windows.net`
- `privatelink.file.core.windows.net`
- `privatelink.queue.core.windows.net`
- `privatelink.table.core.windows.net`
- `privatelink.dfs.core.windows.net`

**Bicep Example** (Blob):
```bicep
resource storageBlobPrivateEndpoint 'Microsoft.Network/privateEndpoints@2023-11-01' = {
  name: '${storageName}-blob-pe'
  location: location
  properties: {
    subnet: {
      id: subnetId
    }
    privateLinkServiceConnections: [
      {
        name: '${storageName}-blob-connection'
        properties: {
          privateLinkServiceId: storageAccount.id
          groupIds: ['blob']
        }
      }
    ]
  }
}
```

**Important**: Create separate private endpoints for blob, file, queue, and table if all are used.

### Azure Key Vault

**Private Endpoint Type**: `Microsoft.KeyVault/vaults`

**Sub-Resources**:
- `vault`: Key Vault endpoint

**Required DNS Zones**:
- `privatelink.vaultcore.azure.net`

**Bicep Example**:
```bicep
resource keyVaultPrivateEndpoint 'Microsoft.Network/privateEndpoints@2023-11-01' = {
  name: '${keyVaultName}-pe'
  location: location
  properties: {
    subnet: {
      id: subnetId
    }
    privateLinkServiceConnections: [
      {
        name: '${keyVaultName}-pe-connection'
        properties: {
          privateLinkServiceId: keyVault.id
          groupIds: ['vault']
        }
      }
    ]
  }
}
```

### Azure Container Registry

**Private Endpoint Type**: `Microsoft.ContainerRegistry/registries`

**Sub-Resources**:
- `registry`: Container registry endpoint

**Required DNS Zones**:
- `privatelink.azurecr.io`

**Bicep Example**:
```bicep
resource acrPrivateEndpoint 'Microsoft.Network/privateEndpoints@2023-11-01' = {
  name: '${acrName}-pe'
  location: location
  properties: {
    subnet: {
      id: subnetId
    }
    privateLinkServiceConnections: [
      {
        name: '${acrName}-pe-connection'
        properties: {
          privateLinkServiceId: containerRegistry.id
          groupIds: ['registry']
        }
      }
    ]
  }
}
```

### Azure Cognitive Search (AI Search)

**Private Endpoint Type**: `Microsoft.Search/searchServices`

**Sub-Resources**:
- `searchService`: Search service endpoint

**Required DNS Zones**:
- `privatelink.search.windows.net`

**Bicep Example**:
```bicep
resource searchPrivateEndpoint 'Microsoft.Network/privateEndpoints@2023-11-01' = {
  name: '${searchName}-pe'
  location: location
  properties: {
    subnet: {
      id: subnetId
    }
    privateLinkServiceConnections: [
      {
        name: '${searchName}-pe-connection'
        properties: {
          privateLinkServiceId: searchService.id
          groupIds: ['searchService']
        }
      }
    ]
  }
}
```

### Azure Cosmos DB

**Private Endpoint Type**: `Microsoft.DocumentDB/databaseAccounts`

**Sub-Resources**:
- `Sql`: For SQL API
- `MongoDB`: For MongoDB API
- `Cassandra`: For Cassandra API
- `Gremlin`: For Gremlin API
- `Table`: For Table API

**Required DNS Zones**:
- `privatelink.documents.azure.com` (SQL)
- `privatelink.mongo.cosmos.azure.com` (MongoDB)
- `privatelink.cassandra.cosmos.azure.com` (Cassandra)
- `privatelink.gremlin.cosmos.azure.com` (Gremlin)
- `privatelink.table.cosmos.azure.com` (Table)

**Bicep Example** (SQL API):
```bicep
resource cosmosPrivateEndpoint 'Microsoft.Network/privateEndpoints@2023-11-01' = {
  name: '${cosmosName}-pe'
  location: location
  properties: {
    subnet: {
      id: subnetId
    }
    privateLinkServiceConnections: [
      {
        name: '${cosmosName}-pe-connection'
        properties: {
          privateLinkServiceId: cosmosAccount.id
          groupIds: ['Sql']
        }
      }
    ]
  }
}
```

### Azure SQL Database

**Private Endpoint Type**: `Microsoft.Sql/servers`

**Sub-Resources**:
- `sqlServer`: SQL Server endpoint

**Required DNS Zones**:
- `privatelink.database.windows.net`

**Bicep Example**:
```bicep
resource sqlPrivateEndpoint 'Microsoft.Network/privateEndpoints@2023-11-01' = {
  name: '${sqlServerName}-pe'
  location: location
  properties: {
    subnet: {
      id: subnetId
    }
    privateLinkServiceConnections: [
      {
        name: '${sqlServerName}-pe-connection'
        properties: {
          privateLinkServiceId: sqlServer.id
          groupIds: ['sqlServer']
        }
      }
    ]
  }
}
```

## DNS Configuration Patterns

### Private DNS Zone Deployment

**All Required Zones for AI Foundry Environment**:
```bicep
// AI Foundry & ML
resource mlApiDnsZone 'Microsoft.Network/privateDnsZones@2020-06-01' = {
  name: 'privatelink.api.azureml.ms'
  location: 'global'
}

resource mlNotebooksDnsZone 'Microsoft.Network/privateDnsZones@2020-06-01' = {
  name: 'privatelink.notebooks.azure.net'
  location: 'global'
}

// AI Services
resource cognitiveServicesDnsZone 'Microsoft.Network/privateDnsZones@2020-06-01' = {
  name: 'privatelink.cognitiveservices.azure.com'
  location: 'global'
}

resource openAIDnsZone 'Microsoft.Network/privateDnsZones@2020-06-01' = {
  name: 'privatelink.openai.azure.com'
  location: 'global'
}

// Storage
resource blobDnsZone 'Microsoft.Network/privateDnsZones@2020-06-01' = {
  name: 'privatelink.blob.core.windows.net'
  location: 'global'
}

resource fileDnsZone 'Microsoft.Network/privateDnsZones@2020-06-01' = {
  name: 'privatelink.file.core.windows.net'
  location: 'global'
}

// Key Vault
resource keyVaultDnsZone 'Microsoft.Network/privateDnsZones@2020-06-01' = {
  name: 'privatelink.vaultcore.azure.net'
  location: 'global'
}

// Container Registry
resource acrDnsZone 'Microsoft.Network/privateDnsZones@2020-06-01' = {
  name: 'privatelink.azurecr.io'
  location: 'global'
}

// AI Search
resource searchDnsZone 'Microsoft.Network/privateDnsZones@2020-06-01' = {
  name: 'privatelink.search.windows.net'
  location: 'global'
}

// Cosmos DB
resource cosmosDnsZone 'Microsoft.Network/privateDnsZones@2020-06-01' = {
  name: 'privatelink.documents.azure.com'
  location: 'global'
}

// SQL Database
resource sqlDnsZone 'Microsoft.Network/privateDnsZones@2020-06-01' = {
  name: 'privatelink.database.windows.net'
  location: 'global'
}
```

### VNet Links

**Link all DNS zones to VNet**:
```bicep
resource vnetLink 'Microsoft.Network/privateDnsZones/virtualNetworkLinks@2020-06-01' = {
  parent: mlApiDnsZone
  name: '${vnetName}-link'
  location: 'global'
  properties: {
    registrationEnabled: false
    virtualNetwork: {
      id: vnet.id
    }
  }
}

// Repeat for all DNS zones
```

**Important**: Every private DNS zone must be linked to the VNet for resolution to work.

## Network Security Configuration

### Subnet Requirements

**Minimum Subnet Configuration**:
```bicep
resource peSubnet 'Microsoft.Network/virtualNetworks/subnets@2023-11-01' = {
  parent: vnet
  name: 'private-endpoints-subnet'
  properties: {
    addressPrefix: '10.0.1.0/24'
    privateEndpointNetworkPolicies: 'Disabled'  // Required for PEs
    privateLinkServiceNetworkPolicies: 'Enabled'
    networkSecurityGroup: {
      id: peNsg.id
    }
  }
}
```

**Important**: Set `privateEndpointNetworkPolicies: 'Disabled'` on subnets with private endpoints.

### Network Security Group Rules

**Required NSG Rules for Private Endpoints**:
```bicep
resource peNsg 'Microsoft.Network/networkSecurityGroups@2023-11-01' = {
  name: 'pe-subnet-nsg'
  location: location
  properties: {
    securityRules: [
      {
        name: 'AllowVNetInbound'
        properties: {
          priority: 100
          direction: 'Inbound'
          access: 'Allow'
          protocol: '*'
          sourceAddressPrefix: 'VirtualNetwork'
          sourcePortRange: '*'
          destinationAddressPrefix: 'VirtualNetwork'
          destinationPortRange: '*'
        }
      }
      {
        name: 'DenyAllInbound'
        properties: {
          priority: 1000
          direction: 'Inbound'
          access: 'Deny'
          protocol: '*'
          sourceAddressPrefix: '*'
          sourcePortRange: '*'
          destinationAddressPrefix: '*'
          destinationPortRange: '*'
        }
      }
      {
        name: 'AllowAzureCloudOutbound'
        properties: {
          priority: 100
          direction: 'Outbound'
          access: 'Allow'
          protocol: 'Tcp'
          sourceAddressPrefix: 'VirtualNetwork'
          sourcePortRange: '*'
          destinationAddressPrefix: 'AzureCloud'
          destinationPortRange: '443'
        }
      }
      {
        name: 'AllowVNetOutbound'
        properties: {
          priority: 110
          direction: 'Outbound'
          access: 'Allow'
          protocol: '*'
          sourceAddressPrefix: 'VirtualNetwork'
          sourcePortRange: '*'
          destinationAddressPrefix: 'VirtualNetwork'
          destinationPortRange: '*'
        }
      }
    ]
  }
}
```

## Azure Verified Modules (AVM) Pattern

**Using AVM for Private Endpoints**:
```bicep
module hubWithPrivateEndpoint 'br/public:avm/res/machinelearningservices/workspace:0.9.0' = {
  name: 'hub-with-pe'
  params: {
    name: hubName
    kind: 'Hub'
    privateEndpoints: [
      {
        name: '${hubName}-pe'
        subnetResourceId: peSubnet.id
        privateDnsZoneGroup: {
          privateDnsZoneConfigs: [
            {
              name: 'amlworkspace'
              privateDnsZoneResourceId: mlApiDnsZone.id
            }
            {
              name: 'notebooks'
              privateDnsZoneResourceId: mlNotebooksDnsZone.id
            }
          ]
        }
      }
    ]
  }
}
```

**Benefits of AVM**:
- Consistent private endpoint configuration
- Automatic DNS zone group creation
- Built-in validation and best practices
- Maintained by Microsoft

## Validation & Troubleshooting

### DNS Resolution Testing

**From Within VNet**:
```bash
# Test DNS resolution
nslookup <hub-name>.api.azureml.ms

# Should resolve to private IP (10.x.x.x), not public IP
```

**From Azure CLI**:
```bash
# Check private endpoint connection state
az network private-endpoint show \
  --name <pe-name> \
  --resource-group <rg-name> \
  --query 'privateLinkServiceConnections[0].privateLinkServiceConnectionState'
```

### Connectivity Testing

**Test from VM in VNet**:
```bash
# Test HTTPS connectivity
curl -v https://<hub-name>.api.azureml.ms

# Should succeed with private IP, no public internet access
```

**Test DNS Zone Configuration**:
```bash
# List DNS records
az network private-dns record-set list \
  --resource-group <rg-name> \
  --zone-name privatelink.api.azureml.ms
```

### Common Issues & Resolutions

**Issue**: DNS resolves to public IP instead of private IP
- **Cause**: DNS zone not linked to VNet
- **Fix**: Link private DNS zone to VNet

**Issue**: Connection timeout to service
- **Cause**: NSG blocking traffic
- **Fix**: Add NSG rule allowing VNet-to-VNet traffic

**Issue**: Private endpoint in "Failed" state
- **Cause**: Subnet has network policies enabled
- **Fix**: Set `privateEndpointNetworkPolicies: 'Disabled'`

**Issue**: "Approval pending" on private endpoint
- **Cause**: Automatic approval not configured
- **Fix**: Approve connection manually or use `privateLinkServiceConnections` (not `manualPrivateLinkServiceConnections`)

## Security Best Practices

### Private Endpoint Security

1. **Disable Public Access**
   ```bicep
   properties: {
     publicNetworkAccess: 'Disabled'
   }
   ```

2. **Use Managed Identity**
   - Avoid access keys/connection strings
   - Leverage Azure RBAC with managed identity
   - Private endpoints work seamlessly with managed identity

3. **Subnet Segregation**
   - Dedicated subnet for private endpoints
   - Separate NSG per subnet
   - Apply principle of least privilege

4. **DNS Security**
   - Lock down DNS zones with RBAC
   - Audit DNS record changes
   - Monitor for unauthorized DNS modifications

### Compliance Considerations

**Data Residency**:
- Private endpoints keep data in region
- No public internet traversal
- Supports regulatory compliance (HIPAA, GDPR, etc.)

**Network Isolation**:
- Complete network isolation from internet
- Service-to-service communication stays private
- Supports Zero Trust architecture

## Cost Optimization

**Private Endpoint Costs**:
- $0.01/hour per private endpoint (~$7.30/month)
- Minimal data processing charges
- DNS zone hosting: $0.50/zone/month

**Optimization Strategies**:
1. **Consolidate where possible**: Use multi-service PEs (Storage: blob+file in one subnet)
2. **Remove unused PEs**: Delete PEs for services not in active use
3. **Shared DNS zones**: Reuse DNS zones across environments
4. **Right-size subnets**: Don't over-allocate IP space

**For Ephemeral Environments**:
- Deploy all PEs in single resource group for easy deletion
- Tag with expiration date
- Automate cleanup of unused private endpoints

## Reference Architecture

**Complete AI Foundry Private Endpoint Pattern**:
```
Resource Group: rg-aiFoundry-dev
├── VNet (10.0.0.0/16)
│   ├── PE Subnet (10.0.1.0/24)
│   │   ├── Hub PE (10.0.1.4)
│   │   ├── Storage Blob PE (10.0.1.5)
│   │   ├── Storage File PE (10.0.1.6)
│   │   ├── Key Vault PE (10.0.1.7)
│   │   ├── Container Registry PE (10.0.1.8)
│   │   ├── AI Services PE (10.0.1.9)
│   │   ├── AI Search PE (10.0.1.10)
│   │   └── Cosmos DB PE (10.0.1.11)
│   └── Compute Subnet (10.0.2.0/24)
├── Private DNS Zones (11 zones)
│   └── VNet Links (11 links)
├── NSGs (2 NSGs)
└── Azure Resources
    ├── AI Foundry Hub
    ├── Storage Account
    ├── Key Vault
    ├── Container Registry
    ├── AI Services
    ├── AI Search
    └── Cosmos DB
```

## Deployment Order

**Critical Sequence**:
1. VNet and Subnets
2. Private DNS Zones
3. VNet Links to DNS Zones
4. NSGs
5. Azure Resources (Storage, Key Vault, etc.)
6. Private Endpoints
7. DNS Zone Groups
8. Validation

**Important**: DNS zones must exist and be linked before creating private endpoints.

## References

- [Azure Private Link Documentation](https://learn.microsoft.com/azure/private-link/)
- [Private Endpoints for AI Services](https://learn.microsoft.com/azure/ai-services/cognitive-services-virtual-networks)
- [Private DNS Zones](https://learn.microsoft.com/azure/dns/private-dns-overview)
- [Network Security Best Practices](https://learn.microsoft.com/azure/security/fundamentals/network-best-practices)
- [Azure Verified Modules](https://azure.github.io/Azure-Verified-Modules/)
