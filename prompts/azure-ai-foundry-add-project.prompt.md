---
mode: 'agent'
description: 'Add a new Project to an existing Azure AI Foundry Hub with proper RBAC and validation'
---

# Add Project to Azure AI Foundry Hub

Add a new isolated Project to an existing Azure AI Foundry Hub. Projects provide security boundaries for multi-customer scenarios.

## What This Does

- Creates a new Project under existing Hub
- Configures Project managed identity
- Assigns RBAC permissions (Storage, Key Vault access)
- Validates Project health and connectivity
- For Cognitive Services model: Creates child `/projects` resource
- For ML Workspaces model: Creates separate workspace with hubResourceId

## Prerequisites

- Existing Azure AI Foundry Hub deployed
- Hub resource group name and Hub name
- Permissions: Contributor on Hub and resource group

## Parameters

**Required**:
- `hubResourceGroup`: Resource group containing the Hub
- `hubName`: Name of the existing Hub
- `projectName`: Name for the new Project

**Optional**:
- `projectDescription`: Description for the Project
- `assignUsersToProject`: User/group IDs to grant access (comma-separated)

## Workflow Steps

### 1. Discover Hub Configuration

```bash
# Get Hub details
az cognitiveservices account show \
  --resource-group <hub-rg> \
  --name <hub-name> \
  --query "{Kind:kind, Location:location, PublicAccess:properties.publicNetworkAccess}"

# Or for ML Workspaces model:
az ml workspace show \
  --resource-group <hub-rg> \
  --name <hub-name> \
  --query "{Kind:kind, Location:location, HubId:id}"

# Identify deployment model (Cognitive Services or ML Workspaces)
```

### 2. Create Project (Cognitive Services Model)

```bicep
// project.bicep
param hubName string
param projectName string
param projectDescription string = 'AI Foundry Project'

resource hub 'Microsoft.CognitiveServices/accounts@2024-04-01-preview' existing = {
  name: hubName
}

resource project 'Microsoft.CognitiveServices/accounts/projects@2024-04-01-preview' = {
  parent: hub
  name: projectName
  location: hub.location
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    displayName: projectName
    description: projectDescription
  }
}

output projectId string = project.id
output projectPrincipalId string = project.identity.principalId
```

**Deploy**:
```bash
az deployment group create \
  --resource-group <hub-rg> \
  --template-file project.bicep \
  --parameters hubName=<hub-name> projectName=<project-name>
```

### 3. Create Project (ML Workspaces Model)

```bicep
// project-ml.bicep
param hubId string
param projectName string
param location string

resource project 'Microsoft.MachineLearningServices/workspaces@2023-08-01-preview' = {
  name: projectName
  location: location
  kind: 'project'
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    friendlyName: projectName
    hubResourceId: hubId
    publicNetworkAccess: 'Disabled'
  }
}

output projectId string = project.id
output projectPrincipalId string = project.identity.principalId
```

**Deploy**:
```bash
az deployment group create \
  --resource-group <hub-rg> \
  --template-file project-ml.bicep \
  --parameters hubId=<hub-id> projectName=<project-name> location=<location>
```

### 4. Configure RBAC for Project

**Get Resource IDs**:
```bash
# Get Project MI principal ID (from deployment output)
PROJECT_MI=$(az deployment group show -g <rg> -n <deployment-name> --query properties.outputs.projectPrincipalId.value -o tsv)

# Get Storage Account ID (from Hub deployment)
STORAGE_ID=$(az storage account list -g <rg> --query "[0].id" -o tsv)

# Get Key Vault ID (from Hub deployment)
KV_ID=$(az keyvault list -g <rg> --query "[0].id" -o tsv)
```

**Assign Service-to-Service Permissions**:
```bash
# Project MI → Storage Blob Data Contributor
az role assignment create \
  --assignee $PROJECT_MI \
  --role "Storage Blob Data Contributor" \
  --scope $STORAGE_ID

# Project MI → Key Vault Secrets User
az role assignment create \
  --assignee $PROJECT_MI \
  --role "Key Vault Secrets User" \
  --scope $KV_ID
```

**Assign User Permissions** (if users provided):
```bash
# Get Project ID
PROJECT_ID=$(az deployment group show -g <rg> -n <deployment-name> --query properties.outputs.projectId.value -o tsv)

# Assign user to Project
az role assignment create \
  --assignee <user-or-group-id> \
  --role "Azure AI Developer" \
  --scope $PROJECT_ID
```

### 5. Validate Project

**Check Project Status**:
```bash
# For Cognitive Services model
az cognitiveservices account show \
  --resource-group <rg> \
  --name <hub-name> \
  --query "properties.allowProjectManagement"

# For ML Workspaces model
az ml workspace show \
  --resource-group <rg> \
  --name <project-name> \
  --query "{Status:provisioningState, Hub:properties.hubResourceId}"
```

**Verify RBAC Assignments**:
```bash
# List Project's role assignments to Storage
az role assignment list \
  --assignee $PROJECT_MI \
  --scope $STORAGE_ID \
  --query "[].{Role:roleDefinitionName, Scope:scope}"
```

**Test Connectivity** (if in VNet):
```bash
# Test Project can access Hub
# For Cognitive Services: Projects inherit Hub endpoint
# For ML Workspaces: Test workspace endpoint
curl -I https://<project-endpoint>

# Expected: 401 Unauthorized (reachable, requires auth)
```

### 6. Verification Report

```markdown
✅ Project Added Successfully

**Project Details**:
- Name: <project-name>
- Hub: <hub-name>
- Model: Cognitive Services / ML Workspaces
- Managed Identity: Enabled
- Location: <location>

**RBAC Configured**:
- ✅ Project MI → Storage: Storage Blob Data Contributor
- ✅ Project MI → Key Vault: Key Vault Secrets User
- ✅ Users → Project: Azure AI Developer (if assigned)

**Validation**:
- ✅ Project provisioned successfully
- ✅ Managed identity created
- ✅ RBAC assignments active
- ✅ Connectivity verified

**Access**:
- Portal: https://ai.azure.com
- Project: <project-name>
- Hub: <hub-name>

**Next Steps**:
1. Access project in AI Foundry portal
2. Deploy models specific to this project
3. Create prompt flows or agents
4. Invite additional users if needed
```

## Security Boundaries

**For Multi-Customer Scenarios**:

With Cognitive Services model:
- Each `/projects` resource is an isolated child under the Hub
- Projects share Hub's private endpoint (same network boundary)
- RBAC provides logical isolation at project level
- Each project has independent managed identity
- Users can be assigned to specific projects only

**Security Considerations**:
- Projects under same Hub share networking (private endpoint)
- For stricter isolation, use separate Hubs
- RBAC boundaries prevent cross-project access
- Data in Storage can be organized per-project with RBAC

## Cleanup (Remove Project)

```bash
# For Cognitive Services model (delete child resource)
az resource delete \
  --resource-group <rg> \
  --name <hub-name>/projects/<project-name> \
  --resource-type Microsoft.CognitiveServices/accounts/projects

# For ML Workspaces model (delete workspace)
az ml workspace delete \
  --resource-group <rg> \
  --name <project-name>
```

## Common Issues

**Issue**: "Project creation failed - parent Hub not found"
- **Fix**: Verify Hub name and resource group
- **Check**: `az cognitiveservices account show -g <rg> -n <hub-name>`

**Issue**: "RBAC assignments not working"
- **Fix**: Wait 60 seconds for identity propagation
- **Retry**: Re-run RBAC assignment commands

**Issue**: "Cannot access project from portal"
- **Fix**: Ensure user has "Azure AI Developer" role on Project
- **Check**: `az role assignment list --scope <project-id>`

## Notes

- Projects are **lightweight** - create as many as needed for isolation
- Each project gets its own managed identity (independent permissions)
- For Cognitive Services model, projects inherit Hub's private endpoint
- For ML model, each project can have separate private endpoint (better isolation)
- Use projects to separate: customers, environments, teams, or workloads
