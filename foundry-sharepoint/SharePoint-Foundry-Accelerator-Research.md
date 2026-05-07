# SharePoint + Microsoft Foundry: AI Knowledge Accelerator -- Deep Research & Recommendations

> **Author:** Research compiled May 2026  
> **Status:** Deep-dive analysis with recommended architectures  
> **Peer-reviewed by:** GPT-5.2 (fact-check), Claude Sonnet 4.5 (architecture), Claude Sonnet 4 (completeness)

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Problem Statement](#problem-statement)
3. [Prerequisites & Licensing](#prerequisites--licensing)
4. [Key Microsoft Building Blocks](#key-microsoft-building-blocks)
5. [Decision Tree: Which Option Is Right for You?](#decision-tree)
6. [Architecture Options](#architecture-options)
   - [Option A -- Foundry Agent + Foundry IQ + Teams (Recommended Start)](#option-a--foundry-agent--foundry-iq--teams-recommended-start)
   - [Option B -- Copilot Studio (Low-Code)](#option-b--copilot-studio-low-code)
   - [Option C -- M365 Copilot Declarative Agent (Zero-Code)](#option-c--m365-copilot-declarative-agent-zero-code)
   - [Option D -- Custom Web App with RAG Pipeline](#option-d--custom-web-app-with-rag-pipeline)
   - [Option E -- Hybrid: Foundry Backend + Custom Web Frontend](#option-e--hybrid-foundry-backend--custom-web-frontend)
7. [Comparison Matrix](#comparison-matrix)
8. [Deep Dive: Foundry IQ vs SharePoint Tool (Key Distinction)](#deep-dive-foundry-iq-vs-sharepoint-tool)
9. [Deep Dive: SharePoint Knowledge Agent (AI in SharePoint)](#deep-dive-sharepoint-knowledge-agent)
10. [Deep Dive: Teams & Bot Integration](#deep-dive-teams--bot-integration)
11. [Conversation History & Q&A Analytics](#conversation-history--qa-analytics)
12. [Security & Compliance](#security--compliance)
13. [Known Limitations & Preview Caveats](#known-limitations--preview-caveats)
14. [Pricing & Cost Considerations](#pricing--cost-considerations)
15. [Anti-Patterns: What NOT to Do](#anti-patterns)
16. [Reference Implementations & Samples](#reference-implementations--samples)
17. [Final Recommendation](#final-recommendation)

---

## Executive Summary

Microsoft provides multiple paths for building AI-powered Q&A agents over SharePoint content, ranging from zero-code to fully custom. The recommended **progressive approach** is:

1. **Start with Option A** (Foundry Agent + Foundry IQ + Teams) for fastest time-to-value
2. **Evolve to Option E** (Hybrid) only when real user feedback demands a custom web app or advanced analytics

The core stack combines:

- **Microsoft Foundry Agent Service** -- orchestrates the AI agent logic
- **Foundry IQ** (Preview) -- managed knowledge layer that indexes SharePoint via Azure AI Search
- **Azure AI Search** -- powers hybrid (keyword + vector) retrieval with ACL enforcement
- **Azure Bot Service / M365 Agents SDK** -- bridges the agent to Teams and/or a custom web app
- **Azure Cosmos DB** (optional) -- adds persistent conversation history and Q&A analytics beyond Foundry's built-in thread management

Alternative paths exist for different team profiles: **Copilot Studio** (low-code), **M365 Copilot declarative agents** (zero-code), or **custom RAG pipelines** (full control).

> **Important:** Foundry IQ and the SharePoint tool in Foundry Agent Service are **two different mechanisms** with different capabilities and limitations. This document clarifies when to use each.

---

## Problem Statement

Organizations store critical knowledge across SharePoint -- sites, pages, document libraries, lists, wikis. Employees struggle to find answers because:

- Content is scattered across dozens of sites and libraries
- Search returns documents, not answers
- No institutional memory of previously answered questions
- No contextual follow-up (each search is isolated)

**Goal:** Build an AI-powered agent that:
1. Answers natural-language questions grounded in SharePoint content
2. Cites specific source documents
3. Respects SharePoint permissions (security trimming)
4. Tracks conversation history for follow-up questions
5. Caches and surfaces popular Q&A pairs to improve response time and provide analytics
6. Is accessible via Teams AND/OR a web application

---

## Prerequisites & Licensing

### Required Licenses & Subscriptions

| Requirement | Details |
|---|---|
| **Azure subscription** | Pay-as-you-go or Enterprise Agreement |
| **Microsoft 365** | E3 or E5 (for SharePoint Online content) |
| **M365 Copilot license** | Required for remote SharePoint queries via Copilot Retrieval API; OR use pay-as-you-go at $0.10/API call (Preview). NOT required if using indexed approach via Azure AI Search. |
| **Azure OpenAI access** | Approved Azure OpenAI resource with a GPT-5.1 deployment (or GPT-5 / GPT-5-mini for cost-optimized scenarios) |
| **Entra ID** | App registrations for authentication (OBO flow for identity passthrough) |

### Azure RBAC Roles Required

| Role | Resource | Purpose |
|---|---|---|
| Azure AI User | Foundry project | Create and run agents |
| Search Index Data Reader | Azure AI Search | Read search indexes |
| Search Index Data Contributor | Azure AI Search | Write to indexes (if agent needs write access) |
| Cosmos DB Contributor | Cosmos DB account | Read/write conversation history |
| Cognitive Services OpenAI User | Azure OpenAI | Call model endpoints |

### SharePoint Admin Settings
- Ensure SharePoint content is indexed by Microsoft Search
- Enable semantic indexing for Microsoft 365 Copilot (if using Copilot Retrieval API)
- Foundry project and SharePoint tenant must be in the **same Microsoft Entra tenant**

### Supported SharePoint Content Formats
- Documents: PDF, DOCX, PPTX, XLSX, TXT, HTML, Markdown
- SharePoint pages, lists, and document libraries
- Note: Recently added content may need time to be indexed by Microsoft Search

---

## Key Microsoft Building Blocks

### 1. Microsoft Foundry Agent Service
- Fully managed agent runtime -- no infrastructure to manage
- Supports Python, C#, JavaScript, Java SDKs
- Built-in tool orchestration (knowledge retrieval, code interpreter, custom functions)
- Publishes directly to M365 Copilot and Teams
- **Docs:** [learn.microsoft.com/azure/foundry/agents](https://learn.microsoft.com/en-us/azure/foundry/agents/)

### 2. Foundry IQ (Knowledge Layer)
- Multi-source knowledge base connecting SharePoint, Azure Blob, OneLake, and web data
- Automated chunking, vector embedding, and metadata extraction
- **Agentic retrieval engine** -- LLM-driven query planning, decomposition, parallel search, and aggregation
- Returns extractive answers with **citations** to source documents
- Enforces ACLs and Microsoft Purview sensitivity labels at query time
- **Docs:** [learn.microsoft.com/azure/foundry/agents/concepts/what-is-foundry-iq](https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/what-is-foundry-iq)

### 3. Azure AI Search
- Powers the indexing and retrieval behind Foundry IQ
- Supports keyword, vector, and hybrid search
- Document-level security via ACL synchronization
- Free tier available for proof-of-concept
- **Docs:** [learn.microsoft.com/azure/search](https://learn.microsoft.com/en-us/azure/search/)

### 4. SharePoint Knowledge Agent ("AI in SharePoint")
- Built-in SharePoint feature (Preview; GA date not yet announced by Microsoft)
- Rebranded from "Knowledge Agent" to "AI in SharePoint"
- Auto-tags documents, classifies content, detects stale/broken pages
- Makes SharePoint content "AI-ready" for Copilot and custom agents
- Requires M365 Copilot license
- **Docs:** [learn.microsoft.com/sharepoint/ai-in-sharepoint-get-started](https://learn.microsoft.com/en-us/sharepoint/ai-in-sharepoint-get-started)

### 5. Azure Bot Service & M365 Agents SDK
- **Azure Bot Service** -- bridges agents to Teams, web, Slack, etc.
- **M365 Agents SDK** -- successor to Bot Framework v4 (BFv4 support ended Dec 31, 2025; repo archived); native Teams/Outlook/Copilot integration
- **Teams SDK (Teams AI Library)** -- for Teams-only scenarios
- **Foundry Agent SDK** -- for advanced multi-agent orchestration
- **Docs:** [learn.microsoft.com/azure/foundry/agents/how-to/publish-copilot](https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/publish-copilot)
- **Migration guide (BFv4 -> M365 Agents SDK):** [learn.microsoft.com/microsoft-365/agents-sdk/bf-migration-guidance](https://learn.microsoft.com/en-us/microsoft-365/agents-sdk/bf-migration-guidance)

### 6. Azure Cosmos DB (Conversation History)
- NoSQL database ideal for chat history and agent memory
- Supports vector search for semantic retrieval over past conversations
- TTL support for short-term vs long-term memory
- Integrates with Semantic Kernel, LangChain, and Microsoft Agent Framework
- **Docs:** [learn.microsoft.com/azure/cosmos-db/gen-ai/agentic-memories](https://learn.microsoft.com/en-us/azure/cosmos-db/gen-ai/agentic-memories)

---

## Architecture Options

> **Important distinction:** Foundry IQ (indexed knowledge base via Azure AI Search) and the SharePoint tool (live queries via Copilot Retrieval API) are **different mechanisms**. See the [Deep Dive section](#deep-dive-foundry-iq-vs-sharepoint-tool) for details.

---

## Decision Tree

```
START: Do you need a SharePoint Q&A agent?
  |
  +-- Do users already have M365 Copilot licenses?
  |    YES --> Do you need heavy customization?
  |    |         NO  --> Option C: M365 Copilot Declarative Agent (zero-code)
  |    |         YES --> Option A: Foundry Agent + Foundry IQ + Teams
  |    NO  --> Continue
  |
  +-- Is your team technical (developers available)?
  |    NO  --> Option B: Copilot Studio (low-code, ~$200/mo per tenant)
  |    YES --> Continue
  |
  +-- Do you need a custom web app (not just Teams)?
  |    NO  --> Option A: Foundry Agent + Foundry IQ + Teams
  |    YES --> Do you need multi-cloud portability?
  |              YES --> Option D: Custom RAG with Semantic Kernel
  |              NO  --> Option E: Hybrid (Foundry backend + custom frontend)
```

---

### Option A -- Foundry Agent + Foundry IQ + Teams (Recommended Start)

```
+---------------------+     +----------------------+
|   Microsoft Teams    |     |   Custom Web App     |
|   (M365 Agents SDK)  |     |   (React/Blazor)     |
+----------+----------+     +----------+-----------+
           |                           |
           v                           v
+----------------------------------------------------+
|              Azure Bot Service                      |
|         (Channel routing & identity)                |
+----------------------+-----------------------------+
                       |
                       v
+----------------------------------------------------+
|          Foundry Agent Service                      |
|    +--------------------------------------+        |
|    |  System Prompt + Tool Orchestration   |        |
|    |  * knowledge_base_retrieve (MCP)      |        |
|    |  * Custom functions (optional)        |        |
|    +--------------------------------------+        |
+----------------------+-----------------------------+
                       |
          +------------+------------+
          v            v            v
+--------------+ +-----------+ +------------------+
|  Foundry IQ  | | Cosmos DB | |  Azure OpenAI    |
|  Knowledge   | | (optional | |  (GPT-5.1 model) |
|  Base        | |  history) | |                  |
+------+-------+ +-----------+ +------------------+
       |
       v
+----------------------------------------------------+
|              Azure AI Search                        |
|  +------------+  +--------------+  +------------+ |
|  | SharePoint |  | Azure Blob   |  | OneLake    | |
|  | Sites/Docs |  | Storage      |  | (optional) | |
|  +------------+  +--------------+  +------------+ |
+----------------------------------------------------+
```

**Why this is recommended as starting point:**
- (+) Foundry IQ handles chunking, embedding, indexing, and retrieval
- (+) SharePoint ACL enforcement via indexed ACLs or identity passthrough
- (+) Publish to Teams via Foundry portal (requires app manifest + admin approval)
- (+) Can also serve a web app via Bot Service Direct Line channel
- (+) Pay-as-you-go, no platform fee for Foundry Agent Service itself
- (+) Foundry Agent Service includes built-in thread/conversation management

> **Caveats (verified May 2026):**
> - Published agents currently do NOT support citations in Teams (Preview limitation)
> - The SharePoint tool (Copilot Retrieval API) does NOT work when agent is published to Teams; use Foundry IQ indexed knowledge sources instead
> - Foundry Agent Service does not support per-request headers for MCP tools (Preview limitation); for per-user auth, consider Azure OpenAI Responses API

**Key components:**
| Component | Purpose |
|---|---|
| Foundry Agent Service | Agent runtime & orchestration |
| Foundry IQ Knowledge Base | Multi-source indexed retrieval over SharePoint |
| Azure AI Search | Index storage, hybrid search, ACL enforcement |
| Azure OpenAI (GPT-5.1) | LLM for response generation |
| Azure Bot Service | Channel bridge (Teams + Web) |
| M365 Agents SDK | Teams-native bot framework |
| Cosmos DB (optional) | Persistent conversation history & Q&A analytics (add when needed) |
| Entra ID | Authentication & identity passthrough |

---

### Option B -- Copilot Studio (Low-Code)

```
+---------------------+     +---------------------+
|   Microsoft Teams    |     |   Web Widget         |
+----------+----------+     +----------+----------+
           |                           |
           v                           v
+----------------------------------------------------+
|              Copilot Studio                         |
|    +--------------------------------------+        |
|    |  Generative Answers (SharePoint)     |        |
|    |  * Built-in SharePoint grounding     |        |
|    |  * Custom topics & triggers          |        |
|    |  * Power Automate integration        |        |
|    +--------------------------------------+        |
+----------------------------------------------------+
```

**Pros:**
- (+) No code required -- drag-and-drop bot builder
- (+) Native SharePoint grounding with generative answers
- (+) Built-in Teams and web widget deployment
- (+) ~$200/month per tenant (Microsoft Copilot Studio license)
- (+) Built-in conversation analytics

**Cons:**
- (-) Limited customization compared to Foundry
- (-) Less control over RAG pipeline and prompt engineering
- (-) Cannot build complex multi-agent workflows
- (-) Dependent on Copilot Studio feature roadmap

**Best for:** Non-technical teams, departmental Q&A bots, rapid prototyping

**Docs:** [learn.microsoft.com/microsoft-copilot-studio](https://learn.microsoft.com/en-us/microsoft-copilot-studio/)

---

### Option C -- M365 Copilot Declarative Agent (Zero-Code)

```
+----------------------------------------------------+
|         Microsoft 365 Copilot                       |
|    +--------------------------------------+        |
|    |  Declarative Agent (JSON manifest)   |        |
|    |  * SharePoint as knowledge source    |        |
|    |  * Graph Connectors                  |        |
|    |  * No custom infrastructure          |        |
|    +--------------------------------------+        |
|    Available in: Teams | Outlook | Copilot          |
+----------------------------------------------------+
```

**Pros:**
- (+) Zero custom code -- JSON manifest only
- (+) Native permission enforcement via M365 Copilot
- (+) Available across Teams, Outlook, and M365 Copilot
- (+) No Azure infrastructure to manage

**Cons:**
- (-) Requires M365 Copilot license for ALL users (~$30/user/month)
- (-) Least customizable option
- (-) Cannot add custom backends, analytics, or caching
- (-) Limited to M365 Copilot knowledge sources (not interoperable with Foundry IQ)

**Best for:** Organizations already licensing M365 Copilot for all users

**Docs:** [learn.microsoft.com/microsoft-365-copilot/extensibility](https://learn.microsoft.com/en-us/microsoft-365-copilot/extensibility/)

> **Note:** Copilot knowledge sources and Foundry IQ knowledge sources are NOT interoperable. You cannot use Foundry IQ sources in M365 Copilot, and vice versa. ([Source](https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/foundry-iq-faq))

---

### Option D -- Custom Web App with RAG Pipeline

```
+-----------------------------------------+
|         Custom Web App (React/Blazor)    |
|         Chat UI + History Panel          |
+------------------+----------------------+
                   |
                   v
+-----------------------------------------+
|   Backend API (FastAPI / .NET / Node)    |
|   +-----------------------------------+ |
|   |  Orchestration Layer               | |
|   |  * Query -> Retrieve -> Generate    | |
|   |  * History management              | |
|   |  * Citation tracking               | |
|   +-----------------------------------+ |
+----------+----------+-------------------+
           |          |
    +------v------+   |
    | Azure AI    |   |
    | Search      |   |
    | (SharePoint |   |
    |  indexed)   |   |
    +-------------+   |
                      v
              +---------------+
              | Azure OpenAI  |
              | + Cosmos DB   |
              +---------------+
```

**Pros:**
- (+) Full control over UI/UX and agent behavior
- (+) Can build Q&A caching and analytics from past conversations
- (+) No M365 Copilot license required (use Microsoft Graph API directly)
- (+) Suitable for external-facing scenarios
- (+) Can use Semantic Kernel for multi-cloud portability (Azure, AWS, on-prem)

**Cons:**
- (-) Must build and maintain the RAG pipeline (chunking, indexing, retrieval, prompt engineering)
- (-) Must implement SharePoint ACL enforcement manually
- (-) Higher development and maintenance cost
- (-) No out-of-box Teams integration

**Alternative orchestrators:** Semantic Kernel (open-source, runs anywhere) or LangChain can replace Foundry Agent Service for teams wanting cloud portability.

**Reference sample:** [github.com/pkumar26/sharepoint-foundryagent](https://github.com/pkumar26/sharepoint-foundryagent)

---

### Option E -- Hybrid: Foundry Backend + Custom Web Frontend

```
+----------------------+    +---------------------+
|  Custom Web App      |    |  Microsoft Teams     |
|  (Full-featured UI)  |    |  (via Bot Service)   |
+----------+-----------+    +----------+----------+
           |                           |
           v                           v
+----------------------------------------------------+
|    Azure Container Apps / App Service               |
|    +----------------------------------------------+|
|    |  Custom Backend (Python/C#)                   ||
|    |  * Foundry Agent SDK client                   ||
|    |  * Custom history management                  ||
|    |  * Q&A caching & analytics (feedback-driven)    ||
|    |  * Web API endpoints                          ||
|    +----------------------------------------------+|
+----------------------+-----------------------------+
                       |
           +-----------+-----------+
           v           v           v
    +------------+ +---------+ +----------+
    | Foundry    | |Cosmos DB| | Azure    |
    | Agent +    | |(History | | OpenAI   |
    | Foundry IQ | |+ Q&A KB)| |          |
    +------------+ +---------+ +----------+
```

**Why consider this:**
- (+) Best of both worlds -- Foundry IQ handles SharePoint retrieval, custom backend adds features
- (+) Full control over the web experience
- (+) Can implement Q&A caching: store Q&A pairs, surface popular questions, reduce model calls
- (+) Can integrate Teams AND a rich web app simultaneously
- (+) Conversation history with full customization

> **Important:** Only pursue this option when Option A has been validated with real users and feedback shows need for custom UI or analytics. Do NOT start here.

**This is ideal if you need:**
- A branded, feature-rich web portal (not just a chat widget)
- Q&A analytics dashboard with usage patterns and content gap analysis
- Admin dashboard with feedback review (thumbs up/down, corrections)
- Custom feedback mechanisms driving prompt refinement

**Hosting choices (evaluate based on your needs):**
- **Azure Functions** -- best for event-driven, low-traffic agents (lowest cost)
- **Azure App Service** -- best for always-on web apps with built-in auth
- **Azure Container Apps** -- best if orchestrating multiple microservices

---

## Comparison Matrix

| Criteria | Option A (Foundry+IQ) | Option B (Copilot Studio) | Option C (Declarative Agent) | Option D (Custom RAG) | Option E (Hybrid) |
|---|---|---|---|---|---|
| **Development effort** | Low | Very Low | Minimal | High | Medium |
| **Maintenance burden** | Low | Very Low | Minimal | High | Medium |
| **SharePoint ACL enforcement** | Via indexed ACLs | Built-in | Built-in | Manual | Via indexed ACLs |
| **Citation support** | In playground (not in published Teams agents) | Built-in | Built-in | Custom | Custom |
| **Teams integration** | Via publish (with caveats) | Native | Native | Custom | Both |
| **Custom web app** | Via Direct Line | Web widget only | No | Full control | Full control |
| **Conversation history** | Built-in threads (basic) | Built-in | Via Copilot | Full control | Full control |
| **Q&A caching & analytics** | Limited | Built-in analytics | Limited | Full control | Full control |
| **Multi-source support** | Built-in (SP, Blob, OneLake) | SharePoint + Dataverse | M365 data | Custom | Built-in |
| **Licensing** | Azure pay-as-you-go (+ Copilot for remote SP) | ~$200/mo tenant | M365 Copilot (~$30/user/mo) | Graph API only | Azure pay-as-you-go |
| **Time to PoC** | 1-2 weeks | Days | Hours | Months | 3-6 weeks |
| **Time to production** | 4-8 weeks | 2-4 weeks | 1-2 weeks | 3-6 months | 2-4 months |
| **Team skills needed** | Azure + Python/C# | Low-code (Power Platform) | JSON config only | Full-stack + Azure + AI | Full-stack + Azure + AI |
| **Complexity** | Low-Medium | Low | Very Low | High | Medium-High |
| **Risk level** | Low (Preview caveats) | Low | Low | Medium (custom maintenance) | Medium (more moving parts) |
| **Cloud portability** | Azure only | Microsoft only | Microsoft only | Any (with Semantic Kernel) | Azure only |

---

## Deep Dive: Foundry IQ vs SharePoint Tool (Key Distinction)

> **Critical:** These are **two different mechanisms** for accessing SharePoint from Foundry agents. Choosing the wrong one leads to broken architectures.

### Foundry IQ (Indexed Knowledge Base)
- Uses **Azure AI Search** to crawl, chunk, embed, and index SharePoint content
- Supports **hybrid search** (keyword + vector) across indexed content
- ACLs can be synchronized via index metadata fields; queries filtered by user identity
- Content is pre-indexed -- queries are fast but data has indexing lag
- Works with published agents (including Teams-published agents)
- Exposes `knowledge_base_retrieve` MCP tool
- **Docs:** [learn.microsoft.com/azure/foundry/agents/how-to/foundry-iq-connect](https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/foundry-iq-connect)

### SharePoint Tool (Copilot Retrieval API)
- Queries SharePoint **in real-time** via the M365 Copilot Retrieval API
- No pre-indexing needed -- always reflects latest content
- Permissions enforced natively by SharePoint via identity passthrough (OBO)
- Requires M365 Copilot license OR pay-as-you-go ($0.10/API call, Preview)
- **Does NOT work when agent is published to Teams** (verified limitation)
- Best for development/playground scenarios or non-Teams deployments
- **Docs:** [learn.microsoft.com/azure/foundry/agents/how-to/tools/sharepoint](https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/tools/sharepoint)

### Which to Use?

| Scenario | Use This |
|---|---|
| Agent published to Teams | Foundry IQ (indexed) -- SharePoint tool does NOT work in Teams |
| Need real-time data freshness | SharePoint tool -- no indexing lag |
| Large-scale retrieval (100K+ docs) | Foundry IQ -- pre-indexed for speed |
| No M365 Copilot licenses | Foundry IQ (indexed) -- only Azure AI Search costs |
| Per-user permissions required | Both support this, but with different mechanisms |

### Foundry IQ Capabilities
1. **Automated document processing** -- chunking, vector embedding, metadata extraction
2. **Multi-source knowledge bases** -- connect multiple SharePoint sites + other sources
3. **Agentic retrieval** -- LLM-driven query planning, decomposition, parallel search, result aggregation
4. **Permission enforcement** -- ACLs synchronized, Purview labels respected
5. **Citations** -- extractive answers traced back to source documents (in playground; not yet in published agents)

### SharePoint as a Knowledge Source
Two approaches:

| Approach | How it works | Licensing |
|---|---|---|
| **Indexed** | Azure AI Search crawls and indexes SharePoint content. Supports hybrid search (keyword + vector). ACLs synced via index metadata. | Standard Azure AI Search pricing |
| **Remote (live)** | Queries SharePoint in real-time via Copilot Retrieval API. No pre-indexing needed. Permissions enforced natively by SharePoint. | M365 Copilot license OR pay-as-you-go ($0.10/API call, Preview) |

### MCP Tool Integration
Foundry IQ exposes the `knowledge_base_retrieve` MCP tool. Agents call this tool to:
- Plan and decompose complex queries
- Search across multiple knowledge sources in parallel
- Return grounded answers with document citations

```python
# Example (illustrative): Create agent with Foundry IQ knowledge base
# `knowledge_base_mcp_tool` is a placeholder for the configured MCP tool
# definition that points to your Foundry IQ knowledge base.
from azure.ai.projects import AIProjectClient

client = AIProjectClient(endpoint="...", credential=credential)
agent = client.agents.create_agent(
    model="gpt-5.1",  # deployment name; GPT-5 series requires Azure OpenAI v1 API
    instructions="Use the knowledge base tool to answer user questions...",
    tools=[knowledge_base_mcp_tool],
)
```

### Relationship to Other IQ Services
| IQ Service | Scope | Data Sources |
|---|---|---|
| **Foundry IQ** | Enterprise knowledge retrieval | Azure, SharePoint, OneLake, web |
| **Fabric IQ** | Business analytics & semantic models | OneLake, Power BI |
| **Work IQ** | M365 collaboration signals | Outlook, Teams, Calendar |

> **Note:** These are standalone but complementary. You can use Foundry IQ for document Q&A and Work IQ for collaboration context.

---

## Deep Dive: SharePoint Knowledge Agent (AI in SharePoint)

The **AI in SharePoint** feature (previously called "SharePoint Knowledge Agent") is a separate but complementary feature that makes your SharePoint content **AI-ready**:

### Key Capabilities (Preview -- GA date not yet announced)
- **Auto-tagging** -- automatically classifies documents and extracts metadata (dates, names, contract numbers)
- **Content hygiene** -- detects stale pages, broken links, duplicates, inconsistent metadata
- **Natural language automation** -- "notify legal when contracts expire in 90 days" -> auto-creates workflow
- **Role-based actions** -- adapts suggestions by role (content manager, site owner, viewer)

### Why It Matters for Your Solution
A Foundry IQ + Foundry Agent solution is only as good as the SharePoint content it retrieves from. The Knowledge Agent ensures:
- Documents are properly tagged and structured -> better retrieval relevance
- Stale content is flagged and cleaned -> fewer inaccurate answers
- Metadata is consistent -> improved filtering and search precision

**Recommendation:** Enable AI in SharePoint alongside your Foundry solution for maximum answer quality.

---

## Deep Dive: Teams & Bot Integration

### SDK Landscape (2026)

| SDK | Best For | Status |
|---|---|---|
| **Teams SDK (Teams AI Library)** | Teams-only bots and apps | Current, recommended for simple Teams bots |
| **M365 Agents SDK** | Multi-channel (Teams, Outlook, Copilot) | Current, replaces Bot Framework v4 |
| **Foundry Agent SDK** | Advanced multi-agent orchestration | Current, for complex AI workflows |
| **Bot Framework v4** | Legacy | Archived -- support ended Dec 31, 2025 ([migration guide](https://learn.microsoft.com/en-us/microsoft-365/agents-sdk/bf-migration-guidance)) |
| **Copilot Studio** | Low-code bots and agents | Current, best for non-technical teams |

### Publishing Foundry Agents to Teams
1. Build agent in Foundry portal or via SDK
2. Foundry generates a stable endpoint + Managed Identity
3. Publish via Foundry UI -> creates Azure Bot Service resource
4. Configure Teams app manifest and permissions
5. Upload to Teams admin center for approval and distribution

> **Limitation (Preview):** Published agents do not currently support citations, streaming responses, or certain tool types. Test thoroughly before production deployment. ([Source](https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/publish-copilot))

### Architecture Flow
```
User -> Teams -> Azure Bot Service -> Foundry Agent -> Foundry IQ -> SharePoint
                                                  -> Cosmos DB (history)
                                                  -> Azure OpenAI (generation)
```

---

## Conversation History & Q&A Analytics

> **Clarification:** This section describes Q&A caching and analytics, not machine learning or model fine-tuning. The LLM does not "learn" from past conversations. What improves over time is: cached answer speed, prompt engineering based on feedback analysis, and content gap identification.

### Built-in vs Custom History

| Approach | What It Provides | When to Use |
|---|---|---|
| **Foundry Agent Service threads** | Built-in conversation threads per session | Sufficient for basic follow-up questions within a session |
| **Cosmos DB (custom)** | Persistent cross-session history, Q&A caching, feedback storage, analytics | When you need long-term analytics, cross-session context, or custom dashboards |

> **Recommendation:** Start with Foundry's built-in thread management. Add Cosmos DB only if you need persistent analytics, cross-session Q&A caching, or feedback collection.

### Architecture for Persistent Memory

```
+---------------------------------------------+
|              Azure Cosmos DB                 |
|  +---------------+  +---------------------+|
|  |  Short-Term    |  |  Long-Term Memory   ||
|  |  (last 10      |  |  * Q&A pairs        ||
|  |   turns, TTL)  |  |  * User preferences ||
|  |               |  |  * Popular questions ||
|  +---------------+  |  * Feedback data     ||
|                     +---------------------+|
+---------------------------------------------+
```

### Data Model
```json
{
  "id": "msg-uuid",
  "conversationId": "conv-abc123",
  "userId": "user@company.com",
  "role": "user|assistant|system",
  "timestamp": "2026-04-09T14:00:00Z",
  "content": "What is the latest HR policy on hybrid work?",
  "citations": [
    { "title": "HR Policy 2026", "url": "https://sp.company.com/...", "snippet": "..." }
  ],
  "feedback": { "rating": "thumbs_up", "correction": null },
  "metadata": { "model": "gpt-5.1", "tokens_used": 1250 }
}
```

### Q&A Caching & Analytics Design
1. **Store Q&A pairs** with citations and feedback ratings
2. **Popular questions surfacing** -- aggregate frequently asked questions -> proactive content or FAQ pages
3. **Feedback integration** -- thumbs up/down drives prompt refinement by developers
4. **Q&A cache** -- high-confidence cached answers reduce model calls and response latency
5. **Content gap analytics** -- track what users ask but the agent can't answer -> feed back to content owners

```
User asks question
       |
       v
Check Q&A cache (Cosmos DB) ---> If high-confidence match -> Return cached answer
       | (miss)
       v
Foundry IQ retrieval -> Azure OpenAI -> Generate answer
       |
       v
Store Q&A pair -> Collect feedback -> Periodically review & curate
```

### Partition Strategy
- **Partition key:** `userId` (for per-user queries) or `conversationId` (for shared bots)
- **Indexes:** vector index on embeddings for semantic search over history
- **TTL:** Short-term context (24h), Long-term Q&A pairs (indefinite)

---

## Security & Compliance

| Layer | Mechanism | Details |
|---|---|---|
| **Identity** | Microsoft Entra ID | SSO, MFA, conditional access |
| **SharePoint permissions** | ACL sync / Identity passthrough | Users see only authorized content |
| **Data residency** | Azure region selection | All processing stays in your tenant |
| **Sensitivity labels** | Microsoft Purview | Enforced at query time by Foundry IQ |
| **Audit trail** | Cosmos DB + Azure Monitor | Full conversation logging |
| **Bot security** | Managed Identity | No stored credentials |
| **Compliance** | SOC 2, ISO 27001, GDPR | Inherited from Azure platform |

### Key Requirement: OBO (On-Behalf-Of) Flow
For SharePoint content retrieval to respect per-user permissions, the agent must use **identity passthrough** (not app-only tokens). This ensures:
- Each user's query is executed under their identity
- SharePoint ACLs are enforced in real-time
- No data leakage between users with different permission levels

---

## Pricing & Cost Considerations

### Cost Components

> **Note:** Pricing is region-dependent and subject to change. Values below are approximate. Always verify at [azure.microsoft.com/pricing](https://azure.microsoft.com/en-us/pricing/).

| Component | Pricing Model | Notes |
|---|---|---|
| **Foundry Agent Service** | No platform fee; pay for consumed resources | You pay for model tokens, tools, and connected services |
| **Azure OpenAI (GPT-5.1)** | Per 1K tokens (input/output) | Varies by model and region; see [Azure OpenAI pricing](https://azure.microsoft.com/en-us/pricing/details/cognitive-services/openai-service/) |
| **Azure AI Search** | Tier-based (Free -> S3 HD) | Free tier for PoC (50MB); Standard tiers per Search Unit, region-dependent; see [pricing page](https://azure.microsoft.com/en-us/pricing/details/search/) |
| **Foundry file search storage** | Per GB/day of vector storage (first 1GB free) | Listed under "Tools" on Foundry pricing page, not Foundry IQ specifically |
| **Cosmos DB** | RU/s + storage | Free Tier: 1000 RU/s + 25 GB (1 account per subscription, lifetime); see [free tier details](https://learn.microsoft.com/en-us/azure/cosmos-db/free-tier) |
| **Azure Bot Service** | Free for standard channels | $0 for Teams channel |
| **M365 Copilot license** | Per-user/month | Required for remote SP queries via Copilot Retrieval API; OR use pay-as-you-go at $0.10/API call (Preview) |
| **Copilot Studio** | Per-tenant/month | ~$200/month for tenant license (alternative to Foundry for low-code) |
| **Azure hosting** | Varies by service | Functions (consumption) vs App Service vs Container Apps |

### Cost Optimization Tips
1. **Start with Free tiers** -- Azure AI Search Free, Cosmos DB Free Tier (1000 RU/s + 25GB), Foundry IQ free token allocation
2. **Use GPT-5-mini** for simpler queries, GPT-5.1 for complex ones (note: GPT-5.1 defaults `reasoning_effort` to `none`; raise it for harder questions)
3. **Cache frequent Q&A** in Cosmos DB to reduce model calls
4. **Use indexed SharePoint** (via Foundry IQ) to avoid per-user Copilot license requirements
5. **Consider pay-as-you-go** for Copilot Retrieval API ($0.10/call, Preview) if user volume is low
6. **Monitor with Azure Cost Management** -- set budgets and alerts
7. **Use the FastTrack Cost Calculator:** [github.com/microsoft/FastTrack/copilot-agents-cost-tool](https://github.com/microsoft/FastTrack/tree/master/copilot-agent-strategy/copilot-agents-cost-tool)

### Scalability Considerations
- **Azure AI Search Free tier:** 50 MB storage -- PoC only
- **Azure AI Search S1:** ~25 GB storage per Search Unit -- sufficient for most mid-size deployments
- **100K+ documents:** May require S2/S3 tiers; indexing can take hours; plan for incremental indexing via scheduled indexer runs
- **Azure OpenAI TPM limits:** Configure request throttling and queuing for burst scenarios
- **Cosmos DB:** 1000 RU/s supports ~100 concurrent users; scale up for larger deployments

---

## Known Limitations & Preview Caveats

> **As of May 2026, several components are in Preview. Review these limitations before committing to an architecture.**

| Limitation | Impact | Source |
|---|---|---|
| **SharePoint tool does NOT work in Teams-published agents** | Must use Foundry IQ (indexed) for Teams deployments, not the SharePoint tool | [MS Learn](https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/tools/sharepoint) |
| **Published agents do not support citations** | Citations work in playground/dev but not in production Teams agents | [MS Learn](https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/publish-copilot) |
| **Foundry Agent Service does not support per-request MCP headers** | Per-user auth for MCP tools requires Azure OpenAI Responses API instead | [MS Learn](https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/foundry-iq-connect) |
| **Foundry IQ is in Preview** | Not recommended for production workloads without SLA | [MS Learn](https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/what-is-foundry-iq) |
| **AI in SharePoint is in Preview** | GA date not announced; features may change | [MS Learn](https://learn.microsoft.com/en-us/sharepoint/ai-in-sharepoint-get-started) |
| **Copilot Retrieval API pay-as-you-go is in Preview** | Pricing ($0.10/call) may change at GA | [MS Learn](https://learn.microsoft.com/en-us/microsoft-365-copilot/extensibility/api/ai-services/retrieval/paygo-retrieval) |
| **Foundry IQ and M365 Copilot knowledge sources are NOT interoperable** | Cannot mix knowledge sources across platforms | [MS Learn](https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/foundry-iq-faq) |
| **SharePoint content indexing has lag** | Recently added content needs time to appear in Microsoft Search and Azure AI Search indexes | [MS Learn](https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/tools/sharepoint) |

---

## Anti-Patterns: What NOT to Do

- **Do NOT start with Option E (Hybrid)** if Option A suffices -- validate with real users first, then evolve
- **Do NOT add Cosmos DB** unless you have specific needs beyond Foundry's built-in thread management
- **Do NOT call Q&A caching "AI learning"** -- the model does not fine-tune or adapt from past conversations
- **Do NOT use the SharePoint tool** if publishing to Teams -- it won't work; use Foundry IQ indexed sources
- **Do NOT use Bot Framework v4** for new projects -- it's archived; use M365 Agents SDK
- **Do NOT assume "fully managed"** means zero configuration -- you still manage Azure AI Search, OpenAI quotas, Entra ID, and Bot Service
- **Do NOT skip security testing** -- test prompt injection defenses, verify ACL enforcement per user, and validate OBO token handling
- **Do NOT ignore content quality** -- enable AI in SharePoint to clean up stale/broken content before indexing

---

## Reference Implementations & Samples

| Sample | Description | Link |
|---|---|---|
| **microsoft/app-with-sharepoint-knowledge** | Official MS sample: web app + Foundry agent + SharePoint via Copilot Retrieval API | [GitHub](https://github.com/microsoft/app-with-sharepoint-knowledge) |
| **pkumar26/sharepoint-foundryagent** | Document Q&A Agent with FastAPI, Azure OpenAI, Cosmos DB | [GitHub](https://github.com/pkumar26/sharepoint-foundryagent) |
| **azure-search-openai-demo** | Full RAG solution with Azure AI Search + OpenAI (adaptable to SharePoint) | [GitHub](https://github.com/Azure-Samples/azure-search-openai-demo) |
| **adhazel/ms365_agents_sdk_with_foundry_agent_sample** | Foundry Agent + M365 Agents SDK for Teams with proactive messaging | [GitHub](https://github.com/adhazel/ms365_agents_sdk_with_foundry_agent_sample) |
| **Origin Digital/Microsoft-Foundry-SharePoint** | Step-by-step SharePoint + Foundry integration | [GitHub](https://github.com/Origin-Digital-LLC/Microsoft-Foundry-SharePoint) |
| **Copilot Camp Lab BMA3** | Lab: Integrate Foundry Agent with M365 Agents SDK | [Lab](https://microsoft.github.io/copilot-camp/pages/custom-engine/agents-sdk/03-agent-configuration/) |

### Official Documentation
- [Foundry IQ Overview](https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/what-is-foundry-iq)
- [SharePoint as Knowledge Source](https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/tools/sharepoint)
- [Connect Foundry IQ to Agent Service](https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/foundry-iq-connect)
- [Publish Agent to Teams](https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/publish-copilot)
- [Foundry IQ FAQ](https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/foundry-iq-faq)
- [Azure Cosmos DB Agent Memory](https://learn.microsoft.com/en-us/azure/cosmos-db/gen-ai/agentic-memories)
- [Foundry Agent Service Pricing](https://azure.microsoft.com/en-us/pricing/details/foundry-agent-service/)

---

## Final Recommendation

### Recommended: Progressive Enhancement (Start with Option A, Evolve as Needed)

For most organizations, the best strategy is **start simple, validate with users, then evolve**:

```
+----------------------------------------------------------------+
|              PROGRESSIVE ENHANCEMENT STRATEGY                   |
+----------------------------------------------------------------+
|                                                                |
|  PHASE 1: VALIDATE (Option A -- 1-2 weeks to PoC)             |
|  +-- Foundry Agent Service + Foundry IQ knowledge base        |
|  +-- Azure AI Search (indexed SharePoint content)             |
|  +-- Publish to Teams via Foundry portal                      |
|  +-- Test with pilot user group                               |
|  +-- Use Foundry's built-in thread management                 |
|                                                                |
|  PHASE 2: ENHANCE (Add based on real feedback)                 |
|  +-- IF users need custom web UI -> add web frontend          |
|  +-- IF need analytics -> add Cosmos DB for Q&A caching       |
|  +-- IF need feedback loop -> add thumbs up/down + dashboard  |
|  +-- IF non-technical teams need bots -> add Copilot Studio   |
|                                                                |
|  PHASE 3: SCALE (Option E -- when justified)                   |
|  +-- Custom backend (Azure Functions or App Service)          |
|  +-- Q&A caching engine for frequent questions                |
|  +-- Admin dashboard (usage analytics, content gaps)          |
|  +-- Enable AI in SharePoint for content hygiene              |
|  +-- Prompt engineering refinement based on feedback data     |
|                                                                |
|  ALTERNATIVE PATHS                                             |
|  +-- No developers? -> Option B (Copilot Studio)              |
|  +-- All users have M365 Copilot? -> Option C (Declarative)  |
|  +-- Need multi-cloud? -> Option D (Semantic Kernel)          |
|                                                                |
+----------------------------------------------------------------+
```

### Why NOT start with the Hybrid (Option E)?

| Concern | Reality |
|---|---|
| "We need a custom web app" | Validate in Teams first -- most users are already there |
| "We need learning from Q&A" | This is Q&A caching, not ML. Add it later with Cosmos DB when you have usage data |
| "We need full control" | Foundry IQ handles 80% of the work. Customize only what you must. |
| "We need analytics" | Start with Foundry's built-in metrics. Add Cosmos DB + dashboard in Phase 3 |

### Key Architecture Decisions Summary

| Decision | Recommendation | Rationale |
|---|---|---|
| SharePoint access method | Foundry IQ (indexed) for Teams; SharePoint tool for dev/testing | SharePoint tool doesn't work in Teams-published agents |
| Conversation history | Start with Foundry built-in threads | Add Cosmos DB only if you need cross-session persistence or analytics |
| Hosting (if custom backend) | Azure Functions (low traffic) or App Service (always-on) | Container Apps only if you need multi-service orchestration |
| Orchestrator | Foundry Agent Service (default) or Semantic Kernel (portability) | Foundry is simpler; SK is more portable |
| Bot framework | M365 Agents SDK | Bot Framework v4 is archived |
| Content readiness | Enable AI in SharePoint | Improves retrieval quality via auto-tagging and content hygiene |

### Implementation Roadmap

**Phase 1 -- Foundation & Validation (1-2 weeks PoC, 4-8 weeks production)**
- Set up Foundry project + Azure AI Search (Free tier for PoC)
- Connect SharePoint sites as Foundry IQ indexed knowledge source
- Create basic agent with system prompt + knowledge_base_retrieve MCP tool
- Test in Foundry playground; validate ACL enforcement per user
- Publish to Teams; test with pilot user group (5-10 users)
- Collect initial feedback on answer quality and relevance

**Phase 2 -- Feedback-Driven Enhancement**
- Based on pilot feedback, decide: Teams-only (stay Option A) or custom web (evolve to Option E)
- If needed: add Cosmos DB for conversation history persistence and Q&A caching
- If needed: build custom web frontend (React/Blazor) with Direct Line or custom API
- Enable AI in SharePoint for content hygiene and auto-tagging

**Phase 3 -- Scale & Optimize (if Option E)**
- Build Q&A caching engine: frequent questions -> cached responses -> reduced model calls
- Add admin dashboard: usage analytics, content gaps, feedback review
- Implement prompt engineering refinements based on feedback data
- Scale Azure AI Search tier based on content volume
- Set up Azure Monitor + Application Insights for observability
- Implement rate limiting and error handling (circuit breakers, retry policies, graceful fallbacks)

---

### Reviewer Credits

This document was peer-reviewed by 3 independent AI models:
- **GPT-5.2** -- Fact-checked all claims against official Microsoft documentation (176 tool calls)
- **Claude Sonnet 4.5** -- Challenged architecture decisions, identified missing alternatives
- **Claude Sonnet 4** -- Reviewed completeness, customer-readiness, and operational gaps

Key corrections applied:
- Fixed: SharePoint tool does NOT work in Teams-published agents (must use Foundry IQ indexed)
- Fixed: Published agents do NOT support citations (Preview limitation)
- Fixed: Renamed "learning" to "Q&A caching & analytics" (no actual model fine-tuning occurs)
- Added: Copilot Studio and M365 Copilot Declarative Agent as missing alternatives
- Added: Decision tree, prerequisites, known limitations, anti-patterns sections
- Changed: Recommendation from "start with Hybrid" to "start simple (Option A), evolve as needed"
- Corrected: Pricing figures to reference official sources rather than potentially stale numbers
- Clarified: Foundry IQ vs SharePoint tool are different mechanisms with different limitations

---

*This research is based on publicly available official Microsoft documentation (learn.microsoft.com), verified GitHub repositories, and Microsoft community resources as of May 2026. Features marked as Preview may change before General Availability. Always verify current pricing and feature availability on the official Azure and Microsoft 365 pricing pages.*

---

## References

### Official Microsoft Documentation (learn.microsoft.com)

1. **Foundry Agent Service -- Overview**  
   https://learn.microsoft.com/en-us/azure/foundry/agents/

2. **Foundry IQ -- What is Foundry IQ (Preview)**  
   https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/what-is-foundry-iq

3. **Foundry IQ -- Frequently Asked Questions**  
   https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/foundry-iq-faq

4. **Connect a Foundry IQ Knowledge Base to Foundry Agent Service**  
   https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/foundry-iq-connect

5. **Use SharePoint Tool with the Agent API (Preview)**  
   https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/tools/sharepoint

6. **Publish Agents to Microsoft 365 Copilot and Microsoft Teams**  
   https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/publish-copilot

7. **Azure AI Search -- Overview**  
   https://learn.microsoft.com/en-us/azure/search/

8. **Azure AI Search -- Create a Knowledge Base (Agentic Retrieval)**  
   https://learn.microsoft.com/en-us/azure/search/agentic-retrieval-how-to-create-knowledge-base

9. **Azure AI Search -- Remote SharePoint Knowledge Source**  
   https://learn.microsoft.com/en-us/azure/search/agentic-knowledge-source-how-to-sharepoint-remote

10. **Azure AI Search -- Document-Level Access Control Overview**  
    https://learn.microsoft.com/en-us/azure/search/search-document-level-access-overview

11. **AI in SharePoint (formerly Knowledge Agent) -- Get Started**  
    https://learn.microsoft.com/en-us/sharepoint/ai-in-sharepoint-get-started

12. **Azure Cosmos DB -- Agent Memory Patterns for NoSQL**  
    https://learn.microsoft.com/en-us/azure/cosmos-db/gen-ai/agentic-memories

13. **Azure Cosmos DB -- Free Tier**  
    https://learn.microsoft.com/en-us/azure/cosmos-db/free-tier

14. **M365 Agents SDK -- Bot Framework Migration Guidance**  
    https://learn.microsoft.com/en-us/microsoft-365/agents-sdk/bf-migration-guidance

15. **Azure Bot Service -- Overview**  
    https://learn.microsoft.com/en-us/azure/bot-service/bot-service-overview

16. **Microsoft Copilot Studio -- Overview**
    https://learn.microsoft.com/en-us/microsoft-copilot-studio/

17. **Microsoft 365 Copilot -- Extensibility Overview**  
    https://learn.microsoft.com/en-us/microsoft-365-copilot/extensibility/

18. **Microsoft 365 Copilot -- Retrieval API Overview**  
    https://learn.microsoft.com/en-us/microsoft-365-copilot/extensibility/api-reference/retrieval-api-overview

19. **Microsoft 365 Copilot -- Pay-as-You-Go Retrieval (Preview)**  
    https://learn.microsoft.com/en-us/microsoft-365-copilot/extensibility/api/ai-services/retrieval/paygo-retrieval

20. **Semantic Indexing for Microsoft 365 Copilot**  
    https://learn.microsoft.com/en-us/microsoftsearch/semantic-index-for-copilot

21. **Teams SDK (Teams AI Library) -- Overview**  
    https://learn.microsoft.com/en-us/microsoftteams/platform/teams-sdk/

22. **Azure OpenAI Responses API**  
    https://learn.microsoft.com/en-us/azure/ai-foundry/openai/how-to/responses

23. **SharePoint Developer -- GitHub Repositories**  
    https://learn.microsoft.com/en-us/sharepoint/dev/community/repositories

### Azure Pricing Pages

24. **Foundry Agent Service -- Pricing**  
    https://azure.microsoft.com/en-us/pricing/details/foundry-agent-service/

25. **Azure OpenAI Service -- Pricing**  
    https://azure.microsoft.com/en-us/pricing/details/cognitive-services/openai-service/

26. **Azure AI Search -- Pricing**  
    https://azure.microsoft.com/en-us/pricing/details/search/

27. **Azure Pricing Calculator**  
    https://azure.microsoft.com/en-us/pricing/

28. **Azure AI Foundry Pricing Guide (PDF)**  
    https://cdn-dynmedia-1.microsoft.com/is/content/microsoftcorp/microsoft/final/en-us/microsoft-product-and-services/azure/pdf/ms-azure-ai-foundry-pricing-guide-e-book-final.pdf

### GitHub Reference Implementations & Samples

29. **microsoft/app-with-sharepoint-knowledge** -- Official MS sample: web app + Foundry agent + SharePoint via Copilot Retrieval API  
    https://github.com/microsoft/app-with-sharepoint-knowledge

30. **pkumar26/sharepoint-foundryagent** -- Document Q&A Agent with FastAPI, Azure OpenAI, Cosmos DB  
    https://github.com/pkumar26/sharepoint-foundryagent

31. **Azure-Samples/azure-search-openai-demo** -- Full RAG solution with Azure AI Search + OpenAI  
    https://github.com/Azure-Samples/azure-search-openai-demo

32. **adhazel/ms365_agents_sdk_with_foundry_agent_sample** -- Foundry Agent + M365 Agents SDK for Teams with proactive messaging  
    https://github.com/adhazel/ms365_agents_sdk_with_foundry_agent_sample

33. **Origin-Digital-LLC/Microsoft-Foundry-SharePoint** -- Step-by-step SharePoint + Foundry integration  
    https://github.com/Origin-Digital-LLC/Microsoft-Foundry-SharePoint

34. **microsoft/FastTrack -- Copilot Agents Cost Tool** -- Cost estimation calculator for agent deployments  
    https://github.com/microsoft/FastTrack/tree/master/copilot-agent-strategy/copilot-agents-cost-tool

35. **AzureCosmosDB/cosmosdb-chat-history-langchaingo** -- Chat history implementation with Cosmos DB  
    https://github.com/AzureCosmosDB/cosmosdb-chat-history-langchaingo

### Microsoft Tech Community & Blog Posts

36. **Introducing Knowledge Agent in SharePoint** -- Official SharePoint Blog announcement  
    https://techcommunity.microsoft.com/blog/spblog/introducing-knowledge-agent-in-sharepoint/4454154

37. **Native Microsoft Agent 365 Integration in Microsoft Foundry** -- Azure AI Foundry Blog  
    https://techcommunity.microsoft.com/blog/azure-ai-foundry-blog/native-microsoft-agent-365-integration-in-microsoft-foundry/4471186

### Labs & Tutorials

38. **Copilot Camp -- Lab BMA3: Integrate Foundry Agent with M365 Agents SDK**  
    https://microsoft.github.io/copilot-camp/pages/custom-engine/agents-sdk/03-agent-configuration/

39. **Azure Lessons -- Azure AI Foundry Deploy to Teams (Tutorial)**  
    https://azurelessons.com/azure-ai-foundry-deploy-to-teams/

40. **Azure Logic Apps Labs -- Implement a SharePoint RAG**  
    https://azure.github.io/logicapps-labs/docs/ai-workloads-on-logicapps/sharepoint-rag/

### Community & Third-Party Analysis

41. **Foundry IQ: The Knowledge Layer for Agents** -- Microsoft Ignite 2025 Session Summary (Technspire)  
    https://technspire.com/blog/ignite-2025-brk196-foundry-iq-knowledge-layer-agents

42. **Foundry IQ: The Ultimate Enterprise AI Knowledge Layer** -- The Power Platform Cave  
    https://www.thepowerplatformcave.com/foundry-iq-the-missing-link-in-your-enterprise-ai-architecture/

43. **Making SharePoint AI-Ready with Knowledge Agent** -- Nanddeep Nachan Blog  
    https://nanddeepnachanblogs.com/posts/2025-09-25-sharepoint-knowledge-agent/

44. **SharePoint Knowledge Agent Deep Dive** -- VladTalksTech  
    https://vladtalkstech.com/microsoft-365/sharepoint/sharepoint-knowledge-agent-deep-dive-how-it-works-setup-and-best-practices/

45. **Microsoft Teams Development SDK Guidance (Fall 2025)** -- Voitanos  
    https://www.voitanos.io/blog/microsoft-teams-sdk-evolution-2025/

46. **Copilot Agents: Agent Builder, Copilot Studio, or Foundry?** -- CloudBuild  
    https://cloudbuild.co.uk/copilot-agents-agent-builder-copilot-studio-or-foundry/

47. **Multi-Agent Orchestration: Copilot Studio vs Microsoft Foundry** -- Ragnar Heil  
    https://ragnarheil.de/multi-agent-orchestration-copilot-studio-vs-azure-ai-foundry-heres-what-you-actually-need-to-know/

48. **Building AI Agents: Choosing Between Copilot Studio and Azure AI Foundry** -- Emergent Software  
    https://www.emergentsoftware.net/blog/building-ai-agents-choosing-between-microsoft-copilot-studio-and-azure-ai-foundry/

49. **Azure AI Foundry Pricing Guide for Enterprises** -- AlRafay Global  
    https://alrafayglobal.com/azure-ai-foundry-pricing-what-enterprises-need-to-know-before-scaling-ai/

50. **How to Store Chat History Using External Storage in Microsoft Agent Framework** -- Will Velida (Dev.to)  
    https://dev.to/willvelida/how-to-store-chat-history-using-external-storage-in-microsoft-agent-framework-3io2

51. **How to Set Up SharePoint Agents Step-by-Step (2025 Edition)** -- SPKnowledge  
    https://spknowledge.com/2025/10/08/how-to-set-up-sharepoint-agents-step-by-step-2025-edition-complete-admin-guide/

52. **Licensing and Pricing for SharePoint Agents** -- SPKnowledge  
    https://spknowledge.com/2025/10/17/licensing-and-pricing-for-sharepoint-agents-microsoft-365-copilot-integration-explained/

53. **Microsoft Foundry: SharePoint Knowledge Integration** -- Origin Digital  
    https://www.origindigital.com/insights/sharepoint-knowledge-connections-to-foundry-agents-the-easy-way

54. **Real-time Enterprise RAG with SharePoint** -- Pathway  
    https://pathway.com/framework/blog/enterprise_rag_sharepoint

### Video Resources

55. **Introduction to Foundry IQ (YouTube)**  
    https://www.youtube.com/watch?v=slDdNIQCJBQ

56. **Foundry IQ Deep Dive (YouTube)**  
    https://www.youtube.com/watch?v=uDVkcZwB0EU

57. **Foundry IQ Portal Demo (YouTube)**  
    https://www.youtube.com/watch?v=bHL1jbWjJUc

58. **How to Deploy Azure AI Agent in Microsoft Teams (YouTube)**  
    https://www.youtube.com/watch?v=HU5sAsD1DYw
