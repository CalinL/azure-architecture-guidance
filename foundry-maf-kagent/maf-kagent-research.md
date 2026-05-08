# kagent vs. Microsoft Agent Framework (MAF)
## A Comparative Research Focused on Microsoft Foundry / Azure Integration

> **Document date:** 2026-05-08
>
> **About this document:** This is a research summary comparing two open-source projects in the AI agent ecosystem. It was prepared with AI assistance under human guidance and is grounded in publicly available documentation (kagent.dev, the kagent GitHub repository, Microsoft Learn for Agent Framework and Foundry Agent Service, the Microsoft Agent Framework GitHub repository, the official MAF 1.0 GA announcement, the Foundry Agent Service Transparency Note, Microsoft Entra Agent ID documentation, and NuGet package metadata — all listed in §12). Both projects evolve quickly; verify version numbers, package names, and feature availability against the linked primary sources before making procurement, architecture, or licensing decisions.
>
> Scope: Compare **[kagent](https://kagent.dev/)** ([github.com/kagent-dev/kagent](https://github.com/kagent-dev/kagent)) and **[Microsoft Agent Framework (MAF)](https://learn.microsoft.com/en-us/agent-framework/)** ([github.com/microsoft/agent-framework](https://github.com/microsoft/agent-framework)) on how each integrates with the Microsoft Foundry / Azure ecosystem, with a particular focus on Foundry Agent Service interoperability, gaps, .NET support, and whether kagent can serve as a deployment substrate for MAF agents.

---

## Table of Contents

1. [TL;DR](#1-tldr)
2. [kagent — overview](#2-kagent-overview)
3. [Microsoft Agent Framework (MAF) — overview](#3-microsoft-agent-framework-maf-overview)
4. [Microsoft Foundry / Azure ecosystem integration](#4-microsoft-foundry-azure-ecosystem-integration)
5. [Platform vs. platform — kagent vs. Foundry Agent Service](#5-platform-vs-platform-kagent-vs-foundry-agent-service)
6. [What you give up by not using Foundry Agent Service (when running MAF standalone)](#6-what-you-give-up-by-not-using-foundry-agent-service-when-running-maf-standalone)
7. [Can you use MAF + kagent? (Build with MAF, deploy on kagent)](#7-can-you-use-maf-kagent-build-with-maf-deploy-on-kagent)
8. [Wiring kagent agents into Agent 365 / Microsoft Entra Agent ID](#8-wiring-kagent-agents-into-agent-365-microsoft-entra-agent-id)
9. [.NET support comparison](#9-net-support-comparison)
10. [Side-by-side cheat sheet](#10-side-by-side-cheat-sheet)
11. [Recommendation matrix](#11-recommendation-matrix)
12. [Sources](#12-sources)

---

## 1. TL;DR

| Dimension | **kagent** | **Microsoft Agent Framework (MAF)** |
|---|---|---|
| Primary identity | **Kubernetes-native runtime / control plane** for agents | **SDK / framework** for building agents and multi-agent workflows |
| Owner / governance | **CNCF Sandbox** project (kagent-dev, founded by Istio creators / Solo.io) | **Microsoft** (open-source on GitHub, **1.0 GA on April 3, 2026**) |
| Runtime languages (engine) | **Go** and **Python** (ADK-based) engines; agents declared as CRDs | **.NET** (`Microsoft.Agents.AI`) and **Python** (`agent-framework`) |
| Where it shines | Day-2 ops on Kubernetes: GitOps, RBAC, mTLS, OTel, multi-tenancy | Building agents and graph-based workflows, with deep Microsoft Foundry / Azure OpenAI / OpenAI / Anthropic / Ollama bindings |
| Foundry integration | Indirect — via Azure OpenAI `ModelConfig` (no first-class Foundry Agent Service binding) | First-class — `Microsoft.Agents.AI.Foundry`, `FoundryChatClient`, `AIProjectClient.AsAIAgent(...)`, **Foundry Hosted Agents** (preview) |
| Best mental model | "Kubernetes for AI agents" | "ASP.NET Core / FastAPI for AI agents" |

**They are not direct competitors.** MAF is a *framework* for writing agents; kagent is a *platform* for running them. The interesting question is whether you stack them (build with MAF, deploy on kagent) — see §7.

---

## 2. kagent — overview

### 2.1 What it is
- Kubernetes-native agent runtime. Agents, model configs, MCP tool servers, and sessions are all **Kubernetes Custom Resources (CRDs)** managed by a controller.
- Four core components:
  - **Controller** — reconciles CRDs into running pods.
  - **Engine** — runs your agents. Native runtimes are **Go and Python** (the Python runtime uses Google ADK). LangGraph, CrewAI, ADK, or your own framework can be brought in via the BYO model.
  - **UI** — web dashboard for managing agents/tools.
  - **CLI** — `kagent` for install/invoke/dashboard.
- CNCF Sandbox project; Apache-2.0; built by ex-Istio/Solo.io engineers.

### 2.2 Key capabilities
- **Declarative agents** as CRDs (`Agent`, `ModelConfig`, `RemoteMCPServer` — formerly `ToolServer`, `Memory`, etc.) — versioned in Git, rolled out via GitOps / `kubectl` / Helm / Argo. (Sessions are managed via the CLI/API rather than CRDs.)
- **BYO LLM provider**: OpenAI, Azure OpenAI, Anthropic, Google Vertex AI, Gemini, xAI, Ollama, Bedrock, custom via AI gateway.
- **BYO framework**: LangGraph, CrewAI, Google ADK, or your own — kagent orchestrates it.
- **Native MCP** + **A2A (Agent-to-Agent)** + OpenAI-compatible endpoints.
- **Long-term memory** (vector-backed), **human-in-the-loop**, prompt templates from ConfigMaps, skills loaded from Git, context compaction.
- **Observability by default**: OpenTelemetry traces, Prometheus metrics, structured logs.
- **Zero-trust**: runs on Istio / Ambient Mesh; mTLS, RBAC, policy-driven egress.
- **Postgres-backed state**, sandboxing, RBAC, security hardening.
- **Multi-runtime**: Go and Python ADK; can mix in same cluster.
- Bundled MCP tool servers for Kubernetes, Istio, Helm, Argo, Prometheus, Grafana, Cilium, etc. — i.e., heavy bias toward **platform/SRE use cases** (incident response, observability copilot, platform self-service).

### 2.3 Pros
- ✅ **Operational fit on Kubernetes is unmatched** — agents are first-class workloads with the same RBAC, GitOps, OTel, mesh, and policy story as everything else.
- ✅ **No platform lock-in** — open standards (MCP, A2A, OTel), Apache-2.0, CNCF.
- ✅ **Helm install + CRDs**, no proprietary database, no SaaS dependency.
- ✅ **Multi-tenant** out of the box via Kubernetes namespaces and RBAC.
- ✅ Strong fit for **platform-engineering / SRE agents** (its built-in tool catalog targets this).
- ✅ Vendor-neutral — works equally well on AKS, EKS, GKE, on-prem.

### 2.4 Cons
- ❌ **No native .NET runtime** — engine is Go + Python ADK. .NET is only possible by packaging your own container that exposes A2A or OpenAI-compatible endpoints.
- ❌ **No first-class Microsoft Foundry Agent Service integration** — Azure OpenAI works as a model provider, but there is no binding to Foundry hosted agents, knowledge tools, M365/SharePoint/Fabric grounding, or Foundry's agent identity model.
- ❌ **You must own a Kubernetes cluster** (and an OTel/metrics stack to fully benefit). Heavy for small teams / simple chatbots.
- ❌ Built-in tool catalog is **infra-centric** — limited business/M365 connector coverage versus Foundry.
- ❌ **Younger ecosystem** (CNCF Sandbox stage), smaller .NET / enterprise-Microsoft community.
- ❌ Multi-agent orchestration patterns are looser than MAF workflows (relies on A2A composition rather than typed graphs with checkpointing / restore).

---

## 3. Microsoft Agent Framework (MAF) — overview

### 3.1 What it is
- Microsoft's open-source **framework** (not a runtime) for building production-grade AI agents and multi-agent workflows in **.NET** and **Python**. The **core framework reached 1.0 GA on April 3, 2026** for both languages. Stable 1.0.x and 1.x lines exist on NuGet/PyPI; the latest in-progress builds (e.g., the 1.5.0 line) are still on a `-preview` cadence.
- Successor / unification of **Semantic Kernel** and **AutoGen** (official migration guides exist for both).
- Distributed as packages:
  - .NET: `Microsoft.Agents.AI` (GA), `Microsoft.Agents.AI.Foundry` (the current Foundry / Azure AI integration adapter — note: an older sibling package `Microsoft.Agents.AI.AzureAI` exists from the RC era but the GA docs and README install instructions point to `…Foundry`), `Microsoft.Agents.AI.OpenAI`, `Microsoft.Agents.AI.Anthropic`, `Microsoft.Agents.AI.Hosting`, `Microsoft.Agents.AI.A2A`, etc. Several adapter / hosting packages still publish 1.x preview builds even though the core is GA.
  - Python: `agent-framework` (with sub-packages per provider).

### 3.2 Key capabilities
- **Two primitives**:
  - **Agents** — single LLM-backed agents with tools, MCP, middleware, memory, sessions.
  - **Workflows** — graph-based orchestration: sequential, concurrent, handoff, group collaboration; with **checkpointing & restore/rehydration, streaming, human-in-the-loop**.
- **Provider matrix** (consistent API across all):
  - Microsoft Foundry Agent Service (server-managed `FoundryAgent`)
  - Foundry Models via ChatCompletion or Responses (`FoundryChatClient`)
  - Foundry Anthropic
  - Azure OpenAI (ChatCompletion / Responses)
  - OpenAI (ChatCompletion / Responses)
  - **Amazon Bedrock**, **Google Gemini**, Anthropic, Ollama, GitHub Copilot SDK
  - Plus any `IChatClient` (Microsoft.Extensions.AI)
- **Foundry Hosted Agents** (**preview** as of MAF 1.0 GA) — host MAF agents on Foundry-managed infrastructure. The minimal *invocation* of a hosted agent from your app is ~2 lines (`AIProjectClient.AsAIAgent(...)`); the full *deployment* path is heavier — build an x86_64 container, include the Foundry hosted-agent protocol library, author `agent.yaml`, then `azd provision` + `azd deploy` (or `create_version` via SDK + poll for `active`).
- **Declarative agents** via YAML.
- **Agent Skills** (domain knowledge bases from files / inline / class libraries).
- **Middleware** pipeline for request/response, exception handling.
- **Built-in OpenTelemetry** for traces, metrics, logs.
- **DevUI** — interactive developer UI for testing and debugging workflows.
- **AF Labs** — experimental packages (benchmarking, RL, research).
- **ASP.NET Core hosting** libraries to expose agents over HTTP / A2A / OpenAI-compatible endpoints.

### 3.3 Pros
- ✅ **GA / production-ready** — `Microsoft.Agents.AI` 1.0 shipped April 3, 2026; stable API surface in both .NET and Python with a long-term support commitment.
- ✅ **First-class .NET and Python** — same API surface, idiomatic in both. Runs on .NET 8 / 9 / 10, .NET Standard 2.0, and .NET Framework 4.7.2 — i.e., wide deployment surface.
- ✅ **Tightest possible Microsoft Foundry / Azure binding** — `FoundryChatClient`, `AIProjectClient.AsAIAgent(...)`, `FoundryAgent` (versioned, server-managed), Foundry Hosted Agents.
- ✅ **Two agent shapes** (Responses vs. Foundry-managed `FoundryAgent`) let you choose between client-defined and server-versioned agents without changing app code.
- ✅ **Workflows** (graph-based, typed, checkpointed, restore/rehydration) are richer than kagent's A2A composition.
- ✅ **Genuinely multi-provider** — Foundry, Azure OpenAI, OpenAI, **Amazon Bedrock**, **Google Gemini**, Anthropic (incl. Foundry Anthropic), Ollama, GitHub Copilot SDK, plus any `IChatClient` (Microsoft.Extensions.AI). Not Azure-only.
- ✅ **Deploys anywhere** — App Service, Azure Container Apps, AKS, Functions, Foundry Hosted Agents (preview), console, **and** any non-Azure host (EKS, GKE, on-prem, on-device). It is a library, not a managed service.
- ✅ **Identity governance via Microsoft Entra Agent ID / Microsoft Agent 365** — Foundry-hosted agents get this automatically; for self-hosted MAF (or kagent — see §8) you wire Entra Agent ID through the host's authentication stack. Note: full Agent 365 governance features require **Microsoft Agent 365 / M365 E7** licensing (Entra ID P1/P2 for Conditional Access, ID Protection, Governance).
- ✅ **Migration paths** from Semantic Kernel and AutoGen are documented.
- ✅ **Microsoft-backed** — long-term roadmap, security/SDL processes, NuGet/PyPI distribution, enterprise support trajectory.
- ✅ Plays well with **Microsoft.Extensions.AI** abstractions (any `IChatClient`).

### 3.4 Cons
- ❌ **Not a runtime** — you bring your own hosting (App Service, ACA, AKS, Functions, console, Foundry Hosted Agents). No control plane, no CRDs, no lifecycle management out of the box.
- ❌ **No native Kubernetes operator** — no CRD/GitOps model for agent definitions in a cluster (declarative YAML exists for the agent itself, but Kubernetes-native lifecycle is wiring-it-yourself).
- ❌ **Mesh / network policy / multi-tenant isolation defer to the host** — MAF gives you identity (via Entra Agent ID) and observability (via OTel), but service-mesh, mTLS between agents, and namespace-grade multi-tenancy depend on where you run it (Foundry, App Service, AKS, kagent, etc.).
- ❌ **Some adapter and hosting packages are still on a `1.x-preview` cadence** even though the core 1.0 is GA — e.g., `Microsoft.Agents.AI.Hosting`, `Microsoft.Agents.AI.A2A`, and others. Worth checking the version of the specific adapter you depend on.
- ❌ **Best-of-breed for Foundry/Azure scenarios** — provider/tooling momentum and most "just works" patterns assume Microsoft Foundry. You can absolutely run on AWS/GCP/on-prem with OpenAI/Anthropic/Ollama, but you'll lose the managed niceties of Foundry Agent Service (knowledge tools, content safety, M365/Teams) — see §6.

---

## 4. Microsoft Foundry / Azure ecosystem integration

### 4.1 MAF ↔ Foundry Agent Service
The integration is native and the lowest-friction path on the market today.

**Two patterns** are exposed through `Microsoft.Agents.AI.Foundry`:

| Pattern | Produced type | When to use |
|---|---|---|
| **Responses Agent** | `ChatClientAgent` via `AIProjectClient.AsAIAgent(...)` | Your app provides model + instructions + tools at runtime. No server-side agent resource. Most flexible. |
| **Foundry Agent** (versioned) | `FoundryAgent` | Definitions are created/versioned in the Foundry portal or via `AIProjectClient.AgentAdministrationClient` and consumed by code via `ProjectsAgentVersion` / `ProjectsAgentRecord` / `AgentReference`. |

**.NET install** (current, GA-aligned package — Microsoft Learn install snippets may still show `--prerelease`; check NuGet for the latest stable version):
```bash
dotnet add package Microsoft.Agents.AI
dotnet add package Microsoft.Agents.AI.Foundry
dotnet add package Azure.AI.Projects
dotnet add package Azure.Identity
```

**Foundry Hosted Agents** (preview as of MAF 1.0 GA) lets the same MAF agent code run locally during dev and as a managed Foundry agent in prod. Wiring an *existing* hosted Foundry agent into your app is ~2 lines (`AIProjectClient.AsAIAgent(...)`). *Deploying* an MAF agent to Foundry-hosted infra is heavier: containerize (x86_64), include the Foundry hosted-agent protocol library, author `agent.yaml`, then `azd provision` / `azd deploy` (or `create_version` via SDK + poll for `active`).

### 4.2 kagent ↔ Foundry Agent Service
There is **no first-class Foundry Agent Service integration** in kagent today.
- Azure OpenAI is supported as a `ModelConfig` (you provide endpoint, deployment name, API version, and either an API key Secret or AAD token).
- That gives you Azure-hosted **models**, but not Foundry **agents**, knowledge tools, or M365 grounding.
- You can call Foundry-published agents from kagent by treating them as remote A2A endpoints, but you lose the declarative CRD experience for those agents — they live in Foundry, not in kagent.

### 4.3 Integration ease — verdict

| Question | kagent | MAF |
|---|---|---|
| Use Azure OpenAI / Foundry **models** | Easy (CRD `ModelConfig`) | Easy (`AzureOpenAIClient`, `FoundryChatClient`) |
| Use **Foundry Agent Service** managed agents | ❌ Not first-class — only via remote A2A calls | ✅ Native (`FoundryAgent`, `AsAIAgent`) |
| Deploy agents to **Foundry Hosted Agents** | ❌ N/A (kagent deploys to Kubernetes) | ✅ Native (~2 lines) |
| Use **Foundry knowledge tools** (File Search, Bing, SharePoint, Fabric, Azure AI Search) | ❌ Not directly — would need MCP wrappers | ✅ Via Foundry Agent Service backing |
| Use **Foundry action tools** (Azure Functions, OpenAPI, Code Interpreter, Computer Use¹, Browser Automation¹, Image Gen, MCP; Logic Apps connectors are integrated by converting them into remote MCP servers; A2A is preview; Deep Research is available via the `o3-deep-research` model) | ❌ Limited (MCP/A2A wrappers required) | ✅ Via Foundry Agent Service backing |
| **Microsoft 365 / Teams** deployment for the agent | ❌ Not in scope | ✅ Built-in via Foundry Agent Service |
| **Identity / agent ID / content safety / moderation** | Custom (you build it) | Built-in via Foundry Agent Service |

> **Bottom line:** if Foundry / Azure / M365 alignment is the priority, MAF is the obvious choice. kagent is Foundry-agnostic and treats Azure OpenAI as just another LLM.

---

## 5. Platform vs. platform — **kagent** vs. **Foundry Agent Service**

Both kagent and Foundry Agent Service are **agent platforms / runtimes** (not SDKs). They sit at the same layer of the stack but answer the question "where do I run my agents?" with very different opinions.

### 5.1 What each platform actually is

| | **kagent** | **Foundry Agent Service** |
|---|---|---|
| Category | Self-hosted, **Kubernetes-native** agent runtime + control plane | Microsoft-managed, **PaaS** agent runtime |
| Operating model | You install (`helm install`), you operate | Microsoft operates; you consume |
| Footprint | Runs in **your** cluster (AKS, EKS, GKE, on-prem, kind) | Runs in **Microsoft Foundry** (Azure regions) |
| Pricing model | Free OSS + your infra cost (cluster, LLM tokens, OTel/Postgres) | Pay-as-you-go: Foundry/Azure OpenAI tokens + service charges + tool charges (Bing, Logic Apps, etc.) |
| Tenancy | Single-tenant per cluster (multi-tenant via K8s namespaces + RBAC) | Multi-tenant SaaS, isolated per Foundry project |
| Authoring surface | YAML (CRDs) + bundled UI + Helm/`kubectl`/CLI | Foundry portal (low-code) + REST/SDK (`AIProjectClient`, `AgentAdministrationClient`) |
| Maturity / governance | CNCF Sandbox, Apache-2.0 | GA Microsoft service (with classic→new migration in progress; classic retires **March 31, 2027**) |

### 5.2 Capabilities head-to-head

| Capability | **kagent** | **Foundry Agent Service** |
|---|---|---|
| **Agent definition** | `Agent` CRD (YAML, GitOps-friendly) | Server-managed agent resource (`FoundryAgent` versioned in portal or via API) |
| **Lifecycle** | Kubernetes controller reconciles CRDs | Foundry service manages versions, threads, runs |
| **Runtime engines** | Go and Python ADK (Google ADK) | Microsoft-managed; supports Foundry models incl. GPT family, o-series, Anthropic via Foundry, etc. |
| **LLM providers** | OpenAI, Azure OpenAI, Anthropic, Vertex/Gemini, xAI, Ollama, custom | Foundry catalog (Azure OpenAI, OSS, Anthropic via Foundry, etc.) — Microsoft-curated |
| **Knowledge tools (RAG)** | BYO via MCP wrappers; no built-in connectors for SharePoint/Fabric/web search | Built-in: **File Search**, **Azure AI Search**, **Web search** (Bing-backed; advanced "Grounding with Bing" tools also available), **SharePoint** (OBO), **Fabric Data Agent**, licensed (Tripadvisor, Morningstar) |
| **Action tools** | MCP tool servers (Kubernetes, Istio, Helm, Argo, Prometheus, Grafana, Cilium, custom MCP) | Built-in: **Code Interpreter**, **Azure Functions**, **OpenAPI 3.0**, **MCP**, **Computer Use** (preview), **Browser Automation** (preview), **Image Generation**, **A2A** (preview); Logic Apps connectors integrated via MCP-server conversion; Deep Research via the `o3-deep-research` model |
| **Multi-agent orchestration** | A2A composition between CRD-defined agents | **Connected Agents** + **Workflows** (`2025-11-15-preview` API), no custom orchestrator needed |
| **Memory / state** | Vector-backed long-term memory CRDs; Postgres-backed | Built-in conversation state + memory management |
| **Human-in-the-loop** | Tool approval gates, cascading HITL | Run-step approval / function-call confirmation patterns |
| **Identity** | Kubernetes RBAC + your IdP (Entra Agent ID is possible if you wire it via the host) | Built-in **Microsoft Entra Agent ID** (Agent 365 governance), Entra ID auth, OBO for SharePoint/Fabric |
| **Content safety / moderation** | Optional via **NVIDIA NeMo Guardrails / Nemotron** (kagent packages this as "NemoClaw"; BYO policies) | Built-in Azure AI Content Safety |
| **Observability** | OTel traces, Prometheus metrics, structured logs (default) | Built-in tracing in Foundry portal; OTel export available |
| **Network / security** | Istio / Ambient Mesh, mTLS, fine-grained RBAC, policy-driven egress, sandboxing | Foundry network isolation, private endpoints, customer-managed keys, BYO VNet |
| **Authoring UX** | Bundled web UI + `kubectl`/Helm/CLI/GitOps | **Foundry portal** with built-in playground, low-code agent design |
| **Versioning** | Git (because everything is YAML) | Server-side `ProjectsAgentVersion` records inside the project |
| **Deployment surface** | Anywhere Kubernetes runs | Microsoft Foundry only (Azure regions) |
| **M365 / Teams deployment** | ❌ Out of scope | ✅ First-class M365 / Teams deployment paths |
| **Air-gapped / on-prem / sovereign** | ✅ Yes (run on any cluster, point at any model) | ❌ Tied to Azure / Foundry regions |
| **Bring-your-own framework** | ✅ Explicit (LangGraph, CrewAI, ADK, anything wrapped as A2A/MCP/OpenAI-compatible) | ⚠️ Limited — Foundry Agent Service expects Foundry-managed agents; you can call external A2A endpoints, but the agent itself is Foundry-shaped |
| **Multi-cluster / multi-cloud** | ✅ Native | ❌ Not in scope |
| **Vendor lock-in** | Low — open standards (MCP, A2A, OTel, K8s) | High — Foundry resource model, Azure RBAC, M365 integrations |

### 5.3 Pros and cons as platforms

#### kagent (as a platform)
**Pros**
- ✅ Runs in **your** infrastructure — sovereignty, data residency, on-prem, air-gapped all possible.
- ✅ **GitOps-native** — agents are CRDs; PR review, RBAC, drift detection, multi-cluster all come for free.
- ✅ **No vendor lock-in** — swap LLMs, frameworks, tools without migration.
- ✅ Same operational stack as the rest of your workloads (mesh, OTel, Prometheus, Postgres).
- ✅ Strong **multi-tenancy** model via K8s namespaces.
- ✅ Free / OSS — you only pay for infra and tokens.

**Cons**
- ❌ **You own day-2** — upgrades, scaling, capacity, backups, security patching are yours.
- ❌ **No built-in enterprise grounding** — no SharePoint, Fabric, Bing, M365 tools out of the box; everything is BYO MCP.
- ❌ **No managed content safety / agent ID** — assemble it yourself.
- ❌ **Smaller ecosystem** vs. Microsoft's catalog of tools and connectors.
- ❌ Requires **Kubernetes expertise** in the team.
- ❌ Not a fit if you need a low-code authoring portal.

#### Foundry Agent Service (as a platform)
**Pros**
- ✅ **Fully managed** — autoscale, patching, capacity, identity, content safety are Microsoft's problem.
- ✅ **Massive built-in tool catalog** — Web search (Bing-backed), SharePoint, Fabric, File Search, Azure AI Search, Code Interpreter, Azure Functions, OpenAPI, MCP, Image Gen; plus Computer Use, Browser Automation, A2A in preview; Logic Apps via MCP-server conversion; Deep Research via the `o3-deep-research` model.
- ✅ **M365 / Teams** deployment is a first-class destination.
- ✅ **Foundry portal** gives PMs/analysts a low-code authoring + playground experience.
- ✅ **Connected Agents / Workflows** for multi-agent without custom orchestration.
- ✅ Tight binding to Entra ID, **Microsoft Entra Agent ID** (Agent 365 governance), Azure RBAC, private networking, customer-managed keys, regulated-cloud regions.
- ✅ Consumed natively by **MAF** (`FoundryAgent`, `AsAIAgent`) and by Semantic Kernel / AutoGen migration paths.

**Cons**
- ❌ **Azure / Foundry only** — no on-prem, no other clouds, no air-gapped.
- ❌ **Higher latency** than in-process MAF (managed, remote service).
- ❌ **Per-call cost** (tokens + service + each premium tool like Web search, Azure Functions executions, etc.).
- ❌ **Less control** over runtime internals; your agent is a Foundry-shaped resource.
- ❌ **Vendor lock-in** to the Foundry resource model and Azure identity.
- ❌ **Migration in flight** — classic Foundry agents are deprecated and retire **March 31, 2027**; new APIs (`2025-11-15-preview`) are the path forward but some patterns are still moving.
- ❌ Multi-tenancy is per-project, not per-namespace — coarser-grained than K8s.

### 5.4 Decision lens

| If your priority is… | Pick |
|---|---|
| Lowest ops burden, fastest path to a working enterprise agent with M365/SharePoint/Fabric grounding | **Foundry Agent Service** |
| Air-gapped / sovereign cloud / on-prem / non-Azure clouds | **kagent** |
| Strict GitOps + multi-cluster + Kubernetes-native RBAC and mesh | **kagent** |
| Need Web search / Deep Research / Computer Use / Browser Automation / Logic Apps connectors out of the box | **Foundry Agent Service** |
| Platform/SRE agents that act on your cluster (Helm, Istio, Argo, Prometheus) | **kagent** (its bundled tool servers target this) |
| Low-code authoring + playground for non-engineers | **Foundry Agent Service** |
| Avoiding vendor lock-in across LLM, framework, and infra | **kagent** |
| Per-agent versioning governed centrally with audit trails inside a portal | **Foundry Agent Service** (`ProjectsAgentVersion`) |
| Hybrid — some workloads managed (Foundry), some self-hosted (kagent), one SDK across both | **MAF as the SDK**, with **Foundry Agent Service for hosted** and **kagent for self-hosted** (see §7) |

### 5.5 Can they coexist?

Yes — and many real architectures will end up here:

- **Foundry Agent Service** as the home for customer-facing or M365-embedded agents that benefit from built-in grounding and managed identity.
- **kagent** as the home for **operational agents** that act inside your cluster (incident response, observability copilots, platform self-service) and need mesh/RBAC/sovereignty.
- A **kagent** agent can call a **Foundry Agent Service** agent via **A2A** (Foundry exposes A2A endpoints; kagent speaks A2A natively). The reverse — Foundry agents calling kagent agents — also works via A2A or via OpenAPI/MCP wrappers around kagent endpoints.
- **MAF** is the natural common SDK if you want the same authoring experience across both platforms.

---

## 6. What you give up by **not** using Foundry Agent Service (when running MAF standalone)

You can absolutely run MAF agents without Foundry Agent Service (using `AzureOpenAIClient` / `FoundryChatClient` for raw model inference, plus other Azure services directly). What you lose, per Microsoft's own comparison, is the *bundled* experience — the table below is "MAF standalone" meaning *without Foundry Agent Service*, **not** "without any Azure service." For example, you can still call **Azure AI Search**, **Azure AI Content Safety**, **Entra Agent ID**, and **Azure AI Evaluation** directly from MAF without going through Foundry Agent Service:

| Capability | MAF standalone (no Foundry Agent Service) | MAF + Foundry Agent Service |
|---|---|---|
| **Performance** | Fast — runs locally / in your process | Slower — managed, remote service |
| **Development model** | Full code, maximum control | Low-code, rapid integration; portal-based agent design |
| **Testing** | Manual / unit tests in code | Built-in **Foundry playground** for quick iteration |
| **Scalability** | App-managed | Azure-managed, autoscaled |
| **Security guardrails** | You wire (Azure AI Content Safety SDK is available standalone) | Built-in content safety + moderation, no wiring |
| **Identity / Agent ID** | You wire (Entra Agent ID can be used in any Entra-aware host) | Auto-provisioned per agent |
| **Memory / state** | You wire (vector DB, sessions) | Built-in memory and conversation state |
| **Knowledge tools** | You wire (RAG pipelines, **Azure AI Search** is callable standalone) | Bundled, server-side: **File Search**, **Web search** (Bing-backed), **SharePoint**, **Fabric Data Agent**, **Azure AI Search**, licensed data (Tripadvisor, Morningstar) |
| **Action tools** | You wire as `AIFunction` (Azure Functions, OpenAPI, MCP all callable) | Bundled, server-side: **Code Interpreter**, **Azure Functions**, **OpenAPI 3.0**, **MCP**, **Image Generation**; **Computer Use**, **Browser Automation**, **A2A** in preview; Logic Apps via MCP-server conversion; Deep Research via the `o3-deep-research` model |
| **Multi-agent orchestration** | MAF Workflows (graph-based, checkpointed, restore/rehydrate) | Foundry **Connected Agents** + **Workflows** (`2025-11-15-preview` API) — no custom orchestrator needed |
| **Evaluation / red-teaming** | `Microsoft.Extensions.AI.Evaluation` SDK (standalone) | Foundry **AI Red Teaming Agent** (PyRIT-based) + Azure AI Evaluation pipelines |
| **Enterprise deployment** | Custom integration | Built-in Microsoft 365 / Teams deployment, integrated M365 tool calls |
| **Versioning / governance** | You implement | Server-managed `FoundryAgent` versions in the Foundry portal |
| **Compliance / data residency** | You configure per Azure resource | Inherited from Foundry project boundary |

**Net trade-off:**
- Choose **MAF standalone** when you want maximum control, lowest latency, full code ownership, simpler deps, multi-cloud.
- Choose **MAF + Foundry Agent Service** when you want managed scalability, built-in M365/SharePoint/Fabric grounding, content safety, agent identity, and a portal-driven authoring experience.

---

## 7. Can you use **MAF + kagent**? (Build with MAF, deploy on kagent)

**Yes, with caveats.** They don't have a native integration today, but they overlap cleanly at the edges thanks to shared open standards.

### Why it works in principle
- kagent's **BYO framework** philosophy explicitly invites LangGraph, CrewAI, Google ADK, "or your own."
- kagent talks **A2A**, **MCP**, and **OpenAI-compatible** endpoints natively.
- MAF **ASP.NET Core hosting** can expose any agent over HTTP / A2A / OpenAI-compatible endpoints.

### The integration pattern
1. Build your agent in **MAF .NET** (or Python).
2. Host it in an **ASP.NET Core** app using `Microsoft.Agents.AI` hosting libraries; expose A2A and/or an OpenAI-compatible endpoint.
3. Containerize it (`mcr.microsoft.com/dotnet/aspnet:9.0` base image).
4. Push to your registry (ACR, GHCR, etc.).
5. Deploy as a **Kubernetes `Deployment` + `Service`** in your kagent cluster.
6. Register it inside kagent as a **remote agent** via A2A (so other kagent agents can discover and delegate to it) or as a **`RemoteMCPServer`** CRD (if exposing MCP tools) — whichever fits.

### What you keep
- ✅ MAF's authoring experience, .NET tooling, workflows, Foundry binding.
- ✅ kagent's GitOps, RBAC, OTel pipeline, mesh/mTLS, multi-tenancy, observability dashboards.

### What you lose
- ❌ The kagent **Agent CRD UX** doesn't natively model MAF agents (they aren't ADK runtime agents) — you manage them as ordinary K8s Deployments rather than `kind: Agent` resources.
- ❌ Some kagent goodies (memory CRDs, prompt-template ConfigMaps, skills-from-Git) won't apply automatically — those are MAF's job in your container.
- ❌ Two control planes to reason about: MAF (in-process workflow + Foundry binding) and kagent (cluster-level lifecycle + tool servers).

### When this combination makes sense
- You're a **.NET shop** that wants Foundry-class developer ergonomics **and** wants to run agents in your own AKS / EKS / on-prem Kubernetes cluster with strict mesh/RBAC policies.
- You want **Foundry Agent Service for some workloads** (managed) and **self-hosted kagent for others** (sovereignty, latency, on-prem data) — MAF is the common SDK between both.

### When it doesn't
- You only need a single hosted agent → just use **MAF + Foundry Hosted Agents**.
- You only need infra/SRE agents on a cluster → just use **kagent** with its bundled Go/Python ADK runtime.

---

## 8. Wiring kagent agents into Agent 365 / Microsoft Entra Agent ID

A common follow-up question: *can a kagent agent get an Entra agent identity and participate in the Microsoft Agent 365 control plane the same way a Foundry-hosted MAF agent can?*

**Short answer: Yes, but it is not built into kagent.** You wire it yourself using the third-party agent integration patterns Microsoft documents for non-Microsoft platforms (the docs explicitly name AWS Bedrock, n8n, Ollama, "any containerized agent on Docker or Kubernetes" — kagent fits that last bucket).

### Two integration patterns (both apply to kagent)

#### Pattern A — Microsoft Entra Auth SDK as a **sidecar** (recommended for kagent)
Documented as the path for "containerized agents on Docker or Kubernetes":
- Deploy Microsoft's Entra Auth SDK image as a sidecar container in the same pod as your kagent agent.
- The agent calls the sidecar on `localhost:7000/token` to acquire an Entra access token.
- The sidecar talks to Microsoft Entra Agent ID; the agent never handles credentials.
- Works on AKS, EKS, GKE, on-prem — anywhere kagent runs.

#### Pattern B — **Workload Identity Federation** (no sidecar)
- Configure a Federated Identity Credential on the Agent Identity in Microsoft Entra.
- Use the cluster's OIDC issuer + the Kubernetes ServiceAccount as the federation subject (e.g., `system:serviceaccount:<namespace>:<serviceaccount>`).
- The kagent pod gets an Entra token via direct OIDC exchange — no secrets, no sidecar.
- Cleanest fit for AKS (Azure Workload Identity) or EKS/GKE with their native OIDC providers.

### What you get once wired up
- ✅ A real **Agent Identity** in Microsoft Entra (registered in the Entra admin center agent registry).
- ✅ **Agent 365 governance** primitives — discovery, audit, lifecycle, retirement.
- ✅ **Conditional Access for agents** (with Entra ID P1 / M365 E3).
- ✅ **ID Protection / ID Governance for agents** (with Entra ID P2 / M365 E5 / Entra Suite).
- ✅ **Network controls** via Global Secure Access (Entra Suite).
- ✅ Tokens to call Microsoft Graph, SharePoint, Fabric, custom APIs — same as any other Entra-registered workload.

### Licensing
- **Microsoft Agent 365** or **Microsoft 365 E7** license per user the agent acts on behalf of. The agent itself does not need a license; it is covered under the user's license.
- Conditional Access / ID Protection / ID Governance for agents are **separately licensed** (Entra ID P1/P2 or M365 E3/E5).

### Effort comparison: kagent vs MAF for getting an Entra Agent ID

| Question | kagent | MAF (self-hosted) | MAF on Foundry Hosted Agents (preview) |
|---|---|---|---|
| Can the agent have an Entra Agent ID? | ✅ Yes — via sidecar or federation pattern (you wire it) | ✅ Yes — via `Microsoft.Identity.Web` SDK or sidecar/federation (you wire it) | ✅ Yes — **auto-provisioned** by Foundry |
| Registered in Microsoft Entra admin center agent registry? | ✅ Yes — once you register the Agent Identity blueprint | ✅ Yes — same path | ✅ Yes — automatically |
| Discoverable / governable in Agent 365 control plane? | ✅ Yes — same as any third-party agent | ✅ Yes | ✅ Yes |
| Effort to set up | **Medium** — deploy sidecar container + register Agent Identity + configure FIC | **Medium-Low** — `Microsoft.Identity.Web` is native to .NET; or sidecar/federation | **Lowest** — provisioned for you |
| Built-in / first-class | ❌ No — kagent has no Agent 365 / Entra Agent ID CRD or operator today | ⚠️ Partial — .NET SDKs are first-class, but you still wire identity per host | ✅ First-class |

### Bottom line
- **Can a kagent agent be a first-class Agent 365 citizen with an Entra Agent ID?** Yes.
- **Will it be as effortless as MAF on Foundry Hosted Agents?** No. You take on the sidecar deployment (or workload-identity-federation setup) and the per-agent registration in Entra. Microsoft's docs walk through the recipe end-to-end for AWS Bedrock and n8n; kagent follows the same recipe because Microsoft treats it as "any containerized agent on Kubernetes."
- **Is there a kagent-native Agent 365 integration on the roadmap?** Not visible in public kagent documentation as of this document's date. The Microsoft Entra third-party integration page explicitly names AWS Bedrock and n8n; kagent is not (yet) a named partner.

> **Practical implication for the §11 recommendation matrix:** if Agent 365 governance is a hard requirement, MAF on Foundry Hosted Agents is the lowest-friction path. If you must run on Kubernetes (sovereignty, on-prem, multi-cluster) and still need Agent 365 governance, kagent + Entra Auth SDK sidecar (or workload-identity federation) is a valid — but DIY — path.

---

## 9. .NET support comparison

| Question | kagent | MAF |
|---|---|---|
| Native .NET runtime in the engine | ❌ No — engine is Go + Python ADK | ✅ Yes — `Microsoft.Agents.AI` |
| .NET NuGet packages | ❌ None for authoring agents | ✅ `Microsoft.Agents.AI` (GA), `Microsoft.Agents.AI.Foundry`, `Microsoft.Agents.AI.OpenAI`, `Microsoft.Agents.AI.Anthropic`, `Microsoft.Agents.AI.Hosting`, `Microsoft.Agents.AI.A2A`, etc. (some adapters still publishing 1.x preview builds) |
| .NET docs / samples | ❌ Not a target audience | ✅ Extensive ([dotnet/samples](https://github.com/microsoft/agent-framework/tree/main/dotnet/samples)) |
| Can run a .NET agent on it | ⚠️ Yes, but only as an opaque container exposing A2A / OpenAI endpoints — not as a kagent-native `Agent` CRD | ✅ Yes, anywhere you can host ASP.NET Core (App Service, ACA, AKS, kagent, on-prem) |
| ASP.NET Core hosting integration | ❌ Out of scope | ✅ First-class hosting libraries |

> **If .NET is a hard requirement**, MAF wins decisively. kagent can run a .NET workload, but it's not a .NET-aware platform.

---

## 10. Side-by-side cheat sheet

| Topic | kagent | MAF |
|---|---|---|
| Project type | Runtime / control plane | SDK / framework |
| Governance | CNCF Sandbox (kagent-dev) | Microsoft OSS |
| License | Apache-2.0 | MIT |
| Languages (build) | YAML CRDs (+ Go/Python for engine extensions) | C# / .NET, Python |
| Languages (run) | Go, Python ADK | .NET, Python |
| Deployment target | Kubernetes (any) | App Service, ACA, AKS, Functions, Foundry Hosted Agents, console |
| Agent definition | `Agent` CRD | Code (`AIAgent`) or YAML (declarative agents) |
| Multi-agent | A2A composition | Workflows (graph, sequential / concurrent / handoff / group, checkpointed with restore/rehydration) |
| Tools | MCP, OpenAI-compatible, A2A | MCP, function tools, hosted MCP, plus Foundry's full tool catalog when backed by Foundry Agent Service |
| LLM providers | OpenAI, Azure OpenAI, Anthropic, Vertex / Gemini, xAI, Ollama, custom | Foundry, Azure OpenAI, OpenAI, Anthropic (incl. Foundry Anthropic), Ollama, GitHub Copilot, any `IChatClient` |
| Memory | Built-in vector memory CRD, Postgres-backed | Context providers + agent sessions; bring your own store |
| HITL | Tool approval gates, cascading HITL | HITL nodes inside Workflows |
| Observability | OTel, Prometheus, structured logs (default) | OTel (default, in-code) |
| Security | Istio/Ambient mesh, mTLS, RBAC, sandboxing | Defer to host for mesh/network; **identity via Microsoft Entra Agent ID + Azure RBAC**, content safety via Foundry when applicable |
| GitOps lifecycle | Native (CRDs) | Not native (deploy via your CI/CD) |
| Foundry Agent Service | ❌ No first-class binding | ✅ First-class (`FoundryAgent`, `AsAIAgent`, Foundry Hosted Agents) |
| Multi-tenant by design | ✅ K8s namespaces + RBAC | ⚠️ Defer to host |
| Best for | Platform/SRE agents on Kubernetes; sovereignty; multi-cluster | Building agents (esp. .NET) tightly aligned with Microsoft Foundry / Azure / M365 |

---

## 11. Recommendation matrix

| If your context is… | Pick |
|---|---|
| .NET shop, building agents that should plug into Foundry / Azure OpenAI / M365 / Teams | **MAF** (with Foundry Agent Service for managed scenarios) |
| Building a single hosted agent, want lowest time-to-prod on Azure | **MAF + Foundry Hosted Agents** |
| Need built-in M365 / SharePoint / Fabric / web search grounding without writing RAG yourself | **MAF + Foundry Agent Service** |
| Already on Kubernetes; want GitOps, mTLS, RBAC, multi-tenant agents at platform scale | **kagent** |
| Building SRE / platform-engineering / observability copilots on a cluster | **kagent** (its bundled tool servers are aimed exactly here) |
| You want both: rich Foundry-native authoring **and** Kubernetes-grade ops | **MAF for authoring, kagent for hosting** (§7) — accept the integration tax |
| You need air-gapped / on-prem / sovereign-cloud deployment with no Microsoft / Azure dependency at all | **kagent** with Ollama or a self-hosted model (MAF can also self-host on the same cluster, but kagent gives you the control-plane / CRD UX) |

---

## 12. Sources

- kagent home — https://kagent.dev/
- kagent GitHub — https://github.com/kagent-dev/kagent
- kagent Azure OpenAI provider — https://kagent.dev/docs/kagent/supported-providers/azure-openai
- kagent quickstart — https://kagent.dev/docs/kagent/getting-started/quickstart
- Microsoft Agent Framework docs — https://learn.microsoft.com/en-us/agent-framework/
- MAF GitHub — https://github.com/microsoft/agent-framework
- MAF NuGet profile (versions, GA / preview status) — https://www.nuget.org/profiles/MicrosoftAgentFramework
- MAF 1.0 GA announcement — https://devblogs.microsoft.com/agent-framework/microsoft-agent-framework-version-1-0/
- MAF — Microsoft Foundry provider (C#) — https://learn.microsoft.com/agent-framework/agents/providers/microsoft-foundry
- MAF — agent types — https://learn.microsoft.com/agent-framework/agents/
- MAF — Workflows checkpointing — https://learn.microsoft.com/en-us/agent-framework/workflows/checkpoints
- MAF + Foundry Agent Service tutorial (App Service, .NET) — https://learn.microsoft.com/azure/app-service/tutorial-ai-agent-web-app-semantic-kernel-foundry-dotnet
- Foundry Agent Service overview — https://learn.microsoft.com/azure/ai-foundry/agents/overview
- Foundry Agent Service tool catalog — https://learn.microsoft.com/azure/foundry/agents/concepts/tool-catalog
- Foundry Agent Service Transparency Note — https://learn.microsoft.com/azure/foundry/responsible-ai/agents/transparency-note
- Foundry Connected Agents (classic) — https://learn.microsoft.com/azure/foundry-classic/agents/how-to/connected-agents
- Foundry Hosted Agents — https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/hosted-agents
- Foundry agent identity — https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/agent-identity
- Microsoft Entra Agent ID (Agent 365 governance) — https://learn.microsoft.com/en-us/entra/agent-id/what-is-agent-id-platform
- Microsoft Entra Agent ID — third-party agents — https://learn.microsoft.com/en-us/entra/agent-id/configure-third-party-agents
- kagent API reference (CRDs) — https://kagent.dev/docs/kagent/resources/api-ref
