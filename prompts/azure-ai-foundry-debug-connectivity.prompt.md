---
mode: 'agent'
description: 'Diagnose and fix Azure AI Foundry private endpoint connectivity issues'
---

# Debug Azure AI Foundry Connectivity Issues

Systematic troubleshooting for Azure AI Foundry Hub/Project connectivity problems when using private endpoints.

## What This Does

Diagnoses common connectivity issues:
- Private endpoint not connecting
- DNS resolution failing (resolves to public IP instead of private)
- Hub/Project inaccessible from VNet
- RBAC permission errors
- Network ACL blocks
- NSG rule problems

Provides specific fixes for each issue discovered.

## Prerequisites

- Deployed Azure AI Foundry infrastructure with private endpoints
- Azure CLI access
- Resource group name and Hub/Project names

## Parameters

**Required**:
- `resourceGroup`: Resource group name
- `hubName`: Hub name
- `symptom`: Issue description (e.g., "cannot access hub", "DNS resolves to public IP")

**Optional**:
- `projectName`: Project name (if issue is project-specific)
- `sourceLocation`: Where connectivity is tested from (VNet name, VM name, or "CloudShell")

## Diagnostic Workflow

### Phase 1: Resource Status Check

**Verify all resources are healthy**:

```bash
# Check Hub status
az cognitiveservices account show \
  --resource-group <rg> \
  --name <hub-name> \
  --query "{Name:name, Status:properties.provisioningState, PublicAccess:properties.publicNetworkAccess}"

# Expected: Status=Succeeded, PublicAccess=Disabled

# Check Storage status
az storage account list \
  --resource-group <rg> \
  --query "[].{Name:name, Status:provisioningState, PublicAccess:publicNetworkAccess}"

# Check Key Vault status
az keyvault list \
  --resource-group <rg> \
  --query "[].{Name:name, PublicAccess:properties.publicNetworkAccess}"
```

**✅ IF ALL SUCCEEDED**: Proceed to Phase 2
**❌ IF ANY FAILED**:
```
Issue: Resource provisioning failed

Fix: Re-deploy failed resource
- Review error in Azure Portal Activity Log
- Check deployment logs for specific error
- Verify quotas and permissions

Command: az deployment group create [re-deploy specific resource]
```

### Phase 2: Private Endpoint Status

**Check private endpoint connection state**:

```bash
# List all private endpoints
az network private-endpoint list \
  --resource-group <rg> \
  --query "[].{Name:name, ConnectionState:privateLinkServiceConnections[0].privateLinkServiceConnectionState.status, ConnectionStatus:privateLinkServiceConnections[0].privateLinkServiceConnectionState.actionsRequired}" \
  --output table

# Expected: All show "Approved" with no actions required
```

**Common Issues**:

**Issue 1: Private Endpoint in "Pending" state**
```
Diagnosis: Manual approval required (using manualPrivateLinkServiceConnections)

Fix: Approve the private endpoint connection
az network private-endpoint-connection approve \
  --resource-group <rg> \
  --resource-name <service-name> \
  --name <pe-connection-name> \
  --type Microsoft.CognitiveServices/accounts  # or appropriate type

OR: Update Bicep to use privateLinkServiceConnections (auto-approve)
```

**Issue 2: Private Endpoint shows "Rejected"**
```
Diagnosis: Connection was rejected (permissions or configuration issue)

Fix:
1. Delete and recreate private endpoint
2. Ensure deploying principal has permissions
3. Verify service allows private endpoints

az network private-endpoint delete -g <rg> -n <pe-name>
# Re-deploy via Bicep
```

**Issue 3: Private Endpoint missing**
```
Diagnosis: Private endpoint was not created

Fix: Deploy private endpoint
[Use appropriate Bicep template or az network private-endpoint create]
```

### Phase 3: DNS Resolution Check

**Verify DNS configuration**:

```bash
# List private DNS zones
az network private-dns zone list \
  --resource-group <rg> \
  --query "[].{Name:name, RecordCount:numberOfRecordSets}"

# Expected zones for Cognitive Services Hub:
# - privatelink.cognitiveservices.azure.com
# - privatelink.openai.azure.com
# - privatelink.blob.core.windows.net
# - privatelink.file.core.windows.net
# - privatelink.vaultcore.azure.net
```

**Check DNS records**:

```bash
# Check Hub DNS record
az network private-dns record-set a list \
  --resource-group <rg> \
  --zone-name privatelink.cognitiveservices.azure.com \
  --query "[?name=='<hub-name>'].{Name:name, IP:aRecords[0].ipv4Address}"

# Expected: Returns private IP (10.0.x.x)
```

**Check VNet links**:

```bash
# Verify DNS zones are linked to VNet
az network private-dns link vnet list \
  --resource-group <rg> \
  --zone-name privatelink.cognitiveservices.azure.com \
  --query "[].{VNet:virtualNetwork.id, Registered:registrationEnabled}"

# Expected: VNet linked, Registered=false
```

**Common Issues**:

**Issue 1: DNS zone missing**
```
Diagnosis: Private DNS zone not created

Fix: Create missing DNS zone and link to VNet
az network private-dns zone create \
  --resource-group <rg> \
  --name privatelink.cognitiveservices.azure.com

az network private-dns link vnet create \
  --resource-group <rg> \
  --zone-name privatelink.cognitiveservices.azure.com \
  --name <vnet-name>-link \
  --virtual-network <vnet-id> \
  --registration-enabled false
```

**Issue 2: DNS zone not linked to VNet**
```
Diagnosis: VNet link missing

Fix: Link DNS zone to VNet (command above)
```

**Issue 3: DNS record missing**
```
Diagnosis: Private endpoint DNS zone group not configured

Fix: Create DNS zone group for private endpoint
az network private-endpoint dns-zone-group create \
  --resource-group <rg> \
  --endpoint-name <pe-name> \
  --name default \
  --private-dns-zone <dns-zone-id> \
  --zone-name cognitiveservices
```

### Phase 4: DNS Resolution Testing

**Test actual DNS resolution** (requires VM in VNet or Cloud Shell with VNet integration):

```bash
# Test Hub endpoint resolution
nslookup <hub-name>.cognitiveservices.azure.com

# Expected output:
# Name: <hub-name>.cognitiveservices.azure.com
# Address: 10.0.x.x  (PRIVATE IP)

# NOT: Public IP (13.x.x.x, 20.x.x.x, etc.)
```

**Issue: Resolves to public IP**
```
Diagnosis: DNS not using private DNS zones

Troubleshooting:
1. Confirm testing from within VNet (not external)
2. Verify VNet uses Azure-provided DNS (168.63.129.16)
3. Check DNS server settings on VNet
4. Verify DNS zone VNet link exists
5. Wait 5-10 minutes for DNS propagation

Fix VNet DNS:
az network vnet update \
  --resource-group <rg> \
  --name <vnet-name> \
  --dns-servers "" # Use Azure DNS

# Restart VMs/resources to pick up new DNS
```

### Phase 5: Network ACL and Firewall Check

**Check if network ACLs are blocking access**:

```bash
# Check Hub network ACLs
az cognitiveservices account show \
  --resource-group <rg> \
  --name <hub-name> \
  --query "properties.networkAcls"

# Expected: defaultAction = "Deny" (private endpoint only)
# Check if virtualNetworkRules or ipRules are needed
```

**Common Issue**: VNet/subnet not added to allowed list (when using hybrid setup)
```
Diagnosis: Source subnet not in network ACL allowed list

Fix: Add subnet to network ACL (only if intentionally using hybrid public/private access)
az cognitiveservices account network-rule add \
  --resource-group <rg> \
  --name <hub-name> \
  --subnet <subnet-id>

Note: For full private endpoint only, defaultAction=Deny is correct
```

### Phase 6: NSG Rules Check

**Verify NSG rules allow private endpoint traffic**:

```bash
# Get NSG for private endpoint subnet
SUBNET_ID=$(az network vnet subnet show -g <rg> --vnet-name <vnet-name> -n <pe-subnet-name> --query id -o tsv)
NSG_ID=$(az network vnet subnet show --ids $SUBNET_ID --query networkSecurityGroup.id -o tsv)

# List NSG rules
az network nsg rule list \
  --ids $NSG_ID \
  --query "[].{Name:name, Priority:priority, Direction:direction, Access:access, Source:sourceAddressPrefix, Dest:destinationAddressPrefix}" \
  --output table
```

**Required Rules**:
```
Inbound:
- Allow VirtualNetwork -> VirtualNetwork (all ports)
- Deny Internet -> * (all ports)

Outbound:
- Allow VirtualNetwork -> AzureCloud:443
- Allow VirtualNetwork -> VirtualNetwork (all ports)
```

**Issue: NSG blocking traffic**
```
Diagnosis: NSG denying private endpoint traffic

Fix: Update NSG rules
az network nsg rule create \
  --resource-group <rg> \
  --nsg-name <nsg-name> \
  --name AllowVNetInbound \
  --priority 100 \
  --direction Inbound \
  --access Allow \
  --protocol '*' \
  --source-address-prefixes VirtualNetwork \
  --destination-address-prefixes VirtualNetwork \
  --source-port-ranges '*' \
  --destination-port-ranges '*'
```

### Phase 7: RBAC and Permissions

**Verify RBAC assignments for managed identities**:

```bash
# Get Hub managed identity
HUB_MI=$(az cognitiveservices account show -g <rg> -n <hub-name> --query identity.principalId -o tsv)

# Check Storage permissions
az role assignment list \
  --assignee $HUB_MI \
  --scope <storage-id> \
  --query "[].{Role:roleDefinitionName}"

# Expected: Storage Blob Data Contributor
```

**Issue: "Authorization failed" errors**
```
Diagnosis: Missing RBAC assignment

Fix: Assign required roles
az role assignment create \
  --assignee $HUB_MI \
  --role "Storage Blob Data Contributor" \
  --scope <storage-id>

# Wait 60-120 seconds for propagation
```

### Phase 8: Connectivity Test

**Test actual connectivity** (from VM in VNet):

```bash
# Test Hub endpoint (expects 401 Unauthorized = reachable)
curl -I https://<hub-name>.cognitiveservices.azure.com

# Success indicators:
# - HTTP/1.1 401 Unauthorized (endpoint reachable, needs auth)
# - HTTP/1.1 403 Forbidden (endpoint reachable, RBAC issue)

# Failure indicators:
# - Connection timeout (network issue)
# - Connection refused (endpoint not listening)
# - Name resolution error (DNS issue)
```

**Test with Azure CLI** (uses authentication):

```bash
# List Hub capabilities (proves connectivity + auth)
az cognitiveservices account list-skus \
  --resource-group <rg> \
  --name <hub-name>

# Success: Returns SKU list
# Failure: Network error or permission denied
```

## Diagnostic Summary Report

After running diagnostics, generate summary:

```markdown
# Connectivity Diagnostic Report

**Resource**: <hub-name>
**Resource Group**: <rg>
**Date**: <date-time>

## Test Results

| Test | Result | Details |
|------|--------|---------|
| Resource Status | ✅ / ❌ | Hub: Succeeded |
| Private Endpoint Status | ✅ / ❌ | All Approved |
| DNS Zones | ✅ / ❌ | All zones present |
| DNS VNet Links | ✅ / ❌ | All linked |
| DNS Resolution | ✅ / ❌ | Resolves to 10.0.1.4 |
| Network ACLs | ✅ / ❌ | Deny by default (correct) |
| NSG Rules | ✅ / ❌ | VNet traffic allowed |
| RBAC | ✅ / ❌ | All assignments present |
| Connectivity | ✅ / ❌ | Hub reachable (401) |

## Issues Found

### ❌ Critical
1. [Issue description]
   - **Cause**: [root cause]
   - **Fix**: [specific fix command/action]

### ⚠️ Warnings
1. [Issue description]
   - **Recommendation**: [suggested action]

## Next Steps

1. [Action required]
2. [Action required]
3. Verify connectivity after fixes: [test command]
```

## Common Connectivity Patterns

### Pattern 1: "Works from Azure Portal, fails from VM"
**Diagnosis**: VM not in correct VNet or DNS not configured
**Fix**:
- Verify VM in same VNet as private endpoints
- Check VNet DNS settings (should be Azure DNS)
- Restart VM after DNS changes

### Pattern 2: "Worked yesterday, failing today"
**Diagnosis**: DNS cache or private endpoint state changed
**Fix**:
- Clear DNS cache: `ipconfig /flushdns` (Windows) or `sudo systemd-resolve --flush-caches` (Linux)
- Check if private endpoint was deleted
- Verify DNS zone records still exist

### Pattern 3: "Works for Hub, fails for Storage"
**Diagnosis**: Storage private endpoint or DNS misconfigured
**Fix**:
- Verify Storage-specific DNS zone: `privatelink.blob.core.windows.net`
- Check Storage private endpoint exists and is approved
- Test Storage endpoint separately

## Quick Fixes Reference

```bash
# Fix 1: Recreate private endpoint
az network private-endpoint delete -g <rg> -n <pe-name>
az deployment group create -g <rg> --template-file pe.bicep

# Fix 2: Reset VNet DNS
az network vnet update -g <rg> -n <vnet-name> --dns-servers ""

# Fix 3: Flush DNS cache (from VM)
# Windows: ipconfig /flushdns
# Linux: sudo systemd-resolve --flush-caches

# Fix 4: Re-assign RBAC (after 60s wait for identity propagation)
az role assignment create --assignee <mi> --role "Storage Blob Data Contributor" --scope <scope>

# Fix 5: Approve pending private endpoint
az network private-endpoint-connection approve \
  --resource-group <rg> --resource-name <service> --name <connection> --type <resource-type>
```

## Prevention Tips

1. **Always link DNS zones to VNet immediately after creating them**
2. **Use privateLinkServiceConnections (not manual) for auto-approval**
3. **Set subnet privateEndpointNetworkPolicies to 'Disabled'**
4. **Wait 5-10 minutes after deployment for DNS propagation**
5. **Test DNS resolution before testing connectivity**
6. **Document source location for connectivity tests (must be in VNet)**

## Escalation

If issue persists after all diagnostics:
1. Check Azure Service Health for region outages
2. Review Azure Portal Activity Log for deployment errors
3. Enable diagnostic logging on resources
4. Contact Azure Support with diagnostic report

Provide:
- Resource IDs
- Diagnostic test results (above)
- Timeline of when issue started
- Source location attempting connectivity
