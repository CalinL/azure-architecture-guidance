# Microsoft Foundry Portal: Private Networking Comparison

## New Foundry Portal vs. Classic Foundry Portal

This document provides a comprehensive comparison of private networking capabilities between the **new Microsoft Foundry portal experience** and the **classic Foundry portal experience**.

> **Last Updated:** January 2026  
> **Important:** This document is based on official Microsoft documentation. Private networking support in the new Foundry portal may change as Microsoft continues to develop the platform.

---

## Executive Summary

| Capability | New Foundry Portal | Classic Foundry Portal |
|------------|-------------------|------------------------|
| End-to-end network isolation | ❌ **Not Supported** | ✅ Supported |
| Projects with disabled public network access | ❌ **Not Supported** | ✅ Supported |
| Private endpoint access to Foundry | ❌ Portal UI doesn't work | ✅ Supported |
| Managed virtual network for Agents | ⚠️ Preview (Classic experience only) | ✅ Supported |
| Custom VNet for Agents | ❌ Not Supported | ✅ Supported (GA) |
| Private MCP servers in VNet | ❌ Not Supported | ❌ Not Supported |
| Hosted Agents with network isolation | ❌ Not Supported | ❌ Not Supported |

---

## Key Limitation: New Foundry Portal Does Not Support Network Isolation

When attempting to switch a network-isolated project to the new Foundry portal experience, users receive the following error:

> *"Your current setup uses a project, resource, region, custom domain, or disabled public network access that isn't supported in the new Foundry experience yet. To continue, select a different project with a supported configuration."*

### Official Documentation Statement

From [How to configure a private link for Microsoft Foundry (Foundry projects)](https://learn.microsoft.com/en-us/azure/ai-foundry/how-to/configure-private-link?view=foundry):

> **"End-to-end network isolation in Foundry is not support in the new Foundry portal experience. End-to-end network isolation in Foundry is not supported for the new version of the Agent service. Use the classic Foundry portal experience with the current version of Agent service to securely access your Foundry projects when network isolation is enabled."**

And:

> **"End-to-end network isolation isn't supported in the new Foundry portal experience. Use the classic Foundry portal experience or the SDK or CLI to securely access your Foundry projects when network isolation is enabled."**

---

## Detailed Feature Comparison

### 1. Private Link / Private Endpoint Support

#### New Foundry Portal
- **Status:** Not Supported for portal access
- Private endpoints can be created for the Foundry resource
- However, the new portal UI **cannot be used** to access projects with private endpoints when public network access is disabled

#### Classic Foundry Portal
- **Status:** Fully Supported
- Private endpoints work for secure access to Foundry projects
- Users can access network-isolated projects through the portal

**Reference:** [How to configure a private link for Microsoft Foundry (Foundry projects)](https://learn.microsoft.com/en-us/azure/ai-foundry/how-to/configure-private-link?view=foundry)

---

### 2. Managed Virtual Network for Foundry Projects

#### New Foundry Portal
- **Status:** ⚠️ Preview Feature - BUT requires Classic experience
- Managed virtual network is a **preview feature** for securing Agents service compute
- **Critical Limitation:** "Supports only Standard BYO resources Agents v1 and the **Foundry classic experience**"

#### Classic Foundry Portal
- **Status:** Supported (Preview)
- Full support for managed virtual network isolation
- Can be deployed via Bicep template

**Supported Regions for Managed VNet:**
- East US, East US2, Japan East, France Central, UAE North
- Brazil South, Spain Central, Germany West Central, Italy North
- South Central US, West Central US, Australia East, Sweden Central
- Canada East, South Africa North, West Europe, West US, West US 3
- South India, UK South

**Reference:** [Configure managed virtual network for Microsoft Foundry projects](https://learn.microsoft.com/en-us/azure/ai-foundry/how-to/managed-virtual-network?view=foundry)

---

### 3. Custom Virtual Network (BYO VNet) for Agents

#### New Foundry Portal
- **Status:** Not Supported

#### Classic Foundry Portal
- **Status:** GA (Generally Available)
- Full network isolation through virtual network injection
- Supports Standard Agent deployment and evaluations

**Reference:** [How to use a virtual network with the Azure AI Agent Service](https://learn.microsoft.com/en-us/azure/ai-services/agents/how-to/virtual-networks)

---

### 4. Agent Service Network Isolation

#### New Foundry Portal
- **Status:** Not Supported
- The new version of the Agent service does not support end-to-end network isolation

#### Classic Foundry Portal
- **Status:** Supported
- Network injection for Standard Agents and evaluations
- Protects against data exfiltration

**Important Limitations (Both Portals):**
- Hosted Agents are **not supported** with end-to-end network isolation
- Private MCP servers deployed in the same virtual network are **not supported** - only publicly accessible MCP servers can be used
- Basic Agents don't require network isolation

**Reference:** [How to configure a private link for Microsoft Foundry (Foundry projects) - Limitations](https://learn.microsoft.com/en-us/azure/ai-foundry/how-to/configure-private-link?view=foundry#limitations)

---

### 5. Disabled Public Network Access

#### New Foundry Portal
- **Status:** Not Supported
- Projects with `publicNetworkAccess: Disabled` cannot be accessed through the new portal

#### Classic Foundry Portal
- **Status:** Supported
- Full support for projects with disabled public network access
- Access via private endpoints, VPN Gateway, ExpressRoute, or Azure Bastion

**Reference:** [How to configure a private link for Microsoft Foundry (Foundry projects)](https://learn.microsoft.com/en-us/azure/ai-foundry/how-to/configure-private-link?view=foundry)

---

## Workarounds for Network-Isolated Projects

If you have a network-isolated Foundry project, you have the following options:

### Option 1: Use the Classic Foundry Portal
Continue using the classic experience at [https://ai.azure.com](https://ai.azure.com) with your private network configuration.

### Option 2: Use SDK or CLI
Access your Foundry projects programmatically when network isolation is enabled:
- Azure CLI
- Azure SDKs (Python, .NET, etc.)
- REST APIs

### Option 3: Enable Public Network Access (Not Recommended for Production)
If you temporarily need new portal access for non-sensitive work:
1. Go to Azure Portal → Your Foundry Project
2. Navigate to **Networking** → **Firewalls and virtual networks**
3. Select **All networks**
4. Save changes

> ⚠️ **Warning:** This exposes your project to the public internet and is not recommended for production environments with sensitive data.

---

## Managed Virtual Network Limitations (Preview)

When using managed virtual network isolation (preview feature), note these limitations:

1. **Deployment Method:** Can only be deployed via the Bicep template in [foundry-samples repository](https://github.com/azure-ai-foundry/foundry-samples/tree/main/infrastructure/infrastructure-setup-bicep/18-managed-virtual-network-preview)

2. **Portal Experience:** Supports only **Foundry classic experience**, not the new portal

3. **Firewall Costs:** FQDN outbound rules in "Allow Only Approved Outbound" mode create a managed Azure Firewall with associated costs

4. **No Rollback:** Cannot disable managed virtual network isolation after enabling it

5. **Outbound Rules:** Must be created through Azure CLI

6. **MCP Tools:** End-to-end network isolation for Agent MCP tools with managed virtual network is not supported

7. **Preview Registration:** Requires preview feature registration for `AI.ManagedVnetPreview` flag

**Reference:** [Configure managed virtual network for Microsoft Foundry projects - Limitations](https://learn.microsoft.com/en-us/azure/ai-foundry/how-to/managed-virtual-network?view=foundry#limitations)

---

## Network Architecture Comparison

### Classic Portal with Custom VNet (GA)

```
┌─────────────────────────────────────────────────────────────────┐
│                     Customer Virtual Network                     │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐         │
│  │   Client    │    │  Private    │    │   Foundry   │         │
│  │   (VM/VPN)  │───▶│  Endpoint   │───▶│   Project   │         │
│  └─────────────┘    └─────────────┘    └─────────────┘         │
│         │                                      │                 │
│         │           Private Endpoints          │                 │
│         │    ┌──────────────────────────┐     │                 │
│         │    │  ┌────────┐ ┌────────┐  │     │                 │
│         └───▶│  │Storage │ │AI Search│  │◀────┘                 │
│              │  └────────┘ └────────┘  │                        │
│              │  ┌────────┐ ┌────────┐  │                        │
│              │  │Cosmos  │ │Key Vault│  │                        │
│              │  └────────┘ └────────┘  │                        │
│              └──────────────────────────┘                        │
└─────────────────────────────────────────────────────────────────┘
```

### New Portal (Public Access Only)

```
┌─────────────────────────────────────────────────────────────────┐
│                        Public Internet                           │
│                              │                                   │
│                              ▼                                   │
│                    ┌─────────────────┐                          │
│                    │   New Foundry   │                          │
│                    │     Portal      │                          │
│                    │  (ai.azure.com) │                          │
│                    └─────────────────┘                          │
│                              │                                   │
│                              ▼                                   │
│                    ┌─────────────────┐                          │
│                    │    Foundry      │                          │
│                    │    Project      │                          │
│                    │ (Public Access) │                          │
│                    └─────────────────┘                          │
│                                                                  │
│  ❌ Projects with private networking cannot be accessed          │
└─────────────────────────────────────────────────────────────────┘
```

---

## Recommendations

### For Production Environments Requiring Network Isolation

1. **Use the Classic Foundry Portal** until the new portal supports private networking
2. Deploy using the **custom virtual network (BYO VNet)** approach which is GA
3. Use **SDK/CLI** for programmatic access to network-isolated resources

### For Development/Testing Without Sensitive Data

1. Consider using the new portal with public network access
2. Use Azure RBAC and identity-based security as alternative protection layers

### For Future Planning

1. Monitor Microsoft documentation for updates on new portal network isolation support
2. Plan migration path when new portal gains network isolation capabilities
3. Consider the **managed virtual network preview** for new deployments if preview features are acceptable

---

## Official References

| Topic | URL |
|-------|-----|
| Configure private link for Foundry projects | https://learn.microsoft.com/en-us/azure/ai-foundry/how-to/configure-private-link?view=foundry |
| Configure managed virtual network for Foundry projects | https://learn.microsoft.com/en-us/azure/ai-foundry/how-to/managed-virtual-network?view=foundry |
| Configure private link for Foundry hubs (hub-based projects) | https://learn.microsoft.com/en-us/azure/ai-foundry/how-to/hub-configure-private-link?view=foundry-classic |
| Configure managed network for Foundry hubs | https://learn.microsoft.com/en-us/azure/ai-foundry/how-to/configure-managed-network?view=foundry-classic |
| Virtual networks with Azure AI Agent Service | https://learn.microsoft.com/en-us/azure/ai-foundry/agents/how-to/virtual-networks?view=foundry-classic |
| Azure AI Foundry security baseline | https://learn.microsoft.com/en-us/security/benchmark/azure/baselines/azure-ai-foundry-security-baseline |
| Foundry samples - Managed VNet Bicep template | https://github.com/azure-ai-foundry/foundry-samples/tree/main/infrastructure/infrastructure-setup-bicep/18-managed-virtual-network-preview |
| Upgrade from Azure OpenAI to Microsoft Foundry | https://learn.microsoft.com/en-us/azure/ai-foundry/how-to/upgrade-azure-openai?view=foundry-classic |

---

## Changelog

| Date | Change |
|------|--------|
| January 2026 | Initial documentation based on Microsoft Learn documentation |

---

*This document is maintained as part of the Azure Architecture Guidance repository. For corrections or updates, please submit a pull request.*
