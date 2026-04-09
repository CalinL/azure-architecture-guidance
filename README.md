---
title: Azure Architecture Guidance
description: Architecture guidance and best practices for building enterprise solutions on Microsoft Azure, covering landing zones, AI/ML platforms, mission-critical SaaS, and security.
---

A curated collection of deep-dive architecture guidance and best practices for enterprise Azure solutions, spanning landing zones, AI platforms, mission-critical SaaS patterns, and security.

## Azure Container Apps

| Document | Description |
|----------|-------------|
| [Easy Auth Deep Dive](aca/container-apps-easy-auth-deep-dive.md) | Complete architecture and token store deep dive for Azure Container Apps Easy Auth |

## Azure Landing Zone

End-to-end guidance for setting up a greenfield Azure tenant for a multi-region SaaS startup under an EA enrollment, using Terraform and GitHub Actions.

| Document | Description |
|----------|-------------|
| [Overview](azure-tenant/README.md) | Scenario context, architecture overview, and implementation roadmap |
| [Identity Landing Zone](azure-tenant/01-identity-landing-zone.md) | Entra ID, management groups, PIM, and break-glass account configuration |
| [Management Landing Zone](azure-tenant/02-management-landing-zone.md) | Log Analytics, Azure Monitor, and centralized management subscription |
| [Connectivity Landing Zone](azure-tenant/03-connectivity-landing-zone.md) | Hub-and-spoke networking, Azure Firewall, and ExpressRoute/VPN design |
| [EA & Subscription Architecture](azure-tenant/04-ea-subscription-architecture.md) | EA enrollment structure, subscription vending, and management group hierarchy |
| [Terraform Implementation](azure-tenant/05-terraform-implementation.md) | Terraform modules, state management, and the CAF Enterprise Scale module |
| [GitHub Actions CI/CD](azure-tenant/06-github-actions-cicd.md) | Federated identity, Terraform pipelines, and secure workflow patterns |
| [Application Landing Zone](azure-tenant/07-application-landing-zone.md) | Per-workload landing zone template with networking, Key Vault, and observability |
| [Compliance Baseline](azure-tenant/08-compliance-baseline.md) | SOC 2 Type II and ISO 27001 controls mapped to Azure Policy assignments |
| [Day 1 / Day 2 Prioritization](azure-tenant/09-day1-day2-prioritization.md) | Phased rollout priorities for new Azure tenant onboarding |
| [DNS Private Zones & Resolver](azure-tenant/connectivity-dns-private-zones-resolver.md) | Private DNS Zones and Azure DNS Private Resolver in the Connectivity Landing Zone |

## Microsoft Foundry

| Document | Description |
|----------|-------------|
| [Comprehensive Models Guide](foundry/microsoft-foundry-models-comprehensive-guide.md) | Complete reference for model catalog, deployment types, and SDK usage in Microsoft Foundry |
| [Deploying Claude Models](foundry/deploying-claude-models-microsoft-foundry.md) | Step-by-step guide for deploying Anthropic Claude models via the Foundry model catalog |
| [Content Safety & Guardrails](foundry/Microsoft_Foundry_Content_Safety_Insurance_Solutions.md) | Content safety filters, guardrails, and responsible AI controls for Foundry deployments |
| [Private Networking Comparison](foundry/microsoft-foundry-private-networking-comparison.md) | Comparison of private networking options in the Microsoft Foundry portal |
| [Tracing & Observability](foundry/microsoft-foundry-tracing-guide.md) | Configuring tracing, SDK instrumentation, and Azure Monitor integration for Foundry agents |

## SharePoint + Microsoft Foundry

| Document | Description |
|----------|-------------|
| [AI Knowledge Accelerator](foundry-sharepoint/SharePoint-Foundry-Accelerator-Research.md) | Deep research and recommendations for building an AI knowledge base on SharePoint with Foundry |

## GPU & AI Models

| Document | Description |
|----------|-------------|
| [GPU Hosting Options](gpu-serverless/azure-gpu-hosting-open-source-models-guidance.md) | Decision guide for hosting open-source AI models on Azure GPU infrastructure |

## Mission-Critical SaaS

| Document | Description |
|----------|-------------|
| [Azure Container Apps Overview](mission-critical-saas/azure-container-apps-overview.md) | Comprehensive architecture guide for Azure Container Apps in mission-critical workloads |
| [API & AI Gateway Architecture](mission-critical-saas/api-ai-gateway-architecture.md) | API Management as an AI gateway with rate limiting, routing, and policy enforcement |
| [Load Balancing with APIM](mission-critical-saas/apim-load-balancing.md) | Global load balancing patterns using Azure API Management across multiple regions |
| [Service Bus with Microservices](mission-critical-saas/service-bus-microservices.md) | Event-driven microservices on Azure Container Apps using Azure Service Bus |

## Monitoring

| Document | Description |
|----------|-------------|
| [Application Insights Guide](monitor/application-insights-comprehensive-guide.md) | Comprehensive guide to Application Insights SDK integration, sampling, and alerting |

## Security

| Document | Description |
|----------|-------------|
| [Defender for Storage](mdc/microsoft-defender-for-storage-malware-scanning.md) | Microsoft Defender for Storage malware scanning for Azure Blob Storage |

## Publishing as a Static Site

This repository is published as a documentation site using [MkDocs with the Material theme](https://squidfunk.github.io/mkdocs-material/), deployed automatically to [GitHub Pages](https://calinl.github.io/azure-architecture-guidance/) on every push to `main`.

To preview locally:

```bash
pip install -r requirements.txt
mkdocs serve
```

See `mkdocs.yml` and `.github/workflows/deploy-docs.yml` for configuration.

## License

[MIT](LICENSE)
