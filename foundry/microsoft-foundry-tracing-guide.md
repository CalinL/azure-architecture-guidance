---
title: Microsoft Foundry Tracing and Observability Guide
description: Complete guide to configuring tracing and observability in Microsoft Foundry, covering environment variable flags, SDK instrumentation, portal-based tracing, and framework integrations.
author: calinl
ms.date: 2026-02-25
ms.topic: reference
keywords:
  - microsoft foundry
  - tracing
  - observability
  - opentelemetry
  - azure ai agents
  - application insights
estimated_reading_time: 12
---

## Overview

Tracing in Microsoft Foundry provides visibility into AI application execution. You can use traces to diagnose inaccurate tool calls, misleading prompts, latency bottlenecks, and low-quality evaluation scores. Foundry uses [OpenTelemetry](https://opentelemetry.io/) as the telemetry standard and stores trace data in Azure Application Insights.

There are three distinct approaches to capturing trace data in Foundry, each with its own mechanism for controlling whether input and output message content is recorded. Choosing the correct approach depends on your instrumentation path: portal-only, OpenAI SDK, or Azure AI Agents SDK.

## Prerequisites

Before configuring tracing, ensure you have:

* A Foundry project created in the [Foundry portal](https://ai.azure.com/)
* An Azure Application Insights resource associated with your Foundry resource
* The [Log Analytics Reader role](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/manage-access?tabs=portal#log-analytics-reader) assigned on your Application Insights resource
* Contributor access to the Foundry resource (to connect Application Insights)

## Enabling Tracing in the Foundry Portal

To associate Application Insights with your Foundry project:

1. Navigate to [Foundry portal](https://ai.azure.com/) and open your project.
2. Select **Tracing** in the left navigation bar.
3. Connect an existing Application Insights resource or create a new one.
4. Copy the project endpoint URI from the project overview page for use in SDK-based tracing.

> [!IMPORTANT]
> Using a project endpoint requires configuring Microsoft Entra ID in your application. If you do not have Entra ID configured, use the Application Insights connection string directly (available under **Tracing > Manage data source > Connection string**).

Reference: [Enable tracing in your project](https://learn.microsoft.com/en-us/azure/ai-foundry/how-to/develop/trace-application?view=foundry-classic#enable-tracing-in-your-project)

## Content Recording Flags

By default, Foundry tracing captures metadata (model name, token counts, latency, status) but **does not** record actual message content (prompts and completions). Three different mechanisms control content recording depending on the instrumentation path.

### Summary Table

| Environment Variable                                  | Instrumentation Path                  | Package                                    | Default |
|-------------------------------------------------------|---------------------------------------|--------------------------------------------|---------|
| `OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT`  | OpenAI SDK via `OpenAIInstrumentor`   | `opentelemetry-instrumentation-openai-v2`  | `false` |
| `AZURE_TRACING_GEN_AI_CONTENT_RECORDING_ENABLED`      | Azure AI Agents SDK                        | `azure-ai-agents` + Azure SDK tracing | `false` |
| (No flag required)                                    | Foundry portal Thread logs            | N/A (server-side)                          | Always captured |

### Flag 1: OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT

This flag controls content recording when you instrument the **OpenAI SDK** directly using the `opentelemetry-instrumentation-openai-v2` package and `OpenAIInstrumentor`.

Set the environment variable before running your application:

```bash
# Bash
export OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT=true
```

```powershell
# PowerShell
setx OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT true
```

When to use this flag:

* You call models via `project_client.get_openai_client()` or the `openai` Python package directly.
* You instrument with `OpenAIInstrumentor().instrument()`.
* You want to see the actual prompt and completion text in the **Tracing** tab of the Foundry portal or in Application Insights.

Reference: [Instrument the OpenAI SDK, Step 2](https://learn.microsoft.com/en-us/azure/ai-foundry/how-to/develop/trace-application?view=foundry-classic#instrument-the-openai-sdk)

### Flag 2: AZURE_TRACING_GEN_AI_CONTENT_RECORDING_ENABLED

This flag controls content recording when you use the **Azure AI Agents SDK** (`azure-ai-agents`). The flag is read by the Azure SDK tracing layer ([`azure-core-tracing-opentelemetry`](https://learn.microsoft.com/en-us/python/api/overview/azure/core-tracing-opentelemetry-readme)).

Set the environment variable before running your application:

```python
import os
os.environ["AZURE_TRACING_GEN_AI_CONTENT_RECORDING_ENABLED"] = "true"  # False by default
```

Or through the shell:

```bash
# Bash
export AZURE_TRACING_GEN_AI_CONTENT_RECORDING_ENABLED=true
```

When to use this flag:

* You build agents using `project_client.agents.*` methods.
* You export traces to Azure Monitor (via `configure_azure_monitor()`) or to a local exporter (via `AIAgentsInstrumentor().instrument()`).
* You want to see the content of chat messages in traces sent to Application Insights.

> [!CAUTION]
> Recorded content might contain personal data. Review your organization's data handling policies before enabling content recording in production.

Reference: [Instrument tracing in your code](https://learn.microsoft.com/en-us/azure/ai-foundry/how-to/develop/trace-agents-sdk?view=foundry-classic#instrument-tracing-in-your-code)

### Portal Thread Logs (No Flag Required)

When you use the **Agents playground** in the Foundry portal, input and output messages are captured server-side by the Agents service. No environment variable or SDK instrumentation is required.

To view thread results:

1. Navigate to the Agents playground in the Foundry portal.
2. Open an active thread and select **Thread logs**.
3. Review thread details, run information, ordered run steps, tool calls, and inputs/outputs.

This approach reads directly from the Agents service internal logging (thread and run data), not from OpenTelemetry traces.

Reference: [View thread results in the Foundry Agents playground](https://learn.microsoft.com/en-us/azure/ai-foundry/how-to/develop/trace-agents-sdk?view=foundry-classic#view-thread-results-in-the-foundry-agents-playground)

## Instrumentation Setup by Path

### Path A: OpenAI SDK with OpenTelemetry

Install the required packages:

```bash
pip install azure-ai-projects azure-monitor-opentelemetry opentelemetry-instrumentation-openai-v2
```

Configure and instrument:

```python
import os
from azure.ai.projects import AIProjectClient
from azure.identity import DefaultAzureCredential
from azure.monitor.opentelemetry import configure_azure_monitor
from opentelemetry.instrumentation.openai_v2 import OpenAIInstrumentor

# Optional: enable content recording
os.environ["OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT"] = "true"

# Connect to your project
project_client = AIProjectClient(
    credential=DefaultAzureCredential(),
    endpoint=os.environ["PROJECT_ENDPOINT"],
)

# Configure Azure Monitor
connection_string = project_client.telemetry.get_application_insights_connection_string()
configure_azure_monitor(connection_string=connection_string)

# Instrument OpenAI SDK
OpenAIInstrumentor().instrument()

# Use the client as usual
client = project_client.get_openai_client()
response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[{"role": "user", "content": "Write a short poem on open telemetry."}],
)
print(response.choices[0].message.content)
```

Reference: [Trace application with OpenAI SDK](https://learn.microsoft.com/en-us/azure/ai-foundry/how-to/develop/trace-application?view=foundry-classic)

### Path B: Azure AI Agents SDK

Install the required packages:

```bash
pip install azure-ai-projects azure-identity
pip install azure-monitor-opentelemetry opentelemetry-sdk
```

> [!NOTE]
> The `azure-ai-agents` package is installed automatically as a dependency of `azure-ai-projects`.

Configure and instrument:

```python
import os
from azure.ai.projects import AIProjectClient
from azure.identity import DefaultAzureCredential
from azure.monitor.opentelemetry import configure_azure_monitor
from opentelemetry import trace

# Enable content recording
os.environ["AZURE_TRACING_GEN_AI_CONTENT_RECORDING_ENABLED"] = "true"

# Connect to your project
project_client = AIProjectClient(
    credential=DefaultAzureCredential(),
    endpoint=os.environ["PROJECT_ENDPOINT"],
)

# Configure Azure Monitor (handles instrumentation of Azure SDK calls automatically)
connection_string = project_client.telemetry.get_application_insights_connection_string()
configure_azure_monitor(connection_string=connection_string)

# Trace your agent workflow
tracer = trace.get_tracer(__name__)
with tracer.start_as_current_span("my-agent-workflow"):
    agent = project_client.agents.create_agent(
        model=os.environ["MODEL_DEPLOYMENT_NAME"],
        name="my-assistant",
        instructions="You are a helpful assistant",
    )
    thread = project_client.agents.threads.create()
    message = project_client.agents.messages.create(
        thread_id=thread.id, role="user", content="Tell me a joke",
    )
    run = project_client.agents.runs.create_and_process(
        thread_id=thread.id, agent_id=agent.id,
    )
```

> [!IMPORTANT]
> When using `configure_azure_monitor()`, you do **not** need to call `AIAgentsInstrumentor().instrument()` explicitly. Azure Monitor handles instrumentation of Azure SDK calls automatically. The `AIAgentsInstrumentor` is only needed when configuring tracing manually (for example, console or OTLP export without Azure Monitor).

Reference: [Instrument tracing in your code](https://learn.microsoft.com/en-us/azure/ai-foundry/how-to/develop/trace-agents-sdk?view=foundry-classic#instrument-tracing-in-your-code)

### Path C: Console and Local Tracing

For local debugging without cloud access, you can export traces to the console or use a local OTLP-compatible viewer such as [Aspire Dashboard](https://aspiredashboard.com/#start).

#### Console tracing with OpenAI SDK

```python
from opentelemetry.instrumentation.openai_v2 import OpenAIInstrumentor

OpenAIInstrumentor().instrument()

from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import SimpleSpanProcessor, ConsoleSpanExporter

span_exporter = ConsoleSpanExporter()
tracer_provider = TracerProvider()
tracer_provider.add_span_processor(SimpleSpanProcessor(span_exporter))
trace.set_tracer_provider(tracer_provider)
```

Reference: [Trace to console](https://learn.microsoft.com/en-us/azure/ai-foundry/how-to/develop/trace-application?view=foundry-classic#trace-to-console)

#### Local tracing with Azure AI Agents SDK

When tracing agents locally (without Azure Monitor), you must install the tracing plugin, enable it explicitly, and call `AIAgentsInstrumentor`:

```bash
pip install azure-core-tracing-opentelemetry opentelemetry-exporter-otlp opentelemetry-sdk
```

```python
from azure.core.settings import settings
settings.tracing_implementation = "opentelemetry"

from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import SimpleSpanProcessor, ConsoleSpanExporter

span_exporter = ConsoleSpanExporter()
tracer_provider = TracerProvider()
tracer_provider.add_span_processor(SimpleSpanProcessor(span_exporter))
trace.set_tracer_provider(tracer_provider)

from azure.ai.agents.telemetry import AIAgentsInstrumentor
AIAgentsInstrumentor().instrument()
```

> [!NOTE]
> The `settings.tracing_implementation = "opentelemetry"` line and the explicit `AIAgentsInstrumentor().instrument()` call are required for local tracing because there is no `configure_azure_monitor()` to handle them automatically.

Reference: [Log traces locally](https://learn.microsoft.com/en-us/azure/ai-foundry/how-to/develop/trace-agents-sdk?view=foundry-classic#log-traces-locally)

#### AI Toolkit for VS Code

You can also use **AI Toolkit for VS Code** for local tracing with a built-in OTLP-compatible collector. See [Tracing in AI Toolkit](https://code.visualstudio.com/docs/intelligentapps/tracing) for setup instructions.

## Framework Integrations

Foundry supports tracing integrations with several agent frameworks. Each framework has its own content recording mechanism.

### LangChain and LangGraph

Install the `langchain-azure-ai` package and use the `AzureAIOpenTelemetryTracer`:

```python
from langchain_azure_ai.callbacks.tracers import AzureAIOpenTelemetryTracer

azure_tracer = AzureAIOpenTelemetryTracer(
    connection_string=os.environ.get("APPLICATION_INSIGHTS_CONNECTION_STRING"),
    enable_content_recording=True,  # Controls content capture for this tracer
    name="My LangChain Agent",
)
```

Content recording is controlled by the `enable_content_recording` parameter on the tracer instance rather than an environment variable.

Reference: [Enable tracing for agents built on LangChain and LangGraph](https://learn.microsoft.com/en-us/azure/ai-foundry/how-to/develop/trace-agents-sdk?view=foundry-classic#enable-tracing-for-agents-built-on-langchain-and-langgraph)

### Semantic Kernel and Microsoft Agent Framework

Agents built on Semantic Kernel and Microsoft Agent Framework get out-of-the-box tracing in Foundry with no additional instrumentation required.

* [Semantic Kernel observability](https://learn.microsoft.com/en-us/semantic-kernel/concepts/enterprise-readiness/observability)
* [Microsoft Agent Framework observability](https://learn.microsoft.com/en-us/agent-framework/user-guide/workflows/observability)

### OpenAI Agents SDK

Use `OpenAIAgentsInstrumentor` to instrument the OpenAI Agents SDK. The following example exports to Azure Monitor if `APPLICATION_INSIGHTS_CONNECTION_STRING` is set, otherwise falls back to the console:

```python
import os
from opentelemetry import trace
from opentelemetry.instrumentation.openai_agents import OpenAIAgentsInstrumentor
from opentelemetry.sdk.resources import Resource
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor, ConsoleSpanExporter

resource = Resource.create({
    "service.name": os.getenv("OTEL_SERVICE_NAME", "openai-agents-app"),
})
provider = TracerProvider(resource=resource)

conn = os.getenv("APPLICATION_INSIGHTS_CONNECTION_STRING")
if conn:
    from azure.monitor.opentelemetry.exporter import AzureMonitorTraceExporter
    provider.add_span_processor(
        BatchSpanProcessor(AzureMonitorTraceExporter.from_connection_string(conn))
    )
else:
    provider.add_span_processor(BatchSpanProcessor(ConsoleSpanExporter()))

trace.set_tracer_provider(provider)

OpenAIAgentsInstrumentor().instrument(tracer_provider=trace.get_tracer_provider())
```

Reference: [Enable tracing for agents built on OpenAI Agents SDK](https://learn.microsoft.com/en-us/azure/ai-foundry/how-to/develop/trace-agents-sdk?view=foundry-classic#enable-tracing-for-agents-built-on-openai-agents-sdk)

## Custom Spans and Attributes

You can extend tracing with custom spans and attributes using the OpenTelemetry SDK:

```python
from opentelemetry import trace

tracer = trace.get_tracer(__name__)

@tracer.start_as_current_span("assess_claims_with_context")
def assess_claims_with_context(claims, contexts):
    current_span = trace.get_current_span()
    current_span.set_attribute("operation.claims_count", len(claims))
    # Your logic here
```

Reference: [Trace custom functions](https://learn.microsoft.com/en-us/azure/ai-foundry/how-to/develop/trace-agents-sdk?view=foundry-classic#trace-custom-functions)

## Service Name Configuration

To identify your application in Application Insights when multiple applications log to the same resource, set the `OTEL_SERVICE_NAME` environment variable:

```bash
export OTEL_SERVICE_NAME="my-ai-application"
```

You can then filter traces in the portal by `Application` or query with:

```sql
| where cloud_RoleName == "my-ai-application"
```

Reference: [Using service name in trace data](https://learn.microsoft.com/en-us/azure/ai-foundry/how-to/develop/trace-agents-sdk?view=foundry-classic#using-service-name-in-trace-data)

## Recommendations

### Content Recording

* Enable content recording flags only in development and debugging environments. Disable them in production to avoid logging sensitive user data.
* If content recording is required in production (for quality monitoring or evaluation), ensure compliance with your organization's data handling and privacy policies.
* Use the correct flag for your instrumentation path. Setting the wrong variable has no effect.

### Instrumentation Strategy

* Use `OpenAIInstrumentor` when calling models directly through the OpenAI SDK.
* When using `configure_azure_monitor()` with Azure AI Agents SDK, Azure Monitor handles instrumentation automatically.
* Use `AIAgentsInstrumentor` explicitly only when configuring local/console tracing without Azure Monitor.
* For LangChain/LangGraph, use the `enable_content_recording` parameter on `AzureAIOpenTelemetryTracer` rather than environment variables.

### Operational Practices

* Set `OTEL_SERVICE_NAME` for each application to differentiate traces when multiple applications share the same Application Insights resource.
* Use `tracer.start_as_current_span()` decorators to capture custom business logic alongside model calls within the same trace.
* Add meaningful span attributes (such as claim counts, batch sizes, or request identifiers) to enrich trace data for analysis.
* Redact sensitive content and avoid storing secrets in span attributes.
* Use consistent span attributes across your applications to simplify querying.

### Access Control

* Assign the **Log Analytics Reader** role to users who need to view traces in the Foundry portal.
* Assign the **Contributor** role on the Foundry resource to users who need to connect or create Application Insights resources.
* Use [Microsoft Entra groups](https://learn.microsoft.com/en-us/azure/ai-foundry/concepts/rbac-foundry?view=foundry-classic#use-microsoft-entra-groups-with-foundry) to manage access at scale.

## Complete Environment Variables Reference

| Variable                                                | Purpose                                      | Values         |
|---------------------------------------------------------|----------------------------------------------|----------------|
| `OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT`    | Record input/output for OpenAI SDK traces    | `true`/`false` |
| `AZURE_TRACING_GEN_AI_CONTENT_RECORDING_ENABLED`        | Record input/output for Azure Agents traces  | `true`/`false` |
| `OTEL_SERVICE_NAME`                                     | Set application identity in traces           | Any string     |
| `APPLICATION_INSIGHTS_CONNECTION_STRING`                 | Application Insights connection string       | Connection string |
| `PROJECT_ENDPOINT`                                      | Foundry project endpoint URI                 | URL            |
| `MODEL_DEPLOYMENT_NAME`                                 | AI model deployment name                     | Deployment name |

## Using Datadog Instead of Azure Monitor

If you use Datadog for observability, there are two approaches for capturing LLM input/output content. Choose one to avoid duplicate spans.

### Option 1: OTEL Instrumentors with OTLP Export to Datadog

Use the same OpenTelemetry instrumentors documented in this guide and export traces to Datadog. The content recording flags (`OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT`, `AZURE_TRACING_GEN_AI_CONTENT_RECORDING_ENABLED`) work as described because they control what the instrumentor captures, not where traces are sent.

| Flag | Effect with Datadog |
|---|---|
| `OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT=true` | Records prompt/completion content into span events; Datadog receives them via OTLP |
| `AZURE_TRACING_GEN_AI_CONTENT_RECORDING_ENABLED=true` | Records Azure AI Agents message content into spans; Datadog receives them via OTLP |

Datadog supports two OTLP ingestion paths:

**Direct to Datadog LLM Observability (no Agent required):**

Replace `configure_azure_monitor()` with an OTLP exporter pointing directly to the Datadog OTLP endpoint. Set the required environment variables:

```bash
export OTEL_EXPORTER_OTLP_TRACES_PROTOCOL=http/protobuf
export OTEL_EXPORTER_OTLP_TRACES_ENDPOINT=https://otlp.datadoghq.com/v1/traces
export OTEL_EXPORTER_OTLP_TRACES_HEADERS="dd-api-key=<YOUR_DATADOG_API_KEY>,dd-otlp-source=llmobs"
```

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.resources import Resource, SERVICE_NAME

resource = Resource(attributes={SERVICE_NAME: "my-ai-app"})
provider = TracerProvider(resource=resource)
provider.add_span_processor(BatchSpanProcessor(OTLPSpanExporter()))
trace.set_tracer_provider(provider)
```

**Via Datadog Agent OTLP receiver:**

If you already run a local [Datadog Agent with OTLP ingestion enabled](https://docs.datadoghq.com/opentelemetry/setup/otlp_ingest_in_the_agent/), point your OTLP exporter to the Agent instead:

```python
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter

provider = TracerProvider()
provider.add_span_processor(
    BatchSpanProcessor(OTLPSpanExporter(endpoint="http://localhost:4317"))
)
```

Reference: [Datadog OpenTelemetry LLM Observability Instrumentation](https://docs.datadoghq.com/llm_observability/instrumentation/otel_instrumentation)

### Option 2: Datadog Native LLM Observability

Use Datadog's `ddtrace` auto-instrumentation instead of OpenTelemetry instrumentors. In this case, the OTEL content recording flags have **no effect**. Content capture is handled automatically by `ddtrace` for [supported integrations](https://docs.datadoghq.com/llm_observability/instrumentation/auto_instrumentation/) (OpenAI, LangChain, Anthropic, and others).

Run your application with `ddtrace-run` and the required environment variables:

```bash
DD_LLMOBS_ENABLED=1 \
DD_LLMOBS_ML_APP="my-ai-app" \
DD_API_KEY="<your-datadog-api-key>" \
ddtrace-run python app.py
```

To send data directly to Datadog without a local Datadog Agent, add `DD_LLMOBS_AGENTLESS_ENABLED=1`.

Traces go to [Datadog LLM Observability](https://docs.datadoghq.com/llm_observability/), a dedicated product separate from standard APM traces.

Reference: [Datadog LLM Observability Quickstart](https://docs.datadoghq.com/llm_observability/quickstart/)

> [!NOTE]
> The two approaches are independent. If you already use `ddtrace` for APM, adding OTEL instrumentors will produce duplicate spans. Pick one instrumentation path per application.

## References

* [View trace results for AI applications using OpenAI SDK](https://learn.microsoft.com/en-us/azure/ai-foundry/how-to/develop/trace-application?view=foundry-classic)
* [Trace and observe AI agents in Microsoft Foundry](https://learn.microsoft.com/en-us/azure/ai-foundry/how-to/develop/trace-agents-sdk?view=foundry-classic)
* [Azure AI Projects client library for Python](https://learn.microsoft.com/en-us/python/api/overview/azure/ai-projects-readme?view=azure-python)
* [Azure Core Tracing OpenTelemetry client library for Python](https://learn.microsoft.com/en-us/python/api/overview/azure/core-tracing-opentelemetry-readme)
* [OpenTelemetry Semantic Conventions for GenAI](https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-agent-spans/)
* [Tracing in AI Toolkit for VS Code](https://code.visualstudio.com/docs/intelligentapps/tracing)
* [Semantic Kernel Observability](https://learn.microsoft.com/en-us/semantic-kernel/concepts/enterprise-readiness/observability)
* [Microsoft Agent Framework Observability](https://learn.microsoft.com/en-us/agent-framework/user-guide/workflows/observability)
* [Azure Monitor OpenTelemetry](https://learn.microsoft.com/en-us/azure/azure-monitor/app/opentelemetry-enable)
* [Datadog OTLP Ingestion by the Agent](https://docs.datadoghq.com/opentelemetry/setup/otlp_ingest_in_the_agent/)
* [Datadog LLM Observability](https://docs.datadoghq.com/llm_observability/)
* [Datadog OpenTelemetry LLM Observability Instrumentation](https://docs.datadoghq.com/llm_observability/instrumentation/otel_instrumentation)
* [Datadog LLM Observability Quickstart](https://docs.datadoghq.com/llm_observability/quickstart/)
