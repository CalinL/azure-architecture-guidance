You are a co-engineer tasked with creating comprehensive documentation for setting up a **greenfield Azure tenant** for a SaaS startup under an existing Enterprise Agreement (EA).

## Context
| Attribute | Value |
|-----------|-------|
| **Scenario** | Greenfield Azure tenant for SaaS startup |
| **Billing** | New tenant under existing EA enrollment |
| **Connectivity** | Cloud-only (no on-premises/hybrid) |
| **Product Type** | Multi-region SaaS application |
| **Deployment** | Multi-region (active-active or active-passive) |
| **Compliance** | SOC 2 Type II, ISO 27001 |
| **IaC Tooling** | Terraform |
| **CI/CD Pipeline** | GitHub Actions |
| **Accelerator** | [Azure Landing Zones Terraform Module](https://github.com/Azure/terraform-azurerm-caf-enterprise-scale) |
| **Date** | January 2026 |

## Framework References
- Cloud Adoption Framework: https://learn.microsoft.com/azure/cloud-adoption-framework/
- Well-Architected Framework: https://learn.microsoft.com/azure/architecture/framework/
- Azure Landing Zones: https://learn.microsoft.com/azure/cloud-adoption-framework/ready/landing-zone/

## Scope & Phases

### Phase 1: Documentation (THIS PROMPT)
This prompt is for **documentation only**. The deliverable is comprehensive written guidance covering:
- Architecture decisions and rationale
- Configuration requirements and specifications
- Step-by-step procedures (not executable code)
- Reference diagrams and decision tables
- Best practices and recommendations

Code snippets included are for **illustration purposes only** — not production-ready implementation.

### Phase 2: Implementation (SEPARATE)
Actual Terraform code, GitHub Actions workflows, and Azure resource deployment will be handled in a **separate implementation phase** after documentation is reviewed and approved.

## Required Deliverables

### 1. Platform Landing Zones (Terraform-based)

#### Identity Landing Zone
- Microsoft Entra ID tenant configuration
- Privileged Identity Management (PIM) for admin roles
- Conditional Access policies (SOC2/ISO 27001 aligned)
- Break-glass emergency access accounts
- Service principal governance for Terraform automation

#### Management Landing Zone
- Management subscription with:
  - Centralized Log Analytics workspace
  - Microsoft Defender for Cloud (all plans evaluation)
  - Azure Monitor baseline (alerts, action groups)
- Azure Policy assignments for SOC2/ISO 27001:
  - Encryption at rest enforcement
  - TLS 1.2+ enforcement
  - Diagnostic settings enforcement
  - Allowed locations
  - Tag governance
- Cost Management configuration and budget alerts
- Microsoft Sentinel evaluation for SIEM

#### Connectivity Landing Zone (Multi-Region Cloud-Only SaaS)
- Hub virtual network per region (simplified, no gateway subnet needed)
- Hub-to-hub peering strategy for cross-region traffic
- Azure DNS Private Zones for PaaS services (multi-region resolution)
- Azure Firewall vs NSG-only decision (cost vs control)
- DDoS Protection plan (Standard tier for multi-region)
- Private Endpoint strategy for PaaS services
- Azure Front Door or Traffic Manager for global load balancing
- Cross-region failover networking considerations

### 2. EA and Subscription Architecture
- EA enrollment integration steps
- Management group hierarchy (CAF-aligned):
  ```
  Tenant Root
  └── Company Root
      ├── Platform
      │   ├── Identity
      │   ├── Management
      │   └── Connectivity
      ├── Landing Zones
      │   ├── Production
      │   └── Non-Production
      ├── Sandbox
      └── Decommissioned
  ```
- Subscription vending strategy for application teams
- Billing scope and cost allocation tagging standards

### 3. Terraform Implementation Guide
- Azure Landing Zones Terraform module configuration
- Remote state management (Azure Storage backend with state locking)
- Multi-region Terraform structure (modules per region vs single deployment)
- Module customization for startup scale (avoid over-provisioning)
- Terraform workspace or directory structure recommendation

### 4. GitHub Actions CI/CD Pipeline
- GitHub repository structure for platform landing zones
- Workload identity federation (OIDC) for GitHub Actions to Azure authentication
- GitHub Actions workflows for:
  - Terraform plan on pull requests
  - Terraform apply on merge to main
  - Drift detection scheduled workflows
- Environment protection rules and required reviewers
- GitHub Environments for dev/staging/production
- Secrets management (GitHub Secrets vs Azure Key Vault integration)
- Reusable workflow templates for application teams

### 5. Application Landing Zone Template
Provide a starter template that application teams can clone for new workloads:
- Subscription vending automation (request → approval → provisioning)
- Baseline Terraform module for application landing zone including:
  - Resource group structure
  - Virtual network (spoke) peered to hub
  - Key Vault for application secrets
  - Storage account with private endpoint
  - Azure Container Apps or AKS baseline (SaaS workload)
  - Application Insights and Log Analytics integration
- Required tags and naming conventions
- Network security baseline (NSG rules template)
- Azure Policy assignments inherited from management groups
- GitHub Actions workflow template for application CI/CD
- Documentation template for application teams

### 6. Compliance Baseline
- SOC 2 Type II control mapping to Azure services
- ISO 27001 Annex A control mapping
- Microsoft Defender for Cloud regulatory compliance dashboard setup
- Evidence collection automation strategy
- Multi-region data residency considerations

### 7. Day 1 / Day 2 Prioritization
- **Day 1 (MVP)**: Minimum viable platform to unblock application teams
- **Day 2 (Hardening)**: Full compliance posture and monitoring

## Output Format
- Markdown with clear sections and subsections
- Mermaid diagrams for:
  - Management group hierarchy
  - Multi-region network topology
  - Identity architecture
  - GitHub Actions workflow flow
  - Application landing zone structure
- Illustrative code snippets (for context, not production use)
- Configuration specification tables
- Decision tables with rationale (startup-appropriate choices)
- References section citing Microsoft Learn and GitHub sources

## Approach
1. **First**: Provide a detailed outline of all documentation sections for review
2. **Then**: Generate each documentation section with architecture guidance and specifications
3. **Finally**: Provide a phased rollout checklist to guide future implementation

**Remember**: This is documentation only. Do NOT generate production Terraform modules or GitHub Actions workflows — those will be created in a separate implementation phase.

Stay curious, use official Microsoft sources, and optimize for a small platform team managing a SaaS product.
