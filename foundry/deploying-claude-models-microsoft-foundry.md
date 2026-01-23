# Deploying Claude Models in Microsoft Foundry

## Overview

This guide provides comprehensive instructions for deploying and using Anthropic's Claude models in Microsoft Foundry. Claude models are frontier AI models useful for complex tasks such as coding, agents, financial analysis, research, and office tasks.

Microsoft Foundry supports the following Claude models through **Global Standard deployment**:

| Model | Description | Context Window | Max Output | Input Types |
|-------|-------------|----------------|------------|-------------|
| **Claude Opus 4.5** | Most intelligent model, industry leader for coding, agents, and enterprise workflows | 200K tokens | 64K tokens | Text, Image, Code |
| **Claude Sonnet 4.5** | Balanced performance and cost, ideal for production workflows | 200K tokens | 64K tokens | Text, Image, Code |
| **Claude Haiku 4.5** | Speed and cost optimization, high-volume processing | 200K tokens | 64K tokens | Text, Image |
| **Claude Opus 4.1** | Industry leader for coding, sustained performance on long-running tasks | 200K tokens | **32K tokens** | Text, Image, Code |

> **Note**: All Claude models support **tool calling** (file search and code execution) and return responses in **Text or JSON** formats.
>
> **Supported Languages**: English (`en`), French (`fr`), Arabic (`ar`), Chinese (`zh`), Japanese (`ja`), Korean (`ko`), Spanish (`es`), Hindi (`hi`)

---

## Prerequisites

### 1. Subscription Requirements

#### Eligible Subscription Types

> ⚠️ **Important**: Only **Enterprise Agreement (EA)** and **Microsoft Customer Agreement - Enterprise (MCA-E)** subscriptions are eligible for Claude model usage in Microsoft Foundry.

#### Restricted Subscription Types

The following subscription types **cannot** deploy Claude models:

- Cloud Solution Providers (CSP)
- Sponsored accounts with Azure credits
- Enterprise accounts in Singapore and South Korea
- Microsoft accounts (personal)
- Standard Pay-As-You-Go (without EA/MCA-E)

#### Billing Account Country/Region

Your Azure subscription must have a billing account in a [Microsoft Managed Country/Region](https://learn.microsoft.com/en-us/partner-center/marketplace/tax-details-marketplace#microsoft-managed-countriesregions), **excluding** Belarus and Russia.

### 2. Regional Availability

Claude models are available for deployment in the following regions:

| Region | Deployment Support | Fine-tuning Support |
|--------|-------------------|---------------------|
| **East US 2** | ✅ Yes | ❌ Not available |
| **Sweden Central** | ✅ Yes | ❌ Not available |

> **Important**: Claude models use the **Global Standard** deployment type. Your Microsoft Foundry project or hub must be created in one of these supported regions to deploy the model. However, once deployed, you can consume the endpoint from infrastructure in other regions.
>
> For alternatives if your infrastructure is in a different region, see [Consume serverless APIs from a different hub or project](https://learn.microsoft.com/en-us/azure/ai-foundry/how-to/deploy-models-serverless-connect).

### 3. Azure Marketplace Access

Claude models are partner models that require Azure Marketplace access. You must:

1. Have Azure Marketplace purchases enabled at the billing level
2. Have proper permissions to subscribe to marketplace offerings

### 4. Required Permissions (RBAC)

Your user account must have one of the following roles:

- **Owner**
- **Contributor**
- **Azure AI Developer**

Or a custom role with these specific permissions:

#### Subscription-level permissions (for Marketplace subscription):
```
Microsoft.MarketplaceOrdering/agreements/offers/plans/read
Microsoft.MarketplaceOrdering/agreements/offers/plans/sign/action
Microsoft.MarketplaceOrdering/offerTypes/publishers/offers/plans/agreements/read
Microsoft.Marketplace/offerTypes/publishers/offers/plans/agreements/read
Microsoft.SaaS/register/action
```

#### Resource Group-level permissions (for SaaS resource):
```
Microsoft.SaaS/resources/read
Microsoft.SaaS/resources/write
```

#### Workspace-level permissions (for deployment):
```
Microsoft.MachineLearningServices/workspaces/marketplaceModelSubscriptions/*
Microsoft.MachineLearningServices/workspaces/serverlessEndpoints/*
```

---

## Step 1: Enable Azure Marketplace Purchases

### For Enterprise Agreement (EA) Subscriptions

1. Sign in to the [Azure Portal](https://portal.azure.com)
2. Navigate to **Cost Management + Billing**
3. Select **Billing scopes** in the left menu
4. Select your billing account scope
5. Select **Policies** in the left menu
6. Under **Azure Marketplace**, set the policy to **On**
7. Click **Save**

> **Note**: Only an Enterprise Administrator can enable this setting. Read-only Enterprise Administrators cannot modify this policy.

### For MCA-E Subscriptions

1. Sign in to the [Azure Portal](https://portal.azure.com)
2. Navigate to **Cost Management + Billing**
3. Select **Billing scopes** in the left menu
4. Select the appropriate billing account scope
5. Select **Billing profile** in the left menu
6. Select **Policies** in the left menu
7. Set the **Azure Marketplace** policy to **On** (Free + Paid)
8. Click **Save**

> **Note**: Billing Account Owner/Contributor or Billing Profile Owner/Contributor can enable this setting.

---

## Step 2: Register Marketplace Resource Providers

Ensure your subscription has the required resource providers registered:

1. Go to **Subscriptions** in the Azure Portal
2. Select your subscription
3. Select **Resource providers** in the left menu
4. Filter by "marketplace"
5. Ensure the following providers are **Registered**:
   - `Microsoft.Marketplace`
   - `Microsoft.MarketplaceOrdering`
   - `Microsoft.SaaS`

If any are not registered, select them and click **Register**.

---

## Step 3: Create a Microsoft Foundry Project

1. Navigate to [Microsoft Foundry Portal](https://ai.azure.com)
2. Create a new project in one of the supported regions:
   - **East US 2**
   - **Sweden Central**
3. Ensure the project has appropriate permissions configured

---

## Step 4: Deploy a Claude Model

### Using the Foundry Portal

1. Navigate to your Microsoft Foundry project
2. Go to **Model catalog** or **Deployments**
3. Search for Claude models (e.g., "Claude Sonnet 4.5")
4. Select the model you want to deploy
5. Click **Deploy**
6. Choose **Global Standard** deployment type
7. Provide a deployment name
8. Review and confirm the deployment

### Deployment Types

Claude models use **Global Standard** deployment, which:
- Provides serverless API access
- Uses pay-as-you-go billing
- Does not require dedicated compute resources
- Routes requests globally for optimal performance

---

## Step 5: Verify Quota Allocation

### Default Quota Limits

| Model | Default RPM | Default TPM | EA/MCA-E RPM | EA/MCA-E TPM |
|-------|-------------|-------------|--------------|--------------|
| claude-haiku-4-5 | 0 | 0 | 4,000 | 4,000,000 |
| claude-opus-4-1 | 0 | 0 | 2,000 | 2,000,000 |
| claude-sonnet-4-5 | 0 | 0 | 4,000 | 2,000,000 |
| claude-opus-4-5 | 0 | 0 | 2,000 | 2,000,000 |

> **Note**: Non-EA/MCA-E subscriptions receive **0 quota by default**.

### Viewing Quota

1. Go to [Microsoft Foundry Portal](https://ai.azure.com)
2. Navigate to **Management** > **Quota**
3. Select your subscription and region
4. View available quota for Claude models

### Requesting Quota Increase

If you need additional quota beyond default limits:

1. Submit a request through the [Quota Increase Request Form](https://aka.ms/oai/stuquotarequest)
2. Provide your subscription ID, region, and requested quota
3. Wait for approval (priority given to customers with existing usage)

---

## Working with Claude Models

### Authentication Methods

Claude models support two authentication methods:

1. **Microsoft Entra ID (Recommended)** - Keyless authentication
2. **API Key** - Traditional key-based authentication

### Endpoint Format

```
Base URL: https://<resource-name>.services.ai.azure.com/anthropic
Messages API: https://<resource-name>.services.ai.azure.com/anthropic/v1/messages
```

### Python SDK Example (Entra ID Authentication)

```python
from anthropic import AnthropicFoundry
from azure.identity import DefaultAzureCredential, get_bearer_token_provider

# Configuration
base_url = "https://<resource-name>.services.ai.azure.com/anthropic"
deployment_name = "claude-sonnet-4-5"

# Create token provider for Entra ID authentication
token_provider = get_bearer_token_provider(
    DefaultAzureCredential(), 
    "https://cognitiveservices.azure.com/.default"
)

# Create client
client = AnthropicFoundry(
    azure_ad_token_provider=token_provider,
    base_url=base_url
)

# Send request
message = client.messages.create(
    model=deployment_name,
    messages=[
        {"role": "user", "content": "What is the capital of France?"}
    ],
    max_tokens=1024,
)

print(message.content)
```

### Python SDK Example (API Key Authentication)

```python
from anthropic import AnthropicFoundry

# Configuration
base_url = "https://<resource-name>.services.ai.azure.com/anthropic"
deployment_name = "claude-sonnet-4-5"
api_key = "YOUR_API_KEY"

# Create client
client = AnthropicFoundry(
    api_key=api_key,
    base_url=base_url
)

# Send request
message = client.messages.create(
    model=deployment_name,
    messages=[
        {"role": "user", "content": "What is the capital of France?"}
    ],
    max_tokens=1024,
)

print(message.content)
```

### JavaScript SDK Example

```javascript
import AnthropicFoundry from '@anthropic-ai/foundry-sdk';
import { getBearerTokenProvider, DefaultAzureCredential } from "@azure/identity";

const baseURL = "https://<resource-name>.services.ai.azure.com/anthropic";
const deploymentName = "claude-sonnet-4-5";

// Create token provider for Entra ID authentication
const tokenProvider = getBearerTokenProvider(
    new DefaultAzureCredential(),
    'https://cognitiveservices.azure.com/.default'
);

// Create client
const client = new AnthropicFoundry({
    azureADTokenProvider: tokenProvider,
    baseURL: baseURL,
    apiVersion: "2023-06-01"
});

// Send request
const message = await client.messages.create({
    model: deploymentName,
    messages: [{ role: "user", content: "What is the capital of France?" }],
    max_tokens: 1024,
});

console.log(message);
```

### REST API Example (cURL)

```bash
# Set environment variables
export AZURE_API_KEY="<your-api-key>"
export RESOURCE_NAME="<your-resource-name>"

# Make request
curl -X POST "https://${RESOURCE_NAME}.services.ai.azure.com/anthropic/v1/messages" \
  -H "Content-Type: application/json" \
  -H "x-api-key: ${AZURE_API_KEY}" \
  -H "anthropic-version: 2023-06-01" \
  -d '{
    "messages": [
      {"role": "user", "content": "What is the capital of France?"}
    ],
    "max_tokens": 1024,
    "model": "claude-sonnet-4-5"
  }'
```

---

## Claude Model Capabilities

### Core Capabilities

| Capability | Description |
|------------|-------------|
| **Extended Thinking** | Enhanced reasoning for complex tasks |
| **Image & Text Input** | Process images and return text outputs (vision capabilities for charts, graphs, diagrams) |
| **Code Generation** | Analysis, generation, and debugging (Claude Sonnet 4.5 and Claude Opus models) |
| **Extended Context Window** | Up to 1 million tokens available as an advanced capability (standard is 200K) |
| **PDF Support** | Process and analyze text and visual content from PDF documents |
| **Prompt Caching** | Reduce costs and latency with explicit prompt caching |
| **Citations** | Ground responses in source documents |
| **Context Editing** | Automatically manage conversation context with configurable strategies |
| **Agent Skills** | Extend Claude's capabilities with custom Skills |

### Tools Support

| Tool | Description |
|------|-------------|
| **MCP Connector** | Connect to remote MCP servers directly from the Messages API without a separate MCP client |
| **Memory** | Store and retrieve information across conversations; build knowledge bases over time, maintain project context, and learn from past interactions |
| **Web Fetch** | Retrieve full content from specified web pages and PDF documents for in-depth analysis |
| **File Search** | Search through uploaded files and documents |
| **Code Execution** | Execute code as part of tool calling workflows |

> **Note**: For a full list of supported capabilities and tools, see [Claude's features overview](https://docs.claude.com/en/docs/build-with-claude/overview).

---

## Troubleshooting

### Common Issues and Solutions

| Issue | Cause | Solution |
|-------|-------|----------|
| **0 quota allocated** | Non-EA/MCA-E subscription | Verify subscription type is EA or MCA-E |
| **Marketplace subscription failed** | Marketplace not enabled | Enable Azure Marketplace at billing level |
| **Permission denied** | Missing RBAC permissions | Assign Owner/Contributor/Azure AI Developer role |
| **Region not available** | Project in unsupported region | Create project in East US 2 or Sweden Central |
| **429 Rate Limit** | Exceeded TPM/RPM limits | Implement retry logic with exponential backoff |
| **Resource providers not registered** | Missing Marketplace providers | Register Microsoft.Marketplace, Microsoft.MarketplaceOrdering, Microsoft.SaaS |
| **Cannot access model catalog** | Project not in supported region | Create a new project in East US 2 or Sweden Central |

### Verifying Subscription Type

1. Go to **Azure Portal** > **Cost Management + Billing**
2. Select **Billing scopes**
3. Review your billing account type:
   - **Enterprise Agreement** = EA
   - **Microsoft Customer Agreement** = Check if MCA-E

### Common Error Messages

For subscription-related errors, see [Common error messages and solutions](https://learn.microsoft.com/en-us/marketplace/purchase-saas-offer-in-azure-portal#common-error-messages-and-solutions).

---

## Best Practices

### Model Selection

| Use Case | Recommended Model | Key Strengths |
|----------|-------------------|---------------|
| Complex coding, agents, computer use, enterprise workflows | Claude Opus 4.5 | Most intelligent, 64K output |
| Production workflows, real-world agents, balanced performance/cost | Claude Sonnet 4.5 | Best balance of capability and speed |
| High-volume processing, cost optimization, scaled sub-agents | Claude Haiku 4.5 | Near-frontier performance, best speed/cost |
| Long-running coding tasks, complex reasoning | Claude Opus 4.1 | Sustained performance, thousands of steps |

### Rate Limit Best Practices

1. **Implement retry logic**: Handle 429 responses with exponential backoff
2. **Batch requests**: Combine multiple prompts when possible
3. **Monitor usage**: Track token consumption and request patterns
4. **Use prompt caching**: Enable explicit prompt caching for repeated contexts

### Security Best Practices

1. **Use Entra ID authentication** over API keys when possible
2. **Store API keys securely** in Azure Key Vault
3. **Implement proper RBAC** with least privilege
4. **Enable diagnostic logging** for monitoring

### Cost Optimization

1. **Choose appropriate models**: Use Haiku for simple tasks, Opus for complex ones
2. **Monitor token usage**: Track input/output tokens
3. **Use prompt caching**: Reduce costs for repeated contexts
4. **Batch requests**: Combine when possible

---

## Responsible AI Considerations

When using Claude models:

1. **Content Filtering**: Microsoft Foundry does not provide built-in content filtering for Claude models at deployment time. Configure AI content safety during model inference.

2. **Acceptable Use**: Ensure applications comply with [Anthropic's Acceptable Use Policy](https://www.anthropic.com/legal/aup).

3. **Safety Evaluations**: Review model safety cards:
   - [Claude Opus 4.5 System Card](http://www.anthropic.com/claude-opus-4-5-system-card)
   - [Claude Sonnet 4.5 System Card](https://assets.anthropic.com/m/12f214efcc2f457a/original/Claude-Sonnet-4-5-System-Card.pdf)
   - [Claude Haiku 4.5 System Card](https://assets.anthropic.com/m/99128ddd009bdcb/Claude-Haiku-4-5-System-Card.pdf)
   - [Claude Opus 4.1 System Card](https://assets.anthropic.com/m/4c024b86c698d3d4/original/Claude-4-1-System-Card.pdf)

---

## Agent Integration

### Microsoft Agent Framework

The [Microsoft Agent Framework](https://learn.microsoft.com/en-us/agent-framework/user-guide/agents/agent-types/anthropic-agent) supports creating agents that use Claude models.

```python
# Python example
pip install agent-framework-anthropic --pre

from agent_framework.anthropic import AnthropicClient
```

### Claude Agent SDK

Build custom AI agents with the [Claude Agent SDK](https://docs.claude.com/en/docs/agent-sdk/overview).

---

## References

### Official Microsoft Documentation

1. **Deploy and use Claude models in Microsoft Foundry**
   - https://learn.microsoft.com/en-us/azure/ai-foundry/foundry-models/how-to/use-foundry-models-claude?view=foundry-classic

2. **Region availability for models in serverless APIs**
   - https://learn.microsoft.com/en-us/azure/ai-foundry/how-to/deploy-models-serverless-availability?view=foundry-classic

3. **Deploy models as serverless API deployments**
   - https://learn.microsoft.com/en-us/azure/ai-foundry/how-to/deploy-models-serverless?view=foundry-classic

4. **Configure Marketplace access for model deployments**
   - https://learn.microsoft.com/en-us/azure/ai-foundry/foundry-models/how-to/configure-marketplace?view=foundry-classic

5. **Azure OpenAI in Microsoft Foundry Models quotas and limits**
   - https://learn.microsoft.com/en-us/azure/ai-foundry/openai/quotas-limits?view=foundry-classic

6. **Manage Azure OpenAI quota**
   - https://learn.microsoft.com/en-us/azure/ai-foundry/openai/how-to/quota?view=foundry-classic

7. **Enable marketplace purchases in Azure**
   - https://learn.microsoft.com/en-us/azure/cost-management-billing/manage/enable-marketplace-purchases

8. **Purchase control options in Microsoft Marketplace**
   - https://learn.microsoft.com/en-us/marketplace/purchase-control-options

9. **Role-based access control in Foundry portal**
   - https://learn.microsoft.com/en-us/azure/ai-foundry/concepts/rbac-foundry?view=foundry-classic

10. **Microsoft Managed Countries/Regions for Marketplace**
    - https://learn.microsoft.com/en-us/partner-center/marketplace/tax-details-marketplace#microsoft-managed-countriesregions

### Anthropic Documentation

11. **Claude API Documentation**
    - https://docs.claude.com/en/api/messages

12. **Anthropic SDK (Python)**
    - https://github.com/anthropics/anthropic-sdk-python

13. **Anthropic SDK (TypeScript)**
    - https://github.com/anthropics/anthropic-sdk-typescript

14. **Claude Agent SDK**
    - https://docs.claude.com/en/docs/agent-sdk/overview

15. **Anthropic Acceptable Use Policy**
    - https://www.anthropic.com/legal/aup

### Additional Resources

16. **Quota Increase Request Form**
    - https://aka.ms/oai/stuquotarequest

17. **Microsoft Foundry Portal**
    - https://ai.azure.com

18. **Common Marketplace error messages and solutions**
    - https://learn.microsoft.com/en-us/marketplace/purchase-saas-offer-in-azure-portal#common-error-messages-and-solutions

19. **Microsoft Agent Framework - Anthropic Agents**
    - https://learn.microsoft.com/en-us/agent-framework/user-guide/agents/agent-types/anthropic-agent

20. **Foundry Models from partners and community (Claude capabilities)**
    - https://learn.microsoft.com/en-us/azure/ai-foundry/foundry-models/concepts/models-from-partners?view=foundry-classic#anthropic

21. **Microsoft Foundry feature availability across cloud regions**
    - https://learn.microsoft.com/en-us/azure/ai-foundry/reference/region-support?view=foundry-classic

22. **Consume serverless APIs from a different hub or project**
    - https://learn.microsoft.com/en-us/azure/ai-foundry/how-to/deploy-models-serverless-connect?view=foundry-classic

23. **Claude's features overview (Anthropic)**
    - https://docs.claude.com/en/docs/build-with-claude/overview

24. **Prompt Caching documentation (Anthropic)**
    - https://docs.claude.com/en/docs/build-with-claude/prompt-caching

---

## Document Information

| Field | Value |
|-------|-------|
| **Last Updated** | January 2026 |
| **Applies To** | Microsoft Foundry (Classic and New portals) |
| **Models Covered** | Claude Opus 4.5, Claude Sonnet 4.5, Claude Haiku 4.5, Claude Opus 4.1 |

---

## Changelog

| Date | Change |
|------|--------|
| January 2026 | Initial documentation created |
