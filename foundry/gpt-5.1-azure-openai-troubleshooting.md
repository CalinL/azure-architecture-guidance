# GPT-5.1 on Azure OpenAI — 500 Errors Troubleshooting Guide

| | |
|---|---|
| **Research Date** | April 28, 2026 |
| **Created by** | Claude Opus 4.6 (1M context) |
| **Reviewed by** | Claude Opus 4.7, Claude Sonnet 4.6, Claude Haiku 4.5, GPT-5.3 Codex |
| **Sources** | Microsoft Learn official documentation |

> **Applies to:** Azure OpenAI resources accessed via both **Microsoft Foundry (classic)** and the **new Foundry portal**. The v1 API endpoint and guidance below are identical for both portals — the underlying Azure OpenAI resource is the same.

## Problem

Calls to Azure OpenAI using model `gpt-5.1` with `api_version="2025-04-01-preview"` return **HTTP 500 (Internal Server Error)** responses.

## Root Cause

`gpt-5.1` (model version `2025-11-13`) was released **7 months after** the `2025-04-01-preview` API version. That API version does not support GPT-5.1. Using an incompatible API version with a newer model can result in server-side errors, including the HTTP 500 responses observed in this case.

Starting in August 2025, Azure OpenAI introduced the **v1 API**, which eliminates the need for dated `api-version` parameters entirely. GPT-5 series models require this v1 API.

> **Note:** Microsoft has announced that the **Azure AI Inference SDK is deprecated** and will be retired on August 26, 2026. The recommended path is to migrate to the OpenAI v1 API using the standard `openai` Python package. See the [migration guide](https://learn.microsoft.com/azure/foundry/how-to/model-inference-to-openai-migration).

## Solution — Migrate to the v1 API

> **Important:** With the v1 API, you use `OpenAI()` (not `AzureOpenAI()`) as the client class, and pass your Azure endpoint as `base_url`. The `model` parameter must match your **deployment name** (not the model name) if they differ. The endpoint format `https://YOUR-RESOURCE-NAME.openai.azure.com/openai/v1/` works for both Foundry (classic) and the new Foundry portal. The alternate format `https://YOUR-RESOURCE-NAME.services.ai.azure.com/openai/v1/` is also accepted.

### Python — API Key Authentication

```python
import os
from openai import OpenAI

client = OpenAI(
    api_key=os.getenv("AZURE_OPENAI_API_KEY"),
    base_url="https://YOUR-RESOURCE-NAME.openai.azure.com/openai/v1/"
)

# Using the Responses API (recommended for GPT-5 series)
response = client.responses.create(
    model="gpt-5.1",  # Use your deployment name
    input="Explain quantum computing in simple terms.",
    reasoning={"effort": "medium"}
)

print(response.output_text)
```

### Python — Microsoft Entra ID Authentication

```python
from openai import OpenAI
from azure.identity import DefaultAzureCredential, get_bearer_token_provider

token_provider = get_bearer_token_provider(
    DefaultAzureCredential(), "https://ai.azure.com/.default"
)

client = OpenAI(
    base_url="https://YOUR-RESOURCE-NAME.openai.azure.com/openai/v1/",
    api_key=token_provider
)

response = client.responses.create(
    model="gpt-5.1",
    input="Explain quantum computing in simple terms.",
    reasoning={"effort": "medium"}
)

print(response.output_text)
```

### Python — Environment Variables (Simplest)

Set these environment variables:

| Variable           | Value                                                          |
| ------------------ | -------------------------------------------------------------- |
| `OPENAI_BASE_URL`  | `https://YOUR-RESOURCE-NAME.openai.azure.com/openai/v1/`      |
| `OPENAI_API_KEY`   | Your Azure OpenAI API key                                      |

Then the client requires no parameters:

```python
from openai import OpenAI

client = OpenAI()
```

### REST / cURL

```bash
curl -X POST https://YOUR-RESOURCE-NAME.openai.azure.com/openai/v1/responses \
  -H "Content-Type: application/json" \
  -H "api-key: $AZURE_OPENAI_API_KEY" \
  -d '{
    "model": "gpt-5.1",
    "input": "Explain quantum computing in simple terms."
  }'
```

## Key Changes from the Legacy API

| Aspect                | Legacy (dated api-version)                 | v1 API                                                  |
| --------------------- | ------------------------------------------ | ------------------------------------------------------- |
| **Client class**      | `AzureOpenAI()`                            | `OpenAI()`                                              |
| **Endpoint**          | `azure_endpoint=...`                       | `base_url="https://...openai.azure.com/openai/v1/"`     |
| **API version**       | `api_version="2025-04-01-preview"` (required) | Not needed                                           |
| **Token refresh**     | Handled by `AzureOpenAI()` client          | Built into `OpenAI()` client via `api_key=token_provider` |

## GPT-5.1 Model Variants

| Model                 | Version      | Chat Completions API | Responses API | Context Window                               |
| --------------------- | ------------ | -------------------- | ------------- | -------------------------------------------- |
| `gpt-5.1`             | 2025-11-13   | ✅                   | ✅            | 400K (272,000 input / 128,000 output)        |
| `gpt-5.1-chat` *(Preview)* | 2025-11-13 | ✅                 | ✅            | 128K (111,616 input / 16,384 output)         |
| `gpt-5.1-codex`       | 2025-11-13   | ❌                   | ✅            | 400K (272,000 input / 128,000 output)        |
| `gpt-5.1-codex-mini`  | 2025-11-13   | ❌                   | ✅            | 400K (272,000 input / 128,000 output)        |
| `gpt-5.1-codex-max`   | 2025-12-04   | ❌                   | ✅            | 400K (272,000 input / 128,000 output)        |

> ⚠️ **`gpt-5.1-chat` is a Preview model.** Microsoft does not recommend using preview models in production. Preview models may be upgraded to future versions without following the standard model lifecycle.

## Important: `reasoning_effort` Defaults to `none`

Unlike earlier reasoning models, **`gpt-5.1` defaults `reasoning_effort` to `none`**. If you expect the model to reason through problems, you must explicitly set it:

```python
response = client.responses.create(
    model="gpt-5.1",
    input="Solve this step by step: what is 127 * 83?",
    reasoning={"effort": "medium"}  # Options: none, low, medium, high
)
```

> **Note:** `gpt-5.1-codex-max` additionally supports `xhigh` (also available on later GPT-5 series models like `gpt-5.2+`). The `minimal` level is **not** supported on `gpt-5.1` or later — it is only available on the original GPT-5 models (`gpt-5`, `gpt-5-mini`, `gpt-5-nano`).

## Package Requirements

```bash
pip install --upgrade openai
```

For Microsoft Entra ID authentication:

```bash
pip install azure-identity
```

The `openai` Python package must be **>=1.0**. The latest version is recommended.

## Diagnosing Errors via Log Analytics

Errors from Azure OpenAI (Cognitive Services) are logged in the **`AzureDiagnostics`** table. Ensure you have a **Diagnostic Setting** on your Cognitive Services resource routing at least the `RequestResponse` category to your Log Analytics workspace.

### KQL — Find 500 errors

```kql
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.COGNITIVESERVICES"
| where ResultSignature == "500"
| where TimeGenerated > ago(24h)
| project TimeGenerated, OperationName, DurationMs,
          ResultSignature, properties_s, _ResourceId
| order by TimeGenerated desc
```

### KQL — Error breakdown by status code

```kql
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.COGNITIVESERVICES"
| where ResultSignature !in ("200", "201")
| where TimeGenerated > ago(24h)
| summarize ErrorCount = count() by ResultSignature, OperationName
| order by ErrorCount desc
```

### KQL — Filter by model

```kql
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.COGNITIVESERVICES"
| where ResultSignature !in ("200", "201")
| where properties_s has "gpt-5.1"
| where TimeGenerated > ago(7d)
| project TimeGenerated, OperationName, ResultSignature,
          DurationMs, properties_s
| order by TimeGenerated desc
```

## Additional Known Issues

### Global Standard deployment latency and transient errors

**Global Standard** deployments (`GlobalStandard` SKU) dynamically route traffic to the data center with the best availability worldwide. While this provides the highest default quota and best model availability, customers with **high sustained usage** may experience greater latency variability. Under heavy load, this can result in transient 500 errors, timeouts (408), or gateway errors (502/504).

The `openai` Python SDK automatically retries 500+ errors **twice** with exponential backoff. If you are seeing persistent 500 errors (not transient), consider:

- **Provisioned deployments** — reserved capacity with predictable latency, no dynamic routing
- **Data Zone Standard** — limits routing to your data zone (US or EU) for lower variance
- **Implementing retry logic** with `max_retries`:

```python
client = OpenAI(
    base_url="https://YOUR-RESOURCE-NAME.openai.azure.com/openai/v1/",
    api_key=os.getenv("AZURE_OPENAI_API_KEY"),
    max_retries=5  # default is 2
)
```

> **Tip:** Microsoft's FAQ also documents known 500 errors for "Unexpected special token" and "invalid Unicode output" — both mitigated by reducing `temperature` below 1.0 and using retry logic.

### `2025-04-01-preview` and Azure API Management

The `2025-04-01-preview` API spec uses OpenAPI 3.1, which is **not fully supported by Azure API Management (APIM)**. If you route requests through APIM, this can also cause failures independent of the model used.

## References

- [Azure OpenAI v1 API lifecycle](https://learn.microsoft.com/azure/foundry/openai/api-version-lifecycle)
- [GPT-5 series reasoning models](https://learn.microsoft.com/azure/foundry/openai/how-to/reasoning)
- [Responses API guide](https://learn.microsoft.com/azure/foundry/openai/how-to/responses)
- [GPT-5.1 model details](https://learn.microsoft.com/azure/foundry/foundry-models/concepts/models-sold-directly-by-azure#gpt-51)
- [GPT-5.1 on Foundry classic (Gov)](https://learn.microsoft.com/azure/foundry-classic/openai/azure-government#gpt-51)
- [Switching between OpenAI and Azure OpenAI endpoints](https://learn.microsoft.com/azure/foundry-classic/openai/how-to/switching-endpoints)
- [Migrate from Azure AI Inference SDK to OpenAI SDK](https://learn.microsoft.com/azure/foundry/how-to/model-inference-to-openai-migration)
- [Python SDK supported languages](https://learn.microsoft.com/azure/foundry/openai/supported-languages)
