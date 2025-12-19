# Mission-Critical SaaS: API Gateway and AI Gateway Architecture with Azure API Management

## Executive Summary

This document provides comprehensive architectural guidance for implementing both an **API Gateway** and a dedicated **AI Gateway** in a mission-critical SaaS application hosted on Azure. The solution leverages **Azure Container Apps** for microservices, with **Azure API Management (APIM)** serving as the foundation for both gateways. This architecture addresses multi-region deployment for high availability and disaster recovery, priority-based AI request handling, and separation of concerns between general API traffic and AI/GenAI workloads.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Architecture Decision: Combined vs. Separate Gateways](#architecture-decision-combined-vs-separate-gateways)
3. [Recommended Architecture: Hybrid Approach](#recommended-architecture-hybrid-approach)
4. [Multi-Region Deployment Strategy](#multi-region-deployment-strategy)
5. [AI Gateway Design Patterns](#ai-gateway-design-patterns)
6. [Priority and Request Handling for AI Services](#priority-and-request-handling-for-ai-services)
7. [Load Balancing and Resilience](#load-balancing-and-resilience)
8. [Security Considerations](#security-considerations)
9. [Monitoring and Observability](#monitoring-and-observability)
10. [Implementation Guidance](#implementation-guidance)
11. [References](#references)

---

## Architecture Overview

### High-Level Architecture

```mermaid
flowchart TB
    subgraph External["External Clients"]
        EC1[Web Apps]
        EC2[Mobile Apps]
        EC3[Partner APIs]
        EC4[External AI Consumers]
    end

    subgraph GlobalRouting["Global Traffic Routing"]
        AFD[Azure Front Door]
    end

    subgraph Region1["Primary Region - East US"]
        subgraph APIM1["Azure API Management Premium"]
            AG1[API Gateway<br/>External APIs]
            AIG1[AI Gateway<br/>GenAI Services]
        end
        
        subgraph ACA1["Azure Container Apps Environment"]
            MS1[Microservice A]
            MS2[Microservice B]
            MS3[Microservice C]
            MS4[AI Orchestrator Service]
        end
        
        subgraph AI1["Azure OpenAI Services"]
            AOAI1_PTU[Azure OpenAI<br/>PTU Instance]
            AOAI1_PAYG[Azure OpenAI<br/>PAYG Instance]
        end
    end

    subgraph Region2["Secondary Region - West US"]
        subgraph APIM2["Azure API Management Premium"]
            AG2[API Gateway<br/>External APIs]
            AIG2[AI Gateway<br/>GenAI Services]
        end
        
        subgraph ACA2["Azure Container Apps Environment"]
            MS1B[Microservice A]
            MS2B[Microservice B]
            MS3B[Microservice C]
            MS4B[AI Orchestrator Service]
        end
        
        subgraph AI2["Azure OpenAI Services"]
            AOAI2_PTU[Azure OpenAI<br/>PTU Instance]
            AOAI2_PAYG[Azure OpenAI<br/>PAYG Instance]
        end
    end

    EC1 --> AFD
    EC2 --> AFD
    EC3 --> AFD
    EC4 --> AFD
    
    AFD --> AG1
    AFD --> AG2
    AFD --> AIG1
    AFD --> AIG2
    
    AG1 --> MS1
    AG1 --> MS2
    AG1 --> MS3
    
    AIG1 --> MS4
    MS4 --> AIG1
    AIG1 --> AOAI1_PTU
    AIG1 --> AOAI1_PAYG
    
    MS1 --> AIG1
    MS2 --> AIG1
    MS3 --> AIG1
    
    AG2 --> MS1B
    AG2 --> MS2B
    AG2 --> MS3B
    
    AIG2 --> MS4B
    MS4B --> AIG2
    AIG2 --> AOAI2_PTU
    AIG2 --> AOAI2_PAYG
    
    MS1B --> AIG2
    MS2B --> AIG2
    MS3B --> AIG2
```

### Key Components

| Component | Purpose | Azure Service |
|-----------|---------|---------------|
| **Global Traffic Router** | Latency-based routing, failover, WAF | Azure Front Door |
| **API Gateway** | External/internal API management, routing, security | Azure API Management Premium |
| **AI Gateway** | GenAI request management, load balancing, token limits | Azure API Management Premium |
| **Microservices Platform** | Containerized workloads | Azure Container Apps |
| **AI Services** | LLM inference, embeddings | Azure OpenAI Service |

---

## Architecture Decision: Combined vs. Separate Gateways

### Option 1: Single Combined Gateway

A single APIM instance handles both traditional API traffic and AI/GenAI traffic.

```mermaid
flowchart LR
    subgraph Clients
        C1[External Clients]
        C2[Internal Services]
    end

    subgraph CombinedGateway["Single APIM Instance"]
        APIs[Traditional APIs]
        AIAPIs[AI APIs]
    end

    subgraph Backend
        MS[Microservices]
        AI[Azure OpenAI]
    end

    C1 --> CombinedGateway
    C2 --> CombinedGateway
    APIs --> MS
    AIAPIs --> AI
```

**Pros:**
- Simpler management with single control plane
- Lower operational overhead
- Unified monitoring and logging
- Cost-effective for smaller deployments

**Cons:**
- Risk of noisy neighbor issues between AI and regular API traffic
- AI workloads may consume disproportionate resources
- Difficult to apply different SLAs and rate limiting strategies
- Scaling constraints (AI traffic spikes affect all APIs)

---

### Option 2: Fully Separate Gateways

Two completely independent APIM instances - one for APIs, one for AI.

```mermaid
flowchart LR
    subgraph Clients
        C1[External Clients]
        C2[Internal Services]
    end

    subgraph APIGateway["APIM Instance 1"]
        APIs[API Gateway]
    end

    subgraph AIGateway["APIM Instance 2"]
        AIAPIs[AI Gateway]
    end

    subgraph Backend
        MS[Microservices]
        AI[Azure OpenAI]
    end

    C1 --> APIGateway
    C1 --> AIGateway
    C2 --> AIGateway
    APIs --> MS
    AIAPIs --> AI
    MS -.-> AIGateway
```

**Pros:**
- Complete isolation between workloads
- Independent scaling for AI-specific demands
- Separate rate limiting and quota management
- Different security policies per gateway
- Easier to implement AI-specific features

**Cons:**
- Higher cost (two Premium APIM instances per region)
- More complex management
- Duplicate configuration for common policies
- Multiple endpoints for clients to manage

---

### Option 3: Hybrid Approach (Recommended)

Single APIM instance with logical separation using **Products**, **Workspaces**, or **distinct API versioning**, combined with dedicated backend pools.

```mermaid
flowchart TB
    subgraph ExternalZone["External Access Zone"]
        AFD[Azure Front Door<br/>+ WAF]
    end

    subgraph APIM["Azure API Management Premium<br/>Multi-Region Deployment"]
        subgraph Products["Logical Separation via Products/Workspaces"]
            P1[Product: External APIs<br/>Rate Limits: Standard]
            P2[Product: AI Services<br/>Rate Limits: Token-based]
            P3[Product: Internal APIs<br/>Rate Limits: High throughput]
        end
        
        subgraph Backends["Backend Pools"]
            BP1[Backend Pool: Microservices]
            BP2[Backend Pool: Azure OpenAI<br/>PTU Priority + PAYG Spillover]
        end
    end

    subgraph Internal["Internal Services Zone"]
        ACA[Azure Container Apps<br/>Microservices]
    end

    subgraph AIServices["AI Services Zone"]
        AOAI1[Azure OpenAI PTU]
        AOAI2[Azure OpenAI PAYG]
    end

    AFD --> P1
    AFD --> P2
    P1 --> BP1
    P2 --> BP2
    P3 --> BP1
    P3 --> BP2
    
    BP1 --> ACA
    BP2 --> AOAI1
    BP2 --> AOAI2
    
    ACA -.->|Internal AI Requests| P3
```

**Pros:**
- Single control plane with logical isolation
- Cost-effective (one Premium instance per region)
- Flexible product-based access control
- Centralized monitoring with workload segregation
- Ability to apply AI-specific policies per product

**Cons:**
- Requires careful capacity planning
- More complex policy configuration
- Shared infrastructure (though logically separated)

---

## Recommended Architecture: Hybrid Approach

For mission-critical SaaS applications, the **Hybrid Approach** provides the best balance of cost, manageability, and separation of concerns.

### Architecture Details

```mermaid
flowchart TB
    subgraph ExternalClients["External Clients"]
        WEB[Web Applications]
        MOBILE[Mobile Apps]
        PARTNER[Partner Systems]
    end

    subgraph GlobalLayer["Global Routing Layer"]
        AFD["Azure Front Door Premium<br/>• WAF Protection<br/>• SSL Termination<br/>• Health Probes<br/>• Latency-based Routing"]
    end

    subgraph PrimaryRegion["Primary Region (East US)"]
        subgraph APIMPrimary["Azure API Management Premium"]
            subgraph ExtAPIs["External API Product"]
                EA1[/orders API/]
                EA2[/products API/]
                EA3[/customers API/]
            end
            
            subgraph AIProduct["AI Gateway Product"]
                AI1[/chat/completions/]
                AI2[/embeddings/]
                AI3[/assistants/]
            end
            
            subgraph IntAPIs["Internal API Product"]
                IA1[/internal/ai/]
                IA2[/internal/workflow/]
            end
        end

        subgraph ACAEnv1["Container Apps Environment"]
            SVC1[Order Service]
            SVC2[Product Service]
            SVC3[Customer Service]
            SVC4[AI Orchestrator]
        end

        subgraph AOAIPrimary["Azure OpenAI"]
            PTU1["PTU Deployment<br/>gpt-4 (High Priority)"]
            PAYG1["PAYG Deployment<br/>gpt-4 (Spillover)"]
        end
    end

    subgraph SecondaryRegion["Secondary Region (West US)"]
        subgraph APIMSecondary["Azure API Management Premium<br/>(Same Instance - Multi-Region)"]
            ExtAPIs2[External APIs]
            AIProduct2[AI Gateway]
            IntAPIs2[Internal APIs]
        end

        subgraph ACAEnv2["Container Apps Environment"]
            SVC1B[Order Service]
            SVC2B[Product Service]
            SVC3B[Customer Service]
            SVC4B[AI Orchestrator]
        end

        subgraph AOAISecondary["Azure OpenAI"]
            PTU2["PTU Deployment<br/>gpt-4 (High Priority)"]
            PAYG2["PAYG Deployment<br/>gpt-4 (Spillover)"]
        end
    end

    WEB --> AFD
    MOBILE --> AFD
    PARTNER --> AFD
    
    AFD -->|"Low Latency"| APIMPrimary
    AFD -->|"Failover"| APIMSecondary
    
    ExtAPIs --> SVC1
    ExtAPIs --> SVC2
    ExtAPIs --> SVC3
    
    AIProduct --> PTU1
    AIProduct -.->|"429 Spillover"| PAYG1
    
    SVC4 -->|"Internal AI Calls"| IntAPIs
    IntAPIs --> PTU1
    
    ExtAPIs2 --> SVC1B
    ExtAPIs2 --> SVC2B
    ExtAPIs2 --> SVC3B
    
    AIProduct2 --> PTU2
    AIProduct2 -.->|"429 Spillover"| PAYG2
    
    SVC4B --> IntAPIs2
    IntAPIs2 --> PTU2
```

### Product Configuration Strategy

| Product | Target Consumers | Rate Limiting | Features |
|---------|------------------|---------------|----------|
| **External APIs** | External clients, partners | Requests/sec per subscription | OAuth 2.0, API keys, standard throttling |
| **AI Gateway (External)** | External AI consumers | Token-based (TPM) limits | Semantic caching, content safety, priority queuing |
| **Internal APIs** | Backend microservices | Higher limits, service identity | Managed identity auth, circuit breaker |

---

## Multi-Region Deployment Strategy

### Active-Active Multi-Region Configuration

For mission-critical workloads requiring 99.99%+ SLA, deploy APIM Premium with multi-region gateways.

```mermaid
flowchart TB
    subgraph Global["Global Resources"]
        AFD[Azure Front Door]
        DNS[Azure DNS]
    end

    subgraph EastUS["East US (Primary)"]
        APIM_E["APIM Gateway<br/>3 Units + AZ"]
        ACA_E[Container Apps]
        AOAI_E[Azure OpenAI]
        COSMOS_E[(Cosmos DB<br/>Multi-Region Write)]
    end

    subgraph WestUS["West US (Secondary)"]
        APIM_W["APIM Gateway<br/>3 Units + AZ"]
        ACA_W[Container Apps]
        AOAI_W[Azure OpenAI]
    end

    subgraph WestEurope["West Europe (Tertiary)"]
        APIM_EU["APIM Gateway<br/>2 Units + AZ"]
        ACA_EU[Container Apps]
        AOAI_EU[Azure OpenAI]
    end

    AFD --> |"Latency Routing"| APIM_E
    AFD --> |"Latency Routing"| APIM_W
    AFD --> |"Latency Routing"| APIM_EU
    
    APIM_E --> ACA_E
    APIM_E --> AOAI_E
    ACA_E --> COSMOS_E
    
    APIM_W --> ACA_W
    APIM_W --> AOAI_W
    ACA_W --> COSMOS_E
    
    APIM_EU --> ACA_EU
    APIM_EU --> AOAI_EU
    ACA_EU --> COSMOS_E
```

### Region-Aware Backend Routing

Use APIM policies to route requests to regional backend services:

```xml
<policies>
    <inbound>
        <base />
        <choose>
            <!-- Route to regional Azure OpenAI based on gateway region -->
            <when condition="@("East US".Equals(context.Deployment.Region, StringComparison.OrdinalIgnoreCase))">
                <set-backend-service base-url="https://aoai-eastus.openai.azure.com/" />
            </when>
            <when condition="@("West US".Equals(context.Deployment.Region, StringComparison.OrdinalIgnoreCase))">
                <set-backend-service base-url="https://aoai-westus.openai.azure.com/" />
            </when>
            <when condition="@("West Europe".Equals(context.Deployment.Region, StringComparison.OrdinalIgnoreCase))">
                <set-backend-service base-url="https://aoai-westeurope.openai.azure.com/" />
            </when>
            <otherwise>
                <set-backend-service base-url="https://aoai-eastus.openai.azure.com/" />
            </otherwise>
        </choose>
    </inbound>
</policies>
```

---

## AI Gateway Design Patterns

### 1. Load Balancing with Circuit Breaker

```mermaid
flowchart LR
    subgraph AIGateway["AI Gateway (APIM)"]
        LB[Backend Load Balancer]
        CB[Circuit Breaker]
    end

    subgraph Backends["Azure OpenAI Backend Pool"]
        PTU1[PTU Instance 1<br/>Priority: 1]
        PTU2[PTU Instance 2<br/>Priority: 1]
        PAYG[PAYG Instance<br/>Priority: 2]
    end

    Request[AI Request] --> LB
    LB --> CB
    CB -->|"Healthy"| PTU1
    CB -->|"Healthy"| PTU2
    CB -.->|"Spillover/429"| PAYG
```

### Backend Pool Configuration

```json
{
  "backends": [
    {
      "url": "https://aoai-ptu-primary.openai.azure.com",
      "priority": 1,
      "weight": 50
    },
    {
      "url": "https://aoai-ptu-secondary.openai.azure.com",
      "priority": 1,
      "weight": 50
    },
    {
      "url": "https://aoai-payg-spillover.openai.azure.com",
      "priority": 2,
      "weight": 100
    }
  ],
  "circuitBreaker": {
    "rules": [
      {
        "failureCondition": {
          "count": 3,
          "interval": "PT10S",
          "statusCodeRanges": [
            { "min": 429, "max": 429 },
            { "min": 500, "max": 599 }
          ]
        },
        "tripDuration": "PT30S",
        "acceptRetryAfter": true
      }
    ]
  }
}
```

### 2. PTU to PAYG Spillover Strategy

```mermaid
sequenceDiagram
    participant Client
    participant AIGateway as AI Gateway
    participant PTU as Azure OpenAI (PTU)
    participant PAYG as Azure OpenAI (PAYG)

    Client->>AIGateway: POST /chat/completions
    AIGateway->>PTU: Forward Request
    
    alt PTU Available
        PTU-->>AIGateway: 200 OK + Response
        AIGateway-->>Client: 200 OK + Response
    else PTU Throttled (429)
        PTU-->>AIGateway: 429 Too Many Requests
        Note over AIGateway: Circuit Breaker Activates<br/>Route to PAYG
        AIGateway->>PAYG: Forward Request
        PAYG-->>AIGateway: 200 OK + Response
        AIGateway-->>Client: 200 OK + Response
    end
```

### 3. Token Rate Limiting

Apply token-based rate limiting per consumer:

```xml
<policies>
    <inbound>
        <base />
        <!-- Token limit policy for AI APIs -->
        <llm-token-limit 
            counter-key="@(context.Subscription.Id)" 
            tokens-per-minute="10000"
            estimate-prompt-tokens="true"
            remaining-tokens-variable-name="remainingTokens">
            <llm-token-limit-backend-id>aoai-backend</llm-token-limit-backend-id>
        </llm-token-limit>
    </inbound>
</policies>
```

---

## Priority and Request Handling for AI Services

### Priority Queue Architecture

```mermaid
flowchart TB
    subgraph Consumers["Request Sources"]
        HC[High Priority<br/>Critical Business Ops]
        MC[Medium Priority<br/>User-Facing Features]
        LC[Low Priority<br/>Batch Processing]
    end

    subgraph AIGateway["AI Gateway"]
        PQ[Priority Queue<br/>Classification]
        RL[Rate Limiter]
        CB[Circuit Breaker]
    end

    subgraph Processing["Backend Processing"]
        PTU[PTU Instances<br/>Reserved Capacity]
        PAYG[PAYG Instances<br/>Burst Capacity]
    end

    HC -->|"Priority: 1"| PQ
    MC -->|"Priority: 2"| PQ
    LC -->|"Priority: 3"| PQ
    
    PQ --> RL
    RL --> CB
    
    CB -->|"High Priority First"| PTU
    CB -.->|"Spillover"| PAYG
```

### Priority-Based Routing Policy

```xml
<policies>
    <inbound>
        <base />
        <!-- Extract priority from header or subscription -->
        <set-variable name="requestPriority" 
            value="@(context.Request.Headers.GetValueOrDefault("X-Priority", "medium"))" />
        
        <choose>
            <!-- High priority: Direct to PTU with no throttling -->
            <when condition="@(context.Variables.GetValueOrDefault<string>("requestPriority") == "high")">
                <set-backend-service backend-id="aoai-ptu-primary" />
                <set-header name="X-Route" exists-action="override">
                    <value>ptu-priority</value>
                </set-header>
            </when>
            
            <!-- Medium priority: PTU with spillover to PAYG -->
            <when condition="@(context.Variables.GetValueOrDefault<string>("requestPriority") == "medium")">
                <set-backend-service backend-id="aoai-backend-pool" />
            </when>
            
            <!-- Low priority: PAYG only, with aggressive rate limiting -->
            <otherwise>
                <rate-limit-by-key 
                    calls="10" 
                    renewal-period="60" 
                    counter-key="@(context.Subscription.Id)" />
                <set-backend-service backend-id="aoai-payg" />
            </otherwise>
        </choose>
    </inbound>
</policies>
```

### Consumer-Based Quota Allocation

| Consumer Type | TPM Quota | Priority | Backend Pool |
|---------------|-----------|----------|--------------|
| **Critical Operations** | 50,000 | High | PTU Only |
| **User-Facing Apps** | 20,000 | Medium | PTU + PAYG Spillover |
| **Batch Processing** | 5,000 | Low | PAYG Only |
| **Development/Test** | 1,000 | Low | PAYG (Shared) |

---

## Load Balancing and Resilience

### Multi-Backend Load Balancing

```mermaid
flowchart TB
    subgraph Gateway["AI Gateway"]
        LB["Load Balancer<br/>Round-Robin + Priority"]
    end

    subgraph PTUPool["PTU Backend Pool (Priority 1)"]
        PTU1["PTU East US<br/>Weight: 50%"]
        PTU2["PTU West US<br/>Weight: 50%"]
    end

    subgraph PAYGPool["PAYG Backend Pool (Priority 2)"]
        PAYG1["PAYG East US<br/>Weight: 50%"]
        PAYG2["PAYG West US<br/>Weight: 50%"]
    end

    LB -->|"Active"| PTU1
    LB -->|"Active"| PTU2
    LB -.->|"Spillover"| PAYG1
    LB -.->|"Spillover"| PAYG2
```

### Retry and Circuit Breaker Configuration

```xml
<policies>
    <backend>
        <retry condition="@(context.Response.StatusCode == 429 || context.Response.StatusCode >= 500)" 
               count="3" 
               interval="1" 
               delta="1" 
               max-interval="10" 
               first-fast-retry="true">
            <forward-request buffer-request-body="true" />
        </retry>
    </backend>
    
    <on-error>
        <base />
        <choose>
            <when condition="@(context.Response.StatusCode == 429)">
                <!-- Return Retry-After header to client -->
                <return-response>
                    <set-status code="429" reason="Too Many Requests" />
                    <set-header name="Retry-After" exists-action="override">
                        <value>@(context.Response.Headers.GetValueOrDefault("Retry-After", "30"))</value>
                    </set-header>
                </return-response>
            </when>
        </choose>
    </on-error>
</policies>
```

---

## Security Considerations

### Authentication Architecture

```mermaid
flowchart LR
    subgraph ExternalAuth["External Authentication"]
        OAuth[OAuth 2.0 / OIDC]
        APIKey[API Key]
    end

    subgraph Gateway["AI Gateway"]
        Validate[Token Validation]
        Transform[Credential Transform]
    end

    subgraph InternalAuth["Internal Authentication"]
        MI[Managed Identity]
    end

    subgraph Backend["Azure OpenAI"]
        AOAI[Azure OpenAI Service]
    end

    OAuth --> Validate
    APIKey --> Validate
    Validate --> Transform
    Transform --> MI
    MI --> AOAI
```

### Security Best Practices

1. **Terminate client credentials at the gateway** - Use managed identity for backend connections
2. **Apply Content Safety policies** - Integrate Azure AI Content Safety
3. **Implement PII detection** - Scan prompts before forwarding
4. **Network isolation** - Deploy APIM and backends in private virtual networks

### Content Safety Integration

```xml
<policies>
    <inbound>
        <base />
        <!-- Content Safety Check -->
        <llm-content-safety backend-id="content-safety-backend">
            <text-blocklist-ids>
                <id>custom-blocklist-1</id>
            </text-blocklist-ids>
            <categories>
                <category name="Hate" threshold="Medium" />
                <category name="Violence" threshold="Medium" />
                <category name="Sexual" threshold="Medium" />
                <category name="SelfHarm" threshold="Medium" />
            </categories>
        </llm-content-safety>
    </inbound>
</policies>
```

---

## Monitoring and Observability

### Observability Architecture

```mermaid
flowchart TB
    subgraph Gateway["AI Gateway"]
        Metrics[Token Metrics Emission]
        Logs[Request/Response Logging]
    end

    subgraph Monitoring["Azure Monitor"]
        AppInsights[Application Insights]
        LogAnalytics[Log Analytics]
        Alerts[Azure Alerts]
    end

    subgraph Dashboards["Visualization"]
        Workbooks[Azure Workbooks]
        Grafana[Azure Managed Grafana]
    end

    Gateway --> AppInsights
    Gateway --> LogAnalytics
    AppInsights --> Alerts
    LogAnalytics --> Alerts
    AppInsights --> Workbooks
    LogAnalytics --> Grafana
```

### Token Metrics Emission Policy

```xml
<policies>
    <outbound>
        <base />
        <!-- Emit token metrics for chargeback and monitoring -->
        <llm-emit-token-metric namespace="genai-metrics">
            <dimension name="Subscription" value="@(context.Subscription.Id)" />
            <dimension name="Product" value="@(context.Product.Name)" />
            <dimension name="API" value="@(context.Api.Name)" />
            <dimension name="Region" value="@(context.Deployment.Region)" />
            <dimension name="Model" value="@(context.Request.Headers.GetValueOrDefault("X-Model", "unknown"))" />
        </llm-emit-token-metric>
    </outbound>
</policies>
```

### Key Metrics to Monitor

| Metric | Description | Alert Threshold |
|--------|-------------|-----------------|
| **Total Tokens** | Total tokens consumed per subscription | 80% of quota |
| **Prompt Tokens** | Input tokens per request | Anomaly detection |
| **Completion Tokens** | Output tokens per request | Anomaly detection |
| **429 Rate** | Throttling frequency | > 5% of requests |
| **Latency P95** | 95th percentile response time | > 5 seconds |
| **Circuit Breaker Trips** | Backend failures | Any occurrence |

---

## Implementation Guidance

### Phase 1: Foundation (Weeks 1-2)

1. Deploy Azure API Management Premium in primary region
2. Configure availability zones (minimum 3 units)
3. Set up virtual network integration
4. Create Product structure (External APIs, AI Gateway, Internal APIs)

### Phase 2: AI Gateway Configuration (Weeks 3-4)

1. Import Azure OpenAI API definitions
2. Configure backend pools with PTU and PAYG instances
3. Implement load balancing policies
4. Set up circuit breaker rules

### Phase 3: Multi-Region Expansion (Weeks 5-6)

1. Add secondary region to APIM instance
2. Deploy regional Azure OpenAI instances
3. Configure region-aware routing policies
4. Set up Azure Front Door with health probes

### Phase 4: Security and Monitoring (Weeks 7-8)

1. Implement managed identity authentication
2. Configure content safety policies
3. Set up Application Insights integration
4. Create monitoring dashboards and alerts

### Deployment Checklist

- [ ] APIM Premium tier deployed with availability zones
- [ ] Multi-region gateways configured
- [ ] Backend pools defined for PTU/PAYG spillover
- [ ] Token rate limiting policies applied
- [ ] Circuit breaker configured
- [ ] Managed identity authentication enabled
- [ ] Content safety integration complete
- [ ] Monitoring and alerting configured
- [ ] Disaster recovery runbooks documented

---

## Summary

For your mission-critical SaaS application, the **Hybrid Approach** with a single APIM Premium instance provides:

| Requirement | Solution |
|-------------|----------|
| **High Availability** | Multi-region APIM deployment with active-active configuration |
| **Disaster Recovery** | Automatic failover via Azure Front Door |
| **AI Request Priority** | Product-based segregation with priority routing policies |
| **Cost Optimization** | PTU for predictable workloads, PAYG for spillover |
| **Security** | Managed identity, content safety, network isolation |
| **Observability** | Token metrics, request logging, alerting |

The architecture maintains logical separation between API Gateway and AI Gateway functionality while sharing infrastructure for cost efficiency and simplified management.

---

## References

1. [AI Gateway in Azure API Management](https://learn.microsoft.com/en-us/azure/api-management/genai-gateway-capabilities) - Microsoft Learn
2. [Use a Gateway in Front of Multiple Azure OpenAI Deployments](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/azure-openai-gateway-multi-backend) - Azure Architecture Center
3. [GenAI Gateway Reference Architecture using APIM](https://learn.microsoft.com/en-us/ai/playbook/solutions/generative-ai/genai-gateway/reference-architectures/apim-based) - AI Playbook
4. [Key Considerations for Designing a GenAI Gateway Solution](https://learn.microsoft.com/en-us/ai/playbook/solutions/generative-ai/genai-gateway/key-considerations) - AI Playbook
5. [Deploy Azure API Management to Multiple Regions](https://learn.microsoft.com/en-us/azure/api-management/api-management-howto-deploy-multi-region) - Microsoft Learn
6. [Reliability in Azure API Management](https://learn.microsoft.com/en-us/azure/reliability/reliability-api-management) - Microsoft Learn
7. [Azure API Management Landing Zone Architecture](https://learn.microsoft.com/en-us/azure/architecture/example-scenario/integration/app-gateway-internal-api-management-function) - Azure Architecture Center
8. [Mission-Critical Architecture Pattern](https://learn.microsoft.com/en-us/azure/well-architected/mission-critical/mission-critical-architecture-pattern) - Well-Architected Framework
9. [Microservices with Azure Container Apps](https://learn.microsoft.com/en-us/azure/container-apps/microservices) - Microsoft Learn
10. [API Gateway Pattern for Microservices](https://learn.microsoft.com/en-us/azure/architecture/microservices/design/gateway) - Azure Architecture Center

---

*Document Version: 1.0*  
*Last Updated: December 2024*  
*Author: Architecture Team*
