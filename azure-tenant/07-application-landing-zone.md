# Application Landing Zone Template

> **Related:** [README](./README.md) | [Connectivity Landing Zone](./03-connectivity-landing-zone.md) | [EA & Subscription Architecture](./04-ea-subscription-architecture.md)

## Overview

This document provides a template and guidelines for application teams to deploy workloads into their landing zone subscriptions. The template includes baseline infrastructure, networking, security controls, and CI/CD patterns.

## Application Landing Zone Architecture

```mermaid
graph TB
    subgraph "Application Landing Zone Subscription"
        subgraph "Networking"
            RG_NET[rg-{app}-network-{region}]
            SPOKE[Spoke VNet]
            SNET_APP[App Subnet]
            SNET_DATA[Data Subnet]
            SNET_PE[Private Endpoints]
            NSG_APP[NSG - App]
            NSG_DATA[NSG - Data]
        end
        
        subgraph "Shared Resources"
            RG_SHARED[rg-{app}-shared-{region}]
            KV[Key Vault]
            ST[Storage Account]
            APPI[App Insights]
        end
        
        subgraph "Application Resources"
            RG_APP[rg-{app}-app-{region}]
            ACA[Container Apps / AKS]
            ACR[Container Registry]
        end
        
        subgraph "Data Resources"
            RG_DATA[rg-{app}-data-{region}]
            SQL[(Azure SQL)]
            COSMOS[(Cosmos DB)]
            REDIS[(Redis Cache)]
        end
    end
    
    subgraph "Platform (Connectivity Subscription)"
        HUB[Hub VNet]
        DNS[Private DNS Zones]
        AFD[Azure Front Door]
    end
    
    SPOKE <-->|Peering| HUB
    PE --> DNS
    AFD --> ACA
    
    style SPOKE fill:#e3f2fd
    style ACA fill:#c8e6c9
```

## Subscription Baseline

When an application team receives a vended subscription, the following is automatically provisioned:

### Inherited from Management Groups

| Configuration | Source | Details |
|---------------|--------|---------|
| Azure Policies | Landing Zones MG | Tag requirements, diagnostics, security |
| RBAC | Subscription vending | Team permissions |
| Budget alerts | Automated | Based on request |
| Diagnostic settings | Policy (DINE) | Activity logs to central LA |

### Provisioned by Vending Process

| Resource | Name Pattern | Purpose |
|----------|--------------|---------|
| Resource Group | `rg-{app}-network-{region}` | Networking resources |
| VNet | `vnet-spoke-{app}-{region}-001` | Application network |
| VNet Peering | `peer-to-hub-{region}` | Connectivity to hub |
| Diagnostic Setting | `diag-activity-to-law` | Activity log forwarding |

## Resource Group Structure

```yaml
resource_groups:
  # Networking - managed by platform team initially
  - name: "rg-{app}-network-{region}"
    purpose: "VNet, subnets, NSGs, route tables"
    owner: "Platform Team (initial), App Team (ongoing)"
    
  # Shared resources - application team
  - name: "rg-{app}-shared-{region}"
    purpose: "Key Vault, Storage, App Insights"
    owner: "Application Team"
    
  # Application compute - application team
  - name: "rg-{app}-app-{region}"
    purpose: "Container Apps, AKS, App Service"
    owner: "Application Team"
    
  # Data resources - application team
  - name: "rg-{app}-data-{region}"
    purpose: "Databases, caches, queues"
    owner: "Application Team"
```

## Network Configuration

### Spoke VNet Template

```yaml
spoke_virtual_network:
  name: "vnet-spoke-{app}-{region}-001"
  address_space: ["10.{x}.0.0/16"]  # Allocated during vending
  
  subnets:
    - name: "snet-app"
      address_prefix: "10.{x}.0.0/24"
      delegations: []  # Or container apps delegation
      network_security_group: "nsg-{app}-app-{region}"
      service_endpoints:
        - "Microsoft.Storage"
        - "Microsoft.KeyVault"
        - "Microsoft.Sql"
        
    - name: "snet-containerapp"
      address_prefix: "10.{x}.1.0/23"  # /23 for Container Apps
      delegations:
        - name: "Microsoft.App/environments"
          service: "Microsoft.App/environments"
          actions:
            - "Microsoft.Network/virtualNetworks/subnets/join/action"
      network_security_group: "nsg-{app}-containerapp-{region}"
      
    - name: "snet-data"
      address_prefix: "10.{x}.10.0/24"
      network_security_group: "nsg-{app}-data-{region}"
      
    - name: "snet-privateendpoints"
      address_prefix: "10.{x}.20.0/24"
      private_endpoint_network_policies: "Disabled"
      network_security_group: "nsg-{app}-privateendpoints-{region}"
```

### NSG Rules for Application Subnet

```yaml
nsg_rules:
  inbound:
    - name: "Allow-FrontDoor"
      priority: 100
      source_address_prefix: "AzureFrontDoor.Backend"
      destination_port_ranges: ["443", "80"]
      access: "Allow"
      
    - name: "Allow-Hub-Management"
      priority: 200
      source_address_prefix: "10.0.0.0/16"  # Hub VNet
      destination_port_ranges: ["443", "22", "3389"]
      access: "Allow"
      
    - name: "Allow-VNet-Internal"
      priority: 300
      source_address_prefix: "VirtualNetwork"
      destination_port_range: "*"
      access: "Allow"
      
    - name: "Deny-All-Inbound"
      priority: 4096
      source_address_prefix: "*"
      destination_port_range: "*"
      access: "Deny"
      
  outbound:
    - name: "Allow-Internet-HTTPS"
      priority: 100
      destination_address_prefix: "Internet"
      destination_port_range: "443"
      access: "Allow"
      
    - name: "Allow-Azure-Services"
      priority: 200
      destination_address_prefix: "AzureCloud"
      destination_port_ranges: ["443", "1433", "5432", "6379"]
      access: "Allow"
```

## Shared Resources Template

### Key Vault

```yaml
key_vault:
  name: "kv-{app}-{env}-{region}-001"  # Max 24 chars
  resource_group: "rg-{app}-shared-{region}"
  
  sku: "standard"
  
  # Security configuration
  tenant_id: "{tenant-id}"
  soft_delete_retention_days: 90
  purge_protection_enabled: true
  
  # Network configuration
  public_network_access_enabled: false
  network_acls:
    default_action: "Deny"
    bypass: "AzureServices"
    
  # RBAC (preferred over access policies)
  enable_rbac_authorization: true
  
  # Private endpoint
  private_endpoint:
    name: "pe-{app}-kv-{region}"
    subnet_id: "/subscriptions/.../snet-privateendpoints"
    private_dns_zone_id: "/subscriptions/.../privatelink.vaultcore.azure.net"
```

### Application Insights

```yaml
application_insights:
  name: "appi-{app}-{env}-{region}-001"
  resource_group: "rg-{app}-shared-{region}"
  
  application_type: "web"
  
  # Connect to central Log Analytics
  workspace_id: "/subscriptions/.../law-platform-{region}-001"
  
  # Sampling (cost optimization)
  sampling_percentage: 100  # Reduce in production if needed
  
  # Disable IP masking for troubleshooting (evaluate privacy)
  disable_ip_masking: false
  
  # Tags
  tags:
    Application: "{app}"
    Environment: "{env}"
```

### Storage Account

```yaml
storage_account:
  name: "st{app}{env}{region}001"  # No hyphens, max 24 chars
  resource_group: "rg-{app}-shared-{region}"
  
  account_tier: "Standard"
  account_replication_type: "GRS"  # Geo-redundant
  account_kind: "StorageV2"
  
  # Security
  min_tls_version: "TLS1_2"
  allow_nested_items_to_be_public: false
  shared_access_key_enabled: false  # Use Entra ID
  public_network_access_enabled: false
  
  # Blob properties
  blob_properties:
    versioning_enabled: true
    delete_retention_policy:
      days: 30
    container_delete_retention_policy:
      days: 30
      
  # Private endpoint
  private_endpoints:
    - name: "pe-{app}-blob-{region}"
      subresource: "blob"
    - name: "pe-{app}-file-{region}"
      subresource: "file"
```

## Compute Options

### Option 1: Azure Container Apps (Recommended for SaaS)

```mermaid
graph TB
    subgraph "Container Apps Environment"
        ENV[Managed Environment<br/>cae-{app}-{env}-{region}]
        
        subgraph "Container Apps"
            API[API Service<br/>ca-api-{app}]
            WORKER[Background Worker<br/>ca-worker-{app}]
            WEB[Web Frontend<br/>ca-web-{app}]
        end
        
        subgraph "Configuration"
            DAPR[Dapr Components]
            SECRETS[Secrets from Key Vault]
            SCALE[Scale Rules]
        end
    end
    
    ENV --> API
    ENV --> WORKER
    ENV --> WEB
    
    DAPR --> API
    SECRETS --> API
    SCALE --> API
```

```yaml
container_apps_environment:
  name: "cae-{app}-{env}-{region}-001"
  resource_group: "rg-{app}-app-{region}"
  
  # Networking
  infrastructure_subnet_id: "/subscriptions/.../snet-containerapp"
  internal_load_balancer_enabled: true
  
  # Workload profiles (consumption or dedicated)
  workload_profile:
    - name: "Consumption"
      workload_profile_type: "Consumption"
      
  # Log Analytics connection
  log_analytics_workspace_id: "/subscriptions/.../law-platform-{region}"
  
container_app:
  name: "ca-api-{app}-{env}"
  
  # Container configuration
  template:
    containers:
      - name: "api"
        image: "{acr}.azurecr.io/{app}/api:latest"
        cpu: 0.5
        memory: "1Gi"
        
        env:
          - name: "APPLICATIONINSIGHTS_CONNECTION_STRING"
            secret_ref: "appinsights-connection"
          - name: "DATABASE_CONNECTION"
            secret_ref: "db-connection"
            
    # Scaling
    min_replicas: 2
    max_replicas: 10
    
    scale:
      rules:
        - name: "http-scaling"
          http:
            metadata:
              concurrentRequests: "100"
              
  # Ingress
  ingress:
    external: false  # Internal only, use Front Door for external
    target_port: 8080
    transport: "http"
    
  # Identity
  identity:
    type: "SystemAssigned"
```

### Option 2: Azure Kubernetes Service (AKS)

```yaml
aks_cluster:
  name: "aks-{app}-{env}-{region}-001"
  resource_group: "rg-{app}-app-{region}"
  
  # Kubernetes version
  kubernetes_version: "1.29"
  
  # Identity
  identity:
    type: "SystemAssigned"
    
  # Network configuration
  network_profile:
    network_plugin: "azure"
    network_policy: "azure"  # Or "calico"
    service_cidr: "10.{x}.200.0/24"
    dns_service_ip: "10.{x}.200.10"
    
  # Default node pool
  default_node_pool:
    name: "system"
    vm_size: "Standard_D4s_v5"
    node_count: 3
    vnet_subnet_id: "/subscriptions/.../snet-aks"
    zones: ["1", "2", "3"]
    
  # Additional node pools
  node_pools:
    - name: "workload"
      vm_size: "Standard_D8s_v5"
      min_count: 2
      max_count: 10
      enable_auto_scaling: true
      
  # Add-ons
  addon_profile:
    azure_policy:
      enabled: true
    oms_agent:
      enabled: true
      log_analytics_workspace_id: "/subscriptions/.../law-platform-{region}"
```

## Data Services

### Azure SQL Database

```yaml
sql_server:
  name: "sql-{app}-{env}-{region}-001"
  resource_group: "rg-{app}-data-{region}"
  
  version: "12.0"
  
  # Authentication
  administrator_login: "sqladmin"
  administrator_login_password: "from-key-vault"
  
  # Entra ID admin (recommended)
  azuread_administrator:
    login_username: "SQL Admins Group"
    object_id: "{group-object-id}"
    
  # Security
  minimum_tls_version: "1.2"
  public_network_access_enabled: false
  
sql_database:
  name: "sqldb-{app}-{env}"
  
  sku:
    name: "GP_S_Gen5_2"  # Serverless
    tier: "GeneralPurpose"
    
  # Serverless configuration
  auto_pause_delay_in_minutes: 60
  min_capacity: 0.5
  max_size_gb: 32
  
  # Backup
  short_term_retention_days: 7
  long_term_retention:
    weekly_retention: "P4W"
    monthly_retention: "P12M"
    yearly_retention: "P5Y"
```

### Cosmos DB

```yaml
cosmosdb_account:
  name: "cosmos-{app}-{env}-{region}"
  resource_group: "rg-{app}-data-{region}"
  
  offer_type: "Standard"
  kind: "GlobalDocumentDB"
  
  # Multi-region (for production)
  geo_locations:
    - location: "eastus2"
      failover_priority: 0
    - location: "westus2"
      failover_priority: 1
      
  # Consistency
  consistency_policy:
    consistency_level: "Session"
    
  # Networking
  public_network_access_enabled: false
  is_virtual_network_filter_enabled: true
  
  # Backup
  backup:
    type: "Continuous"
    tier: "Continuous7Days"
```

## Tagging Requirements

All resources must include these tags (enforced by Azure Policy):

```yaml
required_tags:
  Environment: "{Production|Development|Staging|Sandbox}"
  CostCenter: "{cost-center-code}"
  Owner: "{team-email}"
  Application: "{application-name}"
  DataClassification: "{Public|Internal|Confidential|Restricted}"
  
recommended_tags:
  Project: "{project-code}"
  CreatedBy: "terraform"
  Repository: "{github-repo-url}"
```

## Naming Convention Summary

| Resource | Pattern | Example |
|----------|---------|---------|
| Resource Group | `rg-{app}-{purpose}-{region}` | `rg-saas-app-eastus2` |
| Virtual Network | `vnet-spoke-{app}-{region}-{###}` | `vnet-spoke-saas-eastus2-001` |
| Subnet | `snet-{purpose}` | `snet-app`, `snet-data` |
| NSG | `nsg-{app}-{purpose}-{region}` | `nsg-saas-app-eastus2` |
| Key Vault | `kv-{app}-{env}-{region}-{###}` | `kv-saas-prod-eus2-001` |
| Storage Account | `st{app}{env}{region}{###}` | `stsaasprodeus2001` |
| Container App Env | `cae-{app}-{env}-{region}-{###}` | `cae-saas-prod-eastus2-001` |
| Container App | `ca-{service}-{app}-{env}` | `ca-api-saas-prod` |
| SQL Server | `sql-{app}-{env}-{region}-{###}` | `sql-saas-prod-eastus2-001` |
| Cosmos DB | `cosmos-{app}-{env}-{region}` | `cosmos-saas-prod-eastus2` |
| App Insights | `appi-{app}-{env}-{region}-{###}` | `appi-saas-prod-eastus2-001` |

## Sample Terraform Module

### Application Landing Zone Module

```hcl
# modules/app-landing-zone/main.tf

terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 4.0"
    }
  }
}

variable "app_name" {
  description = "Application name"
  type        = string
}

variable "environment" {
  description = "Environment (prod, dev, staging)"
  type        = string
}

variable "location" {
  description = "Azure region"
  type        = string
}

variable "spoke_address_space" {
  description = "VNet address space"
  type        = list(string)
}

variable "hub_vnet_id" {
  description = "Hub VNet ID for peering"
  type        = string
}

variable "log_analytics_workspace_id" {
  description = "Central Log Analytics workspace ID"
  type        = string
}

variable "private_dns_zone_ids" {
  description = "Map of private DNS zone IDs"
  type        = map(string)
}

variable "tags" {
  description = "Resource tags"
  type        = map(string)
}

locals {
  name_prefix = "${var.app_name}-${var.environment}"
}

# Resource Groups
resource "azurerm_resource_group" "network" {
  name     = "rg-${local.name_prefix}-network-${var.location}"
  location = var.location
  tags     = var.tags
}

resource "azurerm_resource_group" "shared" {
  name     = "rg-${local.name_prefix}-shared-${var.location}"
  location = var.location
  tags     = var.tags
}

resource "azurerm_resource_group" "app" {
  name     = "rg-${local.name_prefix}-app-${var.location}"
  location = var.location
  tags     = var.tags
}

resource "azurerm_resource_group" "data" {
  name     = "rg-${local.name_prefix}-data-${var.location}"
  location = var.location
  tags     = var.tags
}

# Virtual Network
resource "azurerm_virtual_network" "spoke" {
  name                = "vnet-spoke-${local.name_prefix}-${var.location}-001"
  location            = var.location
  resource_group_name = azurerm_resource_group.network.name
  address_space       = var.spoke_address_space
  tags                = var.tags
}

# Subnets
resource "azurerm_subnet" "app" {
  name                 = "snet-app"
  resource_group_name  = azurerm_resource_group.network.name
  virtual_network_name = azurerm_virtual_network.spoke.name
  address_prefixes     = [cidrsubnet(var.spoke_address_space[0], 8, 0)]
  
  service_endpoints = ["Microsoft.Storage", "Microsoft.KeyVault"]
}

resource "azurerm_subnet" "containerapp" {
  name                 = "snet-containerapp"
  resource_group_name  = azurerm_resource_group.network.name
  virtual_network_name = azurerm_virtual_network.spoke.name
  address_prefixes     = [cidrsubnet(var.spoke_address_space[0], 7, 1)]
  
  delegation {
    name = "containerapp"
    service_delegation {
      name    = "Microsoft.App/environments"
      actions = ["Microsoft.Network/virtualNetworks/subnets/join/action"]
    }
  }
}

resource "azurerm_subnet" "privateendpoints" {
  name                              = "snet-privateendpoints"
  resource_group_name               = azurerm_resource_group.network.name
  virtual_network_name              = azurerm_virtual_network.spoke.name
  address_prefixes                  = [cidrsubnet(var.spoke_address_space[0], 8, 20)]
  private_endpoint_network_policies = "Disabled"
}

# VNet Peering to Hub
resource "azurerm_virtual_network_peering" "spoke_to_hub" {
  name                      = "peer-to-hub"
  resource_group_name       = azurerm_resource_group.network.name
  virtual_network_name      = azurerm_virtual_network.spoke.name
  remote_virtual_network_id = var.hub_vnet_id
  
  allow_virtual_network_access = true
  allow_forwarded_traffic      = true
}

# Key Vault
resource "azurerm_key_vault" "main" {
  name                = "kv-${var.app_name}-${var.environment}-${substr(var.location, 0, 4)}"
  location            = var.location
  resource_group_name = azurerm_resource_group.shared.name
  tenant_id           = data.azurerm_client_config.current.tenant_id
  sku_name            = "standard"
  
  purge_protection_enabled   = true
  soft_delete_retention_days = 90
  enable_rbac_authorization  = true
  
  public_network_access_enabled = false
  
  tags = var.tags
}

# Key Vault Private Endpoint
resource "azurerm_private_endpoint" "keyvault" {
  name                = "pe-${var.app_name}-kv-${var.location}"
  location            = var.location
  resource_group_name = azurerm_resource_group.shared.name
  subnet_id           = azurerm_subnet.privateendpoints.id
  
  private_service_connection {
    name                           = "psc-keyvault"
    private_connection_resource_id = azurerm_key_vault.main.id
    subresource_names              = ["vault"]
    is_manual_connection           = false
  }
  
  private_dns_zone_group {
    name                 = "pdzg-keyvault"
    private_dns_zone_ids = [var.private_dns_zone_ids["keyvault"]]
  }
  
  tags = var.tags
}

# Application Insights
resource "azurerm_application_insights" "main" {
  name                = "appi-${local.name_prefix}-${var.location}-001"
  location            = var.location
  resource_group_name = azurerm_resource_group.shared.name
  workspace_id        = var.log_analytics_workspace_id
  application_type    = "web"
  
  tags = var.tags
}

# Outputs
output "spoke_vnet_id" {
  value = azurerm_virtual_network.spoke.id
}

output "key_vault_id" {
  value = azurerm_key_vault.main.id
}

output "key_vault_uri" {
  value = azurerm_key_vault.main.vault_uri
}

output "application_insights_connection_string" {
  value     = azurerm_application_insights.main.connection_string
  sensitive = true
}

data "azurerm_client_config" "current" {}
```

## Documentation Template for Application Teams

Application teams should maintain a `README.md` in their infrastructure repository with the following structure:

```markdown
# {Application Name} Infrastructure

## Overview
Brief description of the application and its purpose.

## Architecture
{Insert architecture diagram}

## Azure Resources

| Resource | Name | Purpose |
|----------|------|---------|
| Subscription | sub-{app}-{env} | Application hosting |
| Resource Groups | rg-{app}-* | Resource organization |
| Container Apps | ca-{app}-* | Application compute |
| Key Vault | kv-{app}-* | Secrets management |
| Cosmos DB | cosmos-{app}-* | Data persistence |

## Network Configuration

| Subnet | CIDR | Purpose |
|--------|------|---------|
| snet-app | 10.x.0.0/24 | Application workloads |
| snet-data | 10.x.10.0/24 | Data services |
| snet-privateendpoints | 10.x.20.0/24 | Private endpoints |

## Deployment

### Prerequisites
- Azure CLI v2.x+
- Terraform v1.7.0+
- Access to application landing zone subscription

### Steps
1. Clone this repository
2. Run `terraform init`
3. Run `terraform plan -var-file="environments/{env}.tfvars"`
4. Submit PR for review
5. After approval, merge to trigger apply

## Monitoring

- **Application Insights**: {workspace-name}
- **Log Analytics**: {workspace-id}
- **Alerts**: Configured via Azure Monitor

## Security

- [ ] All secrets stored in Key Vault
- [ ] Private endpoints for PaaS services
- [ ] NSG rules reviewed
- [ ] Managed identity used (no service principal secrets)

## Contacts

| Role | Team/Person | Contact |
|------|-------------|---------|
| Owner | {team-name} | {email} |
| Platform Support | Platform Team | platform@company.com |

## References
- [Application Landing Zone Guide](./07-application-landing-zone.md)
- [Company Azure Standards](./README.md)
```

## Onboarding Checklist

Application teams should complete this checklist when setting up their landing zone:

### Pre-Deployment

- [ ] Submit subscription request via vending process
- [ ] Receive subscription with baseline configuration
- [ ] Verify VNet peering to hub is active
- [ ] Confirm diagnostic settings are in place

### Shared Resources

- [ ] Deploy Key Vault with private endpoint
- [ ] Store database connection strings in Key Vault
- [ ] Configure Application Insights
- [ ] Set up Storage Account (if needed)

### Networking

- [ ] Review and customize NSG rules
- [ ] Create additional subnets if needed
- [ ] Create private endpoints for PaaS services
- [ ] Verify DNS resolution for private endpoints

### Application Deployment

- [ ] Deploy Container Apps Environment or AKS
- [ ] Configure managed identity
- [ ] Assign Key Vault access to managed identity
- [ ] Deploy application containers

### Security & Compliance

- [ ] Verify all resources are tagged correctly
- [ ] Confirm Defender for Cloud is enabled
- [ ] Review security recommendations
- [ ] Enable threat protection on data services

---

**Previous:** [06 - GitHub Actions CI/CD](./06-github-actions-cicd.md) | **Next:** [08 - Compliance Baseline](./08-compliance-baseline.md)
