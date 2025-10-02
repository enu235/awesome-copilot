---
description: 'Azure AI Foundry hub and project deployment patterns, architecture best practices, and infrastructure requirements'
applyTo: '**/*.bicep, **/*.tf, **/.aif-planning-files/**'
---

# Azure AI Foundry Infrastructure Instructions

## Overview

Azure AI Foundry provides a comprehensive platform for building, deploying, and managing AI applications. Understanding the Hub-Project architecture and infrastructure requirements is critical for successful deployments.

## ⚠️ IMPORTANT: Two Deployment Approaches

Azure AI Foundry can be deployed using **TWO different resource provider models**. Choose based on your requirements:

### Approach 1: Cognitive Services Model (Recommended - Newer/Simpler)

**Use When:**
- Starting new AI Foundry deployments
- Need simplified resource model
- Want native AI Services integration
- Deploying in regions with latest features

**Resource Types:**
```bicep
// Hub: Cognitive Services Account with AIServices kind
resource aiHub 'Microsoft.CognitiveServices/accounts@2024-04-01-preview' = {
  kind: 'AIServices'
  // ... properties
}

// Project: Child resource under Cognitive Services Account
resource aiProject 'Microsoft.CognitiveServices/accounts/projects@2024-04-01-preview' = {
  parent: aiHub
  // ... properties
}
```

**Key Characteristics:**
- Hub is `Microsoft.CognitiveServices/accounts` with `kind: 'AIServices'`
- Projects are child resources: `Microsoft.CognitiveServices/accounts/projects`
- Simpler model, fewer dependencies
- Native AI Services integration
- Managed by Cognitive Services RP

### Approach 2: Machine Learning Workspaces Model (Established/Full-Featured)

**Use When:**
- Migrating from Azure Machine Learning
- Need full ML workspace features
- Require extensive compute options
- Using existing ML infrastructure

**Resource Types:**
```bicep
// Hub: ML Workspace with hub kind
resource aiHub 'Microsoft.MachineLearningServices/workspaces@2023-08-01-preview' = {
  kind: 'hub'
  // ... properties
}

// Project: ML Workspace with project kind (deployed separately)
resource aiProject 'Microsoft.MachineLearningServices/workspaces@2023-08-01-preview' = {
  kind: 'project'
  properties: {
    hubResourceId: aiHub.id
  }
  // ... properties
}
```

**Key Characteristics:**
- Hub is `Microsoft.MachineLearningServices/workspaces` with `kind: 'hub'`
- Project is separate workspace with `kind: 'project'`
- Full ML workspace capabilities
- Azure ML compute integration
- Managed by ML Services RP

### Decision Matrix

| Feature | Cognitive Services Model | ML Workspaces Model |
|---------|-------------------------|---------------------|
| Simplicity | ✅ Simpler | ⚠️ More complex |
| AI Services Integration | ✅ Native | ⚠️ Via connections |
| ML Compute Options | ⚠️ Limited | ✅ Full featured |
| Private Endpoints | ✅ Supported | ✅ Supported |
| **Multi-Project Isolation** | ⚠️ **Projects inherit Hub** | ✅ **Separate workspaces** |
| **Security Boundaries** | ⚠️ **Shared Hub PE** | ✅ **Independent PEs** |
| API Version | 2024-04-01-preview+ | 2023-08-01-preview+ |
| Maturity | 🆕 Newer | ✅ Established |

### Recommendation

**For Multi-Project Customer Scenarios**: Use **ML Workspaces Model (Approach 2)**
- ✅ Each project has independent security boundary
- ✅ Separate private endpoints per project (better isolation)
- ✅ Individual RBAC and access controls per project
- ✅ Customer environments often need strict project separation

**For Single Project/Simple Scenarios**: Use **Cognitive Services Model (Approach 1)**
- ✅ Simpler deployment, fewer resources
- ✅ Projects inherit Hub security (less management)
- ✅ AI-first development experience

**This Collection**: Supports **BOTH approaches** with emphasis on ML Workspaces for customer environment reproduction.

## Hub-Project Architecture

### Hub (Central Infrastructure)
A **Hub** is the top-level resource that provides shared infrastructure and governance:

**Core Responsibilities:**
- Shared security configuration (VNet, managed identity, private endpoints)
- Centralized Azure AI Services connection
- Common storage, key vault, and optional container registry
- Compute and quota management across all projects
- Policy enforcement and RBAC inheritance

**Key Characteristics:**
- One hub supports multiple projects
- Managed VNet isolation (cannot be disabled once enabled)
- All projects inherit hub's security posture
- Provisions: Storage Account (required), Azure AI Services (required), Key Vault (optional), Container Registry (optional), Application Insights (optional)

### Project (Development Workspace)
A **Project** is an isolated workspace within a hub for development activities:

**Core Responsibilities:**
- Isolated development container for specific use cases
- Agents, flows, models, and evaluations
- Project-specific file storage, thread storage, and search indexes
- Granular RBAC at project level

**Key Characteristics:**
- Inherits resources and configurations from parent hub
- Multiple projects can share hub infrastructure
- Project-level isolation for security and collaboration
- Independent lifecycle from other projects in same hub

## Deployment Modes

### Managed VNet (Recommended for Secure Deployments)

Azure AI Foundry manages the virtual network with automatic private endpoint creation:

**Isolation Modes:**
- `Allow Internet Outbound`: Auto-creates private endpoints for hub resources; allows internet access
- `Allow Only Approved Outbound`: Creates private endpoints for all resources; blocks internet unless explicitly approved

**Automatic Private Endpoints Created For:**
- Azure Storage (blob, file, queue, table)
- Azure Key Vault
- Azure Container Registry
- Azure AI Services
- Azure Machine Learning workspace

**Important Notes:**
- Once managed VNet is enabled, it cannot be disabled
- Private endpoints created before July 11, 2024 require recreation for serverless API support
- Managed VNet is the only supported option for Hubs

### BYO VNet (Limited Support)

Bring Your Own VNet is currently only supported for:
- **Azure AI Foundry Agent Service** (via Bicep deployment only)
- Projects that need to operate within existing organizational VNets

**Not supported for standard Hub deployments**

## Required Azure Resources

### Mandatory Resources

1. **Azure AI Services** (required)
   - Provides base models and endpoints
   - Configured at Hub level
   - Shared across all projects in hub

2. **Azure Storage Account** (required)
   - Stores data, artifacts, and outputs
   - Auto-provisioned during hub creation
   - Supports private endpoints in managed VNet

### Optional Resources

3. **Azure Key Vault** (recommended)
   - Stores API keys, secrets, and encryption keys
   - Required for customer-managed keys
   - Auto-configured with private endpoint in managed VNet

4. **Azure Container Registry** (required for custom environments)
   - Stores custom container images
   - Required for Prompt Flow custom environments
   - Supports private endpoints

5. **Application Insights** (required for Prompt Flow)
   - Monitoring and telemetry
   - Required for Prompt Flow deployments
   - Diagnostics and performance tracking

## Supporting Service Dependencies

### Common Integration Services

1. **Azure Cognitive Search / Azure AI Search**
   - Vector search capabilities
   - RAG pattern implementation
   - Requires private endpoint for secure access

2. **Azure Cosmos DB**
   - Document storage for RAG
   - Vector search with MongoDB vCore
   - Private endpoint recommended

3. **Azure OpenAI Service**
   - LLM deployments
   - Can be connected to Hub via AI Services or separately
   - Supports private endpoints

4. **Azure SQL Database / PostgreSQL**
   - Structured data storage
   - Application databases
   - Private endpoint support

## Networking Patterns

### Private Endpoint Configuration

**Single VNet Pattern (Recommended for Single Subscription):**
```
VNet (10.0.0.0/16)
├── AI Foundry Subnet (10.0.1.0/24)
│   ├── Private Endpoint: Hub
│   ├── Private Endpoint: Storage
│   ├── Private Endpoint: Key Vault
│   └── Private Endpoint: Container Registry
├── Data Services Subnet (10.0.2.0/24)
│   ├── Private Endpoint: AI Search
│   ├── Private Endpoint: Cosmos DB
│   └── Private Endpoint: SQL Database
└── Compute Subnet (10.0.3.0/24)
    └── (Reserved for compute resources if needed)
```

**Private Endpoint Requirements:**
- Each Azure service requires specific subnet delegation
- DNS resolution via Azure Private DNS Zones
- Network Security Group (NSG) rules must allow PE traffic
- Service-specific private endpoint types:
  - Storage: blob, file, queue, table (separate PEs)
  - Cosmos DB: SQL or MongoDB
  - AI Search: searchService
  - AI Services: account

### DNS Configuration

**Required Private DNS Zones:**
- `privatelink.blob.core.windows.net` (Storage blob)
- `privatelink.file.core.windows.net` (Storage file)
- `privatelink.vaultcore.azure.net` (Key Vault)
- `privatelink.azurecr.io` (Container Registry)
- `privatelink.api.azureml.ms` (ML workspace)
- `privatelink.notebooks.azure.net` (ML notebooks)
- `privatelink.search.windows.net` (AI Search)
- `privatelink.documents.azure.com` (Cosmos DB)
- `privatelink.cognitiveservices.azure.com` (AI Services)
- `privatelink.openai.azure.com` (OpenAI)

**DNS Configuration:**
- Link all DNS zones to VNet
- Auto-registration enabled for private endpoints
- Conditional forwarding for hybrid scenarios

## Security Best Practices

### Identity & Access

1. **Managed Identity (Recommended)**
   - System-assigned for Hub and Projects
   - RBAC assignments to dependent resources
   - Avoid access keys where possible

2. **RBAC Roles for AI Foundry**
   - `Azure AI Developer`: Full project access
   - `Azure AI Inference Deployment Operator`: Deploy models
   - `Cognitive Services OpenAI User`: Use OpenAI resources
   - `Storage Blob Data Contributor`: Access storage
   - `Key Vault Secrets User`: Read secrets

### Network Security

1. **Disable Public Access**
   - Set `publicNetworkAccess: 'Disabled'` on all resources
   - Enforce private endpoints only
   - Configure firewall rules as backup

2. **NSG Rules**
   - Allow intra-subnet traffic
   - Allow traffic to private endpoints
   - Deny all inbound from internet
   - Allow specific outbound (Azure services)

### Data Protection

1. **Encryption**
   - Data at rest: Azure-managed keys or customer-managed keys (CMK)
   - Data in transit: TLS 1.2+ enforced
   - Credential encryption via Key Vault

2. **Data Classification**
   - PII/PHI in Cosmos DB or SQL with encryption
   - Training data in Storage with access policies
   - Model artifacts protected with RBAC

## Cost Optimization for Ephemeral Environments

Since these environments are short-lived and reproducible:

1. **Use Dev/Basic SKUs**
   - Azure AI Services: S0 (standard, pay-per-use)
   - Storage: Standard LRS (locally redundant)
   - Key Vault: Standard (not Premium)
   - Container Registry: Basic (if needed)

2. **Minimize Data Retention**
   - Short retention policies (7-30 days)
   - Delete unused deployments promptly
   - Use lifecycle management for storage

3. **Compute Optimization**
   - Serverless compute when possible
   - Auto-shutdown for dev instances
   - Spot instances for training (if applicable)

4. **Resource Group Lifecycle**
   - Deploy entire environment in single resource group
   - Tag with expiration date
   - Enable automated deletion/cleanup

## Bicep Deployment Patterns

### Resource Naming Convention

```bicep
// Naming pattern: {workload}-{environment}-{region}-{resource}
var hubName = '${workloadName}-aih-${environment}-${location}'
var projectName = '${workloadName}-aip-${environment}-${location}'
var storageName = toLower(replace('${workloadName}st${environment}', '-', ''))
var keyVaultName = '${workloadName}-kv-${environment}'
```

### Approach 1: Cognitive Services Model (Recommended)

**Hub Deployment (AIServices)**:
```bicep
param location string = resourceGroup().location
param hubName string

resource aiHub 'Microsoft.CognitiveServices/accounts@2024-04-01-preview' = {
  name: hubName
  location: location
  kind: 'AIServices'
  sku: {
    name: 'S0'
  }
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    publicNetworkAccess: 'Disabled'
    allowProjectManagement: true
    networkAcls: {
      defaultAction: 'Deny'
    }
  }
}
```

**Project Deployment (Child Resource)**:
```bicep
resource aiProject 'Microsoft.CognitiveServices/accounts/projects@2024-04-01-preview' = {
  parent: aiHub
  name: projectName
  location: location
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    displayName: 'AI Foundry Project'
    description: 'Project for AI development'
  }
}
```

**Complete Example with Dependencies**:
```bicep
// Storage Account (required)
resource storage 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: storageName
  location: location
  kind: 'StorageV2'
  sku: { name: 'Standard_LRS' }
  properties: {
    publicNetworkAccess: 'Disabled'
  }
}

// Key Vault (optional but recommended)
resource keyVault 'Microsoft.KeyVault/vaults@2023-07-01' = {
  name: keyVaultName
  location: location
  properties: {
    sku: { family: 'A', name: 'standard' }
    tenantId: tenant().tenantId
    publicNetworkAccess: 'Disabled'
  }
}

// AI Hub with dependencies
resource aiHub 'Microsoft.CognitiveServices/accounts@2024-04-01-preview' = {
  name: hubName
  location: location
  kind: 'AIServices'
  sku: { name: 'S0' }
  identity: { type: 'SystemAssigned' }
  properties: {
    publicNetworkAccess: 'Disabled'
    allowProjectManagement: true
  }
}

// Project under Hub
resource aiProject 'Microsoft.CognitiveServices/accounts/projects@2024-04-01-preview' = {
  parent: aiHub
  name: projectName
  location: location
  identity: { type: 'SystemAssigned' }
  properties: {
    displayName: projectName
  }
}
```

### Approach 2: ML Workspaces Model

**Hub Deployment (ML Workspace)**:
```bicep
resource hub 'Microsoft.MachineLearningServices/workspaces@2023-08-01-preview' = {
  name: hubName
  location: location
  kind: 'hub'
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    friendlyName: hubName
    storageAccount: storageAccount.id
    keyVault: keyVault.id
    applicationInsights: appInsights.id // optional
    containerRegistry: acr.id // optional
    publicNetworkAccess: 'Disabled'
    managedNetwork: {
      isolationMode: 'AllowOnlyApprovedOutbound'
    }
  }
}
```

**Project Deployment (Separate Workspace)**:
```bicep
resource project 'Microsoft.MachineLearningServices/workspaces@2023-08-01-preview' = {
  name: projectName
  location: location
  kind: 'project'
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    friendlyName: projectName
    hubResourceId: hub.id
    publicNetworkAccess: 'Disabled'
  }
}
```

### Azure Verified Modules (AVM) Pattern

**For ML Workspaces Approach**:
```bicep
module hub 'br/public:avm/res/machinelearningservices/workspace:0.9.0' = {
  name: 'hub-deployment'
  params: {
    name: hubName
    kind: 'Hub'
    managedNetworkSettings: {
      isolationMode: 'AllowOnlyApprovedOutbound'
    }
    privateEndpoints: [
      {
        subnetResourceId: peSubnet.id
        privateDnsZoneGroup: {
          privateDnsZoneConfigs: [
            { privateDnsZoneResourceId: mlDnsZone.id }
          ]
        }
      }
    ]
  }
}
```

**Note**: AVM modules for Cognitive Services AIServices are still in development. Use raw resources for Approach 1.

## Validation & Testing

### Pre-Deployment Validation

1. **Check Subnet Availability**
   - Sufficient IP addresses for private endpoints
   - No conflicts with existing resources
   - NSG rules compatible

2. **RBAC Prerequisites**
   - Deploying principal has required permissions
   - Service principals configured
   - Managed identities ready

3. **Quota Verification**
   - Azure AI Services quota available
   - Storage account limits not exceeded
   - Regional capacity available

### Post-Deployment Validation

1. **Connectivity Tests**
   - Private endpoint DNS resolution
   - Hub accessible from VNet
   - Project creation successful

2. **RBAC Validation**
   - Managed identity permissions working
   - User access via private endpoint
   - Service-to-service auth functional

3. **Operational Checks**
   - Model deployment successful
   - Storage access verified
   - Logging/monitoring active

## Common Patterns & Anti-Patterns

### ✅ Recommended Patterns

- Use managed VNet for simplified networking
- Deploy Hub and all dependencies in same resource group
- Use Azure Verified Modules for consistent infrastructure
- Enable managed identity for all resources
- Tag resources with environment and expiration
- Use Azure MCP tools for deployment validation

### ❌ Anti-Patterns to Avoid

- Mixing public and private access modes
- Hardcoding secrets in Bicep templates
- Ignoring DNS zone linking
- Over-provisioning for ephemeral environments
- Creating private endpoints without DNS configuration
- Deploying without RBAC validation

## Tool Integration

### Azure MCP Tools

Use Azure MCP tools for:
- `azmcp-subscription-list`: Find target subscription
- `azmcp-group-list`: Verify resource group
- `azmcp-monitor-log-query`: Check deployment logs
- `azmcp-bestpractices-get`: Retrieve latest guidance

### Azure CLI Common Commands

```bash
# Verify private endpoint connectivity
az network private-endpoint show \
  --name <pe-name> \
  --resource-group <rg-name>

# Check DNS resolution
nslookup <hub-name>.api.azureml.ms

# Validate RBAC assignments
az role assignment list \
  --scope <resource-id> \
  --output table

# Check managed network status
az ml workspace show \
  --name <hub-name> \
  --resource-group <rg-name> \
  --query managedNetwork
```

## References

- [Azure AI Foundry Documentation](https://learn.microsoft.com/azure/ai-foundry/)
- [Hub and Project Architecture](https://learn.microsoft.com/azure/ai-foundry/concepts/ai-resources)
- [Managed VNet Configuration](https://learn.microsoft.com/azure/ai-foundry/how-to/configure-managed-network)
- [Private Link Configuration](https://learn.microsoft.com/azure/ai-foundry/how-to/configure-private-link)
- [Azure Verified Modules](https://azure.github.io/Azure-Verified-Modules/)
- [OWASP Security Guidelines](https://owasp.org/www-project-top-ten/)
